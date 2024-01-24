## 로그인과 회원가입 실제로 할 때 주의점

### 서버 액션

- 서버 액션은 네트워크 탭에서 API 요청에 실패해도 성공이라고 뜬다.
  - 실제 요청이 성공했는지 보려면 터미널로 확인해야 한다.

### 쿠키 전달

- credentials 옵션으로 API 요청에 쿠키나 인증 헤더 정보를 포함시킬 수 있다.
  - `credentials : 'include'`를 넣어주어야 (회원가입 시)서버로 쿠키 전달이 가능하고 (로그인을 할 때) 세션 쿠키가 브라우저에 등록이 될 수 있다.
  - axios에서는 전역설정으로 처리할 수 있다.
    - `axios.defaults.withCredentials = true`

### rewrite

Next.js에서 제공하는 `rewrite`를 이용해 요청 경로를 다른 대상 경로에 매핑할 수 있습니다.

- 이미지 source가 백엔드 서버에 있어 `/upload/:slug`에 대한 요청 경로를 `http://localhost:9090/upload/:slug`로 바꿔야하는 경우

  ```tsx
  const nextConfig = {
    async rewrites() {
      return [
        {
          source: "/upload/:slug",
          destination: "http://localhost:9090/upload/:slug",
        },
      ];
    },
  };

  module.exports = nextConfig;
  ```

## 업로드 이미지 미리보기

### textarea 크기 자동으로 늘리기

`react-textarea-autosize`를 사용하면 스크롤 없이 textarea의 크기를 자동으로 늘릴 수 있습니다.

```tsx
import TextareaAutosize from 'react-textarea-autosize';
...
<TextareaAutosize/>
```

### 업로드 이미지 미리보기

`FileReader.readAsDataURL()` 메서드는 컨텐츠를 특정 Blob 이나 File에서 읽어 오는 역할을 합니다. 읽어오는 read 행위가 종료되는 경우에, readyState의 상태가 DONE이 되며, loadend 이벤트가 트리거 됩니다.

- Blob(Binary Large OBject): 바이너리로 표현 가능한 데이터(주로 이미지, 오디오, 영상 등)를 다룰 때 사용한다.

  ```tsx
  const [preview, setPreview] = useState<Array<{dataUrl: string, file: File} | null>>([]);

  ...

  const onUpload: ChangeEventHandler<HTMLInputElement> = (e) => {
  e.preventDefault();

  if (e.target.files) {
      Array.from(e.target.files).forEach((file, index) => {
      const reader = new FileReader();

      reader.onloadend = () => {
          setPreview((prevPreview) => {
          const prev = [...prevPreview];
          prev[index] = {
              dataUrl: reader.result as string, // image src에 해당하는 문자열
              file,
          };
          return prev;
          });
      };

      reader.readAsDataURL(file); // 파일을 url로 읽어옴, 종료 후 reader.onloadend 실행
      });
  }
  };

  ...

  {preview.map((v, index) => (
      v && (<div key={index} style={{flex: 1}} onClick={onRemoveImage(index)}>
      <img src={v.dataUrl} alt="미리보기" style={{width: '100%', objectFit: 'contain', maxHeight: 100}}/>
      </div>)
  ))}
  ```

## 서버 쿠키 공유하기 & 게시글 업로드 완성

파일 업로드는 로그인 되어있는 사용자만 가능하고, 로그인 여부는 쿠키 유무로 결정합니다.

현재 세션 토큰을 이용해 프론트 서버에만 로그인 된 상태이고,
백엔드 서버에서는 세션 토큰을 사용하지 않습니다.

따라서, 서버에 로그인 하기 위해서는 `Connect.sid`토큰을 사용해야 합니다.

- 프론트: 세션토큰
- 백엔드: Connect.sid

```tsx
// 로그인 요청(auth)을 통해 가져온 쿠키를 꺼내오는 코드
let setCookie = authResponse.headers.get("Set-Cookie");
console.log("set-cookie", setCookie);

if (setCookie) {
  const parsed = cookie.parse(setCookie);
  cookies().set("connect.sid", parsed["connect.sid"], parsed); // 브라우저에 쿠키를 심어주는 코드
}

if (!authResponse.ok) {
  return null;
}
```

`cookie` : 문자열 쿠키를 객체로 변경해주는 라이브러리

프론트 서버는 공용이기 때문에 개인 정보 보호를 위해 프론트 서버에 쿠키를 심으면 안 되고 개인 브라우저에 심어야 합니다.

