# HTTP의 다양한 매칭 컨디션 기반의 고급라우팅

이번 실습에서는 HTTP의 다양한 매칭 컨디션을 활용하여 고급 라우팅 설정에 대한 테스트를 진행합니다. 다양한 방식으로 HTTP 정보를 매칭시키고 원하는 애플리케이션 서비스로 라우팅을 수행할 수 있습니다. 

Lab 실습을 위해 2번째 Lab 경로로 이동합니다. 

```code
이전 Lab 디렉토리에 있다면,
cd ../3.nginx-gateway-fabric/labs/2.advanced-routing

실습 메인 디렉토리(2026-F5-DevOps-Academy)에 있다면,
cd 3.nginx-gateway-fabric/labs/2.advanced-routing
```

2개의 예제 애플리케이션을 먼저 배포합니다.

```code
kubectl apply -f 0.coffee.yaml
kubectl apply -f 1.tea.yaml
```

배포 후 2개의 애플리케이션의 `running` 상태를 확인합니다. 

```code
kubectl get all
```

아래와 같은 결과를 확인할 수 있습니다.

```
NAME                              READY   STATUS    RESTARTS   AGE
pod/cafe-nginx-7444846d75-cgmms   1/1     Running   0          91s
pod/coffee-v1-c48b96b65-5trnr     1/1     Running   0          91s
pod/coffee-v2-685fd9bb65-dz5pp    1/1     Running   0          91s
pod/coffee-v3-7fb98466f-478hw     1/1     Running   0          91s
pod/tea-596697966f-hzjw5          1/1     Running   0          91s
pod/tea-post-5647b8d885-5xxvf     1/1     Running   0          91s

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/cafe-nginx      NodePort    10.103.90.239    <none>        80:31436/TCP   91s
service/coffee-v1-svc   ClusterIP   10.107.70.64     <none>        80/TCP         91s
service/coffee-v2-svc   ClusterIP   10.102.153.99    <none>        80/TCP         91s
service/coffee-v3-svc   ClusterIP   10.110.117.58    <none>        80/TCP         91s
service/kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        268d
service/tea-post-svc    ClusterIP   10.105.108.172   <none>        80/TCP         91s
service/tea-svc         ClusterIP   10.102.222.60    <none>        80/TCP         91s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cafe-nginx   1/1     1            1           91s
deployment.apps/coffee-v1    1/1     1            1           91s
deployment.apps/coffee-v2    1/1     1            1           91s
deployment.apps/coffee-v3    1/1     1            1           91s
deployment.apps/tea          1/1     1            1           91s
deployment.apps/tea-post     1/1     1            1           91s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/cafe-nginx-7444846d75   1         1         1       91s
replicaset.apps/coffee-v1-c48b96b65     1         1         1       91s
replicaset.apps/coffee-v2-685fd9bb65    1         1         1       91s
replicaset.apps/coffee-v3-7fb98466f     1         1         1       91s
replicaset.apps/tea-596697966f          1         1         1       91s
replicaset.apps/tea-post-5647b8d885     1         1         1       91s
```

`Gateway` 오브젝트를 배포합니다. NGINX Gateway Fabric의 데이터플레인을 현재 `namespace`에 배포합니다.

```code
kubectl apply -f 2.gateway.yaml
```

NGINX Gateway Fabric 데이터플레인 pod의 상태를 확인합니다.

```
kubectl get pods
```

`cafe-nginx-7444846d75-cgmms`는 NGINX Gateway Fabric 데이터플레인 pod 입니다.

```
NAME                          READY   STATUS    RESTARTS   AGE
cafe-nginx-7444846d75-cgmms   1/1     Running   0          113s
coffee-v1-c48b96b65-5trnr     1/1     Running   0          113s
coffee-v2-685fd9bb65-dz5pp    1/1     Running   0          113s
coffee-v3-7fb98466f-478hw     1/1     Running   0          113s
tea-596697966f-hzjw5          1/1     Running   0          113s
tea-post-5647b8d885-5xxvf     1/1     Running   0          113s
```

Gateway 배포 정보를 확인합니다.

```code
kubectl get gateway
```

아래와 같은 결과를 확인할 수 있습니다.

```code
NAME   CLASS   ADDRESS         PROGRAMMED   AGE
cafe   nginx   10.103.90.239   True         2m43s
```

그리고 NGINX Gateway Fabric 데이터플레인의 서비스 설정 정보도 함께 확인합니다.

```code
kubectl get service
```

`cafe-nginx`는 NGINX Gateway Fabric 데이터플레인의 서비스입니다.

```code
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
cafe-nginx      NodePort    10.103.90.239    <none>        80:31436/TCP   28s
coffee-v1-svc   ClusterIP   10.107.70.64     <none>        80/TCP         28s
coffee-v2-svc   ClusterIP   10.102.153.99    <none>        80/TCP         28s
coffee-v3-svc   ClusterIP   10.110.117.58    <none>        80/TCP         28s
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        268d
tea-post-svc    ClusterIP   10.105.108.172   <none>        80/TCP         28s
tea-svc         ClusterIP   10.102.222.60    <none>        80/TCP         28s
```

