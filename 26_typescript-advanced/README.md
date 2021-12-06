# 타입스크립트에서 몰랐던 것들 정리

<이펙티브 타입스크립트>를 읽고 밑줄친 부분

## 런타임에는 타입 체크가 불가능하다.

```ts
interface Rectanle {
  width:number;
  height:number;
}

if (shape instanceof Rectangle) {
  // type만 참조하는데 여기서는 값으로 사용하고 있군여! error 뿜뿜
  ...
}
```

- type만 참조하지만 여기서는 값으로 사용하고 있다는 에러. 런타임에 타입 정보를 유지할 수 있는 방법이 없으므로 에러가 나는 것이다.
- 이런 에러를 방지하고 런타임간 타입 정보를 가지고 뭘 해야 하는게 있다면 다음과 같은 방법을 사용해볼 수 있다.
  - 속성의 존재유무 체크
  - 런타임에 접근 가능한 타입을 명시적으로 지정하는 태그 기법 (type:rectangle)
  - 타입을 클래스로 만들면 얘는 타입과 값 둘 다에서 참조가 가능함(instanceof 사용 가능)

## any가 물을 흐리는 경우

- 앞에서 특정 타입으로 선언된 변수에서도 as any로 단언을 떼려버리면 다른 타입을 할당할 수 있게 되어버린다.
- any는 함수 시그니처를 무시해 버린다. any로 설정해서 인자로 넣으면 시그니처를 무시한다.
- any는 타입 설계를 감춰버린다 : 특히 객체를 정의할때 문제가 되는데 상태 객체의 설계를 감춰버려 알 수가 없게 만들어버린다.

## 타입은 집합

never(공집합) < unit(단일한 값) < unit union(단일한 값의 묶음) < type(모두 포함)

## extends의 참의미

- extends는 포함관계를 의미한다. ~의 부분집합. 해당 인터페이스를 포함해야 한다는 어떤 조건을 제시한다.
- extends 키워는 제네릭 타입에서 **한정자**로 사용한다.

## typeof의 신기한 사용례

typeof는 타입에서 쓰일 때와 값에서 쓰일 때 다른 기능을 한다.

```ts
type T1 = typeof p; // "Person" 이미 정의해뒀던 특정 인터페이스나 타입
type T2 = typeof email; // () => Response 시그니처

const v1 = typeof p; // "object"
const v2 = typeof email; // "function" (문자열)
```

값에다가 typeof를 쓰면 자바스크립트의 런타임 타입 시스템을 이용하는 것이라고 할 수 있다. 이는 타입스크립트의 그것보다 훨씬 간단한데, 타입스크립트는 타입의 종류가 무수히 많은 반면에 자바스크립트는 6개의 런타임 타입만이 존재한다. 

## 기본형과 객체 래퍼 타입의 차이

타입스크립트는 기본형과 객체 래퍼 타입을 별도로 모델링한다. 대문자로 시작하는게 객체 래퍼 타입이다. 오른쪽의 경우 타입스크립트에서 사용하는 경우 "가능한 경우 string을 사용하라"는 에러가 뜬다.

- string, String
- number, Number
- boolean, Boolean
- symbol, Symbol
- bigint, BigInt

우리가 흔히 `string.prototype.뭐시기`라고 말을 많이 하지만, string 기본형에는 사실 메서드가 없다. 자바스크립트에는 메서드를 가지는 String 객체 타입이 정의되어 있고 JS는 기본형과 객체 타입을 서로 자유롭게 변환한다. 

string 기본형에 charAt같은 메소드를 사용할 때, 자바스크립트는 기본형을 String 객체로 래핑하고, 메서드를 호출하고, 마지막에 래핑한 객체를 버린다.(!)

## interface의 고유 기능

타입은 인터페이스의 기능을 대부분 가지고 있고 표현이 가능하지만, 인터페이스에 고유한 기능이 몇가지가 있다. 

