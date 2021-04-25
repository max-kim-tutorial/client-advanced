# hook 동작 심화

- state 업데이트는 비동기적일 수 있음 => 업데이트 실패하는 경우?
- hook 기반 리액트에서는 컴포넌트 렌더링 이후 useEffect 일괄적으로 실행(state가 바뀜 -> 재랜더링 -> 재랜더링 과정에서 컴포넌트 내부 변수, 함수 다시 만들어짐 -> useEffect 실행 -> 재랜더링 이하 어느 과정에서 state가 또 바뀐다면 다시 재랜더링 -> 함수변수 다시 만들어짐...)
- 최적화
  - useMemo, useCallback : 재랜더링 과정에서 함수나 변수를 다시 만들어지지 않게 해줌
  - useRef : 랜더링을 유발하지 않는 고정 메모리값
  - 일단 useState를 최소한으로 선언해야함 => state는 진짜로 컴포넌트 렌더링에 필수적인 값들이어야 함

## [useRef](https://dev.to/dylanju/useref-3j37?fbclid=IwAR0fl7xMjn_Hp6sImCU-EQt4gJ0ob_YY6hS3cwn4ARyClTUYD2KN0R6X-O0)

- ref는 일반적인 자바스크립트 객체 : 매번 렌더링할때마다 동일한 객체를 제공함. 함수가 실행될때 메모리에 할당되었다가 종료되면서 해제되는 stack 영역이 아니라 heap에 저장되어있기 때문에 어플리케이션이 종료되거나 GC될때까지 참조할때마다 같은 메모리값을 가짐.
- heap은 전역변수와 참조타입의 변수를 할당하고, 가비지 컬렉터를 이용해 사용하지 않는(참조 카운트가 없는) 메모리를 해제시킴. 자바스크립트 객체로 만드는 변수들은 모두 heap에 할당되었다가 해제됨
- 값이 변경되어도 리렌더링이 되지 않음 : 정확히 말하면 같은 메모리 주소를 가지고 있는 객체이기 때문에 변경사항을 감지할 수가 없음

```js
function useRef(initialValue) {
  var dispatcher = resolveDispatcher();
  return dispatcher.useRef(initialValue);
}

// 걍 말그대로 그냥 객체에 넣음
function resolveDispatcher() {
  var dispatcher = ReactCurrentDispatcher.current;
  if (!(dispatcher !== null)) {
    {
      throw Error("Invalid hook call Hooks can only be called inside~~");
    }
  }
  return dispatcher;
}

// current를 프로퍼티로 갖고있는 플레인 오브젝트
var ReactCurrentDispatcher = {
  current: null,
};
```

- 함수형 컴포넌트는 인스턴스를 찍어내면 그대로 모든 변수와 함수가 살아있는 클래스 컴포넌트와는 달리, 렌더링 될 때마다 매번 새로운 변수를 스택에 할당해 값이 초기화되기도 하고, 불필요한 성능낭비를 하게 되기가 쉬움. 클래스 컴포넌트는 인스턴스를 생성해서 렌더링 메소드만 재실행하는 구조였다면, 함수형 컴포넌트는 매번 함수를 실행하기 때문(비교적 stateless함)
- 다른 변수와의 차이
  - state, context : 이런 변수들은 값이 바뀔때마다 리렌더링 유발
  - const, let, var : 렌더링 될때마다 값이 초기화됨. 컴포넌트 생애주기 동안 관리해주는 변수를 선언하기에 적당하지 않음
  - 컴포넌트 바깥에 변수선언 : 컴포넌트를 재사용하면서 값을 따로 관리하는게 불가능하고 재사용하는 컴포넌트들간 변수를 공유함
