[블로그 바로가기](https://velog.io/@parkd/%ED%94%84%EB%9F%B0%ED%8A%B8%EC%97%94%EB%93%9C-%EA%B0%9C%EB%B0%9C%EC%9D%84-%EC%9C%84%ED%95%9C-%EB%B3%B4%EC%95%88-%EC%9E%85%EB%AC%B8-%EC%8A%A4%ED%84%B0%EB%94%94-%ED%9A%8C%EA%B3%A0)

![](https://velog.velcdn.com/images/parkd/post/a0abae2f-2cbe-4775-8892-4dc6d7f5b777/image.png)

<br/>

요즘 보안 관련 이슈들이 많습니다. 재직 중인 회사에서도 실제 보안 사고로 인해 적지 않은 어려움을 겪은 바 있으며, 관련 사건들을 살펴보던 중 보안에 대해 다시금 정리하고자 해당 스터디를 진행하게 되었습니다.

선택한 도서는 '[프런트엔드 개발을 위한 보안 입문](https://product.kyobobook.co.kr/detail/S000001766491)'이며, 아래의 목차대로 총 6주간 스터디를 진행하였습니다. 이 기간 동안 쌓인 다양한 질문과 토론을 중심으로 회고를 써 내려가 보겠습니다.

> **회고를 끝까지 읽으신다면, 다음과 같은 질문에 대해 실마리를 얻을 수 있습니다.**

- [Mixed Content로 인해 특정 기종에서만 이미지가 미 노출된 사건](#q1-mixed-content로-인해-특정-기종에서만-이미지가-미-노출된-사건)
- [Simple Request에서 POST가 포함되는 이유는?](#q1-simple-request에서-post가-포함되는-이유는)
- [XSS를 프레임워크단에서 어떻게 막고 있을까?](#q1xss를-프레임워크단에서-어떻게-막고-있을까)
- [실제 일어난 CSRF, 클릭재킹, 오픈 리다이렉트 이슈들](#q1csrf--twitter의-악성-트윗-자동-업로드-사건-2009)
- [라이브러리 타이포스쿼팅 공격 사례](#q1라이브러리-타이포스쿼팅-공격-사례)

각 챕터별 학습 포인트를 먼저 살펴본 후, 교재를 학습하고 회고 내용을 함께 따라가 보시면 실제 스터디에 참여하는 듯한 몰입감을 경험하실 수 있습니다.

---

아래부터는 각 챕터별 학습 포인트와 함께, 실제 경험과 토론을 중심으로 정리한 회고를 이어갑니다.

## 📚 목차

- [1. HTTP 기초](#1http)
- [2. Origin 정책과 CORS](#2origin-정책과-cors)
- [3. XSS](#3xss)
- [4. CSRF, 클릭재킹, 오픈 리다이렉트](#4csrf-클릭재킹-오픈-리다이렉트)
- [5. 라이브러리 보안 리스크](#5라이브러리-보안-리스크)

# 1.HTTP

## 💡 학습 포인트

- HTTP는 요청 1개, 응답 1개가 전부인 프로토콜이며, stateless라 불린다. 즉 서버는 이전 요청을 알 수 없다. 이를 위해선 어떻게 해야할까?
- HTTP는 데이터를 주고 받는 규약이며, TCP는 데이터를 어떻게 안정적으로 전달할지 담당하는 통로라고 보면 된다. 이는 어떻게 구현되어 있을까?
- secure context 전용 API가 있다. secure context는 무엇일까
- HTTPS로만 제어하고싶은 사이트에서 사용자가 강제로 HTTP로 접속한다면? 대응은 어떻게 할 수 있을까

> [학습 링크](https://github.com/dev-bookclub/fe-security/blob/main/dhyun2/2.HTTP.md)

## 💬 회고

### Q1. Mixed Content로 인해 특정 기종에서만 이미지가 미 노출된 사건

![](https://velog.velcdn.com/images/parkd/post/bb7c49a5-e303-4a6a-9902-e534e6279572/image.png)

1주차에서 다룬 내용 중, 실제 업무에서 경험한 **Mixed Content** 이슈와 맞닿아 있어 공유되었습니다.

당시 등록된 이미지를 배너로 출력하는 단순한 작업 중, iOS의 특정 버전에서 이미지가 로드되지 않는 현상이 발생했습니다.

문제의 원인을 추적하는 과정에서 **ATS(App Transport Security)** 정책에 의해 HTTP 이미지가 차단되고 있었음을 확인하게 되었고, 이를 계기로 그동안 혼용되던 **http/https** 리소스를 전면적으로 **https** 기반으로 통합하는 계기가 되었습니다.

### Q2. Next.js middleware에서 발견된 취약점

![](https://velog.velcdn.com/images/parkd/post/82941436-4261-4ac9-8f5a-366722230e28/image.png)

1주차 진행중 이슈화된 취약점으로, 같이 논의하고 토론하고자 공유되었습니다.

> “단순히 외부에서 오는 요청만 차단하면 되는 것이라면, 이 헤더는 애초에 어떤 목적이었을까?”,
> “내부 요청에 한정해 작동하는 용도였다면, 왜 외부에서 접근이 가능했을까?”

이와 같은 다양한 의문점이 제기되었고, 이를 바탕으로 생산적인 토론을 진행하였습니다.
<br/>

---

# 2.Origin 정책과 CORS

cors는 HTTP 헤더에서 설정하는 정책으로, 웹 사이트와 애플리케이션이 다른출처에서 발생하는 특정 요청에 대한 보호를 선택할 수 있게 합니다. [MDN HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cross-Origin_Resource_Policy)

## 💡 학습 포인트

- Simple Request는 안전한가?
- Simple Request에서 POST가 포함되는 이유는 무엇일까?
- iframe 등에서 교차 출처(Cross-Origin) 제한이 중요한데, 왜 postMessage는 예외적으로 안전하게 허용되는 걸까?
- Preflight의 정의 및 우회방법

> [학습 링크](https://github.com/dev-bookclub/fe-security/blob/main/dhyun2/3.%20Origin%EC%97%90%20%EB%8C%80%ED%95%9C%20%EC%95%A0%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98%20%EA%B0%84%20%EC%A0%91%EA%B7%BC%20%EC%A0%9C%ED%95%9C.md)

## 💬 회고

### Q1. Simple Request에서 POST가 포함되는 이유는?

![](https://velog.velcdn.com/images/parkd/post/3324045e-14a0-480f-a933-25c5f9c2d0b9/image.png)

POST는 민감한 요청이지만, Simple Request로 분류되어 이에대한 논의를 진행하였습니다.

간결하게 요약하자면

웹 초창기부터 **`<form>`** 방식의 POST 요청이 널리 사용되어 왔고, 이를 막으면 기존 웹 **호환성**이 깨지기 때문입니다.

실제로 **Simple Request는 Content-Type이** `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`일 때만 허용됩니다.

더불어 [Evert Pot](https://evertpot.com/no-cors/)의 글을 인용하자면 `CORS`는 보안을 강화하는 것이 아니라, 기존 브라우저 보안(SOP)에서 허용 범위를 선택적으로 해제하는 메커니즘으로 보면 좋습니다.
`no-cors`는 그 선택적 해제를 아예 요청하지 않는 제한 모드이며, 실질적인 활용도는 매우 낮습니다.

이처럼 Simple Request의 존재 이유와 no-cors의 역할은 보안과 호환성 사이에서 균형을 맞춘 결과물로 볼 수 있다는 논의가 오갔습니다.

### Q2. Vite 등으로 웹 애플리케이션 빌드 시 crossorigin 속성이 포함되는 이유

![](https://velog.velcdn.com/images/parkd/post/4419c572-7ff8-4a03-b35b-c85e778755bf/image.png)

번들러로 웹 앱을 빌드하면 종종 `crossorigin` 속성이 자동으로 포함된 `<script>` 태그가 생성되는데 이에대한 논의를 진행하였습니다.

이를 간결하게 요약하자면 아래와 같습니다.

1. ES Modules는 CORS 기반 동작이 기본
   `<script type="module">`은 내부적으로 CORS 모드 요청으로 처리됨
   브라우저는 "출처가 다른 script 모듈이면 보안 체크하자"는 정책

2. 정확한 오류 추적을 위해 필요
   crossorigin이 없으면 JS 로드 실패 시 **콘솔에 “Script error”**만 출력됨
   crossorigin="anonymous"가 있으면 정확한 에러 메시지와 위치 추적 가능

3. CDN 등 외부 JS 로딩 시 필수
   정적 리소스가 외부 도메인(예: <https://cdn.example.com/app.js)일> 경우 CORS 처리 필수

<br/>

---

# 3.XSS

XSS(Cross-Sitre Scripting)은 공격자가 웹 사이트에 악성 스크립트를 삽입하여 다른 사용자의 브라우저에 실행되게 하는 공격 입니다.
동일 출처 정책(SOP)으로도 방어되지 않으며, 웹앱의 입력이나 DOM조작의 취약점을 타고 침투합니다.

## 💡 학습 포인트

- XSS에는 어떤 종류들이 있고 어떻게 대응해야할까?
- DOM 조작시 innerHTML, document.write()는 위험하다. 어떤식으로 대처해야할까?
- 프레임워크에서는 XSS를 어떻게 막고있을까?

> XSS방어를 위한 모범사례
>
> - 모든 사용자 입력을 신뢰하지 말고 검증하기
> - 무조건 필요한 경우가 아니라면 innerHTML, outerHTML, document.write() 사용 피하기
> - 컨텐츠 보안 정책(CSP) 구현하기
> - 쿠키에 HttpOnly, Secure, SameSite 플래그 사용하기
> - 프레임워크의 기본 보안 기능 활용하기
> - HTML 정화(sanitization) 라이브러리 사용하기
> - 정기적인 보안 검사 및 취약점 스캔 수행하기
> - XSS 공격 대응 계획 수립하기

> [학습링크](https://github.com/dev-bookclub/fe-security/blob/main/dhyun2/XSS/4.%20XSS.md)

## 💬 회고

### Q1.XSS를 프레임워크단에서 어떻게 막고 있을까?

실제 프런트엔드 개발에서는 사용자의 입력값이나 외부 콘텐츠를 DOM에 렌더링해야 하는 상황이 종종 발생합니다. 이때 단순히 innerHTML이나 dangerouslySetInnerHTML, v-html과 같은 API를 사용하는 것은 XSS(교차 사이트 스크립팅) 공격에 취약할 수 있습니다.

이에 따라 “불가피하게 HTML 문자열을 DOM에 삽입해야 할 경우, 어떤 방식으로 이를 안전하게 처리할 수 있을까?”에 대한 토론을 진행하였고, 프레임워크별 보안 전략에 대해서도 비교해 보았습니다.

> [VUE의 XSS방어 메커니즘](https://github.com/dev-bookclub/fe-security/blob/main/dhyun2/XSS/vue%EC%97%90%EC%84%9C%EC%9D%98%20XSS%20%EB%8C%80%EC%9D%91.md) > [React에서의 XSS방어 매커니즘](https://github.com/dev-bookclub/fe-security/blob/main/dhyun2/XSS/react%EC%97%90%EC%84%9C%EC%9D%98%20XSS%20%EB%8C%80%EC%9D%91.md)

### Q2.브라우저 기반 XSS가 얼마나 쉽게 fetch 요청을 탈취할 수 있는지

공격자가 js실행권한을 얻으면(XSS) window.fetch를 가로채는 것이 가능합니다. 즉 monkey patching에 관하여 논의를 진행하였습니다.

![](https://velog.velcdn.com/images/parkd/post/c57f0a09-9190-45f6-b9ac-f218d6eaae69/image.png)

위와 같이 너무나 쉽게도 다양한 정보를 탈취할 수 있다는걸 공유해주셨습니다.

<br/>

---

# 4.CSRF, 클릭재킹, 오픈 리다이렉트

XSS와 더불어 실무에서 자주 마주하는 대표적인 수동적 공격 3종입니다.

`CSRF(Cross Site Request Forgery)`는 공격자가 준비한 함정에 의해 웹 애플리케이션이 원래 갖고 있는 기능이 사용자의 의도와는 상관없이 호출되는 공격입니다.

`클릭재킹(Clickjacking)`은 사용자가 의도하지 않은 클릭을 하도록 유도하여 공격자가 원하는 작업을 수행하게 하는 공격입니다.

`오픈 리다이렉트(open redirect)`는 웹 리다리덱트 기능을 이용해 피싱 사이트 등 공격자가 준비한 사이트로 강제 이동시키는 공격이다.

## 💡 학습 포인트

- CSRF는 어떻게 사용자의 의지와 무관하게 요청을 보낼 수 있을까?
- 클릭재킹은 어떻게 감지하고 방어할 수 있을까?
- 오픈 리다이렉트는 어떻게 진행될까?

> [학습링크](https://github.com/dev-bookclub/fe-security/blob/main/dhyun2/4.CSRF%2C%20%ED%81%B4%EB%A6%AD%EC%9E%AC%ED%82%B9%2C%20%EC%98%A4%ED%94%88%20%EB%A6%AC%EB%8B%A4%EC%9D%B4%EB%A0%89%ED%8A%B8.md)

## 💬 피싱 케이스

이번 주차는 토론을 진행하지 않아, 피싱 케이스를 공유하겠습니다.

### Q1.CSRF – Twitter의 악성 트윗 자동 업로드 사건 (2009)

**사건 개요**: 트위터 로그인 상태에서 사용자가 링크를 클릭하면, 계정으로 자동 트윗이 올라간 사건

```html
<img src="https://twitter.com/status/update?status=Follow+@attacker" />
.
```

- 당시 트위터는 GET 요청만으로 트윗 전송이 가능했고, CSRF 토큰도 없었습니다.
- 사용자는 클릭 한 번 없이 악성 트윗을 자기 계정으로 전파하게 되었음

**대응 이후**

- 모든 POST 요청에 CSRF 토큰 필수화
- SameSite 쿠키 및 Origin 검증 강화

### Q2. 클릭재킹 – Facebook “Like 버튼 낚시” 사건 (2010)

**사건 개요**: 투명 iframe을 이용해 사용자가 Facebook Like 버튼을 의도치 않게 클릭하게 만든 사건

```html
<iframe src="https://facebook.com/like/attacker" style="opacity: 0; z-index: 9999;"></iframe>
```

- 사용자는 “무료 상품 받기” 버튼을 클릭했지만 실제로는 Facebook Like 클릭이 발생

**대응 이후**

- X-Frame-Options: DENY 도입
- frame busting 스크립트 및 iframe 제한 강화

### Q3. 오픈 리다이렉트 – Google OAuth redirect 취약점 (2014)

**사건 개요**: OAuth 인증 redirect_uri 검증이 느슨하여 피싱 사이트로 리디렉션이 가능했던 사건

```bash
https://accounts.google.com/o/oauth2/auth?client_id=...&redirect_uri=https://attacker.com/fake-login
```

- 사용자는 인증 후 공격자의 가짜 로그인 페이지로 이동, 자격 증명을 탈취당할 위험 존재

**대응 이후**

- 사전 등록된 URI와 정확히 일치할 경우만 허용
- OAuth 명세에서도 redirect_uri 고정 권장

<br/>

---

# 5.라이브러리 보안 리스크

라이브러리에는 **공급망 공격(Supply chain Attack)**이라는 잠재적인 보안 위협이 숨어있습니다.

## 💡 학습 포인트

- 오픈소스 라이브러리 사용 시 어떤 경로로 취약점이 발생할 수 있을까?
- npm audit 을 통한 취약점 확인 및 자동 업데이트

> [학습링크](https://github.com/dev-bookclub/fe-security/blob/main/dowoon/chapter08.md)

## 💬 회고

### Q1.라이브러리 타이포스쿼팅 공격 사례

**croessenv 사건**
crossenv 패키지는 인기 있는 cross-env 패키지의 이름을 모방하여 사용자들을 속이기 위해 만들어진 악성 npm 패키지입니다.

이를 활용하여 공격자는 환경변수 등을 수집할 수 있었습니다.

> 관련 이슈 링크
> [github advisories](https://github.com/advisories/GHSA-hwhq-3hrj-v6v5) > [npm blog](https://blog.npmjs.org/post/163723642530/crossenv-malware-on-the-npm-registry)

`npm-lockfile-lint`나 `Private registry`를 통해 npm install에 제약을 거는 형태등을 논의하였습니다.

### Q2.npm audit 사용 경험에 대한 논의

![](https://velog.velcdn.com/images/parkd/post/95764b1c-cbca-44c2-9dc9-2e174782d0e4/image.png)

실제 현업에서의 npm audit 사용 빈도등, 패키지 보안을 어떻게 관리하는지 논의하였습니다.

# 스터디 회고

## 마무리

스터디 마무리로 남은 15분간 figjam에 교재내용을 다이어그램으로 그려보았습니다.
![](https://velog.velcdn.com/images/parkd/post/0ceef373-69ba-46a4-aef9-60d9ebafb826/image.png)

스터디 마무리로 15분간 FigJam에 회고 다이어리를 그려보며 스터디는 마무리 되었습니다.

![](https://velog.velcdn.com/images/parkd/post/7ff2d54a-b950-4ebc-863d-535b9f6a1c6f/image.png)
이번 스터디를 통해 도운님의 후기처럼, 저 역시 최근 잇따르는 보안 이슈들로 인해 보안에 대한 경각심을 가지게 되었고,
프론트엔드 개발자가 제어할 수 있는 범위 안에서 어떤 방식으로 문제를 예방하고 대응해나갈 수 있을지에 대해 많은 실마리를 얻을 수 있었습니다.

열심히 참여해준 팀원분들께 감사의 인사를 드리며. 이번 스터디를 마무리하겠습니다.

앞으로의 여정도 각자의 방향성에 따라 흔들림 없이 나아가시길 진심으로 응원합니다.
박동현 드림

## 스터디 정보

> [스터디 레포 링크](https://github.com/dev-bookclub/fe-security) > [교재 링크](https://product.kyobobook.co.kr/detail/S000211709203)

---

읽어주셔서 감사합니다.
