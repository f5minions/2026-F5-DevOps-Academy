# Ingress Controller Lab #1 - Basic Ingress with CIS 

## Lab 개요 
* NGINX Ingrss Controller 를 통한 서비스 배포 ( Ingress / VirtualServer )
* IngressLink 를 통한 F5 BIG-IP 와의 서비스 통합 


* 랩의 결과 
    - https://cafe.ing.example.com/coffee , /tea 접속
    - https://cafe.vs.example.com/coffee, /tea 접속
    - F5 BIGIP 에 IngressLink 서비스 배포 


## 코드 배포 

### #1 Sample App 배포 

* 샘플 서비스로 활용할 Application 및 기본 Ingress / Virtual Server 리소스 배포 
    ```Basic
    kubectl apply -f 0.cafe.yaml
    kubectl apply -f 1.cafe-secret.yaml
    ```
### #2 기본 Ingress 및 Virtual Server 배포 

* 서비스 배포 

    ```Basic
    kubectl apply -f 2.cafe-ingress.yaml
    kubectl apply -f 3.cafe-virtualserver.yaml 
    ```
* 리소스 생성 여부 검증 

    ```Basic
    kubectl get ingress 
    kubectl get virtualservers.k8s.nginx.org
    ```

### #3. Ingress Link 서비스 배포 

* IngressLink 서비스 배포 
    * k8s 에는 IngressLink 서비스를 생성하고, 
    F5 BIG-IP 에는 해당 정보를 토대로 Virtual Server 등 리소스가 생성된다 
    ```Basic
    kubectl apply -f 4-ingresslink.yaml 
    ```

* 리소스 생성 여부 검증 
    ```Basic
    kubectl get ingresslink -A 

    NAMESPACE   NAME            IPAMVSADDRESS   AGE
    nginx       nginx-ingress   10.1.10.100     14m
    ```

### #4. 서비스 접속 테스트
* 배포된 NGINX Ingress Controller / Virtual Server 에 대한 접속 시도 
    ```Basic
    * Ingress 서비스 접속
    curl -k https://cafe.ing.example.com/coffee
    curl -k https://cafe.ing.example.com/tea

    * Virtual Server 서비스 접속
    curl -k https://cafe.vs.example.com/coffee
    curl -k https://cafe.vs.example.com/tea
    ```

### #5. F5 BIG-IP 상태 확인 

* [Web] - [Local Traffic]-[Virtual Servers] - 우측 상단 파티션 k8s-1으로 설정 
    * F5 BIG-IP 에 Virtual Server 생성 (10.1.10.100)
![Lab](bigip3.png)

* [Web] - [Local Traffic] - [Pools] 
    * F5 BIG-IP 가 서비스 보내는 대상 (NGINX Ingress Controller)
![Lab](bigip4.png)