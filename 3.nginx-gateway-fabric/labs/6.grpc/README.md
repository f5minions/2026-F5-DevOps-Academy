# gRPC support

이번 활용사례는 NGINX Gateway Fabric을 이용해서 가장 인기있는 API 프로토콜인 gRPC기반 트래픽 관리 실습을 진행합니다.

Lab 진행을 위한 실습 경로로 이동합니다.

```code
이전 Lab 디렉토리에 있다면,
cd ../6.grpc/

Lab 기본경로(2026-F5-DevOps-Academy)에 있다면,
cd 3.nginx-gateway-fabric/labs/6.grpc
```

gRPC기반의 애플리케이션을 먼저 배포합니다.

```code
kubectl apply -f 0.helloworld.yaml
```

배포한 애플리케이션의 `running` 상태를 확인합니다.

```code
kubectl get all
```

아래와 유사한 결과를 확인할 수 있습니다.

```code
NAME                                        READY   STATUS    RESTARTS   AGE
pod/grpc-infra-backend-v1-bc4bc48dc-jkwfx   1/1     Running   0          26s
pod/grpc-infra-backend-v2-67fd996d5-qn4sp   1/1     Running   0          26s

NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/grpc-infra-backend-v1   ClusterIP   10.109.98.121   <none>        8080/TCP   27s
service/grpc-infra-backend-v2   ClusterIP   10.108.28.155   <none>        8080/TCP   26s
service/kubernetes              ClusterIP   10.96.0.1       <none>        443/TCP    268d

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grpc-infra-backend-v1   1/1     1            1           27s
deployment.apps/grpc-infra-backend-v2   1/1     1            1           26s

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/grpc-infra-backend-v1-bc4bc48dc   1         1         1       26s
replicaset.apps/grpc-infra-backend-v2-67fd996d5   1         1         1       26s
```

이제 Gateway를 배포하고 method 매치 기반의 gRPC route 설정을 합니다. 동일한 namespace에 NGINX Gateway Fabric 데이터플레인 pod가 배포됩니다.

```code
kubectl apply -f 1.grpcroute-exactmethod.yaml
```

NGINX Gateway Fabric 데이터플레인 pod의 상태를 확인합니다.

```
kubectl get pods
```

`same-namespace-nginx-8c55bff94-mxmpx`는 NGINX Gateway Fabric 데이터플레인 pod 입니다.

```
NAME                                    READY   STATUS    RESTARTS   AGE
grpc-infra-backend-v1-bc4bc48dc-jkwfx   1/1     Running   0          75s
grpc-infra-backend-v2-67fd996d5-qn4sp   1/1     Running   0          75s
same-namespace-nginx-8c55bff94-mxmpx    1/1     Running   0          9s
```

Gateway 설정을 확인합니다.

```code
kubectl get gateway
```

아래와 유사한 결과를 확인할 수 있습니다.

```code
NAME             CLASS   ADDRESS          PROGRAMMED   AGE
same-namespace   nginx   10.110.235.199   True         63s
```

NGINX Gateway Fabric 서비스 설정도 확인합니다.

```code
kubectl get service
```

`same-namespace-nginx`는 NGINX Gateway Fabric 서비스입니다.

```code
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
grpc-infra-backend-v1   ClusterIP   10.109.98.121    <none>        8080/TCP       2m21s
grpc-infra-backend-v2   ClusterIP   10.108.28.155    <none>        8080/TCP       2m20s
kubernetes              ClusterIP   10.96.0.1        <none>        443/TCP        268d
same-namespace-nginx    NodePort    10.110.235.199   <none>        80:32081/TCP   75s
```

그리고 Gateway API spec 중 설정한 gRPC routes의 설정을 확인합니다.

```code
kubectl get grpcroutes
```

아래와 유사한 결과를 확인할 수 있습니다.

```code
NAME             HOSTNAMES   AGE
exact-matching               2m41s
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP 주소와 HTTP PORT 정보를 확인 후 변수로 저장합니다. 

```code
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc same-namespace-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

저장된 NGINX Gateway Fabric 데이터플레인 인스턴스의 IP와 HTTP PORT정보를 확인합니다.

```code
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

이제 첫번쨰 애플리케이션을 테스트를 진행합니다. 

```code
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "exact"}' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

결과는 아래와 같습니다.

```code
{
  "message": "Hello exact"
}
```

method 매칭기반의 gRPC roue를 제거하고 다음 실습을 진행합니다.

```code
kubectl delete -f 1.grpcroute-exactmethod.yaml
```

hostname기반의 gRPC route를 설정합니다. 

```code
kubectl apply -f 2.grpcroute-hostname.yaml
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP와 HTTP PORT 정보를 추출하여 테스트를 위한 환경변수에 저장합니다.

```code
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc grpcroute-listener-hostname-matching-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP 및 HTTP PORT 정보를 확인합니다.

```code
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

`bar.com` 호스트명으로 애플리케이션 요청에 대한 테스트를 진행합니다.

```code
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "bar server"}' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

