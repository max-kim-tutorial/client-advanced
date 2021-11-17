# 트리 쉐이킹

라이브러리를 제공하는 쪽에서 더 중요하기는 하다.

## 트리 쉐이킹이란?

- 사용하지 않는 코드를 제거하기 위해 JS 컨텍스트에서 일반적으로 사용되는 용어
- **정적 분석**을 통해 안 쓰는 코드를 번들에 포함시키지 않는다.

## 왜 ESM이 트리쉐이킹에 더 유리한가?

JS 모듈 시스템에 대해서는 `22_module-system` 참조

### ESM 모듈의 정적 구조(Static Module Structure)

- ES6의 `import`, `export`는 정적 모듈 구조를 가지고 있다. 
- 이 말은 컴파일 타임에 임포트와 익스포트를 결정할 수 있다는 말이 된다. 기존의 commonJS의 `require`는 런타임에 실행되어야 하는 코드였다. 그래서 파일 맨 위에다가 써야 하고, 동적인 부분은 없다.
- CommonJS에서는 바벨같은 거 없이 require만 가지고 이렇게 하는게 가능했다. 

```js
var my_lib;
if (Math.random()) {
    my_lib = require('foo');
} else {
    my_lib = require('bar');
}
```

#### 정적 구조를 강제함으로써 오는 장점 

- **트리 쉐이킹은 이 정적 구조때문에 가능하다** : 개발 과정 중에는 모든 코드가 존재할 수 있지만 프로덕션 빌드는 사용이 실질적으로 되는 코드들만 포함이 되어야 한다. 정적 모듈 시스템을 가지고 있다면 번들 타이밍에 분석을 통해 사용하지 않는 코드들을 가려낼 수 있다.
- 정적 구조는 **특정 모듈이 어떤 다른 모듈에서 확실히 사용되는지 아닌지 제대로 판단할 수 있게 해준다.** `require()`같은 경우는 조건부로 로딩될 수도 있기 때문에 불확실하다.
- import는 export의 read-only 버전으로, 해당 모듈을 복사해서 가져오지 않고 바로 참조하도록 만든다.
- CommonJS와 ESM의 차이 중 하나는 객체 프로퍼티 lookup을 하느냐에 대한 것이다.
  - CommonJS는 모듈 하나를 객체에 담아서 리턴하는 식이기 때문에 프로퍼티 이름으로 참조해야하니 동적이고, Lookup이 일어나며, 느리다.
  - ESM을 쓰면 정적으로 사용할 자원들을 namedExport로 미리 알 수 있고 접근을 최적화할 수 있다.

## 트리쉐이킹을 돕는 도구들

### 1. package.json/sideEffects

- sideEffect : 웹팩의 트리 쉐이킹은 사이드 이펙트가 없는 경우에만 이루어지고, 분석하기 어렵거나 사이드 이펙트를 일으키는 경우는 트리쉐이킹 되지 않는다.
  - 특정 코드를 쓰지 않는다고 해서 다 제거하면 안된다. 쓰지 않더라도 다른 변수나 함수에 영향을 미칠 수 있는 코드일 수 있기 때문에, 이런 경우에는 트리쉐이킹 할 수 없어진다.
- 정적 구조의 장점에서 보았듯이, 100%의 ESM 모듈에서는 어떤 코드가 쓰이는지, 혹은 쓰이지 않는지에 대해 사이드 이펙트를 쉽게 식별할 수 있다.
- 하지만 100% ESM만으로만 개발하는 것은 아직 쉽지 않으므로(라이브러리까지 다 생각해보면) 코드의 순수성에 대한 힌트를 webpack 컴파일러에 제공할 수 있다. 그게 package.json의 sideEffects 속성이다.

```json
{
  "name": "your-project",
  "sideEffects": false
}
```

- false로 표시하면 사용하지 않는 export는 모두 제거해도 괜찮다는 것을 webpack에 알릴 수 있다. 사용하지 않는 것들이 사이드 이펙트를 발생시키지 않을 것을 개발자가 보장하는 셈

```js
{
  "name": "your-project",
  "sideEffects": ["./src/some-side-effectful-file.js"]
}
```
- 배열과 정규 표현식등을 바탕으로 개발자 입장에서 사이드 이펙트가 없다고 확신이 되는 파일을 지정해줄 수가 있다. 이렇게 지정해놓으면 트리쉐이킹의 대상이 된다. sideEffect가 발생할 수는 있지만 실제로는 영향이 없는 경우에 이렇게 지정해줄 수 있다.

#### Pure Annotation

- 함수 호출 앞에 붙여 해당 함수 호출이 side-effect가 없음을 알릴 수 있음. 
- Webpack뿐 아니라 uglify를 실행하는 terser도 트리 쉐이킹에 관여한다. export가 제거된 경우 해당 코드를 제거하는 역할을 하고 있다.
- pure annotation이 앞에 붙으면 side-effect가 없음을 알릴 수 있다. 
- typescript를 babel로 컴파일하면 pure annotation이 붙는다고 하는데 이거는 잘 모르겠다

#### 그외

- CJS는 아예 트리 쉐이킹의 대상이 아니다.
- 바벨을 사용한다면 바벨단에서 ESM을 CJS로 바꾸지 않도록 설정해야한다.
- 타입스크립트 컴파일을 사용하는 경우에도 ESM을 유지해야 할 듯 싶다.
- production 모드에서 최적화가 더 빡세게 된다.

## 실험 - 문법 관점 트리쉐이킹

export가 되었는데 안 쓰이는 특정 함수(혹은 메서드)가 트리쉐이킹이 잘 되는지 어쩌는지 살펴보면서 인사이트를 얻어보자. export import는 주로 함수나 객체 단위로 이루어지기 때문에 함수로 실험한다. 제일 알고 싶은것은 뭉쳐서 import를 했을때와 따로 import를 했을때의 차이인것 같군

webpack이 알아서 제거해줘야할 함수는
1. sideEffect가 없고
2. 번들에서 안 쓰인다

사실 아예 이 실험이 별로... 의미가 없을 수 있다.

### 1. require vs ESM

### 2. Object export vs function named export

함수 뭉치를 객체에 넣어 export default 해서 사용했을 때 VS 각 함수를 따로 export해서 사용했을 때

### 3. class default export vs object default export

### 4. class default export vs function named export

### 5. function export로 해놓고 쓰지 않았을 때

뭐 eslint같은걸 키면 이런 상황을 막아줄 수 있지만 혹시모르니까 해본다


## 레퍼런스

- https://medium.com/naver-fe-platform/webpack%EC%97%90%EC%84%9C-tree-shaking-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0-1748e0e0c365
- https://webpack.kr/guides/tree-shaking/