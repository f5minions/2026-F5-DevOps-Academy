# Lab 1 — 기본 URI 기반 라우팅

> URI 경로 기반의 라우팅으로 2개의 애플리케이션을 배포하고, NGINX Gateway Fabric을 통해 트래픽이 어떻게 전달되는지 확인합니다.

---

## 실습 경로 이동

```bash
cd 3.nginx-gateway-fabric/labs/1.basic-app
```

---

## Step 1 — 테스트 애플리케이션 배포

```bash
kubectl apply -f 0.cafe.yaml
```

배포 상태가 `Running`인지 확인합니다.

```bash
kubectl get all
```

```
NAME                          READY   STATUS    RESTARTS   AGE
pod/coffee-56b44d4c55-nm5rx   1/1     Running   0          8m39s
pod/tea-596697966f-lk2gp      1/1     Running   0          8m39s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/coffee       ClusterIP   10.102.183.198   <none>        80/TCP    8m39s
service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP   38d
service/tea          ClusterIP   10.111.232.2     <none>        80/TCP    8m39s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coffee   1/1     1            1           8m39s
deployment.apps/tea      1/1     1            1           8m39s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/coffee-56b44d4c55   1         1         1       8m39s
replicaset.apps/tea-596697966f      1         1         1       8m39s
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

`gateway-nginx-c9bcdf4d4-4hl7c`는 NGINX Gateway Fabric 데이터플레인 Pod입니다.

```
NAME                            READY   STATUS    RESTARTS   AGE
coffee-56b44d4c55-6drv2         1/1     Running   0          47s
gateway-nginx-c9bcdf4d4-4hl7c   1/1     Running   0          24s
tea-596697966f-fwf2r            1/1     Running   0          47s
```

### Gateway 상태 확인

```bash
kubectl get gateway
```

```
NAME      CLASS   ADDRESS        PROGRAMMED   AGE
gateway   nginx   10.102.76.40   True         5s
```

### Service 상태 확인

`gateway-nginx`는 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```bash
kubectl get service
```

```
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
coffee          ClusterIP   10.107.171.2    <none>        80/TCP         2s
gateway-nginx   NodePort    10.100.81.10    <none>        80:32604/TCP   15s
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        268d
tea             ClusterIP   10.96.115.255   <none>        80/TCP         2s
```

---

## Step 3 — HTTPRoute 배포

사용자 요청을 각 애플리케이션으로 전달하기 위한 HTTP Route를 배포합니다.

```bash
kubectl apply -f 2.httproute.yaml
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
export HTTP_PORT=`kubectl get svc gateway-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

```bash
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

### coffee 애플리케이션 접속

```bash
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

```
Server address: 192.168.36.115:8080
Server name: coffee-56b44d4c55-nm5rx
Date: 24/Mar/2025:21:08:19 +0000
URI: /coffee
Request ID: 5136f3dd98058fc9edcad13998902e79
```

### tea 애플리케이션 접속

```bash
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/tea
```

```
Server address: 192.168.36.116:8080
Server name: tea-596697966f-lk2gp
Date: 24/Mar/2025:21:08:23 +0000
URI: /tea
Request ID: 09603099f3ad42da023a6184019ffbb6
```

---

## 실습 종료 — 리소스 삭제

```bash
kubectl delete -f .
```