이 테스트의 요청은 `grpc-infra-backend-v1` pod로 트래픽을 라우팅합니다.

```code
kubectl logs -l app=grpc-infra-backend-v1
```

`grpc-infra-backend-v1` pod의 로그를 확인하면 다음과 같은 결과를 확인할 수 있습니다.

```code
2025/06/12 11:27:59 server listening at [::]:50051
2025/06/12 11:37:05 Received: exact
2025/06/12 11:39:48 Received: bar server
```

이번에는 `foo.bar.com` 호스트명으로 요청을 전송합니다.

```code
grpcurl -plaintext -proto grpc.proto -authority foo.bar.com -d '{"name": "bar server"}' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

이 요청은 `grpc-infra-backend-v2`로 라우팅이 됩니다.

```code
kubectl logs -l app=grpc-infra-backend-v2
```

결과는 아래와 같이 출력됩니다.

```code
2025/06/12 11:28:02 server listening at [::]:50051
2025/06/12 11:40:13 Received: bar server
```

다음 실습을 위해 hostname 기반의 gRPC route 설정을 제거합니다. 

```code
kubectl delete -f 2.grpcroute-hostname.yaml
```

이번 실습에서는 headers 기반의 gRPC route 설정을 배포합니다.

```code
kubectl apply -f 3.grpcroute-header.yaml
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP와 HTTP PORT 정보를 추출하여 테스트를 위한 환경변수에 저장합니다.

```code
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc same-namespace-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP 및 HTTP PORT 정보를 확인합니다.

```code
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

HTTP header에 `version: one`이라는 값을 포함하여 애플리케이션 요청을 전송합니다.

```code
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "version one"}' -H 'version: one' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

이 요청은 `grpc-infra-backend-v1` pod로 라우팅이 되었을 것입니다.

```code
kubectl logs -l app=grpc-infra-backend-v1
```

위 명령으로 해당 pod의 로그를 확인합니다.

```code
2025/06/12 11:27:59 server listening at [::]:50051
2025/06/12 11:37:05 Received: exact
2025/06/12 11:39:48 Received: bar server
2025/06/12 11:41:52 Received: version one
```

HTTP header에 `version: two` 값을 포함하여 동일하게 요청을 애플리케이션으로 전송합니다.

```code
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "version two"}' -H 'version: two' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

이 요청은 `grpc-infra-backend-v2` pod로 라우팅됩니다.

```code
kubectl logs -l app=grpc-infra-backend-v2
```

위 명령으로 해당 애플리케이션 pod의 로그를 보면 아래와 같은 결과를 확인할 수 있습니다.

```code
2025/06/12 11:28:02 server listening at [::]:50051
2025/06/12 11:40:13 Received: bar server
2025/06/12 11:42:12 Received: version two
```

이제는 HTTP header에 `regexHeader: grpc-header-a`라는 값을 포함하여 애플리케이션으로 요청을 전송합니다.

```code
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "grpc-header-a"}' -H 'grpcRegex: grpc-header-a' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

이 요청은 `grpc-infra-backend-2` pod로 라우팅됩니다.

```code
kubectl logs -l app=grpc-infra-backend-v2
```

테스트 결과는 아래와 같습니다.

```code
2025/09/18 22:24:44 server listening at [::]:50051
2025/09/18 22:28:45 Received: bar server
2025/09/18 22:29:53 Received: version two
2025/09/18 22:34:08 Received: grpc-header-a
```

이번엔 HTTP header `color: blue`를 삽입하고 애플리케이션에 요청을 전달합니다.

```code
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "blue 1"}' -H 'color: blue' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

이 요청은 `grpc-infra-backend-v1`으로 라우팅됩니다.

```code
kubectl logs -l app=grpc-infra-backend-v1
```

위 명령으로 해당 애플리케이션 pod의 로그를 확인합니다.

```code
2025/09/18 22:24:44 server listening at [::]:50051
2025/09/18 22:26:22 Received: exact
2025/09/18 22:27:34 Received: bar server
2025/09/18 22:30:09 Received: version one
2025/09/18 22:35:22 Received: blue 1
```

마지막으로 HTTP header에 `color: red`를 삽입하여 애플리케이션으로 요청을 전송합니다.

```code
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "red 2"}' -H 'color: red' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

이 요청은 `grpc-infra-backend-v2` pod로 라우팅됩니다.

```code
kubectl logs -l app=grpc-infra-backend-v2
```

테스트 결과는 아래와 유사하게 보입니다.

```code
2025/09/18 22:24:44 server listening at [::]:50051
2025/09/18 22:28:45 Received: bar server
2025/09/18 22:29:53 Received: version two
2025/09/18 22:34:08 Received: grpc-header-a
2025/09/18 22:36:03 Received: red 2
```

다음 실습을 위해 이번 Lab에서 진행했던 설정을 모두 제겅합니다.

```code
kubectl delete -f .
```
