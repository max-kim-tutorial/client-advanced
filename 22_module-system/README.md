# 모듈 시스템(CJS, ESM)

초기에 자바스크립트는 독립적인 작업을 수행하며 큰 스크립트가 필요하지 않았다. 하지만 제이쿼리가 생겨나고 애플리케이션의 규모가 커지면서 script 파일을 나누기 시작했고, 파일간의 변수, 함수 등을 전달하고 받는 방법이 필요했다. 

모듈이 나오기 이전에 브라우저에서는 각각의 script 파일을 전역 스코프처럼 활용했다. HTML 파일에서 보다 위에 있는 script 파일은 전역 스코프처럼 하위의 script 태그에서 접근과 변경이 가능했다.

## CJS vs ESM

둘은 호환이 띄엄띄엄 되지만 구현체는 전혀 다르다

- **CJS** : node.js 서버를 위해 만들어짐
- **ESM** : ES6에 표준으로 등재되었음

## require vs import/export

![개념이 다르다](https://oopy.lazyrockets.com/api/v2/notion/image?src=https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fbd7c9a98-f6fe-420a-818c-18ef1f5578b9%2FUntitled.png&blockId=8fe26dec-cb39-4ca6-9195-20477cfac55b)

- **require()** : CommonJS에서 require()은 동기 동작으로 정적으로 쓸 수 없다. 디스크 혹은 네트워크에서 파일을 읽어 그 즉시 스트립트를 실행한다. 따라서 스스로 I/O나 부수효과를 실행하고, `module.exports`에 설정되어 있는 값을 리턴한다. **`require()`는 실행문이고 모듈은 하나의 독립적인 프로그램이다.**
  - export 객체에 값을 복사해서 넣는다. -> 만일 export 하는 파일에서 비동기로 값이 바뀐다면 Common.js는 반영이 되지 않지만 ESM은 다시 반영된다.
- **import/export** : 반면에 ESM은 모듈 로더를 비동기 환경에서 실행한다. 먼저 가져온 스크립트를 바로 실행하지 않고, import/export 구문을 찾아 스크립트를 파싱한다.
  - export는 참조를 반환하는 함수를 정의한다.
  - 오..이래서 동적으로 import를 쓸 경우 then을 사용할 수 있었던 것일까?
  - 파싱 단계에서 실제로 ESM 로더는 종속성이 있는 코드를 실행하지 않고도 names imports에 있는 오타를 감지하여 에러를 발생시킨다. 
  - ESM 모듈 로더는 가져온 스크립트를 비동기로 다운로드하여 파싱한 다음, import된 스크립트를 가져오고 더 이상 import할 것이 없어질 때까지 import를 찾은 다음 의존성 모듈 그래프를 만들어내고 스크립트 실행 준비를 마친다.
- 브라우저에서 `<script>` 사용시 ESM이 기본이 아니다. `type=module` 옵션을 주고, this가 global object를 참조할 수 없게 해야되는 등 많은 부분에 변경이 필요하다.
- CJS(동기)가 ESM(비동기) 모듈을 require하지 못하는 이유는, ESM은 Top Level Await가 가능하지만 CJS는 안되기 때문이다. => `await require()`가 안된다. 

### 모듈 시스템이 섞여있을 때 가능한 것

- CJS는 ESM에서 즉시 실행 함수로 임포트가 가능하다. 근데 모양이 좀..

```js
;(async () => {
  const { foo } = await import('./foo.mjs')
})()
```
- ESM은 CJS의 names.exports를 import할 수 없다. CJS는 named exports를 실행단계에서 연산하지만, ESM은 named exports를 파싱 단계에서 연산하기 때문

```js
import _ from './lodash.cjs' // X
import { shuffle } from './lodash.cjs' // O

// 이렇게 하면 되는데 딱봐도 tree shaking에 지장을 주게 생김
import _ from './lodash.cjs'
const { shuffle } = _
```

- ESM을 CJS에서 require()로 가져올 수도 있지만 별로 권장되지 않는다. 이를 사용하기 위해서 많은 boilerplate가 필요하고, 번들러도 필요하다. ESM은 require()가 어떻게 작동하는지 모른다.
- CJS는 Node의 default이므로 ESM 모드를 사용하기 위해서는 .js를 .mjs로 바구거나, package.json의 `"type":"module"` 옵션을 넣어준다. 

## 라이브러리에서의 CJS, ESM 지원

- CJS는 유저들에게 더 친숙하고, 오래된 노드버전도 지원이 가능하기 때문에 CJS로 라이브러리를 제공하는 것은 좋은 방법이다.
- 그리고 별도의 디렉토리에 ESM 래퍼를 만들어 ESM으로 익스포트한다.
- 혹은 라이브러리 빌드 과정에서 ESM 빌드를 해도 될듯? - Rollup.. 이건 따로 알아봅시다
- package.json에 `exports.map`이란걸 만들 수 있다 : exports map은 사용자가 내가 의도적으로 노출한 entry point만 require/import를 할 수 있게 만든다. 그런데 이렇게 하면 breaking change라서, 이미 서비스 중인 라이브러리라면 버전을 올리는 과정에서 에러가 발생할 수 있다.

```json
"exports": {
    "require": "./index.js",
    "import": "./esm/wrapper.js"
}
```
