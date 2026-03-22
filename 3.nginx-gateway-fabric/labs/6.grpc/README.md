# Lab 6 — gRPC 트래픽 관리

> NGINX Gateway Fabric을 이용하여 gRPC 트래픽을 Method, Hostname, Header 기반으로 라우팅하는 방법을 실습합니다.

---

## 실습 경로 이동

```bash
# 이전 Lab 디렉토리에 있다면
cd ../6.grpc/

# Lab 기본경로(2026-F5-DevOps-Academy)에 있다면
cd 3.nginx-gateway-fabric/labs/6.grpc
```

---

## Step 1 — 테스트 애플리케이션 배포

```bash
kubectl apply -f 0.helloworld.yaml
```

배포 상태가 `Running`인지 확인합니다.

```bash
kubectl get all
```

```
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

---

## Step 2 — Method 매칭 기반 gRPC Route

Gateway와 Method 매칭 기반의 gRPCRoute를 함께 배포합니다.

```bash
kubectl apply -f 1.grpcroute-exactmethod.yaml
```

### Pod / Gateway / Service 상태 확인

```bash
kubectl get pods
```

`same-namespace-nginx-8c55bff94-mxmpx`는 NGINX Gateway Fabric 데이터플레인 Pod입니다.

```
NAME                                    READY   STATUS    RESTARTS   AGE
grpc-infra-backend-v1-bc4bc48dc-jkwfx   1/1     Running   0          75s
grpc-infra-backend-v2-67fd996d5-qn4sp   1/1     Running   0          75s
same-namespace-nginx-8c55bff94-mxmpx    1/1     Running   0          9s
```

```bash
kubectl get gateway
```

```
NAME             CLASS   ADDRESS          PROGRAMMED   AGE
same-namespace   nginx   10.110.235.199   True         63s
```

`same-namespace-nginx`는 NGINX Gateway Fabric의 서비스입니다.

```bash
kubectl get service
```

```
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
grpc-infra-backend-v1   ClusterIP   10.109.98.121    <none>        8080/TCP       2m21s
grpc-infra-backend-v2   ClusterIP   10.108.28.155    <none>        8080/TCP       2m20s
kubernetes              ClusterIP   10.96.0.1        <none>        443/TCP        268d
same-namespace-nginx    NodePort    10.110.235.199   <none>        80:32081/TCP   75s
```

```bash
kubectl get grpcroutes
```

```
NAME             HOSTNAMES   AGE
exact-matching               2m41s
```

### 환경 변수 설정

```bash
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc same-namespace-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

```bash
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

### 테스트 — Exact Method 매칭

```bash
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "exact"}' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

```json
{
  "message": "Hello exact"
}
```

### 다음 시나리오 준비

```bash
kubectl delete -f 1.grpcroute-exactmethod.yaml
```

---

## Step 3 — Hostname 매칭 기반 gRPC Route

```bash
kubectl apply -f 2.grpcroute-hostname.yaml
```

### 환경 변수 업데이트

```bash
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc grpcroute-listener-hostname-matching-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

```bash
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

### 테스트 1 — `bar.com` → grpc-infra-backend-v1

```bash
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "bar server"}' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

`grpc-infra-backend-v1` Pod 로그로 라우팅 확인:

```bash
kubectl logs -l app=grpc-infra-backend-v1
```

```
2025/06/12 11:27:59 server listening at [::]:50051
2025/06/12 11:37:05 Received: exact
2025/06/12 11:39:48 Received: bar server
```

### 테스트 2 — `foo.bar.com` → grpc-infra-backend-v2

```bash
grpcurl -plaintext -proto grpc.proto -authority foo.bar.com -d '{"name": "bar server"}' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

`grpc-infra-backend-v2` Pod 로그로 라우팅 확인:

```bash
kubectl logs -l app=grpc-infra-backend-v2
```

```
2025/06/12 11:28:02 server listening at [::]:50051
2025/06/12 11:40:13 Received: bar server
```

### 다음 시나리오 준비

```bash
kubectl delete -f 2.grpcroute-hostname.yaml
```

---

## Step 4 — Header 매칭 기반 gRPC Route

```bash
kubectl apply -f 3.grpcroute-header.yaml
```

### 환경 변수 업데이트

```bash
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc same-namespace-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

```bash
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

### 테스트 1 — `version: one` → grpc-infra-backend-v1

```bash
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "version one"}' -H 'version: one' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

```bash
kubectl logs -l app=grpc-infra-backend-v1
```

```
2025/06/12 11:27:59 server listening at [::]:50051
2025/06/12 11:37:05 Received: exact
2025/06/12 11:39:48 Received: bar server
2025/06/12 11:41:52 Received: version one
```

### 테스트 2 — `version: two` → grpc-infra-backend-v2

```bash
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "version two"}' -H 'version: two' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

```bash
kubectl logs -l app=grpc-infra-backend-v2
```

```
2025/06/12 11:28:02 server listening at [::]:50051
2025/06/12 11:40:13 Received: bar server
2025/06/12 11:42:12 Received: version two
```

### 테스트 3 — `grpcRegex: grpc-header-a` → grpc-infra-backend-v2 (Regex 매칭)

```bash
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "grpc-header-a"}' -H 'grpcRegex: grpc-header-a' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

```bash
kubectl logs -l app=grpc-infra-backend-v2
```

```
2025/09/18 22:24:44 server listening at [::]:50051
2025/09/18 22:28:45 Received: bar server
2025/09/18 22:29:53 Received: version two
2025/09/18 22:34:08 Received: grpc-header-a
```

### 테스트 4 — `color: blue` → grpc-infra-backend-v1

```bash
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "blue 1"}' -H 'color: blue' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

```bash
kubectl logs -l app=grpc-infra-backend-v1
```

```
2025/09/18 22:24:44 server listening at [::]:50051
2025/09/18 22:26:22 Received: exact
2025/09/18 22:27:34 Received: bar server
2025/09/18 22:30:09 Received: version one
2025/09/18 22:35:22 Received: blue 1
```

### 테스트 5 — `color: red` → grpc-infra-backend-v2

```bash
grpcurl -plaintext -proto grpc.proto -authority bar.com -d '{"name": "red 2"}' -H 'color: red' ${NGF_IP}:${HTTP_PORT} helloworld.Greeter/SayHello
```

```bash
kubectl logs -l app=grpc-infra-backend-v2
```

```
2025/09/18 22:24:44 server listening at [::]:50051
2025/09/18 22:28:45 Received: bar server
2025/09/18 22:29:53 Received: version two
2025/09/18 22:34:08 Received: grpc-header-a
2025/09/18 22:36:03 Received: red 2
```

---

## 실습 종료 — 리소스 삭제

```bash
kubectl delete -f .
```
