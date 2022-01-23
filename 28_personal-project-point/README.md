# 개인 프로젝트 리팩토링, 리라이팅 하면서 배운것들 정리

최근 2개의 개인 프로젝트를 리팩토링하면서 새로 알게 되었던 것들,  
생각해보았던 것들 정리하면서 기술 위주로 회고를 해본다.

## FE Tooling

어쩔바벨 저쩔웹팩 개킹받쥬... 구글링 해봐도 사람들 설정과 답이 매우 제각각이고, 독스는 별로 친절하지 않거니와, 멘탈 모델을 잡기가 어려워서 꽤나 고통스러운 공부였다.

### emotion + Typescript React SPA 설정

emotion, jsx-transform, typescript를 모두 사용할 수 있는 프로젝트의 툴링 방식은 크게 두 가지다. 도구가 다르기 때문에 ts-loader와 babel-typescript의 차이는 있기는 하겠으나 성능이나 속도의 유의미한 차이는... 잘 모르겠다. 

종강시계 프로젝트에 둘 다 적용해 봤는데,,,, 체감상 뭔가 달라졌다, 눈에 띄게 빨라졌다(babel-typescript가 조금 더 빠른거 같긴 하다), 이게 더 좋다 이런 느낌은 잘 안든다. 그래도 preset-typescript를 쓰는 경우가 조금 더 쓰기 좋다고 느껴지는 지점이 있긴 했다.

#### 1. ts-loader을 쓰는 경우

```json
// tsconfig.json
{
  "compilerOptions" : {
    "target": "esnext",
    "module": "esnext",
    "jsx": "react-jsx",
    "jsxImportSource" : "@emotion/react",
  }
}
```

```js
// babel.config.js
// 요런 경우는 폴리필 설정이 필요없다면 바벨을 아예 안 써도 된다. 

module.exports = (api) => ({
   presets: [
      [
        "@babel/preset-env",
        {
          targets: {"chrome": "58"},
          modules: false,
          loose: true,
          corejs: 3,
          useBuiltIns: "usage"
        }
      ]
})
```

타입스크립트 컴파일을 할 때 jsxImportSource를 @emotion으로 설정하여, jsx를 `_jsx` 함수가 아닌 emotion 라이브러리의 `jsx` 함수로 바꿔 css-props를 컴파일할 수 있도록 설정한다. 타입스크립트 컴파일 단에서 많은 것을 해결할 수 있으니 폴리필이 필요없는 경우 babel 설정을 신경쓰지 않아도 되고, 잡다한 바벨 프리셋을 설치하지 않아도 잘 돌아간다. 빠르게 개발 환경을 세팅하기에 좋은 방법인듯 하다.

#### 2. babel-typescript를 쓰는 경우

```json
// tsconfig.json
// preset-typescript랑 tsconfig.json은 상관이 없다.
```

```js
module.exports = (api) => {
  const isTest = api.env('test');
  api.cache.forever();

  return {
    presets: [
      [
        "@babel/preset-env",
        {
          targets: {"chrome": "58"},
          modules: isTest ? "commonjs": false, // 테스트용(jest는 commonjs만 돌린다)
          loose: true,
          corejs: 3,
          useBuiltIns: "usage"
        }
      ],
      ["@babel/preset-react", {
        runtime: "automatic", // jsx transform 적용
        importSource: "@emotion/react" 
      }],
      "@babel/preset-typescript"
    ],
    plugins: ["@emotion"]
  }
}
```

아까 typescript에서 해줬던 것들을 babel로 모아준다. 프리셋의 순서는 아래에서, 위(뒤에서 앞) typescript -> react -> preset env 순이다.

react preset의 runtime automatic을 설정하면 트랜스파일시 자동으로, 원래는 `_jsx` 함수를 변환하여 넣어주는데, import source 옵션을 설정하면 해당 라이브러리가 자동으로 임포트되어 해당 라이브러리를 통해 JSX를 변경할 수 있도록 설정한다.

emotion관련해서 babel에 넣을 수 있는 옵션이 크게 2개인데, 하나는 `@emotion/babel-preset-css-prop` 프리셋이고, 다른 하나는 `@emotion/babel-plugin`이다. `@emotion/babel-preset-css-prop`이 `@emotion/babel-plugin`를 포함하고 있다. 

emotion-plugin은 css 템플릿 리터럴 함수를 css함수로 바꾸고 내부에 작성한 css를 동적으로 적용된 요소나 다른 css를 참조하는지 살피며 변환한다. 이 과정에서 pure annotation도 붙여준다고도 한다. 

