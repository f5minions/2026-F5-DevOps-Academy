
## CIS Deploy 
Kubernetes 에 F5 CIS Controller 를 배포하는 과정입니다. 

### Helm Repo Add

    helm repo add f5-stable https://f5networks.github.io/charts/stable

### Secret 확인 (SKIP)

* BIG-IP 계정정보 생성 여부 확인 

    ```Basic
    kubectl get secret -n bigip 
    ```
    ```yaml
    apiVersion: v1
    items:
    - apiVersion: v1
    data:
        password: SW5ncmVzc2xhYjEyMw==
        username: YWRtaW4=
    kind: Secret
    metadata:
        annotations:
        kubectl.kubernetes.io/last-applied-configuration: |
            {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{},"name":"bigip-login","namespace":"bigip"},"stringData":{"password":"Ingresslab123","username":"admin"},"type":"kubernetes.io/basic-auth"}
        creationTimestamp: "2023-08-25T06:39:00Z"
        name: bigip-login
        namespace: bigip
        resourceVersion: "9881"
        uid: df34cdc6-75f2-449f-a781-3d86a047495b
    type: kubernetes.io/basic-auth
    kind: List
    metadata:
    resourceVersion: ""
    

### CIS Install

* 아래의 Helm Chart 의 values.yaml 로 배포 

    ```yaml
    bigip_login_secret: bigip-login  ## F5 BIG-IP 접속정보 (Pre-Defined)
    rbac:
    create: true
    serviceAccount:
    create: true
    name: k8s-bigip-ctlr-ingressonly
    namespace: bigip
    ingressClass:
    create: false
    ingressClassName: f5
    isDefaultIngressController: false
    args:
    bigip_url: 10.1.20.5              ## F5 BIG-IP SELF IP (Pre-Defined)
    bigip_partition: k8s-1            ## F5 BIG-IP Partition (Pre-Defined)
    verify_interval: 30
    node-poll_interval: 30
    log_level: DEBUG
    pool_member_type: nodeport
    insecure: true
    agent: as3
    custom-resource-mode: true
    log-as3-response: true
    image:
    user: f5networks
    repo: k8s-bigip-ctlr
    pullPolicy: Always
    version: 2.20.1
    affinity:
    ```
* CIS POD Deploy 

    * --skip-crds : UDF Lab 에 CRD 가 먼저 배포되어 있어서 해당 인자값 Helm 배포시 추가 
    ```BASIC
    helm install -f values.yaml f5-cis-deploy f5-stable/f5-bigip-ctlr  --skip-crds
    ```
* Deploy Status Check
    ```BASIC
    kubectl get po -n bigip 

    NAME                                           READY   STATUS    RESTARTS   AGE
    f5-cis-deploy-f5-bigip-ctlr-794d56576b-r4l4h   0/1     Running   0          4s

    ...Deployed F5 CIS POD 

    NAME                                           READY   STATUS    RESTARTS   AGE
    f5-cis-deploy-f5-bigip-ctlr-794d56576b-r4l4h   1/1     Running   0          94s
    ```


<br>

### CIS Issue Trouble Shooting 

* 서비스 구성간 문제 발생시 CIS POD 로그로 모니터링 가능  

    ```BASIC
    kubectl logs -f -n bigip f5-cis-deploy-f5-bigip-ctlr-794d56576b-7lc26

    2026/03/19 05:48:43 [INFO] Successfully updated status of TransportServer:argocd/argo-cd in Cluster 
    2026/03/19 05:49:11 [DEBUG] [2026-03-19 05:49:11,022 __main__ DEBUG] config handler woken for reset
    2026/03/19 05:49:11 [DEBUG] [2026-03-19 05:49:11,023 __main__ DEBUG] loaded configuration file successfully
    2026/03/19 05:49:11 [DEBUG] [2026-03-19 05:49:11,023 __main__ DEBUG] NET Config: {}
    2026/03/19 05:49:11 [DEBUG] [2026-03-19 05:49:11,023 __main__ DEBUG] loaded configuration file successfully
    2026/03/19 05:49:11 [DEBUG] [2026-03-19 05:49:11,023 __main__ DEBUG] updating tasks finished, took 0.0005867481231689453 seconds
    2026/03/19 05:49:41 [DEBUG] [2026-03-19 05:49:41,023 __main__ DEBUG] config handler woken for reset
    2026/03/19 05:49:41 [DEBUG] [2026-03-19 05:49:41,023 __main__ DEBUG] loaded configuration file successfully
    2026/03/19 05:49:41 [DEBUG] [2026-03-19 05:49:41,023 __main__ DEBUG] NET Config: {}
    2026/03/19 05:49:41 [DEBUG] [2026-03-19 05:49:41,024 __main__ DEBUG] loaded configuration file successfully
    2026/03/19 05:49:41 [DEBUG] [2026-03-19 05:49:41,024 __main__ DEBUG] updating tasks finished, took 0.0008664131164550781 seconds
    ```
### NEXT Step

* NGINX Ingress Controller Deploy
* NGINX Ingress Link 서비스 배포 