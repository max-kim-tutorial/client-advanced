# Babel

## Babel이 해주는 일

- 바벨은 ES6+ 코드를 ES5로 변환하는 도구이다. 
- 크로스 브라우징 : 브라우저마다 사용하는 언어가 달라서 FE 코드는 일관적이지 못하기 쉽다. IE는 Promise를 이해하지 못한다. 이때 ES6+로 작성한 코드를 모든 브라우저에서 동작하도록 호환성을 지켜준다.
- 컴파일과 다른 점 : 트랜스파일은 추상화 수준을 유지한 상태로 코드를 변환한다. 그러면 TS로더도 트랜스파일이군
- ES5에 존재하지 않는 ES6의 메서드나 생성자는 지원하지 않는다. => 바벨로 트랜스파일링 해도 Promise와 같은 키워드는 그대로 남아있다.

### 바벨의 컴파일 동작

- 파싱 : 코드를 읽고 추상 구문 트리(AST)로 변환한다. 이것은 빌드 작업을 처리하기에 적합한 자료구조로, 컴파일러 이론에 사용되는 개념이다.
- 변환 : 추상 구문 트리를 변경한다. 실제로 코드를 변경하는 작업을 한다.
- 출력 : 추상 구문 트리를 다시 코드로 변경한다
- 바벨은 파싱과 출력만 담당하고, 변환 작업은 플러그인이 담당한다.

## plugin

### 커스텀 플러그인

커스텀 플러그인은 아래와 같이 만든다.

```js
// myplugin.js:
// visitor 객체를 가진 함수를 반환한다.
// 바벨이 파싱하여 만든 추상 구문 트리(AST)에 접근할 수 있는 메소드를 제공한다.
module.exports = function myplugin() {
  return {
    visitor: {
      Identifier(path) {
        const name = path.node.name

        // 바벨이 만든 AST 노드를 출력한다
        console.log("Identifier() name:", name)

        // 변환작업: 코드 문자열을 역순으로 변환한다
        // path를 통해 코드 조각에 접근할 수 있다.
        path.node.name = name.split("").reverse().join("")
      },
    },
  }
}
```

아래는 const를 var로 변경하는 플러그인이다. block-scoping 플러그인은 const, let처럼 블록 스코핑을 따르는 예약어를 함수 스코핑을 사용하는 var로 변경한다.

```js
// myplugin.js:
module.exports = function myplugin() {
  return {
    visitor: {
      // https://github.com/babel/babel/blob/master/packages/babel-plugin-transform-block-scoping/src/index.js#L26
      VariableDeclaration(path) {
        console.log("VariableDeclaration() kind:", path.node.kind) // const

        // 이런 식으로 키워드에 접근할 수 있다.
        if (path.node.kind === "const") {
          path.node.kind = "var"
        }
      },
    },
  }
}
```
## preset

필요한 플러그인을 일일이 설정하는 일은 무척 지루한 일이다. 코드 몇 줄 작성하려고 플러그인 세팅을 몇개나 해야할 수도 있다. 목적에 맞게 여러가지 플러그인을 세트로 모아놓는 것을 프리셋이라고 한다.

```js
// 프리셋은 플러그인들의 집합이다.
module.exports = function mypreset() {
  return {
    plugins: [
      "@babel/plugin-transform-arrow-functions",
      "@babel/plugin-transform-block-scoping",
      "@babel/plugin-transform-strict-mode",
    ],
  }
}

module.exports = {
  presets: ["./mypreset.js"],
}
```
### 주요한 프리셋

- preset-env : ES6를 변환할때 사용한다. 바벨 7 이전에는 연도별로 각 프리셋을 제공했지만 지금은 env 하나로 합쳐졌다.
- preset-react : React의 JSX를 React의 CreateElement 함수로 변경한다.
- preset-typescript: 타입스크립트 변환을 위한 프리셋이다.

### @babel/preset-env

