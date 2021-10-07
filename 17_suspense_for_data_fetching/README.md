# Suspense for Data Fetching + 대수적 효과 살짝

데이터를 가져오기 위한 Suspense는 `<Suspense>`를 사용하여 선언적으로 데이터를 비롯한 **무엇이든** 기다릴 수 있도록 해주는 기능이다.

React 16.6부터 스크립트등의 동적 임포트와 연계하여서만 사용할 수 있었지만, React 18 부터는 데이터 패칭에도 사용될 수 있을 전망이다. 아직은 실험적인 기능

https://jbee.io/react/error-declarative-handling-1/
https://overreacted.io/algebraic-effects-for-the-rest-of-us/

## Suspense

### Suspense란

- 데이터 불러오기 라이브러리가 아니다. 데이터 불러오기 라이브러리에서 사용할 수 있는 매커니즘이다.
- 장기적인 관점으로 Suspense가 **데이터 출처와 상관없이** 컴포넌트로부터 비동기 데이터를 읽는 데에 사용되는 주된 방식으로 거듭나길 바란다고 한다.

### Suspense가 아닌 것

- 바로 사용할 수 있는 클라이언트가 아니다
- 데이터 불러오기 작업과 뷰레이어를 결합해주지 않는다. UI 상에 로딩 상태를 표시할 수 있도록 조정하는 것을 돕지만 이는 네트워크 로직을 React 컴포넌트에 종속시키는 것은 아니다.

### Suspense로 가능한 것

- 데이터 패칭 라이브러리들이 React와 깊게 결합할 수 있도록 해줌: React에서 사용하는 것이 자연스럽다고 느끼게 해준다.
- 의도적으로 설계된 로딩 상태를 조정할 수 있도록 해줌 : 데이터가 어떻게 불려와야 하는지를 정하지는 않고, 앱의 시각적인 로딩 단계를 밀접하게 통제할 수 있도록 해준다
- 경쟁 상태를 피함 : await을 사용하더라도 비동기 코드는 종종 오류가 발생하기 쉬움. Suspense를 사용하면 **데이터를 동기적으로 읽어오는 것처럼** 느끼게 해줌

### 기존 데이터 패칭과 Suspense의 관점 차이

```jsx
<ErrorBoundary fallback={<MyErrorPage />}>
  <Suspense fallback={<Loader />}>
    <FooBar />
  </Suspense>
</ErrorBoundary>
```

- 컴포넌트는 데이터 요청을 성공한 경우만 다루고, loading이나 error 상태의 경우는 다른 컴포넌트에 위임한다. 비동기 로직 여러개를 적당하게 처리해줄 수도 있다

### 다른 비동기 관리 방법과 비교

#### try-catch 명령형 처리 + 렌더링 직후 불러오기(useEffect)

컴포넌트의 상태로 데이터를 관리해야 하며 useEffect, useState등의 훅을 사용한다.

- useEffect를 사용하기 때문에 최초 렌더링 직후 불러오는 꼴이 된다
- waterfall : 한 데이터 페칭이 다른 페칭으로 가져와야 하는 데이터에 의존한다면, 첫번째 데이터를 패칭하고 응답을 받고 나서야 다른 데이터 패칭에 들어갈 수 있다.
- 라이브러리는 데이터를 불러오는 데 있어 보다 중앙화된 방식을 제공하는 것으로 워터폴을 방지할 수 있음 : 워터폴을 아예 방지하려면 데이터 패칭 라이브러리(Relay, Recoil 등등)의 힘이 필요하기는 하다
- Suspense는 `불러올 때 렌더링`이다 : 렌더링을 시작하기 전에 **응답이 오기를 기다리지 않아도 상관 없다.** 네트워크 요청을 발동시키고서, 바로 렌더링을 발동시키게 된다. 데이터의 예외 상황을 처리하는 로직을 컴포넌트 내부에 남겨놓지 않아도 비동기 데이터의 적절한 처리가 가능하다.
  - 비동기 처리를 포함하고 있지만, 응답이 오기를 따로 기다리지 않는다는 점에서 이 과정은 굉장히 **동기적**으로 처리된다
  - 예외 로직을 컴포넌트 안에 두는 것 자체가 응답이 오기를 기다리고 적절한 처리를 해주기 위해서이다. 근데 둘 필요 없으므로 **컴포넌트 내부만 보면 모든 것이 동기적으로 처리되는 느낌이 든다**
  - 불러오기 시작 -> 렌더링 시작 -> 불러오기 완료

