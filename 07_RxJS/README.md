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

## 2. Observable

### lazy push collection of multiple values

```js
import { Observable } from "rxjs";

const observable = new Observable((subscriber) => {
  // 즉시 값을 밀어넣음
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);

  setTimeout(() => {
    subscriber.next(4);
    // 완결됨
    subscriber.complete();
  }, 1000);
});
```

- 옵저버의 값을 사용하려면 옵저버블을 구독하면 된다

```js
import { Observable } from "rxjs";

const observable = new Observable((subscriber) => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  setTimeout(() => {
    subscriber.next(4);
    subscriber.complete();
  }, 1000);
});

console.log("just before subscribe");

// 구독 => 이러니까 얘는 이터레이터같네..
// observable에서 순차적으로 벌어지는 일을 관찰하고 값을 사용할 수 있음
observable.subscribe({
  next(x) {
    console.log("got value " + x);
  },
  error(err) {
    console.error("something wrong occurred: " + err);
  },
  complete() {
    console.log("done");
  },
});
console.log("just after subscribe");

/*
just before subscribe
got value 1
got value 2
got value 3
just after subscribe
got value 4
done
*/
```

### pull, push

- data producer과 consumer와 소통을 어떻게 하는지 설명해주는 용어
- pull : 소비자가 데이터를 프로듀서로부터 언제 받을지 결정한다. 프로듀서는 컨슈머에게 언제 데이터가 전달될지 신경을 안 쓴다
  - 모든 자바스크립트 함수 : 함수는 데이터의 프로듀서이고, 함수는 '호출' 되는 방식으로 호출을 하는 주체, 즉 소비자가 데이터를 언제 받을지 결정한다.
  - 제네레이터, 이터레이터 : 호출을 하는 쪽에서 next를 호출하여 값을 받는다.
- push : 프로듀서가 컨슈머에게 언제 데이터를 넘길지 결정한다.
  - Promise : 프로미스는 프로듀서가 완결시켜야 값을 받을 수 있다
  - Observable : Observable에게 데이터를 푸쉬하고, 온전히 observable이 이를 언제 넘길지 결정하게 된다.

### 옵저버블 vs 일반 함수와의 차이

```js
// 같은 동작 다른 구현

// 함수
function foo() {
  console.log("Hello");
  return 42;
}

// 호출을 해와서 데이터를 불러와서 쓰기
const x = foo.call(); // same as foo()
console.log(x);
const y = foo.call(); // same as foo()
console.log(y);

// 옵저버블
import { Observable } from "rxjs";

const foo = new Observable((subscriber) => {
  console.log("Hello");
  subscriber.next(42);
});

// 구독을 해서 동작을 이용하기
foo.subscribe((x) => {
  console.log(x);
});

foo.subscribe((y) => {
  console.log(y);
});
```

- lazy computation : 호출 혹은 구독을 하기 전까지 아무일도 안일어난다.
- 함수를 호출하는 것은 옵저버블을 구독하는 것과 비슷한 느낌
- 비동기로 동작하지 않는다. => 앞뒤의 동기 로직들과 어우러져 순차적으로 실행되고 block되지 않음
- 옵저버블은 **multiple value를 return할 수 있다**

```js
// function
function foo() {
  console.log("Hello");
  return 42;
  return 100; // dead code. will never happen
}

// observable
import { Observable } from "rxjs";

const foo = new Observable((subscriber) => {
  console.log("Hello");
  subscriber.next(42);
  subscriber.next(100); // "return" another value
  subscriber.next(200); // "return" yet another
});

console.log("before");
foo.subscribe((x) => {
  console.log(x);
});
console.log("after");

// 비동기 리턴도 가능하다 => 엥 일반 함수 리턴문을 setTimeout으로 감싸면 어캐되지
// 지금 느낌만 봐서는 뭔가.. 제어할 수 없는 이터레이터같다..
import { Observable } from "rxjs";

const foo = new Observable((subscriber) => {
  console.log("Hello");
  subscriber.next(42);
  subscriber.next(100);
  subscriber.next(200);
  setTimeout(() => {
    subscriber.next(300); // happens asynchronously
  }, 1000);
});

console.log("before");
foo.subscribe((x) => {
  console.log(x);
});
console.log("after");
```

## 옵저버블을 둘러싼 동작들

### 생성

- Observable 생성자는 subscribe 함수 하나를 인자로 받는다.
- 이렇게 생성자로 생성할 수 있지만 뒤에서 배울 다른 연산자를 통해 생성되기도 한다.

```js
// hi를 매 초마다 subscriber에게 전달하는 코드

import { Observable } from "rxjs";

const observable = new Observable(function subscribe(subscriber) {
  const id = setInterval(() => {
    subscriber.next("hi");
  }, 1000);
});
```

### 구독/실행

- 데이터를 전달할 콜백을 제공해 함수를 호출하는 것
- 옵저버블 내부에서
  - Next : 옵저버블 밖으로 값을 보낸다
  - Error : 에러나 예외를 보낸다
  - complete: 옵저버블의 끝을 알림

```js
import { Observable } from "rxjs";

const observable = new Observable(function subscribe(subscriber) {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  subscriber.complete();
  subscriber.next(4); // Is not delivered because it would violate the contract
});

// 이런식으로 예외처리도 가능
const observable = new Observable(function subscribe(subscriber) {
  try {
    subscriber.next(1);
    subscriber.next(2);
    subscriber.next(3);
    subscriber.complete();
  } catch (err) {
    subscriber.error(err); // delivers an error if it caught one
  }
});
```

### 구독 취소

- subscribe 메서드가 리턴하는 값인 subscription은 실행 취소를 위한 API를 가지고 있음

```js
mport { from } from 'rxjs';

const observable = from([10, 20, 30]);
const subscription = observable.subscribe(console.log);

subscription.unsubscribe();
```
