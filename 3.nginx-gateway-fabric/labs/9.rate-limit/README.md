# Lab 9 — Rate Limiting 활용사례

> HTTP Route와 gRPC Route에서 `RateLimitPolicy`를 사용해 요청 수를 제한하는 방법을 실습합니다.

---

## 실습 경로 이동

```bash
# 이전 Lab 디렉토리에 있다면
cd ../9.rate-limit/

# Lab 기본경로(2026-F5-DevOps-Academy)에 있다면
cd 3.nginx-gateway-fabric/labs/9.rate-limit
```

---

## Step 1 — 테스트 애플리케이션 배포

```bash
kubectl apply -f 0.apps.yaml
```

배포 상태가 `Running`인지 확인합니다.

```bash
kubectl get all
```

```
NAME                                READY   STATUS    RESTARTS   AGE
pod/coffee-56b44d4c55-gdxkc         1/1     Running   0          41s
pod/grpc-backend-68ff5cb6c9-c565t   1/1     Running   0          40s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
service/coffee         ClusterIP   10.97.137.125   <none>        80/TCP            41s
service/grpc-backend   ClusterIP   10.109.137.38   <none>        8080/TCP          40s
service/kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP           506d
service/nginx-svc      ClusterIP   10.101.127.99   <none>        80/TCP,8080/TCP   13d

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coffee         1/1     1            1           41s
deployment.apps/grpc-backend   1/1     1            1           40s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/coffee-56b44d4c55         1         1         1       41s
replicaset.apps/grpc-backend-68ff5cb6c9   1         1         1       40s
```

---

## Step 2 — Gateway 및 RateLimitPolicy 배포

`RateLimitPolicy`를 함께 포함한 Gateway를 배포합니다.

```bash
kubectl apply -f 1.gateway.yaml
```

### 데이터플레인 Pod 상태 확인

```bash
kubectl get pods
```

`gateway-nginx-7b79d89c-p8v8v`가 NGINX Gateway Fabric 데이터플레인 Pod입니다.

```
NAME                            READY   STATUS    RESTARTS   AGE
coffee-56b44d4c55-gdxkc         1/1     Running   0          89s
gateway-nginx-7b79d89c-p8v8v    1/1     Running   0          14s
grpc-backend-68ff5cb6c9-c565t   1/1     Running   0          88s
```

### Gateway 상태 확인

```bash
kubectl get gateway
```

```
NAME      CLASS   ADDRESS       PROGRAMMED   AGE
gateway   nginx   10.97.59.36   True         31s
```

### Service 상태 확인

`gateway-nginx`가 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```bash
kubectl get service
```

```
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
coffee          ClusterIP   10.97.137.125   <none>        80/TCP            13m
gateway-nginx   NodePort    10.97.59.36     <none>        80:32700/TCP      12m
grpc-backend    ClusterIP   10.109.137.38   <none>        8080/TCP          13m
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP           506d
nginx-svc       ClusterIP   10.101.127.99   <none>        80/TCP,8080/TCP   13d
```

### Gateway 레벨 RateLimitPolicy 확인

```bash
kubectl get ratelimitpolicy
```

```
NAME                 AGE
gateway-rate-limit   107s
```

`describe` 명령으로 상세 설정을 확인합니다. 현재 **Gateway 레벨**에서 **10 req/sec** (IP 기준)으로 설정되어 있습니다.

```bash
kubectl describe RateLimitPolicy gateway-rate-limit
```

```
Name:         gateway-rate-limit
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  gateway.nginx.org/v1alpha1
Kind:         RateLimitPolicy
Metadata:
  Creation Timestamp:  2026-02-05T17:27:25Z
  Generation:          1
  Resource Version:    94045758
  UID:                 dadb9661-dde0-429f-986a-67b797e4f97a
Spec:
  Rate Limit:
    Local:
      Rules:
        Key:        $binary_remote_addr
        Rate:       10r/s
        Zone Size:  10m
  Target Refs:
    Group:  gateway.networking.k8s.io
    Kind:   Gateway
    Name:   gateway
Status:
  Ancestors:
    Ancestor Ref:
      Group:      gateway.networking.k8s.io
      Kind:       Gateway
      Name:       gateway
      Namespace:  default
    Conditions:
      Last Transition Time:  2026-02-05T17:27:26Z
      Message:               The Policy is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
    Controller Name:         gateway.nginx.org/nginx-gateway-controller
Events:                      <none>
```

---

## Step 3 — HTTPRoute 및 gRPCRoute 배포

```bash
kubectl apply -f 2.routes.yaml
```

### HTTPRoute 확인

```bash
kubectl get httproute
```

```
NAME     HOSTNAMES              AGE
coffee   ["cafe.example.com"]   82s
```

### gRPCRoute 확인

```bash
kubectl get grpcroute
```

```
NAME         HOSTNAMES              AGE
grpc-route   ["grpc.example.com"]   113s
```

---

## Step 4 — Route 레벨 RateLimitPolicy 적용

`HTTPRoute`와 `gRPCRoute`에 Rate Limit 정책을 연결합니다.

```bash
kubectl apply -f 3.route-ratelimit.yaml
```

전체 RateLimitPolicy 목록을 확인합니다.

```bash
kubectl get ratelimitpolicy
```

```
NAME                 AGE
gateway-rate-limit   8m24s
route-rate-limit     81s
```

Route 레벨 정책의 상세 내용을 확인합니다. **HTTPRoute** 및 **gRPCRoute** 레벨에서 **1 req/sec**으로 설정된 것을 확인할 수 있습니다.

