# RxJS

빨리 써먹기 위해 독스를 정리해 봅니다

## 1. 오버뷰

### 정의

- observable sequences를 사용해 비동기와 이벤트 기반 프로그램을 합성하기 위한 라이브러리
- 이벤트에 사용하는 lodash 처럼 생각해라
- 순차적 이벤트를 Managing하기 위해 옵저버 패턴, 이터레이터 패턴, 함수형 프로그래밍 개념을 결합한다.
- 기본적인 개념
  - Observable : 호출 가능한 미래 값들과 이벤트 모임
  - Observer : Observable이 전달해주는 값을 어떻게 처리할지에 대한 콜백
  - Subscription : Observable의 수행, 기본적으로 실행 취소에 유용
  - Operators : map, filter, concat, reduce와 같은 연산으로 대상을 처리하는 함수형 프로그래밍 스타일을 가능하게 하는 순수함수들
  - Subject : EventEmitter. 값이나 이벤트를 여러 옵저버들에게 동시 전달해주는 유일한 방법
  - Schedulers : 중앙 집중된 동시성 컨트롤을 위한 dispatcher

```js
// 보통
document.addEventListener("click", () => console.log("Clicked!"));

// RxJS
import { fromEvent } from "rxjs";

fromEvent(document, "click").subscribe(() => console.log("Clicked!"));
```

### 순수함

순수함수가 RxJS의 파워풀한 점. 에러에 덜 취약하다. 일반적으로 불순한 로직도 순수한 로직으로

```js
// 상태오염 => count를 직접 수정한다
let count = 0;
document.addEventListener("click", () =>
  console.log(`Clicked ${++count} times`)
);

// 상태 격리 => 메소드에서 변수의 순수성 유지?
// 초기값과 콜백함수를 메소드에 넣어서 제공
import { fromEvent } from "rxjs";
import { scan } from "rxjs/operators";

fromEvent(document, "click")
  .pipe(scan((count) => count + 1, 0))
  .subscribe((count) => console.log(`Clicked ${count} times`));
```

### 흐름

이벤트들의 흐름을 Observable을 통해서 컨트롤할 수 있게 도와주는 연산자를 가지고 있음

```js
// 일반 자바스크립트
let count = 0;
let rate = 1000;
let lastClick = Date().now() - rate;

document.addEventListener("click", () => {
  if (Date.now() - lastClick >= 1000) {
    console.log(`Clicked ${++count} times`);
    lastClick = Date.now();
  }
});

import { fromEvent } from "rxjs";
import { throttleTime, scan } from "rxjs/operators";

fromEvent(document, "click")
  .pipe(
    throttleTime(1000),
    scan((count) => count + 1, 0)
  )
  .subscribe((count) => console.log(`Clicked ${count} times`));
```

### 값

observable에게 넘겨줌으로써 값을 변경할 수 있음

```js
// Plain
let count = 0;
const rate = 1000;
let lastClick = Date.now() - rate;
document.addEventListener("click", (event) => {
  if (Date.now() - lastClick >= rate) {
    count += event.clientX;
    console.log(count);
    lastClick = Date.now();
  }
});

// rxjs => 좀더 함수형스럽고 간결하다. 스코프 바깥에 변수를 선언할 필요가 없는듯
import { fromEvent } from "rxjs";
import { throttleTime, map, scan } from "rxjs/operators";

fromEvent(document, "click")
  .pipe(
    throttleTime(1000),
    map((event) => event.clientX),
    scan((count, clientX) => count + clientX, 0)
  )
  .subscribe((count) => console.log(count));
```
