# 생소한(?) 훅들

useImperativeHandle, useLayoutEffect, useDebugValue

## useImperativeHandle

ref를 넘겨줄 일이 많이 없기는 하다

- ref를 사용할때 부모 컴포넌트에 노출되는 인스턴스 값을 사용자화 함, forwardRef랑 사용함
- ref를 선언한 컴포넌트가 아닌 외부에서 참조할때, 부모에서 자식으로 ref를 넘겨줄때 사용할 수 있음
- 부모는 자식의 DOM에 직접 접근하는게 아니라 useImperativeHandle로 전달된 메서드에만 접근이 가능해짐
- 약간 proxy를 이용한 객체 증강? 그런 느낌이 난다 => 부모에게 자식의 실제 reference를 보내지 않고, 프록시 객체를 보낸다

```jsx
const FormControl = React.forwardRef((props, ref) => {
 const inputRef = useRef(null);

  // 부모에서 prop으로 넘긴 ref와 이어?준다
 useImperativeHandle(ref, () => ({getValue(){
  return inputRef.current.value
 }) }, [inputRef]); // deps도 추가가능하다.

 return (
  <div className="form-control">
    <input type="text" ref={inputRef} />
  </div>
 );
})

const App = () => {
 // 부모에서 선언된 ref
 const inputRef = React.useRef();
 return (
  <div className="App">
    <FormControl ref={inputRef} />
    <button
     onClick={() => {
      console.log(inputRef.current.getValue());
     }}>
     focus
    </button>  
  </div>
 );
}
```

```jsx
// ref를 Prop으로 받고있다
function FancyInput(props, ref) {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus(); // 부모에서 넘겨받은 ref에 대한 사용자 메서드
    }
  }));
  return <input ref={inputRef} ... />;
}

FancyInput = forwardRef(FancyInput);
```

### forwardRef

```jsx
import React, { forwardRef, useRef } from "react";

// ref을 prop으로 받아야 하는 컴포넌트를 감싸준다
const Input = forwardRef((props, ref) => {
  return <input type="text" ref={ref} />;
});

function Field() {
  const inputRef = useRef(null);

  function handleFocus() {
    inputRef.current.focus();
  }

  return (
    <>
      <Input ref={inputRef} />
      <button onClick={handleFocus}>입력란 포커스</button>
    </>
  );
}

```

- ref를 자식으로 전달하는 기능
- ref는 일반적으로 HTML 접근이라는 특수한 용도로 사용되기 때문에 일반적인 prop으로 사용이 안됨
- HTML elem이 아닌 React 컴포넌트에서 ref prop을 사용하려면 React에서 제공하는 forwardref()을 사용하면 됨
- 컴포넌트를 감싸주면, 해당 컴포넌트는 Props말고 ref라는 매개변수를 갖게 되는데 이를 통해 넘겨주면 된다.

## useLayoutEffect

- useEffect와 동일한 함수 시그니처를 가지고 있지만, 얘는 모든 DOM 변경 이후에 동기적으로 발생한다
- DOM에서 레이아웃을 읽고 동기적으로 리렌더링하는 경우에 응용 가능
- useLayoutEffect의 내부에 예정된 갱신은 브라우저가 화면을 그리기 이전 시점에 동기적으로 수행
- useEffect : 렌더링이 된 후 비동기적으로 실행(컴포넌트 렌더링 - 화면 업데이트 - useEffect실행)
- useLayoutEffect: 렌더링 후 화면이 업데이트 되기 전에 동기적으로 실행(컴포넌트 렌더링 - useLayoutEffect 실행 - 화면 업데이트)
- 상태가 업데이트될때 요소가 깜빡거리는 경우나, DOM을 변경하려는 경우에 응용 가능
- 근데 독스에서는 최대한 useEffect를 써보고 문제가 있는 경우에 또 써보라고 함

## useDebugValue

- React 개발자 도구에서 사용자 Hook 레이블을 표시하는데에 사용할 수 있음 별로 중요하지는 않군