고급 라우팅을 위한 HTTP routes 설정을 배포합니다. 배포 전 자세한 라우팅 설정은 3.cafe-routes.yaml 파일을 내용을 한번 살펴보시기 바랍니다.

```code
kubectl apply -f 3.cafe-routes.yaml
```

배포한 HTTP routes 설정을 확인합니다.

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
export HTTP_PORT=`kubectl get svc cafe-nginx -o jsonpath='{.spec.ports[0].nodePort}'`
```

NGINX Gateway Fabric 데이터플레인 인스턴스의 IP와 HTTP PORT 정보를 확인합니다.

```code
echo -e "NGF address: $NGF_IP\nHTTP port  : $HTTP_PORT"
```

`coffee-v1` 애플리케이션으로 접속을 테스트합니다.

```code
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee
```

아래와 같은 결과를 확인할 수 있습니다.

```code
Server address: 10.0.156.109:8080
Server name: coffee-v1-c48b96b65-5trnr
Date: 12/Jun/2025:11:00:28 +0000
URI: /coffee
Request ID: 1a0b8a08ec4f94f6a5f02e7649165b18
```

이제 `coffee-v2` 애플리케이션으로 접속을 하기 위해 URI의 쿼리스트링에 버전을 명시하여 접속을 시도합니다. 

```code
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee?TEST=v2
```

결과는 Gateway는 HTTP Route에 설정된 매칭 컨디션을 참조하여 `coffee-v2 `애플리케이션으로 요청을 라우팅합니다. 

```code
Server address: 10.0.156.121:8080
Server name: coffee-v2-685fd9bb65-dz5pp
Date: 12/Jun/2025:11:00:44 +0000
URI: /coffee?TEST=v2
Request ID: eac31b251dfdf398033d8e8373df14c9
```

이번에 HTTP header에 동일하게 `coffee-v2`를 삽입하여 요청을 시도합니다.

```code
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee -H "version: v2"
```

결과는 아래와 같이 HTTP header 정보를 매칭하여 `coffee-v2` 애플리케이션을 요청을 라우팅합니다.

```code
Server address: 10.0.156.121:8080
Server name: coffee-v2-685fd9bb65-dz5pp
Date: 12/Jun/2025:11:01:00 +0000
URI: /coffee
Request ID: 0f5cd7a2f62965279c4bdc52c660c97d
```

추가로 HTTP 쿼리 스트링에 `coffee-v3`을 삽입하여 요청을 시도합니다.

```code
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee?queryRegex=query-a
```

결과는 아래와 같이 `coffee-v3` 애플리케이션으로 요청을 라우팅 합니다.

```code
Server address: 192.168.169.141:8080
Server name: coffee-v3-7fb98466f-tgq8k
Date: 18/Sep/2025:21:26:26 +0000
URI: /coffee?queryRegex=query-a
Request ID: 2c755a8391ebd2df87f416510fb5478b
```

동일하게 HTTP header에 `coffee-v3` 값을 삽입하여 요청을 시도합니다.

```code
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/coffee -H "headerRegex: header-a"
```

아래와 같이 `coffee-v3` 애플리케이션으로 요청을 라우팅한 것을 확인할 수 있습니다.

```code
Server address: 192.168.169.141:8080
Server name: coffee-v3-7fb98466f-tgq8k
Date: 18/Sep/2025:21:25:13 +0000
URI: /coffee
Request ID: 81431954b4b52edc01d707b4e5822792
```

이번에 다른 애플리케이션인 `tea`에 HTTP `GET` 요청을 시도합니다. 

```code
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/tea
```

아래와 같은 `tea` 애플리케이션으로 정상 라우팅 후 응답을 확인할 수 있습니다.

```code
Server address: 10.0.156.108:8080
Server name: tea-596697966f-hzjw5
Date: 12/Jun/2025:11:01:13 +0000
URI: /tea
Request ID: 7a45019ce6b1380b5d5402be89103703
```

동일한 `tea` 애플리케이션으로 HTTP `POST` 메소드로 요청을 시도합니다. 

```code
curl --resolve cafe.example.com:$HTTP_PORT:$NGF_IP http://cafe.example.com:$HTTP_PORT/tea -X POST
```

아래와 같이 HTTP `POST` 메소드로 요청을 할 경우 `tea` `POST` 애플리케이션으로 처리된 응답을 확인할 수 있습니다. 

```code
Server address: 10.0.156.122:8080
Server name: tea-post-5647b8d885-5xxvf
Date: 12/Jun/2025:11:01:32 +0000
URI: /tea
Request ID: 92c6bb8c35b24c1ca0e68eaaf4bbbf40
```

여기까지 결과를 모두 확인하였다면 다음 실습을 위해 적용했던 모든 설정을 삭제합니다.

```code
kubectl delete -f .
```
