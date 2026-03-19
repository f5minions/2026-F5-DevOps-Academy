# 여러 URL 경로에서 멀티 WAF 활성화하기

이번 실습에서는 NGINX Ingress Controller(NIC)의 VirtualServer 리소스를 통해 게시된 애플리케이션의 서로 다른 URL 경로에 대해 각각 다른 WAF 정책을 정의하는 방법을 알아봅니다. 구성은 다음과 같습니다.

- **/tea** 경로: **nap-tea** 정책 사용
- **/coffee** 경로: **nap-coffee** 정책 사용
- **그 외 모든 경로**: **nap-cocoa** 정책 사용

### 실습 내용 수행

> *Client-vscode의 터미널에서 데모를 진행합니다.*

1. 작업 디렉토리로 이동합니다.

```bash
cd ~/2026-F5-DevOps-Academy/4.f5-waf-for-nginx/4.2_nic_multi_waf
```

2. apps.yml 파일을 배포합니다.

Deployment 및 Service를 생성합니다. 총 3개의 Deployment와 3개의 Service가 추가로 생성됩니다.

```bash
kubectl apply -f apps.yml
```

```bash
kubectl get pod -n nap && kubectl get service -n nap
```

3. 3가지 고유한 APPolicy를 생성합니다.

애플리케이션 별로 Policy를 적용하기 위해 3개의 APPolicy를 생성합니다. 각 Policy을 쉽게 구분할 수 있도록, 정책 이름이 포함된 서로 다른 차단 메시지를 설정할 것입니다.

```bash
kubectl apply -f appolicy-coffee.yml
kubectl apply -f appolicy-tea.yml
kubectl apply -f appolicy-cocoa.yml
```

```bash
kubectl get APPolicy -n nap
```

4. log.yml 파일을 배포합니다.
   모든 Policy에 공통으로 사용할 로그를 구성합니다. 4.1 실습에서 사용한 Log와 동일하기 때문에 unchanged 출력은 정상입니다.

```bash
kubectl apply -f log.yml
```

```bash
kubectl get APLogConf -n nap
```

5. policy.yml 파일을 배포합니다.

앞서 생성한 APPolicy와 APLogConf를 활용하여 각 애플리케이션에 적용하기 위한 최종 NGINX Policy를 생성합니다. Log 정보는 Elasticsearch로 전송합니다.

```bash
kubectl apply -f policy-coffee.yml
kubectl apply -f policy-tea.yml
kubectl apply -f policy-cocoa.yml
```

```bash
kubectl get Policy.k8s.nginx.org -n nap
```

6. virtual-server.yml 파일 내용을 확인합니다.

다음 구성을 사용하여 VirtualServer 리소스를 생성하게 됩니다.

- **/tea** 경로 요청은 tea 서비스로 보내고, **nap-tea** WAF 정책을 사용합니다.
- **/coffee** 경로 요청은 coffee 서비스로 보내고, **nap-coffee** WAF 정책을 사용합니다.
- **그 외 모든 경로** 요청은 cocoa 서비스로 보내고, **nap-cocoa** WAF 정책을 사용합니다.

```bash
cat virtual-server.yml
```

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: nap-cafe
  namespace: nap-vs
spec:
  host: nap-cafe.f5k8s.net
  policies:
  - name: cocoa-policy     # <---- Catch-all (기본) NAP 정책
  upstreams:
  - name: cocoa
    service: cocoa-svc
    port: 80
  - name: coffee
    service: coffee-svc
    port: 80
  - name: tea
    service: tea-svc
    port: 80  
  routes:
  - path: /
    action:
      pass: webapp
  - path: /tea
    policies:
    - name: tea-policy            # <---- /tea 경로 전용 NAP 정책
    action:
      pass: webapp
  - path: /coffee
    policies:
    - name: coffee-policy         # <---- /coffee 경로 전용 NAP 정책
    action:
      pass: webapp
```

7. virtual-server.yml 파일을 배포합니다.

```bash
kubectl apply -f virtual-server.yml
```

```bash
kubectl get virtualserver.k8s.nginx.org -n nap
```

8. XSS 공격을 보내 WAF가 각 애플리케이션 별로 적용되었는지 확인합니다.

**/tea** 요청을 보냅니다.

```bash
curl "http://nap-cafe.f5k8s.net/tea/?user=<script>"
```

```html
<html><head><title>Custom Reject Page</title></head><body>Blocked from NAP policy: NAP-TEA <br><br>Your support ID is: 13956862429059552813<br><br></body></html>
```

**/coffee** 경로로 요청을 보냅니다.

```bash
curl "http://nap-cafe.f5k8s.net/coffee/?user=<script>"
```

```html
<html><head><title>Custom Reject Page</title></head><body>Blocked from NAP policy: NAP-COFFEE <br><br>Your support ID is: 9293616283769441734<br><br></body></html>
```

`/tea` 또는 `/coffee`가 아닌 다른 경로로 요청을 보냅니다.

```bash
curl "http://nap-cafe.f5k8s.net/unknown/?user=<script>"
```

```html
<html><head><title>Custom Reject Page</title></head><body>Blocked from NAP policy: NAP-COCOA <br><br>Your support ID is: 13980750205359556035<br><br></body></html>
```

예상대로 각 경로별로 다른 WAF 정책이 적용된 것을 확인할 수 있습니다.

# LAB 4.2 End