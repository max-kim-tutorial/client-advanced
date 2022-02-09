# React 프로젝트 구성하기

구직 사전과제 대비 정리

## 프로젝트 전체 디렉토리 구성

```shell
src
|- @shared
  |- atoms(store) # 전역에서 사용하는 atom, store
  |- utils # 전역에서 쓰이는 헬퍼함수(format 등)
  |- hooks # 전역에서 쓰이는 훅
  |- images # 이미지
  |- services # api나 storage관련한 부분의 헬퍼
  |- styles
    |- common # 전역에서 공통적으로 필요하고 적용하는 스타일
    |- token
      |- color.ts
      |- size.ts
      |- font.ts
      |- media.ts
    |- global.ts # 전역 스타일
  |- components
    |- fundamentals # 가장 작은 UI
    |- commons # fundamental보다는 크면서, 여러 도메인에서 공통적으로 사용하는 컴포넌트
    |- boundaries # suspense, errorboundary 사용시
    |- errors
|- @pages # 페이지 단위의 개발이 아닐 경우 Layout
  |- index.tsx # reexport 적당히, 혹은 라우터 구성해서 export
  |- page1.tsx
  |- page2.tsx 
|- feature1
  |- components # 디렉토리로, 여기는 딱히 reexport필요 없음
    |- index.ts # 기능 단위에서 사용하는 컴포넌트들의 최종 종단 결과물(은닉)
    |- feature1Component1 # 폴더
      |- index.tsx # 컴포넌트
      |- feature1Component1hook
      |- feature1Component1Children
        |- index.tsx
    |- feature1Component2
      |- index.tsx
  |- atoms(store)
  |- hooks # 기능 단위에서 공통적으로 사용하는 훅(없을수도 있음)
  |- test # 필요한 부분에 테스트를 붙이기 위해 기능 단위 폴더에 test 폴더 넣기
  |- remote
    |- queries # 쿼리
    |- adapters # 쿼리로 얻은 데이터를 수정하는 순수함수
|- feature2
app.tsx # 앱에서 전역으로 제공하는 무언가들을 붙임, 아니면 여기 라우터 붙여도됨
index.ts # react를 붙이는 부분 + 필요한 서드파티 붙이는 부분
```
## 전역 상태관리

- Redux Toolkit Slice : Redux를 쓰고있는 팀이라면 이렇게 해도 댈듯? 근데 디렉토리 구조랑 좀 안맞는 부분이 있음 store을 전역에다가 셋업해줘야 되는데... 물론 각 피처 디렉토리에서 slice 구성하고 configure store에 붙이면 되겠지만 디렉토리 로직 상으로는 이러한 동작은 예외적이라서 맘에는 안든다(그렇지만 막 너무 복잡한 유스케이스는 또 아니다).
- Zustand : 개인적인 바람으로는 Recoil에서 액션만 좀 지정할 수 있으면 좋을거같은데 Redux랑 Recoil의 딱 중간 정도 인터페이스를 제공하는 도구다. 근데 공부가 부족해서 잘 쓸 수 있을지는 자신이 없음
  - selector memoization이 필요하다 Redux의 useSelector처럼 shallow 옵션을 제공하긴 함
- Recoil : 잘 쓰고 있긴 했지만 setState를 바로 노출한다는 것만은 맘에 안들었다..

개인적으로 recoil과 redux 사이 어딘가 수준이 좋을거같다

## 디자인 시스템

이게 젤어렵네

너무 프롭 단의 스타일링에 집착하지 않는게 좋을까? 그냥 text같은거는 fundamental하지말고 바로 스타일링 하는게 좋을까? 귀찮거나 덜 선언적이지 않을까?
css props? styled? 스타일은 어디까지 스타일시트와 분리되어야 하는가????????

- html에 1rem 10px로 해볼까
- CSS variable 주입
- spacer나 divider은 이제 잘 모르겠다..한다면 가로세로 모두 지원하는게 좋을듯하고
- CSS Variable? prop과는 어떻게 동기화? : CSS variable로 통일하는거 에바임? 문자열로 넣어두고 `--color-${props.color}`이런식이면?
  - 어짜피 인자를 받아야한다면 이런 방식조차 크게 의미가 없다. 그냥 하던대로 할까???
- 전역 스타일? 
- theming? : Context API? 또 어떤 스타일은 theming과 상관없어야한다면?(???) 이럴 경우에는 그냥 프롭단 스타일링에 집착하지 않는게 좋을수도 있을듯? theme을 잘 쓰려면 내생각에 styled사용해야함
- CSS Normalize : 이건 걍 스리슬쩍 하자 global style로 주기..
- 전체에서 재적용해야하는 스타일? 

## 컴포넌트 짜는 법

