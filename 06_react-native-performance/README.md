# React Native 성능

좀 아리송한 부분이 있는 거 같아 독스에 있는거 정리함

## overview

- 성능 향상의 설득력있는 목표 : 60fps을 유지해서 native 앱과 진배없이 보이는 것
- RN의 성능 자체가 좋으면 개발자는 비즈니스 로직에만 집중할 수 있겠지만 아직 거기까지는 아님. 성능이 시원찮을때는 뭔가 좀 처리가 필요할때가 있음.

### frame에 대해 알아야 할 것

- quickly changing static image들이 일정한 속도를 만들어내는게 실제같은 video임 => 울 할아버지 세대들이 영화를 moving pictures라고 했던게 그 이유임. 이 이미지들이 frame임
- 초당 프레임은 실제같고 자연스러운 움직임을 만드는데 중요함 -> ios 디바이스들은 60fps인데, 이거는 ios앱의 UI 시스템이 16.67ms 안에 모든 준비를 끝내고 정적 이미지를 만들어내기 시작해야한다는 뜻과 같음.(동작을 시작하기 전에 준비할 시간?) 이 시간 안에 이미지 만들어내는 것을 시작하지 못한다면, drop frame이 발생해서 되게 unresponsive해짐(뚝뚝끊김)
- 아이폰에 빌드 시켜놓고 흔들면 메뉴가 뜨는데 거기에 Show Perf Monitor라는 메뉴가 있음 그걸 누르면 2개의 다른 frame rate를 확인할 수 있음(UI, JS)

### JS frame rate (javascript thread)

- RN앱의 비즈니스 로직은 대부분 JS스레드에서 돌아간다. 만약 자바스크립트 스레드가 frame에 unresponsive하다면, frame drop이 났다고 이야기할 수 있음.
- 스레드 위에서 일어나는 일, 예를들면 엄청 많은 컴포넌트들의 재랜더링을 유발하는 state 변경의 경우, 대략 100ms가 넘을때까지 그 일이 안끝나버리면 frame drop이 일어남.
- navigator transition에서 이런일이 종종 일어남. route push를 하면 새로운 스크린을 렌더링하기까지 꽤 많은 재랜더링이 일어날 수 있다. transition, animation 등도 JS 스레드 위에서 처리되기 때문에, 요약하면 **렌더링 성능이 UI 성능에 영향을 미친다.**
- touch에 반응하는 것도 지연될 수 있음. thread가 busy해서 이벤트가 일어난 직후 반응하지 못하는 경우

### UI frame rate (main thread)

- NavigatorIOS를 사용하는 것이 Navigator을 사용하는 것보다 성능이 더 좋을 때가 있는데, 이것의 이유는 NavigatorIOS의 animation은 main thread에서 진행되기 때문에 JS 스레드의 영향을 받지 않기 때문임.
- 비슷하게 ScrollView의 렌더링도 main thread에서 일어남. 스크롤 이벤트는 JS 스레드에서 디스패치되서 JS 스레드가 인식하게끔 되지만 JS 스레드가 뭐 어쩌고 저쩌는지는 스크롤이 일어나는데는 전혀 문제가 없다.

## 성능 문제의 일반적인 원인

### 1. development 모드로 run해서

- dev mode로 돌렸을때 성능이 더 구리다. 이거슨 당연함. 에러나 경고 메시지도 띄워야함. 그래서 전반적인 성능은 release build에서 테스트를 해봐야함

### 2. console.log를 써서

- console.log는 JS 스레드에 꽤 큰 병목을 가져올 수 있다. console.log를 삽입하면 redux-logger와 같은 추가적인 디버깅 라이브러리를 불러오는데, 이게 꽤 부하가 크다.
- 바벨 플러그인 같은걸로 콘솔로그를 지운채 빌드하거나 해봐라

### 3. ListView의 초기 렌더링이 너무 느리거나, scroll performance가 거대한 리스트일때 너무 느린 문제

- FlatList나 SectionList 컴포넌트를 써라. 성능적으로 향상이 이루어진 컴포넌트들이기 때문.
- FlatList의 렌더링이 느리다면, getItemLayout prop 적용을 고려해봐라.

### 4. 불필요한 렌더링이 계속 일어나는 경우

- ListView의 rowHasChanged와 같이 리렌더링 여부를 명시해줄 수 있는 프롭이나, shouldComponentUpdate같이 state나 prop이 바뀔때 리렌더링을 어떻게 해줄지 로직을 작성해줄 수 있다면 좋다. pureComponent를 사용한다던지
- 불변 데이터 구조를 쓰면 빠르다 - immutable같은거 말하는건가

### 5. 동시에 여러가지 일이 JS 스레드에서 일어나는 경우

- 네비게이션 트랜지션이 느린 경우가 주로 이런 경우. LayoutAnimation 사용하거나 useNativeDriver옵션 사용 고려. (이때는 non-layout 애니메이션)

### 기타등등

1. 이미지 사이즈를 변화시키는 애니메이션 - non layout 애니메이션으로 전환하기
2. 네비게이션 트랜지션 - JS 스레드를 통해 컨트롤됨(아직 잘 모르겠음)

## reference

- [RN Docs](https://reactnative.dev/docs/performance)