- useRef는 리렌더링을 유발하지 않고, 리렌더링될때도 이전의 값을 기억하고 있으며 컴포넌트마다 각각의 값을 가질 수 있음. 컴포넌트가 생성될때 처음 할당되고, state가 변화하여 리렌더링이 몇번 되든간에 새로운 값을 current에 할당하지 않는 이상은 값도 안바뀌고, 메모리 주소도 안바뀜
- 일종의 클래스 인스턴스 프로퍼티와 같다고 생각하면 됨.
- [Math.random으로 렌더링간 값이 유지되는지 실험](https://flyingsquirrel.medium.com/react-%EC%BD%94%EB%93%9C-%EA%B9%8C%EB%B3%B4%EA%B8%B0-useref%EB%8A%94-dom%EC%97%90-%EC%A0%91%EA%B7%BC%ED%95%A0-%EB%95%8C-%EB%BF%90%EB%A7%8C-%EC%95%84%EB%8B%88%EB%9D%BC-%EB%8B%A4%EC%96%91%ED%95%98%EA%B2%8C-%EC%9D%91%EC%9A%A9%ED%95%A0-%EC%88%98-%EC%9E%88%EC%96%B4%EC%9A%94-f0359ad23f3b)

## useMemo, useCallback

섣부른 사용이 발적화라고 해도 몰라도 되는 건 아니다  
저번 hook reference에서 자세히는 정리하지 못했던 내용이라 정리해봄니다  
그냥 의존성 배열이라는 조건을 달아준 useRef인데, 여전히 스택 영역에 존재하는지라 좀 special treatment가 필요한 친구들이라고 생각하면 옳은듯.

### 발적화

- [Performance optimizations ALWAYS come with a cost but do NOT always come with a benefit. Let's talk about the costs and benefits of useMemo and useCallback.](https://ideveloper2.dev/blog/2019-06-14--when-to-use-memo-and-use-callback/)
- useCallback을 예로 들어보면 - 쌩 배열 하나를 새로 선언해줘야되고, 논리적인 표현들을 통해 property들을 세팅하는 useCallback까지 임포트해와서 불러야함. 모든 render에서 함수 정의를 위해 메로리를 할당하고, 함수 정의를 위해 사실 더 많은 메모리를 할당하게 됨
- useMemo는 이러한 메모이제이션을 함수 뿐 아니라 다른 value type에 상관없이 적용시킬 수 있다는 점에서 비슷.

### 용례 1) 참조 동일성을 따져야 할때

```js
function Foo({ bar, baz }) {
  const options = { bar, baz };

  useEffect(() => {
    buzz(options);
  }, [options]);
  // 의존성 배열에 떡하니 객체가 들어가 있고
  // bar이나 baz 프로퍼티가 변할때마다 effect를 실행하고 싶음

  return <div>foobar</div>;
}

function Blub() {
  return <Foo bar="bar value" baz={3} />;
}
```

- 얘는 이제 options객체가 변화할때마다 effect를 실행시키고 싶은 상황인건데, 의존성 배열에 들어있는 options에 대해 reference equality check를 매 렌더링마다 일단 하게는 되고, 자바스크립트의 작동 방식 때문에 options는 매번 다시 렌더링됨. 그래서 이렇게 options를 매번 새로 만들어 넣는게 의존성 배열의 요소로 역할을 못하게됨
- useEffect안으로 의존성이 있는 친구들을 넣으면 해결이 됨. 그치만 bar이나 baz가 참조형 자료형(객체배열함수)일 수도 있음. 그러면 효용이 또 없어진다

```js
// option 1
function Foo({ bar, baz }) {
  // 하지만 만약에 bar과 baz가 불변형 타입이 아니라면, 또 매번 새로운 레퍼런스가 내려올 것이므로
  // bar, baz는 계속 존재 자체만으로 useEffect의 실행을 유발할 것이다.
  React.useEffect(() => {
    const options = { bar, baz };
    buzz(options);
  }, [bar, baz]); // 이러면 bar, baz의 변화를 useEffect가 따라간다.

  return <div>foobar</div>;
}

function Blub() {
  const bar = () => {};
  const baz = [1, 2, 3];
  return <Foo bar={bar} baz={baz} />;
}
```

- 이런 상황을 useMemo나 useCallback으로 고칠 수 있다.

```js
// option 1
function Foo({ bar, baz }) {
  // bar, baz는 최초 한번 선언된후 메모이제이션 되므로 바뀌지 않을 것
  React.useEffect(() => {
    const options = { bar, baz };
    buzz(options);
  }, [bar, baz]);

  return <div>foobar</div>;
}

function Blub() {
  const bar = React.useCallback(() => {}, []);
  const baz = React.useMemo(() => [1, 2, 3], []);
  return <Foo bar={bar} baz={baz} />;
}
```

### 용례 2) 컴포넌트를 React.memo로 감싸서 불필요한 렌더링 없애기

- 개발하다보면 state의 변화로 state와 직접적인 관련이 없는 컴포넌트들이 렌더링되는 상황이 존재할 수 있음. 아무것도 바뀌지 않았지만 리렌더링이 되는 일을 방지하기 위해 memo를 사용 가능하다.
- 이때 memo로 감싼 컴포넌트의 프롭이 참조 타입이라면, useCallback으로 감싸줄 경우 제대로 reference check이 가능하다.

```js
// 이제 얘는 prop 값이 바뀔때만 리렌더를 하게 됨
const CountButton = React.memo(function CountButton({ onClick, count }) {
  return <button onClick={onClick}>{count}</button>;
});

// 하지만 얘는 callback이랑 같이 써줘야하는데, 함수 prop은 매번 새로 만들어져 reference를 갱신하기 때문임
function DualCounter() {
  const [count1, setCount1] = React.useState(0);
  const increment1 = React.useCallback(() => setCount1((c) => c + 1), []);

  const [count2, setCount2] = React.useState(0);
  const increment2 = React.useCallback(() => setCount2((c) => c + 1), []);

  return (
    <>
      <CountButton count={count1} onClick={increment1} />
      <CountButton count={count2} onClick={increment2} />
    </>
  );
}
```

### [용례 3) 함수를 의존성 배열에 넣고 싶을때](https://rinae.dev/posts/a-complete-guide-to-useeffect-ko#%ED%95%98%EC%A7%80%EB%A7%8C-%EC%A0%80%EB%8A%94-%EC%9D%B4-%ED%95%A8%EC%88%98%EB%A5%BC-%EC%9D%B4%ED%8E%99%ED%8A%B8-%EC%95%88%EC%97%90-%EB%84%A3%EC%9D%84-%EC%88%98-%EC%97%86%EC%96%B4%EC%9A%94)

- async 함수를 useEffect 내부에서 쓰고 싶다면, effect 안으로 옮겨 의존성 배열에 거짓말을 하지 않는 상황이 바람직하다.
- 그런데 async 함수 하나를 선언해놓고, 두 개 이상의 useEffect에서 쓰고 싶다면 이런 동작이 달갑지는 않다.
- 그러면 따로 async 함수를 선언하고 이를 useCallback으로 감싸서 의존성 배열에 집어넣는 방법을 사용할 수 있음.
- 함수 자체가 필요한 때에만 바뀌고 => useEffect가 호출될 수 있게 만든다
- **함수를 명백하게 데이터 흐름에 포함시킬 수 있는 방법**
- 클래스 컴포넌트의 경우에는 state가 바뀔때 메서드 자체를 바꿀 수 있는 방법이 없다. 걍 메서드는 클래스 인스턴스에 바인딩된 메서드이기 때문에 참조값이 똑같아서 똑같은게 계속 내려옴. => 함수만 필요할 때도 차이를 감지하기 위해 변수를 쓸데없이 계속 내려보내서 캡슐화를 깨트려야만 했음
- 매번 새로운 변수와 함수를 만들어내는 함수형 컴포넌트이기 때문에, 함수를 레이어 하나로 더 싸서 특정 변수가 바뀔때마다 새로운 함수를 만들어낼 수 있게 만드는 것이 가능하다.

```js
function SearchResults() {
  // query => useCallback => useEffect
  const [query, setQuery] = useState("react");

  // ✅ 여기 정의된 deps가 같다면 항등성을 유지한다
  const getFetchUrl = useCallback(() => {
    return "https://hn.algolia.com/api/v1/search?query=" + query;
  }, [query]); // ✅ 콜백의 deps는 OK

  useEffect(() => {
    const url = getFetchUrl(query);
    // ... 데이터를 불러와서 무언가를 한다 ...
  }, [getFetchUrl]); // ✅ 이펙트의 deps는 OK

  useEffect(() => {
    const url = getFetchUrl(query);
    // ... 데이터를 불러와서 무언가를 한다 ...
  }, [getFetchUrl]); // ✅ 이펙트의 deps는 OK
  // ...
}
```

### 정리

- 레퍼런스 비교를 하는 리액트답게, props가 참조형이라면 매번 객체나 함수나 배열의 레퍼런스는 계속 달라지므로, useEffect 등의 의존성 배열에 쓰일 때 의미가 없어진다.
- useMemo나 useCallback을 적절히 사용하면 useEffect등의 올바른 동작이나 의존성 배열에 뻥을 치지 않는 상황에 도움이 된다.
- 하지만 최적화란 그 비용이 최적화로 얻을 수 있는 이득을 상회하지 않을때 적절한 것이므로, 실익을 따져야하고 정말 필요할 때 사용해야 한다. => 사용해보고 메모리나 속도에 유의미한 차이를 가져오는지 확인한다거나,,

## 리액트의 레퍼런스 비교

## state 업데이트가 바로 반영되지 않는 경우
