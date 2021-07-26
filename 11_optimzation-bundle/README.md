# Bundle Optimization With Webpack

더 작고 더 빠르게!!

## 번들러(webpack)를 쓰는 이유

- 브라우저에서는 node처럼 모듈 구문(AMD, commonjs, ES6, script module)을 제대로 지원하지 않기 때문에 개발한 JS파일을 묶어 번들링을 하고, HTML에 붙여서 서빙해야한다. 모듈 관련한 문법들은 웹팩에서 지원한다.
- 브라우저에서 자원 요청의 수를 최대한으로 줄여야하므로 `<script></script>`를 몇줄이나 쓰는 것 보다 한번에 묶어서, 한번에 묶기 너무 크다면 몇 개의 조각으로 나누어 번들링을 하는게 더 경제적이다.
- 패키지간의 의존 관계 해결 : 하나의 JS 파일로 복잡한 여러 개의 의존성을 하나로 합쳐줌. 
- 로더 : JS 패키지 뿐 아니라 스타일이나 이미지 같은 것들도 쉽게 빌드 결과물에 포함되도록 해준다.

## 번들을 줄여야 하는 이유

- 브라우저에서 필요한 자원을 내려받는 네트워크 비용은 꽤 비쌈. 그중에서도 JS 번들 처리에 필요한 비용이 꽤 큼. 이미지같은 자원은 로딩만 하면 되지만, JS 번들같은 경우는 내려받고, 컴파일하고, 실행하는 과정까지 여러 과정을 거쳐야 해서 웹 다운로드 성능에 가장 큰 영향을 미치는 요소임
- 유저는 기다려주지 않음. 유저가 최대한 빨리 웹앱과 인터렉션 할 수 있도록 JS 번들을 내려받고 실행하는 시간을 최대한 줄이는 것이 비즈니스에 긍정적인 영향을 끼칠 수 있음.

## 번들 줄이는건 알겠는데 큰 번들을 여러개로 나누는게 도움이 되나?

- 브라우저마다 다르긴 한데, 기본적으로 concurrent한 network call을 보낼 수 있다. 크롬은 한번에 6번으로 제한되어 있음
- 무제한으로 보낼 수 있었다면 서버에 부담을 줄 가능성이 높고, 브라우저를 통해서 디도스 공격을 해버리는(...) 악용도 발생할 수 있음
- 일단 병렬 네트워크 콜이 가능하기 때문에 하나의 커다란 번들을 작은 여러개로 나누는게 네트워크 비용에 이점을 가져올 수 있다(고 말할 수 있는게 맞나..?)
- 더 자세히는 브라우저 단 최적화 문서에 정리하자


## 최적화 방법론

### 1. bundle analyzing

- webpack bundle analyzer로 report를 떠보면 어떤 모듈이 전체 번들에서 얼마나 차지하는지 알 수 있음
- 비정상적으로, 유난히 큰 모듈이 있을 경우 프로덕트에서 실제로 사용하고 있는 코드만 import하고 있는지 살펴볼 필요가 있음
- 사용하지 않는 기능이 많이 포함된 라이브러리 사용은 지양한다

```js
// 전체를 임포트하고 있다면 그만큼 끼워넣어지는 모듈의 크기도 커짐
import * as _ from "lodash";

// 필요한 것만 임포트하기
import { find } from "lodash";
```

### 2. 라이브러리 크기 자체를 줄이기

#### 라이브러리 압축을 지원하는 웹팩 플러그인 적용(Lodash, moment)

- 필요한 메소드만 Import해서 사용해도 라이브러리 전체가 번들에 들어가있는 경우가 생길 수 있음
  - 왜 이런 경우가 생기나? : https://www.blazemeter.com/blog/the-correct-way-to-import-lodash-libraries-a-benchmark 라이브러리에 따라 좀 다른듯..
- lodashModuleReplacementPlugin
- plugin이 있는 라이브러리의 경우 적용하면 되고, 아니면 다른 경량 라이브러리를 고려해보는 게 좋을 듯 싶음

#### node_modules의 늪과 dependency conflict

- 각각 다른 라이브러리에서 같은 라이브러리에 의존하고 있지만 버전이 다른 경우가 있음 : dependency conflict
- npm에서는 tree구조로 필요한 버전을 모두 받음 : 서드파티 라이브러리들에 package.json있으니까 그거 가지고 node_modules를 만듬(디펜던시의 디펜던시) 상황에 맞게 적합한 폴더를 선택함 => 과도하게 큰 용량
- npm dedupe : 설치 과정에서 놓친 중복 라이브러리의 버전 충돌을 해결
- yarn에서는 알아서 잘 해준다고는 하는데 yarn2에서는 dedupe이 생겼음
- package.lock을 보면 어떤 라이브러리가 중복되었는지(버전까지) 확실히 알 수 잇음

