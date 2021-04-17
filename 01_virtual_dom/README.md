# React의 렌더링

## [리액트 엘리먼트](https://ko.reactjs.org/docs/rendering-elements.html)

```jsx
const element = <h1>Hello, world</h1>;

// 돔 노드 렌더링
ReactDOM.render(element, document.getElementById("root"));
```

- **React element는 불변객체**, 엘리먼트를 생성한 이후에는 해당 엘리먼트의 자식이나 속성을 변경할 수 없음. 엘리먼트는 영화에서 하나의 프레임과 같이 특정 시점의 UI를 보여줌
- UI를 업데이트하는 유일한 방법은 새로운 element 생성하고, 이를 ReactDOM.render()로 전달하는 것
- 변경된 부분만 업데이트 : React DOM은 해당 엘리먼트와 그 자식 엘리먼트를 이전의 엘리먼트와 비교하고, DOM을 원하는 상태로 만드는데 필요한 경우에만 DOM을 업데이트한다. => 상태가 바뀌었을 때 개발자 도구를 사용해서 업데이트 양상 살펴보기

```jsx
// 인터벌로 매초 render을 실행하지만, 개발자 도구로 살피면 업데이트가 되는 부분만 바뀜
function tick() {
  const element = (
    <div>
      <h1>Hello, world!</h1>
      <h2>It is {new Date().toLocaleTimeString()}.</h2>
    </div>
  );
  ReactDOM.render(element, document.getElementById("root"));
}

setInterval(tick, 1000);
```

- 엘리먼트 렌더링에서의 state: **render을 캡슐화.** 이 컴포넌트에 한해서는 스스로 타이머를 설정하고, 매초 스스로 업데이트하게 만드는 것 => 하위 컴포넌트 안에서의 state 변경은 render을 호출하게 만든다
- 하향식, 단방향식 데이터 흐름 : 모든 state는 항상 특정한 컴포넌트가 소유하고 있으며, 그 state로부터 파생된 UI또는 데이터는 오직 트리구조에서 자신의 아래에 있는 컴포넌트에만 영향을 미치게 됨.

## 직접 해보기(Tiny React)

바벨은 JSX를 React Element로 트랜스파일링한다.

```js
{
  "presets": ["@babel/preset-react"]
}
```

가상돔은 Real Dom으로 렌더링되기 전에 React ELement는 대충 그 가상돔 element에 대하여 다음과 같은 구조를 가진다.

```js

<div id="root">
  <span>아무말</span>
</div>

{
  tagName: 'div',
  props : {
    id: 'root',
    className: 'container'
  },
  children : [
    {
      tagName: 'span',
      props: {},
      children: [
        '아무말'
      ]
    }
  ]
}
```

```jsx
// react.js

function renderRealDOM(vdom) {
  if (node === undefined) return;
  // 바닥조건(string이면 마지막 요소)
  if (typeof vdom === "string") {
    return document.createTextNode(vdom);
  }
  // 재귀
  const $el = document.createElement(vdom.tagName);
  vdom.children.map(renderRealDom).forEach((node) => {
    $el.appendChild(node);
  });
  return $el;
}

// 렌더링을 하는 함수 => 가상돔 객체가 넘어옴
// tagName은 컴포넌트(함수 자체)일수도 있고, html태그일 수도 있는데 컴포넌트의 경우는 앞글자가 대문자로 시작하게끔 강제함
// 바벨은 그 차이를 캐치해서 tagName에 직접 함수 자체를 넣어줄수도, 문자열(div, span 등등)을 넣어줄수도 있음

// 클로저를 사용한 이전 돔과의 비교 + 즉시실행함수로 클로저를 초기에 선언
export const render = (function () {
  let prevVDom = null;

  return function (nextVdom, container) {
    if (prevDom === null) {
      prevVDom = nextVdom;
    }
    // prev와 next사이의 diff 비교
    container.appendChild(renderRealDom(nextVdom));
  };
})();

// 리액트 요소로 만드는 함수(바벨로 바꿔줄때 들어가는 인자 고려해서)
// 클래스도 렌더링이 가능하게 하려면, 클래스와 함수 컴포넌트를 분리할 수 있어야 함(Component를 상속했느냐를 체크)
// 클래스의 경우는 인스턴스를 만들면 그냥 지 안에서 state를 가질 수 있음

// 컴포넌트의 위치값 => state가 바뀌면 재랜더링
const hooks = [];
let currentComponent = -1;

export function createElement(tagName, props, ...children) {
  if (typeof tagName === "function") {
    // 함수형 컴포넌트의 개수가 보존된다는 전제로 훅과 맵핑
    currentComponent++;
    return tagName.apply(null, [props, ...children]);
  }

  return { tagName, props, children };
}
```

컴포넌트

```jsx
import { createElement, render } from "../react.js";

function Title() {
  return <h2>ㄹㅇ루 동작하나?</h2>;
}

console.log(Title());
// 렌더의 캡슐화
render(<Title />, document.querySelector("#root"));
// 바벨을 통해서는 이런식으로 컴파일 : return createElement("h2", null, "ㄹㅇ루 동작하냐?");
// 즉 함수형 컴포넌트는 리액트 요소를 리턴하게 됨
```

state 훅