- 추상화 레벨이 너무 깊어지면 안된다. 신경쓰기..
- 적절히 의존성을 컴포넌트의 prop이나 hooks의 param으로 노출해준다
- 한가지 역할에만 집중하면 나머지는 의존성 주입 해주는 방식으로도 사용 가능하게 됨
- children의 경우 한가지만 들어간다치면 자식으로, 여러개가 들어간다치면 prop으로 넣는다. named slot이 필요할때도 prop으로 넣어야함

### 커스텀 훅 사용방안

- 컴포넌트 한개가 한가지 역할과 관심사만 있다면 상관없음 : 가장 이상적
- 여러개가 있다면 커스텀 훅으로 분리하고 컴포넌트 디렉토리 내부에 위치시키기

### useEffect

- useEffect같은 경우 이름을 꼭 부여해주는 방식을 생각해보기(useCallback 래핑, useEffect 내부의 함수)

### 최적화, 메모이제이션 사용방안

useMemo와 useCallback은 발적화의 가능성이 높기 때문에 의미없는 useMemo나 useCallback은 없어야 한다. 또한, 매번 모든 순간 렌더링을 최적화할 필요도 없다.

- **필요없는 메모이제이션을 사용하지 않는다**
  - 특정 컴포넌트의 메모이제이션이 특정 프롭들 중 아주 잘 바뀌는 프롭과 의존하고 있다면 의미가 없다. 이거는 그냥 매 랜더링마다 함수가 만들어지는 로직과 다를게 없기 때문이다.
  - 특정 컴포넌트가 단 한번만 렌더링되는 것이 보장되는 상황이라면 걔네도 굳이 메모이제이션이 필요 없을 것이다.
- **레퍼런스 비교가 필요할때 메모이제이션을 사용한다** : (피하는게 좋겠지만) 개발하다보면 useEffect 등에 직접 state값을 넣는 것보다, state를 의존성으로 가지는 메모이제이션 로직을 useEffect에 넣어야 하는 상황도 충분히 생길 수 있다. 이때는 메모이제이션을 통해 useEffect가 제대로 동작하도록 함수를 useCallback등으로 감싸야 한다.
- **렌더링을 줄여주는 방식으로 메모이제이션을 사용한다** : 유저 인터랙션(거듭된 토글이나 스크롤, 인터벌) 등으로 많은 리랜더링이 불가피할 경우 메모이제이션으로 최적화한다.
  - 근데 사실 이것도 발적화일...수도 있다. 리랜더링 자체를 그렇게 너무 통제하려고 하지 말라는 개발자들도 좀 많은듯 함. 확실히 성능에 영향이 있는 경우에만 인스펙션해서 적용해야 할지도..? 이건 근데 실무를 더 경험해봐야 할 것 같기도 하다.
- **부모 컴포넌트 랜더링이 너무 자주 일어나서 자식 컴포넌트의 랜더링도 덩달아 많이 일어나는 경우에는 React.memo 래퍼를 사용한다.**
  - 애초에 설계가 잘못된 걸수도 있다는 생각을 좀 해볼 수 있지 않을까 싶긴 하다. 불가피한 경우인지 타진한다!
  - 애초에 리렌더링을 많이 피해야한다 : 컴포넌트 내부 state은 정말 필요한 최소만 선언하고, 데이터 패칭 로직은 그것을 소비하는 컴포넌트와 최대한 가깝게 위치시키는게 좋다. 상태 끌어올리기도 신중하게 하자.

## 비즈니스 로직 분리

- 로컬하게 쓰이는 커스텀 훅은 컴포넌트에 같이 정리하고, 같이 쓰이는 훅은 훅 디렉토리 만들어서 정리, 프로젝트 전체에서 쓰이는 훅은 @shared로 분리
- 데이터 어댑팅 로직을 분리, 이때 어떤 데이터를 변형하는 로직인지 잘 알아볼 수 있어야함
  - 데이터 관련 로직을 Remote로 분리하고, 여기 안에 query, adapter 이렇게 분리하는건 어떨까? 기능안에 components, store(리코일을 사용한다면 atoms), hooks, remote(query, adapter), test(엄선된 통합 테스트 케이스 위주), 
  - 쿼리 데이터 기준으로 data2Adapter, data1Adapter
  - 한 리모트 순수함수에서 2가지 이상의 데이터를 사용해 새로운 데이터를 만들어내는 로직의 경우는? compoundAdapter? data1data2Adapter?
  - 이런 순수함수들을 이런 식으로 분리하는 이유 : 유사시에 유닛테스트 작성 가능, 데이터에 어떤 변경이 필요한지 한눈에 알아볼 수 있음, 컴포넌트에서 로직을 분리하여 컴포넌트 복잡성 하락(이정도는 괜찮은 추출 비용이라고 생각), 컴포넌트 내부에 함수를 두거나, JSX내부에 함수를 선언해서 놓거나 하지 않는 방식으로 렌더링에 영향을 미치지 않음
  - 그리고 사실 얘네들은 utils이라고 할수 없음 utils보다는 좀더 중요함... 얘네들은 진짜 utils보다 더 코어로직에 가까움
