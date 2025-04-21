---
title: "CSR, SSR의 데이터 패칭"
author: Bumgu
date: 2024-01-26
categories: []
tags:
    - CSR
    - SSR
    - next.js
slug: csr_ssr_data_fetching
---
개발하다보면 만나게 되는 `CSR`, `SSR` 이 두가지의 방식으로 next.js에서 get요청으로 json 데이터를 받아오는걸 해보겠습니다.


# 1. CSR, SSR의 차이

#### CSR (Client-Side Rendering)
- 개념
웹의 초기 로딩시, 서버는 HTML과 기본적인 자바스크립트 코드만 클라이언트에게 보내며, 실제로 보이는 내용은 비어있는 상태.

- 동작 방식
클라이언트에서 자바스크립트 코드가 실행되면, 클라이언트는 서버에 필요한 데이터를 요청해 받아온 후, 그 데이터를 이용해 동적으로 화면을 구성하고 렌더링함.


#### SSR(Server-Side Rendering)
- 개념
웹의 초기 로딩시, 서버에서 초기 HTML을 완성하여 클라이언트에게 전달하므로, 초기 로딩 시 이미 완성된 HTML이 화면에 나타남.

- 동작 방식
클라이언트가 페이지에 접속할 때, 서버는 요청된 페이지에 필요한 데이터를 가져와 HTML을 완성한뒤, 클라이언트에게 전달.

#### 요약
CSR : 빈 방(브라우저)에 가구(데이터)를 직접 배치 가구(데이터)를 가져오기 위한 추가 작업이 필요함
SSR : 이미 가구(데이터)가 배치된 방을 제공하므로 초기 로딩 시 빈 화면이 나타나지 않음


#### 언제 쓸까?
##### SSR
- 검색 엔진 최적화 (SEO): SSR은 초기 로딩 시에 완성된 HTML을 제공하므로 검색 엔진이 페이지의 콘텐츠를 쉽게 이해하고 인덱싱할 수 있다.

- 초기 로딩 성능: SSR은 초기 페이지 로딩 시에 필요한 내용이 이미 포함되어 있으므로 사용자가 빠르게 콘텐츠를 볼 수 있다.

- 소셜 미디어 공유: 완성된 HTML을 제공하므로 소셜 미디어에서 페이지를 공유할 때 미리보기가 적절하게 생성됨.

- 단일 페이지 애플리케이션(SPA)이 필요하지 않은 경우: 만약 애플리케이션이 복잡하지 않고 여러 페이지로 구성되어 있을 경우, SSR을 통해 간단한 페이지들을 빠르게 서빙할 수 있음.

##### CSR

- 높은 상호작용성이 필요한 단일 페이지 애플리케이션(SPA): 사용자와의 상호작용이 많이 필요한 경우, CSR이 더 적합할 수 있다. 예를 들어 현재 페이지에서 동적으로 데이터를 로드하거나 변경해야 하는 경우가 있음.

- 애플리케이션 초기 로딩 이후에도 빠른 성능이 요구되는 경우: SPA에서는 초기 로딩 이후에는 서버에서 필요한 데이터만 요청하여 동적으로 업데이트할 수 있기 때문에 사용자 경험이 부드럽게 유지됨.

- 서버 부하 감소가 중요한 경우: CSR은 초기 로딩 시에 서버 부하가 낮아지므로, 초기 로딩 후에는 클라이언트에서 필요한 데이터를 비동기적으로 요청하므로 서버 부하를 분산할 수 있음.

- 풍부한 사용자 경험을 제공하고자 하는 경우: CSR을 통해 페이지 간의 전환이 부드럽게 이루어지며, 화면 갱신이 더 빠르게 이루어질 수 있어 풍부한 사용자 경험을 제공할 수 있음.


# 2. 앱 생성
먼저 next앱을 생성하겠습니다.
`yarn create next-app next-data-fetching`
![](/images/post/2-CSR_SSR_data_fetching/1.png)

데이터 패칭만 할것이기 때문에 `ESLint`나 `TailwindCSS`는 포함하지 않았습니다.

이동 `cd next-data-fetching`
실행 `yarn dev`
![](/images/post/2-CSR_SSR_data_fetching/2.png)
next앱의 기본 화면입니다.




# 3. Data fetching - CSR
get 요청할 주소 https://nomad-movies.nomadcoders.workers.dev/movies


`app/page.tsx`의 내용을 전부 지워주고 `useState`와 `useEffect`를 임포트합니다.
```ts
import { useState, useEffect } from 'react';

export default function Home() {
  const [movies, setMovies] = useState([]);
  return (
  <div>
  </div>
  );
}
```

### 3-1. "use client"
Next.js의 app은 기본적으로 `Server Component`기 때문에 `useState`, `useEffect`를 사용하지 못합니다.
그래서 맨 위에 `"use client"`를 적어줌으로써 이 컴포넌트는 클라이언트 컴포넌트라는 것을 선언합니다.

```ts
"use client"
import { useState, useEffect } from 'react';

export default function Home() {
  const [movies, setMovies] = useState([]);
  return (
  <div>
  </div>
  );
}
```

### 3-2. getMovies()
이후 `fetch`를 사용해 데이터를 받아오겠습니다.
`getMovies()`라는 함수를 생성합니다.

```ts
const getMovies = async () => {
	const response = await fetch("https://nomad-movies.nomadcoders.workers.dev/movies");
    const json = await response.json();
    setMovies(json);
    return json
}
```
`getMovies()`는 비동기 함수로 지정된 URL에 요청을 보내고 받아온 응답을 json으로 바꾸고 `setMovies()`를 통해 `movies`를 세팅합니다. 그리고 그 json을 반환합니다.