```bash
kubectl describe ratelimitpolicy route-rate-limit
```

```
Name:         route-rate-limit
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  gateway.nginx.org/v1alpha1
Kind:         RateLimitPolicy
Metadata:
  Creation Timestamp:  2026-02-05T17:34:28Z
  Generation:          1
  Resource Version:    94046926
  UID:                 2edd598e-fdc2-4c67-8742-13d409c8137d
Spec:
  Rate Limit:
    Local:
      Rules:
        Burst:      0
        Key:        $binary_remote_addr
        Rate:       1r/s
        Zone Size:  10m
  Target Refs:
    Group:  gateway.networking.k8s.io
    Kind:   HTTPRoute
    Name:   coffee
    Group:  gateway.networking.k8s.io
    Kind:   GRPCRoute
    Name:   grpc-route
Status:
  Ancestors:
    Ancestor Ref:
      Group:      gateway.networking.k8s.io
      Kind:       HTTPRoute
      Name:       coffee
      Namespace:  default
    Conditions:
      Last Transition Time:  2026-02-05T17:34:29Z
      Message:               The Policy is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
    Controller Name:         gateway.nginx.org/nginx-gateway-controller
    Ancestor Ref:
      Group:      gateway.networking.k8s.io
      Kind:       GRPCRoute
      Name:       grpc-route
      Namespace:  default
    Conditions:
      Last Transition Time:  2026-02-05T17:34:29Z
      Message:               The Policy is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
    Controller Name:         gateway.nginx.org/nginx-gateway-controller
Events:                      <none>
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

---

### HTTP Rate Limit 테스트

**단일 요청** — 정상 응답 확인

```bash
curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

```
HTTP/1.1 200 OK
Server: nginx
Date: Thu, 05 Feb 2026 17:41:00 GMT
Content-Type: text/plain
Content-Length: 162
Connection: keep-alive
Expires: Thu, 05 Feb 2026 17:40:59 GMT
Cache-Control: no-cache

Server address: 10.0.156.109:8080
Server name: coffee-56b44d4c55-gdxkc
Date: 05/Feb/2026:17:41:00 +0000
URI: /coffee
Request ID: 03f15068dc0890cc33ac117322041ecc
```

**연속 2회 요청** — Rate Limit 동작 확인

```bash
curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee;echo "---";curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

1 req/sec 제한으로 인해 두 번째 요청에서 `429 Too Many Requests`가 반환됩니다.

```
HTTP/1.1 200 OK
Server: nginx
Date: Thu, 05 Feb 2026 17:42:10 GMT
Content-Type: text/plain
Content-Length: 162
Connection: keep-alive
Expires: Thu, 05 Feb 2026 17:42:09 GMT
Cache-Control: no-cache

Server address: 10.0.156.109:8080
Server name: coffee-56b44d4c55-gdxkc
Date: 05/Feb/2026:17:42:10 +0000
URI: /coffee
Request ID: 19f252331dad133413ca1c7a395ab630
---
HTTP/1.1 429 Too Many Requests
Server: nginx
Date: Thu, 05 Feb 2026 17:42:10 GMT
Content-Type: text/html
Content-Length: 162
Connection: keep-alive

<html>
<head><title>429 Too Many Requests</title></head>
<body>
<center><h1>429 Too Many Requests</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

---

### gRPC Rate Limit 테스트

**단일 요청** — 정상 응답 확인

```bash
grpcurl -plaintext -proto grpc.proto -authority grpc.example.com -d '{"name": "exact"}' $NGF_IP:$HTTP_PORT helloworld.Greeter/SayHello
```

```json
{
  "message": "Hello exact"
}
```

**연속 2회 요청** — Rate Limit 동작 확인

```bash
grpcurl -plaintext -proto grpc.proto -authority grpc.example.com -d '{"name": "exact"}' $NGF_IP:$HTTP_PORT helloworld.Greeter/SayHello;echo "---";grpcurl -plaintext -proto grpc.proto -authority grpc.example.com -d '{"name": "exact"}' $NGF_IP:$HTTP_PORT helloworld.Greeter/SayHello
```

두 번째 요청은 NGINX에서 차단되어 `Unknown` 에러가 발생합니다. NGINX Gateway 로그에서도 동일한 내용을 확인할 수 있습니다.

```
{
  "message": "Hello exact"
}
---
ERROR:
  Code: Unknown
  Message: unexpected HTTP status code received from server: 204 (No Content)

(NGINX Gateway Logs)
10.1.1.10 - - [21/Mar/2026:14:01:33 +0000] "POST /helloworld.Greeter/SayHello HTTP/2.0" 200 18 "-" "grpcurl/v1.8.7 grpc-go/1.48.0"
2026/03/21 14:01:33 [info] 84#84: *146 recv() failed (104: Connection reset by peer) while processing HTTP/2 connection, client: 10.1.1.10, server: 0.0.0.0:80
2026/03/21 14:01:33 [error] 85#85: *148 limiting requests, excess: 0.973 by zone "default_rl_route-rate-limit_rule0", client: 10.1.1.10, server: grpc.example.com, request: "POST /helloworld.Greeter/SayHello HTTP/2.0", host: "grpc.example.com"
```

---

## 실습 종료 — 리소스 삭제

```bash
kubectl delete -f .
```

> 이번 NGINX Gateway Fabric 실습은 여기까지 마무리가 되었습니다. 수고하셨습니다.
