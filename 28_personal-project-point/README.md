# 개인 프로젝트 리팩토링, 리라이팅 하면서 배운것들 정리

최근 2개의 개인 프로젝트를 리팩토링하면서 새로 알게 되었던 것들,  
생각해보았던 것들 정리하면서 기술 위주로 회고를 해본다.

## FE Tooling

어쩔바벨 저쩔웹팩 개킹받쥬... 구글링 해봐도 사람들 설정과 답이 매우 제각각이고, 독스는 별로 친절하지 않거니와, 멘탈 모델을 잡기가 어려워서 꽤나 고통스러운 공부였다.

### emotion + Typescript React SPA 설정

emotion, jsx-transform, typescript를 모두 사용할 수 있는 프로젝트의 툴링 방식은 크게 두 가지다. 도구가 다르기 때문에 ts-loader와 babel-typescript의 차이는 있기는 하겠으나 성능이나 속도의 유의미한 차이는... 잘 모르겠다. 

종강시계 프로젝트에 둘 다 적용해 봤는데,,,, 체감상 뭔가 달라졌다, 눈에 띄게 빨라졌다(babel-typescript가 조금 더 빠른거 같긴 하다), 이게 더 좋다 이런 느낌은 잘 안든다. 그래도 preset-typescript를 쓰는 경우가 조금 더 쓰기 좋다고 느껴지는 지점이 있긴 했다.

preset-typescript를 쓰면서 ts-jest를 babel-jest로 바꿨는데 이거는 확실히 빨라졌다.

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