emotion-css-props-preset은 emotion-plugin의 CSS 변환에 더불어 React.createElement 대신에 emotion의 jsx 함수로 컴파일하는 역할을 수행한다. JSX-transform을 사용한다면 css-props-preset 대신에 위처럼 preset-react에서의 옵션 설정을 사용하고, plugin만 사용한다. 만약에 사용하지 않는다면, typescript나 react-preset보다 앞서 css-props-preset을 [사용해야한다고 docs에 나와이따](https://emotion.sh/docs/@emotion/babel-preset-css-prop#usage). JSX를 다 해결하고 다른거 시작하라는 말인듯.

처음에 원래 그렇게 해왔듯 ts-loader를 셋업하고 emotion 처리를 tsconfig에 넣었는데 꽤나 간편한 방식으로 느껴져서 이게 더 낫나..?하는 생각이 들었다. 디펜던시에 넣는 바벨 플러그인도 줄일 수 있고, 종강시계는 최신 크롬 브라우저에서만 돌아가면 되었기 때문에 폴리필의 필요성도 별로 없어서 바벨을 아예 때도 되겠다는 생각이 들 정도였다.

그런데 babel-typescript로 세팅을 바꾸고, React-preset을 다시 끼우고 난 다음에 느껴졌던 이점은, **타입스크립트 트랜스파일러가 정말 타입스크립트 트랜스파일만 한다는 점**인 것 같다고 느껴졌다. babel-typescript의 TS 트랜스파일러는 JSX에 접근하지 않고, 타입스크립트만 제거한 후, React-preset에서 JSX Trasform과 emotion을 해결하는 방식이 좀더 각자의 역할에 더 충실한 방향이며, 바벨을 이해한다면 preset을 통해 트랜스파일에 이르는 모든 과정을 보여줄 수 있다는 생각이 들었다.

#### typescript-preset에서 JSX 컴파일?? 

- 이거 ts-loader에서는 되는데, babel-typescript에서는 안되는 걸까..? 싶어서 프리셋 옵션을 봤는데 약간 묘하다. jsxPragma, jsxPragmaFrag 옵션을 제공하고, JSX 컴파일을 대체할 라이브러리?와 함수를 지정할 수 있는 것 같다. 그런데 emotion은 어떻게 설정해야 컴파일이 되는지 모르겠다...
- [React Docs에서는 JSX-transform을 적용하려면](https://ko.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html#manual-babel-setup) `preset-react`나 `plugin-transform-react-jsx`(preset-react에는 포함되어 있다)를 사용하라고 말하고 있는걸 보니 안되나..? 싶긴한데 typescript-babel의 저 옵션들은 뭐지..? 궁금궁금..

### preset-typescript

원래 [요런 글](https://ui.toast.com/weekly-pick/ko_20181220)에 나오는 통상적인 차이 정도나 알고 있었지만, 약간의 차이를 더 발견할 수 있었다. 

- babel-typescript는 tsconfig를 참조하지 않는다. 그냥 타입스크립트를 죄다 없앨 뿐이다. babel-typescript를 사용한다면 tsconfig는 타입 체크를 위한 설정일 뿐이므로 빌드 결과물과 상관없이 타입 체크를 위해서만 설정을 해줘도 괜찮다.(noEmit같은거...)
- 바벨은 Enum과 Namespace를 지원하지 않는다.
  - 그냥 enum은 안 좋은 것으로 악명이 높다. ECMAScript기능이 아니므로 트랜스파일이 필요하며 이때 문법이 많이 바뀌고, 즉시 실행 함수(전역 범위를 건드는)처럼 스코핑이 되기 때문에 [트리쉐이킹에 쥐약](https://engineering.linecorp.com/ko/blog/typescript-enum-tree-shaking/)이다. 대신에 const enum은 컴파일시 이넘이 상수로 대체되기 때문에 이런 면에서 나쁘지는 않다.
    - 함수의 인자로 enum을 지정하면, 개발 중에는 enum을 참조한 리터럴만 넣을 수 있는데, 컴파일되고 난 뒤의 자바스크립트에서는 문자열도 넣을 수 있다. 이런식으로 동작이 다름
  - 하지만 typescript-preset은 const enum도 지원 [안 한다.](https://github.com/babel/babel/issues/8741) 유니언 타입을 쓰자
- preset-typescript 자체로 ES 문법을 다지원하는 것이 아니기 때문에 preset-env, 혹은 다른 문법 지원 플러그인이 필요할 수 있다.

#### preset-typescript의 런타임 타입체크를 보완하는 방법

babel/preset-typescript는 빌드시에 따로 타입체크를 하지 않기 때문에, 빌드간 타입 에러가 나도 알지 못한다. 이런 단점을 극복할 수 있는 몇가지 도구를 같이 사용하면 좋다.

- tsc로 타입체킹을 할 수 있도록 tsconfig를 구성하고, package.json 명령어에 넣어둬서 배포 스크립트나 CLI에서 실행할 수 있도록 설정한다.
  - noEmit: 빌드 결과물이 발생하지 않고 타입 채킹만 한다.
  - isolatedModules : babel은 한 번에 한 파일만 컴파일하는데 이때 런타임 버그가 생길 수 있다. enum이나 namespace, module 등의 코드를 추가 검사하여 바벨이 수행할 컴파일이 안전한지 미리 확인한다.
    - 프로젝트 내에 모든 각각의 소스코드 파일을 모듈로 만들기를 강제한다. 모듈로 소스코드를 작성하지 않았을 경우 에러를 출력한다. 
  - declaration: d.ts가 생성되었는지 확인
- typescript관련 eslint 설정을 빡세개 하고, 웹팩 등의 HMR과 연동해서 빌드할때마다 eslint 오류를 표시하도록 한다.
  - 이러면 왠만한건 다 잡을 수 있다


### Storybook 설정


## React Code Manners

### 기존 element의 prop type들

### 추상화와 hooks

### useMemo, useCallback의 기준 잡기

## Optimization

### font 최적화, 자원 preload, prefetch

### network 탭의 waterfall과 작동방식

### Framer Motion의 Lazy Motion

### 스레드 블락과 최적화, 이벤트 시의 UI 변화

## Accessibility

### ARIA ROLE에 대한 정확한 이해

## Testing

### react-test-library 사용시 팁

### 테스팅 전략

이거 구체화해서 블로그에 정리

## etc

### State Management - (Recoil VS Recoil + React Query)

이거는 POC 해보며 따로 정리

### CSS-IN-JS, emotion 관련