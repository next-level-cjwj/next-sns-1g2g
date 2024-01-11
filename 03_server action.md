# Server Action

서버 액션은 폼의 뮤테이션(생성, 업데이트, 삭제)을 할 수 있는 기능을 제공합니다. 서버 액션을 통해 클라이언트에 데이터를 가져오고 변경하는 것에 대한 보안적인 이슈를 해결할 수 있습니다.

서버 액션을 사용하면, API 엔드포인트를 생성하지 않고도 db에 직접 접근해 함수 내용을 자동으로 서버 API로 만들어 요청을 한 번에 처리할 수도 있습니다.

```js
async function handleSubmit(formData) {
  "use server";

  const db = (await connectDB).db("test");

  await db.collection("post_test").insertOne({ title: formData.get("post1") });
}
```

네트워크 호출이 항상 비동기적이기 때문에 'use server'는 반드시 비동기 함수에서만 사용할 수 있습니다.

서버 액션은 서버 측 상태를 업데이트하는 뮤테이션에 사용되도록 설계되었습니다. 데이터 검색에는 권장되지 않습니다.

### 사용 방법

#### 서버 컴포넌트

서버 컴포넌트에서는 컴포넌트 내에 서버 액션 함수를 정의하고 함수 상단에 인라인으로 `use server`를 작성하면 됩니다.

```tsx
// Server Component
export default function Page() {
  // Server Action
  async function serverAction() {
    'use server'
    ...
  }
 ...
}
```

#### 클라이언트 컴포넌트

`'use server'`는 서버 컴포넌트에서만 사용할 수 있습니다.

따라서, 클라이언트 컴포넌트에서는 서버 액션 함수를 따로 모듈로 관리해야 합니다.

```tsx
// Server Action
// 'use server' => 파일 전체가 서버 액션인 경우
export async function serverAction(){
    'use server'
    ...
}
```

```tsx
import { serverAction } from "@/app/_lib/actions";

// Server Component
export default function Page() {
    ...
}
```

### 서버 컴포넌트에서 server actions 활용하기

서버 액션은 기존 리액트에서 click 이벤트 또는 submit 이벤트로 폼을 제출하는 방식이 아닌, 폼의 action을 사용해 서버로 폼의 데이터를 제출합니다.

```tsx
// Server Component
export default async function Home() {
  const submit = async (formData: FormData) => {
    "use server";
    const response = await fetch(
      `${process.env.NEXT_PUBLIC_BASE_URL}/api/users`,
      {
        method: "post",
        body: formData,
        credentials: "include",
      }
    );
  };

  return (
    <form action={submit}>
      <input type="text" name="title" required />
      <input type="text" name="content" required />
      <button>submit</button>
    </form>
  );
}
```

네트워크 탭을 확인하면 제출되는 것을 확인할 수 있습니다.

<img width="333" alt="image" src="https://github.com/Java-and-Script/pickple-log/assets/87280835/bb1dcfd0-1268-492b-9af4-a93790d37789">

### 클라이언트 컴포넌트에서 server actions 활용하기

클라이언트 컴포넌트에서는 서버 액션 함수를 따로 모듈로 관리해야하기 때문에 따로 빼주었습니다.

```tsx
"use server";

export const onSubmit = async (prevState: any, formData: FormData) => {
  const response = await fetch(`localhost:9000/api/users`, {
    method: "post",
    body: formData,
    credentials: "include",
  });
};
```

react-dom에서 제공하는 `useFormState`와 `useFormStatus` 훅을 이용하여 입력 필드의 값을 상태로 관리하고 폼의 제출 여부 등을 알 수 있습니다.

```tsx
"use client";

import { useFormState, useFormStatus } from "react-dom";
import { onSubmit } from "../_component/onSubmit";

export default function Home() {
  const [state, formAction] = useFormState(onSubmit, { message: null });
  const { pending } = useFormStatus();

  return (
    <form action={formAction}>
      <input type="text" name="title" required />
      <input type="text" name="content" required />
      <button disabled={pending}>submit</button>
    </form>
  );
}
```

결과는 아래와 같습니다.

<img width="333" alt="image" src="https://github.com/Java-and-Script/pickple-log/assets/87280835/14fd40b4-088b-4d2d-8471-138ba581d75b">

### 참고

https://codingapple.com/unit/nextjs-server-actions/
https://react.dev/reference/react/use-server
https://react.dev/reference/react-dom/hooks/useFormState
