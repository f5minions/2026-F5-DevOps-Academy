# F5 WAF for NGINX 소개

**F5 WAF for NGINX**(구 NGINX App Protect WAF)는 현대적인 애플리케이션과 API를 보호하기 위해 설계된 엔터프라이즈급 보안 솔루션입니다. NGINX Plus에 강력한 F5 WAF 엔진을 결합하여, DevOps 환경에서의 민첩성을 유지하면서도 강력한 위협 방어 기능을 제공합니다.

### 주요 특징

- **고성능 및 경량화:** 컨테이너 및 마이크로서비스 환경에 최적화된 뛰어난 성능을 제공합니다.
- **DevSecOps 통합:** 선언적(Declarative) 정책 관리를 통해 CI/CD 파이프라인에 보안을 원활하게 통합할 수 있습니다.
- **강력한 위협 방어:** OWASP Top 10, Layer 7 DoS, 봇 공격 등 다양한 정교한 공격으로부터 애플리케이션을 보호합니다.
- **API 보안:** OpenAPI 스키마 준수 및 API 관련 공격에 대한 강력한 방어 기능을 수행합니다.

<p align="center">
  <img src="images/waf-architecture.png">
  <br><em>[F5 WAF for NGINX 아키텍처]</em>
</p>

---

# 다양한 공격 유형으로부터 보호

이 섹션에서는 다양한 유형의 공격에 대해 **F5 WAF for NGINX**를 구성하는 방법을 알아봅니다. 이러한 공격 유형에 대해 차단(Blocking) 모드를 활성화하고, 대시보드에서 로그를 검토하며, 공격이 *오탐(False Positive)*인 경우 예외를 생성하는 과정을 살펴봅니다. 다룰 공격 유형은 다음과 같습니다:

- **공격 시그니처 (Attack Signatures)**
- **HTTP 메소드 (HTTP Methods)**

### 기본 환경 구성

> *Client-vscode의 터미널에서 데모를 진행합니다.*

이번 실습은 앞서 구성된 K8S 환경과 NIC를 이용하여 애플리케이션을 배포하고 이를 보호하기 위한 기본 WAF 정책을 적용합니다.
(NIC Pod에 WAF가 포함되어 있습니다.)

K8S 프로파일을 변경하고 작업 디렉토리로 이동합니다.

```bash
kubectl config use-context rancher1
```

```bash
cd ~/2026-F5-DevOps-Academy/4.f5-waf-for-nginx/4.1_nic_waf
```

1. app.yml 파일 내용을 확인합니다.

```bash
cat app.yml
```

Webapp을 배포하기 위해 Deployment와 Service가 정의되어 있습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: nap
# .................. 생략 ..................
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
  namespace: nap   
# .................. 생략 .................. 
```

2. app.yml 파일을 배포합니다.

```bash
kubectl create namespace nap
kubectl apply -f app.yml
```

```bash
kubectl get pod -n nap && kubectl get service -n nap
```

3. appolicy.yml 파일 내용을 확인합니다.

```bash
cat appolicy.yml
```

해당 정책은 기본 템플릿(POLICY_TEMPLATE_NGINX_BASE)를 사용합니다. **POLICY_TEMPLATE_NGINX_BASE**는 F5 WAF for NGINX에서 정책(policy)을 작성할 때 사용하는 기본 템플릿입니다. 모든 정책의 출발점 역할을 하며, 수정하지 않은 상태의 기본 정책과 동일하게 동작합니다.

```yaml
apiVersion: appprotect.f5.com/v1beta1
kind: APPolicy
metadata:
  name: nap-demo
  namespace: nap
spec:
  policy:
    applicationLanguage: utf-8
    enforcementMode: blocking
    name: nap-demo
    template:
      name: POLICY_TEMPLATE_NGINX_BASE
```

4. apppolicy.yml 파일을 배포합니다.

```bash
kubectl apply -f appolicy.yml
```

```bash
kubectl get APPolicy -n nap
```

5. log.yml 파일 내용을 확인합니다.

```bash
cat log.yml
```

출력되는 WAF의 로그 Format을 정의하고, 최대 매세지 크기, 요청 크기 등을 정의합니다.

```yaml
apiVersion: appprotect.f5.com/v1beta1
kind: APLogConf
metadata:
  name: logconf
  namespace: nap