#### 모듈간 중복되는 의존성은 어떻게 해결?(lodash)

- webpack의 alias를 통해 각 써드파티 라이브러리에서 사용되는 다양한 lodash, lodash/세부기능, lodash-es 같은 친구들을 lodash 하나만 참조하도록 설정이 가능하고 이렇게하면 불필요한 라이브러리가 번들링 되는 것을 막을 수 있음
- lodash 관련한 babel plugin도 있음
- lodash는 소잡는 칼로 닭잡는 꼴이 될 수 있음. 네이티브 함수를 쓰거나 할 수 있으면 그렇게 하는게 더 좋을지도

#### 더 작은 라이브러리를 찾기

- [bundle phobia](https://bundlephobia.com/)

### 3. Minify

[minifier 비교](https://blog.logrocket.com/terser-vs-uglify-vs-babel-minify-comparing-javascript-minifiers/)

- minification : 코드에서 실행 자체에는 불필요한 요소들을 제거하여 코드를 난독화하고 압축하는 과정
- 소스코드의 용량을 줄일 수 있어 네트워크를 좀 더 효율적으로 사용할 수 있게 한다
- 잘 알려진 툴들
  - Terser : 가장 유명하다. Rollup이랑 같이 쓰는게 제일 효율이 좋다. webpack에는 4이상이면 terser가 내장되어 있다. 

```js
module.exports = {
  //...
  optimization: {
    minimize: false // 이렇게 끌 수 있음 default가 true
  }
};

// terser 옵션은 이렇게 준다
module.exports = {
  optimization: {
    minimizer: [
      new TerserPlugin({
        parallel: true,
        terserOptions: {
          // https://github.com/webpack-contrib/terser-webpack-plugin#terseroptions
        },
      }),
    ],
  },
};
```
  - uglify
  - babel minify : 바벨단에서 해주는 것도 가능하다

### 4. Code Splitting

- dynamic import 문법 자체를 지원하지 않는 브라우저도 있기 때문에 웹팩의 힘을 빌린다. 
- build시점에 import모듈을 chunk로 만들고, 필요한 시점에 **html에 script를 세팅하여 파일을 다운로드 한다**
- 특이한게 head에 넣는다. 이미 HTML 파싱이 끝났으니 head에 넣어도 괜찮다는 것일까..
- chunkname을 명시적으로 지정해줄 수 있다.
- Magic Comment : prefetch/preload 옵션을 주석으로 넣어줄 수 있다 
  - webpackPrefetch : 브라우저가 판단하여 유휴한 시간이 미리 리소스를 받아 놓는다.
  - webpackPreload : 필요하지만 당장은 필요하지 않은 리소스를 미리 로드 해놓도록 하기 위한 옵션

#### UX 관점

보통 하나의 번들로만 구성된 SPA의 경우 유저가 페이지를 방문한 최초에 커다란 번들을 한 번에 다운로드해서 실행시키기 때문에, 아직 유저가 사용하지 않은 페이지까지 모든 자바스크립트 코드를 다운로드 받아야 한다. 유저가 필요할 때 필요한 번들을 다운로드 해줘도 무방할 경우에는 번들을 나눠 필요한 상황에서만 필요한 리소스를 요청하는 방법을 사용할 수 있다. 

#### splitchunks(vendor splitting)

https://simsimjae.medium.com/webpack4-splitchunksplugin-%EC%98%B5%EC%85%98-%ED%8C%8C%ED%97%A4%EC%B9%98%EA%B8%B0-19f5de32425a

- 프로젝트 내에서 여기저기 import되어 사용되는, node_modules에 존재하는 모듈들을 아래처럼 별개의 chunk로 분리할 수 있다. 
- 여기저기서 사용되서 다수의 번들에 포함된 모듈들이 있다면 중복을 막기 위해 path를 지정해서 vendor로 분리할 수 있는 기능이 바로 splitchunks 기능 (웹팩 4부터 내장)
- chunk 옵션 : 정적, 동적 임포트에 따라 chunk를 다르게 생성할 수 있음
  - async: 동적 임포트만 따로 분리해서 독립적인 하나의 다른 번들을 만듬(동적 임포트가 많으면 여러개 만들수도 있음) 
  - initial : 정적 import에 대해서 무조건 별도의 청크로 만들고, 남아있는 비동기 모듈들도 분리시켜줌(따로 파일을 분리해야 동적으로 불러올 수 있기 때문 - 사실 비동기 import된 모듈은 따로 옵션을 주지 않아도 웹팩이 알아서 분리를 잘 해주는듯)
  - all : 정적이든 동적이든 상관하지 않음 import된 모든 파일을 별도의 파일로 분리하는 옵션

```js
   optimization: {
    splitChunks: {
      minSize: 20000, // 사이즈 지정 : 번들의 크기가 기준보다 작거나 클 경우 번들 분리
      minRemainingSize: 0,
      minChunks: 1, // 공유를 허락하는 청크 수(?맞나)
      maxAsyncRequests: 30,
      maxInitialRequests: 30,
      enforceSizeThreshold: 50000,
      cacheGroups: {
        defaultVendors: { // 이거 자체가 이름
          chunks: 'async',
          test: /[\\/]node_modules[\\/]/,
          priority: -10, // 모듈이 여러개의 캐시 그룹에 속할 수도 있는데
          // 우선순위가 높은 순서대로 번들에 들어간다
          reuseExistingChunk: true, // 모듈이 이미 분리되었다면 reuse하기
        },
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true,
        },
      },
    },
  },
```

### 5. Tree Shaking

https://webpack.js.org/guides/tree-shaking/

- 정적 분석을 통해 사용하지 않는 코드(죽은 코드)들을 번들링하지 않는 것.
- 죽은 코드 : 모듈에서 사용하지 않은 코드, import가 되지 않은 코드
- 근데 단순히 안 쓰인다고 바로 제거할 수 있는 것은 아니고, 프로젝트에서 부수 효과가 없는 것이어야만 제거가 가능
- 사이드 이펙트가 있거나 분석이 어려운 경우 트리쉐이킹 하지 않음
- 프로덕션 빌드에서는 terser을 사용해 트리쉐이킹을 함
  - pure annotation : 주석, 사이드이펙트가 없다고 판단. babel typescript가 이 주석을 붙여준다고 한다..
  - babel-plugin-transform-react-pure-annotation

#### 특징

https://medium.com/naver-fe-platform/webpack%EC%97%90%EC%84%9C-tree-shaking-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0-1748e0e0c365

https://webpack.js.org/guides/tree-shaking/

- optimization에 usedExports 옵션을 켜주는 경우, 임포트 되지 않은 코드들에 `unused harmony export 모듈이름`과 같은 주석이 붙고 export를 제거한다. 이거는 최적화에 활용된다. => minifier(uglify같은 게)가 이런 코드를 제거
- require가 아닌 import, export로 => es6만 지원
  - import, export가 babel에 의해 require, module.exports로 바뀌지 않게 babelrc에 "modules:false"로 지정해줘야함
- 사용되지 않더라도 코드에 포함되는 경우가 있다 : 전역함수(Object, Math)를 사용하는 경우, 함수실행 코드에서 멤버변수를 변경하고 반환하는 경우, static class property를 사용하는 경우
  - import한 라이브러리를 사용한 함수를 사용하지 않는 경우 => 참조한 함수는 제거되더라도 원 함수는 제거되지 않는다
- 다른 코드, 외부(이 외부가 뭐지)에 영향을 끼치지 않는다고 자신할 수 있는 코드라면 side effect가 없다고 package.json에 명시해줄 수 있다(false) `sideEffect:false`를 설정하면 side effect가 일어나도 사용하지 않으면 제거된다
  - 배열로 path를 나열하면 일부의 코드에 부수효과가 있는 것을 표현해줄 수 있음
  - 그러니까 약간 import된 것 중에 안쓰이는 코드 제거하는 것 정도는 크게 어렵지는 않은데 사이드 이펙트가 있는 경우에 판단하기 어려워하는 듯 => 그래서 어쩌면 굉장히 보수적으로? 뺄 코드를 판단해야 할지도
  - export가 있는 함수에 들어가있는 다른 함수가 export가 없는 경우는? : 제거되지 않는 듯 => 한번 해봐야겠다
  - 당당(?)하게 package.json에 false라고 설정할 수 있는 경우는 어떤 경우일까? : 
- babel 7 이후 버전에서는 트랜스파일링시 pure annotation을 붙여준다 => 그 다음에 uglify같은 minifier가 코드를 없애줌
- export default대신 module.exports를 사용하면 트리쉐이킹에서 제외된다. 사이드이펙트를 발생시키지 않지만 모듈에 꼭 들어가야 한다면 이런 방법 사용해도 ㄱㅊ
- 웹팩 4에서는 참조되더라도 사용되지 않으면 전부 제거됨(side effect 일어나는거 제외)