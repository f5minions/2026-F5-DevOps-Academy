# Rate Limiting 활용사례

이번 실습에서는 HTTP와 gRPC route에서 어떻게 요청의 제한(limits)을 적용할 수 있는가에 대해서 알아보겠습니다.

이번 랩의 진행을 위해 해당 디렉토리로 이동합니다.

```code
이전 Lab 디렉토리에 있다면,
cd ../9.rate-limit/

Lab 기본경로(2026-F5-DevOps-Academy)에 있다면,
cd 3.nginx-gateway-fabric/labs/9.rate-limit
```

실습을 위한 테스트 애플리케이션을 먼저 배포합니다.

```code
kubectl apply -f 0.apps.yaml
```

배포한 테스트 애플리케이션의 `running` 상태를 확인합니다. 

```code
kubectl get all
```

결과는 아래와 유사합니다. 

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

이제 동일한 namespace에  `RateLimitPolicy`를 함께 적용한 NGINX Gateway Fabric 데이터플레인을 배포하여 gateway 오브젝트로 생성합니다. 

```code
kubectl apply -f 1.gateway.yaml
```

NGINX Gateway Fabric 데이터플레인 pod 배포 상태를 확인합니다.

```
kubectl get pods
```

`gateway-nginx-7b79d89c-p8v8v` 가 NGINX Gateway Fabric 데이터플레인 pod 입니다.

```
NAME                            READY   STATUS    RESTARTS   AGE
coffee-56b44d4c55-gdxkc         1/1     Running   0          89s
gateway-nginx-7b79d89c-p8v8v    1/1     Running   0          14s
grpc-backend-68ff5cb6c9-c565t   1/1     Running   0          88s
```

Gateway 배포를 확인합니다.

```code
kubectl get gateway
```

아래의 결과와 유사하게 확인이 되어야 합니다.

```code
NAME      CLASS   ADDRESS       PROGRAMMED   AGE
gateway   nginx   10.97.59.36   True         31s
```

동일하게 NGINX Gateway Fabric 데이터플레인 서비스 상태도 함께 확인합니다.

```code
kubectl get service
```

`gateway-nginx`가 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```code
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
coffee          ClusterIP   10.97.137.125   <none>        80/TCP            13m
gateway-nginx   NodePort    10.97.59.36     <none>        80:32700/TCP      12m
grpc-backend    ClusterIP   10.109.137.38   <none>        8080/TCP          13m
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP           506d
nginx-svc       ClusterIP   10.101.127.99   <none>        80/TCP,8080/TCP   13d
```

Gateway에 함께 배포된 rate limit policy의 배포상태도 확인합니다. 

```code
kubectl get ratelimitpolicy
```

아래와 같은 결과를 확인할 수 있습니다.

```code
NAME                 AGE
gateway-rate-limit   107s
```

Describe명령으로 `RateLimitPolicy`의 설정 내용을 확인합니다. 현재 랩의 경우 Gateway 레벨에 초당 10개의 요청(10 req/sec)로 설정이 되어 있습니다. 

```code
kubectl describe RateLimitPolicy gateway-rate-limit
```

아래와 같은 내용을 확인할 수 있으며, 설정에서 key는 "$binary_remote_addr"이기 때문에 기준은 요청한 사용자의 IP주소가 됩니다.

```code
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

두번째로 gRPC 애플리케이션에 대한 Rate Limit 테스트를 위해 gRPC Route도 함께 배포합니다.

```code
kubectl apply -f 2.routes.yaml
```

먼저 배포한 HTTP Route 설정을 확인합니다. 

```code
kubectl get httproute
```

결과는 아래와 같이 확인이 되어야 합니다.

```code
NAME     HOSTNAMES              AGE
coffee   ["cafe.example.com"]   82s
```

두번째 배포한 gRPC route 설정을 함께 확인합니다.

```code
kubectl get grpcroute
```

아래와 같은 결과를 확인할 수 있습니다.

```code
NAME         HOSTNAMES              AGE
grpc-route   ["grpc.example.com"]   113s
```

이제 `HTTPRoute`와 `gRPCRoute`에 Rate Limit을 적용하기 위해 `RateLimitPolicy` 정책을 연결합니다. 

```code
kubectl apply -f 3.route-ratelimit.yaml
```

지금까지 설정한 전체 rate limit 내용을 확인합니다.

```code
kubectl get ratelimitpolicy
```

결과는 아래와 같이 보입니다.

```code
NAME                 AGE
gateway-rate-limit   8m24s
route-rate-limit     81s
```

Gateway 레벨이 아닌 `HTTPRoute` 및 `gRPCRoute` 레벨에 적용한 `RateLimitPolicy`의 내용을 Describe하여 설정내용을 확인합니다. 

```code
kubectl describe ratelimitpolicy route-rate-limit
```

아래와 유사한 결과를 확인할 수 있어야 합니다. HTTPRoute 레벨 및 GRPCRoute 레벨로 적용된 Rate Limit은 초당 1개의 요청( 1 req/sec )로 설정된 것을 내용에서 확인할 수 있습니다.

```code
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

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP 주소와 HTTP PORT 정보를 확인 후 변수로 저장합니다.

```code
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc gateway-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

저장된 NGINX Gateway Fabric 데이터플레인 인스턴스의 IP와 HTTP PORT정보를 확인합니다.

```code
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

첫번째 테스트로 HTTP 애플리케이션으로 1번의 요청을 전송해서 정상 접속을 먼저 확인합니다. 

```code
curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

결과는 아래와 같이 정상적인 요청에 대한 응답이 제공됩니다.

```code
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

이번엔 2번의 요청을 동시에 전송합니다. 

```code
curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee;echo "---";curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

결과는 아래와 같이 적용된 1 req/sec rate limit이 적용되어 2번째 요청은 `429 Too Many Requests` 응답이 제공되는 것을 확인할 수 있습니다. 

```code
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

동일한 방식으로 gRPC 애플리케이션에 대한 요청을 1번만 전송합니다. 

```code
grpcurl -plaintext -proto grpc.proto -authority grpc.example.com -d '{"name": "exact"}' $NGF_IP:$HTTP_PORT helloworld.Greeter/SayHello
```

예상한 바와 같이 1번의 요청은 정상적으로 처리되어 200 응답이 제공됩니다. 

```code
{
  "message": "Hello exact"
}
```

이번엔 2번의 gRPC 요청을 한번에 전달합니다. 

```code
grpcurl -plaintext -proto grpc.proto -authority grpc.example.com -d '{"name": "exact"}' $NGF_IP:$HTTP_PORT helloworld.Greeter/SayHello;echo "---";grpcurl -plaintext -proto grpc.proto -authority grpc.example.com -d '{"name": "exact"}' $NGF_IP:$HTTP_PORT helloworld.Greeter/SayHello
```

아래와 같은 결과를 확인할 수 있습니다. 2번째 요청은 Gateway를 통해서 서버로 전달되지 않고 NGINX에서 제한했기 때문에 아래와 같이 unknown 에러가 발생하였습니다. 이 내용은 NGINX Gateway 로그를 통해서도 확인할 수 있습니다. 

```code
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

위 결과를 모두 확인하셨다면 다음 실습을 위해 이번 실습의 설정은 모두 삭제합니다.

```code
kubectl delete -f .
```

이번 NGINX Gateway Fabric 실습은 여기까지 마무리가 되었습니다. 수고하셨습니다^^
