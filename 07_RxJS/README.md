# RXJS

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

구독을 당하는 친구

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
import { from } from "rxjs";

const observable = from([10, 20, 30]);
const subscription = observable.subscribe((x) => console.log(x));

subscription.unsubscribe();
```

## 3. Observer

구독을 하는 얘

### 설명

- Observable이 제공하는 값을 소비하는 친구. 콜백의 나열로 이루어진 객체이며, 각각의 프로퍼티들은 옵저버블이 제공한 값에 대해 notification을 표시함

```js
const observer = {
  next: (x) => console.log("Observer got a next value: " + x),
  error: (err) => console.error("Observer got an error: " + err),
  complete: () => console.log("Observer got a complete notification"),
};

// 이 객체를 subscribe 메소드의 인자로 넣어준다
observable.subscribe(observer);
```

- observer에 콜백을 태우지 않는다면 observable의 호출은 일반적으로 진행되지만, corresponding callback이 없을 경우 특정 notification이 무시될 수 있다. => 콜백을 안쓰면 특정 동작에서 걍 넘어간다

```js
// 컴플릿 콜백이 없는 경우
const observer = {
  next: (x) => console.log("Observer got a next value: " + x),
  error: (err) => console.error("Observer got an error: " + err),
};
```

- 그냥 인자로 콜백을 하나만 넣으면 그건 next다

```js
observable.subscribe((x) => console.log("Observer got a next value: " + x));
```

## 3. 각종 Operator

- 오퍼레이터들은 함수다. 크게 두가지 종류가 있음
  - Pipeable : pipe(operator())의 형태로 사용할 수 있는 것들. 순수함수처럼 옵저버블 인스턴스를 수정하지 않는다. 대신에 새로운 옵저버블을 리턴하고, 첫번째 옵저버블을 구독하는데 썼던 subscription logic을 그대로 유지한다.
  - Creation : Pipe를 이용하지 않고 단독으로 사용되며 새로운 옵저버블을 만들 수 있는 함수들이다. of(1,2,3)은 1,2,3을 순차적으로 뿜는 옵저버블을 만든다

```js
import { of } from "rxjs";
import { map } from "rxjs/operators";

// 옵저버블 생성
of(1, 2, 3)
  // 뭔가 구독 로직으로 이어지기 전에 필터링을 하는 친구인거같다. 파이프라이닝?
  .pipe(map((x) => x * x))
  .subscribe((v) => console.log(`value: ${v}`));

// Logs:
// value: 1
// value: 4
// value: 9
```

```js
import { of } from "rxjs";
import { first } from "rxjs/operators";

of(1, 2, 3)
  .pipe(first())
  .subscribe((v) => console.log(`value: ${v}`));

// Logs:
// value: 1
```

### piping

파이핑 연산자들은 여러개를 동시에 사용할 수 있다.

```js
obs.pipe(op1(), op2(), op3(), op4());
```

### creation operator

- 옵저버블을 만들어낸다. 걍 new 연산자로 만들어내고 내부에 로직을 작성하는 것과는 달리 미리 predetermine된 로직을 사용해서 옵저버블을 만들 수 있다.

```js
import { interval } from "rxjs";

const observable = interval(1000 /* number of milliseconds */);
```

### 고차 옵저버블

- pipe에 새로운 옵저버블을 리턴하는 연산자를 사용하는 경우

```js
const fileObservable = urlObservable.pipe(map((url) => http.get(url)));

// 파이프 옵저버블을 일반 옵저버블로 바꾸기
const fileObservable = urlObservable.pipe(
  map((url) => http.get(url)),
  concatAll()
);
```

- 여기서 concatAll()은 바깥 옵저버블에서 나오는 각각의 내부 옵저버블을 구독하여 복사하여 옵저버블이 완결될때까지 emit한다.

### 마블 다이어그램

많은 연산자들은 시간에 의존하고, 모든 로직이 동시적으로 순차적으로 진행되거나 처리되지 않을 수 있다. 옵저버블의 양태를 시계열에 따른 직선 다이어그램으로 표현한 그림이 마블 다이어그램이다.

https://rxmarbles.com/#interval

## 4. 주요 creation 연산자

얘네들은 rxjs에서 export 된다.  
스케쥴러 함수를 마지막 인자로 넣어줄 수 있다.

### from

옵저버블로 변환이 가능한 객체를 옵저버블로 바꿔주는 오퍼레이터. Observable, Array, Promise, Iterable, String 타입을 옵저버블로 바꿔준다.

```js
import { from } from "rxjs";

