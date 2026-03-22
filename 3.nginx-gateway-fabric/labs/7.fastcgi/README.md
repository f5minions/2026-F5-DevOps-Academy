# Lab 7 — SnippetsFilter를 이용한 FastCGI 애플리케이션 배포

> `SnippetsFilter`를 통해 FastCGI 인터페이스를 설정하고, 샘플 PHP 애플리케이션을 NGINX Gateway Fabric으로 서비스하는 방법을 실습합니다.

---

## 실습 경로 이동

```bash
# 이전 Lab 디렉토리에 있다면
cd ../7.fastcgi/

# Lab 기본경로(2026-F5-DevOps-Academy)에 있다면
cd 3.nginx-gateway-fabric/labs/7.fastcgi
```

---

## Step 1 — PHP 애플리케이션 배포

```bash
kubectl apply -f 0.phpapp.yaml
```

배포 상태가 `Running`인지 확인합니다.

```bash
kubectl get all
```

```
NAME                           READY   STATUS    RESTARTS   AGE
pod/php-fpm-7f8d9d598c-wqsj9   1/1     Running   0          4s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    385d
service/php-fpm      ClusterIP   10.111.251.242   <none>        9000/TCP   4s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/php-fpm   1/1     1            1           4s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/php-fpm-7f8d9d598c   1         1         1       4s
```

---

## Step 2 — Gateway 배포

NGINX Gateway Fabric 데이터플레인을 현재 `namespace`에 배포합니다.

```bash
kubectl apply -f 1.gateway.yaml
```

### Pod 상태 확인

```bash
kubectl get pods
```

`gateway-nginx-56678b747f-f6h2w`(실제 배포 시 이름은 다를 수 있음)가 NGINX Gateway Fabric 데이터플레인입니다.

```
NAME                             READY   STATUS    RESTARTS   AGE
gateway-nginx-56678b747f-f6h2w   1/1     Running   0          82s
php-fpm-7f8d9d598c-wqsj9         1/1     Running   0          2m27s
```

### Gateway 상태 확인

```bash
kubectl get gateway
```

```
NAME      CLASS   ADDRESS         PROGRAMMED   AGE
gateway   nginx   10.96.234.240   True         113s
```

### Service 상태 확인

`gateway-nginx`가 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```bash
kubectl get service
```

```
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
gateway-nginx   NodePort    10.96.234.240    <none>        80:31933/TCP   2m23s
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        385d
php-fpm         ClusterIP   10.111.251.242   <none>        9000/TCP       3m27s
```

---

## Step 3 — SnippetsFilter 배포 (FastCGI 설정)

FastCGI 파라미터를 NGINX 설정에 삽입하는 `SnippetsFilter`를 생성합니다.

```bash
kubectl apply -f 2.snippetsfilter-fastcgi.yaml
```

설정 내용을 확인합니다.

```bash
kubectl describe snippetsfilter fastcgi
```

```
Name:         fastcgi
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  gateway.nginx.org/v1alpha1
Kind:         SnippetsFilter
Metadata:
  Creation Timestamp:  2025-10-07T08:10:17Z
  Generation:          1
  Resource Version:    66548537
  UID:                 50958b7f-ebb0-4b8a-b16c-e60794e97c57
Spec:
  Snippets:
    Context:  http.server.location
    Value:    location / { resolver kube-dns.kube-system.svc.cluster.local;fastcgi_param SCRIPT_FILENAME /var/www/html/public/index.php;fastcgi_param DOCUMENT_ROOT /var/www/html/public;fastcgi_param QUERY_STRING $args;fastcgi_param REQUEST_METHOD $request_method;fastcgi_param CONTENT_TYPE $content_type;fastcgi_param CONTENT_LENGTH $content_length;fastcgi_param PATH_INFO $uri;fastcgi_param PATH_TRANSLATED /var/www/html/public$uri;fastcgi_index index.php;fastcgi_buffer_size 32k;fastcgi_buffers 16 16k;fastcgi_pass php-fpm.default.svc.cluster.local:9000;}
Events:       <none>
```

위 설정이 NGINX Gateway Fabric 데이터플레인에 실제로 적용되는 내용은 다음과 같습니다.

```nginx
location / {
    # 1. DNS 설정
    resolver kube-dns.kube-system.svc.cluster.local;

    # 2. FastCGI 파라미터 (PHP-FPM 전달 정보)
    fastcgi_param SCRIPT_FILENAME     /var/www/html/public/index.php;
    fastcgi_param DOCUMENT_ROOT       /var/www/html/public;
    fastcgi_param QUERY_STRING        $args;
    fastcgi_param REQUEST_METHOD      $request_method;
    fastcgi_param CONTENT_TYPE        $content_type;
    fastcgi_param CONTENT_LENGTH      $content_length;
    fastcgi_param PATH_INFO           $uri;
    fastcgi_param PATH_TRANSLATED     /var/www/html/public$uri;

    # 3. FastCGI 기본 설정
    fastcgi_index  index.php;

    # 4. 버퍼 설정
    fastcgi_buffer_size  32k;
    fastcgi_buffers      16 16k;

    # 5. PHP-FPM 연결
    fastcgi_pass  php-fpm.default.svc.cluster.local:9000;
}
```

---

## Step 4 — HTTPRoute 배포

`SnippetsFilter`를 참조하는 HTTPRoute를 배포합니다.

```bash
kubectl apply -f 3.httproute.yaml
```

```bash
kubectl get httproute
```

```
NAME      HOSTNAMES             AGE
php-fpm   ["php.example.com"]   13s
```

---

## Step 5 — 테스트

### 환경 변수 설정

```bash
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc gateway-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

```bash
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

### PHP 애플리케이션 접속

```bash
curl -si --resolve php.example.com:$HTTP_PORT:$NGF_IP http://php.example.com:$HTTP_PORT/phpinfo.php
```

PHP 정보 페이지가 정상적으로 반환되면 FastCGI 설정이 완료된 것입니다.

```
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 07 Oct 2025 08:28:49 GMT
Content-Type: text/html; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
X-Powered-By: PHP/8.2.29

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml"><head>
<style type="text/css">
body {background-color: #fff; color: #222; font-family: sans-serif;}
pre {margin: 0; font-family: monospace;}
[...]
```

---

## 실습 종료 — 리소스 삭제

```bash
kubectl delete -f .
```
