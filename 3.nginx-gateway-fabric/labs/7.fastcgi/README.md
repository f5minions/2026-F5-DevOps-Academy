# Publishing a FastCGI application using SnippetsFilter

이번 실습에서는 SnippetsFilter를 통해 FastCGI 인터페이스를 사용하여 샘플 PHP 애플리케이션을 게시하는 방법을 경험할 수 있습니다.

Lab 진행을 위한 실습 경로로 이동합니다.

```code
이전 Lab 디렉토리에 있다면,
cd ../7.fastcgi/

Lab 기본경로(2026-F5-DevOps-Academy)에 있다면,
cd 3.nginx-gateway-fabric/labs/7.fastcgi
```

예제 PHP 애플리케이션을 먼저 배포합니다.

```code
kubectl apply -f 0.phpapp.yaml
```

배포한 PHP 애플리케이션의 `Running` 상태를 확인합니다.

```code
kubectl get all
```

아래와 유사한 결과를 확인할 수 있습니다.

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

애플리케이션 딜리버리 처리를 위한 Gateway 오브젝트를 생성합니다.  NGINX Gateway Fabric 데이터플레인 pod를 동일한 namespace에 배포합니다. 

```code
kubectl apply -f 1.gateway.yaml
```

배포한 NGINX Gateway Fabric 데이터플레인 pod의 상태를 확인합니다.

```
kubectl get pods
```

`gateway-nginx-56678b747f-f6h2w`(실제 배포 시 동일한 이름은 아님)가 NGINX Gateway Fabric 데이터플레인입니다.

```
NAME                             READY   STATUS    RESTARTS   AGE
gateway-nginx-56678b747f-f6h2w   1/1     Running   0          82s
php-fpm-7f8d9d598c-wqsj9         1/1     Running   0          2m27s
```

배포한 gateway를 확인합니다.

```code
kubectl get gateway
```

아래와 같은 결과를 확인할 수 있습니다.

```code
NAME      CLASS   ADDRESS         PROGRAMMED   AGE
gateway   nginx   10.96.234.240   True         113s
```

배포한 NGINX Gateway Fabric 데이터플레인 서비스를 확인합니다.

```code
kubectl get service
```

`cafe-nginx` 가 NGINX Gateway Fabric 데이터플레인 서비스입니다.

```code
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
gateway-nginx   NodePort    10.96.234.240    <none>        80:31933/TCP   2m23s
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        385d
php-fpm         ClusterIP   10.111.251.242   <none>        9000/TCP       3m27s
```

FastCGI 설정 스니펫을 구성하기 위해 SnippetsFilter를 생성합니다.

```code
kubectl apply -f 2.snippetsfilter-fastcgi.yaml
```

생성한 SnippetFilter를 확인합니다.

```code
kubectl describe snippetsfilter fastcgi
```

아래와 유사한 결과를 확인할 수 있습니다.

```code
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

이 설정으로 실제 Gateway Fabric 데이터플레인에는 다음과 같이 설정이 됩니다.
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

설정한 SnippetsFilter를 참조하는 HTTP Route를 생성합니다. 

```code
kubectl apply -f 3.httproute.yaml
```

HTTP route를 확인합니다.

```code
kubectl get httproute
```

아래와 같은 결과를 확인할 수 있습니다.

```code
NAME      HOSTNAMES             AGE
php-fpm   ["php.example.com"]   13s
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP 주소와 HTTP PORT 정보를 확인 후 변수로 저장합니다.

```code
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc gateway-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

저장된 NGINX Gateway Fabric 데이터플레인 인스턴스의 IP와 HTTP PORT정보를 확인합니다.

```code
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

애플리케이션 딜리버리와 관련된 설정은 모두 완료가 되었고, 이제 PHP application으로 접속을 시도합니다.

```code
curl -si --resolve php.example.com:$HTTP_PORT:$NGF_IP http://php.example.com:$HTTP_PORT/phpinfo.php
```

결과를 아래와 유사하게 출력됩니다.

```code
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
<tr class="v"><td>
<p>
This program is free software; you can redistribute it and/or modify it under the terms of the PHP License as published by the PHP Group and included in the distribution in the file:  LICENSE
</p>
<p>This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
</p>
<p>If you did not receive a copy of the PHP license, or have any questions about PHP licensing, please contact license@php.net.
</p>
</td></tr>
</table>
</div></body></html>
```

모두 확인이 완료되었다면, 다음 실습을 위해 이번 랩의 설정을 모두 삭제합니다.

```code
kubectl delete -f .
```
