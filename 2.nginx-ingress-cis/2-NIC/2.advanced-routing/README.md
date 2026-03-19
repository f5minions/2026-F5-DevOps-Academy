# Ingress Controller Lab #2 - Advanced routing 

## Lab 개요 
* `tea-post-svc`, `tea-svc`, `coffee-v1-svc`, `coffee-v2-svc` 4가지 서비스를 바탕으로 HTTP Cookie / HTTP Method 기반으로의 Advanced L7 Routing 기능 실습 

* 랩의 결과 
    - `/tea`에 대한 POST 요청은 `tea-post-svc`로 라우팅됩니다.
    - `tea`에 대한 POST 이외의 요청은 `tea-svc`로 라우팅됩니다.
    - `version` 쿠키가 `v2`로 설정된 `/coffee`에 대한 요청은 `coffee-v2-svc`로 라우팅됩니다.
    - `version` 쿠키가 없는 `/coffee`에 대한 요청은 `coffee-v1-svc`로 라우팅됩니다.

## 샘플 서비스 배포

* 샘플 서비스 코드 배포 
```code
kubectl apply -f 0.cafe.yaml
kubectl apply -f 1.cafe-secret.yaml
```

* 배포 예상 결과 
```
NAME                             READY   STATUS    RESTARTS   AGE
pod/coffee-v1-c48b96b65-pkvlw    1/1     Running   0          33s
pod/coffee-v2-685fd9bb65-m6zgv   1/1     Running   0          33s
pod/tea-596697966f-26swq         1/1     Running   0          33s
pod/tea-post-5647b8d885-9zq6f    1/1     Running   0          33s

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/coffee-v1-svc   ClusterIP   172.20.122.65    <none>        80/TCP    33s
service/coffee-v2-svc   ClusterIP   172.20.195.88    <none>        80/TCP    33s
service/kubernetes      ClusterIP   172.20.0.1       <none>        443/TCP   22h
service/tea-post-svc    ClusterIP   172.20.194.126   <none>        80/TCP    33s
service/tea-svc         ClusterIP   172.20.188.11    <none>        80/TCP    33s

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coffee-v1   1/1     1            1           33s
deployment.apps/coffee-v2   1/1     1            1           33s
deployment.apps/tea         1/1     1            1           33s
deployment.apps/tea-post    1/1     1            1           33s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/coffee-v1-c48b96b65    1         1         1       33s
replicaset.apps/coffee-v2-685fd9bb65   1         1         1       33s
replicaset.apps/tea-596697966f         1         1         1       33s
replicaset.apps/tea-post-5647b8d885    1         1         1       33s
```

## 샘플 Virtual Server 리소스 배포 

```code
kubectl apply -f 2.advanced-routing.yaml
```

## 테스트 접속 수행 

```BASIC
curl -k https://cafe.vs.example.com/tea
# TEA Service 에서 응답 

curl -k https://cafe.vs.example.com/tea -X POST
# TES-POST Service 에서 응답 

curl -k https://cafe.vs.example.com/coffee --cookie "version=v2"
# Coffee-v2 Service 에서 응답 

curl -k https://cafe.vs.example.com/coffee
# Coffee Service 에서 응답 
```