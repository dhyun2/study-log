[블로그 바로가기](https://velog.io/@parkd/%EB%B8%8C%EB%9D%BC%EC%9A%B0%EC%A0%80-%EB%8F%99%EC%9E%91-%EC%9B%90%EB%A6%AC-%EA%B5%AC%EC%A1%B0%EB%B6%80%ED%84%B0-%EB%A0%8C%EB%8D%94%EB%A7%81%EA%B9%8C%EC%A7%80-%EC%8B%AC%EC%B8%B5%EB%B6%84%EC%84%9D)

![](https://velog.velcdn.com/images/parkd/post/8932af08-c349-4a46-acc5-bc53b10bd9e9/image.png)

<br/>

웹 개발자는 연차가 쌓이며 자연스례 브라우저 관련 다양한 필수 개념들을 자주 접하며 `DNS lookup`, `https`, `V8엔진`, `렌더링 방식` 등을 학습하게 됩니다.

개념은 알지만 전체 플로우로 엮어 설명할 때 각 개념 간 브릿지를 이어줄 사고관이 없을 수도 있습니다. 해당 아티클에서는 브라우저의 실행부터 화면에 렌더링 되는 과정을 자연스럽게 풀어보겠습니다.

**해당 아티클을 읽으시면 아래와 같은 지식을 얻을 수 있습니다.**

1. 브라우저 프로세스 구조
2. 주소창에 URL을 입력하면 어떻게 될까?
3. 렌더러 프로세스는 어떻게 응답을 받아 처리할까?
4. v8엔진에서 자바스크립트를 해석하는 과정
5. 리플로우 리페인트를 고려해야하는 이유

모든 과정을 읽으신다면 여러분은 '브라우저처럼 생각하는 방법'을 익히실 수 있습니다.

---

## 1. 브라우저 프로세스 구조

크롬 브라우저는 하나의 거대한 프로그램처럼 보이지만, 내부적으로는 여러 프로세스 협업하며 동작합니다. 각 프로세스는 브라우저 기능의 일부를 분담하며, 서로**IPC(프로세스 간 통신)**으로 정보를 주고받습니다.

![브라우저 프로세스](https://velog.velcdn.com/images/parkd/post/ce861723-e307-4517-811f-eb857036f3b8/image.png)

> 출처: [chrome for developers](https://developer.chrome.com/blog/inside-browser-part1?hl=ko#:~:text=There%20are%20even%20more%20processes,much%20CPU%2FMemory%20they%20are%20using)

오늘 아티클에 등장할 프로세스는 **브라우저 프로세스**, **렌더러 프로세스**입니다.

**브라우저 프로세스**
주소 표시줄, 북마크, 뒤로 및 앞으로 버튼을 포함하여 애플리케이션의 'Chrome' 부분을 제어합니다. 또한 네트워크 요청 및 파일 액세스와 같이 웹브라우저의 보이지 않는 권한이 있는 부분을 처리합니다.

**렌더러 프로세스**
웹 사이트가 표시되는 탭 내의 모든 항목을 제어합니다.

위와같은 여러 멀티 프로세스 구조를 통해 브라우저는 안전하고 부드럽게 웹 페이지를 표시합니다. 이러한 구조를 바탕으로 사용자가 주소창에 URL을 입력하고 Enter를 누른 순간부터 화면에 페이지가 렌더링되기까지 내부에서 어떤 일이 일어나는지, **브라우저의 여정**을 떠나보겠습니다.

<br/>

---

## 2. 주소창에 URL을 입력하면 어떻게 될까?

해당 파트에서는 주소창에 `URL`을 입력하면 일어나는 `DNSlooup`, `tcp`, `http/https`의 동작을 순서대로 따라가 보겠습니다.

![브라우저 프로세스](https://velog.velcdn.com/images/parkd/post/6add05d3-1232-4846-bacc-ee299a3b6d65/image.png)

> 출처: [chrome for developers](https://developer.chrome.com/blog/inside-browser-part2?hl=ko)

### URL인지 검색어인지 분기

사용자가 URL을 입력하기 시작하면 **브라우저 프로세스**의 **UI스레드**는 먼저 **입력이 URL인지 검색어인지**를 판별합니다. 현대의 브라우저 주소창은 검색창으로도 활용되기 때문입니다.

**URL 판단**

- http://, https://, ftp:// 등으로 시작
- <www>. 또는 도메인 형식 등

**검색어 판단**

- 스페이스 포함, 특수문자 포함, 내부 북마크, 방문기록에 일치

**판단 불가**

- naver.com -> 도메인 존재 여부 조회
- naver -> 검색어, URl 동시 시도 후 내부 판단

### DNS Lookup

URL이라는 판단이 이루어지면 URL중 도메인 이름을 이용해 대상 IP주소를 알아냅니다. 이 과정을 `DNS Lookup(DNS 조회)`이라고 합니다.

브라우저는 자체 DNS 캐시에 해당 도메인의 IP가 캐시되어 있는지 확인하고 있다면 **재사용**, 없다면 `DNS 리졸루션` 단계를 통해 IP주소를 알아냅니다.

> 출처: [DNS Resolution Process](https://cycle.io/learn/dns-resolution-process)

이제 브라우저는 `DNS` 단계를 통하여 얻은 IP를 통해 해당 서버와 `TCP`연결을 설정합니다.

### TCP

TCP란 두 컴퓨터가 데이터를 정확하게, 순서대로, 안정적으로 주고받도록 해주는 규칙이자 방법입니다.

**TCP 3-Way Handshake**

![](https://velog.velcdn.com/images/parkd/post/236ff63b-7ba8-4a4b-ad51-bca2e56b32bf/image.png)

> 출처 [TCP 3-Way Handshake](https://www.guru99.com/tcp-3-way-handshake.html)

1. SYN (Synchronize)
   클라이언트가 서버에 연결 요청을 위해 SYN 플래그가 설정된 패킷을 보냅니다.

2. SYN-ACK (Synchronize-Acknowledge)
   서버는 클라이언트의 요청을 수락하고, SYN과 ACK 플래그가 설정된 패킷을 클라이언트에 보냅니다.

3. ACK (Acknowledge)
   클라이언트는 서버의 응답을 확인하고, ACK 플래그가 설정된 패킷을 서버에 보냅니다. 이로써 연결이 확립됩니다.

도메인 이름이 HTTP 프로토콜이라면 이제 바로 데이터를 주고 받을 수 있습니다!

하지만 HTTP프로토콜은 큰 단점이 있습니다.

1. 통신 데이터 암호화 부족
   데이터를 암호화하지 않기 때문에 공격자가 네트워크를 도청할 수 있습니다.
   ex) 사용자가 로그인할때 패스워드가 평문으로 전송되면 공격자는 이를 쉽게 가로챌 수 있음
2. 통신 상대의 진위 확인 불가
   상대 서버의 신원을 검증하는 시스템이 없기 때문에 공격자가 요청 URL을 조작하여 사용자를 피싱 사이트로 유도할 수 있음
3. 데이터 무결성 보장 어려움
   데이터가 변조되었는지 확인할 수 있는 방법이 없으므로, 공격자가 중간에서 데이터를 조작해도 이를 감지할 수 없음.

### HTTPS

위의 보안 문제를 해결하기 위해 `TLS`를 활용한 `HTTPS`가 사용됩니다. `HTTPS`는 `HTTP` 통신 전에 `TLS`핸드셰이크 과정을 통해 보안을 강화합니다.

1. 통신 데이터 암호화
   통신 데이터를 암호화여 도청을 방지한다.
2. 통신 상대 검증
   전자 인증서를 사용하여 통신 상대의 신원을 확인한다.
3. 데이터 무결성 보장
   데이터 변조 여부를 확인하는 기능 제공한다.

자세한 내용은 아래 출처를 확인해주세요.

> [카툰으로 보는 HTTPS](https://howhttps.works/ko/why-do-we-need-https/)

이제 드디어 데이터를 주고받을 수 있는 보안 채널까지 완성되었습니다. 이제 브라우저는 목표한 URL로 HTTP 요청 메시지를 전송합니다.

### 응답 데이터 확인

![그림 5: UI 스레드에 렌더러 프로세스를 찾으라고 지시하는 네트워크 스레드](https://velog.velcdn.com/images/parkd/post/2888ce2a-8b20-439a-84e7-b4a7f755af26/image.png)

> 출처: [chrome for developers](https://developer.chrome.com/blog/inside-browser-part2?hl=ko)

응답 본문이 수신되기 시작하면 **네트워크 스레드**는 필요한 경우 스트림의 처음 몇 바이트를 확인합니다. 응답의 `Content-type`이 `text/html` 이거나 `MIME` 스니핑으로 html으로 추론이 되면 **UI스레드**에 데이터가 준비되었다고 알립니다. 이후 **UI스레드**가 **렌더러 프로세스**를 찾아 렌더링을 요청합니다.

<br/>

---

## 3. 렌더러 프로세스는 어떻게 응답을 받아 처리할까?

해당 파트에서는 `HTML파싱`부터 `CSS 적용`, `자바스크립트 실행`, `레이아웃`, `페인트`, `합성` 단계까지, 렌더러의 세부 동작을 순서대로 따라가 보겠습니다.

**렌더러 프로세스**는 탭 내에서 발생하는 모든 작업을 담당합니다. **렌더러 프로세스**에서 **메인 스레드**는 사용자에게 전송하는 대부분의 코드를 처리합니다.
웹 워커 또는 서비스 워커를 사용하는 경우 JavaScript의 일부가 **작업자 스레드**에서 처리되는 경우가 있습니다.

**렌더러 프로세스**의 핵심 작업은 HTML, CSS, JavaScript를 사용자가 상호 작용할 수 있는 웹페이지로 변환하는 것입니다.

### HTML 파싱

![](https://velog.velcdn.com/images/parkd/post/b8444115-4ec1-4c6b-bf4d-66ac418f02d1/image.png)

> 출처: [chrome for developers](https://developer.chrome.com/blog/inside-browser-part3?hl=ko)

렌더러 프로세스의 메인 스레드는 브라우저 프로세스로부터 **HTML 응답 스트림**을 받기 시작하면 즉시 HTML파서를 가동하여 텍스트를 해석해나갑니다.

HTML파싱은 크게 **토큰화**와 **트리구축** 과정으로 이루어집니다. 한 글자씩 문자를 읽어 `<html>`같은 **태그 시작 토큰**을 인식하면 새로운 노드를 만들고, 속성을 읽으면 속성 노드로 붙이는 식입니다. 이렇게 문서를 해석하여 각 요소마다 **DOM노드**를 생성하여 DOM트리를 구성해나갑니다.

DOM은 브라우저 내부에서 문서를 표현하는 자료구조이자 API입니다.

파서가 HTML요소를 만나면 DOM트리에 추가하고, 텍스트 노드를 만나면 리프 노드로 추가하는 식으로 진행합니다.

### CSS 파싱

DOM트리를 구성해나가는 도중에 `<link rel='stylesheet'>`와 같은 CSS관련 태그를 만나 CSS파일을 로드하게 되면, CSS 파서도 별도로 해당 파일을 받아 **CSSSOM** 트리를 구축합니다.

**CSSSOM**은 CSS 규칙들을 트리 형태로 표현한 것인데 **DOM**트리와는 별개로 존재합니다. 나중에 이 둘을 결합하여 **렌더 트리**를 만들게 되는데 이 렌더트리는 화면에 실제로 나타나는 노드만으로 구성된 트리입니다.

### 파서와 리소스 로딩 병렬화

HTML을 파싱하면서 외부 리소스(`<img>`, `<link>`등)을 만나면 **렌더러 프로세스**는 **프리로드 스캐너** 라는 병렬 작업을 통해서 HTML파싱을 막지 않고도 미리 리소스 요청을 보내는 최적화를 진행합니다.

> ex)
> `<img src="">`태그를 파서가 발견하면 즉시 브라우저 프로세스 네트워크 스레드에 이미지 요청을 보내고 계속 파싱을 진행함.

이와 별개로 파서가 진행되다가 `<script>`태그를 만나면 브라우저는 **자바스크립트**실행이 HTML구조를 바꿀 수 있기 때문에 HTML파싱을 일시 중지합니다.

이를 완화하기 위한 방법으로는 `<script>`태그에 `defer`또는 `async` 속성을 주어 파서 블록을 완화하거나 `<body>` 끝 부분에 JS를 넣는 등의 최적화를 합니다.

### v8엔진

![](https://velog.velcdn.com/images/parkd/post/0a4167b1-a36f-408b-a4b3-db4cda6313f7/image.png)

HTML파서가 코드를 분석하는 동안 스크립트 태그를 만나면 네트워크나 캐시에서 파일을 가져오려하는데 이때 파일을 가져오면 **Byte Stream** 상태입니다.

브라우저는 이러한 **Byte Stream** 을 **Byte Stream Decoder**를 통해 다양한 토큰들을 생성합니다.

```javascript
// const a = 1 의 바이트 스트림이 왔다 가정
[
  {
    type: 'Keyword',
    value: 'const',
  },
  {
    type: 'Identifier',
    value: 'a',
  },
  {
    type: 'Punctuator',
    value: '=',
  },
  {
    type: 'Numeric',
    value: '1',
  },
];
```

<br/>

이후 해당 토큰들을 가지고 파서는 AST트리를 구성합니다.

![](https://velog.velcdn.com/images/parkd/post/4f8d3c3b-8fe3-422d-bea3-e24491f25da3/image.png)

<br/>

#### v8 엔진에서의 처리 과정

AST가 생성되면 V8의 경량 인터프리터인 **Ignition**로 전달됩니다. **Ignition**의 가장 큰 임무는 전달받은 AST를 **Byte Code**로 변환하는 것입니다.
**Ignition**은 **Byte Code**를 줄별로 해석하며 실행합니다.

```plaintext
[generated bytecode for function: (0x...)
Parameter count 1
Register count 1

0000000 LdaSmi [1]     -> 누산기(Accumulator)에 상수 1을 로드
0000001 Star0          -> 누산기의 값을 레지스터 r0(a 변수)로 저장
...
```

이때 **자수 실행되는 함수**나 **루프**는 **hot**으로 표시되어 **TurboFan**최적화 컴파일러는 통해 기계어로 **JIT Compile**되어 성능을 끌어 올립니다.

```
JS 코드 실행
     ▼
Ignition 해석 ← 메인 스레드
     ▼
Feedback 수집
     ▼
TurboFan 컴파일러  ← 백그라운드 스레드에서 비동기 실행
      ▼
최적화된 native code
      ▼
다음 실행부터 TurboFan 코드로 실행됨 (= 핫 스팟 최적화)
```

> 심화학습: [How V8 JavaScript Engine Works Behind the Scenes](https://www.deepintodev.com/blog/how-v8-javascript-engine-works-behind-the-scenes)
> <br/>

#### 실행 컨텍스트와 호출 스택

![](https://velog.velcdn.com/images/parkd/post/ae2670c4-0d57-48f4-8737-7d69733d5060/image.png)

JS가 실행되면 **V8은 전역 실행 컨텍스트**를 생성하고, 변수/함수 선언 등을 **메모리 힙**에 할당합니다. 이어 실제 코드 실행이 시작되면, 함수 호출이 발생할 때 마다 새로운 **함수 실행 컨텍스트**가 생성되어 **콜 스택**에 쌓입니다. 함수를 리턴하면, 해당 스택 프레임이 사라지고 이전 컨텍스트로 복귀합니다.

비동기 콜백이나 이벤트 리스너등은 태스크큐에 대기하다가 메인 스레드가 여유로울때 이벤트 루프에 의해 콜스텍으로 들어와 실행합니다. 이때 JS가 너무 오래 메인스레드를 점유하면 UI반응이 느려지는 현상이 발생하기도 합니다.

> [이벤트 루프와 비동기 통신 알아보기](https://dhyun2.tistory.com/2)

<br/>

---

## 4.렌더링 파이프라인

HTML/JS 파싱이 진행됨과 동시에, 브라우저는 받아둔 CSS를 파싱하여 스타일 규칙을 준비합니다.
**DOM트리**가 완성되거나 또는 일부 완성된 시점에, 렌더러의 메인 스레드는 **DOM**각 노드에 어떤 CSS 규칙이 적용되는지 계산합니다. 이를 **스타일 계산**단계 라고 하며, 스타일 규칙은 **선택차 매칭**을 통해 각 요소의 **계산된 스타일**을 결정하며, 흔히보는 개발자도구의 **Computed**탭에서 보이는 것이 바로 각 노드의 최종 계산된 스타일입니다.

이렇게 **DOM + CSSOM**이 준비되면, 렌더러는 화면에 그릴 요소만 모은 **렌더트리**를 구성합니다. 렌더 트리는 DOM트리와 유사한 계층 구조이지만, 오직 실제 렌더링 되는 노드들만 포함하며, 각 노드에 계산된 스타일을 연결하며 렌더트리를 완성합니다.

### Layout

> 예전에는 reflow라고 많이 불리는 개념이였으며, [MDN](https://developer.mozilla.org/ko/docs/Glossary/Reflow)에서도 여전히 Reflow라고 명시되어있지만 Chrome DevTools등을 보면 렌더링 파이프라인으로 Layout이라 지칭하고 있습니다.

![](https://velog.velcdn.com/images/parkd/post/0c7a90f9-1561-4bfc-9a66-be3f0354d774/image.png)

> 출처: [chrome for developers](https://developer.chrome.com/blog/inside-browser-part3?hl=ko)

각 좌표와 사이즈를 결정하는 과정입니다. 메인 스레드는 렌더 트리의 루트를 시작으로 자식들을 순차적으로 방문하며, 위치를 할당합니다.
레이아웃이 끝나면, 각 렌더 노드는 자신이 그려질 **박스**영역과 위치를 갖게 됩니다.

### Paint

![](https://velog.velcdn.com/images/parkd/post/68e32ade-2066-423c-afa7-142cfb9089c1/image.png)

> 출처: [chrome for developers](https://developer.chrome.com/blog/inside-browser-part3?hl=ko)

이제 DOM구조, 스타일, 레이아웃 정보가 모두 갖춰졌으므로, 어떤 순서로 무엇을 그릴지 결정하는 단계입니다(z-index등). 이를 페인팅 단계라고 부릅니다.

</br>

지금까지의 **렌더링 파이프라인**을 정리하면 DOM 생성 -> 스타일 계산 -> 레이아웃 -> 페인트로 이어졌고, 각 단계는 이전 단계 결과를 활용하여 다음 데이터 구조를 만들어냈습니다.
이렇듯 렌더링은 여러 단계가 연속적으로 연결되어 있어, 중간의 변화가 후속단계를 연쇄적으로 유발합니다.

### 합성과 실제 화면 표시

지금까지의 렌더링 파이프라인을 정리하면, **DOM 생성 -> 스타일 계산 -> 레이아웃 -> 페인트**로 이어졌고, 각 단계는 이전 단계 결과를 활용하여 다음 데이터 구조를 만들어냈습니다
만약 중간에 **DOM 구조**나 **스타일**이 바뀌어 **레이아웃 트리**가 수정되면, 해당 부분 이후 단계(페인트 등)를 다시 처리해야 합니다

이렇듯 렌더링은 여러 단계가 연속적으로 연결되어있어 중간의 변화가 후속 단계를 연쇄적으로 유발합니다.

최신 브라우저는 일부 단계를 메인 스레드와 독립적으로 처리하여 성능을 향상시킬 수 있습니다.

#### 합성 단계

합성은 여러 그래픽 레이어를 별도로 그린뒤, 나중에 GPU에서 겹쳐 **합성**하는 기법입니다. 예를들어 `transform` 이나 `opacity`속성 변경은 **합성**만으로 처리가 가능한 애니메이션으로, 해당 요소를 독립된 레이어로 만들면 메인 스레드가 다시 레이아웃/페인트 하지 않아도 **GPU 합성**만으로 부드럽게 애니메이션 할 수 있습니다.

합성 단계에서는 앞선 페인트 레코드들을 참고하여 **레이어별 그림 타일**을 생성하고 레스터 작업을 수행합니다. 크기가 큰 레이어는 GPU메모리에 올리기 위해 작은 사각 타일들로 분할되어 처리됩니다. **레스터 스레드**들은 이러한 타일들을 병렬로 **비트맵 래스터라이즈**하여 완성된 비트맵을 **GPU메모리**에 업로드 합니다. 이렇게 준비된 여러 레이어의 타일들을 **합성(컴포지터) 스레드**는 하나의 화면 프레임으로 조립합니다.

각 타일의 **Draw Quad**들을 모아 **컴포지터 프레임**을 만들고, 이 정보를 브라우저 프로세스의 **GPU 프로세스**로 전달합니다. 최종적으로 **GPU프로세스**는 컴포지터 프레임들을 받아 모니터 화면에 출력합니다.

### 정리

이제까지 **HTML 파싱**부터 시작해 **DOM/CSSOM 생성**, **Render Tree 구성**, **Layout 계산**, **Paint 처리**, 그리고 **Composite 단계**까지 이어지는 전체 렌더링 파이프라인을 따라, 렌더러 프로세스가 실제 화면을 그리기 위해 수행하는 핵심 단계들을 살펴보았습니다.

<br/>

---

## 5. 마무리

긴 글을 끝까지 읽어주셔서 감사합니다.
브라우저의 시작부터 렌더링까지의 과정을 깊이 있게 이해하는 데 도움이 되었기를 바랍니다.

감사합니다.

**References**

- <https://www.linkedin.com/pulse/understanding-inner-workings-v8-engine-parsing-ast-ignition-tiwary-gzcic/>
- <https://www.deepintodev.com/blog/how-v8-javascript-engine-works-behind-the-scenes>
- <https://v8.dev/blog>
