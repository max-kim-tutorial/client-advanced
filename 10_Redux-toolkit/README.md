# Redux Toolkit CreateSlice

그리고 Saga랑 쓰는 방법...연구하기

## CreateSlice

- 한 객체에 reducer, action을 때려박는 덕스패턴을 지원
- **reducer에 작성한 로직을 따라 액션을 자동으로 생성하는 미친친구**
- 주로 기능 단위로 Redux를 잘게 나누어 사용하면서 타이핑을 미친듯이 줄여주기 때문에 꼭 사용해야할 기능인거같음...
- reducer로 생성하는 모든 액션은 문자열이다. 일반적이군
- Immer가 있어서 불변성 신경쓸 필요가 없다

## TypeScript와 같이 쓰기

- slice의 프로퍼티들은 typesafe하기 때문에 일단은 신경쓸 필요가 없다. reducer함수에서 action payload 정도만 toolkit에서 제공하는 PayloadAction 타입을 사용해 명시해주면 문법적으로 문제는 없다.
- 그치만
  1. state의 타입은 명시적으로 선언해 놓는게 좋은거같다
  2. Saga를 쓸때 fetch 액션의 경우 action payload를 꼭 reducer 함수에서 사용하지 않을 수도 있는데, 이럴때 함수의 인자로 action을 선언했어도 reducer 함수의 body에서는 action을 사용하지 않아 좀 어색하다(eslint 에러가 뜰 수 있다)
- redux toolkit에 내장된 thunk를 쓰면 함수 action을 dispatch할 수 있어 fetch 액션이 따로 필요하지 않긴 하다...(SUCCESS, FAILURE만 쓰면 됨) 내장되있는거 안쓰려니 좀 엇박자 나는듯ㅋㅋㅋ
- thunk : 디스패치한 함수가 실제 저장소에 도달하는 것을 막은 다음에 미들웨어 단에서 함수액션을 호출하고 다른 액션도 dispatch하는 식으로 처리. 하나의 함수 액션에서 여러 액션 dispatch를 포함할 수 있는 꼴이라 action creator들이 일관성이 없어지고 액션 자체가 순수하지 못하게 된다는게 비판점인듯. (하나의 액션은 하나 단위의 상태를 변화시켜야 하는데,,) 그리고 Saga가 더 할 수 있는게 많음

```ts
// thunk 예제
import { createSlice, PayloadAction } from '@reduxjs/toolkit'

import { AppThunk } from 'app/store'

import { RepoDetails, getRepoDetails } from 'api/githubAPI'

interface RepoDetailsState {
  openIssuesCount: number
  error: string | null
}

const initialState: RepoDetailsState = {
  openIssuesCount: -1,
  error: null
}

const repoDetails = createSlice({
  name: 'repoDetails',
  initialState,
  reducers: {
    getRepoDetailsSuccess(state, action: PayloadAction<RepoDetails>) {
      state.openIssuesCount = action.payload.open_issues_count
      state.error = null
    },
    getRepoDetailsFailed(state, action: PayloadAction<string>) {
      state.openIssuesCount = -1
      state.error = action.payload
    }
  }
})

export const {
  getRepoDetailsSuccess,
  getRepoDetailsFailed
} = repoDetails.actions

export default repoDetails.reducer

export const fetchIssuesCount = (
  org: string,
  repo: string
): AppThunk => async dispatch => {
  try {
    const repoDetails = await getRepoDetails(org, repo)
    dispatch(getRepoDetailsSuccess(repoDetails))
  } catch (err) {
    dispatch(getRepoDetailsFailed(err.toString()))
  }
}
```

### state 타입 명시적으로 선언하기

별거 없다 그냥 위에서 선언하고 아래에서 참조하면 됨

```tsx
type SliceState = { state: 'loading' } | { state: 'finished'; data: string }

// First approach: define the initial state using that type
const initialState: SliceState = { state: 'loading' }

createSlice({
  name: 'test1',
  initialState, // type SliceState is inferred for the state of the slice
  reducers: {}
})
```

### Action Payload를 reducer에서 쓰지 않는 경우(Saga를 트리거하는 fetch action)


## extraReducer는 언제 쓰면 좋을까?

## Saga에서 쓰기(.type)

## reducer 타이핑 줄이기(fetch, success, fail 액션 자동생성하기)

약간 typesafe actions같은게 필요함