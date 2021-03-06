# How React Native works?

리액트 네이티브는 어떻게 JS로 Native UI를 띄우고 상태를 변경시키나

## RN 자체의 구조

크게는 Native부분과 JS부분으로 나누어진다.

1. Native Code/Modules : 말그대로 objective-c나 java로 이루어진 native 모듈. React Native CLI로 앱을 만들면 내부에 존재하는 IOS, Android 앱 디렉토리
2. Javascript VM : React로 작성한 코드를 Metro로 번들한 결과물을 native devices에서 실행시키는 부분(왜 VM이라고 하는건지 잘 모르겠음) - 이거 native device에서 돌리는게 아닌가?
3. bridge: JS스레드의 자바스크립트 번들과 Native 스레드 네이티브 앱의 소통을 담당

## Metro bundler + 빌드(main.bundle.js)

```js
const Metro = require('metro');

Metro.loadConfig()
  .then(config => {
    Metro.runBuild(config, {
      entry: ~~,
      out: ~~
    })
  })
```

- 일단 얘 자체는 기본적으로 자바스크립트를 번들해서 single js file로 만들어주는 일을 한다. 번들러니깐.. 진입점과 옵션을 가지고 있음. 번들 결과물이 웹팩이랑 판이하게 다른 점이 있다거나 하는 건 없는듯?!
- 핫 리로딩 : 번들파일을 서빙하는 서버(Metro.runServer) => 이게 좀 다른데, 서버를 띄워서 웹 주소창으로 접근해보면, 자바스크립트 파일 번들 그 자체를 전달하고 있음을 확인할 수 있음(HTMLWebpackPlugin이나 loader같은 아무런 옵션이 없는 웹팩같음. 뭐 플러그인이나 이것저것을 붙이면 웹 번들러로 사용할 수 있을지도?). 즉 번들링은 별다를게 없지만, 번들해서 제공하는 방법이 웹 번들링하고는 다른 것(아마 native devices에 맞게 제공될거라고 생각할 수 있겠슴)
- 바벨 옵션을 설정해놓으면 아마 metro로 번들되기 전에 적용이 될듯. 타입스크립트도!
- 웹 리액트처럼 브라우저 디렉토리에 폴더가 만들어지는 방식으로 빌드되지는 않음. 빌드된 결과물은 device로 가서 실행됨. JS코드가 Native언어로 translate되는 것이 아니고 device가 직접 JS를 실행시킴
- 이 번들된 코드는 이벤트 루프를 거칠때마다 메인 스레드로 변경 사항에 관한 정보를 전달한다. 전달되는 정보는 어떤 뷰를 표시할 것인지에 대한 정보. 이때 React의 diffing

## native devices에서의 JS 실행

- ios에서는 기기 내부에서 safari를 구동하는 JS 엔진인 JavaScriptCore을 사용.
- 안드로이드에서는 RN이 JavaScriptCore을 어플리케이션과 같이 번들해서 제공(그래서 번들이 큼). Hermes라는 오픈소스 자바스크립트 엔진을 사용할 수도 있음
- RN 내부의 native module을 통해 device에서 앱을 빌드한 후, 빌드한 앱은 metro 서버의 핫 리로딩을 통해 JS 번들(스레드)와 bridge로 소통하며 상태값을 바꾸거나 UI를 그림

## bridge

