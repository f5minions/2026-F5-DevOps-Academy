# Traffic splitting

이번 랩에서는 동일한 애플리케이션 서로 다른 버전으로의 트래픽 분할 활용사례에 대해서 실습을 진행합니다.

실습을 위해서 해당 시나리오의 디렉토리로 이동합니다.

```code
이전 Lab 디렉토리에 있다면,
cd ../3.nginx-gateway-fabric/labs/5.traffic-splitting

실습 메인 디렉토리(2026-F5-DevOps-Academy)에 있다면,
cd 3.nginx-gateway-fabric/labs/5.traffic-splitting
```

이번 실습에 사용할 애플리케이션은 2가지 버전으로 배포가 됩니다. 실제 내용은 0.cafe.yaml 파일의 코드를 클릭해서 직접 확인할 수 있습니다. 

```code
kubectl apply -f 0.cafe.yaml
```

배포한 애플리케이션의 전체 pods 및 services 정상적인 상태를 확인합니다.

```code
kubectl get all
```

아래와 유사한 결과를 확인할 수 있습니다.

```code
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

Gateway 오브젝트의 배포. NGINX Gateway Fabric 데이터플레인을 현재 namespace에 배포합니다.

```code
kubectl apply -f 1.gateway.yaml
```

NGINX Gateway Fabric 데이터플레인 pod의 상태를 확인합니다.

```
kubectl get pods
```

`gateway-nginx-c9bcdf4d4-j7bbg` pod가 NGINX Gateway Fabric 데이터플레인입니다.

```
NAME                            READY   STATUS    RESTARTS   AGE
coffee-v1-c48b96b65-gqkxr       1/1     Running   0          47s
coffee-v2-685fd9bb65-wl56z      1/1     Running   0          47s
gateway-nginx-c9bcdf4d4-j7bbg   1/1     Running   0          10s
```

Gateway 설정을 확인합니다.

```code
kubectl get gateway
```

아래와 유사한 결과를 확인할 수 있습니다.

```code
NAME      CLASS   ADDRESS          PROGRAMMED   AGE
gateway   nginx   10.105.225.176   True         31s
```

NGINX Gateway Fabric Service 상태를 확인합니다.

```code
kubectl get service
```

`gateway-nginx` 는 NGINX Gateway Fabric 데이터플레인 서비스입니다. 

```code
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
coffee-v1       ClusterIP   10.103.1.92      <none>        80/TCP         89s
coffee-v2       ClusterIP   10.109.14.146    <none>        80/TCP         89s
gateway-nginx   NodePort    10.105.225.176   <none>        80:31047/TCP   52s
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        268d
```

이제 HTTP route를 생성해서 2가지 버전의 애플리케이션으로 트래픽을 균등하게 분산합니다. 

```code
kubectl apply -f 2.route-80-80.yaml
```

HTTP routes의 배포 및 설정을 확인합니다.

```code
kubectl get httproute
```

아래와 유사한 결과를 확인할 수 있습니다.

```code
NAME         HOSTNAMES              AGE
cafe-route   ["cafe.example.com"]   17s
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP와 HTTP PORT 정보를 추출하여 테스트를 위한 환경변수에 저장합니다.

```code
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc gateway-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP 및 HTTP PORT 정보를 확인합니다.

```code
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

설정은 모두 완료되었고, 직접 애플리케이션으로 접속을 시도합니다.

```code
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

아래와 같은 결과가 출력됩니다는.

```code
Server address: 10.0.156.127:8080
Server name: coffee-v1-c48b96b65-gqkxr
Date: 12/Jun/2025:11:24:59 +0000
URI: /coffee
Request ID: e5992510df47c30e1e8a232263db5341
```

또는

```code
Server address: 10.0.156.67:8080
Server name: coffee-v2-685fd9bb65-wl56z
Date: 12/Jun/2025:11:24:43 +0000
URI: /coffee
Request ID: 995d20405a70bd5d5468a696e4b95e54
```

동일한 요청을 100회 전송하는 스크립트를 통해서 결과를 확인합니다.

```code
. ./test.sh
```

아래와 유사한 결과를 확인할 수 있습니다. 실제 처리되는 상황에 따라 약간의 오차는 있을 수 있지만 거의 50:50 분산 결과를 확인할 수 있습니다.

```code
....................................................................................................
Summary of responses:
Coffee v1: 50 times
Coffee v2: 50 times
```

이제 HTTP Route의 분산 비율을 80:20으로 업데이트해서 추가 시험을 진행합니다. 

```code
kubectl apply -f 3.route-80-20.yaml
```

동일하게 아래 스크립트를 통해서 100개의 요청을 전송하여 결과를 확인합니다.

```code
. ./test.sh
```

아래와 유사한 결과를 확인할 수 있습니다. 동일하게 상황에 따른 약간의 오차는 있을 수 있지만 거의 80:20 분산 결과를 확인할 수 있습니다.

```code
....................................................................................................
Summary of responses:
Coffee v1: 82 times
Coffee v2: 18 times
```

Delete the lab

```code
kubectl delete -f .
```