- 선언 병합 : 같은 이름의 interface를 여러개 선언해놓은 경우 다수의 선언이 병합되어 하나의 인터페이스가 된다. 일종의 오버라이딩. 새로운 타입이 네임스페이스는 유지한채로 계속 업데이트되는 경우에 기존 interface를 고치지 않고 그냥 붙이는 식으로 활용할 수 있다.

```ts
interface IState {
  name:string;
  capital: string;
}

interface IState {
  population: number;
}

const wyoming:IState = {
  name: 'wyoming',
  capital: 'Cheyenne',
  population: 500_000,
}
```

- 책 구절 : 복잡한 타입이라면 고민할 것도 없이 타입 별칭을 사용해라. 그러나 타입과 인터페이스, 두 가지 방법으로 모두 표현할 수 있는 경우에 일관성과 보강의 관점에서 고려해라. 일관되게 인터페이스를 사용하는 코드베이스에서 작업하고 있다면 인터페이스를 사용해라. 일관되게 타입을 사용 중이라면 타입을 사용해라 => 둘중에 하나만 써서 컨벤션 통일해라

근데 이렇게만 고려해보면 되는걸까? 너무 단순한디? 다른건 없을까? type이냐 interface을 언제 쓸지에 대해서는 다른 이유들을 더 수집해서 고민해보자.

## 타입 추론의 강도 조절하기

한 객체의 타입은 다양한 범위로 추론될 수 있다. 

```ts
const v = {
  x: 1,
}

// 가장 구체적인 경우 {readonly x : 1}
// {x:number}
// {[key:string]:number} or object => 가장 추상적
```

이는 다음과 같은 방식으로 조절이 가능하다.

- 명시적 타입 구문 제공
```ts
const v:{x:1|3|5} = {
  x:1
}
```

- 타입 체커에 추가적인 문맥 제공 : 함수의 매개변수로 값을 전달한다던가..
- const 단언문 사용(as const) : 값 뒤에 const를 작성하면 타입스크립트는 최대한 좁은 타입으로 추론한다.

```ts
const v1 = {
  x:1
  y:2
} // {x:number, y:number}

const v2 = {
  x : 1 as const,
  y: 2
} // {x:1, y:number}

const v2 = {
  x:1,
  y:2
} as const; // {readonly x:1, readonly y:2};
```

## 타입 좁히기 - 타입 가드, 타입 평가, 식별법

- null 체크 : null인지 아닌지
- instanceof
- 내장 함수(Array.isArray 같은)
- 명시적 태그 붙이기(위에서 봤음) : 이거를 컨벤션을 지켜서 이용하면 애플리케이션단에서 좀 편하게 개발할 수 있지 않을까?
- 사용자 정의 타입 가드(is)

```ts
// 함수의 반환이 true인 경우
// 타입 체커에게 매개변수의 타입을 HTMLInputElement로 좁힐 수 있다는 것을 알려줌
// 되게 안직관적이면서 직관적인 문법이네..
function isInputElement(el:HTMLElement):el is HTMLInputElement {
  return 'value' in el; // 해당 속성이 있는지
}

function getElementContent(el:HTMLElement) {
  if (isInputElement(el)) {
    return el.value
  }
  return el.textContent
}
```
- 효과적이지 않은 방법 : `!`이나 `typeof`
  - typeof는 `typeof null이 object`로 나오는 유명한 버그 때문에 좁히기가 잘 안될수도 있고
  - `!`의 경우도 거짓값은 많기때문에(0, false, '') 잘 좁히기가 안 될수도 있음


## 상수 문맥

타입 추론이 이루어질때 선언자가 `let`이냐 `const`이냐에 따라서도 타입 추론이 어떻게 되는지가 달라진다. 

- let : 재할당이 가능해지기 때문에 초기에 할당된 변수 바탕으로 넓게 추론된다.
  - 문자열 리터럴이 처음에 할당될 경우 string으로 넓게 추론
  - 할당시에 타입 정보를 제공하거나 const를 사용하면 이런 문제가 없어진다.
