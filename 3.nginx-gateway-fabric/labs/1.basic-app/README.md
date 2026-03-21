# 기본 URI 기반 라우팅

이번 실습에서는 URI 기반의 라우팅을 통해 2개의 애플리케이션을 배포하고 어떻게 동작하는가를 확인합니다.

Lab 실습을 위한 디렉토리로 이동합니다.

```code
cd 3.nginx-gateway-fabric/labs/1.basic-app
```

2개의 예제 애플리케이션을 먼저 배포합니다. 

```code
kubectl apply -f 0.cafe.yaml
```

아래 명령으로 배포한 애플리케이션이 정상 running 상태인지 확인할 수 있습니다. 아래 결과와 같이 running 상태가 아니라면 알려주세요!!

```code
kubectl get all
```

아래와 같은 결과를 확인할 수 있습니다.

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

`Gateway` 오브젝트를 배포합니다. NGINX Gateway Fabric의 데이터플레인을 현재 `namespace`에 배포합니다.

```code
kubectl apply -f 1.gateway.yaml
```

NGINX Gateway Fabric 데이터플레인 pod의 상태를 확인합니다.

```
kubectl get pods
```

 `gateway-nginx-c9bcdf4d4-4hl7c`는 NGINX Gateway Fabric 데이터플레인 pod 입니다.

```
NAME                            READY   STATUS    RESTARTS   AGE
coffee-56b44d4c55-6drv2         1/1     Running   0          47s
gateway-nginx-c9bcdf4d4-4hl7c   1/1     Running   0          24s
tea-596697966f-fwf2r            1/1     Running   0          47s
```

Gateway 배포 정보를 확인합니다.

```code
kubectl get gateway
```

아래와 같은 결과를 확인할 수 있습니다.

```code
NAME      CLASS   ADDRESS        PROGRAMMED   AGE
gateway   nginx   10.102.76.40   True         5s
```

그리고 NGINX Gateway Fabric 데이터플레인의 서비스 설정 정보도 함께 확인합니다.

```code
kubectl get service
```

`gateway-nginx`는 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```code
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
coffee          ClusterIP   10.107.171.2    <none>        80/TCP         2s
gateway-nginx   NodePort    10.100.81.10    <none>        80:32604/TCP   15s
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        268d
tea             ClusterIP   10.96.115.255   <none>        80/TCP         2s
```

사용자의 요청을 전달하기 위한 HTTP Route 설정을 배포합니다. 

```code
kubectl apply -f 2.httproute.yaml
```

HTTP routes 설정 정보를 확인합니다.

```code
kubectl get httproute
```

아래와 같은 결과를 확인할 수 있습니다.

```code
NAME     HOSTNAMES              AGE
coffee   ["cafe.example.com"]   8s
tea      ["cafe.example.com"]   8s
```

NGINX Gateway Fabric의 데이터플레인 인스턴스의 IP와 HTTP PORT정보를 가져와서 변수로 저장합니다.

```code
export NGF_IP=`kubectl get pod -l app.kubernetes.io/instance=ngf -o json|jq '.items[0].status.hostIP' -r`
export HTTP_PORT=`kubectl get svc gateway-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP와 HTTP PORT 정보를 확인합니다.

```code
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

이제 아래 명령을 사용하고 `coffee`라는 첫번째 애플리케이션으로 접속을 테스트 합니다. 

```code
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

결과는 아래와 같이 `coffee` 애플리케이션으로 정상적으로 접속이 됩니다.

```code
Server address: 192.168.36.115:8080
Server name: coffee-56b44d4c55-nm5rx
Date: 24/Mar/2025:21:08:19 +0000
URI: /coffee
Request ID: 5136f3dd98058fc9edcad13998902e79
```

두번째 `tea` 애플리케이션으로 접속을 테스트 합니다. 

```code
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/tea
```

결과는 아래와 같이 두번째 `tea` 애플리케이션으로 접속을 확인할 수 있습니다.

```code
Server address: 192.168.36.116:8080
Server name: tea-596697966f-lk2gp
Date: 24/Mar/2025:21:08:23 +0000
URI: /tea
Request ID: 09603099f3ad42da023a6184019ffbb6
```

이번 실습에서는 Gateway와 기본 URI 기반의 HTTP Route 설정을 통해서 간단하게 애플리케이션 접속을 실습했습니다. 위 테스트가 모두 완료되었다면 다음 실습을 위해 전체 설정을 삭제 합니다. 

```code
kubectl delete -f .
```
