# NIM을 통한 Policy 배포 프로세스 구성

4.1 Lab과 4.2 Lab에서는 Kubernetes 환경에서 NIC(NGINX Ingress Controller)를 활용한 WAF 정책 배포를 경험했습니다. 이번 Lab에서는 NIM(NGINX Instance Manager)을 통해 기존 NGINX Plus 인스턴스에 WAF 정책을 중앙에서 관리하고 배포하는 과정을 직접 수행해보겠습니다.

## NGINX Instance Manager 접속

1. NIM에 접속하기 위한 인증 정보를 확인합니다.

<p align="center">
  <img src="images/nim-info1.png">
</p>

<p align="center">
  <img src="images/nim-info2.png">
</p>

2. NIM UI로 접속합니다.

<p align="center">
  <img src="images/nim-info3.png">
</p>

<p align="center">
  <img src="images/nim-info4.png">
</p>

## NGINX Plus 인스턴스 NAP 구성 상태 확인

1. NIM UI에서 인스턴스(NGINX Plus)를 확인하고, WAF 구성 상태를 확인합니다.

<p align="center">
  <img src="images/nim-info5.png">
</p>

<p align="center">
  <img src="images/nim-info6.png">
</p>

<p align="center">
  <img src="images/nim-info7.png">
</p>

NGINX Plus 인스턴스에 NAP이 구성되어 있음을 확인하였습니다. 해당 인스턴스가 NAP을 활용하여 방화벽을 구성하였는지 확인합니다.

2.해당 인스턴스가 NAP을 활용하여 방화벽을 구성하였는지 확인하기 위해 NGINX Plus Config 파일을 체크합니다.

<p align="center">
  <img src="images/nim-info8.png">
</p>

모든 설정 파일을 확인합니다.

<p align="center">
  <img src="images/nim-info9.png">
</p>

예상대로 NAP 설정 컨텍스트는 작성되어 있지 않습니다. 현재 NGINX Plus 인스턴스는 NAP이 적용되고 있지 않은 상황입니다. NAP을 적용하려면 아래와 같은 컨텍스트가 포함되어 있어야 합니다.

```
app_protect_enable on;
app_protect_policy_file /etc/nginx/waf/nac-policies/nap_nap-coffee;
app_protect_security_log_enable on;
app_protect_security_log /etc/nginx/waf/nac-logconfs/nap_logconf syslog:server=fluentd-svc.fluentd:8515;
app_protect_security_log /etc/nginx/waf/nac-logconfs/nap_logconf syslog:server=127.0.0.1:1514;
```

3. XXS 공격을 보내 WAF가 작동하고 있는지 추가 검증합니다.

> *Nginx-plus-apigw의 터미널에서 데모를 진행합니다.*

```bash
curl "http://localhost:9000/?test=<script>"
```

요청이 정상적으로 반환되어 WAF가 동작하고 있지 않음을 확인할 수 있습니다.

## NIM UI를 활용한 NAP Policy 적용

1. 현재 NIM에서 관리하고 있는 Policy를 확인합니다.

<p align="center">
  <img src="images/nim-info10.png">
</p>

<p align="center">
  <img src="images/nim-info11.png">
</p>

<p align="center">
  <img src="images/nim-info12.png">
</p>

기본적으로 제공하고 있는 **NginxDefaultPolicy**를 사용하겠습니다.

2. NGINX Plus 인스턴스의 설정 파일을 수정하여 기본 템플릿이 적용된 Policy를 적용합니다.

<p align="center">
  <img src="images/nim-info13.png">
</p>

<p align="center">
  <img src="images/nim-info14.png">
</p>

nignx.conf에 아래와 같은 값을 추가하여 WAF 기능을 활성화합니다. Log 정보는 NIM으로 전송합니다.

```bash
load_module modules/ngx_http_app_protect_module.so;

http {
  app_protect_enable on;
  app_protect_policy_file /etc/nms/NginxDefaultPolicy.tgz;
  app_protect_security_log_enable on;
  app_protect_security_log /etc/nms/custom_log.tgz syslog:server=localhost:514;
}
```

<p align="center">
  <img src="images/nim-info15.png">
</p>

<p align="center">
  <img src="images/nim-info16.png">
</p>

NGINX Plus 인스턴스의 설정이 성공적으로 변경되었습니다.

<p align="center">
  <img src="images/nim-info17.png">
</p>

3. 다시 XXS 공격을 보내 WAF가 작동하고 있는지 검증합니다.

```bash
curl "http://localhost:9000/?test=<script>"
```

WAF가 정상적으로 동작하여 차단된 것을 확인할 수 있습니다.

```html
<html><head><title>Request Rejected</title></head><body>The requested URL was rejected. Please consult with your administrator.<br><br>Your support ID is: 5627465829415946071<br><br><a href='javascript:history.back();'>[Go Back]</a></body></html>
```

## NIM을 이용한 이벤트 모니터링

1. WAF 정책으로 인해 **차단(Blocked)** 되었으므로, NIM UI에서 이벤트를 확인합니다.

<p align="center">
  <img src="images/nim-info18.png">
</p>

NIM Security Monitoring Dashboard는 NAP이 탐지한 보안 이벤트를 한눈에 확인할 수 있는 대시보드입니다.

<p align="center">
  <img src="images/nim-info19.png">
