## `useSelectedLayoutSegment()`와 `useSelectedLayoutSegments()`

### `useSelectedLayoutSegment()`

`useSelectedLayoutSegment()`는 클라이언트 컴포넌트에서 사용되는 훅으로, 호출된 레이아웃보다 한 수준 아래에 있는 segment를 반환합니다.

![image](https://github.com/Java-and-Script/pickple-front/assets/87280835/dedf2e32-b6ca-47b0-95ad-ac3cf71d8370)

### `useSelectedLayoutSegments()`

`useSelectedLayoutSegments()`는 호출된 레이아웃보다 아래 있는 모든 segment를 배열로 반환합니다. breadcrumb만들 때 유용하겠습니다.

![image](https://github.com/Java-and-Script/pickple-front/assets/87280835/cad2d4a4-724e-4cda-9eeb-c25db7177f76)

### good to know

> Since useSelectedLayoutSegment is a Client Component hook, and Layouts are Server Components by default, useSelectedLayoutSegment is usually called via a Client Component that is imported into a Layout.

`useSelectedLayoutSegment()`는 클라이언트 컴포넌트 훅이고, 레이아웃은 서버 컴포넌트이므로, `useSelectedLayoutSegment()`는 **레이아웃**으로 임포트된 클라이언트 컴포넌트를 통해서만 사용됩니다.

- 강의에서는 클라이언트 컴포넌트인 `NavMenu`를 서버 컴포넌트인 layout.tsx로 가져와 `useSelectedLayoutSegment()`를 이용해 segment를 가져오고 있습니다.

### 현재 경로를 가져오기 위해 `useSelectedLayoutSegment()`를 아무데서나 사용해도 되는가?

- `layout.tsx`가 아닌 `page.tsx`에서 클라이언트 컴포넌트 `NavMenu`를 임포트해서 `useSelectedLayoutSegment()` 사용하면 현재 segment로 null을 반환합니다.
- `page.tsx`에서 현재 url을 가져오고 싶다면 `usePathname()`을 사용하는 게 좋습니다.

## useSelectedLayoutSegment로 Active Link 적용해보기

### Active Link란

현재 표시 중인 링크의 아이콘이 강조 표시하는 기능입니다.

강의에서는 `useSelectedLayoutSegment()`를 이용해 segment를 가져와 segment와 일치하는 Active Link를 강조해서 보여주고 있습니다.

> 가정 : `NavMenu` 컴포넌트는 `src/app/layout.tsx`에서 사용된다. home과 user 페이지가 있으며 각각의 경로는 `src/app/page.tsx`와 `src/app/user/page.tsx`이다.

- home 페이지: http://localhost:3000
- user 페이지 : http://localhost:3000/user

우선, home 페이지는 호출된 레이아웃과 같은 수준에 있으므로 segment는 null이 되고, home 페이지의 path와 url이 일치하여 segment가 null과 같으면 볼드 처리를 해주었습니다.

user 페이지는 호출된 레이아웃보다 한 단계 아래 있으므로 segment는 user가 됩니다. 일치하는 경우 볼드 처리해줍니다.

```tsx
"use client";

import Link from "next/link";
import { useSelectedLayoutSegment } from "next/navigation";

export default function NavMenu() {
  const segment = useSelectedLayoutSegment();

  return (
    <>
      {segment === null ? (
        <Link href="/">
          <b>home</b>
        </Link>
      ) : (
        <Link href="/">
          <p>home</p>
        </Link>
      )}
      {segment === "user" ? (
        <Link href="/user">
          <b>user</b>
        </Link>
      ) : (
        <Link href="/user">
          <p>user</p>
        </Link>
      )}
    </>
  );
}
```

# `usePathname()`와 `useSearchParams()`

## `usePathname()`

`usePathname()`은 현재 URL의 경로를 문자열로 반환합니다. 쿼리스트링은 포함하지 않습니다.

![image](https://github.com/Java-and-Script/pickple-front/assets/87280835/2cb1759d-3293-4747-b363-cc4a4437e9b6)

## `useSearchParams()`

현재 URL 경로의 쿼리스트링을 읽어올 때 사용합니다.

예를 들어, url이 `http://localhost:3001/user?name=wonji`인 페이지에서 쿼리 스트링을 가져오고 싶다면 아래와 같이 사용하면 됩니다.

```tsx
"use client";

import { useSearchParams } from "next/navigation";

export default function Component() {
  const params = useSearchParams();

  console.log(params.get('name')); // wonji
  ...
}

```

get()은 찾으려는 파라미터의 첫번째 value를 반환합니다.

### URLSearchParams 메서드

- URLSearchParams.get() : 파라미터의 첫번째 value를 반환
- URLSearchParams.has() : 파라미터의 존재 여부 반환
- URLSearchParams.getAll() : 모든 파라미터의 value를 문자열로 반환
- URLSearchParams.keys() : 모든 파라미터의 key를 문자열로 반환
- URLSearchParams.entries() : 모든 파라미터의 key와 value를 문자열로 반환
- URLSearchParams.toString() : 현재 쿼리 스트링을 문자열로 반환

# `useParams()`

현재 URL에서 dynamic params를 읽어 올 수 있는 훅입니다.

<img width="671" alt="image" src="https://github.com/Java-and-Script/pickple-front/assets/87280835/5822055b-1c32-4114-8a53-1be11f80f739">

- 경로에 동적 매개변수가 없으면 null을 반환합니다.
- slug 개수에 따라 배열이 반환되기도 합니다.
