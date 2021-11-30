# 프론트엔드 툴링과 선택의 여지

## 순서가 중요한 것

React 프로젝트에서 tsx 컴파일 일반적인 순서 : 타입스크립트 -> JSX -> 자바스크립트

### babelrc vs webpack babel loader

루트 디렉토리에 babelrc 두기 vs Webpack Babel Loader에서 설정하기

- 결과의 차이는 없다.
- 기본적으로 루트에 babelrc를 두면 webpack loader가 찾아서 적용한다.
- babelrc의 경우 storybook을 사용할때 기본 바벨 설정이 되기도 해서, 루트 디렉토리에 바벨 설정 파일을 두는게 확장성에는 살짝 좋을 수 있다. 웹팩만을 위한 로더 설정인지 다른 런타임에도 적용 가능한 설정인지가 좀 다른 것

### Babel preset, plugin running order

바벨 플러그인과 프리셋은 어떤 순서로 적용될까?

- [바벨 독스에 따르면](https://babeljs.io/docs/en/plugins#plugin-ordering) 플러그인들은 프리셋 이전에 실행된다.
- 순서가 좀 독특함. 플러그인은 배열의 앞 인덱스부터 뒤로, 위에서 아래로 실행되고
- 프리셋은 배열의 맨 뒤 인덱스부터 앞으로, 즉 아래에서 위로 실행된다.
- 프리셋이 플러그인의 집합 정도의 의미라는 점에서, 사실 플러그인과 프리셋은 동등한 가치를 갖는다. 그렇기 때문에 플러그인과 프리셋을 섞어쓰면 플러그인들의 의도한 호출 순서가 꼬일 가능성이 있다. 어쩌면 바벨에 내제된 리스크일지도..?
- 관련해서 [논의와 PR](https://github.com/babel/babel/pull/5735)이 진행중이었다가 엎어진 것 같은데 최근에는 잘 모르겠다.
- 라이브러리 툴링 관련해서 문제가 생긴다면 바벨 플러그인과 프리셋의 순서를 살펴봐도 좋을 듯 싶다

### Webpack Loader applying order

웹팩 로더는 어떤 순서로 적용될까?

[스택오버플로우](https://stackoverflow.com/questions/32234329/what-is-the-loader-order-for-webpack)에 따르면

이거랑

```js
{
    test: /\.css$/,
    loaders: ['style'],
},
{
    test: /\.css$/,
    loaders: ['css'],
},
```

이게 같다

```js
{
    test: /\.css$/,
    loaders: ['style', 'css'],
},
```

- 수직으로 로더 전체 배열 안에 나열하면 위에 오는게 먼저 적용되고
- 수직으로 test아래 로더 배열에 적용하면 뒤에 오는게 먼저 적용된다
- 함수로 연달아 감싼다고 이해하면 쉽다 `style(css(file))`
- 그래서 SCSS 같은 경우 로더로 처리할때 `css-loader` `sass-loader` 이 순서로 두는 것
- 근데 한 파일 형식에 여러 개의 로더 설정을 두는게 쓸데없는 반복인거 같아서, 후자가 더 깔끔한 방식인듯 하다. 프로젝트의 어떤 자원이 있는지를 더 잘 표현하는 방식이기도 하고.
- 가령 tsx파일을 ts-loader와 babel-loader 두 개로 트랜스파일한다고 하면 대충 이런 식이 되는 것이다.

```js
{
    test: /\.tsx?$/,
    loaders: ['babel-loader', 'ts-loader'],
},
```

## 두 군대에서 할 수 있어서 선택의 여지가 있는 것

일반적인 리액트 프로젝트 기준 바벨과 타입스크립트 컴파일에서 모두 할 수 있는 트랜스파일 : Typescript, JSX, ES5

한 쪽에서 다 하는게 가능하다

### Typescript

> babel-typescript preset vs webpack ts-loader

- [이거는 꽤나 잘 알려져있는](https://ui.toast.com/weekly-pick/ko_20181220) 선택의 여지라고 할 수 있다.
- 위 글에서 두 개의 컴파일러를 함께 엮어 사용하는 것이 쉬운 일이 아니라는 말이 나오는데 동감한다. 순서도 굉장히 헷갈리고 신경써야 할 부분이 많다(TS -> TS Compile -> JS -> Babel -> JS) 
- 바벨이 컴파일이 더 빠르다 : 바벨은 타입스크립트를 **몽땅 제거**한다. 
  - ts-loader는 node_module 등에서 d.ts를 스캔하고 올바르게 동작하는지 확인하고, 옵션에 따라 index.d.ts를 만들어 줄 수도 있기 때문에 더 느리다.
  - 패키지 타입 선언이 포함되어야 하는 라이브러리 제작시에는 못 쓸 수도..? : 타입 선언을 만들어내는게 [지향점이 아니라고 한다.](https://github.com/babel/babel/issues/9668#issuecomment-602221154) 프리셋은 그냥 트랜스파일링만 한다. [이렇다고도 하고](https://stackoverflow.com/questions/62732580/creating-react-component-libraries-using-typescript-vs-babel-compiler)
  - [빌드 타임 비교](https://github.com/JaeYeopHan/tip-archive/issues/30)
  - 정말 엄청큰 프로젝트를 다뤄보고 비교해보지는 않았는데, 빌드 시간을 줄여야 할 경우 프로덕션 빌드에 바벨을 사용할 수도 있을듯
- **빌드시 타입을 체크하지 않는다** : 난 이게 좀 걸려서 아직까지는 ts-loader을 쓰는 편이기는 한데, eslint나 isolatedModules등의 기능을 켜면 어느정도 보완이 가능하다.
  - 그래도 [보일러플레이트 만들어보며](https://github.com/MaxKim-J/react-boilerplate) 꽤 괜찮다는 생각이 들었다..
- 미지원 문법 : 초창기의 babel-preset에는 const enum, class proeprties, object-rest-spread 등의 문법이 미지원이었음
  - preset-env랑 같이 쓰면 어느정도는 해결이 됨
  - 필요하면 플러그인 사용
  - tsc 쪽이 더 잘 지원된다고 알고 있는데 업데이트 거듭하면서 babel 쪽도 어떻게 개선되었는지 궁금

### JSX

#### JSX에 대한 사전정보

이거 따로 폴더 파서 알아봐도 좋을듯~

- React.createElement 함수에 대한 문법적 설탕이다.
- (원래는) React가 스코프 내에 존재해야 했다. React.createElement를 호출하는 코드로 컴파일되기 때문에 React 라이브러리 역시 JSX 코드와 같은 스코프 내에 존재해야만 했음
  - (이제는) 아님 [React 17 New Transform JSX](https://ko.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)
  - React.createElement 이런식으로 React에 의존하는 코드가 아니라, react/jsx-runtime이라는 패키지의 jsx 함수로 바꿔 JSX 문법을 핸들한다.

#### babel react-preset vs typescript jsx react option

- JSX에 대한 컴파일 옵션은 @babel/react-preset을 사용하거나 Typescript의 tsconfig의 JSX 옵션에서 사용할 수 있다.
- [이 글에 따라](https://ko.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html) 아래와 같이 하면 new JSX Transform을 지원하면서 결과물이 같기 때문에, 결국 순서의 차이라고도 할 수 있다.

```jsx
// babelrc
{
  "presets": [
    ["@babel/preset-react", {
      "runtime": "automatic" // 이렇게 설정해주면 jsx() 이렇게 바뀐다 default는 classic이다.
    }]
  ]
}

// tsconfig
{
  jsx: 'react-jsx',
  ...
}
```
- JSX가 처리되어야 하는 시점은 타입스크립트 트랜스파일 끝나고 ES5 컴파일을 하기 이전 시점이다. 그 타이밍이 보장될 수 있게만 처리해주면 된다.
- 타입스크립트를 아예 쓰지 않는 프로젝트라면 @babel/preset-react를 쓸수밖에 없을듯? 타입스크립트를 쓰는 환경이라 선택지가 생긴 것

#### Emotion을 쓰는 경우

이 경우에도 두 가지 선택지가 있다.

- babel에서 JSX를 처리하는 경우

tsconfig에서 JSX를 건드리지 않을 것이라면 `JSX:'preserve'`로 설정하고 React Preset을 사용하며, babel-preset-css-prop이라는 프리셋을 사용한다.

https://github.com/emotion-js/emotion/blob/main/packages/react/src/jsx.js

```json
{
  "plugins": [
    ["@babel/plugin-transform-runtime", {"corejs":3}]
  ],
  "presets": [
     [
      "@babel/preset-env",
      {
        "targets": {"chrome": "58"},
      }
    ],
    "@babel/preset-react",
    "@emotion/babel-preset-css-prop", // Emotion의 JSX Fragment를 제거하고
    // React.createElement를 감싼 emotion 패키지의 jsx 함수로 css prop을 적용시킨다
  ]
}
```

JSX Transform을 사용하는 경우에 emotion은 다음과 같은 구조를 추천하고 있다.

`@emotion/babel-preset-css-prop`에서 하는 [emotion 패키지의 jsx 함수가 React.createElement를 포함하고 있기 때문](https://github.com/emotion-js/emotion/blob/main/packages/react/src/jsx.js)에 React를 사용하는 것은 매한가지이기 때문이다.

```json
{
  "plugins": [
    "@emotion", // 가장 먼저 이모션 플러그인이 먼저 템플릿 리터럴 CSS를 함수로 바꾼다
    ["@babel/plugin-transform-runtime", {"corejs":3}]
  ],
  "presets": [
    [
      "@babel/preset-env",
      {
        "targets": {"chrome": "58"},
      }
    ],
    [
      "@babel/preset-react", // 이모션 플러그인 작동 이후 JSX를 emotion에 맞는 포맷으로 처리
      {
        "runtime": "automatic",
        "importSource": "@emotion/react"
      }
    ],
  ]
}
```

- tsconfig에서 해결하는 경우 : 이렇게 하면 자동적으로 JSX Transform도 적용된다.

```json
{
  "compilerOptions" : {
    "jsx": "react-jsx",
    "jsxImportSource": "@emotion/react",
    ...
  }
}
```

- tsconfig에서 해결하는 경우, 바벨 동작에 앞서 무조건 JSX와 Emotion을 해결할 수 있어 바벨보다 먼저 해결되는게 보장된다.
- 개인적으로 tsconfig에서 하는게 좋은 것 같은게, 바벨 동작보다 먼저 돌아가는게 보장되고 추가적인 바벨 플러그인을 설치하지 않아도 되기 때문이다.
- preset-react는 자바스크립트 프로젝트에서만 쓰는게 맞지 않을까..? 싶은 생각이 살짝 든다.

### ES5

> babel preset-env vs typescript target ES5

- 이거는 바벨이 그냥 넘사다. 바벨의 원래 존재 이유였기 때문에 ts-loader같은 친구보다 더 잘할 수 밖에 없다
- tsconfig로는 Module Target이랑 ES Target 정도밖에 설정 못하지만 babel로는 타겟 브라우저와 여러 플러그인 설정으로 정말 원하는 결과물을 더 다채롭게 낼 수 있다.
- 여러 편의 기능도 지원한다. 트리 쉐이킹을 위해 클래스 문법에 [퓨어 어노테이션도 알아서 붙여준다.](https://babeljs.io/blog/2018/08/27/7.0.0#pure-annotation-support)

## 기타 등등 알면 좋은 것

### tsconfig의 JSX 옵션

![음](./jsx.png)

- preserve : JSX를 몽땅 살리고 babel같은 다음 변환 단계에서 JSX를 트랜스파일
- react : React.createElement로 바꾸고 다음 단계에서 JSX 변환이 필요 없다.(= @babel/react-preset이 필요 없다)
- react-jsx : React.createElement가 아니라 _jsx로 바꾼다 => new transform JSX, import React 이 구문을 쓸 필요가 없어진다.

### Babel Tree Shaking friendly Setting

- ESM이 정적 분석과 트리쉐이킹에 최적화되어있기 때문에 ESM으로 작성된 상태를 webpack에서의 terser같은 도구가 dead code를 제거하기 전까지 유지해야한다.

```json
// tsconfig
{
  "compilerOptions" : {
    "target": "ESNext",
    "module": "ESNext",
    ...
  }
}

// babel
{
  "plugins": [
    ["@babel/plugin-transform-runtime", {"corejs":3}]
  ],
  "presets": [
    [
      "@babel/preset-env",
      {
        "targets": {"chrome": "58"},
        "modules": false,
        "loose": true
      }
    ]
  ]
}
```
  
- tsconfig 설정: tsconfig에서도 ESM을 유지해야 한다.
  - 모듈 설정을 ESNext, 혹은 ES6로 한다. ESNext로 하면 동적 임포트를 사용할 때 에러가 안 뜬다.
  - 그러면 모듈이 그대로 살아 babel로 넘어간다.
  - [모듈을 CommonJS로 설정하지 마쎄용](https://webpack.kr/guides/typescript/#loader)
- preset-env의 modules 설정 : false로 설정하면 preset env에서 ES6 모듈만 남도록 할 수 있다. 
  - 처음에 preset-env가 ES5 결과물을 내야 하는데 commonjs가 아니라 ESM이 그대로 남아있으면 어떡하지? 싶었는데 이 과정은 호환성 문제를 일으키지 않는다. 결국 webpack이 코드를 [범용적으로 사용할 수 있는 형태로 변환해주기 때문이다.](https://ui.toast.com/weekly-pick/ko_20180716#babel%EB%A1%9C-es6-%EB%AA%A8%EB%93%88%EC%9D%B4-commonjs-%EB%AA%A8%EB%93%88%EB%A1%9C-%EB%B3%80%ED%99%98%EB%90%98%EB%8A%94-%EA%B2%83-%EB%A7%89%EA%B8%B0)(여러 디펜던시들을 하나로 번들링)
