---
title: "Next.js getStaticProps()에 대해서"
author: Bumgu
date: 2024-01-23
categories: []
tags:
    - "next.js"
    - "pre-rendering"
    - "typescript"
slug: GetStaticProps
---
안녕하세요. Next.js의 특징중 하나인 pre-rendering에 대해 알아보겠습니다.


# 1. pre-rendering
![](/images/post/1-getStaticProps/1.png)

>데이터 불러오기에 대해 이야기하기 전에 Next.js에서 가장 중요한 개념 중 하나에 대해 이야기해 보겠습니다: 프리 렌더링.
기본적으로 Next.js는 모든 페이지를 미리 렌더링합니다. 즉, 클라이언트 측 자바스크립트가 모든 작업을 수행하는 대신 Next.js가 각 페이지의 HTML을 미리 생성합니다. 사전 렌더링은 성능과 SEO를 개선할 수 있습니다.
생성된 각 HTML은 해당 페이지에 필요한 최소한의 자바스크립트 코드와 연결됩니다. 브라우저에서 페이지가 로드되면 해당 JavaScript 코드가 실행되어 페이지가 완전한 대화형 페이지로 만들어집니다. (이 프로세스를 하이드레이션이라고 합니다.)

[공식문서](https://nextjs.org/learn-pages-router/basics/data-fetching/pre-rendering)에 나온 `pre-rendering`에 대한 설명입니다.

CSR(Client Side Rendering) 같은 경우는 자바스크립트 파일을 브라우저에서 해석해 랜더링하는 방식을 사용합니다.

SSR(Server Side Rendering)은 미리 서버에서 HTML파일로 랜더링해 클라이언트로 전송해주는 방식입니다. 이로인해 서버와 통신을 할 때 미리 만들어둔 HTML파일을 전달받으므로 CSR보다 랜더링 속도가 빠릅니다.

### 1-1 pre-rendering의 두가지 방식

![](/images/post/1-getStaticProps/2.png)
>Next.js에는 두 가지 형태의 사전 렌더링이 있습니다: Static Generation(정적 생성 ) Server Side Rendering(서버 사이드 렌더링). 이 두 가지 방식은 페이지의 HTML을 생성하는 시점에 차이가 있습니다.
정적 생성은 빌드 시점에 HTML을 생성하는 사전 렌더링 방식입니다. 이렇게 미리 렌더링된 HTML은 각 요청에서 재사용됩니다.
서버측 렌더링은 각 요청에 대해 HTML을 생성하는 사전 렌더링 방법입니다.

### 1-2 SSG, SSR

Static Generation : getStaticProps()
Server Side Rendering : getServerSideProps()

SSG 는 정적사이트생성, SSR은 서버사이드 렌더링입니다.
둘의 가장 큰 차이점은

**SSG : 빌드시 HTML이 생성됨
SSR : 매 요청 시 HTML이 생성됨**

데이터가 자주 바뀌지 않는다 => SSG 사용
데이터가 자주 바뀐다 => SSR 사용

이라고 볼 수 있습니다. 빌드 시의 데이터로 HTML을 생성하기 때문에 수시로 데이터가 바뀌는 경우에는 SSR보단 SSG가 적합합니다.

그중 이 포스팅은 SSG를 구현해보도록 하겠습니다.

# 2. 적용해보기
markdown을 표시해주는 간단한 블로그를 만들어 보겠습니다.

### 2-1. 앱 생성, 의존성 설치
`yarn create next-app next-ssg`
로 next app을 생성해줍니다.
이후 나오는 옵션은 아래처럼 했습니다.
![](/images/post/1-getStaticProps/3.png)

생성된 앱에 `cd next-ssg`로 들어가고, markdown을 파싱하기 위한 라이브러리를 다운받습니다.
`yarn add gray-matter`


### 2-2. 필요 폴더 생성
현재 폴더 구조입니다.
```sh
.
├── README.md
├── next-env.d.ts
├── next.config.mjs
├── package.json
├── pages
│   ├── _app.tsx
│   ├── _document.tsx
│   ├── api
│   │   └── hello.ts
│   └── index.tsx
├── public
│   ├── favicon.ico
│   ├── next.svg
│   └── vercel.svg
├── styles
│   ├── Home.module.css
│   └── globals.css
├── tsconfig.json
└── yarn.lock
```
여기서 root경로에
1.`lib/posts.ts`를 생성해주고
2.`pages`폴더안에 `posts` 폴더를 생성합니다. 여기에는 markdown파일들이 위치합니다.

최종 폴더구조
```sh
.
├── README.md
├── lib
│   └── posts.ts
├── next-env.d.ts
├── next.config.mjs
├── package.json
├── pages
│   ├── _app.tsx
│   ├── _document.tsx
│   ├── api
│   │   └── hello.ts
│   ├── index.tsx
│   └── posts
├── public
│   ├── favicon.ico
│   ├── next.svg
│   └── vercel.svg
├── styles
│   ├── Home.module.css
│   └── globals.css
├── tsconfig.json
└── yarn.lock
```

# 3. lib/posts.ts 작성


### 3-1. import, interface
```js
import fs from 'fs';
import path from 'path';
import matter from 'gray-matter';

export interface PostsData {
  id: string;
  date: string;
  title: string;
  content: string;
}
```
필요한 모듈들을 import하고 포스트들의 데이터 타입을 정의할 인터페이스를 만듭니다.

### 3-2. 파일 읽기
```ts
// markdown이 위치해 있는 폴더
const postsDirectory = path.join(process.cwd(), 'pages/posts');

export function getSortedPostsData(): PostsData[] {
  const fileNames = fs.readdirSync(postsDirectory);
  const allPostsData: PostsData[] = fileNames.map((fileName) => {
    // '.md'를 지우고 id에 저장 id => 파일 이름
    const id = fileName.replace(/\.md$/, '');

    // id에 담긴 파일을 문자열로 읽습니다.
    const fullPath = path.join(postsDirectory, fileName);
    const fileContents = fs.readFileSync(fullPath, 'utf8');
```
 `getSortedPostsData()`는 `PostsData`타입의 **배열**을 반환하기 때문에 반환값은 `PostsData[]` 입니다.

함수 안에선 파일을 읽기 위해 폴더명을 가져요고, `fs.readdirSync`를 통해 파일의 이름을 얻습니다. 이 파일이름은 확장자명인 `.md`까지 가져오기에
`map`함수안에서 `replace`함수로 `.md`부분을 자르고 `id`라는 변수에 넣어줍니다.

이후 `path.join`으로 `파일위치/파일명` 을 얻고 변수에 담습니다.
드디어 `fs.readFileSync`를 사용해 파일을 문자열로 읽을 수 있습니다.
### 3-3. 파일 파싱
```ts
    // gray-matter 를 이용해 markdown을 파싱합니다.
    const matterResult = matter(fileContents);

    return {
      id,
      date: matterResult.data.date,
      title: matterResult.data.title,
      content: matterResult.content
    }
  })
  ```
`gray-matter`를 사용해 markdown파일을 파싱합니다. 인자로 넣어준 `fileContents`는 위에서 문자열로 읽은 파일 내용입니다.

그후, `id`,`date`,`title`,`content`를 반환합니다. (map함수 끝)

### 3-4. 반환하기
  ```ts
  // 파일의 date값 순서로 정렬합니다.
  return allPostsData.sort((a, b) => {
    if (a.date < b.date) {
      return 1;
    } else {
      return -1;
    }
  })
}
```
`id`,`date`,`title`,`content` 가 들어있는 `PostsData`타입의 배열을 `map`함수로부터 받았습니다.

이제 이걸 날짜순으로 정렬해주고 반환합니다. 이 파일의 끝입니다.

### 3-5. 최종코드
```ts
import fs from 'fs';
import path from 'path';
import matter from 'gray-matter';

export interface PostsData {
  id: string;
  date: string;
  title: string;
  content: string;
}


// markdown이 위치해 있는 폴더
const postsDirectory = path.join(process.cwd(), 'pages/posts');

export function getSortedPostsData(): PostsData[] {
  const fileNames = fs.readdirSync(postsDirectory);
  const allPostsData: PostsData[] = fileNames.map((fileName) => {
    // '.md'를 지우고 id에 저장 id => 파일 이름
    const id = fileName.replace(/\.md$/, '');

    // id에 담긴 파일을 문자열로 읽습니다.
    const fullPath = path.join(postsDirectory, fileName);
    const fileContents = fs.readFileSync(fullPath, 'utf8');

    // gray-matter 를 이용해 markdown을 파싱합니다.
    const matterResult = matter(fileContents);

    return {
      id,
      date: matterResult.data.date,
      title: matterResult.data.title,
      content: matterResult.content
    }
  })

  // 파일의 date값 순서로 정렬합니다.
  return allPostsData.sort((a, b) => {
    if (a.date < b.date) {
      return 1;
    } else {
      return -1;
    }
  })
}
```

# 4. pages/index.tsx 작성
### 4-1. import, interface
인덱스도 import와 인터페이스 생성부터 해주겠습니다.
```ts
import { getSortedPostsData, PostsData } from '../lib/posts';

interface HomeProps {
  allPostsData:  PostsData[];
}
```
`lib/posts.ts`에서 작성한 함수와 인터페이스를 import합니다.
`HomeProps`란 인터페이스는 import한 `PostsData`타입의 배열 타입입니다.


### 4-2. getStaticProps()
```ts
export async function getStaticProps () {
  const allPostsData = getSortedPostsData();

  return {
    props: {
      allPostsData
    }
  }
}

```
오늘의 주제인 `getStaticProps()`입니다.
`allPostsData`에 `lib/posts.ts`에서 작성한 `getSortedPostsData()`의 반환값을 담아줍니다. markdown파일에서 읽어온 정보들이 담긴 객체 형태의 배열이 담겨있습니다.
그리고 그걸 `props`로 반환해줍니다.

### 4-3. Home 컴포넌트
```ts
const Home: React.FC<HomeProps> = ({ allPostsData }) => {
  return (
    <>
      <section>
        <ul>
          {allPostsData.map(({ id, date, title, content}) => (
            <li key={id}>
              <h1>id: {id}</h1>
              <br/>
              <h1>title: {title}</h1>
              <br/>
              <h1>content: {content}</h1>
              <br/>
              <h1>date: {date}</h1>
              <br/>
            </li>
          ))}
        </ul>
      </section>
    </>
  )
}


```
`Home`컴포넌트는 `React.FC<HomeProps>`타입의 함수입니다.
`React.FC`는 리액트의 `Functional Component`이며, 이 함수의 프로퍼티의 타입은 위에서 정의한 인터페이스 `HomeProps`타입입니다.

그리고 props는 위에서 `getStaticProps`로 받아온 `allPostsData`입니다. 이 props에는 `lib/posts.ts`에서부터 넘겨온 `getSortedPostsData()`함수의 반환값, 즉 파일들의 데이터가 날짜순으로 정렬되어 담겨있습니다.

이후 밑에 return문에서 `allPostsData`를 `map`함수를 통해 `id,`,`date`,`title`,`content`를 가져오고,
그 값들을 `<h1>`태그로 보여줍니다.


# 5. 테스트

이제 직접 마크다운 파일을 작성하고 작동하는지 확인해보겠습니다.
`pages/posts`폴더안에 `.md`파일을 생성합니다. 저는 두개 생성하겠습니다.


```md
---
title: 'first-test'
date: '2024-01-23'
---

안녕하세요 첫번째 게시글입니다..
```
```md
---
title: 'second-test'
date: '2024-01-22'
---

안녕하세요 두번째 게시글입니다..
```
비교를 위해 날짜를 1-23, 1-22로 했습니다. 이제 확인해보도록 하겠습니다.

`yarn dev`명령어로 실행합니다.

![결과](/images/post/1-getStaticProps/4.png)


id : 파일명
title : 파일의 맨위에서 설정한 title 값
content : 파일의 내용
date : 파일의 맨위에서 설정한 date 값

가 정상적으로 나오며 날짜순으로 정렬된것을 확인할 수 있습니다.

혹시 정상적으로 되지 않는다면 [제 깃허브](https://github.com/bonzonkim/next-ssg)에도 올렸으니 확인해보면 되겠습니다.

---

그동안 Next.js를 써도 SSG를 제대로 사용해본적이 없었는데, 이번에 사용 해보게 되었습니다.
읽어주셔서 감사하고, 다음에 다른 포스팅을 들고 오겠습니다. 감사합니다.