## useMutation

1. 상태관리

   isError, isIdle, isPending, isPaused, isSuccess, failureCount, failureReason 등 상태 관리를 간단하게 할 수 있다.

2. optimistic update

   성공했다고 가정하고 제출한 내용을 바로 보여준다. 유저의 반응속도를 빠르게 할 수 있다는 점에서는 긍정적이지만 에러가 난다면 유저가 기분이 상할 수 있다.

   하트 누르기 정도가 ok

### options

```tsx
useMutation({
  mutationFn: (_) => {},
  onMutate: (variables) => {},
  onError: (error, variables, context) => {},
  onSuccess: (data, variables, context) => {},
  onSettled: (data, error, variables, context) => {},
});
```

- mutationFn : API 요청
- onMutate : 뮤테이션 함수가 호출되었을 때 실행됨
- onSuccess : 성공했을 때
- onError : 실패했을 때
- onSettled : 성공이든 에러든 이 두 개 다 끝났을 때 (= mutation이 완전히 종료되었을 때) 호출되는 함수

### options의 props

```tsx
async onSuccess(response, variable, context) {}
```

- response : mutationFn()의 응답, fetch해온 데이터
- variable : mutationFn()의 매개변수
- context : onMutate()의 return 한 값

### pending과 fetching 차이

- pending : 초기에 데이터를 아직 안 불러온 상태
- fetching : 데이터를 불러오고 있는 상태

## 주소에 해시가 들어가면 문제가 됩니다.