emotion의 경우 [storybook이 emotion11을 지원하지 않기 때문에](https://github.com/storybookjs/storybook/issues/7540#issuecomment-766982659), 스토리북의 바벨 설정에 `@emotion/babel-preset-css-prop` 프리셋을 넣어줘야 동작하는 듯 하다. 폴더 내부에 바벨 설정 파일을 넣어줘도 동작한다고 한다. 기본적으로 Storybook의 바벨 설정은 루트의 바벨 설정과는 자체적으로 다른 바벨 설정을 가지고 있다.

```js
module.exports = {
  babel: async (options) => {
    options.presets.push('@emotion/babel-preset-css-prop');
    return options;
  },
}
```
## React Code Manners

React 코드를 작성하면서 든 생각들
### HTML element의 prop type들

[이 포스팅](https://jbee.io/web/components-should-be-flexible/)을 보고는 기존 HTML 요소의 속성을 컴포넌트의 prop으로 활용할 수 있을 때에는, prop 타이핑을 [@types/React](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/react/index.d.ts)의 HTMLElementType을 이용해서 하면 경제적이겠다 싶었다. 
선언된 타입 중 비슷한게 많아서 좀 헷갈리는 편이다.

#### [`HTMLAttributes<HTML~Element>`](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/5809b163a7d9c52388504ef4a24836a87a49664c/types/react/index.d.ts#L1827)

`AriaAttributes`와 `DOMAttributes`를 상속하는 속성 타입으로, aria-role 관련 속성들과, on으로 시작하는 여러 이벤트 핸들러 속성을 포함하고 있는 타입이다. 다만 전달된 제네릭을 `HTMLAttriubtes`의 속성 내부에서는 사용을 안해서 각 모든 element들이 기본적으로 사용할 수 있는 속성들만 가지고 있다.

```ts
// HTML 요소로써 갖고 잇는 다른 요소들과의 공통 속성 요소들과
// 해당 Element에 속한 이벤트 요소, ARIA 요소들을 포함한다.
type CommonProps = HTMLAttributes<HTMLInputElement>;
```

#### [`~HTMLAttributes<HTML~Element>`](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/5809b163a7d9c52388504ef4a24836a87a49664c/types/react/index.d.ts#L2200)

이걸 써야 기존 HTML 속성에 들어있는 모든 속성을 타입으로 사용 가능하다. 제네릭에는 해당 요소의 Element타입을 넣는데, 이게 extends로 HTMLAttributes를 extend 한다. 위에서 설명한 공통요소, Aria요소, DOM이벤트 요소, 그리고 이 요소만 가지고 있는 유니크한 요소들이 타입의 속성으로 들어간다.

```ts
// 해당 요소의 유니크한 속성, 다른 요소들과의 공통 속성 요소
// 해당 element에 속한 이벤트 요소, ARIA 요소 포함
type InputProps = HTMLInputAttributes<HTMLInputElement>;
```

이 타입을 가지고 확장 prop 타입을 만드려면 다음과 같이 만들 수 있다.

```ts
export interface TextInputType extends InputHTMLAttributes<HTMLInputElement> {
  size?: TextType;
  widthFigure?: number;
}

export type TextInputType = {
  size?: TextType;
  widthFigure?: number;
} & InputHTMLAttributes<HTMLInputElement>
```

이 두개의 차이는 인터페이스와 타입의 차이와도 비슷한 느낌이 있는데, prop타입이 잘못 짜졌을 경우 interface는 에러가 인터페이스 선언문에서 나지만 type의 경우는 프롭이 잘 못 입력된 컴포넌트에서 발생한다는 차이가 있다.

또한 기본적으로 이러한 기본 속성 타입을 사용할때는 커스텀으로 설정하는 속성의 이름과 겹치면 안 된다. 인터페이스나 타입은 자동으로 오버라이딩 되지 않는다. 똑같은 이름으로 하고 싶으면 인터페이스를 수정하면 가능하지만, 이미 기존 속성이 맡은 역할이 있다는 점에서 같은 이름의 프롭을 넣는 것은 충돌의 여지가 높으니 unique한 이름을 적용하도록 하자.
### 추상화와 hooks

기존에 Custom Hook을 좀 더러우면 거기다가 뭉쳐서 분리하려고 무지성으로 사용했었던(...)시절이 있었다. 이제는 너무 아무생각없이 훅 분리는 하지 않고... [이 컨퍼런스 세션](https://www.youtube.com/watch?v=edWbHp_k_9Y)을 보고 또 다시 반성하게 되었다. 

리액트 개발하면서 hook으로 분리해야겠다고 생각했던 상황은 다음과 같다.

- **컴포넌트 사이에서 재사용성이 높은 저수준의 로직** - 커스텀 훅의 취지에도 맞는 내용이다.. input이나 modal을 쓴다던가 하는
- **useEffect를 사용해야 할 때** - 이거는 취향에 가까운데, useEffect는 react가 제공하는 기본 훅들 중에서 유일하게 네임스페이스가 없는 훅이라서 useEffect 콜백 내부와 디펜던시 배열을 봐야만 얘가 뭐하는 얜지 파악을 할 수 있다. 그런 파악을 돕기 위해 이름을 주면서 useEffect와 이와 관련있는 state을 훅으로 옮겨 사용하는 편이다.(use~Effect 요런식으로)
  - useEffect 추상화는 추상화 비용을 지불하는게 크게 나쁘지 않다고 생각하는데 다른 분들은 어캐 생각하시는지...
- **팀 컨벤션 등으로 추상화를 '약속'한 경우** : 아브라모브가 [Container-Presentational 구조를 더이상 추천하지 않는다고 말하면서](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0), React에서 새롭게 추상화를 할 수 있는 방법을 hook이라는 말을 했다. 커스텀 훅을 "추상화의 도구"라는 관점에서, Container-Presentation을 분리했던 과거의 유스케이스에 적용해보자면, 컴포넌트가 쓰는 쿼리같은 경우에는 팀의 니즈로 훅을 통해 컴포넌트와 분리해서 쓰자는 컨벤션을 만들 수도 있다.

hook으로 분리해야 할까 말까 긴가민가 하는 상황이 있는데, 이런 상황에서도 판단하는 기준이 살짝 있다.

- **컴포넌트 내부의 로직이 세부구현으로 무지 복잡해졌을 때는 컴포넌트로 추상화하는게 더 나을 수 있다** : 한 컴포넌트 내부의 로직이 무지 복잡해서 훅으로 추출해 분리하고 싶다면, 그 로직들이 혹시 컴포넌트의 자식 컴포넌트와 바인딩되는 로직이 아닌지, 혹은 특정 컴포넌트에 들어가는게 훨씬 더 좋은 추상화인지 생각할 필요가 있다. 굳이 훅으로 추상화할 필요가 없고 컴포넌트 내부의 로직을 위치하게 만드는게 더 가성비 좋은 추상화일 수 있다.
  - 컴포넌트도 좋은 추상화의 매개다. 프롭으로 원하는 정보를 노출시킬 수 있기 때문에..
- **파라미터가 없는 훅은 아예 만들지 않는게 더 나았을 확률이 높다** : 파라미터가 없다면 그 훅은 **외부에서 넣어야 하는 값이 없는 훅**이다. 추상화라는게 세부 구현은 감추고 필요한 부분만 인자로 노출시키는게 강점인데, 인자로 노출시킬게 없다면 추상화한 훅을 호출하는 부분에서는 **훅의 이름이나 반환값 말고는** 아무것도 그 훅에 대해서 알 수 있는게 없어지기 때문에 좋은 추상화가 아니다. 다만, 나같은 경우는 useEffect에 대해서는 아주 드물게 이런 훅을 쓰는 편이다.

추상화의 비용은 추상화 단계 상승으로 인한 복잡성이다. 파일을 꼬리를 물며 하나를 더 봐야 하고, 파일이 어디에 있는지 한번 더 찾아야 한다는 말이다. 어떤 로직이 분리되어야만 한다면 그 이유를 깊게 생각해보아야 하고, 응집도를 약간 해치면서 추상화 비용을 지불해도 이익을 얻을 수 있는지 타진해보아야 한다.

### useMemo, useCallback의 기준 잡기

useMemo와 useCallback은 발적화의 가능성이 높기 때문에 의미없는 useMemo나 useCallback은 없어야 한다. 또한, 매번 모든 순간 렌더링을 최적화할 필요도 없다. 

- 필요없는 메모이제이션을 사용하지 않는다
  - 특정 컴포넌트의 메모이제이션이 특정 프롭들 중 아주 잘 바뀌는 프롭과 의존하고 있다면 의미가 없다. 이거는 그냥 매 랜더링마다 함수가 만들어지는 로직과 다를게 없기 때문이다.
  - 특정 컴포넌트가 단 한번만 렌더링되는 것이 보장되는 상황이라면 걔네도 굳이 메모이제이션이 필요 없을 것이다.
- 레퍼런스 비교가 필요할때 메모이제이션을 사용한다 : (피하는게 좋겠지만) 개발하다보면 useEffect 등에 직접 state값을 넣는 것보다, state를 의존성으로 가지는 메모이제이션 로직을 useEffect에 넣어야 하는 상황도 충분히 생길 수 있다. 이때는 메모이제이션을 통해 useEffect가 제대로 동작하도록 함수를 useCallback등으로 감싸야 한다.
- 렌더링을 줄여주는 방식으로 메모이제이션을 사용한다 : 유저 인터랙션(거듭된 토글이나 스크롤, 인터벌) 등으로 많은 리랜더링이 불가피할 경우 메모이제이션으로 최적화한다.
  - 근데 사실 이것도 발적화일...수도 있다. 리랜더링 자체를 그렇게 너무 통제하려고 하지 말라는 개발자들도 좀 많은듯 함. 확실히 성능에 영향이 있는 경우에만 인스펙션해서 적용해야 할지도..? 이건 근데 실무를 더 경험해봐야 할 것 같기도 하다.
- 부모 컴포넌트 랜더링이 너무 자주 일어나서 자식 컴포넌트의 랜더링도 덩달아 많이 일어나는 경우에는 React.memo 래퍼를 사용한다.
  - 애초에 설계가 잘못된 걸수도 있다는 생각을 좀 해볼 수 있지 않을까 싶긴 하다. 불가피한 경우인지 타진한다!
- 애초에 리렌더링을 많이 피해야한다 : 컴포넌트 내부 state은 정말 필요한 최소만 선언하고, 데이터 패칭 로직은 그것을 소비하는 컴포넌트와 최대한 가깝게 위치시키는게 좋다. 상태 끌어올리기를 신중하게 하자.

### types와 interface
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