# Lab 2 — HTTP 매칭 컨디션 기반 고급 라우팅

> HTTP URI, Query String, Header, Method 등 다양한 매칭 컨디션을 활용하여 원하는 애플리케이션으로 트래픽을 라우팅하는 방법을 실습합니다.

---

## 실습 경로 이동

```bash
# 이전 Lab 디렉토리에 있다면
cd ../3.nginx-gateway-fabric/labs/2.advanced-routing

# 실습 메인 디렉토리(2026-F5-DevOps-Academy)에 있다면
cd 3.nginx-gateway-fabric/labs/2.advanced-routing
```

---

## Step 1 — 테스트 애플리케이션 배포

```bash
kubectl apply -f 0.coffee.yaml
kubectl apply -f 1.tea.yaml
```

배포 상태가 `Running`인지 확인합니다.

```bash
kubectl get all
```

```
NAME                              READY   STATUS    RESTARTS   AGE
pod/cafe-nginx-7444846d75-cgmms   1/1     Running   0          91s
pod/coffee-v1-c48b96b65-5trnr     1/1     Running   0          91s
pod/coffee-v2-685fd9bb65-dz5pp    1/1     Running   0          91s
pod/coffee-v3-7fb98466f-478hw     1/1     Running   0          91s
pod/tea-596697966f-hzjw5          1/1     Running   0          91s
pod/tea-post-5647b8d885-5xxvf     1/1     Running   0          91s

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/cafe-nginx      NodePort    10.103.90.239    <none>        80:31436/TCP   91s
service/coffee-v1-svc   ClusterIP   10.107.70.64     <none>        80/TCP         91s
service/coffee-v2-svc   ClusterIP   10.102.153.99    <none>        80/TCP         91s
service/coffee-v3-svc   ClusterIP   10.110.117.58    <none>        80/TCP         91s
service/kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        268d
service/tea-post-svc    ClusterIP   10.105.108.172   <none>        80/TCP         91s
service/tea-svc         ClusterIP   10.102.222.60    <none>        80/TCP         91s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cafe-nginx   1/1     1            1           91s
deployment.apps/coffee-v1    1/1     1            1           91s
deployment.apps/coffee-v2    1/1     1            1           91s
deployment.apps/coffee-v3    1/1     1            1           91s
deployment.apps/tea          1/1     1            1           91s
deployment.apps/tea-post     1/1     1            1           91s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/cafe-nginx-7444846d75   1         1         1       91s
replicaset.apps/coffee-v1-c48b96b65     1         1         1       91s
replicaset.apps/coffee-v2-685fd9bb65    1         1         1       91s
replicaset.apps/coffee-v3-7fb98466f     1         1         1       91s
replicaset.apps/tea-596697966f          1         1         1       91s
replicaset.apps/tea-post-5647b8d885     1         1         1       91s
```

---

## Step 2 — Gateway 배포

NGINX Gateway Fabric 데이터플레인을 현재 `namespace`에 배포합니다.

```bash
kubectl apply -f 2.gateway.yaml
```

### Pod 상태 확인

```bash
kubectl get pods
```

`cafe-nginx-7444846d75-cgmms`는 NGINX Gateway Fabric 데이터플레인 Pod입니다.

```
NAME                          READY   STATUS    RESTARTS   AGE
cafe-nginx-7444846d75-cgmms   1/1     Running   0          113s
coffee-v1-c48b96b65-5trnr     1/1     Running   0          113s
coffee-v2-685fd9bb65-dz5pp    1/1     Running   0          113s
coffee-v3-7fb98466f-478hw     1/1     Running   0          113s
tea-596697966f-hzjw5          1/1     Running   0          113s
tea-post-5647b8d885-5xxvf     1/1     Running   0          113s
```

### Gateway 상태 확인

```bash
kubectl get gateway
```

```
NAME   CLASS   ADDRESS         PROGRAMMED   AGE
cafe   nginx   10.103.90.239   True         2m43s
```

### Service 상태 확인

