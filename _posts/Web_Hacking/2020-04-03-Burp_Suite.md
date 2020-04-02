---
title: Burp Suite & Proxy
category:
    - Web Hacking
tags:
    - Web
    - Hacking
---

수업시간에 다룬 내용 정리

### 설치 및 설정

크롬을 기준으로 작성함

#### 1. Burp Suite

일단 공식 사이트에서 설치.

Burp Suite에서는 추후 Proxy를 적용할 때, https에서도 동작할 수 있도록 인증서를 받아서 등록해야한다.

- Proxy > Options > Proxy Listeners > Import / export CA certificate
- Export의 `Certificate in DER format` 체크
- 파일명과 경로 설정함.

인증서를 등록하려면 크롬창을 열어서 다음과 같이 등록하면 된다.

- 설정 : "인증서" 검색 > 더보기 > 인증서 관리
- 가져오기에서 파일 불러옴
- 신뢰할 수 있는 루트 인증 기관에 등록

기본적인 설정은 마쳤고 추후에 사용할 때의 체크할 옵션은 다음과 같음.

- User options > Display > HTTP Message Display 에서 Font를 한글 지원폰트로 설정
- Proxy > Options > Intercept Server Responses 에서 `Intercept responses based on the following rules` 체크

#### 2. SwithyOmega

크롬 확장 프로그램에서 검색 후 설치

Option창에 들어가서 Proxy관련 설정을 한다.

- PROFILES > Proxy > Proxy servers
- Server : 127.0.0.1 / Port : 8080
- Apply changes 클릭

Direct / proxy를 클릭하여 설정할 수 있고 특정 사이트를 대상으로만 proxy를 켤 수 있음.



### 기타

- Target의 Site map을 이용하여 초기 조사
- Repeater로 특정 request보내서 유효한 cookie 체크하는 등 가능