spec:
  content:
    format: user-defined
    format_string: "{\"campaign_names\":\"%threat_campaign_names%\",\"bot_signature_name\":\"%bot_signature_name%\",\"bot_category\":\"%bot_category%\",\"bot_anomalies\":\"%bot_anomalies%\",\"enforced_bot_anomalies\":\"%enforced_bot_anomalies%\",\"client_class\":\"%client_class%\",\"client_application\":\"%client_application%\",\"json_log\":%json_log%}"
    max_message_size: 30k
    max_request_size: "500"
    escaping_characters:
    - from: "%22%22"
      to: "%22"
  filter:
    request_type: illegal
```

6. log.yml 파일을 배포합니다.

```bash
kubectl apply -f log.yml
```

```bash
kubectl get APLogConf -n nap
```

7. policy.yml 파일 내용을 확인합니다.

```bash
cat policy.yml
```

앞서 생성한 APPolicy와 APLogConf를 활용하여 Webapp에 적용하기 위한 최종 NGINX Policy를 생성합니다. Log 정보는 Elasticsearch로 전송합니다.

```yaml
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata:
  name: waf-policy-demo
  namespace: nap
spec:
  waf:
    enable: true
    apPolicy: "nap-demo"
    securityLogs:
    - enable: true
      apLogConf: "logconf"
      logDest: "syslog:server=fluentd-svc.fluentd:8515"
    - enable: true
      apLogConf: "logconf"
      logDest: "syslog:server=127.0.0.1:1514"
```

8. policy.yml 파일을 배포합니다.

```bash
kubectl apply -f policy.yml
```

```bash
kubectl get Policy.k8s.nginx.org -n nap
```

9. virtual-server.yml 파일 내용을 확인합니다.

```bash
cat virtual-server.yml
```

Webapp에 Policy를 적용하여 NIC를 통해 외부로 유출시키는 구성임을 확인합니다.

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: webapp
  namespace: nap
spec:
  host: nap.f5k8s.net
  policies:
  - name: waf-policy-demo
  upstreams:
  - name: webapp
    service: webapp-svc
    port: 80
  routes:
  - path: /
    action:
      pass: webapp
```

10. virtual-server.yml 파일을 배포합니다.

```bash
kubectl apply -f virtual-server.yml
```

```bash
kubectl get virtualserver.k8s.nginx.org -n nap
```

11. 정상적인 요청을 보내 애플리케이션을 테스트합니다:

```bash
curl http://nap.f5k8s.net/
```

```bash
Server address: 10.221.0.34:80
Server name: webapp-67dccc8679-kk6r4
Date: 19/Mar/2026:02:22:40 +0000
URI: /
Request ID: 09abb04dfd4d762f2d206caceaf0d90b
```

12. Cross-Site Scripting(XSS) 공격을 보내 WAF가 제대로 적용되었는지 확인합니다.

```bash
curl "http://nap.f5k8s.net/?test=<script>"
```

```html
<html><head><title>Request Rejected</title></head><body>The requested URL was rejected. Please consult with your administrator.<br><br>Your support ID is: 9293616283769441224<br><br><a href='javascript:history.back();'>[Go Back]</a></body></html>
```

13. Grafana에 로그인하여 대시보드에 위반 사항이 표시되는지 확인합니다. (이벤트 모니터링은 NIM에서도 가능하며, NIM을 이용한 이벤트 모니터링은 4.3 시나리오에서 다룹니다.)

<p align="center">
  <img src="images/grafana-info4.png">
</p>

Grafana의 주소 및 접속 계정 정보는 아래 스크린샷을 참고해주세요.

<p align="center">
  <img src="images/grafana-info1.png">
</p>

<p align="center">
  <img src="images/grafana-info2.png">
</p>

<p align="center">
  <img src="images/grafana-info3.png">
</p>

## Attack Signature 작업

