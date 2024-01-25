# React Query

### hydrate란

서버에서 렌더링된 html 파일의 DOM요소를 클라이언트에서 JS와 연결시켜주는 과정입니다.

### React Query에서는 ..

hydrate는 서버에서 온 데이터를 클라이언트의 형식에 맞추어 보여주는 것이고,

dehydrate는 `hydrate()` 또는 `<HydrationBoundary>`를 통해 hydrate 될 수 있는 내용을 캐싱해두는 과정입니다.

### Next.js+React Query로 서버 사이드에서 데이터 가져오기

1. 넥스트에서는 리액트 쿼리를 활용해서 서버 사이드 렌더링 시점에 데이터를 프리 패칭할 수 있다.(dehydrate)

2. 그다음 클라이언트 측에서 캐싱해온 데이터를 hydrate한다.

이렇게 하면 Markup->JS->Query의 과정을 Markup(with content AND initial data)->JS로 줄일 수 있습니다.

### get 요청 함수

```tsx
export async function getPostRecommends() {
  const res = await fetch(`http://localhost:9090/api/postRecommends`, {
    next: {
      tags: ["posts", "recommends"], // 태그 값에 따라 캐싱을 수행
    },
    cache: "no-store", // 캐싱이 필요없을 때 사용
  });

  if (!res.ok) {
    throw new Error("Failed to fetch data");
  }

  return res.json();
}
```

next에서는 자체적으로 서버에서 요청을 태그 값에 따라 캐싱해둡니다.

캐싱해둔 데이터를 새로고침하고 싶을 때는 아래 메서드를 사용하면 됩니다.

- `revalidateTag("recommends")` : 태그에 관련된 요청만 새로고침
- `revalidatePath('home')` : 'home'의 경로에 대한 모든 요청을 새로고침

### get 요청을 사용하는 서버 컴포넌트(홈페이지)

```tsx
export default async function Home() {
  const queryClient = new QueryClient();

  await queryClient.prefetchQuery({
    queryKey: ["posts", "recommends"],
    queryFn: getPostRecommends,
  });

  const dehydratedstate = dehydrate(queryClient);

  return (
    <main className={style.main}>
      <HydrationBoundary state={dehydratedstate}>
        <TabProvider>
          <Tab />
          <PostForm />
          <Post />
        </TabProvider>
      </HydrationBoundary>
    </main>
  );
}
```

### prefetchQuery해온 데이터를 활용하는 방법

- 데이터를 꺼내오려면 `queryClient.getQueryData(["posts", "recommends"])`
- 수정하려면 `queryClient.setQueryData(["posts", "recommends"])`

## 리액트쿼리를 사용하는 이유

1. 트래픽 관리와 UX 측면

   - 트래픽 관리를 위해 캐싱을 요청을 조절하는 것이 중요한데, React Query는 만들어진 목적에 따라 서버 데이터 캐싱 기능을 제공한다.
     - 서버 상태 관리의 목적은 데이터를 가져오는 것, 클라이언트 상태 관리의 목적은 컴포넌트 간 데이터를 공유하는 것.
     - 데이터가 잘 변하지 않는 경우 db에서 매번 꺼내오기보다는 캐싱해온 데이터를 사용하고 일정 기간 마다 데이터를 패치해오는 것이 좋다. 또는 db와 캐시를 함께 업데이트 하는 방법도 있다.
   - 사용자가 매우 빠르게 응답 결과를 볼 수 있다는 장점이 있다. 즉, 유저 이탈을 막을 수 있다.

2. 인터페이스 표준화
   - 데이터 가져올 때의 상태(로딩, 성공, 실패)를 표준 API로 사용할 수 있다.
   - key 시스템을 통해 편리하게 개발 가능하다(한 번에 업데이트 등).

## 사용법

### 기본 설정값

- 모든 데이터는 fresh하지 않다.
- 데이터는 가져오자마자 stale하다.

### stale한 데이터의 refetch 타이밍

- refetchOnWindow : 탭 전환에서 돌아온 경우
- retryhOnMount : 컴포넌트가 언마운트 되었다가 다시 마운트 될 때(다시 특정 페이지로 돌아오거나 새로고침)
- refetchOnReconnect : 인터넷 재연결 시

### useQuery 내부 속성

- retry : 데이터 패치에 실패하는 경우 몇 번 더 시도할 건지
- staleTime: n초 뒤에 fresh -> stale로 바뀐다. infinity는 데이터가 항상 fresh로 유지된다(= 더이상 업데이트하지 않는다).
- gcTime(cacheTime) : 기본 5분. inactive(현재 페이지에서 해당 쿼리를 사용하고 있지 않음)한 값을 메모리에 저장하고 있는 시간.
  - staleTime < gcTime
- initialData : 패칭 전 초기 데이터

## Actions - DevTools

- refetch : 무조건 새로 데이터를 가져온다.
- invalidate : 페이지를 보고 있지 않을 때 데이터를 요청하면 바로 수행하지 않고 페이지로 돌아가야 fetch를 수행한다.
- reset : 초기 데이터가 있을 경우 초기 데이터로 리셋되고 없을 경우 새로 데이터를 fetch해온다.
- remove : 데이터 지운다.
- restore loading : 로딩 상태를 보여준다.
- trigger error : 에러 상태 확인할 떄 사용한다.

## Query Function Variables

쿼리 키는 가져오는 데이터를 고유하게 식별하는 데 사용될 뿐만 아니라 QueryFunctionContext의 일부로 쿼리 함수에 편리하게 전달됩니다.

### useQuery

```tsx
export default function SearchResult({ searchParams }: Props) {
  const { data } = useQuery<
    IPost[],
    Object,
    IPost[],
    [_1: string, _2: string, Props["searchParams"]] // 쿼리 키 타입
  >({
    queryKey: ["posts", "search", searchParams],
    queryFn: getSearchResult,
    staleTime: 60 * 1000,
    gcTime: 300 * 1000,
  });

  return data?.map((post) => <Post key={post.postId} post={post} />);
}
```

### 쿼리 함수

```tsx
import { QueryFunction } from "@tanstack/query-core";
import { Post } from "@/model/Post";