```jsx
// fakeAPI.js

export function fetchProfileData() {
  let userPromise = fetchUser();
  let postsPromise = fetchPosts();
  return {
    user: wrapPromise(userPromise),
    posts: wrapPromise(postsPromise),
  };
}

// Suspense integrations like Relay implement 라고 한다.. GraphQL 공부도 언젠가 해봐야겠다..
function wrapPromise(promise) {
  let status = "pending";
  let result;
  let suspender = promise.then(
    (r) => {
      status = "success";
      result = r;
    },
    (e) => {
      status = "error";
      result = e;
    }
  );
  return {
    read() {
      if (status === "pending") {
        throw suspender; // Pending Promise를 던지면 Suspense?
      } else if (status === "error") {
        throw result; // Error을 던졌을때 ErrorBoundary
      } else if (status === "success") {
        return result; // 성공시 값을 리턴
      }
    },
  };
}

// 몇초 뒤 데이터가 리졸브되는 fake API
function fetchUser() {
  console.log("fetch user...");
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("fetched user");
      resolve({
        name: "Ringo Starr",
      });
    }, 1000);
  });
}

function fetchPosts() {
  console.log("fetch posts...");
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log("fetched posts");
      resolve([
        {
          id: 0,
          text: "I get by with a little help from my friends",
        },
        {
          id: 1,
          text: "I'd like to be under the sea in an octupus's garden",
        },
        {
          id: 2,
          text: "You got that sand all over your feet",
        },
      ]);
    }, 1100);
  });
}
```

```jsx
// Suspense

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
  // 아직 로딩이 완료되지 않았더라도, 사용자 정보 읽기를 시도합니다
  // 모든 것이 동기적으로 처리되는 느낌이 든다
  const user = resource.user.read();
  return <h1>{user.name}</h1>;
}

function ProfileTimeline() {
  // 아직 로딩이 완료되지 않았더라도, 게시글 읽기를 시도합니다
  const posts = resource.posts.read();
  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.text}</li>
      ))}
    </ul>
  );
}
```

- 데이터 페칭이 완료되지 않은 컴포넌트의 경우 컴포넌트는 정지한다. React는 이 컴포넌트를 넘겨버리고 다른 컴포넌트의 렌더링을 시도한다. 렌더링을 시도할 컴포넌트가 남아있지 않으면, 트리 상에서 존재하는 것 중 가장 가까운 Suspense Fallback을 찾는다.
- **데이터가 계속 흘러들어옴에 따라** React는 렌더링을 다시 시도하며, 그때마다 React가 더 깊은 곳 까지 처리할 수 있게 된다
- 응답이 계속 흘러들어오도록 하면 컨텐츠를 더 일찍 표시할 수 있게 된다 => 응답이 끝이 났을 때 명령적으로 어디에 넣어서 렌더링할 필요도 없다 컴포넌트가 그냥 정지되기 때문에 => 리액트의 UI 시스템과 비동기 처리가 잘 결합하고 있다

#### Hooks로 데이터 패칭, 처리 부분을 감싸기, Redux/Redux Saga 등의 중앙화된 솔루션으로 관리

- 이 과정에서 제일 귀찮은게, 로딩 상태인지 에러 상태인지를 매번 확인하고 정의해줘야 한다. 수많은 에러들을 각각의 컴포넌트, 혹은 hooks에서 처리해줘야 하기 때문에 손이 많이 간다.
- Redux Saga를 사용할 때는 에러가 났을때 적절한 처리를 통해 store을 업데이트 시켜줘야 한다. 그리고 이런 로직들은 몇 개를 제외하면 거의 달라지지 않는 경우가 많기때문에 수많은 비동기 요청에 전부 비슷한 에러 핸들링 처리 코드가 필요한지 의문이 될 수 있다.

### 참고

- 아직은 서버 사이드 렌더링 환경에서 지원하지 않는다. 대응하기 위해서 서버 사이드 환경에서는 전달받은 fallback 컴포넌트를 먼저 렌더링 할 수 있도록 확장하여 사용해야 한다.

