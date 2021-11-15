#  CSS-In-JS

SPA 개발에서의 UI 스타일링 그리고 여러 옵션들  
모든 상황에 완벽한 하나의 해결책은 없다는 것을 명심하자  

## CSS-In-JS가 무엇인가

- 자바스크립트 코드에서 CSS를 작성하는 방식
- 라이브러리간의 중요한 차별화 요소는 스타일을 얼마나 동적으로 작성할 수 있는가, JS 변수를 사용해서 스타일을 작성할 수 있는가
- 현대 웹은 컴포넌트로 작성되는데, CSS는 그동안 컴포넌트 기반의 방식을 위해 만들어진 적이 한번도 없다.
- CSS 모델을 문서 레벨이 아니라 컴포넌트 레벨로 추상화한다. 자바스크립트 환경을 최대한 활용하여 CSS를 향상시킨다.
- [CSS Loader로 CSS나 SCSS 그대로를 React에서 classname과 함께 쓰면 스타일링 작업에서 애매모호한 지점이 생긴다.](https://environmentset.github.io/2019/01/06/Stop-using-CSS-with-React/)
  - CSS의 선택자는 원래 대상을 찾아서 스타일링을 한다.
  - 리액트에서 className을 동적으로 줄 경우 선택자는 제대로 기능하겠지만 CSS가 해야하는 스타일링이라는 작업의 일부를 React Component가 하고 있게 된다. CSS가 선택자를 찾아가는게 아니라 React Component가 스타일을 입혀주고 있는 것
  - CSS와 React Component가 완벽하게 분리된 것도, 완벽하게 결합된 것도 아닌 어중간한 결합을 갖게 된다. 그래서 CSS를 수정하는 것 만으로 스타일링을 의도대로 바꿀 수 있을지에 대한 보장이 없어진다. UI 스타일을 바꾸고 싶은데 React Component와 CSS를 동시에 바꿔야 하는 경우가 생길 수 있다.
  - CSS-in-js는 CSS 코드를 React Component와 묶어버림으로써 CSS와 JS를 결합시켜버려, 이러한 어중간한 결합을 깬다.

### 간단한 역사

#### 1. 빌드할때 CSS 전처리

```js
import styles from './Button.module.css'

const Button = () => {
  return <button className={styles.submit}>제출</button>;
}
```


- **Webpack의 CSS loader, style-loader** : CSS loader는 CSS 파일을 읽어서 JS에서 사용 가능한 스트링으로 변환한다. 이때 CSS내부의 URL이나 import등은 path를 읽어서 JS에서의 import나 require로 바꿔줌. style-loader는 반환해준 값을 실제로 DOM에 style 태그로 삽입한다. 하는 역할이 다르기 때문에 두 개를 한번에 써준다.
- **CSS Module** : 처음부터 CSS를 JS 파일에서 사용할 수 있었던 것은 아니고, `*.(module).css` 으로 파일을 생성하고 CSS 전처리기를 사용하여 불러왔었다. Webpack CSS loader 설정에서 `modules:true` 옵션을 주면 사용이 가능하다. 컴포넌트에서 사용한 클래스명을 시스템에서 알아서 고유한 값으로 변화시켜주므로 다른 컴포넌트에 선언되어 있는 클래스명이 중복이 될 걱정 없이 스타일링을 할 수 있다.
- 이런 방식으로 CSS를 사용하면 런타임에서의 동작이 없으므로(컴파일시 다 들어가 있음) Zero runtime css-in-js가 된다.(CSS-in-js가 맞나?)

#### 2. JS 변수를 활용하여 CSS 작성 가능해짐

- Radium: JS 변수를 활용하여 CSS를 작성할 수 있으나 인라인 스타일을 사용하므로 가상 셀렉터(nth-childemd)를 사용할 수 없고, CSS의 모든 스펙을 사용할 수 없다는 단점

#### 3. JS 템플릿으로 CSS 작성하면 `<style>`태그를 생성하여 주입

- aphrodite, glamor
- 동적으로 변경되는 스타일은 정의가 까다로웠음 

#### 4. Runtime 개념 도입

```js
templateFunction.attrs = <Props = OuterProps>(attrs: Attrs<Props>) =>
  constructWithOptions<Constructor, Props>(componentConstructor, tag, {
    ...options,
    attrs: Array.prototype.concat(options.attrs, attrs).filter(Boolean),
  });
```

- StyledComponents, emotion
- prop이 변할 때마다 스타일을 동적으로 생성하여 JS 코드로 동적인 스타일링이 가능.
- build time에서 모든 스타일을 생성하는 것이 아니라 런타임을 활용하므로 계산 비용이 발생한다. 대부분 괜찮지만 스타일이 복잡한 컴포넌트에서는 차이가 발생할수도 있다. [이런것도 있군](https://necolas.github.io/react-native-web/benchmarks/)

#### 5. 다시 Zero runtime 

- 다시 컴파일 상황에서 스타일을 생성하되 동적 제어가 가능한 방법들이 시도되고 있다.
- linaria : StyledComponents와 비슷한 API를 가졌으며 제로 런타임으로 동작. babal plugin과 webpack loader을 통해 사용된 CSS 코드를 추출해서 정적인 스타일시트를 생성. 내부적으로 CSS variable을 사용하는데 스타일 시트를 새로 만들지 않고 CSS variable만 수정하여 달라지는 조건에 대해 스타일을 다르게 준다. IE는 지원하지 않는다

### Critical CSS 문제

초기 렌더링 최적화를 위해 현재 화면에서 필요한 CSS만 효율적으로 먼저 로딩하는 방법이 있어야 함. SSR 환경에서는 이미 완성된 HTML이 렌더링을 기다리고 있기 때문에 CSS를 먼저 로드하지 않으면 FOUC가 발생할 수 있음

- StyledComponents : Next, SSR을 이용하는 경우 페이지에서 사용하는 CSS만 head에 삽입됨. collectStyles API를 통해 현재 페이지에서 사용되고 있는 스타일만 별도로 로딩이 가능하게 함
- emotion : extraCritical, 초기 페이지 렌더링에 필요한 Crirical Css를 추출하고 이후 동적인 스타일은 runtime에 생성
- Linaria : mini-css-extract-plugin과 같은 플러그인을 사용하여 critical css를 추출. code splitting을 사용하지 않거나 inital css chunk가 초기 로드에 필요한 CSS가 아닐 경우 mini-css-extract-plugin에 의해 critical css를 판단할 수 없는 경우 linaria에서 제공하는 collect 사용 가능. SSR에서 활용할 수 있을 듯

### Performance

런타임에서 돌아가는 게 반드시 성능 저하를 초래하는 것은 아니다. 평가가 필요하다.

- **런타임 CSS-in-js의 리렌더링** : 컴포넌트에서 runtime에 스타일을 수정한다면 그때마다 CSS를 Parsing하는 시간이 추가로 들어가고, 이 시간만큼 렌더링이 블락된다. prop을 받는 스타일 컴포넌트를 만들어 적용했을 때, 스타일 컴포넌트에 넣는 prop이 바뀌면 스타일을 파싱하고 다시 인젝션을 해야함
- **CSS 수정과 Critical-Rendering-path** : 브라우저는 DOM및 CSSOM 트리를 결합하여 렌더링 트리를 형성하고 레이아웃을 계산한 뒤 렌더링하게 됨. 이때 DOM트리는 수정하지 않고 CSSOM 트리를 수정하는 방식을 선택해 렌더링 트리를 형성하고 레이아웃을 계산한 뒤 렌더링하는 방법이 있음.
  - **DOM injection** : DOM에 Style Tag 추가 후 appendChild를 이용해 Style Node를 추가하는 방식. textContent, innerHTML을 추가하여 스타일 시트 업데이트 => 이럴 경우 DOM트리부터 다시 그림. emotion과 styled-component의 development
  - **CSSStyleSheet.insertRule을 사용해 CSSOM에 직접 삽입** : Styled 태그에서는 빈 내용이 보이고, DevTools에 직접 선택하여 rule을 확인해야만 결과를 볼 수 있음. emotion과 styled-component의 production
- 병목을 일으키는 부분에 부분적으로 zero-runtime을 도입하여 개선할 수도 있음
  - css 선택자가 아니라 element에 `data-` 속성을 주고 그걸 토대로 CSS를 작성하면 제로 런타임이 적용되어 라이브러리가 CSS를 파싱하는 시간을 줄일 수 있음. 렌더링과 상관이 없어짐. 값이 변경된다고 스타일이 죄다 리렌더링이 되지 않고 즉시 바뀜.
  ```js
  import { useMemo, useState } from 'react';
  import styled from '@emotion/styled';

  const darkTheme = {
    name: 'dark',
    buttonBg: 'black',
    buttonColor: 'white'
  };

  const lightTheme = {
    name: 'light',
    buttonBg: 'white',
    buttonColor: 'black'
  };

  const makeCssTheme = (jsTheme, namespace) =>
    Object.entries(jsTheme).reduce(
      (cssTheme, [key, value]) => ({
        ...cssTheme,
        [`--${namespace}-${key}`]: value
      }),
      {}
    );

  export default function App() {
    const [theme, setTheme] = useState(darkTheme);
    const cssTheme = useMemo(() => makeCssTheme(theme, 'xx'), [theme]);

    function toggleTheme() {
      setTheme(theme.name === 'dark' ? lightTheme : darkTheme);
    }

    return (
      <div className="App" style={cssTheme}> // CSS 변수를 여기다 선언해놓으면
        <Button onClick={toggleTheme}>Click to toggle theme</Button>
      </div>
    );
  }
  // styled component 안에서 사용할 수 있는게 신기하다
  // 확인할 수 있듯이 prop을 사용하지 않는 스타일 컴포넌트이기 때문에 변수를 바꾸어도 리렌더링을 유발하지 않는다. 단순히 CSS 변수만 바꾸는 것이기 때문에
  const Button = styled.button`
    background: var(--xx-buttonBg);
    color: var(--xx-buttonColor);
    padding: 10px;
    border: none;
  `;
  ```
  - 혹은 `--data` 속성을 붙여 사용하고 var을 이용한 토글이 가능하다.
  ```js
      const composedStyle = {
        '--btn-color': theme.derivedColors.button[color],
        ...(style ?? {})
      }

    // 역시 인자를 받지 않는다.
    const ButtonView = styled.button` 
      &[data-color="primary"] {
        color: ${({theme }) => theme.derivedColors.text.primary};
      }

      &[data-emphasis="fill"] {
        background-color: var(--btn-color);
      }
    `

    <ButtonView
      type="button"
      style={composedStyle} // 이렇게 인젝션
      data-color={color}
      data-emphasis={emphasis}
      {...props}
    />
  ```
- 당연하겠으나 동적이지 않고 static as possible한 스타일을 유지하는 것이 성능에 도움이 된다.
- 개발자 도구에 Bottom-Up 탭 확인하고 Style 파싱이 실행에 오래 걸리는지 찾아보는 것도 좋을 듯 하다. 성능 체크를 해봐야 한다.
- 아예 atomic CSS를 사용해 커스텀의 영역을 줄여버리는 것도 CSS 최적화 작업이라고 볼 수 있다. [이런 사례](https://engineering.fb.com/2020/05/08/web/facebook-redesign/)

## 무엇을 제일 많이 쓰는가?

npm-trends보니 styled-components가 1위고 @emotion/react가 2위 인데 둘이 2배 정도 차이가 난다. 2021년을 기준으로 이모션이 많이 올라오긴 했다.

## Runtime css-in-js 라이브러리들 탐구

- 어떻게 컴파일되며 돌아가는가
- 어떻게 JS 코드를 파싱하고 CSS를 붙이는가

### Styled Components

#### Tagged Template Literals

```js
const name = 'John'
const location = 'seoul'

const tag = (strs, firstExpr, secondExpr) =>
  console.log(strs, firstExpr, secondExpr)

tag`나는 ${location}에 살고있는 ${name}이야` // [나는 ", "에 있는 ", "이야"], "seoul", "john"
```

- es6 문법, 함수 호출 문법이다
- 첫번째 인자로 문자열 부분만 들어간 배열이 전달되고, 나머지 인자들에는 표현식이 순서대로 전달됨
- 문자열 부분과 표현식 부분을 효과적으로 분리해 함수 인자로 넣어줄 수 있다.

#### styled 함수의 멘탈 모델

```jsx
// myStyled.js
import React, { useRef, useEffect } from 'react'
import domElements from './domElements'

// 3단 고차함수, tag나 컴포넌트 => style => 프롭
const myStyled = TargetComponent => (strs, ...exprs) => props => {
  const elementRef = useRef(null)

  useEffect(() => {
    // prop을 바탕으로 완성된 스타일 문자열 만들기
    const style = exprs.reduce((result, expr, index) => {
      const isFunc = typeof expr === 'function'
      const value = isFunc ? expr(props) : expr
      return result + value + strs[index + 1]
    }, strs[0])

    // 스타일 생성 - 실제로는 이렇게 작동 안함
    elementRef.current.setAttribute('style', style)
  })

  return <TargetComponent {...props} ref={elementRef} />
}

// 모든 HTML 태그를 사용할 수 있게 해주는 로직
domElements.forEach(domElement => {
  myStyled[domElement] = myStyled(domElement)
})

export default myStyled

// index.js
const Button = myStyled.button`
  color: ${props => (props.color ? props.color : 'red')};
  border: 2px solid red;
  border-radius: 3px;
`

ReactDOM.render(<Button color="blue">Click me</Button>, rootElement)
```

#### 내부 동작

- Styled-components는 자신을 통해 생성된 모든 컴포넌트의 개수를 계산하는 내부 카운터 변수를 만듬. 이후 새로운 컴포넌트를 만들게 되면 내부 식별자 componentId가 생성되고(해쉬값) 내부 카운터의 값이 1이 올라감
- 식별자가 생성되면, Styled-components는 해당 컴포넌트가 앱에서 처음 생성된 컴포넌트이면서 element가 아직 DOM에 추가되지 않았을 경우 **`<head>`에 `<style>`을 삽입하고 componentId가 포함된 주석을 사용할 `<style>` element에 추가**
```js
<style data-styled>/* sc-component-id: sc-bdVaJa */</style>
```
- 컴포넌트 생성을 감지하면, 해당 컴포넌트의 componentId로 미리 생성해놓은 componentId를 할당하고 target으로 생성할 element tag name 할당
  - 그냥 스타일 컴포넌트를 만들어 놓는건 만으로는 오버헤드가 없다. `<style>` 태그만 생성될 뿐
  ```js
    StyledComponent.componentId = componentId;
    StyledComponent.target = TargetComponent;
  ```
- Tagged Template Literals내 스타일 문자열을 파싱하고 componetId와 더해 고유한 className 생성. 컴포넌트 ID와 스타일 문자열을 더해 또 해싱한다
  - 스타일이 변하면 이 클래스 해시값도 변한다는 것을 알 수 있다.
  - 이 클래스 값은 styled 컴포넌트 내부의 상태값에 저장됨
  - prop 값에 대한 평가도 끝나야함
- CSS 전처리기를 사용하여 만들어놓은 클래스 이름을 바탕으로 CSS를 만든다. 이때 뭐 여러 선택자들의 경우도 다 잘 처리됨
- CSS를 아까 만들어 두었던 `<Style>` 태그에 삽입한다. 이때 당연히 아까 만들어 두었던 클래스 이름을 지정
- 여기까지 끝나고 나면 인자로 넘긴 컴포넌트를 마운트한다.
  - 이 앞 전처리까지를 클래스 컴넌으로 따지면 componentWillMount에서 처리하는 것임
  - 실제 해당 컴포넌트가 렌더링하는 클래스네임은 다음과 같이 3개로 이루어진다.
    - 개발자가 임의로 설정한 클래스이름
    - 아까 해시된 componentId로 만든 클래스 이름
    - componentId
- 계속 나오지만 프롭이 바뀔때마다 이 과정이 반복된다. 
- CSS가 다시 파싱될때 이전에 만들었던 CSS 선택자는 head에 유지된다. [걔네를 그대로 두는게 없애는것 보다 오버헤드가 덜하기 때문이란다](https://github.com/styled-components/styled-components/issues/1431#issuecomment-358097912)
- 아까 이야기한대로 production은 CSSOM을 건들기 때문에 더 빠르다.

#### theming에 대해

https://medium.com/@BogdanSoare/why-its-better-to-use-css-variables-instead-of-js-variables-for-theming-91a3b6f98166

- CSS variable로 theming하면 추가적인 provider을 쓰지 않아도 되고
- devtool에서 css variable을 수정할 수 있으므로 수정한 사항을 보기에도 유용한 편이며
- **미디어 쿼리로 쉽게 바꿀수도 있게 된다**
- styled-component와 @emotion/css의 `InjectGlobal`로 이를 실현할 수 있다. 아니면 SPA면 걍 CSS 만들어서 index.html에 때려넣어도..? => CSS는 Rendering Block 요소라서 JS보다 더 빨리 렌더링될 수 있으니까
- theme-provider로 넣어주는 theme은 결국 styled가 만든 컴포넌트에 props에 바인딩되어 함께 들어가는(`props.theme.color` 이런식으로 접근함), 연산되야 하는 **동적 값**이기 때문에 이러한 동적 포인트를 css 변수로 바꿔준다면 성능 향상을 기대해볼 수 있을 듯 하다.

### Emotion

- You shouldn’t have to sacrifice runtime performance for good developer experience when writing CSS. emotion minimizes the runtime cost of css-in-js dramatically by parsing your styles with babel and PostCSS 라고 한다.

#### inline 모드 작동 방식

일단 사용하면서 더 상세한 부분은 체크해보도록 하자

- Styled 컴포넌트 코드는 바벨에 의해 다음과 같은 코드로 바뀐다. 바벨을 안쓰는 경우는..? 타입스크립트를 쓰는 경우는..? 템플릿 리터럴 문법을 이런식으로 컴파일한다는건가
```js
const H1 = styled(
  'h1',
  ['css-H1-duiy4a'], // generated class names
  [props => props.color], // dynamic values
  function createEmotionStyledRules (x0) {
    return [`.css-H1-duiy4a { font-size:48px; color:${x0} }`]
  }
)
```
- 저 스타일 컴포넌트가 렌더링이 될 때마다
  - CSS block의 동적 값들을 순회하면서 현재 상태에 맞는 prop 값으로 바뀐다.
  - `createEmotionStyledRules` 함수는 앞에서 최신화한 props value를 인자로 삼는 함수로, 최신화된 props 값을 이용하여 CSS Block의 문자열을 완성시킨다.
  - 완성된 CSS block을 StyleSheet으로 집어넣는다.
  - 컴포넌트의 className으로 생성된 클래스 이름이 들어간다.

### 이 둘의 차이점

이번에 종강시계에서 emotion을 한번 써봐도 좋을 것 같다

- 이모션이 번들 크기가 더 작다 -> 기능이 비슷한데 번들 크기 더 작으면 이모션이 더 좋을듯? 그리고 번들이 `lodash.` 처럼 조각조각 나뉘어 있어 필요한 부분만 설치할 수도 있다.
- Styled Component 보다 마운트, 렌더 시간이 [벤치마크에서 앞선다](https://medium.com/@tkh44/emotion-ad1c45c6d28b).
- emotion의 `CSS props` : 인라인으로 CSS를 작성할 수 있게 하는데 이 CSS에 미디어 쿼리, nested Selector을 이용하는 방법으로 인라인 스타일을 확장한다. tagged template literal 뿐 아니라 object도 사용할 수 있다.
  - 원래 CSS로 만든 디자인 시스템이나 컴포넌트가 있을 때 이걸 쓰면 마이그레이션하기 너무 좋을 것 같다.
  - 바벨 프리셋과 함께 사용해야 한다.
- SSR에서 Styled-Component보다 이점이 있다고 하는데 아직 Next를 많이 다뤄보지 않아서 잘 모르겠다.

## CSS-in-CSS

또 일각에서는 Runtime CSS보다 SASS랑 CSS-module을 가지고 스타일링을 하는게 낫다는 [의견도 있다.](https://blueshw.github.io/2020/09/14/why-css-in-css/)

- Runtime CSS-in-js의 단점
  - 번들이 커진다. SSR을 하더라도 추출된 critical CSS까지 포함하니 더커진다
  - 동적 값을 연산해야하는 시간이 필요하기 때문에 인터랙션이 늦어질 수 있다.
- 물론 성능이 매우 중요하고 CSS-in-js가 병목이라는게 밝혀졌을 경우 최적화는 반드시 필요하다.
- 그렇지 않은 경우, 개인적으로 런타임 CSS-in-JS 개발 편의성과 성능은 어느정도 타협이 가능한 지점이라고 생각함. 런타임 CSS-in-JS가 제공하는 동적 스타일링 방식 등의 편의기능은 개발 과정에서 확실히 도움이 되기 때문이다.