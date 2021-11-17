# React에서의 선언적인 에러 처리

https://jbee.io/react/error-declarative-handling-1/  
https://ko.reactjs.org/docs/error-boundaries.html

UI의 일부분의 존재하는 자바스크립트 에러가 전체 애플리케이션을 중단시켜서는 안된다  
깨진 UI를 보여주지 말자

## 에러 처리

## React Error Boundary

- 하위 컴포넌트 트리의 어디에서든 자바스크립트 에러를 기록하며 깨진 컴포넌트 트리 대신 fallback UI를 보여주는 React 컴포넌트
- 생명주기 메서드인 `static getDerivedStateFromError` 와 `componentDidCatch()`중 하나를 정의하면 클래스 컴포넌트 자체가 에러 경계가 됨
- 함수형 컴포넌트에서는 아직 에러를 캐치할 수 있는 훅이 따로 없다. [아직 없지만 곧 추가한댄다](https://ko.reactjs.org/docs/hooks-faq.html#do-hooks-cover-all-use-cases-for-classes)
- SWR이나 React Query같은 라이브러리들에서는 자체적인 ErrorBoundary를 제공하는 것 같다.

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    // 다음 렌더링에서 폴백 UI가 보이도록 상태를 업데이트 합니다.
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // 에러 리포팅 서비스에 에러를 기록할 수도 있습니다.
    logErrorToMyService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      // 폴백 UI를 커스텀하여 렌더링할 수 있습니다.
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children;
  }
}
...
// 이런 식으로 사용 가능
<ErrorBoundary>
  <MyWidget />
</ErrorBoundary>
```

- 에러 경계는 자바스크립트의 catch {}와 유사하게 동작하지만 컴포넌트에 적용되며, 오직 클래스 텀포넌트만이 에러 경계가 될 수 있음
- 에러 경계는 트리 내에서 하위에 존재하는 컴포넌트의 에러만을 포착함. 에러 경계 자체적으로는 에러를 포착할 수 없음
- 손상된 UI를 그대로 남겨두는 것은 좋지 않다. 유저의 잘못된 행위를 유발할 수도 있기 때문

## 확장시키기

### Fallback UI 커스텀

- Fallback UI를 prop으로 받아 커스텀을 해줄 수 있다.

### Reset 문제

- ErrorBoundary 내부 상태에 hasError 값이 상태로 존재하기 때문에 다시 초기화해줄 인터페이스가 필요하다. 부모 컴포넌트를 다시 mount 시키지 않는 이상 ErrorBoundary에서 캡쳐된 에러는 다시 초기값으로 돌아가지 않는다.
- 에러가 발생했지만, 사용자의 인터렉션에 의해, 혹은 코드에 의해 다시 패칭이 진행될 경우 hasError는 다시 false로 돌아가야할 필요성이 생긴다.
- 다시 패칭을 할 수 있는 로직에서 사용할 수 있도록 ErrorBoundary의 인터페이스로 제공해주거나, 선언적으로 호출할 수 있는 인터페이스를 사용한다.
  - 선언적으로 처리하는 후자가 더 나은 듯 하다
  - React Query 에서는 쿼리키를 기반으로 캐싱 여부를 판단한다.

```jsx
interface Props {
  // ...
  resetKeys: unknown[]
}

// DidUpdate에서 이전과 이후의 key들을 비교한다
componentDidUpdate(prevProps: Props) {
  if (this.state.error == null) {
    return;
  }
  if (isDifferentArray(prevProps.resetKeys, this.props.resetKeys)) {
    // Trigger Reset
     this.setState({ hasError: false });
  }
}
```

## 확장된 컴포넌트

뭐 대충 요런 꼴로 사용할 수 있다.

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    logErrorToMyService(error, errorInfo);
  }

  componentDidUpdate(prevProps: Props) {
    if (!this.state.hasError) {
      return;
    }

    if (isDifferentArray(prevProps.resetKeys, this.props.resetKeys)) {
      this.setState({ hasError: false });
    }
  }

  render() {
    const { children, renderFallback } = this.props;
    const { hasError } = this.state;

    if (hasError) {
      return renderFallback;
    }

    return children;
  }
}
```

## 참고

- React 16부터는 에러 경계에서 포착되지 않은 에러로 인해 전체 React 컴포넌트 트리의 마운트가 해제된다.
- 에러 바운더리는 인베트 핸들러 내부에서는 에러를 포착하지 않는다. 이벤트 핸들러 함수에서 try/catch를 사용해바라