`cafe-nginx`는 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```bash
kubectl get service
```

```
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
cafe-nginx      NodePort    10.103.90.239    <none>        80:31436/TCP   28s
coffee-v1-svc   ClusterIP   10.107.70.64     <none>        80/TCP         28s
coffee-v2-svc   ClusterIP   10.102.153.99    <none>        80/TCP         28s
coffee-v3-svc   ClusterIP   10.110.117.58    <none>        80/TCP         28s
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        268d
tea-post-svc    ClusterIP   10.105.108.172   <none>        80/TCP         28s
tea-svc         ClusterIP   10.102.222.60    <none>        80/TCP         28s
```

---

## Step 3 — HTTPRoute 배포

고급 라우팅 설정을 배포합니다. 배포 전 `3.cafe-routes.yaml` 파일의 내용을 확인해보시기 바랍니다.

```bash
kubectl apply -f 3.cafe-routes.yaml
```

```bash
kubectl get httproute
```

```
NAME     HOSTNAMES              AGE
coffee   ["cafe.example.com"]   8s
tea      ["cafe.example.com"]   8s
```

---

## Step 4 — 테스트

### 환경 변수 설정

```bash
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc cafe-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

```bash
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

---

### coffee-v1 — 기본 URI 매칭

```bash
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

```
Server address: 10.0.156.109:8080
Server name: coffee-v1-c48b96b65-5trnr
Date: 12/Jun/2025:11:00:28 +0000
URI: /coffee
Request ID: 1a0b8a08ec4f94f6a5f02e7649165b18
```

### coffee-v2 — Query String 매칭

URI 쿼리스트링에 버전 정보를 포함하여 요청합니다.

```bash
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee?TEST=v2
```

```
Server address: 10.0.156.121:8080
Server name: coffee-v2-685fd9bb65-dz5pp
Date: 12/Jun/2025:11:00:44 +0000
URI: /coffee?TEST=v2
Request ID: eac31b251dfdf398033d8e8373df14c9
```

### coffee-v2 — Header 매칭

HTTP 헤더에 버전 정보를 포함하여 요청합니다.

```bash
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee -H "version: v2"
```

```
Server address: 10.0.156.121:8080
Server name: coffee-v2-685fd9bb65-dz5pp
Date: 12/Jun/2025:11:01:00 +0000
URI: /coffee
Request ID: 0f5cd7a2f62965279c4bdc52c660c97d
```

### coffee-v3 — Query String Regex 매칭

```bash
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee?queryRegex=query-a
```

```
Server address: 192.168.169.141:8080
Server name: coffee-v3-7fb98466f-tgq8k
Date: 18/Sep/2025:21:26:26 +0000
URI: /coffee?queryRegex=query-a
Request ID: 2c755a8391ebd2df87f416510fb5478b
```

### coffee-v3 — Header Regex 매칭

```bash
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee -H "headerRegex: header-a"
```

```
Server address: 192.168.169.141:8080
Server name: coffee-v3-7fb98466f-tgq8k
Date: 18/Sep/2025:21:25:13 +0000
URI: /coffee
Request ID: 81431954b4b52edc01d707b4e5822792
```

### tea — GET 메소드

```bash
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/tea
```

```
Server address: 10.0.156.108:8080
Server name: tea-596697966f-hzjw5
Date: 12/Jun/2025:11:01:13 +0000
URI: /tea
Request ID: 7a45019ce6b1380b5d5402be89103703
```

### tea-post — POST 메소드 매칭

동일한 `/tea` 경로에 `POST` 메소드로 요청하면 `tea-post` 애플리케이션으로 라우팅됩니다.

```bash
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/tea -X POST
```

```
Server address: 10.0.156.122:8080
Server name: tea-post-5647b8d885-5xxvf
Date: 12/Jun/2025:11:01:32 +0000
URI: /tea
Request ID: 92c6bb8c35b24c1ca0e68eaaf4bbbf40
```

---

## 실습 종료 — 리소스 삭제

```bash
kubectl delete -f .
```
