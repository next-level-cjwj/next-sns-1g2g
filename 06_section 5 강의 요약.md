# Next.js의 캐싱

## Request Memoization과 Data Cache

- Duration: 지속 기간. 캐시가 얼마나 오래가는지.
- Revalidating: 갱신 방법. 캐시를 어떻게 새로 가져오는지.
- Opting Out: 캐시를 사용하지 않는 방법.

### Request Memoization

한 페이지를 렌더링 할 때, 동일한 url과 옵션을 가진 요청을 메모해서 여러 개의 동일한 요청을 한 번만 보내도록 처리할 수 있습니다.

![image](https://nextjs.org/_next/image?url=%2Fdocs%2Flight%2Fdeduplicated-fetch-requests.png&w=1920&q=75&dpl=dpl_BgBz72DvgwwwWh4xq5inDH2WcEcU)

- Duration: 캐싱된 데이터는 리액트 컴포넌트 트리의 렌더링이 끝날 때까지 서버 요청 life cycle 동안 유지된다.
- Revalidating: 페이지를 렌더링하면 다음에는 새로운 요청을 보낸다.

### Data Cache

Next.js에 들어오는 fetch 요청 결과에 대한 캐싱을 의미합니다. 백엔드 또는 DB로 보낸 요청을 얼마나 오래 캐시할 것인지 설정할 수 있습니다.

- 요청 내부에 `cache: "no-store"`을 하면 캐시 하지 않는다.
  - 새로운 데이터를 매번 갱신해야 하는 경우에 사용한다.
- Duration: 한 번 캐시하면 그 데이터를 계속 유지한다.
  - `revalidate`나 `opt-out`으로 적절히 설정 해주어야 한다.
- Revalidating
  - **Time-based Revalidation**: 일정 시간이 지나면 자동으로 데이터 갱신
    - fetch 요청 내부에 `revalidate` 시간을 설정해준다.
    ```ts
    const res = await fetch(
      `${process.env.NEXT_PUBLIC_BASE_URL}/api/posts/${id}`,
      {
        next: {
          revalidate: 3600,
          tags: ["posts", id],
        },
        credentials: "include",
        headers: { Cookie: cookies().toString() },
      }
    );
    ```
  - **On-demand Revalidation**: 직접 코드를 작성하여 데이터 갱신
    - `revalidateTag` 또는 `revalidatePath` 이용하기
      - `revalidateTag("recommends")` : 태그에 관련된 요청만 새로고침
      - `revalidatePath('home')` : 'home'의 경로에 대한 모든 요청을 새로고침
- Opting out

  - `cache: "no-store"`
    요청 별로 캐싱 옵션을 설정해줄 수 있습니다.

    ```tsx
    const res = await fetch(`${process.env.NEXT_PUBLIC_BASE_URL}/api`, {
      cache: "no-store",
    });
    ```

  - `force-dynamic`

    패이지에서 보내는 모든 요청(라이브러리 포함)을 캐싱하지 않습니다. 써드 파티 라이브러리에서 보내는 요청은 캐싱을 막을 수 없기 때문에 아래 코드를 추가해서 모든 캐싱을 중단할 수 있습니다.

    ```tsx
    export const dynamic = "force-dynamic";
    ```

### 정리

- 인기글, 새로운 글과 같이 업데이트가 필요한 글은 `cache: "no-store"`로 데이터를 캐싱하지 않는 것이 좋다.
- 블로그나 뉴스 기사와 같이 데이터가 바뀔 일이 많지 않은 글은 Time-based Revalidation 또는 On-demand Revalidation으로 일정 조건에 의해 캐시를 지우고 데이터를 갱신할 수 있습니다.

## Full Route Cache와 Router Cache

### Full Route Cache

페이지를 빌드 타임에 렌더링하고 캐시하여 페이지를 매 요청 마다 서버에서 렌더링하지 않고 캐시된 페이지를 보여줄 수 있는 최적화 기능입니다. 변하는 컨텐츠가 있으면 안 됩니다.

- Duration: 기본적으로 유지된다.
- Invalidation
  - data cache가 수정되는 순간: 보내는 요청 중에 `cache: "no-store"`이 있다면 매번 데이터를 새로 받아오게 된다.
  - 재배포 시 빌드를 하면서 캐시가 갱신된다.
- Opting out
  - `dynamic = 'force-dynamic'` 또는 `revalidate = 0`과 같이 데이터 캐시를 사용하지 않는 경우
  - Dynamic Function(cookies, headers,the searchParams)을 사용하여 런타임에 동적으로 바뀌는 경우

### Router Cache

클라이언트에서 컴포넌트 별로 캐싱을 수행합니다. 예를 들어, 페이지 이동 시에 `layout.tsx`가 겹치는 경우 `layout.tsx`을 새로 그리지 않고 캐싱된 `layout.tsx`을 사용합니다.

- Duration

  - Session: 페이지를 새로고침하기 전까지 유지된다.

  - Automatic Invalidation Period

    - Statically Rendered-5분, Dynamically Rendered-30초

    ![image](https://github.com/1g2g/my-little-algorithm/assets/87280835/6970b7cf-e646-4c35-ae19-17e4093432ba)

    - Dynamic Function(cookie, searchParams) X + Data cached => 컨텐츠가 고정된 블로그나 뉴스 기사에 사용할 수 있다.
    - 그외에는 거의 Dynamically Rendered
    - Dynamically Rendered의 캐싱 시간을 5분으로 늘리고 싶으면 `prefetch=true` 속성을 넣으면 된다.

- Invalidation

  - `revalidateTag` 또는 `revalidatePath` 이용하기
  - 쿠키 설정 또는 삭제
  - router refresh 호출

- Opting out

  - 불가능

### 정리

- Request Memoization: 중복된 같은 요청 묶어서 하나의 요청만 보내는 것
- Data Cache: 프론트 서버에서 백엔드 서버로 요청 보낼 때 응답을 얼마나 캐싱할 것인지
- Full Route Cache: 정적인 페이지에 대한 캐싱
- Router Cache: 방문했던 공통 컴포넌트에 대한 캐싱

# 배포

Next.js의 배포 모드에는 Static Mode와 Dynamic Mode 두 가지가 있습니다.

## Static Mode

next 서버 없이 html 페이지로만 구성된 정적인 사이트(SSG). 빌드하는 순간 보든 컨텐츠가 구성됩니다.

next.config.js에 아래 설정을 추가하면 됩니다.

```js
const nextConfig = {
  output: "export",
  ...
};
```

- ISR: 페이지 라우터 시절에 지원하던 기능. 빌드 시 정적인 페이지를 모두 만든 이후에 주기적으로 확인해 추가된 데이터에 대한 빌드를 해준다.
- App Router에서는 Full Route Cache와 Revalidating Data로 ISR 구현
  - 새로운 컨텐츠 추가: Full Route Cache로 구현(데이터 캐시나 다이나믹을 사용하지 않은 새로 발행한 콘텐츠에 한 번 접속하기만 하면 계속 캐싱됨)
  - 기존 콘텐츠 수정: Data Cache Revalidating(캐시 날리기)

## Dynamic Mode

아래 요소들 중 하나라도 사용하고 있으면 Dynamic Mode를 사용합니다.

- Dynamic Routes with dynamicParams: true
- Dynamic Routes without generateStaticParams()
- Route Handlers that rely on Request
- Cookies
- Rewrites
- Redirects
- Headers
- Middleware
- Incremental Static Regeneration
- Image Optimization with the default loader
- Draft Mode

## 배포 직전 빌드하기

- npm run build 또는 `next build`로 빌드를 수행하여 코드를 production용으로 변환할 수 있다.
- 터미널을 통해 페이지 별 빌드 수행 결과를 확인할 수 있다.
  - js파일 사이즈가 너무 큰 경우 경고가 뜬다. 수정이 필요하다.
  - 대부분의 경우는 코드 스플리팅 되어있어 용량이 낮음
  - 라이브러리 등으로 인해 용량이 큰 경우는 코드 스플리팅 또는 트리쉐이킹을 적용해보자.
- `npm run start` 또는 `next start`를 통해 실제 서버를 실행할 수 있다.
  - 로컬에서 돌아가는 서버이므로 aws 등 호스팅 플랫폼을 통해 연결을 해야 실제로 배포할 수 있다.

## 보너스: 카톡 공유용 데이터 넣기

### Meta Data Files

공유하기 기능을 사용할 때 데이터를 추가하고 싶으면 meta data files를 추가하면 된다. App 디렉토리 내에 있으면 자동으로 인식한다.

- favicon, icon, apple-icon
- manifest.json
- opengraph-image, twitter-image
- robots.txt
- sitemap.xml

### Next.js의 Functions

generateMetadata로 텍스트 데이터를 수정할 수 있고, open Graph로 공유 시 보이는 이미지 데이터를 설정해줄 수 있다.

```tsx
export const metadata = {
  title: "Acme",
  openGraph: {
    title: "Acme",
    description: "Acme is a...",
  },
};
```

- 메타 데이터 추가 후 배포하면 og가 붙은 메타태그들이 생성된 것을 알 수 있다.
- `https://www.opengraph.xyz/`에서 openGraph가 적용되었는지 디버깅 해볼 수 있다.

## /message 페이지 수정하기

- 주소은 `http://localhost:8080/messages/내 아이디/상대방 아이디`로 구성
- 서버 컴포넌트에서 데이터를 가져오기 위해 사용자 정보를 넘기려면(쿠키를 보내려면) fetch 요청에 `credential:'include'`와 `headers: { Cookie: cookies.toString() }`이 포함되어야 함.

## 웹소켓으로 실시간 채팅 구현하기

- 백엔드와 동일한 버전의 Socket.io-client를 설치해야한다.
- 메세지 뿐만 아니라 실시간 알림도 구현할 수 있다.
- 서버 컴포넌트에서 Socket.io를 쓰지 않도록 주의해야 한다.
  - 메모리 누수 발생할 수 있다.
  - 소켓을 관리하는 훅을 따로 생성한다.
- `const socket = io(주소)`로 클라이언트와 서버를 연결하는 소켓을 생성한다.
  - `transports: [’websocket’]`옵션을 통해 소켓을 지원하지 않는 구형 브라우저에서는 HTTP 폴링 방식을 지원한다.
- 훅 안에 있는 상태는 공유되지 않지 않기 때문에 커스텀 훅 간에 공유해야 할 상태는 외부에 선언해야 한다.

  - 훅 외부에 `let socket`으로 선언하고 export해서 사용한다.

### 메서드

- emit: 클라이언트에서 발생한 이벤트를 서버로 보낸다.
  ```ts
  socket.emit("sendMessage", {
    senderId: "sender",
    receiverId: "receiver",
    content: "content",
  });
  ```
- on: 어떤 이벤트에 대한 구독. 서버로부터 이벤트가 발생한 것을 수신할 수 있다.
  ```ts
  socket.on("receiveMessage", (data) => {});
  ```
- off: 이벤트 연결을 끊는다.
  ```ts
  socket.off("receiveMessage");
  ```

## Vanilla Extract

### 장점

- css 문법을 그대로 활용한다.
  - tailwind만의 불편함을 초래할 수 있는 문법을 사용하지 않는다.
- styled component와 달리 서버 사이드 렌더링이 잘 된다.

### 설치

```
$ npm install @vanilla-extract/css
$ npm install --save-dev @vanilla-extract/next-plugin // next
```

next.config.js에 아래 내용을 추가해주면 된다.

```ts
const { createVanillaExtractPlugin } = require("@vanilla-extract/next-plugin");
const withVanillaExtract = createVanillaExtractPlugin();

/** @type {import('next').NextConfig} */
const nextConfig = {};

module.exports = withVanillaExtract(nextConfig);
```

- `css.ts`라는 확장자를 통해 Vanilla Extract 작성

### globalStyle

`globalStyle`을 통해 전역 선택자에 스타일을 부여할 수 있습니다.

```tsx
// vanilla-extract 버전
import { globalStyle } from "@vanilla-extract/css";

globalStyle("html, body", {
  margin: 0,
});
```

```css
/* css 버전 */
html,
body {
  margin: 0;
  boxsizing: "border-box";
}
```

- 객체 내부에는 대쉬(-)가 들어가지 않도록 캐멀케이스 사용
- 기본 단위는 픽셀

- 다크모드일 때 미디어쿼리

  ![Alt text](image-1.png)

  ```ts
  globalStyle(":root", {
    "@media": {
      "(prefers-color-scheme: dark)": {
        // 루트가 다크모드일 때
        colorSheme: "dark",
      },
    },
  });
  ```

### css 전역변수 선언하기

createGlobalThemeContract: 테마에서 사용할 요소의 이름을 정해둘 수 있다.
createGlobalTheme: 테마가 적용될 범위 및 값을 정할 수 있다.

```ts
// vanilla-extract 버전
import {
  createGlobalThemeContract,
  createGlobalTheme,
  style,
} from "@vanilla-extract/css";

export const vars = createGlobalThemeContract({
  color: {
    brand: "color-brand",
  },
  font: {
    body: "font-body",
  },
});

createGlobalTheme(":root", vars, {
  color: {
    brand: "blue",
  },
  font: {
    body: "arial",
  },
});

export const brandText = style({
  color: vars.color.brand,
  fontFamily: vars.font.body,
});
```

```css
/* css 버전 */
:root {
  --color-brand: blue;
  --font-body: arial;
}
.themes_brandText__1k6oxph0 {
  color: var(--color-brand);
  font-family: var(--font-body);
}
```

- createGlobalThemeContract로 선언한 변수는 globalStyle에서 바로 사용 가능하다.
- 변수명으로 넣어두는 이유는 테마가 화이트/다크 모드에 따라서 바뀔 수 있기 때문이다.

```ts
export const vars = createGlobalThemeContract({
  color: {
    brand: "color-brand",
  },
  font: {
    body: "font-body",
  },
});

globalStyle("body", {
  color: vars.color.brand, // "color-brand"
});
```

### 컴포넌트 스타일 적용하기

```ts
import { style } from "@vanilla-extract/css";

export const flexContainer = style({
  display: "flex",
});
```

```ts
import * as styles from "../main.css";

...

<div className={styles.flexContainer}></div>
...
```

        리액트에서는 `.ts`확장자를 명시하면 안 됨. 리액트가 알아서 ts인지 js인지 구분한다.

### 자식 선택자

```css
.left img {
}
```

```ts
export const left = style({ });

glbalStyle({`${left} img`},{ })
```

- selectors로는 자식선택자를 선택하지 못한다.
- `export const leftImage = style({ });` 새로운 클래스 네임 생성 방식 추천한다.
- 상태 선택자(hover)는 내부에서 사용할 수 있ㅇ다.

  ```ts
  export const button = style({
    color: "blue",
    ":hover": {
      color: "red",
    },
  });
  ```

  - 두 개 이상의 중첩된 상태 선택자는 사용할 수 없다.
    ```tsx
    export const button = style({
      ":disabled:hover": {
        color: "red",
      },
    });
    ```
  - selectors를 이용해 두 개 이상의 중첩된 상태 선택자 사용한다.
    ```tsx
    export const button = style({
      selectors: {
        "&:disabled:hover": {
          color: "red",
        },
      },
    });
    ```
    - selectors는 자기 자신을 가리킬 때만 사용 가능하다.

- 여러 미디어 쿼리도 내부에서 사용할 수 있다.
  ```ts
  export const button = style({
    "@media": {
      "(prefers-color-scheme: dark)": {
        colorSheme: "dark",
      },
      "(min-width: 687px)": {
        color: "white",
      },
    },
  });
  ```
- 내부에서 변수를 사용할 수 있다.
  ```ts
  export const button = style({
    backgroundColor: global.background.color;
    });
  ```
