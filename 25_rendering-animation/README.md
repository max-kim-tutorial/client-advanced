# React 마운트, 언마운트간 애니메이션

지금까지 왠만한 애니메이션은 CSS로 구현했는데  

아예 화면에 없는 DOM 노드, 컴포넌트의 등장과 소멸 사이에 애니메이션을 걸으려니 꽤나 복잡하고 코드도 지저분해져서 대안을 알아본다. 실무 할때는 이런거를 거의 

애니메이션을 구현할 때의 기준을 좀 잡으면 좋을 듯 하다  

## 기본 CSS로 mount/unmount시 애니메이션 주기

기본적으로 CSS keyframe animation을 이용하는 방법  
컴포넌트가 마운트되는 시점과 언마운트 되는 시점에 한 번 적용되는 CSS 애니메이션을 적용해야 한다.  

확실히 이런 접근에는 생명주기 접근 방식이 더 유리하긴 한 것 같다. 시계열로 컴포넌트를 관리할 수 있었다면 직관적으로 로직을 넣을 곳을 확실히 찾았을 수 있었을 것 같다.

### mount

컴포넌트 렌더링 시에 keyframe으로 애니메이션이 적용될 수 있게 처리해준다.

```jsx
function SomeComponent() {
  const [isShow, setIsShow] = useState(false);

  return (
    <>
    {
      isShow && 
      <div css={divStyle}>
        블라블라
      </div>
    }
    </>
   
  )
}

const fadeIn = keyframes`
  from {
    opacity: 0;
  }

  to {
    opacity: 1;
  }
`

const divStyle = css`
  animation: ${fadeIn} 1s ease 1;
`
```

### unmount

사실 mount보다 더 문제가 되는건 unmount다. state로 컴포넌트를 토글할 경우 변수가 바귀지마자 재랜더링이 되서 컴포넌트가 사라진다던가 하는데 이렇게 컴포넌트를 토글할 경우 애니메이션을 적용할 틈도 없이 컴포넌트가 사라지기 때문이다.

그래도 unmount시 작동시키는 방법은 있지만 컴포넌트의 state이 더 많아질 수 있고, 로직도 지저분해진다. 

- unmount시 애니메이션이 실행되는 시간을 벌기 위해 setTimeOut으로 애니메이션의 실행 시간만큼 State 변화를 늦춘다던지 하는 로직을 써야할 수도 있다. 이런 상황에서 추가적으로 state를 더 선언해야할 필요도 있을 수 있다. 이렇게 시간을 명시적으로 선언하면 CSS의 시간과 state를 늦추는 시간을 맞춰줘야 하므로 유지보수 부담이 상승하고 응집도도 떨어진다.
- [CSS의 animation 프로퍼티가 이미 element의 스타일에 존재한다면, 이를 수정하는 것으로 새로운 애니메이션을 fire할 수 없다.](https://velog.io/@dosilv/React-mountunmount-%EC%8B%9C-%EC%95%A0%EB%8B%88%EB%A9%94%EC%9D%B4%EC%85%98-%EC%A3%BC%EA%B8%B0) 이미 애니메이션을 적용한 DOM에 새로운 애니메이션을 적용하고 싶으면 animation 프로퍼티를 삭제한 후 다시 새로운 애니메이션 프로퍼티를 넣어줘야 한다.

이런 식으로 불편한 로직을 작성할 바에는 라이브러리를 사용하는게 낫다....


## 라이브러리 사용하기

### React Transition Group

reactcommunity에 있는, 렌더링간 애니메이션 구현을 위한 low level api를 제공하는 라이브러리이다. css 프로퍼티를 기술하는 방식으로(키프레임과 비슷하게) 애니메이션을 구현할 수 있다.

Hooks 기반 개발에서는 Transition 컴포넌트를 사용해 애니메이션을 선언적으로 지정할 수 있다. 근데 그렇게 직관적인 편은 또 아닌듯? 스타일을 미리 작성해줘야 하고 지정도 해줘야하기 때문에 이런 점은 키프레임과 크게 다른 점이 없다. 

본질적으로 컴포넌트의 렌더링 변화 양상을 변수를 통해(state) 얻을 수 있게 하는 도구다. 순수 CSS가 아니라 리액트의 상태를 이용해서 키프레임을 제어하는 정도의 편리함 정도를 얻을 수 있다. 여전히 키프레임 비스무레한 스타일 객체는 사용해야 한다.

[속을 들여다보니](https://github.com/reactjs/react-transition-group/blob/master/src/Transition.js) 클래스형 컴포넌트의 생명주기를 바탕으로 렌더링의 특정 status를 내놓는 방식으로 구현되어 있다.

```js
import { Transition } from 'react-transition-group';

const duration = 300;

const defaultStyle = {
  transition: `opacity ${duration}ms ease-in-out`,
  opacity: 0,
}

const transitionStyles = {
  entering: { opacity: 1 },
  entered:  { opacity: 1 },
  exiting:  { opacity: 0 },
  exited:  { opacity: 0 },
};

const Fade = ({ in: inProp }) => (
  // in은 컴포넌트를 토글하는 prop
  <Transition in={inProp} timeout={duration}>
    // Transition의 자식 컴포넌트에서 컴포넌트 렌더링 양상 변화에 따른 State값을
    // 받아서, 미리 정의한 스타일을 통해 애니메이션을 준다
    {state => (
      <div style={{
        ...defaultStyle,
        ...transitionStyles[state]
      }}>
        I'm a fade Transition!
      </div>
    )}
  </Transition>
);
```
CSSTransition을 이용하면 CSS의 이름을 기반으로 상황에 따라 다른 클래스네임을 적용하는 방식으로 애니메이션을 적용할 수도 있다.

### react-spring

```jsx

function MyComponent() {
  const [isVisible, setIsVisible] = useState(false); 

  const transition = useTransition(isVisible, {
    from: {opacity: 0},
    enter: {opacity: 1},
    leave: {opcity: 0},
  })

  return (
    <div>
      {transition((style, item) => item ? <animated.div style={style} /> : null)}
    </div>
  )
}


```

- 훅을 사용해 선언적이며 위의 예제보다 훨씬 간편해졌다
- keyframe스럽게 일종의 시계열을 제시하여 제어하는건 똑같다. 응집도 때문이라도 왠만하면 CSS가 return문 이상으로 안 넘어왔으면 좋겠다는 생각도 든다.

### framer-motion

```jsx
import { motion, AnimatePresence } from "framer-motion"

function MyComponent() {
  const [isVisible, setIsVisible] = useState(false); 

  return (
    <AnimatePresence>
      {isVisible && (
        <motion.div
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          exit={{ opacity: 0 }}
        />
      )}
    </AnimatePresence>
  )
}
```

- React-Spring보다 더 선언적이다. 훅이 리턴하는 함수가 아니라 부모 컴포넌트로 감싸는 방법을 사용했다.
- return 문 이상으로 Style 로직을 작성 안해줘도 되서 편할듯 하다
- [번들 포비아](https://bundlephobia.com/package/framer-motion@5.3.3)로 확인해보면 React-Spring보다 크기도 더 작고 side-effect free다.
- 독스가 너무 멋있ㄷ...
- https://yrnana.dev/post/2021-02-13-framer-motion-react-motion-gesture