from([1, 2, 3, 4]).subscribe({
  next: console.log,
  complete: () => console.log("complete"),
});

// 1
// 2
// 3
// 4
// complete

// 이터레이터 함수 사용
function* generateDoubles(seed: number) {
  let i = seed;
  while (true) {
    yield i;
    i = 2 * i;
  }
}

const iterator = generateDoubles(3);
const observable = from(iterator).pipe(take(10));

observable.subscribe(console.log);

// Logs:
// 3
// 6
// 12
// 24
// 48
// 96
// 192
// 384
// 768
// 1536
```

### fromEvent

EventEmitter의 객체, 또는 브라우저 이벤트를 Observable로 바꿔야 하는 경우 사용. 특정 이벤트를 내보내는 옵저버블을 만든다 => 이거 확실한 용례를 가져와야할듯

```js
import { fromEvent } from "rxjs";

const clicks = fromEvent(document, "click");
clicks.subscribe((x) => console.log(x)); // 클릭 할때마다 이벤트 객체 리턴
```

```js
// React 에서의 예제 => 근데 그냥... 여기서는 리스너 붙이는게 낫겟다 근데
// svelte에서는 어떻게 활용할 수 있을지 생각해보기

const InputBox = () => {
  const buttonEl = React.useRef(null);

  useEffect(() => {
    const clicky = fromEvent(buttonEl.current, "click").subscribe((clickety) =>
      console.log({ clickety })
    );

    return () => clicky.unsubscribe();
  }, []);

  return (
    <button ref={buttonEl} type="button">
      Click Me
    </button>
  );
};
```

### of

나열된 인자 값들을 순서대로 발행하는 옵저버블 만들어줌. new로 만드는것보다 간단

```js
import { of } from "rxjs";

const complete = () => console.log("complete");

of(1, 2, "a", "b", 3, 4, ["string", 10]).subscribe({
  next: console.log,
  complete,
});

// 1
// 2
// a
// b
// 3
// 4
// [ 'string', 10 ]
// complete
```

### range

일정 범위에 안에 있는 숫자 값을 next로 발행하는 Observable을 만든. 반복문을 실행하는 것과 같음

```js
import { range } from "rxjs";
// 부터 몇개 => range(start, count)
range(1, 5).subscribe(console.log);
// 1
// 2
// 3
// 4
// 5

range(2, 5).subscribe(console.log);
// 2
// 3
// 4
// 5
// 6

range(5, 1).subscribe(console.log);
// 5
```

### interval

ms 단위의 값을 인자 값으로 넣으면 그 텀마다 음이 아닌 정수를 차례대로 반환하는 함수 setInterval

```jsx
import { interval } from "rxjs";

// 옵저버블 생성
interval(1000).subscribe(console.log); // 1초마다 1씩 늘어난 값을 내보냄
```

### timer

지정한 시간이 지난 다음 값을 한개 내보내는 함수. 두번째 값으로 그 다음 값에 대한 주기를 줄 수 있음. setTimeOut

```js
import { timer } from "rxjs";

timer(1000).subscribe(console.log);
// 0

timer(1000, 500).subscribe(console.log);
// 0
// 1
// 2 ...
```

### throwError

값을 발행하다가 특정 에러를 발생시키고 종료해야하는 상황에 사용할 수 있는 함수

```js
import { throwError } from "rxjs";

throwError(new Error("error")).subscribe({
  next: console.log,
  error: (err) => console.error(`error: ${err.message}`),
  complete: () => console.log("complete"),
});

// error: error
```

## 5. scheduler

A Scheduler lets you define in what execution context will an Observable deliver notifications to its Observer.

- 언제 구독이 시작되고 notification이 언제 딜리버리 되는지 결정하는 옵저버블의 내부 동작 로직
- 자료구조다 : 우선순위에 따라 어떤 로직을 실행하고 저장할지를 결정
- 실행 컨텍스트다 : 로직의 순서 정함
- 가상의 시계가 있다 : now()를 통해 시간 개념을 실현한다.

```js
import { Observable, asyncScheduler } from "rxjs";
import { observeOn } from "rxjs/operators";