### 3-3. useEffect()
```ts
useEffect(() => {
	getMovies();
}, [])
```
`getMovies()`는 초기 로딩시 한번만 렌더링할 것이기 때문에 `useEffect()`안에서 호출하고, 의존배열은 빈칸으로 둡니다.

이제 데이터 준비는 다됐으니 화면에 띄우도록 하겠습니다.

```ts
return (
<div>
{JSON.stringify(movies)}
</div
)
```
json 형태의 데이터이기 때문에 `stringify()`로 문자열로 바꿔줍니다.

```ts
"use client"
import { useState, useEffect } from 'react';

export default function Home() {
  const [movies, setMovies] = useState([]);

  const getMovies = async () => {
    const response = await fetch("https://nomad-movies.nomadcoders.workers.dev/movies");
    const json = await response.json();
    setMovies(json)
    return json;
  }

  useEffect(() => {
    getMovies();
  }, [])
  return (
  <div>
     {JSON.stringify(movies)}
    </div>
  );
}
```
최종 코드의 모습은 이렇습니다.

![](/images/post/2-CSR_SSR_data_fetching/csr결과.png)

받아온 데이터를 잘 보여줍니다.
하지만 위에서 설명했듯이, CSR방식은 데이터를 초기 로딩후 가져옵니다. 그렇다면 정말로 데이터 가져오는데 시간이 걸릴까요? 한번 확인해보겠습니다.

### 3-4. isLoading 만들기

로딩중이라면 `Loading...` 이란 문구를 띄워주고 데이터를 다 가져오면 데이터를 보여주도록 하겠습니다.
먼저 `isLoading`, `setIsLoading`을 만들어줍니다.
`const [isLoading, setIsLoading] = useState(true);`

이후 `getMovies()`함수안에 `setMovies(json)` 밑에 `setIsLoading(false)`를 적어줍니다.
그리고 `return`문안에 삼항연산자를 이용해 렌더링합니다.
`{isLoading ? 'Loading.....' : JSON.stringify(movies)}`

```ts
"use client"
import { useState, useEffect } from 'react';

export default function Home() {
  const [movies, setMovies] = useState([]);
  const [isLoading, setIsLoading] = useState(true);

  const getMovies = async () => {
    const response = await fetch("https://nomad-movies.nomadcoders.workers.dev/movies");
    const json = await response.json();
    setMovies(json);
    setIsLoading(false);
    return json;
  }

  useEffect(() => {
    getMovies();
  }, [])
  return (
  <div>
     {isLoading ? 'Loading.....' : JSON.stringify(movies)}
    </div>
  );
}
```
데이터 패칭이 완료되면 `IsLoading`은 `false`가 되어 `movies`가 렌더링되고 패칭되기전에는 `Loading.....`이란 문구가 렌더링됩니다.

![](/images/post/2-CSR_SSR_data_fetching/csr결과.gif)

많은 양의 데이터가 아니기 때문에 오래걸리진 않지만, 페이지가 나타나고 그 뒤에 데이터를 가져오기 때문에 로딩하는 시간이 잠시 있습니다.


# 4. Data fetching - SSR
이제 SSR방식으로 데이터 패칭을 해보겠습니다.

다시 초기 상태로 코드를 되돌립니다.

```ts
export default function Home() {
  return (
  <div>
      {JSON.stringify(movies)}
    </div>
  );
}
```
`JSON.stringify(movies)`부분은 그대로 사용할 것이라 그대로 뒀습니다.

### 4-1. getMovies()

CSR과 같이 `getMovies()`비동기 함수를 작성합니다.

```ts
const getMovies = async () => {
  const response = await fetch("https://nomad-movies.nomadcoders.workers.dev/movies");
  const json = await response.json();
  return json;
}
```
데이터를 패칭하는것 자체는 CSR방식과 다른것은 없습니다.
차이점은 **렌더링 하는 방식**이 다릅니다.

### 4-2. 데이터 렌더링

```ts
export default async function Home() {
  const movies = await getMovies();
  return (
  <div>
      {JSON.stringify(movies)}
    </div>
  );
}
```
`Home`컴포넌트를 비동기 함수로 바꿔주고
`movies`라는 변수에 `getMovies();`의 반환값을 담아줍니다. `getMovies()`는 비동기 함수이기 때문에 `await`키워드가 있어야합니다.

최종 코드
```ts
const getMovies = async () => {
  const response = await fetch("https://nomad-movies.nomadcoders.workers.dev/movies");
  const json = await response.json();
  return json;
}
export default async function Home() {
  const movies = await getMovies();
  return (
  <div>
      {JSON.stringify(movies)}
    </div>
  );
}
```




# 5. 결과 차이
- CSR
![](/images/post/2-CSR_SSR_data_fetching/4.gif)

- SSR
![](/images/post/2-CSR_SSR_data_fetching/5.gif)

CSR은 초기 로딩이후 데이터를 패칭해오고 보여주기 때문에 잠깐의 로딩시간이 있는반면,
SSR은 데이터를 미리 캐싱해놓기 때문에 기다림 없이 바로 데이터를 보여주고 있습니다.

하지만 SSR이 만능 무조건 좋다 이것은 아니기 때문에 상황에 맞게 사용하면 될것 같습니다.

이상으로 포스팅을 마치겠습니다. 감사합니다.