- 백엔드 측에서 내 정보를 고려한 응답을 주기 위해 **로그인한 상태**를 알아야 하면 `credentials:'include'` 넣어주어야 한다.
- 객체를 `Object.toString()`해서 주소에 넣어주면 에러가 발생한다.
  - URL의 쿼리 문자열을 사용하는 `URLSearchParams()`를 문자열로 바꾸면(`toString()`) 여러 형태의 값을 url로 넘길 수 있다.
  ```tsx
  const urlSearchParams = new URLSearchParams(searchParams);
  const res = await fetch(
    `http://localhost:9090/api/posts?${urlSearchParams.toString()}`
  );
  ```
- Hash(#)은 특수기호로 분류되어 서버에 전달되지 않는다.
  - `${encodeURIComponent('#해시)}`로 Hash를 사용할 수 있다.

## 하트 누를 때 optimistic update 적용하기

- 하트를 클릭할 때 상위의 link 태그에 영향을 받지 않으려면 `e.stopPropagation()` 코드를 추가하면 된다.

### mutation 작성

- mutationFn : 하트 클릭에 대한 서버 요청을 보내는 메서드
- onMutate : 뮤테이션 함수가 호출되었을 때 실행. 하트에 대한 낙관적 업데이트 실행한다.(=> 하트 상태를 true로 바꿔줌).
  - 방금 누른 하트에 대한 쿼리키 정보가 없기 때문에 쿼리 클라이언트로 모든 쿼리키를 가져와 posts를 찾아 하트와 관련된 낙관적 업데이트를 실시한다.
    - `queryClient.getQueryCache()`로 쿼리 캐시를 꺼내오고 `queryCache.getAll().map(cache => cache.queryKey)`로 모든 쿼리키를 가져온다.
  - `queryClient.setQueryData(queryKey, shallow);`로 쿼리 클라이언트에 하트 클릭을 낙관적으로 업데이트한다.
- onError : 실패했을 때 하트 이벤트에 대한 롤백을 실시한다.(하트 에러 발생 -> 언하트 onMutate 실행)
- onSettled : 성공이든 에러든 최종적으로 수행되는 메서드. `queryClient.invalidateQueries({queryKey:['posts']})`를 통해 posts와 관련된 쿼리들을 모두 업데이트
  - 하트 한 번에 전체 포스트 업데이트는 무리이므로 선택사항
  - 서버와 정확하게 일치시키고 싶은 경우에만 사용한다.

### immer를 통해 간편하게 불변성 관리하기

```tsx
import produce from "immer";

const baseState = [
  { id: 1, name: "A" },
  { id: 2, name: "B" },
];

const nextState = produce(baseState, (draftState) => {
  draftState.push({ id: 3, name: "C" });
});
```

## 서로 다른 컴포넌트 간 query 일치하게 하기

### 문제

낙관적 업데이트가 적용된 팔로우 버튼 클릭 시, 다른 컴포넌트의 팔로우 버튼은 업데이트가 되지 않는다.

    유저 프로필 컴포넌트에서 팔로우에 대한 상태가 업데이트 되지 않는다.

### 해결방법

해당 컴포넌트에서도 낙관적 업데이트가 되도록 팔로우 관련 쿼리에 낙관적 업데이트 필요하다.

    내가 팔로우하고 있는 목록에 대한 낙관적 업데이트 필요하다.

## 완전 로그아웃 하기

- 로그아웃을 수행할 때 로그인 상태와 연관되어 있는 쿼리(유저 정보, 포스트 등)에 대한 캐시를 날려주어야 한다.

  ```tsx
  const onLogout = () => {
    queryClient.invalidateQueries({
      queryKey: ["posts"],
    });
    queryClient.invalidateQueries({
      queryKey: ["users"],
    });

    signOut({ redirect: false }).then(() => {
      fetch(`${process.env.NEXT_PUBLIC_BASE_URL}/api/logout`, {
        method: "post",
        credentials: "include",
      });
      router.replace("/");
    });
  };
  ```

- 백엔드 쿠키도 지워주어야 백엔드 유저 정보에 접근이 되지 않는다.
  - 백엔드에 로그아웃 메서드를 요청하면 쿠키가 사라진다
- 서버에서 prefetch하는 API 요청 메서드가 실행될 때 쿠키가 전달되지 않는 문제 발생한다.
  - 클라이언트 컴포넌트와 달리 `credentials: "include"`로 해결x
  - 헤더에 직접 쿠키를 넣어서 해결
    - `headers: {Cookie: cookies().toString()}`
    - 클라이언트 컴포넌트는 쿠키를 직접 넣을 수 없으므로 서버용, 클라이언트용 요청 함수를 분리해서 사용한다.

## 메타데이터

아래와 같이 사용할 수 있다.

```tsx
import { metadata } from "next";

export const metadata: Metadata = {
  title: "홈",
  description: "홈",
};
```

### 메타데이터를 동적으로 바꿔야하는 경우

`generateMetadata` 메서드를 이용해 메타데이터를 생성할 수 있다.

```tsx
export async function generateMetadata({ params }: Props) {
  const user: User = await getUserServer({
    queryKey: ["users", params.username],
  });
  return {
    title: `${user.nickname} (${user.id}) / Z`,
    description: `${user.nickname} (${user.id}) 프로필`,
    openGraph: {
      images: ["/some-specific-page-image.jpg", ...previousImages],
    },
  };
}
```

## SSR 적용 여부 판단 기준

개발자도구 > 네트워크 > 미리보기에서 서버 사이드 렌더링 미리보기를 할 수 있습니다.

### SSR이 필요한 경우

- 로그인을 안 해도 페이지에 접근할 수 있는 경우
- 공유가 활발한 페이지
  - 카카오톡 공유 등

SSR이 필요한 경우가 아니라면 굳이 SSR로 구현할 필요는 없습니다.(프론트엔드 서버에 부담이 많이 가기 때문)

## 재게시, 답글 기능 zustand로 만들어보기

### zustand

- 보일러 플레이트 코드가 많은 redux보다 간단하게 사용할 수 있다.
- 상태만 공유하는 Context API와 달리 최적화가 잘 되어있다.
- React Query에서도 stale time과 GC time을 조정해 상태 공유를 할 수 있으나 서버 데이터를 가져온다는 원래의 용도와는 다르다.

### store.tsx

```tsx
import { create } from "zustand";
import { Post } from "@/model/Post";

interface ModalState {
  mode: "new" | "comment";
  data: Post | null;
  setMode(mode: "new" | "comment"): void;
  setData(data: Post): void;
  reset(): void;
}

export const useModalStore = create<ModalState>((set) => ({
  mode: "new",
  data: null,
  setMode(mode) {
    set({ mode });
  },
  setData(data) {
    set({ data });
  },
  reset() {
    set({
      mode: "new",
      data: null,
    });
  },
}));
```

불변성을 지키기 위해 set함수를 이용해서만 데이터를 변경할 수 있습니다.

### 상태를 쓰는 컴포넌트

```tsx
import { useModalStore } from "@/store/modal";

const modalStore = useModalStore();

const onClickComment: MouseEventHandler<HTMLButtonElement> = (e) => {
  e.stopPropagation();

  modalStore.setMode("comment");
  modalStore.setData(post);
  router.push("/compose/tweet");
};
```
