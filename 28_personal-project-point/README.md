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

또한 기본적으로 이러한 기본 속성 타입을 사용할때는 커스텀으로 설정하는 속성의 이름과 겹치면 안 된다. 인터페이스나 타입은 (자바의 그것과는 다르게) 자동으로 오버라이딩 되지 않는다. incompatible 에러뜸

```ts
// 뭐 요런 느낌으로 해야함
type Modify<T, R> = Omit<T, keyof R> & R;

interface OriginalInterface {
  a: string;
  b: boolean;
  c: number;
}

type ModifiedType  = Modify<OriginalInterface , {
  a: number;
  b: number;
}>
```

똑같은 이름으로 하고 싶으면 인터페이스를 수정하면 가능하지만, 이미 기존 속성이 맡은 역할이 있다는 점에서 같은 이름의 프롭을 element에 넣는 것은 문제의 여지가 높으니 unique한 이름을 적용하도록 하자.
### 추상화와 hooks

기존에 Custom Hook을 좀 더러우면 거기다가 뭉쳐서 분리하려고 무지성으로 사용했었던(...)시절이 있었다. 이제는 너무 아무생각없이 훅 분리는 하지 않고... [이 컨퍼런스 세션](https://www.youtube.com/watch?v=edWbHp_k_9Y)을 보고 또 다시 반성하게 되었다. 

리액트 개발하면서 hook으로 분리해야겠다고 생각했던 상황은 다음과 같다.

- **컴포넌트 사이에서 재사용성이 높은 저수준의 로직** - 커스텀 훅의 취지에도 맞는 내용이다.. input이나 modal을 쓴다던가 하는
- **useEffect를 사용해야 할 때** - 이거는 취향에 가까운데, useEffect는 react가 제공하는 기본 훅들 중에서 유일하게 네임스페이스가 없는 훅이라서 useEffect 콜백 내부와 디펜던시 배열을 봐야만 얘가 뭐하는 얜지 파악을 할 수 있다. 그런 파악을 돕기 위해 이름을 주면서 useEffect와 이와 관련있는 state을 훅으로 옮겨 사용하는 편이다.(use~Effect 요런식으로)
  - useEffect 추상화는 추상화 비용을 지불하는게 크게 나쁘지 않다고 생각하는데 다른 분들은 어캐 생각하시는지...
- **팀 컨벤션 등으로 추상화를 '약속'한 경우** : 아브라모브가 [Container-Presentational 구조를 더이상 추천하지 않는다고 말하면서](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0), React에서 새롭게 추상화를 할 수 있는 방법을 hook이라는 말을 했다. 커스텀 훅을 "추상화의 도구"라는 관점에서, Container-Presentation을 분리했던 과거의 유스케이스에 적용해보자면, 컴포넌트가 쓰는 비동기 데이터 쿼리같은 경우에는 팀의 니즈로 훅을 통해 컴포넌트와 분리해서 쓰자는 컨벤션을 만들 수도 있다. 이런 경우에는, 팀원들이 똑같은 이해를 가지고 있으므로 추상화로 인한 복잡성은 상쇄된다.

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

### +) types와 interface

React에서의 types와 interface 중 뭐를 어떻게 써야하나 생각을 많이 해봤는데, 일반적인 상황에서도 많이 고민을 했었던 주제다.

일단 대부분의 경우, 특히 객체 타입을 만든다는 관점에서 types는 interface에서 할 수 있는 모든게 된다. interface의 경우 extend키워드를 쓸 수 있다는 것과, 같은 이름을 가진 Interface를 여러개 선언해서 얻는 객체 보강 등을 사용할 수 있다. types 같은 경우는 유니언, 인터섹션 타입, 혹은 유틸리티 타입(인터페이스 같은 경우에는 타입으로 다시 선언해야함) 등을 쉽게 사용할 수 있다.

개인적으로는, 개발 중에 **변경이 많이 일어나는 타입**은 타입 별칭을 쓰는게 낫다는 생각이 든다. 유틸리티 타입, 인터섹션 타입 등을 다채롭게 활용해서 원하는 결과물을 쉽게 얻을 수 있는 상태로 남겨야 하기 때문이다. 대표적인 것이 컴포넌트의 props(기존의 HTML Element의 prop을 extend하는 경우 interface를 사용할 수도 있긴 하겠다), 혹은 enum대신에 사용하는, 다양한 선택지를 표현해야 하는 유니언 타입 등이 있다.

변경이 많이 일어나지 않고, extends, implements를 사용할 여지가 있는 객체타입의 경우는 interface를 사용해 표현한다. 대표적인 것이 백엔드 리스폰스의 스키마, 혹은 에러 객체도 인터페이스로 표시할 수 있을 것이다(클래스 쪽이 더 좋은거 같기는 하다.) 혹은 클래스의 추상을 제공하는 인터페이스라던가(이거는 추상클래스 쓰면 될거같긴 한데)

확실히 extends를 쓰고싶은 상황은 있는 것 같다. 인터섹션 타입 쓰는게 뭔가... 맛이 안 살때가 있음 인터섹션은 뭔가 곱하기나 합집합같은 느낌인데, extends는 뭔가 얘로부터 떨어져 나온 타입이라는 뜻, 상속되었다는 뜻을 표현하기에 적합해서 그런 것 같다..

## Optimization

최적화에 대한 몇가지 인사잇트

### font 최적화, 자원 preload, prefetch

[이 글은 매우 좋았다](https://velog.io/@vnthf/%EC%9B%B9%ED%8F%B0%ED%8A%B8-%EC%B5%9C%EC%A0%81%ED%99%94-%ED%95%98%EA%B8%B0)

- 폰트 자체는 바뀔 일이 거의 없기때문에 max-age를 길게 잡아도 상관없다. 직접 호스팅하는 경우 GZIP으로 서빙한다.
- subset이나 unicode range를 설정할 수 있는 경우 필요한 range의 폰트 부분만 받아올 수 있도록 한다.
- `font-display:fallback` : 짧은 시간만 block, 기본 폰트를 보여주고 응답이 오면 해당 폰트로 바꾼다.
- `FontFaceObserver`같은 라이브러리를 사용해 폰트가 다 다운받아졌을 때 컨텐츠를 노출하는 것도 가능(JS에서)
- preload는 다른 어떤 것보다 먼저 리소스를 요청하고 렌더링을 블락함. 화면에 꼭 필수적인 폰트를 로딩할때 사용
#### link meta tag 속성

link 태그는 현재 문서와 외부 리소스 사이의 연관 관계를 명시한다.

- preload : 사용자가 앞으로 요청할 가능성이 있으므로 브라우저가 대상 리소스를 미리 가져와 캐시하도록 명시
- prefetch : 사용자가 요청할 가능성이 있으므로, 브라우저가 대상 리소스를 미리 가져와 캐시하도록 명시
- as : preload, prefetch 특성을 지정했을때만 사용하며, 컨텐츠의 유형을 지정한다. 요청 매칭, 올바른 컨텐츠 보안 정책 적용, 올바른 Accept 헤더 적용에 필요. 요청 우선순위에도 관여
  - audio, document, embed, fetch, font, image, script 등..
- crossorigin : 리소스를 가져올 때 crossorigin을 사용해야 하는지 나타내는 특성
  - anonymous : 교차 출처 요청을 수행하지만 인증 정보를 전송하지 않음
  - use-credential : 인증 정보를 전송함
  - 둘다 ACAO 설정 안되있으면 막힘(이건 당연)

### network 탭의 waterfall과 작동방식

기본적으로 HTTP(1.1) 요청은 순차적이다. 현재의 요청에 대한 응답을 받고 나서 다음 요청을 실시한다. 네트워크 지연과 대역폭 제한 때문에 다음 요청을 보내는 데는 딜레이가 발생할 수 있다.

크롬은 **한 도메인**당 6개까지 동시 요청이 가능하다. 그러니까 TCP 커넥션을 한번에 6개까지 유지하는게 가능하다. 네트워크 요청 자체는 그보다 더 많이 병렬적으로 가능하다. [이 포스트에서는](https://medium.com/@syalot005006/%EB%B8%8C%EB%9D%BC%EC%9A%B0%EC%A0%80-http-%EC%B5%9C%EB%8C%80-%EC%97%B0%EA%B2%B0%EC%88%98-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0-3f7aa1453bc2) 256개까지는 된다고.

브라우저에서는 네트워크 요청을 보낼 때 그냥 단순하게 요청만 보내고 받는게 아니라 전후로 여러가지 작업을 수행할 수도 있다. 자원과 도메인의 성격에 따라 어떤 동작을 취할지가 달라진다.
- queuing : 브라우저가 요청들의 우선순위를 판단하고, 디스크 캐시의 공간을 할당한다.
- stalled : 큐잉이 될때까지 대기한다
- DNS Lookup : DNS를 찾아봐야 할 수도 있다
- initial connection : TCP 커넥션 첫 설립 + SSL
- waiting(TTFB) : 응답의 첫번째 바이트가 올 때까지 기다리는 시간(초록색)
- content download : 브라우저가 리스폰스를 받고 해석하는데 걸리는 시간, 이 시간은 리스폰스 바디를 읽는데 걸리는 모든 시간을 포함함. 

Head of Line Blocking : 파이프라이닝이 가능함에 따라 HTTP(1.1) 레벨에서 이미 맺은 하나의 커넥션으로 요청은 한번에 여러개를 보낼 수 있는데, 요청을 병렬적으로 보낸 경우 하나의 처리가 지연 되었을때 모든 응답이 지연된다.
  - HTTP2는 멀티플렉싱으로 해결한다.
  - 애초에 한번에 여러개를 보내지 않고 keep-alive된 커넥션을 통해 순차적으로 요청을 한다면 별로 네트워크 탭에서는 해당사항 없는 이야기인가? 네트워크 탭에서는 어떻게 HOLB를 확인할 수 있는거지?
  - TCP단에서의 HOLB : HTTP로 통신하는 두 엔드포인트 사이 네트워크 어딘가에서 하나의 패킷이 빠지거나 없어진다면 없어진 패킷을 다시 전송하고 목적지를 찾는 동안 전체 TCP 동작이 중단된다. => 이거는 HTTP2에서도 해결이 안되는 문제다

### 스레드 블락과 최적화, 이벤트 시의 UI 변화

[스크롤이나 휠, 마우스오버같은 유저 이벤트 시에도 이벤트가 프레임의 변화를 만들어내고 있다면 layout, repaint나 composite이 일어날 수 있다](https://www.html5rocks.com/en/tutorials/speed/scrolling/). [특정 요소를 건드릴때도 발생할 가능성이 있다.](https://gist.github.com/paulirish/5d52fb081b3570c81e3a)

스타일이 명시적으로 바뀐다고만 해서 reflow, repaint가 일어나는 것은 아니다. 결국 브라우저는 유저 인터랙션에 맞게 화면을 바꾸기 위해 새로운 프레임을, 똑같이 16.6ms안에 만들어줘야 한다. 스크롤만 한 상태로 퍼포먼스 탭을 찍어봐도 paint와 composite는 항상 일어나게 되있다.

또한 새로운 자원(스크립트, CSS)의 네트워크 동작, 로드 동작은 렌더링 자체를 블락하며, 마이크로태스크(Promise, Mutation Observer, nextTick)는 렌더링이나 페인팅을 블락한다.

만약 스크롤에 맞춰 다른 스타일 변화, 다른 UI 변화가 함께 일어난다면 스레드가 더 고통받고 성능에 무지 안 좋을 수도 있다. 일단 프레임 렌더링하는 특정 시점에 무슨 일들이 너무 많이 발생한다면 스레드를 블락하기 십상이므로 이런 경우에는 잘 생각해서 스크롤 하는 중에 스레드에 발생하는 일을 줄이기 위한 최적화가 필요하다.

스크롤 이벤트 때문에 laggy한 애니메이션이 나타나는 경우, frame drop이 있는 프레임이 어디있는지 퍼포먼스 탭에서 살펴보고, 해당 프레임의 스레드를 확인해서 너무 많은 일이 몰려있지는 않는지, 스레드의 여러 일들을 나누는 방법은 없을지 인스펙트 해봐야한다.

쫌 추상적으로 적은거같은데, 눈으로 애플리케이션에 성능 문제가 발생하는 것 같으면, 퍼포먼스탭 열고 어디가 병목인지, 특정 프레임이나 시점에 너무 많은 일이 오래 발생하고 있지는 않는지 보자.

이거 진짜 제대로 공부해보고 싶긴 한데...모든 정보가 너무 산재되어있다.. [이런글도 있다](https://medium.com/naver-fe-platform/%EB%AC%B4%ED%95%9C-dom-%EB%A0%8C%EB%8D%94%EB%A7%81-%EC%B5%9C%EC%A0%81%ED%99%94-%EA%B2%BD%ED%97%98%EA%B8%B0-237e6e9088e8)

### Framer Motion의 Lazy Motion

framer motion의 Motion 컴포넌트는 모든 기능을 프리번들하여 가지고 있다, 대신에 lazyMotion을 사용하면 프리로드 없이, 애니메이션을 사용하는 시점에 애니메이션을 붙여 번들 크기를 줄인다고 한다.

## Accessibility

### WAI-ARIA

[레진이 넘 좋은 문서를...만들었다](https://github.com/lezhin/accessibility/tree/master/aria)

HTML 자체만으로 해결되지 않는 접근성 문제를 보완하는 W3C 명세. WAI-ARIA는 요소에 role, 또는 aria-* 속성을 추가하여 컨텐츠의 역할, 상태, 속성 정보를 보조기기에 제공한다. 모든 요소에 무분별하게 사용할 수 있는 것은 아니며, 요소의 속성을 사용할 수 있는지 [명세](https://www.w3.org/TR/html-aria/)를 보면서 타진해야한다.

HTML을 의미있게 작성하면 WAI-ARIA 사용을 최소화할 수 있다. 

#### 주요 Role

- tablist, tab, tabpanel : 스타일을 의미하는 것이 아니라 현재 페이지 내용에 색인을 제공하는 구조
- tooltip : 앵커 또는 폼 컨트롤 요소에 대한 참고형 컨텐츠. 보통 마우스 오버 또는 키보드 초점을 받으면 표시하는 내용
- status : 성공, 상태 메시지를 사용자에게 전달하는 컨텐츠. 사용자의 현재 작업을 방해하지 않고(초점을 옮기지 않고) 보조기기 사용자에게 조언할만한 메시지 전달. role-alive:polite
- alert : 오류, 제안 메시지. 시간에 민감하고 중요한 메시지를 사용자에게 전달. 사용자의 사용 흐름을 끊을 수 있는 메시지를 즉시 읽도록 할 수도 있다.
- alertDialog : 사용자 동의, 또는 확인이 필요한 인터랙션 요소. 다른 과업을 차단함. aria-modal:true와 같이 쓰인다. 사용자가 반드시 입력해야할 내용. 안에 읽는걸 다 읽는다.
- dialog : 사용자 인터렉션이 필요한 현재 문서의 하위창. 사용자 정보를 입력하거나 응답하도록 하는 내용을 반드시 포함한다. 과업을 차단하는 경우에는 alertDialog
  - dialog html 요소가 있지만 모든 브라우저에서 지원하지는 않는다고 한다,,
- nav : 현재 페이지 또는 연결된 페이지를 탐색하는 주요 탐색 블럭. 문서의 주요 내용을 탐색하는 경우에 사용하면 적절. 모든 링크 집합이 탐색 블럭은 아니다.
  - 스크린리더가 화면 위의 nav를 바로 찾는 방식으로 작동할 수도 있다.
- aside : 주요 내용을 보완하는 블럭. 보충을 제거해도 주요 내용에 변함이 없어야 한다. 주요 내용에서 보충을 분리한 경우 
- none : 의미 없는 역할을 선언한 경우 보조기기는 마크업의 의미를 제거한후 내용만 사용자에게 전달한다. aria-hidden은 요소와 내용을 모두 감추지만, role="none"은 내용을 드러내고 의미만 감춘다
- aria-modal : 요소가 모달인지 여부를 보조기기에 전달한다. 본문 위에 대화상자를 띄워 본문을 차단한 상태로 상호작용하는 요소를 의미함. role=dialog나 role=alertDialog와 함께 사용하낟. 모달 컨텐츠를 화면에 표시할때는 사용할 수 없는 요소에 aria-hidden 속성을 선언해서 보조기기가 무시하도록 설정해야 한다. 
  - 본문을 차단하는데, modal을 모두 탐색한 후에 focus를 잃어버리는 단점이 있다. 이걸 어떻게 해결해야할지 모르것다 지원하지 않는 스크린리더기도 있다(...)

#### 라벨링

- aria-label : 간결한 설명. 요소를 밝히면서 같이 읽는다. aria-label은 현재 요소를 설명할 다른 참조 요소가 없는 경우에만 사용한다. aria-labelledby 속성과 함께 선언하는 경우 aria-label이 우선순위가 낮기 대문에 aria-labelledby로 설명한다.
  - 내부 content에 우선한다.
- aria-describedby : id값을 이용해 상세한 내용을 연결하는 방식으로 설명한다. 링크, 폼 컨트롤, 알럿, 알럿 대화상자 등에 사용하면 적합. 요소를 읽고 나서 다음 호흡?에 읽는다.
  - 속성 자체를 설명한다기보다, 속성을 이해하는데 필요한 부수사항을 알려주는데 쓰면 좋은듯 하다 [이 예제처럼](https://github.com/lezhin/accessibility/tree/master/aria#22-%EC%9E%90%EC%84%B8%ED%95%9C-%EC%84%A4%EB%AA%85-%EC%B0%B8%EC%A1%B0aria-describedbyid-reference-list-)
- aria-labelledby : id값을 이용해 aria-label로 쓰이는 간결한 내용을 연결한다. 주로 헤딩을 연결한다.

## Testing

### [react-test-library 사용시 팁](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)

중요한 것은,,, 사용자 입장에서 테스트를 작성하자는 것인듯 하다. 개발자 입장에서 벗어날수록 깨지는 테스트를 작성할 확률이 줄어드는 듯 하다. 만든 사람이 쓴 팁을 좀 정리해봤다

- Testing Library Eslint 플러그인이 있다
- wrapper는 엔자임 쓸때나 쓰던 거니까 쓰지마라
- cleanup은 자동으로 이루어지니까 afterEach에 넣어서 쓸 필요 없다
  - 설정에서 disable해줄 수 있는 것 같다. 렌더링을 최소로 줄이기 위해서 한번 렌더링하고 클린업 하지 않은 상태에서 테스트 돌리는 프랙티스는 어떤지..? 종강시계에서는 그렇게 해봤는데 안티패턴은 아닌가..??
- screen 사용하는게 좋다 render문을 구조분해할 필요가 없고 스코프에서 자유롭게 테스트를 작성할 수 있다.
- jest의 기본 기능으로만 테스트하지 말고 jestDom 써라(toBeIndocument, toBeDisabled 등)
- act 불필요하게 쓰지 마라. render, fireEvent는 기본적으로 act로 감싸져 있다.
- 쿼리 우선순위 따라라(대충 role->label->text->testId)
  - 가능한 최종 사용자가 하는 방식에 DOM을 쿼리하는게 좋다고 한다. 배다른 이슈지만 [testId로 바로 찝어서 테스트하는 방식이 성능이 가장 좋다]() 그렇지만 남발하는 것은 라이브러리에 의도에 반한다.
- querySelector 쓰지 마라. 뭐 이건 자명하다. DOM 구조에 의존하는 테스트는 좋지 않다.
  - 메인테이너의 관점이 조금 흥미로운데, data-testId, id, classname같은걸로 쿼리를 하기보다 innerText로 쿼리를 하는게 더 낫다고 생각하는 이유가, text는 개발자가 명백히 알아야하고 아는 변경점이기 때문에 망가지는 상황이 있더라도 분류하고 문제를 고치는게 쉽다고 말한다.
- ByRole 많이 써라. name 옵션을 사용하면 accesible name으로 요소를 쿼리할 수 있다. 
  - 확실히 ByRole이 매우 쿼리하기 자연스럽다는 생각이 든다. 사용자 중심적이기도 하고
  - 제대로 활용하려면 역시..... 접근성을 준수하는 마크업을 만들어야겟다
  - 기본 Role을 가진 요소 쿼리할때 좋을듯?
  - 버튼같은거 쿼리할때 좋을듯?
- role 설정 남발하거나 잘못 쓰지 마라. 위의 접근성 부분에서도 봤지만 WAI-ARIA는 마크업을 잘 쓰면 최소한으로만 사용할 수 있다. 쿼리도 이상해진다.
- user-event 써라. 사용자 상호작용과 더 유사한 메서드를 제공한다. keyDown, keyup, keypress까지 테스트하고 싶다면 이게 맞다. 특정 작업을 수행할때 유저가 실행할 모든 동일한 이벤트를 실행한다.
  - 모든 상황에 사용하는 것은 오버킬 아닐까..? 테스트 느려질거같은데 성능은?
- 존재 여부를 확인하는 경우 외에는 get이나 find말고 query를 써라. query는 에러를 발생시키지 않는다.
- waitfor 남발하지 말고 find 써라. 더 간단하고 나은 에러메시지를 받을 수 있다
  - 음 종강시계에서는 비동기 waitFor밖에 몰라서 waitFor 남발했다 수정해야지..
- waitFor에 빈콜백 넘겨두지 말아라. 이거 테스트 일관성 해친다.
- waitFor 내부에 다중 statement 쓰지 말아라. 기다릴 부수효과를 확실하게 선언하라
  - 음 이것도..
- waitFor에서 부수효과, 이벤트 실행하지 마라. 단언문만 써라 waitFor은 수행한 작업과 단언문 전달 사이에 undeterministic한 시간이 있는 테스트를 위한 것이다. 이벤트가 여러번 실행될 수 있다(체험했다..)
  - 이것도....
- get을 expect대신 쓰지 마라. get이 테스트케이스에서 에러를 던지기 때문에 대신 사용할 수 있지만, 가독성을 위해, 이게 걍 쿼리가 아니라 테스트 맥락 속에서 존재함을 알려줘야 한다.

### 테스팅 전략

이거 구체화해서 블로그에 정리 - [했지롱](https://maxkim-j.github.io/posts/effective-react-test-strategy)

## etc

### State Management - (Recoil VS Recoil + React Query)

이거는 POC 해보며 따로 정리

### CSS-IN-JS, emotion 관련