```jsx
// 리액트가 상태를 만들어 주고 함수 컴포넌트에게 제공해주는 구조
// 어캐 간단한 함수가 상태를 계속 가지고 있게 되는지? 어떻게 넘겨주는거지?
// 짜피 가상돔가지고 리얼돔을 렌더링할때 가상돔 내의 함수들은 개수와 순서가 똑같다
// 어딘가에 그 함수 컴포넌트의 호출 순서를 저장해놓을 수 있다면 훅도 관리가 되지 않을까?

export function useState(initValue) {
  // 클로저! => 최초호출이냐 아니냐에 따라 값을 배열에 인덱스로 저장해버리기
  const position = currentComponent;
  if (!hooks[position]) {
    hooks[position] = initValue;
  }
  return [
    value,
    (nextValue) => {
      hooks[position] = nextValue;
    },
  ];
}
```

- 훅은 그렇기때문에 함수형 컴포넌트 안 **최상위**에서만 사용하고, 분기를 통해 조건부로 훅을 선언할 수 없다 : 항상 그 컴포넌트에서 동일한 렌더링 순서가 보장이 되어야 하는 상황에서만 사용할 수 있다 => 인덱스가 꼬임

## [재조정(Reconciliation), diff 알고리즘](https://ko.reactjs.org/docs/reconciliation.html)

- state나 prop이 갱신되면 render() 함수는 새로운 React 엘리먼트 트리를 반환할 것 => 이때 React는 방금 만들어진 트리에 맞게 가장 효과적으로 UI를 갱신하는 방법을 알아낼 필요가 있음
- 하나의 트리를 가지고 다른 트리로 변환하기 위한 연산수를 구하는 알고리즘 => 대충 O(n^3) 정도의 무지느린 성능을 지님
- React에 이 알고리즘을 적용하면, 1000개의 엘리먼트를 그리기 위해 10억번의 비교연산을 수행해야함. 너무나 비쌈. React는 대신, 두가지 가정을 기반하여 O(n) 복잡도의 휴리스틱 알고리즘을 구현
  1. 서로 다른 타입의 두 엘리먼트는 서로 다른 트리를 만들어냄
  2. 개발자가 key prop을 통해 여러 렌더링 사이에서 어떤 자식 엘리먼트가 변경되지 않아야 할지 표시해 줄 수 있다 => key가 바뀌면 렌더링

### 비교 알고리즘

- **두 루트 elem의 타입이 다르면** React는 이전 트리를 버리고 완전히 새로운 트리를 구축.
- 트리를 버리면 이전 DOM 노드들은 모두 파괴 => 생명주기로 따지면 돔 노드들이 파괴될때 unmount 실행, 새로운 노드는 willMount, DidMount 실행
- 안에 있었던게 동일해도 루트 elem 태그가 다르면 그 안에있는것도 다 사라지고 처음부터 다시 마운트
- **타입이 같으면** 두 엘리먼트의 속성을 확인하여 동일한 내역은 유지하고 변경된 속성들만 갱신. 여기서 속성은 className같은거..
- DOM노드 처리가 끝나면 React는 이어서 해당 노드의 자식들을 재귀적으로 처리
- 컴포넌트 갱신시 인스턴스는 동일하게 유지되어 렌더링간 state가 유지. React는 새로운 엘리먼트의 내용을 반영하기 위해 현재 컴포넌트 인스턴스의 prop을 갱신

### keys를 쓰는 이유

- 리스트 렌더링시, diff를 할때 react에게 어떤 elem과 비교해야 하는지 힌트를 주는 속성
- 동일 돔 노드의 자식들을 재귀적으로 처리할때 React는 기본적으로 동시에 두 리스트를 순회하고 차이점이 있으면 변경을 생성함
- 자식의 끝에 엘리먼트를 추가하면 두 트리 사이의 변경은 잘 작동함(일치하는 것 계속 확인 가능). 하지만 맨 앞에 추가하는 경우 다 달라버리기 때문에 모두 새로 그리는 불상사가 일어남
- 자식들이 key를 가지고 있다면, React는 key를 통해 기존 트리와 이후 트리의 자식들이 일치하는지 확인함.(key가 동일한 것끼리 비교함). key는 형재 사이에서 유일하면 됨
- 배열의 인덱스를 key로 사용하는 것은 피해야함. 인덱스가 elem의 배열에 따라 숫자가 계속 바뀔 수 있기 때문에 최적화가 잘 안됨. 배열 자체에서 만약 항목들이 재배열되지 않는다면 잘 작동하지만, 배열이 완전 재배열되는 경우 비효율적으로 작동함
- 인덱스를 키로 사용할때 배열이 재배열되면 컴포넌트의 state와 관련된 문제가 발생 가능. 컴포넌트 인스턴스는 key를 기반으로 갱신되고 재사용됨. 인덱스를 Key로 사용하면, 항목의 순서가 바뀌면 key또한 바뀌기 때문. 그 결과로 컴포넌트의 state가 엉망이 되거나 [의도하지 않은 방식으로 바뀔 수 있음](https://ko.reactjs.org/redirect-to-codepen/reconciliation/index-used-as-key)

### 그래서 가상돔이란?

- **이 과정 전체를 추상화해서 표현한 단어가 바로 가상돔인거같음**
- state값이 바뀔때마다 모든 노드의 변경점을 파악하고 브라우저에 다시 렌더트리 돔트리 만들어서 그리는 행위는 매우 비용이 높은데, React는 가상돔을 사용해서 미리 메모리에서 react element를 모두 만든 후, 재조정을 통해 diff를 골라내서 그 부분만 따로 반영시킨다.
- 일종의 버퍼인 셈
