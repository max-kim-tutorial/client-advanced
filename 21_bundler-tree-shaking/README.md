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

## Side Effects

- sideEffect : 웹팩의 트리 쉐이킹은 사이드 이펙트가 없는 경우에만 이루어지고, 분석하기 어렵거나 사이드 이펙트를 일으키는 경우는 트리쉐이킹 되지 않는다.
  - 특정 코드를 쓰지 않는다고 해서 다 제거하면 안된다. 쓰지 않더라도 다른 변수나 함수에 영향을 미칠 수 있는 코드일 수 있기 때문에, 이런 경우에는 트리쉐이킹 할 수 없어진다.

### SideEffect가 발생하는 경우

1. 전역 함수 사용(Object, Math, String, RegExp)
2. 함수 실행 코드에서 멤버변수를 변경하고 반환하는 경우
3. Static Class Properties를 사용하는 경우
4. Class를 사용하는 경우 => babel 7부터는 class를 babel로 변환시 PURE Annotation이 붙어 트리 쉐이킹을 한다.
5. Import한 라이브러리를 사용한 함수를 사용하지 않는 경우 => b를 사용하지 않으면 b는 제거되더라도 b안에 a가 있기 때문에 a는 side effect가 뜨면서 제거되지 않음(Webpack3)
6. import한 라이브러리를 다시 export default 한 경우(Webpack 3) => export default만 한 경우 사용하지 않더라도 참조하는 코드 추가

## 트리쉐이킹을 돕는 도구들

### 1. package.json/sideEffects

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

export가 되었는데 안 쓰이는 특정 함수(혹은 메서드)가 트리쉐이킹이 잘 되는지 어쩌는지 살펴보면서 인사이트를 얻어보자. export import는 주로 함수나 객체 단위로 이루어지기 때문에 함수로 실험한다. 제일 알고 싶은것은 뭉쳐서 import를 했을때와 따로 import를 했을때의 차이

webpack이 알아서 제거해줘야할 함수는
1. sideEffect가 없고
2. 번들에서 안 쓰인다

사실 아예 이 실험이 별로... 의미가 없을 수 있다. 코드가 허접하고 간단하게만 테스트해볼거라... 사실 트리쉐이킹을 하는데는 많은 알고리즘이 필요하고, 애매한 상황도 엄청 많을 것이다.

### 실험 환경

난수화된 상태 그대로 파일을 inspect할 경우 어떤 함수가 사라졌는지 알지를 못한다  
따라서 Terser 플러그인과 몇가지 optimization 설정을 통해 확인을 잘 할 수 있는 상황을 만든다.

- TerserPlugin 옵션을 설정해 함수와 클래스 이름을 남겨준다
  - 참고) Terser을 아예 끄면 스플리팅이 되지 않는다. 죽은 코드를 삭제하는 일을 하기 때문
- 런타임 청크를 분리하여 번들에는 깨끗한 프로젝트 코드만 남긴다
- 바벨, 타입스크립트 등 번들링에 영향을 줄 수 있는 도구들을 (일단) 사용하지 않는다.

#### 코드

```js
// b.js
const add = (a,b) => {
  return a + b
}

const multiply = (a,b) => {
  return a*b
}

const subtract = (a,b) => {
  return a - b
}

const divide = (a,b) => {
  return a/b
}

module.exports = {
  add,
  multiply,
  subtract,
  divide
}
```

```js
// a.js
let c = 1;
export const add = (a,b) => {
  c++;
  return a + b + c;
}

// Side Effect가 없음
export const multiply = (a,b) => {
  return a*b
}

export const subtract = (a,b) => {
  return a - b
}

export const divide = (a,b) => {
  return a/b
}

```

```js
import { add, multiply } from './a';
const b = require('./b');

const c = add(1,2) + b.divide(4,2)
console.log(c)

console.log(multiply(2,4))
console.log(add(1,2))
console.log(b.divide(4,2))
```

### 1. `require` vs `ESM`

CJS named import?

```js
(self.webpackChunkbare_webpack_experiment = self.webpackChunkbare_webpack_experiment || []).push([[179], {
    996: e=>{ // require로 부른 모듈은 트리쉐이킹이 되지 않음
        e.exports = {
            add: (e,o)=>e + o,
            multiply: (e,o)=>e * o,
            subtract: (e,o)=>e - o,
            divide: (e,o)=>e / o
        }
    }
    ,
    104: (e,o,l)=>{
        "use strict";
        let s = 1;
        const add = (e,o)=>(s++, // add는 사이드이펙트가 있으므로 번들에 포함
        e + o + s)
          , c = l(996)
          , i = add(1, 2) + c.divide(4, 2);
        console.log(i),
        console.log(((e,o)=>e * o)(2, 4)), // export를 한 multiply
        console.log(add(1, 2)),
        console.log(c.divide(4, 2))
    }
}, e=>{
    var o;
    o = 104,
    e(e.s = o)
}
]);

```
- add는 이름이 그대로 유지된채 번들에 포함되었고 multiply는 익명 화살표 함수로 간소화되어 번들에 포함되었다. 왜 이런 차이가 발생하는 걸까? side effect 때문일지?
- CJS 모듈은 divide만 사용함에도 번들에 전부 다 합쳐졌다.