export const getSearchResult: QueryFunction<
  Post[],
  [_1: string, _2: string, searchParams: { q: string; pf?: string; f?: string }]
> = async ({ queryKey }) => {
  const [_1, _2, searchParams] = queryKey;
  const res = await fetch(
    `http://localhost:9090/api/search/${
      searchParams.q
    }?${searchParams.toString()}`,
    {
      next: {
        tags: ["posts", "search", searchParams.q], // next 캐싱은 string만 가능
      },
      cache: "no-store",
    }
  );

  if (!res.ok) {
    throw new Error("Failed to fetch data");
  }

  return res.json();
};
```

## Infinity Scroll

가져와야 할 정보가 많아 fetch해오는 속도가 느릴 때 사용자 경험을 향상시키기 위해 사용됩니다. 사용자가 페이지를 스크롤할 때 동적으로 정보의 일부를 끊어서를 가져옵니다.

### 트리거

1. scroll event : 스크롤 바의 위치를 계산해 데이터를 패칭해오기
2. intersection observer : viewport에 타겟이 들어오는 순간 데이터를 패칭

### 요청

데이터를 요청할 때는 서버에서 cursor id를 기준으로 값을 읽습니다.
예를 들어 서버로 요청하는 uri가 `/alarms?cursorId={cursorId}&size={size}`이라면 cursorId 0부터 클라이언트가 size 만큼 요청합니다.

### `prefetchInfiniteQuery`

- 서버컴포넌트에서 prefetchInfiniteQuery로 infinite scroll를 구현한다.
- prefetchInfiniteQuery를 사용할 때는 `initialPageParam`(초기 cursor 값)을 넣어주어야 한다.

```tsx
await queryClient.prefetchInfiniteQuery({
  queryKey: ["posts", "recommends"],
  queryFn: getPostRecommends,
  initialPageParam: 0,
});
```

### `useInfiniteQuery`

- `useInfiniteQuery`를 사용할 때는 `initialPageParam`(초기 cursor 값)을 넣어주어야 한다.
- `useInfiniteQuery`를 사용할 때 다음 페이지의 커서 아이디를 가져오는 콜백함수를 `getNextPageParam`에 넣어주어야 한다.

```tsx
const {
  data,
  fetchNextPage, // 옵저버에 닿으면 fetch 해온다
  hasNextPage, // 더이상 불러올 페이지가 없는 경우
  isFetching, // 쿼리를 호출할 때마다 true가 됨
  isPending, // 처음, 데이터를 가져오지 않았을 때 true
  isLoading, // isPending && isFetching
  isError,
} = useInfiniteQuery<
  IPost[],
  Object,
  InfiniteData<IPost[]>,
  [_1: string, _2: string],
  number
>({
  //InfiniteData<IPost[]>는 infiniteQuery용 데이터 타입 정의
  queryKey: ["posts", "recommends"],
  queryFn: getPostRecommends,
  initialPageParam: 0,
  getNextPageParam: (lastPage) => lastPage.at(-1)?.postId, // 서버에서 데이터를 가져올 때 숫자의 배수로 가져올 경우 중간에 삭제된 데이터가 존재할 수 있으므로 최근에 가져온 페이지의 가장 마지막 id를 제출
  staleTime: 60 * 1000, // fresh -> stale, 5분이라는 기준
  gcTime: 300 * 1000,
});
```

### `react-intersection-observer`

`react-intersection-observer`로 새로 데이터를 패칭해오는 트리거를 구현할 수 있습니다.

```tsx
"use client"

import {useInView} from "react-intersection-observer";

