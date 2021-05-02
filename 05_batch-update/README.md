# React Batched Update

생명주기 함수에서의 state 업데이트, 혹은 이벤트 핸들러 함수 내부의 연달은 state 업데이트는 batch되어 렌더링은 한번만 일어나게끔 설게되었다. 로직만 좀 달리 짜도 render 함수 호출을 줄일 수 있다는 말임

## unstable_batchedUpdates()

- 리액트가 내부적으로 자체 이벤트에 batched update를 지원할 수 있는 것은 이 API 덕분으로, 이로 인해 리액트에서 한 이벤트 핸들러에서 발생하는 모든 업데이트(함수 단위)를 단일 렌더 패스로 일괄 처리 가능. => 불필요한 렌더 함수 호출이 줄어든다.
- state가 바뀔 때마다, setState가 호출 될때마다 렌더링을 해준다고 생각할 수 있지만 어느정도의 최적화 로직이 렌더링 과정에도 있다.
- 초기 렌더링을 제외하면, useEffect 안에서도 동작한다(!!)

## redux 안에서도 동작함

- 한개의 이벤트에 값을 set하는 다수의 액션을 dispatch하는 경우, 이때 스토어의 각각의 값은 상응하는 액션별로 업데이트되지만, 이 해당되는 값을 구독하는 컴포넌트에서는 리액트의 내부적인 batched update 로직에 따라 props를 모아서 받고, 이로 인해 비효율적인 렌더 함수의 호출을 줄일 수 있음
- 2개의 액션을 연달아 디스패치 시킬때, 콘솔로그를 찍어보면 액션의 순서대로 렌더링이 3번 일어날 것 같지만, 2번만 일어난다
- 리덕스 자체는 예상대로 동작하고 prop도 순차적으로 넘기지만, react 내부의 unstable_batchedUpdates()를 사용하여 렌더링을 한번 줄인다.

## 모든 경우에 작동하는 것은 아니다

- 함수 안에서 async await, setTimeOut 등 비동기 처리를 사용하며 동시에 state를 업데이트할 경우 동작하지 않는다.
- However, if we use setState() within setTimeout(), async, and Promise, we will not be able to setup ReactDefaultBatchingStrategy.isBatchingUpdates as true. Instead, ReactUpdateQueue.enqueueSetState() is called, which sets isBatchingUpdates to true but calls transaction.perform() directly. In such case, each setState() is handled independently and triggers a re-render.
- 이런 경우는 명시적으로 `ReactDOM.unstable_batchUpdate`를 사용해 업데이트할 state들의 실행문들을 묶어주면 가능하다.

## 대충의 동작

React는 몇가지 시나리오를 가지고 batch를 다르게 처리한다

> - 생명주기 함수 내부에서
> - 리덕스랑 붙어있는 경우
> - 이벤트 핸들러 함수

- [시간이 날때 이것도 정리해보자](https://gaopinghuang0.github.io/2020/12/21/react-batched-update)

## 왜 unstable인가?

- 아브라모브찡이 17이후 언젠가는 리액트가 모든 업데이트를 배치로 처리할 예정이라 이 API자체는 없어질 예정이라고 말함

## reference

- [React batched update 그리고 react-redux](https://medium.com/tapjoykorea/react-batched-update-%EA%B7%B8%EB%A6%AC%EA%B3%A0-react-redux-connect-%ED%95%A8%EC%88%98-c9779e8db1fe)
- [React State Batch Update](https://medium.com/swlh/react-state-batch-update-b1b61bd28cd2)