기본 정책에는 **High Accuracy Signatures**가 차단(Blocking) 모드로 설정되어 있는 반면, 다른 많은 서명들은 알람(Alarm) 모드로만 설정되어 있습니다.
(기본 정책에 포함된 시그니처 세트를 검토하려면 [링크](https://docs.nginx.com/waf/policies/attack-signatures/#signature-settings)를 참고해주세요.)

### 특정 Signature 비활성화

시스템이 동작하기 위한 필수 요청이 WAF 정책에 막힐 경우, 예외 처리를 해야합니다. 차단 모드(Blocking)로 설정된 **High Accuracy Signatures**에 포함된 요청을 전송합니다. 이 요청을 시스템이 동작하기 위한 필수 요청이라고 가정합니다.

```bash
curl "http://nap.f5k8s.net/index.php?id=0;%20ls%20-l"
```

응답이 차단되었는지 확인합니다.

```html
<html><head><title>Request Rejected</title></head><body>The requested URL was rejected. Please consult with your administrator.<br><br>Your support ID is: 9276162884747326652<br><br><a href='javascript:history.back();'>[Go Back]</a></body></html>
```

응답에서 `support id`를 복사하고, Grafana SupportID 대시보드를 열어 해당 `support id`로 위반 사항을 검색합니다.

요청 상태가 차단(blocked)인 것을 확인하고, Hit된 Signature ID 3개의 값을 복사합니다.

<p align="center">
  <img src="images/grafana-info5.png">
</p>

특정 Signature를 예외처리 하려면, 아래의 새로운 APPolicy로 교체하여 Hit된 3개의 Signature를 비활성화 해야합니다.
다음 명령을 실행하여 APPolicy를 수정합니다.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: appprotect.f5.com/v1beta1
kind: APPolicy
metadata:
  name: nap-demo
  namespace: nap
spec:
  policy:
    applicationLanguage: utf-8
    enforcementMode: blocking
    name: nap-demo
    template:
      name: POLICY_TEMPLATE_NGINX_BASE
    signatures:
    - signatureId: 200003041
      enabled: false
    - signatureId: 200003913
      enabled: false
    - signatureId: 200103492
      enabled: false
EOF
```

몇 초(5-10초) 기다린 후 동일한 요청을 다시 실행하여 요청이 차단되지 않는지 확인합니다.

```bash
curl "http://nap.f5k8s.net/index.php?id=0;%20ls%20-l"
```

```bash
Server address: 10.221.0.34:80
Server name: webapp-67dccc8679-kk6r4
Date: 19/Mar/2026:02:34:31 +0000
URI: /index.php?id=0;%20ls%20-l
Request ID: 189afe98a106447bc40944385098879c
```

### 시그니처 활성화

이제 **Medium accuracy signatures**에 포함된 요청을 전송합니다. 해당 요청은 기본 정책 기준으로 Block이 아닌 Alarm 모드로 설정되어 있습니다.

```bash
curl "http://nap.f5k8s.net/phpinfo.php"
```

```bash
Server address: 10.221.0.34:80
Server name: webapp-67dccc8679-kk6r4
Date: 19/Mar/2026:03:53:35 +0000
URI: /phpinfo.php
Request ID: 2096a8bcda32e6f5918adb0b5836cad7
```

성공적으로 요청이 반환되었으며, NGINX App Protect에 의해 차단되지 않았습니다.

Grafana Main 대시보드를 열고 Logs를 확인합니다. `phpinfo.php` URL이 표시되어야 합니다. 이를 통해 **경고(Alerted)**는 되었지만 **차단(Blocked)**되지 않았음을 확인합니다.

<p align="center">
  <img src="images/grafana-info6.png">
</p>

이제 이 특정 Signature를 활성화하도록 구성을 변경해 보겠습니다. 이는 사용자 정의 Signature Sets를 생성하고 이 특정 Signature를 추가하여 수행할 수 있습니다. 다음 명령을 실행하여 APPolicy를 수정합니다.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: appprotect.f5.com/v1beta1
kind: APPolicy
metadata:
  name: nap-demo
  namespace: nap
spec:
  policy:
    applicationLanguage: utf-8
    enforcementMode: blocking
    name: nap-demo
    template:
      name: POLICY_TEMPLATE_NGINX_BASE
    signature-sets:
      - name: Custom-picked-signatures
        block: true
        alarm: true
        signatureSet:
          signatures:
            - signatureId: 200010015
EOF
```

몇 초(5-10초) 기다린 후 동일한 요청을 다시 실행하여 요청이 차단되는지 확인합니다.

```bash
curl "http://nap.f5k8s.net/phpinfo.php"
```

```html
<html><head><title>Request Rejected</title></head><body>The requested URL was rejected. Please consult with your administrator.<br><br>Your support ID is: 13956862429059551793<br><br><a href='javascript:history.back();'>[Go Back]</a></body></html>
```

이제 모든 **Medium accuracy signatures**를 활성화하도록 구성을 변경해 보겠습니다.
다음 명령을 실행하여 APPolicy를 수정합니다.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: appprotect.f5.com/v1beta1
kind: APPolicy
metadata:
  name: nap-demo
  namespace: nap
spec:
  policy:
    applicationLanguage: utf-8
    enforcementMode: blocking
    name: nap-demo
    template:
      name: POLICY_TEMPLATE_NGINX_BASE
    signature-sets:
    - name: "Medium Accuracy Signatures"
      block: true
      alarm: true
EOF
```

몇 초(5-10초) 기다린 후 동일한 요청을 다시 실행하여 요청이 차단되는지 확인합니다. 이제 모든 **Medium accuracy signatures**가 차단 모드이므로 요청이 **NGINX App Protect**에 의해 동일하게 차단될 것입니다.

```bash
curl "http://nap.f5k8s.net/phpinfo.php"
```

```html
<html><head><title>Request Rejected</title></head><body>The requested URL was rejected. Please consult with your administrator.<br><br>Your support ID is: 6706763416785726235<br><br><a href='javascript:history.back();'>[Go Back]</a></body></html>
```

## HTTP 메소드 작업

`허용되지 않는 메소드(Illegal Methods)`는 기본적으로 **경고(Alerted) 모드**로만 구성되어 있습니다. 기본 정책에는 미리 정의된 `허용(allowed)` HTTP 메소드 목록(GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS)이 포함되어 있습니다.

이제 위 목록에 정의되지 않은 HTTP 메소드(`LINK`)를 사용하여 애플리케이션에 액세스해 보겠습니다.

```bash
curl -X LINK "http://nap.f5k8s.net"
```

위반 사항이 **경고(Alerted) 모드**이므로 결과는 다음과 같습니다.

```text
Server address: 10.221.0.34:80
Server name: webapp-67dccc8679-kk6r4
Date: 19/Mar/2026:04:07:37 +0000
URI: /
Request ID: fe7da455c2335202478004d235a048f5
```

Grafana Main 대시보드를 열고 마지막 트랜잭션들을 확인합니다. `LINK` 메소드가 보여야 합니다.

<p align="center">
  <img src="images/grafana-info7.png">
</p>

이제 `허용되지 않는 메소드` 위반을 차단하도록 APPolicy 구성을 변경해 보겠습니다. 다음 명령을 실행하여 APPolicy를 수정합니다.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: appprotect.f5.com/v1beta1
kind: APPolicy
metadata:
  name: nap-demo
  namespace: nap
spec:
  policy:
    applicationLanguage: utf-8
    enforcementMode: blocking
    name: nap-demo
    template:
      name: POLICY_TEMPLATE_NGINX_BASE
    blocking-settings:
      violations:
      - name: VIOL_METHOD
        alarm: true
        block: true
EOF
```

`curl` 명령을 다시 실행하고 트랜잭션이 차단되는지 확인합니다.

```bash
curl -X LINK "http://nap.f5k8s.net"
```

```html
<html><head><title>Request Rejected</title></head><body>The requested URL was rejected. Please consult with your administrator.<br><br>Your support ID is: 13956862429059552303<br><br><a href='javascript:history.back();'>[Go Back]</a></body></html>
```

# LAB 4.1 End