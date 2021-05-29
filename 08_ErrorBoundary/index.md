# React에서의 에러 처리

UI의 일부분의 존재하는 자바스크립트 에러가 전체 애플리케이션을 중단시켜서는 안된다  
깨진 UI를 보여주지 말아라

## React Error Boundary

- 하위 컴포넌트 트리의 어디에서든 자바스크립트 에러를 기록하며 깨진 컴포넌트 트리 대신 fallback UI를 보여주는 React 컴포너트
- 생명주기 메서드인 `static getDerivedStateFromError` 와 `componentDidCatch()`중 하나를 정의하면 클래스 컴포넌트 자체가 에러 경계가 됨

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

## 선언형 비동기 처리 (데이터를 가져오기 위한 Suspense)

-  16.6에서는 코드를 불러오는 동안 기다릴 수 있고, 기다리는 동안 로딩 상태를 선언적으로 지정할 수 있도록 <Suspense>가 추가됨(코드 스플리팅 관점에서)
- 데이터를 가져오기 위한 Suspense는 Suspense를 사용하여 선언적으로 데이터를 비롯한 무엇이든 기다릴 수 있도록 해주는 기능. 사용 사례 가운데 데이터 로딩에 초점을 두지만, 이 기능은 이미지, 스크립트, 그 밖의 비동기 작업을 기다리는 데에도 사용할 수 있다

```jsx
const resource = fetchProfileData();

function ProfilePage() {
  return (
    <Suspense fallback={<h1>Loading profile...</h1>}>
      <ProfileDetails />
      <Suspense fallback={<h1>Loading posts...</h1>}>
        <ProfileTimeline />
      </Suspense>
    </Suspense>
  );
}

function ProfileDetails() {
  // 비록 아직 불러오기가 완료되지 않았겠지만, 사용자 정보 읽기를 시도합니다
  const user = resource.user.read();
  return <h1>{user.name}</h1>;
}

function ProfileTimeline() {
  // 비록 아직 불러오기가 완료되지 않았겠지만, 게시글 읽기를 시도합니다
  const posts = resource.posts.read();
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.text}</li>
      ))}
    </ul>
  );
}
```
- 불러올 때 렌더링 (예를 들어, Suspense와 함께 Relay 사용): 최대한 일찍 다음 화면에서 필요한 데이터 불러오기를 시작하고, 다음 화면 렌더링을 네트워크 응답을 받기 전에 즉시 시작합니다. 데이터가 흘러들어옴에 따라, React는 **모든 데이터가 준비될 때까지 데이터를 필요로 하는 컴포넌트의 렌더링을 다시 시도합니다.** => 데이터가 없을때(거짓값일때?) fallback을 띄워준다
- 데이터가 계속 흘러들어옴에 따라 React는 렌더링을 다시 시도하고, 그대마다 더 깊은 곳까지 처리할 수 있게 됨
- loading state를 평가할 필요도 없어진다

```jsx
// 이것은 프라미스가 아닙니다. Suspense 통합에서 만들어낸 특별한 객체입니다.
const resource = fetchProfileData(); // 가장 빨리 시작한다는 점에서 의미가 있을수도

function ProfilePage() {
  return (
    <Suspense fallback={<h1>Loading profile...</h1>}>
      <ProfileDetails />
      <Suspense fallback={<h1>Loading posts...</h1>}>
        <ProfileTimeline />
      </Suspense>
    </Suspense>
  );
}

function ProfileDetails() {
  // 아직 로딩이 완료되지 않았더라도, 사용자 정보 읽기를 시도합니다
  const user = resource.user.read();
  return <h1>{user.name}</h1>;
}

function ProfileTimeline() {
  // 아직 로딩이 완료되지 않았더라도, 게시글 읽기를 시도합니다
  const posts = resource.posts.read();
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.text}</li>
      ))}
    </ul>
  );
}
```

- 장기적 관점으로 Suspense가 데이터 출처와 상관없이 컴포넌트로부터 비동기 데이터를 읽는 데에 사용되는 주된 방식으로 거듭나길 바라고 있는 React 팀
- suspense는 불러오기에 대한 구현체가 아니며, 특정 전송 프로토콜을 사용한다고 가정하지 않는다.
- suspense는 데이터 불러오기 작업과 뷰 레이터를 결합해주지 않는다 : UI 상에 로딩 상태를 표시할 수 있도록 조정하는 것을 돕지만, 네트워크 로직을 React에 종속시키는 것은 아니다(???뭔가 컴포넌트에 바인딩하는 느낌인데 아닌가)
- **경쟁 상태를 피할 수 있도록 돕는다** : await을 사용하더라도 비동기 코드는 종종 오류가 발생하기 쉽다. Suspense를 사용하면 데이터를 동기적으로 읽어오는 것처럼 느껴지게 해줌. 마치 이미 불러오기가 완료된 것처럼...
- 경쟁 상태를 불러오는 것 : 코드가 실행되는 순서에 대한 부정확한 가정. useEffect나 componentDidUpdate와 같은 생명주기 메서드 내에서 데이터를 불러오면 경쟁 상태가 생길 수 있음. 어쩔때는 오고 어쩔때는 안오고 그렇거나... 

- 에러 처리 : 여기서도 에러 경계 사용하면 좋음

```jsx
// 함수형 컴포넌트에서 에러바운더리 하는게 아직은 없다고 어디서 본듯..??

class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };
  static getDerivedStateFromError(error) {
    return {
      hasError: true,
      error
    };
  }
  render() {
    if (this.state.hasError) {
      return this.props.fallback; // fallback을 대신 보여줌
    }
    return this.props.children;
  }
}

 <ErrorBoundary fallback={<h2>Could not fetch posts.</h2>}>
    <Suspense fallback={<h1>Loading posts...</h1>}>
      <ProfileTimeline />
    </Suspense>
  </ErrorBoundary>
```
