# RN 주제연구

## [어떻게 JS스레드와 네이티브가 같이 돌아가는가? 당최 무슨일이..??](https://levelup.gitconnected.com/wait-what-happens-when-my-react-native-application-starts-an-in-depth-look-inside-react-native-5f306ef3250f)

- 되게 마법같다 이벤트를 받아서 JS에서 처리하는것도 신기하고.. 한번만 쓰면 두 플랫폼에서 같이 돌아가는것도 신기하고...

### infrastructure

![구조](https://miro.medium.com/max/2846/1*ZBOTTHTIOTZT0h62mvFxLQ.png)

- RN은 런타임간에서 빌드한 구조에 의존해서 돌아간다. 위의 구현체들은 RN앱을 킬 때마다 매번 만들어짐. 빌드해서 배포하기 전까지는,,,(???)

### 앱을 시작할 때

- 앱 아이콘을 눌러서 RN앱을 실행하면 어플리케이션의 코드, 그리고 독립적인 **메인 스레드**를 돌려야한다. 스레드는 디바이스의 OS에서 할당함
- 간단하게는 프레임워크의 코드와(맨날 수정할 필요가 없음), 커스텀 코드로 나눠진다(니 앱을 설명하는 코드) 둘 다 자바스크립트와 네이티브에 분배된다
- 자바스크립트 UI코드는 종국에는 네이티브로 바뀌어 화면에 나타난다.
- RN은 자바스크립트 컴포넌트와 native view를 연결하는 일, native view를 쌓아두고 보여주는 일을 한다.

#### RootView

RootView는 네이티브 뷰를 쌓아놓는 컨테이너로, 커다란 트리로 조직된다. 디바이스에 나타나는 모든 UI는 RootView에 store된다. 모든것은 RootView가 만들어짐으로 시작함

#### Bridge Interface

- C++ 구현체다
- Bridge Interface는 API처럼 작동하는데, 네이티브와 JS가 소통하도록 한다. 
- 이 API의 종단점 중 하나는 Native Module인데 JS 개발환경에서 접근 가능하다. 이것 말고는 JS 개발 환경에서 접근이 불가능하다.
- Native Module은 여러가지로 이루어져 있다
  - Custom Module : 개발자가 만든 모듈
  - Core Module
  - UIManagerModule : JS UI를 저장하고 native view로 사상하는 모듈. JS 컴포넌트가 생성되고 업뎃되고 삭제될때마다 이 모듈이 네이티브 UI에도 똑같이 수행되게끔 한다. 
- 모든 Native Module은 관련해서 인스턴스가 만들어지면 자바스크립트에서도 접근할 수 있게 된다. 네이티브 모듈에서도 JS를 그대로 호출할 수 있게 된다. 
- 결과적으로는 브릿지를 통해 JS Thread와 NativeModuleThread라는 두개의 쓰레드가 생성된다.
- 대충
  - 네이티브 모듈이 메인 스레드에 생성됨(앱에서 할당되는 스레드)
  - 스레드가 3개인거임?
  - 아직 JS가 처리되지는 않음

### JS 번들 load

- JS는 그대로 돌아갈 수 없다. byteCode로 컴파일되어야함. 이거는 Javascript virtual machine이 담당한다(????)
- 자바스크립트 엔진을 활용한다. v8, hermes, JavaScriptCore(이게 기본)...
- JavaScriptCore은 안드로이드에 내장되어있지 않기 때문에 안드로이드에는 이것도 함게 들어간다. 안드로이드 앱이 ios앱보다 조금 무거움
- 

## native UI를 어떻게 RN에서 사용할 수 있는가?


## Virtual List들