// 이렇게 하면 asyncScheduler 함수에 따라 로직 전체가 비동기로 작동한다.
const observable = new Observable((observer) => {
  observer.next(1);
  observer.next(2);
  observer.next(3);
  observer.complete();
}).pipe(observeOn(asyncScheduler));

console.log("just before subscribe");
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

// just before subscribe
// just after subscribe
// got value 1
// got value 2
// got value 3
// done
```

- If you do not provide the scheduler, RxJS will pick a default scheduler by using the principle of least concurrency. This means that the scheduler which introduces the least amount of concurrency that satisfies the needs of the operator is chosen.

### 빌트인 스케쥴러 종류

- null : 안넘기면 동기적, 순차적 실행
- queueScheduler
- asapScheduler : 마이크로태스크큐에 큐잉(프로미스)되서 실행됨. 마이크로태스크 큐 비동기 스케쥴러
- asyncScheduler : 셋인터벌처럼 사용되는 태스크 큐 비동기 스케쥴러
- animaitonFrameScheduler : requestAnimationFrame처럼 브라우저 리페인트 이후 직후 실행

## 주요 filtering 연산자

pipe와 함께 사용된다

### filter

배열의 filter 함수처럼 조건을 통과하면 값을 발행하도록 만든다

```js
import { range } from "rxjs";
import { filter } from "rxjs/operators";

range(1, 5)
  // 필터링의 인자로 사용되는 함수를 predicate 함수라고 함
  .pipe(filter((x) => x % 2 === 0))
  .subscribe(console.log);
```

### last

마지막 값 한개만 내보내는 연산자. next로 내보내지는 값을 모아두다가 complete이 호출되기 바로 전의 값을 내보낸다. 다만 complete이 없는 observable에서는 값을 내보내지 않는다(이러면 마지막값을 특정하기가 어려운가봄)

```js
import { range } from "rxjs";
import { last } from "rxjs/operators";

range(1, 10).pipe(last()).subscribe(console.log);
// 10

range(1, 10)
  .pipe(last((x) => x <= 3))
  .subscribe(console.log);
// 3
```

### take

정해진 개수만큼 구독하고 구독을 해제한다. 별도로 unscribe 같은걸 동작하지 않아도 되기 때문에 코드가 간결해지고 동작 파악이 쉬워진다

```js
import { interval } from "rxjs";
import { take } from "rxjs/operators";

interval(1000).pipe(take(5)).subscribe(console.log);
// 1
// 2
// 3
// 4
// 5
```

### takeUntil

take은 개수 제한을 두는 형태로 동작하지만, takeUntil은 조건 제한을 둔다. 특정 조건이 만족할때 unsubscribe한다.

```js
import { fromEvent, interval } from "rxjs";
import { takeUntil } from "rxjs/operators";

const source = interval(1000);
const clicks = fromEvent(document, "click");

// 이벤트가 발생되면 구독을 멈추고 인터벌을 중지한다.
const result = source.pipe(takeUntil(clicks));

result.subscribe(console.log);
```

### takeWhile

predicate의 조건을 만족하는 동안은 구독을 유지한다.

### skip

원하는 만큼 내보내지는 값을 생략하고 그 다음 값부터 내보내진다.

```js
interval(250).pipe(skip(3)).subscribe(console.log);

// 3
// 4
// 5
// 6
// 7
// ...
```

### skipUntil

인자 값으로 observable을 받고, 인자로 받은 Observable이 구독 시작되는 조건일때부터 값을 내보낸다.

```js
// 클릭이 일어난 시점부터 값을 내보내기 시작함

import { interval, fromEvent } from "rxjs";
import { skipUntil } from "rxjs/operators";

const intervalObservable = interval(1000);
const click = fromEvent(document, "click");

const emitAfterClick = intervalObservable.pipe(skipUntil(click));

const subscribe = emitAfterClick.subscribe((value) => console.log(value));
```

### skipWhile

predicate로 들어가는 인자 함수가 만족하면 값을 건너뛴다.

```js
import { interval } from "rxjs";
import { skipWhile } from "rxjs/operators";

interval(300)
  .pipe(skipWhile((x) => x < 4))
  .subscribe(console.log);
// 4
// 5
// 6
// 7
// ...
```

## reference

- https://changhoi.github.io/categories/rxjs/
- https://rxjs-dev.firebaseapp.com/