![브릿지](https://www.reactnative.guide/assets/images/rn-architecture.png)

![bridge](/_img/bridge.png)

- C++ 구현체라고한다
- 서로 분리되어있는 JS 스레드와 native 스레드의 소통은 bridge라는 것으로 이루어지는데, 다음과 같은 3가지 특징을 가짐
  - asynchronous : 스레드간 비동기적으로 소통한다. 서로가 서로의 스레드를 블락하지 않는다. 단점으로는 UI 반응이 늦을 수도 있음. UI가 업데이트되도 동기적으로 state가 업데이트 되지 않을수도 있음(근데 이건 뭐 감수해야 될듯). 네이티브는 스레드 통신 없이 바로 state를 업데이트하고 view를 갈아치우는게 가능.
  - batched : 소통은 최적화된 방식으로 한다. (JSON의 형태로 소통한다고)
  - serializable : 두 스레드는 같은 데이터를 공유하지 않는다. 대신에 스레드는 직렬화된 메시지를 교환한다.
- Native Touch Event(Native) => business logic(JS) => Change State(Js) => Render New View(Native)
- 브릿지를 이용해 JS 스레드를 평가하여 device의 앱으로 렌더링
- 성능이 좋아지려면 Native Bridge를 최소한으로 건너야함

## 메인 스레드

- 어플리케이션이 실행 되자마다 실행되고, 앱을 로드하고 JS 스레드를 실행시킨다. JS스레드의 이벤트 루프마다 브리지를 통해 보내오는 메시지를 받아 해석한 후 UI를 화면에 표시한다.
- 이 과정에서 shadow thread는 JS 스레드로부터 넘어오는 정보를 활용하여 화면의 레이아웃을 계산한다. 웹에서의 reflow와 비슷. 
- 또한 사용자들이 기기에 내리는 UI이벤트 명령들을 받고 Native Bridge를 경유해 JS스레드로 넘긴다.(press, touch와 같은 이벤트)

## 성능과 관련된 부분

- 아이폰에 경우 60fps인데 1프레임을 표시하기 위해 16.67ms가 필요하다. RN 앱은 그 시간 안에 하나의 프레임을 생성해야 사용자가 봤을때 애니메이션이 끊기지 않는다.
- 만약 JS 스레드의 이벤트 루프에서 16.67안에 작업을 처리하지 못하면 병목이 생기고 프레임이 끊긴다

## 스레드가 2개

- native thread(native devices) : standard native app에서 사용하는 스레드로, 각 모바일 디바이스에서 UI를 그림
- JS thread(RN) : JS를 실행하는 스레드. 이것도 device안에 있음
- 이 두 스레드는 서로 직접 소통하지 않고 서로 블락을 한다거나 하지도 않음

## 프로젝트 내부의 Native 모듈

- android studio나 Xcode에서 RN내부의 android, ios 프로젝트 디렉토리에 들어가 앱을 빌드하면 앱이 잘 빌드됨
- 아마 이런식으로 껍데기만 빌드를 한 후, main thread에서 실행이 되면, 나머지는 Metro로 서빙한 JS 번들이 브릿지를 통해서 상태를 바꾸거나 뷰를 업데이트하거나 알아서 하는듯???
- 네이티브를 잘 몰라서 정확하게 말할수가 없지만 아마 브릿지를 통해 JS 스레드와 소통하도록 네이티브 코드들이 짜여져 있지 않을까..생각해볼 수 있다.

## 그렇다면 Native Module을 직접 건드려야 할 때는 어딜 건드려야 되는거지

- 각각의 native module내부에서 native 언어로 UI를 짜고, RN의 모듈과 연결(link)시켜서 RN에서 쓰면 되는걸까?
- [그런거 같음](https://reactnative.dev/docs/native-modules-ios) => 이거는 추후에 또 과정을 좀 보기

## 실행 과정

1. 앱이 시작하면서 메인 스레드가 시작되고 메인 스레드는 JS 스레드를 실행시키고 자바스크립트 번들을 로드
2. JS 스레드가 실행되면서 React는 diffing해서 가상돔 생성하고, 해당 결과를 Native Bridge를 경유하여 shadow 스레드로 전달(오.. React 요소를 JSON으로 전달하는???느낌)
3. Shadow 스레드는 변경사항 메시지를 통해 화면의 레이아웃을 계산하고 계산이 끝난 레이아웃의 파라미터나 객체를 메인 스레드로 보낸다
5. 사용자가 화면에 입력한 UI 이벤트 정보들이 브리지를 경유하여 JS 스레드로 보내짐
6. UI이벤트 메시지를 활용하여 JS 스레드에서 비즈니스 로직들이 실행되고, React는 다시 가상돔을 생성하여 브리지를 경유하여 쉐도우 스레드로 전달

## 심화) Fabric

향상된 RN의 아키텍처

- bridge의 문제는 UI가 복잡할때, 동기적으로 업데이트가 되지 않고 브리지를 통해서 주고받는 데이터들을 받아야 state를 업데이트하던지 뷰를 렌더링하던지 하기 때문에 state의 변화가 UI에 바로, 직접 반영되지 않을 수 있다는 것이었음
- React-native-fabric은 RN의 C++ 레이어에 직접 shadow tree를 만들수 있으며, 이는 터널링을 줄여서 프로세스를 더 빠르게 만들어 이 문제를 해결한다고 함
- 추후에 또 공부해보기

## reference

- https://www.reactnative.guide/3-react-native-internals/3.1-react-native-internals.html
- https://facebook.github.io/metro/docs/concepts
- https://www.youtube.com/watch?v=VMoRoD4YjcE&t=467s
- https://velog.io/@koreanhole/React-Native%EC%9D%98-%EC%9E%91%EB%8F%99%EC%9B%90%EB%A6%AC