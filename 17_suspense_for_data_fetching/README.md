# Suspense for Data Fetching

데이터를 가져오기 위한 Suspense는 `<Suspense>`를 사용하여 선언적으로 데이터를 비롯한 무엇이든 기다릴 수 있도록 해주는 기능이다. 원래는 이미지나 스크립트등의 동적 임포트와 연계하여서만 사용할 수 있었지만, React 18 부터는 데이터 패칭에도 사용될 수 있을 전망이다.

https://jbee.io/react/error-declarative-handling-1/
https://overreacted.io/algebraic-effects-for-the-rest-of-us/

## Suspense로 가능 한 것

- 데이터 패칭 라이브러리들이 React와 깊게 결합할 수 있도록 해줌
- 의도적으로 설계된 로딩 상태를 조정할 수 있도록 해줌
- 경쟁 상태를 피함 : await을 사용하더라도 비동기 코드는 종종 오류가 발생하기 쉬움. Suspense를 사용하면 데이터를 동기적으로 읽어오는 것처럼 느끼게 해줌

## 기존 데이터 패칭과 Suspense의 관점 차이

어떤 부분에서 비동기를 동기로 바꾼다는 느낌이 나는거임?

## 구현체

https://gist.github.com/sebmarkbage/2c7acb6210266045050632ea611aebee
https://github.com/JustSift/ReSift/issues/16