```js
module.exports = {
  presets: [
    [
      "@babel/preset-env",
      {
        targets: {
          chrome: "79",
          ie: "11", // ie 11까지 지원하는 코드를 만든다
        },
        useBuiltIns: "usage", // 폴리필 사용 방식 지정
        // false가 아니라 다른 걸로 설정하면 core-js를 모듈로 가져옴
        corejs: {
          // corejs 버전 지정
          version: 2,
        },
      },
    ],
  ],
}
```
- 명시된 target에서 지원하지 않는 JS 문법을 확인하여 @babel/plugin-을 추가해주는 방식으로 작동한다.
- target: 브라우저 버전명만 지정하면 env 프리셋은 이에 맞는 플러그인들을 찾아 최적의 코드를 출력한다.
- polyfill : Promise와 같은 ES6문법은 ES5문법에서 대체할만한 키워드가 없다. 바벨은 ES6+를 ES5로 변환할 수 있는 것만 빌드한다. 그렇지 못한 것들은 폴리필이라는 코드 조각을 추가해서 해결한다. 폴리필을 추가하면 아래와 같이 core-js에서 코드 조각을 가져와 Promise같은 문법을 트랜스파일한다. Promise를 포함해서 Object.Assign, Array.from 등이 이런 문법들에 해당한다.

```js
npx babel src/app.js
"use strict";

// 이렇게 어디서 뭔가를 가져오게 된다
require("core-js/modules/es6.promise");
require("core-js/modules/es6.object.to-string");

new Promise();
```
- useBuiltins : preset-env의 폴리필 삽입 방식 지정. false이외의 옵션을 설정하면 core-js 모듈을 가져오는 코드를 타깃 브라우저에 맞게 삽입, 수정한다. 
  - entry로 설정하면 transpile 하는 시작점에 import된 core-js모듈과 regenerator-runtime모듈의 삽입문을 하위 특정 모듈들의 삽입문으로 변경시켜  target에 맞게 변경한다.
  - usage로 설정하면 실제 사용한 폴리필만 삽입한다. import문 변경이 아닌 삽입. 전역 스코프에 삽입하지 않는다.
```js
//in
var a = new Promise();
var b = new Map();

//out(실제 사용한 폴리필만 삽입하도록 import 문이 추가된다)
import "core-js/modules/es.promise";
import "core-js/modules/es.map";
var a = new Promise();
var b = new Map();
```

## polyfill 더 자세히 알기

- babel@7.4.0 이전에는 @babel/polyfill을 많이 사용했지만 이제는 @babel/preset-env로 통합하여 사용한다.
- @babel/polyfill : polyfill regenerator runtime과 ES5/6/7 polyfill인 core-js를 dependency로 가지고 있는 패키지이다. 두 의존성에 있는 것을 임포트하는 역할만 한다.
  - 전역 공간에 폴리필을 채워 넣는 방식(전역 객체를 직접 수정)이기 때문에 전역 공간이 오염되어 폴리필 충돌 가능성이 있다는 단점이 있다.
  - 특정 서비스를 다른 서비스에 붙이는 경우(채팅 SDK같은거) 붙이는 프로그램의 전역 객체에 바벨 폴리필이 들어가 있는 경우 기존 사이트의 전역 스코프를 오염시킬 수 있다
  - 브라우저에서 이미 구현된 필요하지 않은 폴리필까지 전부 번들에 포함되어 번들 크기가 커지는 단점이 있다 => 아 그렇네 polyfill은 **실제하는 코드 조각을 import 해온다.**
- core-js: 전역에 Polyfill을 추가하기 이전에 해당 기능이 있는지를 체크하기 때문에 최신 브라우저에서는 polyfill 없이 동작하게 되어 plugin-transform-runtime을 사용하는 것에 비해 빠르다.

### @babel/plugin-transform-runtime

무거운 폴리필 대신에 필요한 부분에만 자체 헬퍼함수를 적용할 수 있도록 돕는 플러그인. 이것 자체가 polyfill을 삽입하도록 하는 또 하나의 방법이다.