### 2. `Object export` vs `function named export`

함수 뭉치를 객체에 넣어 export default 해서 사용했을 때 VS 각 함수를 따로 export해서 사용했을 때. 어쨋든 둘다 ESM으로 불렀다.

```jsx
"use strict";
(self.webpackChunkbare_webpack_experiment = self.webpackChunkbare_webpack_experiment || []).push([[179], {
    29: ()=>{
        let e = 1;
        const add = (o,l)=>(e++, // add는 여전히 번들에 포함되었다.
        o + l + e)
          , b_divide$0 = (e,o)=>e / o // 오 이경우에는 모듈 b의 속성을 빼서 새로운 변수에 정의했다.
          , o = add(1, 2) + b_divide$0(4, 2);
        console.log(o),
        console.log(((e,o)=>e * o)(2, 4)),
        console.log(add(1, 2)),
        console.log(b_divide$0(4, 2))
    }
}, e=>{
    var o;
    o = 29,
    e(e.s = o)
}
]);
```

### 3. `class default export` vs `object default export`

비슷한 함수들로 클래스를 짰다.

```js
//c.js
class Calculator {
  constructor() {
    this.c = 1;
  }

  add(a, b) {
    c++;
    return a + b + c;
  }

  multiply(a, b) {
    return a * b
  }

  subtract(a, b) {
    return a - b
  }

  divide(a, b) {
    return a / b
  }
}

export default Calculator
```

```js
import Caculator from './c'
import b from './b'

const cal = new Caculator()

const c = cal.add(1,2) + b.divide(4,2)
console.log(c)

console.log(cal.multiply(2,4))
console.log(cal.add(1,2))
console.log(b.divide(4,2))

```

결과물은 이렇다.

```js
"use strict";
(self.webpackChunkbare_webpack_experiment = self.webpackChunkbare_webpack_experiment || []).push([[179], {
    202: ()=>{
        const e = class {
            constructor() {
                this.c = 1
            }
            add(e, r) {
                return c++,
                e + r + c
            }
            multiply(e, c) {
                return e * c
            }
            subtract(e, c) {
                return e - c
            }
            divide(e, c) {
                return e / c
            }
        }
          , b_divide = (e,c)=>e / c
          , r = new e
          , s = r.add(1, 2) + b_divide(4, 2);
        console.log(s),
        console.log(r.multiply(2, 4)),
        console.log(r.add(1, 2)),
        console.log(b_divide(4, 2))
    }
}, e=>{
    var c;
    c = 202,
    e(e.s = c)
}
]);
```

- 메서드를 일부만 쓴다고 하더라도 **클래스는 트리쉐이킹이 되지 않는다!!!!!!** 클래스와 생성자까지 온전히 번들에 포함된다. (물론 export한 번들에 한해) => 하긴 이게 상식적이다. 클래스는 런타임에 로직을 결정할 수 있어야 하기 때문에
- 원래 Object를 반환하는 export에도 비슷한 동작을 기대했었는데 아니었다. 
- 그러면 named export로 클래스를 임포트하면 어떨까 여전히 똑같다

### 4. side effect가 있는 코드를 named export로 해놓고 쓰지 않았을 때

```js
import b from './b'
import {add, multiply} from './a'

const c = multiply(1,2) + b.divide(4,2)
console.log(c)

console.log(multiply(2,4))
console.log(b.divide(4,2))
```

```js
"use strict";
(self.webpackChunkbare_webpack_experiment = self.webpackChunkbare_webpack_experiment || []).push([[179], {
    29: ()=>{
        const b_divide = (e,o)=>e / o;
        const a_multiply = (e,o)=>e * o
          , e = a_multiply(1, 2) + b_divide(4, 2);
        console.log(e),
        console.log(a_multiply(2, 4)),
        console.log(b_divide(4, 2))
    }
}, e=>{
    var o;
    o = 29,
    e(e.s = o)
}
]);
```

- 단순히 export만하고 쓰지 않은 경우에는 번들에 포함되지 않는다

### 6. 그러면 클래스는 어떻게 트리쉐이킹하나

몰흐게따..babel7, Pure Annotation 써봐도 잘 안되는데 잘 모르겠다  
찾아도 안나온다.. Rollup Treeshaking여기서 돌여도 class는 안사라진다.

### 7. import export가 많은게 영향을 미치나? 

뷰모델로 따로 분리하는 것과 컴포넌트 안에 로직들을 넣는 것에 대해 차이가 있는가? 보일러플레이트 코드가 추가되는가?

- React 컴포넌트 내부에 들어있는 경우 : 함수의 생성이 매 렌더링마다 생성이 되게 만들어야 하나?
- 밖에서 안으로 들어오는 경우
  - 같은 파일의 컴포넌트 밖에 있는 경우
  - 다른 파일에서 임포트해오는 경우

## 레퍼런스

- https://medium.com/naver-fe-platform/webpack%EC%97%90%EC%84%9C-tree-shaking-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0-1748e0e0c365
- https://webpack.kr/guides/tree-shaking/