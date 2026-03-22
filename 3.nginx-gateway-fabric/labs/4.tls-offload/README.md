# Lab 4 — TLS Offload

> TLS 인증서를 Gateway에서 처리하는 TLS Offload 설정과 HTTP → HTTPS 리다이렉트를 구성하고 테스트합니다.
>
> 참고: [Gateway API TLS Spec](https://gateway-api.sigs.k8s.io/guides/tls/) · [Redirect Spec](https://gateway-api.sigs.k8s.io/guides/http-redirect-rewrite/)

---

## 실습 경로 이동

```bash
cd 3.nginx-gateway-fabric/labs/4.tls-offload
```

---

## Step 1 — TLS 인증서 및 ReferenceGrant 생성

TLS 인증서와 키를 생성하고, Gateway에서 해당 인증서를 참조할 수 있도록 `ReferenceGrant`를 함께 설정합니다.

> `ReferenceGrant`를 설정하지 않으면 다른 namespace의 Gateway에서 인증서를 참조할 수 없습니다.

```yaml
# ReferenceGrant 설정 예시
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: access-to-cafe-secret
  namespace: certificate
spec:
  to:
  - group: ""
    kind: Secret
    name: cafe-secret # 생략 시 default namespace의 모든 Gateway가 접근 가능
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
    namespace: default
```

```bash
kubectl apply -f 0.certificate.yaml
```

---

## Step 2 — 테스트 애플리케이션 배포

```bash
kubectl apply -f 1.coffee.yaml
```

배포 상태가 `Running`인지 확인합니다.

```bash
kubectl get all
```

```
NAME                          READY   STATUS    RESTARTS   AGE
pod/coffee-56b44d4c55-jdst2   1/1     Running   0          3s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/coffee       ClusterIP   10.101.48.47   <none>        80/TCP    3s
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   268d

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coffee   1/1     1            1           3s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/coffee-56b44d4c55   1         1         1       3s
```

---

## Step 3 — Gateway 배포

NGINX Gateway Fabric 데이터플레인을 현재 `namespace`에 배포합니다.

```bash
kubectl apply -f 2.gateway.yaml
```

### Pod 상태 확인

```bash
kubectl get pods
```

`cafe-nginx-758ff7574c-kpbqx`는 NGINX Gateway Fabric 데이터플레인 Pod입니다.

```
NAME                          READY   STATUS    RESTARTS   AGE
cafe-nginx-758ff7574c-kpbqx   1/1     Running   0          24s
coffee-56b44d4c55-jdst2       1/1     Running   0          57s
```

### Gateway 상태 확인

```bash
kubectl get gateway
```

```
NAME   CLASS   ADDRESS        PROGRAMMED   AGE
cafe   nginx   10.43.28.152   True         84s
```

### Service 상태 확인

`cafe-nginx`는 NGINX Gateway Fabric 데이터플레인의 서비스입니다. HTTP(80)와 HTTPS(443) 두 포트가 모두 열려 있습니다.

```bash
kubectl get service
```

```
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
cafe-nginx   NodePort    10.110.127.86   <none>        80:32417/TCP,443:32657/TCP   75s
coffee       ClusterIP   10.101.48.47    <none>        80/TCP                       108s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP                      268d
```

---

## Step 4 — HTTPRoute 배포

HTTP → HTTPS 리다이렉트와 TLS 종단 처리를 위한 HTTPRoute를 배포합니다.

```bash
kubectl apply -f 3.httproute.yaml
```

```bash
kubectl get httproute
```

```
NAME                HOSTNAMES              AGE
cafe-tls-redirect   ["cafe.example.com"]   4s
coffee              ["cafe.example.com"]   4s
```

---

## Step 5 — 테스트

### 환경 변수 설정

```bash
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc cafe-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
export HTTPS_PORT=`kubectl get svc cafe-nginx -o jsonpath='{.spec.ports[1].nodePort}'`
```

```bash
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT\nHTTPS port : $HTTPS_PORT"
```

---

### 테스트 1 — HTTP 접속 (302 리다이렉트)

HTTP로 접속하면 HTTPS로 리다이렉트되는 것을 확인합니다.

```bash
curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

```
HTTP/1.1 302 Moved Temporarily
Server: nginx
Date: Thu, 12 Jun 2025 11:19:04 GMT
Content-Type: text/html
Content-Length: 138
Connection: keep-alive
Location: https://cafe.example.com/coffee

<html>
<head><title>302 Found</title></head>
<body>
<center><h1>302 Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

### 테스트 2 — HTTPS 접속 (TLS Offload)

HTTPS로 접속하면 Gateway에서 TLS를 처리하고 애플리케이션으로 전달합니다.

```bash
curl -k --resolve cafe.example.com:$HTTPS_PORT:$NGF_IP https://cafe.example.com:$HTTPS_PORT/coffee
```

```
Server address: 10.0.156.120:8080
Server name: coffee-56b44d4c55-jdst2
Date: 12/Jun/2025:11:19:24 +0000
URI: /coffee
Request ID: 6cb931a24c1c1bbff763d5ba7481a2f3
```

---

## 실습 종료 — 리소스 삭제

```bash
kubectl delete -f .
```