- 트랜스파일 과정에서 polyfill이 필요한 부분의 동작을 helper(자체 구현한 함수 - core-js-pure)로 치환한다. alias 목록에 따라 전역 객체에 polyfill을 우겨넣는 수정 없이 내부 helper 함수로 치환하는 방식으로 적용되도록 한다.
- Babel은 내부적으로 _extend처럼 공통 함수를 위한 작은 헬퍼들을 사용하는데, 기본적으로 모든 파일에 이런 헬퍼들이 추가되기 때문에 낭비를 막기 위해 @babel/runtime, corejs를 참조하는 방식으로 변경된다.
- babel polyfill은 전역을 직접 수정하기 때문에 npm을 통해 설치한 디펜던시들이 Promise와 같은 새로운 내장 객체를 사용하는지 여부를 걱정할 필요가 없다. 전역 공간에 polyfill이 존재해야 하는 경우에 트랜스파일이 제대로 안 될 수 있다. 외부 모듈이 전역 공간에 선언된 최신 객체를 필요로 할 경우 매번 webpack의 include 옵션에 포함시켜줘야 함
- Object.assign 처럼 내장 객체의 static 메소드는 사용 가능하지만 새롭게 추가된 인스턴스 메서드와([].includes(1)) 같은 메서드는 트랜스파일링 되지 않기때문에 오류가 발생한다. => [현재 core.js3에서는 인스턴스 메소드를 지원한다](https://github.com/zloirock/core-js/blob/master/docs/2019-03-19-core-js-3-babel-and-a-look-into-the-future.md#changes-in-javascript-standard-library-content)
#### 하는 일

- @babel/runtime/generator을 require함 - async, await 해결
- Core-js으로 폴리필 대체
- inline Babel Helper을 제하고 @babel/runtime/helpers에 있는 모듈을 사용하여 재사용성 극대화
### core-js@3

궁극체. 전역 폴리필이 없어져 plugin-transform-runtime의 ES6 모듈들도 core-js-pure를 사용할 수 있도록 babel-loader에 포함해줘야 한다는 단점과, 폴리필의 전역 객체 오염 문제를 해결함. 인스턴스 메서드도 지원
### preset env core-js 3 vs runtime core js 3 설정

corejs@3 옵션은 preset인 env와 plugin인 transform-runtime에서 동작하는데 둘 중 하나만 적용하면 됨. transform runtime을 사용하면 폴리필이 전역 객체에 삽입되지 않으므로 useBuiltIns 옵션이 필요 없게 되는 듯 함

```js
// preset env에 넣기
"presets": [
    ["@babel/preset-env", {
      "targets": {
        "browsers" : ["last 2 versions", "ie >= 11"]
      },
      "useBuiltIns": "usage",
      "corejs":3,
      "shippedProposals": true
    }]
  ],

// plugin에 넣기
  {
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "corejs": 3,
        "regenerator": true,
      }
    ]
  ]
}
```
## 그 외

### regenerator-runtime/runtime

- babel이 async/await 문법을 변환하는데 사용하는 구현 모듈
- babel-polyfill 안에는 들어있었다.
- babel-runtime 안에도 들어있다.
- core-js3에는 포함이 되어 있는듯?
- ts es-nest로 설정하고 preset env에서 corejs 2 사용하도록 설정하면 에러가 나는 경우가 있었음

### target 설정

- target 설정이 따로 없으면 target 브라우저에서 돌아가는 문법인지를 따지지 않고 모든 문법을 ES5로 변환한다. 필요한 변환만 하기 위해서는 필요하다.

## 어디까지 트랜스파일링 해야하나? 

- 뭔가... 정답이 없는 문제 같다.
- IE 지원을 해야한다면 당연히 타겟에 추가를 해야하겠다
- 하지만 크롬 버전만 가지고 따진다면... 너무 옛날 버전 정도만 버리는 정도로 타겟을 설정하면 되지 않을까 함. 58? 59(2017년쯤 나옴) 정도로 많이 맞추는 듯

# 레퍼런스

- https://jeonghwan-kim.github.io/series/2019/12/22/frontend-dev-env-babel.html
- https://babeljs.io/docs/en/babel-plugin-transform-runtime
- https://so-so.dev/web/you-dont-know-polyfill/
- https://tech.kakao.com/2020/12/01/frontend-growth-02/
- https://programmingsummaries.tistory.com/401