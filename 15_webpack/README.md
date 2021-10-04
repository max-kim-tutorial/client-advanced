# Webpack을 열심히 한 번 살펴봤다

## 웹팩의 내부 동작

[Webpack Concept](https://webpack.js.org/concepts/)  
[webpack-how-it-generates-the-bundle](https://nearsoft.com/blog/webpack-how-it-generates-the-bundle/)

- 이건 한번 시간을 내서 튜토리얼로 번들러 만들어보면 좋지 않을까 싶다

## output

컴파일된 파일을 디스크에 쓴다

- 진입점은 여러개일 수 있지만 outputPath는 하나다

- assetModuleFileName : 에셋 모듈용 파일 아웃풋 네임(Webpack 5 에셋 모듈)
- charset: 최신이 아닌 브라우저와의 호환성을 위해 webpack은 여전히 charset을 기본적으로 추가한다.
- chunkFileName : 초기 번들이 아닌 청크 파일의 이름을 결정한다.
- publicPath : 애플리케이션 모든 asset들의 기본 경로를 지정할 수 있다.
- compareBeforeEmit : 출력 파일 시스템에 쓰기 전에 내보낼 파일이 이미 존재하고 동일한 내용이 있는지 확인
- filename : 각 출력 번들의 이름

### Template Strings

[name] 이렇게 생긴것들 종류

- 청크 수준
  - id : 청크의 identifier
  - name : 청크의 이름, 설정되지 않은 경우 청크의 ID
  - chunkhash : 청크의 모든 요소를 포함한 청크의 해시
  - contenthash : 콘텐츠 타입의 요소만 포함하는 청크의 해시
- 모듈 수준
  - id: 모듈 id
  - hash : 모듈의 해시
  - contenthash : 모듈 콘텐츠의 해시
- 파일 수준(file loader)
  - file : 파일 이름
  - query : 앞에 ?가 있는 쿼리
  - fragment : 앞에 #가 있는 쿼리
  - base : 경로 없이 확장자를 포함한 파일 이름
  - path : 파일 이름을 제외한 경로
  - ext : . 으로 시작하는 확장자

## loader + 자주 쓰는 loader들의 동작

당연하지만 자원에 따라 동작이 다르다

- loader는 모듈을 입력받아 원하는 형태로 변환한 후 새로운 모듈을 출력한다.
- 로더 적용 순서 : 특정 파일에 대해 여러개의 로더를 사용하는 경우 오른쪽에서 왼쪽 순서로 적용되기 때문에 scss는 sass-loader을 쓴 다음에 css-loader로 이동해야 한다.

```js
module: {
  rules: [
    {
      test: /\.scss$/,
      use: ["css-loader", "sass-loader"],
    },
  ];
}
```

- file-loader : 주로 이미지 파일(png, svg, jpg)등을 처리할때 사용한다. 번들링된 결과물은 dist내부로 복사된다. output의 publicPath 속성을 설정하면 img 태그의 src 속성들이 public 디렉토리 내부의 번들링된 이미지를 가리킨다.
  - outputPath: 로더를 거친 파일들이 번들링될 위치 설정
  - publicPath: 타겟 파일들에 대한 publicPath를 설정. 이거 따로 설정 안하면 output의 publicPath를 따라가는 듯 함
- css-loader, style-loader : css-loader는 CSS 파일을 평가하고, style-loader는 HTML 문서의 head 영역 안에 인터널 방식으로 스타일 코드를 추가한다.
  - MiniCssExtractPlugin : 이렇게 head로 넣는 방식이 아니라 별도 파일로 추출한다. CSS 코드가 포함된 JS 파일 별로 CSS 파일을 생성한다.
  ```js
   {
      test: /\.css$/i,
      use: [MiniCssExtractPlugin.loader, "css-loader"],
    },
  ```
  - CSS에서 url 함수에 파일명을 지정할 수 있는데 이를 모듈에서 발견하면 웹팩은 file-loader을 통해 파일을 복사한다.
- url-loader : 작은 이미지나 글꼴 파일은 복사하지 않고 문자열 형태로 변환하여 번들 파일에 넣어버린다. Data URL Scheme을 사용한다. 아이콘처럼 용량이 적거나 반복해서 사용하지 않는 이미지의 경우 이렇게 처리하면 유용하다.
  - 특정 용량을 넘지 않는 특정 확장자의 파일들을 URL로더로 처리할 수 있게 옵션을 줄 수 있다.
  ```js
  {
    test: /\.svg)$/,
    use: {
      loader: 'url-loader',
      options: {
        name: '[name].[ext]?[hash]',
        publicPath: './dist/',
        limit: 10000 // 10kb
      }
    }
  }
  ```
- babel-loader : 번들링 결과물에 바벨을 적용할 수 있게 해준다
- ts-loader : tsconfig.json을 찾아서 타입스크립트를 컴파일한다

## Optimization

minimize랑 splitchunks를 제외하면 일반적인 경우에 크게 커스텀할 것은 없어보이고, 대부분 production에서 기능하는 최적화 옵션을 키고 끄기 위해 이용되는 듯 하다.

- **minimize**: TerserWebpackPlugin을 사용하여 번들 결과물을 minimize(압축)하는 옵션
  - [terser](https://github.com/terser/terser) : ES6+를 위한 mangler, compressor 툴이다. uglify-es의 포크로 uglify를 계승한 압축 툴이다.
  - terser plugin의 설정을 변경하려면 minimizer 옵션을 쓴다
- **splitchunks**: 공통 청크 옵션
- nodeEnv : webpack이 process.env.NODE_ENV를 주어진 문자열로 설정하게 한다. 원래는 mode 설정값을 그대로 따라간다
- removeAvailabeModules : 모듈이 이미 상위에 포함되어있는 경우 청크에서 해당 모듈을 감지하고 제거한다. Production 모드에서는 기본적으로 활용함
- providedExports : export 절에서 보다 효율적인 코드를 생성하기 위해 모듈에서 어떤 export가 제공되는지 파악한다. 기본적으로 활성화
- removeEmptyChunks : 빈 청크를 감지하고 제거한다 디폴트로 true
- sideEffects: package.json의 sideEffect를 인식한다. 사이드 이펙트 없음으로 플래그된 모듈은 export가 사용되지 않을 때(모듈이 사용되지 않을때?) 건너뛴다. => 뒤에서 더 자세히 알아보자
- innerGraph : 사용되지 않은 export에 대해 내부 그래프 분석을 수행할지 결정한다

## SplitChunks 옵션의 동작과 활용

## Code Splitting (dyanamic import)

## TreeShaking

package.json의 sideEffects 이해하기

### 웹팩의 TreeShaking

## Sourcemap

- Inline sourcemap

## Output의 모듈 시스템 그리고 Rollup

### Webpack의 Output 속성

## 뭐 하는지 잘 몰랐던 속성들
