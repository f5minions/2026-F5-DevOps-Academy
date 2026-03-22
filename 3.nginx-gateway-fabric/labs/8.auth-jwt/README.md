# Lab 8 — SnippetsFilter를 이용한 JWT 인증 적용

> `SnippetsFilter`를 활용하여 NGINX JWT 인증 모듈을 Gateway에 적용하고, 토큰 유무에 따른 인증 동작을 확인합니다.

---

## 실습 경로 이동

```bash
# 이전 Lab 디렉토리에 있다면
cd ../8.auth-jwt/

# Lab 기본경로(2026-F5-DevOps-Academy)에 있다면
cd 3.nginx-gateway-fabric/labs/8.auth-jwt
```

---

## Step 1 — 테스트 애플리케이션 배포

```bash
kubectl apply -f 0.coffee.yaml
```

배포 상태가 `Running`인지 확인합니다.

```bash
kubectl get all
```

```
NAME                          READY   STATUS    RESTARTS   AGE
pod/coffee-56b44d4c55-g9gtj   1/1     Running   0          3s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/coffee       ClusterIP   10.107.232.5   <none>        80/TCP    3s
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   385d

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coffee   1/1     1            1           3s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/coffee-56b44d4c55   1         1         1       3s
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

`gateway-nginx-56678b747f-rrx4d`가 NGINX Gateway Fabric 데이터플레인 Pod입니다. (실제 이름은 다를 수 있음)

```
NAME                             READY   STATUS    RESTARTS   AGE
coffee-56b44d4c55-g9gtj          1/1     Running   0          37s
gateway-nginx-56678b747f-rrx4d   1/1     Running   0          13s
```

### Gateway 상태 확인

```bash
kubectl get gateway
```

```
NAME      CLASS   ADDRESS       PROGRAMMED   AGE
gateway   nginx   10.97.70.64   True         98s
```

### Service 상태 확인

`gateway-nginx`가 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```bash
kubectl get service
```

```
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
coffee          ClusterIP   10.107.232.5   <none>        80/TCP         2m17s
gateway-nginx   NodePort    10.97.70.64    <none>        80:30496/TCP   114s
kubernetes      ClusterIP   10.96.0.1      <none>        443/TCP        385d
```

---

## Step 3 — SnippetsFilter 배포 (JWT 인증 설정)

JWT 인증 설정을 NGINX에 삽입하는 `SnippetsFilter`를 배포합니다.

```bash
kubectl apply -f 2.snippetsfilter-jwtauth.yaml
```

설정 내용을 확인합니다.

```bash
kubectl describe snippetsfilter auth-jwt
```

```
Name:         auth-jwt
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  gateway.nginx.org/v1alpha1
Kind:         SnippetsFilter
Metadata:
  Creation Timestamp:  2025-10-07T11:41:02Z
  Generation:          1
  Resource Version:    66576310
  UID:                 e7ff7827-3ea8-42e8-8166-e758ddd6bc40
Spec:
  Snippets:
    Context:  http.server
    Value:    location = /_auth/_jwks_uri { internal;return 200 '{"keys":[{"k":"ZmFudGFzdGljand0","kty":"oct","kid":"0001"}]}'; }
    Context:  http.server.location
    Value:    auth_jwt "JWT token required";auth_jwt_type signed;auth_jwt_key_request /_auth/_jwks_uri;
Status:
  Controllers:
    Conditions:
      Last Transition Time:  2025-10-07T11:41:02Z
      Message:               SnippetsFilter is accepted
      Observed Generation:   1
      Reason:                Accepted
      Status:                True
      Type:                  Accepted
    Controller Name:         gateway.nginx.org/nginx-gateway-controller
Events:                      <none>
```

위 설정이 NGINX Gateway Fabric 데이터플레인에 실제로 적용되는 내용은 다음과 같습니다.

```nginx
location = /_auth/_jwks_uri {   # = : 정확한 경로 매칭 (prefix 매칭보다 우선순위 높음)
    internal;                    # 외부 직접 접근 불가
    return 200 '{ JWKS JSON }';  # JWK Set 반환
}
```

```json
{
  "keys": [
    {
      "k":   "ZmFudGFzdGljand0",
      "kty": "oct",
      "kid": "0001"
    }
  ]
}
```

> `k` 값을 디코딩하면 실제 HMAC Secret을 확인할 수 있습니다.
>
> ```bash
> echo "ZmFudGFzdGljand0" | base64 -d
> # fantasticjwt
> ```

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
NAME     HOSTNAMES              AGE
coffee   ["cafe.example.com"]   4s
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

### 테스트 1 — 인증 토큰 없음 (401 Unauthorized)

```bash
curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT
```

JWT 토큰이 없으면 `401 Unauthorized`가 반환됩니다.

```
HTTP/1.1 401 Unauthorized
Server: nginx
Date: Tue, 07 Oct 2025 11:43:46 GMT
Content-Type: text/html
Content-Length: 172
Connection: keep-alive
WWW-Authenticate: Bearer realm="JWT token required"

<html>
<head><title>401 Authorization Required</title></head>
<body>
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

### 테스트 2 — 유효한 인증 토큰 포함 (200 OK)

```bash
curl -i --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT -H "Authorization: Bearer `cat token.jwt`"
```

유효한 JWT 토큰을 함께 전달하면 인증이 완료되고 애플리케이션 응답이 반환됩니다.

```
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 07 Oct 2025 11:45:36 GMT
Content-Type: text/plain
Content-Length: 156
Connection: keep-alive
Expires: Tue, 07 Oct 2025 11:45:35 GMT
Cache-Control: no-cache

Server address: 10.0.156.110:8080
Server name: coffee-56b44d4c55-g9gtj
Date: 07/Oct/2025:11:45:36 +0000
URI: /
Request ID: 5afec06d5432b68456477e806cf8a52e
```

---

## 실습 종료 — 리소스 삭제

```bash
kubectl delete -f .
```