export default function PostRecommends() {

...

  const { ref, inView } = useInView({
    threshold: 0,
    delay: 0,
  });

  useEffect(() => {
    if (inView) { // 화면에 ref가 보여서 inView가 true가 될 때마다 다음 페이지를 패칭
      !isFetching && hasNextPage && fetchNextPage();
    }
  }, [inView, isFetching, hasNextPage, fetchNextPage]);

  if (isError) {
    return '에러 처리해줘';
  }

  return (
    <>
      {data?.pages.map((page, i) => (
        <Fragment key={i}>
          {page.map((post) => <Post key={post.postId} post={post}/>)}
        </Fragment>))}
      <div ref={ref} style={{height: 50}}/>
    </>
  )
}
```

페이지 간 이동에는 스크롤 저장이 되지 않고, 같은 컴포넌트에서는 스크롤 보관이 됩니다.

```tsx
💡 Tips
fragment에 prop을 넣어주고 싶을 때는 `<Fragment key={_}>`를 사용하면 된다.
```

# next에서 에러와 로딩 다루기

`loading.tsx`->`page.tsx`의 과정을 거치고 중간에 에러가 발생하면 `error.tsx`를 보여줍니다.
리액트의 suspense와 errorBoundary를 활용한 개념입니다.

## 로딩

![image](https://nextjs.org/_next/image?url=%2Fdocs%2Flight%2Floading-overview.png&w=3840&q=75&dpl=dpl_y7QdznKAQ3dHs2QTSh3XNu44H11u)

### Suspense Componenet를 활성화시키는 상황

1. 서버에서 데이터 패치해올 떄, 서버 컴포넌트를 가져올 때
2. Lazy Loading을 하는 경우(ex. code spliting)
3. use라는 훅으로 값을 가져오는 경우
   - use 키워드 하나로 사용
   - 내부에 context나 promise를 넣는다.
     - context를 넣는 경우 useContext와 유사한 효과
     - promise를 넣는 경우 resolve될 때까지 기다준다.
   - 훅이나 컴포넌트 내부 뿐만 아니라 if문 안에도 들어갈 수 있다(일반 함수 안에서는 사용 불가).

넥스트에서 처음 화면 렌더링을 할 때 서버 사이트 렌더링을 수행하기 때문에 로딩이 뜨지 않습니다(페이지에서 새로고침을 해도 로딩x).

다른 페이지로 이동하는 경우, 이동하려는 페이지를 클라이언트에서 다시 로딩하기 때문에 `loading.tsx`가 보이게 됩니다.

### 전체 페이지를 서버 사이드 렌더링 하면...

![image](https://nextjs.org/_next/image?url=%2Fdocs%2Flight%2Fserver-rendering-without-streaming-chart.png&w=3840&q=75&dpl=dpl_y7QdznKAQ3dHs2QTSh3XNu44H11u)

A,B - 프리패치로 데이터를 넣은 상태로 ssr수행

C, D - 클라이언트 측에서 하이드레이션 및 렌더링 수행

### 서스펜스 사용 시 특정 부분만 렌더링 가능

![image](https://relic-close-429.notion.site/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F1f026584-1df1-470f-8187-07c35af76396%2F20f7afde-2a68-453c-aad6-defd9b74861f%2FUntitled.png?table=block&id=8b66fde4-9909-487c-97f5-f6f09aad6e0d&spaceId=1f026584-1df1-470f-8187-07c35af76396&width=2000&userId=&cache=v2)

- 로딩이 필요한 부분은 suspense로 분리할 수 있다.
- layout은 next에서 자체적으로 먼저 보내준다.
- 페이지에서 로딩이 필요한 부분(ex 데이터 패칭해오는 부분)만 나중에 받을 수 있도록 `suspense`로 감싸서 처리한다.

### 정리

- page의 로딩은 `loading.tsx`에서 자체적으로 suspense 처리
- 서버 컴포넌트에서 페이지 전체의 렌더링보다 데이터 패칭등이 필요한 일부 컴포넌트의 로딩 제어가 필요한 경우 `<Suspense fallback={<loading/>}>`로 감싸서 처리
- 클라이언트 컴포넌트에서 리액트 쿼리를 이용할 경우 isPending이나 isLoading을 이용해 로딩 화면 처리
  - `useSuspenseQuery`, `useSuspenseInfiniteQuery` 사용 시 가장 가까운 `<Suspense>`에서 isPending을 캐치해 fallback을 수행

## 에러

![image](https://nextjs.org/_next/image?url=%2Fdocs%2Flight%2Ferror-overview.png&w=3840&q=75&dpl=dpl_y7QdznKAQ3dHs2QTSh3XNu44H11u)

## Next.js에서 loading과 에러를 동시에 사용하는 경우

폴더 내에 `loading.tsx`와 `error.tsx`가 동시에 존재하는 경우에는 errorBoundary가 suspense를 감싸고 있다.

넥스트에서 알아서 계층 구조를 만들어준다.

```tsx
<ErrorBoundary fallback={<Error />}>
  <Suspense fallBack={<Loading />}>
    <Page />
  </Suspense>
</ErrorBoundary>
```

# 참고

https://tanstack.com/query/latest/docs/react/guides/ssr
