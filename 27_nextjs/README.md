# Next.js 리마인드

완전 대세가 되버린...  
당장 필요한 개념만 쏙쏙 빼먹어 보쟈

## Pages

- 페이지 디렉토리 기반으로 라우팅을 한다
- 다이나믹 라우팅은 `pages/posts/[id].js` 이런식으로 한다
- Next는 최소한의 자바스크립트만 가진 상태로 모든 페이지를 프리렌더링한다. 
- static generation과 serverside rendering 둘다 가능하다. 정적 사이트 생성이 더 빠르다. 유저 리퀘스트보다 더 빨리 사이트를 렌더링할 수 있다면(의존하지 않는다면) 정적 사이트 쪽이 더 낫다. 그리고 정적 사이트 HTML은 캐싱된다.
- SSR은 사용자 요청이 들어올때 HTML을 만들어 보내줄 수 있다. 자원 내용이 달라질 수 있어서 이건 캐싱이 안된다(같으면..?)
  - getServerSideProps : export, async를 써야하는 Next 고유의 함수로 컴포넌트와 같이 선언해서 프리랜더링 되는 HTML을 채우는데 필요한 데이터 패칭 동작 등 프리랜더링 이전의 컴포넌트 동작을 안에다가 작성할 수 있다. 페이지 디렉토리 내부의 컴포넌트에서만 되는 듯 하다. 해당 함수 안에서 html req, query 등등을 파싱할수도 있다.
  ```jsx
  // 클래스 컴포넌트때에는 생명주기처럼 다뤘는데 함수형은 함수를 하나 더 만드는 늒임
  function Page({ data }) {
    // Render data...
  }

  // This gets called on every request
  // 위에다가? 밑에다가?
  export async function getServerSideProps() {
    // Fetch data from external API
    const res = await fetch(`https://.../data`)
    const data = await res.json()

    // Pass data to the page via props
    return { props: { data } }
  }

  export default Page
  ```
  - 안해봐서 모르겠는데 SWR 등으로 getServerSideProps에서 먼저 패칭하고+캐싱한 후에 하위 컴포넌트에서 참조만 하면 렌더링 시간을 줄일 수 있지 않을까...? 아 근데 이런 경우에는 서버에서 동작하는 코드니까 사실은 SWR로 패칭을 먼저 해도 클라이언트에서는 패칭 했는지 어땠는지 모르겠네..
  - 페이지 프리 렌더링할 때 꼭 들어가야 하는 데이터가 있을때 사용하라고 만들어놓은 느낌?(메타태그라던가...)
  - 프리랜더링된 HTML은 최초 URL로 접속했을때 제공됨 그 이후에는 CSR
- 레이아웃 : _APP에다가 작성하면 쉽게 적용 가능

## Head

custom head 태그 사용하기

## Routing

- 라우터의 파람, 쿼리 가져오기

```jsx
import { useRouter } from 'next/router'

const Post = () => {
  const router = useRouter()
  const { pid } = router.query

  return <p>Post: {pid}</p>
}

export default Post
```

- 클라이언트 사이드 라우팅 : 요청한 프리랜더 HTML 먼저 다 받아온 이후에는 클라이언트 사이드에서 라우팅
  - 선언형 : Link
  ```jsx
    import Link from 'next/link'

    function Home() {
      return (
        <ul>
          <li>
            <Link href="/">
              <a>Home</a>
            </Link>
          </li>
          <li>
            <Link href="/about">
              <a>About Us</a>
            </Link>
          </li>
          <li>
            <Link href="/blog/hello-world">
              <a>Blog Post</a>
            </Link>
          </li>
        </ul>
      )
    }

    export default Home
  ```
  - 명령형 : router.push
  ```jsx
      import { useRouter } from 'next/router'

      export default function ReadMore() {
      const router = useRouter()

      return (
        <button onClick={() => router.push('/about')}>
          Click here to read more
        </button>
      )
    }
  ```

## ENV

Next.js allows you to set defaults in .env (all environments), .env.development (development environment), and .env.production (production environment). 이렇게 하면 댈듯

## 부가기능

### next/head

- 페이지별 해드 테그 커스텀

```jsx
import Head from 'next/head'

function IndexPage() {
  return (
    <div>
      <Head>
        <title>My page title</title>
        <meta name="viewport" content="initial-scale=1.0, width=device-width" />
      </Head>
      <p>Hello world!</p>
    </div>
  )
}

export default IndexPage
```

- external data와 관계 없는 경우(caching header라던지) next.config.js에서 작성 : 기존 SPA와는 다르게 인프라단을 만질 필요가 없어진다 넘나 유용...!

```jsx
module.exports = {
  async headers() {
    return [
      {
        source: '/about', // 여기다가 확장자로 자원을 명시해줄 수 있다 '/:all*(svg|jpg|png)', 이렇게 한다든지
        headers: [
          {
            key: 'x-custom-header',
            value: 'my custom header value',
          },
          {
            key: 'x-another-custom-header',
            value: 'my other custom header value',
          },
        ],
      },
    ]
  },
}
```

## 더 궁금한 것들

이정도면 온보드 하는데는 대충 문제가 없을 것 같긴한데, 궁금한게 있다면

- 라우팅간 번들 요청응답 및 실행 동작 : 첫 페이지 요청에 프리랜더링된 HTML 받아오고 그 이후에는 SPA처럼 동작하는데 그 이후에, 다른 페이지로 클라이언트 사이드 라우팅이 되었을때는 또다른 JS 요청이 어떤 방식으로 발생하는지, 어떤 번들을 어떤 상황에 가져오는지, 그런 번들들은 따로 캐싱할 수 없는지
  - 대충 웹팩 config 정도는 알고있긴 함
- 빌드파일의 구조 분석
- getServerSideProps의 활용 방안
- 서버 커스텀이 필요한 경우
- 웹팩이나 바벨 커스텀이 필요한 경우
- SWR, React Query등의 SSR 지원이 어떠한 방식으로 이뤄지는가
- CSS-in-JS가 핸들하는 critical css 문제