```js
unction useMounted() {
  const [mounted, setMounted] = useState(false)

  useEffect(() => {
    setMounted(true)
  }, []) // mount 여부만 살짝

  return mounted
}

export default function SSRSafeSuspense(
  props: ComponentProps<typeof Suspense>
) {
  const isMounted = useMounted()

  // 이 상태값이 바뀌었을때 Suspense를 넣는다
  if (isMounted) {
    return <Suspense {...props} />
  }

  // 맨 처음에는 fallback UI만 렌더링하고
  return <>{props.fallback}</>
}
```

- 경쟁 상태를 해결하는데도 도움이 된다. 비동기 패칭을 너무 자주 실행하면 이전의 요청이 현재의 state으로 덮어 씌워지는 경쟁 상태가 발생할 수도 있는데, Suspense는 비동기 요청 시간에 대한 것을 그다지 고려하지 않아도 된다. Suspense를 사용할때는 데이터가 흘러들어올때 즉시 state를 설정하기 때문에 => 이거는 개발을 해봐야 좀 와닿을 듯
- ErrorBoundary와 함께 쓰면 로딩상태와 에러처리 모두 선언적으로 처리할 수 있다. ErrorBoundary와 Supense를 다 붙여주는 래퍼 컴포넌트가 있으면 좋을 것 같기도 하다

## 더 자세한 배경 지식

### 대수적 효과

https://overreacted.io/ko/algebraic-effects-for-the-rest-of-us/  
Dan쨩... 그는 너무 똑똑해...

- 연구중인 프로그래밍 언어 기능이다. 현재 사용할 수 없는 경우가 더 많다
- ES2025(ㅋㅋ)를 가정하고 댄이 만든 대수적 효과 예제

```js
function getName(user) {
  let name = user.name;
  if (name === null) {
    // 2. 여기서 효과를 수행
    name = perform 'ask_name'; // !
    // 5. 그리고 여기로 돌아옴, name값은 핸들 블락에서 넣은 Arya Stark
  }
  return name;
}

function makeFriends(user1, user2) {
  user1.friendNames.add(getName(user2));
  user2.friendNames.add(getName(user1));
}

const arya = { name: null };
const gendry = { name: 'Gendry' };
try {
  // 1. 함수 실행
  makeFriends(arya, gendry);
} handle (effect) {
  // 3. 효과를 수행하면 Handle 구문으로
  if (effect === 'ask_name') {
    // 4. try, catch와는 다르게 값을 전달하면서 기존 try문 내부의 코드를 이어서 실행
    resume with 'Arya Stark';
  }
}
```

- 원래는 에러가 발생하면 코드 블럭은 이어서 진행될 수 없다. catch 블럭에 도달하면 원래 코드를 이어서 실행될 수 없고 exit된다. 대수적 효과는 이걸 가능케 한다.
- 에러를 던지는 대신, 효과를 수행한다 (perform the effect).
- resume 구문은 효과가 수행된 곳으로 다시 돌아갈 수 있고, 핸들러를 통해 무언가 전달할 수 있다. => 재시작할 수 잇는 try catch
- 우리가 에러를 throw할 때, 자바스크립트 엔진은 프로세스에 있는 지역 변수들을 제거하면서 “스택을 해제”한다. 하지만 효과를 perform할 때, 가상의 엔진은 함수의 나머지 부분들로 콜백 함수를 만들어낸 다음, resume with를 통해 그 함수를 호출한다.
- **대수적 효과는 무엇(what - try)과 어떻게(how - handle)를 분리할 수 있게 해주는 강력한 도구가 될 수 있다.** => 라인의 순서에 상관 없이, 두 블락(try, handle) 의순서를 겁나 왔다갔다하면서 필요한 부분의 처리를 알아서 끝내기 때문이다. 이해하기가 어렵긴 하지만 익숙해진다면 코드의 응집도를 상승시킬 수 있는 부분이다.