</p>

앞서 XSS 공격을 전송했으므로, Status 필터를 Blocked로 설정하여 차단된 이벤트만 확인해보겠습니다.

<p align="center">
  <img src="images/nim-info20.png">
</p>

<p align="center">
  <img src="images/nim-info21.png">
</p>

이벤트 로그를 클릭하면 해당 요청에 대한 상세 정보를 확인할 수 있습니다. 요청의 Method, URI, Raw Request 등 기본 요청 정보와 함께, 어떤 Violation이 탐지되었는지, 공격의 Severity 및 Violation Rating 등 보안 분석에 필요한 정보를 한 화면에서 확인할 수 있습니다.

<p align="center">
  <img src="images/nim-info22.png">
</p>

## NIM UI를 활용한 Custom Policy 생성 및 배포

특정 요청이 Policy에 의해 막힐 경우, 예외 처리를 해야합니다. 4.1 실습에서 했던 것 처럼 아래 요청을 시스템이 동작하기 위한 필수 요청이라 가정하고 진행합니다.

1. 필수 요청을 전송합니다.

```bash
curl "http://localhost:9000?id=0;%20ls%20-l"
```

응답이 차단되었는지 확인합니다.

```html
<html><head><title>Request Rejected</title></head><body>The requested URL was rejected. Please consult with your administrator.<br><br>Your support ID is: 17707074359114218079<br><br><a href='javascript:history.back();'>[Go Back]</a></body></html>
```

2. 응답에서 `support id`를 복사하고, NIM의 **Support ID Details** 페이지에서 해당 `support id`로 위반 사항을 검색합니다.

<p align="center">
  <img src="images/nim-info23.png">
</p>

복사한 `support id` 값을 입력합니다.

<p align="center">
  <img src="images/nim-info24.png">
</p>

해당 로그의 상세 페이지로 진입합니다.

<p align="center">
  <img src="images/nim-info25.png">
</p>

3. 상세 페이지에서 위반된 Attack Signature ID를 복사합니다.

<p align="center">
  <img src="images/nim-info26.png">
</p>

4. Custom Policy를 생성합니다.

<p align="center">
  <img src="images/nim-info27.png">
</p>

기존에 사용중이던 **NginxDefaultPolicy** 를 복사하여 **custom_policy** 를 생성합니다.

<p align="center">
  <img src="images/nim-info28.png">
</p>

<p align="center">
  <img src="images/nim-info29.png">
</p>

생성한 **custom_policy** 를 수정합니다.

<p align="center">
  <img src="images/nim-info30.png">
</p>

Policy 편집 페이지에 진입합니다.

<p align="center">
  <img src="images/nim-info31.png">
</p>

**Attack Signature Exceptions** 탭을 확장하고 **Add Item** 버튼을 클릭합니다.

<p align="center">
  <img src="images/nim-info32.png">
</p>

복사한 Signature ID를 찾아서 체크 후, 추가합니다.

<p align="center">
  <img src="images/nim-info33.png">
</p>

<p align="center">
  <img src="images/nim-info34.png">
</p>

<p align="center">
  <img src="images/nim-info35.png">
</p>

총 3개의 Attack Signatrue가 추가됨을 확인합니다.

<p align="center">
  <img src="images/nim-info36.png">
</p>

NIM에서 최종 적용된 Policy를 JSON 형태로 확인할 수 있습니다. NAP의 Policy는 기본적으로 JSON 형식으로 정의되며, NIM UI를 통해 별도의 파일 편집 없이 편리하게 수정하고 배포할 수 있습니다. 저장하여 편집 페이지에서 나갑니다.
<p align="center">
  <img src="images/nim-info37.png">
</p>

저장한 Policy를 Compile합니다. Compile은 JSON 형태로 작성된 Policy를 NAP 엔진이 실제로 적용할 수 있는 바이너리 형식으로 변환하는 과정입니다. NIM은 Compile을 중앙에서 수행한 뒤 컴파일된 결과물을 각 NGINX 인스턴스에 배포하므로, 인스턴스마다 별도로 컴파일할 필요 없이 일관된 Policy를 유지할 수 있습니다.

<p align="center">
  <img src="images/nim-info38.png">
</p>

<p align="center">
  <img src="images/nim-info39.png">
</p>

5. NGINX Plus 인스턴스의 설정 파일을 수정하여 Custom Policy를 적용합니다.

<p align="center">
  <img src="images/nim-info40.png">
</p>

<p align="center">
  <img src="images/nim-info41.png">
</p>

`app_protect_policy_file`의 값을 `custom_policy.tgz`로 수정합니다.

```bash
http {
  app_protect_policy_file /etc/nms/custom_policy.tgz;
}
```

<p align="center">
  <img src="images/nim-info42.png">
</p>

<p align="center">
  <img src="images/nim-info43.png">
</p>

<p align="center">
  <img src="images/nim-info44.png">
</p>

6. 필수 요청을 전송합니다.

```bash
curl "http://localhost:9000?id=0;%20ls%20-l"
```

응답이 차단되지 않고 정상적으로 반환되는지 확인합니다.

오늘 실습에서 다룬 내용 외에도 NIM은 다양한 기능을 제공합니다. 자유롭게 UI를 탐색하며 어떤 기능들이 있는지 직접 확인해보세요.

# LAB 4.3 End