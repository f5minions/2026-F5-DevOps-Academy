# Lab 5 — Traffic Splitting

> 동일한 애플리케이션의 서로 다른 버전으로 트래픽을 비율에 따라 분산하는 방법을 실습합니다.

---

## 실습 경로 이동

```bash
# 이전 Lab 디렉토리에 있다면
cd ../3.nginx-gateway-fabric/labs/5.traffic-splitting

# 실습 메인 디렉토리(2026-F5-DevOps-Academy)에 있다면
cd 3.nginx-gateway-fabric/labs/5.traffic-splitting
```

---

## Step 1 — 테스트 애플리케이션 배포

`coffee-v1`과 `coffee-v2` 두 버전의 애플리케이션을 배포합니다. 배포 전 `0.cafe.yaml`의 내용을 확인해보시기 바랍니다.

```bash
kubectl apply -f 0.cafe.yaml
```

배포 상태가 `Running`인지 확인합니다.

```bash
kubectl get all
```

```
NAME                               READY   STATUS    RESTARTS   AGE
pod/coffee-v1-c48b96b65-gqkxr    1/1     Running   0          4s
pod/coffee-v2-685fd9bb65-wl56z   1/1     Running   0          4s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/coffee-v1    ClusterIP   10.103.1.92     <none>        80/TCP    4s
service/coffee-v2    ClusterIP   10.109.14.146   <none>        80/TCP    4s
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   268d

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coffee-v1   1/1     1            1           4s
deployment.apps/coffee-v2   1/1     1            1           4s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/coffee-v1-c48b96b65    1         1         1       4s
replicaset.apps/coffee-v2-685fd9bb65   1         1         1       4s
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

`gateway-nginx-c9bcdf4d4-j7bbg`가 NGINX Gateway Fabric 데이터플레인 Pod입니다.

```
NAME                            READY   STATUS    RESTARTS   AGE
coffee-v1-c48b96b65-gqkxr       1/1     Running   0          47s
coffee-v2-685fd9bb65-wl56z      1/1     Running   0          47s
gateway-nginx-c9bcdf4d4-j7bbg   1/1     Running   0          10s
```

### Gateway 상태 확인

```bash
kubectl get gateway
```

```
NAME      CLASS   ADDRESS          PROGRAMMED   AGE
gateway   nginx   10.105.225.176   True         31s
```

### Service 상태 확인

`gateway-nginx`는 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```bash
kubectl get service
```

```
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
coffee-v1       ClusterIP   10.103.1.92      <none>        80/TCP         89s
coffee-v2       ClusterIP   10.109.14.146    <none>        80/TCP         89s
gateway-nginx   NodePort    10.105.225.176   <none>        80:31047/TCP   52s
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        268d
```

---

## Step 3 — 환경 변수 설정

```bash
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc gateway-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

```bash
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

---

## Step 4 — 트래픽 분산 테스트: 50:50

`coffee-v1`과 `coffee-v2`로 트래픽을 균등하게 분산하는 HTTPRoute를 배포합니다.

```bash
kubectl apply -f 2.route-80-80.yaml
```

```bash
kubectl get httproute
```

```
NAME         HOSTNAMES              AGE
cafe-route   ["cafe.example.com"]   17s
```

단일 요청을 전송하면 `coffee-v1` 또는 `coffee-v2` 중 하나로 응답이 옵니다.

```bash
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

```
Server address: 10.0.156.127:8080
Server name: coffee-v1-c48b96b65-gqkxr
...
```

또는

```
Server address: 10.0.156.67:8080
Server name: coffee-v2-685fd9bb65-wl56z
...
```

100회 연속 요청으로 분산 비율을 확인합니다.

```bash
. ./test.sh
```

약 **50:50** 비율로 분산되는 것을 확인할 수 있습니다.

```
....................................................................................................
Summary of responses:
Coffee v1: 50 times
Coffee v2: 50 times
```

---

## Step 5 — 트래픽 분산 테스트: 80:20

HTTPRoute를 업데이트하여 분산 비율을 **80:20**으로 변경합니다.

```bash
kubectl apply -f 3.route-80-20.yaml
```

```bash
. ./test.sh
```

약 **80:20** 비율로 분산되는 것을 확인할 수 있습니다.

```
....................................................................................................
Summary of responses:
Coffee v1: 82 times
Coffee v2: 18 times
```

---

## 실습 종료 — 리소스 삭제

```bash
kubectl delete -f .
```