```js
function enumerateFiles(dir) { // 무엇
  const contents = perform OpenDirectory(dir);
  perform Log('Enumerating files in ', dir);
  for (let file of contents.files) {
    perform HandleFile(file);
  }
  perform Log('Enumerating subdirectories in ', dir);
  for (let directory of contents.dir) {
    // 재귀를 사용하거나 효과를 갖는 다른 함수를 호출할 수도 있다
    enumerateFiles(directory);
  }
  perform Log('Done');}
}

let files = [];

try {
  enumerateFiles('C:\\'); // what
} handle (effect) { // how
  // enumerateFiles 로직의 effect 타입에 따라 적당한 값을 주입하고 resume 시킨다
  if (effect instanceof Log) {
    myLoggingLibrary.log(effect.message);
    resume;
  } else if (effect instanceof OpenDirectory) {
    myFileSystemImpl.openDir(effect.dirName, (contents) => {
      resume with contents;
    });
  } else if (effect instanceof HandleFile) {
    files.push(effect.fileName);
    resume;
  }
}

// 이런 식으로 라이브러리화도 가능하다
// 꼴이 낯이 익다!
import { withMyLoggingLibrary } from 'my-log';
import { withMyFileSystem } from 'my-fs';

function ourProgram() {
  enumerateFiles('C:\\');
}

withMyLoggingLibrary(() => { // try문에서 실행할 하나의 로직에 대한 적절한 effect 처리
  withMyFileSystem(() => {
    ourProgram();
  });
});

// 아마 대충 이런 구현체겠지
function withMyFileSystem(event) {
  try {
    event()
  } handle(effect) {
    if (effect instanceof Log) {
      // do something...
      resume 'log event';
    } else if (effect instanceof OpenDirectory) {
      // do something...
      resume 'open dir event';
    } else if (effect instanceof HandleFile) {
     // do something...
      resume 'handle file event';
    }
  }
}
```

- 중간에 있는 코드는(try문에 있는 로직, 걔는 그냥 effect를 던지기만 할 뿐이다) 효과의 처리에 대해 알 필요가 없다는 점, 효과 핸들러를 중첩해서 사용할 수 있다는 점 때문에 대수적 효과를 이용하면 표현력이 아주 뛰어난 추상화를 만들어낼 수 있다.

### 이게 React와 무슨 상관인가?

#### Suspense

- 위에서 다뤘던 React Docs의 Suspense 예제의 read의 내부 구현은 이렇다

```jsx
 read() {
  if (status === "pending") {
    throw suspender; // Pending Promise를 던지면 Suspense
  } else if (status === "error") {
    throw result; // Error을 던졌을때 ErrorBoundary
  } else if (status === "success") {
    return result; // 성공시 값을 리턴
  }
},
```

- Suspense를 사용하면 React는 Pending Promise를 잡아다가 그 프로미스가 resolve 되었을때 컴포넌트를 다시 렌더링 하기위해 기억해둔다.
- 이러한 기법은 대수적 효과에서 착안한 것이다. ( 그 자체로 대수적 효과라고 할 수는 없지만) 동일한 목적을 갖는다. 호출 스택(리액트)의 중간에 있는 함수들을 신경 쓸 필요가 없고, 그 함수들을 async나 제네레이터 함수 등으로 강제로 변경할 필요 없이, **호출 스택의 아래쪽에 있는 코드가 윗쪽에 있는 코드에게 무언가를 전달할 수가 있다.** => 새로운 방법의 DI가 가능한 것
- 리액트의 입장에서 프로미스가 완결되었을 때(어떤 효과가 발생했을 때) 컴포넌트 트리를 다시 렌더링하는게 사실상 비슷한 개념이다.

#### Hooks

- React 객체에는 바로 지금 사용되고 있는 구현체를 가리키는 "현재 디스패치"라는 변경 가능한 상태가 있으며, "현재 컴포넌트"라는 속성도 있다. 이를 통해 useState가 무엇을 할지 알고 있다.
- 개념적으로 useState를 컴포넌트가 실행될 때 리액트에 의해 처리되는 perform State() 효과라고 넓게 볼 수 있다.

### Generator?

재미삼아서 함 해본 하찮은 실험이다. 제네레이터로 대수적 효과를 비슷하게 구현해보면 어떨까?

