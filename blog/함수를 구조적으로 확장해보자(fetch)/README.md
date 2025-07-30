[블로그 바로가기](https://velog.io/@parkd/%ED%95%A8%EC%88%98%EB%A5%BC-%EA%B5%AC%EC%A1%B0%EC%A0%81%EC%9C%BC%EB%A1%9C-%ED%99%95%EC%9E%A5%ED%95%B4%EB%B3%B4%EC%9E%90fetch)

![](https://velog.velcdn.com/images/parkd/post/2c6d6e03-dc82-42cb-8b5b-6a3a9062e240/image.png)

<br/>

Next.js의 patchFetch 메커니즘을 분석하면서, 저수준의 함수 확장에서 고수준의 추상화 설계로 진화하는 과정을 학습하게 되었습니다. 이 과정속 얻은 지식들을 정리하여, 게임과 같은 플로우로 설명드리겠습니다.

해당 아티클을 읽으시면 아래와 같은 지식을 얻을 수 있습니다.

- 함수를 몽키패치 해보기
- 함수를 래핑 해보기
- 어댑터 패턴 도입 해보기

> ⚠️ 주의:&nbsp 본 아티클의 코드는 학습 및 예시 목적의 저수준 구현으로, 실제 서비스 환경에서는 권장되지 않습니다.

---

## 1. 이슈 발생

![](https://velog.velcdn.com/images/parkd/post/fd899510-f9ba-4a72-976e-a4057c33a212/image.png)

> 나는 신입오리다. 현재 ui를 데이터가 받아오기 전까지는 무조건 로딩바를 띄웠지만, 응답 속도가 빠른경우 이는 UX를 저해하는 경험을 주고있따. 이를 개선하는 미션을 받으니 이를 해결해보자.

---

## 2. 자료 수집

우선 신입오리는 해당 이슈를 해결하기위해 `응답이 빠른경우는 어떤 기준일까` 와 `어떻게 응답속도를 비교할까`를 고민해보며 아래와 같은 재료들을 손에 넣었습니다.

### delay(100)

닐슨의 UX 반응 시간 기준을 따르자면 인간의 표준 인내 임계정은 100ms입니다. 100ms를 기준으로 로딩바를 띄우고, 띄우지 않을것입니다.

> [Response Times: The 3 Important Limits](https://www.nngroup.com/articles/response-times-3-important-limits/)

이를 근거로 인자로받은 time이 지나면 value값을 반환하는 비동기 함수를 구현하였습니다.

```typescript
const delay<T> = (time: number, value: T): Promise<T> => {
  return new Promise((resolve) => setTimeout(resolve, time, value));
}
```

</br>

### promise.race()

이제 실제 요청할 프로미스와 delay를 병렬로 실행하여, 누가 더 빠르게 도착하는지 확인할 메서드가 필요합니다. 이를위해 promise.race()를 사용하였습니다

> [promise.race()](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Promise/race)

이를 활용하여, 실제 요청할 프로미스와 delay중 누가 더 빨리 도착하는지 확인할 수 있습니다.

```javascript
const fetchPromise = originalFetch(...args);
const delayPromise = delay(100, '__DELAY__');

const winner = await Promise.race([fetchPromise, delayPromise]);
```

<br/>

이제 모든 재료가 준비되었고, 실제 fetch를 확장하여 보겠습니다.

---

## 3.구현

### 1. Monkey Patch

> 몽키패치란? -> 런타임 중에 기존 객체나 함수의 동작을 바꿔버리는 것을 의미

Javascript에서는 `window.fetch`처럼 전역 객체에 정의된 함수도 자유롭게 덮어 쓸 수 있습니다. 이 특성을 이용해 신입오리는 fetch를 확장하여 `monkey patch`를 시도하였습니다.

**구현**

```javascript
//window.fetch 몽키패치
window.fetch = async (...args) => {
  const fetchPromise = originalFetch(...args);
  const delayPromise = delay(100, '__DELAY__');

  const winner = await Promise.race([fetchPromise, delayPromise]);

  if (winner === '__DELAY__') {
    return { response: fetchPromise, slow: true };
  }

  return winner === '__DELAY__'
    ? { response: fetchPromise, slow: true }
    : { response: Promise.resolve(winner), slow: false };
};

//사용 예시

const getUser = async () => {
  const { response, slow } = await fetch('/api/user');

  let data = null;

  if (slow) {
    toggleLoadingIndicator(true);
    // response는 Promise<Response> 상태
    const res = await response;
    data = await res.json();
    ...
  } else {
    // response는 Response 객체
    data = await response;
    ...
  }
}

```

위 코드는 기존의 `fetch api`를 몽키패치하여 응답의 속도에따라 분기하여, 응답 객체를 `{response, slow}` 형태로 확장한 구조입니다.

<br/>

#### 코드 리뷰

이제 신입오리는 미션을 완수하고 사수오리에게 코드리뷰를 신청하였습니다.

![](https://velog.velcdn.com/images/parkd/post/d78d7455-8714-45c0-86e4-ff4798002bd2/image.png)

1. 전역 오염 위험성
   window.fetch를 직접 덮어쓰는것은 전역 환경을 오염시키며, 라이브러리나, 기존 코드 등 모든 fetch호출이 영향을 받아 모든 코드가 깨질 위험히 있어

2. 타입시스템과의 충돌
   ts는 fetch():Promise<Response>를 기대하기 때문에 컴파일 에러가 발생할꺼야.

3. 테스트와 디버깅 복잡성 증가
   에러 스택트레이스가 originalFetch가 아닌 patched fetch 기준으로 나오기 때문에 문제 원인을 추적하기 어려워질 수 있어.

![](https://velog.velcdn.com/images/parkd/post/9efa3c5f-de5a-49fd-9718-16bf31ba9dbe/image.png)

<br/>

---

### 2. wrapper 함수 구현

신입 오리는 사수가 지적한 다음의 문제점들을 마주하게 됩니다:

1. 기존의 fetch를 몽키패치하면 글로벌하게 오염된다.

- 모든 라이브러리, 외부 코드, 테스트 코드 등에서 예상하지 못한 동작을 유발할 수 있다.
- 디버깅이 어렵고, 유지보수 부담이 커진다.

  2.fetch가 반환하는 response의 타입이 상황에 따라 달라진다.

- 느린 응답이면 Promise<Response>, 빠르면 Response 객체.
- 사용하는 쪽에서 이를 구분하고 처리해야 해 실수 가능성이 크고, 코드도 지저분해진다.

신입 오리는 고민 끝에 이렇게 결심합니다:

"기존 fetch는 그대로 두고, 새로운 요구사항은 fetchWithDelay라는 래퍼 함수로 분리하자."

<br/>

#### 2.1 구현

```typescript
export const fetchWithDelay = async (...args: Parameters<typeof fetch>) => {
  const fetchPromise = fetch(...args);
  const delayPromise = new Promise<'__DELAY__'>((resolve) =>
    setTimeout(() => resolve('__DELAY__'), 100)
  );

  const winner = await Promise.race([fetchPromise, delayPromise]);

  return winner === '__DELAY__'
    ? { response: fetchPromise, slow: true }
    : { response: Promise.resolve(winner), slow: false };
}

//사용 예시
import { fetchWithDelay } from './fetchWithDelay';

const getUser = async () => {
  const { response, slow } = await fetchWithDelay('/api/user');

  if (slow) {
    toggleLoadingIndicator(true);
    // response는 Promise<Response> 상태
    const res = await response;
    ...
  } else {
    // response는 Response 객체
    data = await response;
    ...
  }
};

```

fetchWithDelay는 기존 fetch를 오염시키지 않고 안전하게 감싸는 방식으로, 타입 안정성과 호출부의 명확성을 높여줍니다.

<br/>

#### 2.2 코드 리뷰

신입오리는 리팩토링된 코드를 사수오리에게 코드리뷰를 신청하였습니다.

![](https://velog.velcdn.com/images/parkd/post/dd8c5a3d-2e24-42f5-a02a-a545fbe15cc2/image.png)

1. 관심사 분리 실패
   `fetch` 요청, 로딩 판단, fallback 처리 등 서로 다른 관심사가 하나의 함수에 혼합되어 있어 유지보수 및 변경이 어려워. 즉 코드의 응집도는 낮고 결합도가 높은 상태야
   SRP를 위배했다고도 볼 수 있어.

2. 재사용 어려움
   타임아웃 시간, 정책을 변경하려면 래퍼 함수 자체를 수정하거나 또 다른 버전을 만들어야해서 확장성이 부족해

3. 디버깅 복잡성
   try/catch 또는 네트워크 오류 처리 시 실제 원인이 래퍼 내부에서 추상화되므로 에러 위치가 불명확해질 수 있어. 특히 조건 분기에 따라 Promise와 Response가 혼재되면 디버깅 난이도 증가하고있어.

![](https://velog.velcdn.com/images/parkd/post/cb4f917a-c25d-4e6f-8dee-bdb1dc546e7c/image.png)

<br/>

---

### 3. 어댑터 패턴을 활용한 wrapper 함수 구현

이전 코드는 기능 자체는 잘 작동했지만, 사수 오리의 피드백처럼 관심사 분리, 확장성, 재사용성 측면에서 개선 여지가 많았습니다. 이에 신입 오리는 Next.js의 patchFetch 구조를 참고하면서 전략 패턴과 어댑터 패턴을 학습하게 되었습니다.

이제 신입 오리는 강해졌습니다. 학습한 내용을 바탕으로 fetch 래퍼를 리팩토링해보겠습니다.

#### 3.1 기존 코드의 문제점 분석

```ts
export const fetchWithDelay = async (...args: Parameters<typeof fetch>) => {
  const fetchPromise = fetch(...args);
  const delayPromise = new Promise<'__DELAY__'>(resolve => setTimeout(() => resolve('__DELAY__'), 100));

  const winner = await Promise.race([fetchPromise, delayPromise]);

  return winner === '__DELAY__' ? { response: fetchPromise, slow: true } : { response: Promise.resolve(winner), slow: false };
};
```

문제점 요약:

1. fetch와 delay 로직이 강하게 결합되어 있어 재사용성과 확장성이 떨어짐
2. 다른 전략(로깅 등)을 주입하기 어려운 구조
3. 모든 요청에 대해 동일한 로직을 적용해야 하므로 유연성이 없음
4. 테스트하기 어렵고, 단위 테스트 작성이 힘듬

이를 개선하기위해 신입 오리는 strategy패턴과 어댑터 패턴을 조합하여 리팩토링을 진행하였습니다.

#### 3.2 리팩토링

**관심사 분리: 전략 패턴을 이용한 strategy 추출**

```typescript
export type FetchStrategy = <T = any>(fetchPromise: Promise<T>, signal: AbortSignal) => Promise<T>;

export const fetchImpl = async (fetchFn: typeof fetch, input: RequestInfo, init?: RequestInit, strategy: FetchStrategy = defaultTimeoutStrategy) => {
  const controller = new AbortController();
  const signal = controller.signal;

  const fetchPromise = fetchFn(input, { ...init, signal });
  const winner = await strategy(fetchPromise, signal);

  return winner;
};
```

- fetchImpl은 실제 요청을 수행하고 전략을 적용하는 순수 로직만을 담당합니다.
- 다양한 전략을 FetchStrategy 형태로 주입할 수 있어 유연하고 테스트하기 쉬운 구조입니다.

<br/>

**전략 구현 (Strategy Pattern)**

```typescript
export class SlowRequestError extends Error {
  constructor() {
    super('Request was slow');
    this.name = 'SlowRequestError';
  }
}

export class LoggableRequestError extends Error {
  constructor(public payload: any) {
    super('Request needs logging');
    this.name = 'LoggableRequestError';
  }
}

export const slowStrategy: FetchStrategy = async (fetchPromise, signal) => {
  return Promise.race([
    fetchPromise,
    new Promise<never>((_, reject) => {
      const id = setTimeout(() => reject(new SlowRequestError()), 100);
      signal.addEventListener('abort', () => clearTimeout(id));
    }),
  ]);
};

export const loggingStrategy: FetchStrategy = async fetchPromise => {
  try {
    const result = await fetchPromise;
    if (!result.ok) {
      throw new LoggableRequestError({ status: result.status, url: result.url });
    }
    return result;
  } catch (e) {
    if (!(e instanceof Error)) throw e;
    throw new LoggableRequestError({ error: e });
  }
};
```

- slowStrategy: 느린 요청을 감지하여 SlowRequestError로 처리합니다.
- loggingStrategy: 실패 응답이나 예외를 LoggableRequestError로 감싸서 로그 수집에 활용합니다.

이처럼 다양한 전략을 조합하거나 교체할 수 있어 확장성과 재사용성이 뛰어납니다.

<br/>

**어댑터 패턴을 적용한 fetchClient 생성기**

```typescript
type FetchClientOptions = {
  fetchFn?: typeof fetch;
  strategy?: FetchStrategy;
  onAction?: (payload: any) => void;
};

export const createFetchClient = ({ fetchFn = fetch, strategy, onAction }: FetchClientOptions) => {
  return async (input: RequestInfo, init?: RequestInit) => {
    try {
      const response = await fetchImpl(fetchFn, input, init, strategy);
      return { response };
    } catch (e) {
      if (e instanceof SlowRequestError) {
        onAction?.();
        const res = fetchFn(input, init);
        return { response: res, slow: true };
      }

      if (e instanceof LoggableRequestError) {
        onAction?.(e.payload);
        throw e;
      }

      throw e;
    }
  };
};
```

- createFetchClient는 fetchImpl을 감싸서, 호출자의 환경에 맞는 후처리를 수행합니다.
- 여기서 어댑터 패턴은 전략에 따라 예외를 처리하거나 콜백(onAction)을 수행하는 부분에 해당합니다.

<br/>

#### 3.3 사용 예시

```typescript
const fetchClient = createFetchClient({
  strategy: slowStrategy,
  onAction: () => toggleLoadingIndicator(true),
});

const getUser = async () => {
  const { response, slow } = await fetchClient('/api/user');
  if (slow) toggleLoadingIndicator(false);

  const res = slow ? await response : response;
  return await res.json();
};
```

자 모든 코드가 완성되었습니다! 이제 마지막 코드리뷰를 받아보겠습니다.

#### 3.4 코드 리뷰

![](https://velog.velcdn.com/images/parkd/post/02c4f929-872f-4cfc-8c25-cd29ee78cf7f/image.png)

---

## 후기

![설명 문구](https://velog.velcdn.com/images/parkd/post/dd157fe2-40fd-460f-93ac-df8d8d6f8488/image.png)

읽어주셔서 감사합니다.
최근 진행중인 `Next.js 톺아보기` 스터디에서 fetch를 cache하는 로직을 살펴보며 설계부분에서 감명을 받고 있었는데 마침 `멀티패러다임 프로그래밍` 교재에서 `Promise.race` 를 활용해 fetch 응답 시간을 기준으로 분기하는 로직을 접하게 되었습니다.
이 두 가지를 조합해서 글을 써보면 재미있겠다 싶어 시작했는데… 쓰다 보니 살짝 억지가 섞인 글이 되어버린 것 같습니다..

그렇게 탄생한 것이 바로 `오리`입니다.
귀엽지 않나요? 그러면 됐습니다! 크하핫

아무튼, 읽어주셔서 정말 감사합니다!
