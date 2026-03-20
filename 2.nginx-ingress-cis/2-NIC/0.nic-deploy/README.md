# Ingress Controller Lab #0 - NIC Deploy 

## Lab 개요
* Helm Chart 를 통한 NIC Deploy 

* 랩의 결과
    - NGINX Ingress Controller 4.0.1 Deploy 
    - NodePort Service 생성 

* Lab 사전 필수 
    - F5 CIS Deploy 

## NGINX Ingress Controller Deploy 
* Helm Chart 를 이용한 NIC Deploy 

```Basic
helm install nic nginx-stable/nginx-ingress --namespace nginx -f values.yaml --skip-crds

## Remove
helm uninstall -n nginx nic
```

* 배포 현황 확인 
    * 배포가 잘 되지 않을때는 해당 리소스들의 상태를 체크하면 도움이 될 수 있습니다. 

```Basic
kubectl get deployment -n nginx 
kubectl get replicaset -n nginx 
kubectl get po -n nginx 
kubectl get svc -n nginx 
```

---