- what과 how의 로직 분리는 얼추 가능하다
- 대수적 효과처럼 쓰기에는 인터페이스가 별로다(next를 거듭 호출해줘야 한다거나)
- 명시적으로 제네레이터를 다 사용했으면 해제를 해줘야 한다
- 함수의 호출 방향이 반대다(resume하는 함수가 perform하는 함수를 감싸야 하는데 이거는 반대로 resume하는 제네레이터를 perform하는 함수가 감싼다) 그렇게 되면, 아래와 같이 사용 못하기 때문에 대수적 효과에 입각한 추상화가 불가능하다.

```js
withMyLoggingLibrary(() => {
  // resume
  withMyFileSystem(() => {
    // resume
    ourProgram(); // perform
  });
});
```

```js
// how => next로 전달한 effect를 적절하게 처리 => resume하는 곳
function* processEffect() {
  while (true) {
    const effect = yield null;
    if (effect === "getName") {
      yield "Arya Stark"; // resume
    } else if (effect === "getAge") {
      yield 17;
    }
  }
}

// what => effect를 던진다 => perform하는 곳
const throwEffect = (generator) => {
  const gen = generator();
  gen.next();
  const name = gen.next("getName").value; // perform
  gen.next();
  const age = gen.next("getAge").value;
  console.log(name, age);
  gen.return();

  return `${name}, age ${age}`;
};

// 이거 때문에 대수적 효과가 못 된다
console.log(throwEffect(processEffect));
```

근데 이미 있넹 https://dev.to/yelouafi/algebraic-effects-in-javascript-part-4---implementing-algebraic-effects-and-handlers-2703 ㅋㅋㅋㅋㅋ

### Suspense의 원리를 엿볼 수 있는 구현체

Suspense를 창안한 Sebastian Markbåge의 코드 조각이다. Suspense는 대수적 효과에서 착안한 것이다.

```js
let cache = new Map();
let pending = new Map();

//! Resume 하는 로직이다 - what
function fetchTextSync(url) {
  if (cache.has(url)) {
    return cache.get(url); // 캐시 맵객체
  }
  if (pending.has(url)) {
    throw pending.get(url); // Pending Promise throw
  }
  // 비동기 로직
  let promise = fetch(url)
    .then((response) => response.text()) // 처리되는 경우
    .then((text) => {
      pending.delete(url);
      cache.set(url, text);
    });
  pending.set(url, promise); // 팬딩 객체에 팬딩인거 표시
  throw promise;
}

// perform하는 로직 - how
async function runPureTask(task) {
  for (;;) {
    // 태스크를 리턴할 수 있을 때까지 바쁜대기를 함
    try {
      return task();
    } catch (x) {
      if (x instanceof Promise) {
        await x; // pending promise가 throw된 경우 await으로 resolve 시도
      } else {
        throw x; // Error가 throw된 경우 그대로 error throw
      }
    }
  }
}
```

```js
function getUserName(id) {
  var user = JSON.parse(fetchTextSync("/users/" + id)); // 비동기 로직
  return user.name;
}

function getGreeting(name) {
  if (name === "Seb") {
    return "Hey";
  }
  return fetchTextSync("/greeting"); // 비동기 로직
}

function getMessage() {
  let name = getUserName(123);
  return getGreeting(name) + ", " + name + "!";
}

runPureTask(getMessage).then((message) => console.log(message));
```

## 대수적 효과 recap

https://www.youtube.com/watch?v=FvRtoViujGg

- 토스 : 코드 조각을 감싸는 맥락으로 책임을 분리하는 방식을 대수적 효과라고 함
- 대수적 효과를 사용하면 필요한 함수를 선언적으로 사용할 수 있다.
- 대수적 효과란 콜스택에 먼저 쌓인 함수가 그 뒤에 쌓인 함수와 상호작용하며 값을 처리하는, 연구중인 프로그래밍 기능이다.
- 콜스택에 먼저 쌓이는 함수에 콜백 함수의 로직을 분리하고 함수의 로직을 추상화하고 책임을 분리할 수 있다.
- 원래는 catch에서 무언가 throw되었다면 콜스택을 타고 모든 함수의 호출을 끝내고 에러를 뱉는다. 대수적 효과를 사용하면 catch에서 뭔가 throw하거나 값을 낸 이후에 다시 돌아가 로직을 계속 진행시킬 수 있다.
- React의 Suspense, Hooks가 어느 정도는 이런 대수적 효과를 활용한 개념들이라고 할 수 있다.