- const : 재할당이 안되므로 좁게 추론된다.

const는 단지 값이 가리키는 참조가 변하지 않는 얕은 상수인 반면에, as const는 그 값이 내부까지(deeply) 상수라는 사실을 타입스크립트에게 알려준다.

```ts
const loc = [10, 20] // number[]로 추론
const lco2 = [10, 20] as const // readonly [number, number] => 매우 좁게 추론
// 근데 이런 경우면 그냥 튜플 타입선언 사용하는게 나을듯?
```

as const는 문맥 손실과 관련한 문제를 해결할 수 있지만 타입 정의에 실수가 있을 경우 오류는 타입 정의가 아니라 호출되는 곳에서 발생하기 때문에 근본적인 원인을 파악하기 어려울 수도 있다.

## 타입스크립트는 함수형 기법과 찰떡이다

타입 정보가 그대로 유지되면서 타입 흐름이 계속 전달되도록 하기 때문에 타입 추론이 쉽고 개발자가 별다른 타입 평가나 단언에 대한 고려를 할 필요가 없기 때문이다. 만약에 직접 루프를 구현해서 뭔가 하려고 할때는 타입 체크에 대한 관리도 직접 해야 한다.

당연하지만 함수 호출시 전달된 매개변수 값을 건드리지 않고 매번 새로운 값을 반환하니 새로운 타입으로 안전하게, 부수효과 없이 반환하는게 보장되는 상황이 된다.

## 제네릭으로 any 쓸 일 없애기

인자의 타입을 제네릭으로 지정해주고 다른 제네릭, 혹은 인자의 타입까지 추론하게끔 할 수 있음

```ts
type K = keyof Album;

// d
function pluck<T>(records:T[], key:keyof T) {
  return records.map((r => r[key]));
}

// K를 자동으로 T의 key이게 하기
function pluck<T, K extends keyof T>(records: T[], key:K):T[K][] {
  return records.map(r => r[key]);
}

function pluck<T>(records:T[], key:keyof T) {
  return records.map((r => r[key]));
}

const arr = [
  {a: 1,b: 2},
  {a: 2,b: 3},
  {a: 3,b: 4},
]

const a = pluck(arr, 'a') // 제네릭 안 써도 문제없음!
```

## 타입 설계 팁

- 매우 일반적인 용어를 최대한 구체화하기 (name, id 등등)
- 잘 표현이 되는 속성인지 다시 생각해보기 (특히 불리언)
- 좁혀야할 속성이 없는지 생각해보기 (그냥 string말고 유니언 타입이 필요할수도)
- 변수명과의 관계 생각해보기

## 타입을 오염시키는 any

함수가 any를 반환해버리는 경우 그 영향력은 프로젝트 전반에 전염병처럼 퍼진다. any를 사용해야 한다면 사용 범위를 최대한 좁게 제한하는 함수를 사용해야한다. any가 함수 바깥으로 나가서 영향을 미치지 않도록,,,

```ts
function f1 () {
  const x : any = expressionReturningFoo();
  processBar(x);
}

function f2() {
  const x = expressionReturningFoo();
  processBar(x as any); // 이게 낫다 이러면 인자 말고 다른 코드에는 영향을 안미침
}

function f1() {
  const x:any = expressionReturningFoo();
  processBar(x);
  return x
}

function g() {
  const foo = f1(); // any를 반환
  foo.fooMethod(); // 이 함수의 타입 체크가 안됨 any라서..
}


```

이런식으로 any를 쓸 바엔 차라리 ts-ignore을 쓰는게 나을수도 있다.

## 이런것도 가능하다!

```ts

// 유니언 타입으로 key 타입들 선언
type TopNavState = {
  [k in 'userId' | 'pageTitle' | 'recentFiles'] : State[k];
}

type ActionType = Action['type'] // 이렇게 속한 키 타입으로 인덱싱 가능
```