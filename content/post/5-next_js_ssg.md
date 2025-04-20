---
title: "Next.js 프로젝트에 SSG 적용하기"
author: Bumgu
date: 2024-02-11
categories: []
tags:
    - next.js
    - ssg
    - typescript
slug: "nextjs_ssg"
---
제 [홈페이지](https://bumgu.com)는 next.js로 만들어져 있습니다.
여기의 `Project`탭과 `Career`탭을 작은 웹 페이지지만 SSG로 pre-rendering해보고 싶어서 하게 되었습니다.

![](/images/post/5-nextjs-ssg/1.png)
이 페이지에 적용 해보겠습니다.

- 개발환경
Next.js 13
Typescript

---
# 1. 기존의 코드



```ts
import thumbnailLively from '../public/images/works/Lively_main.png';
import thumbnailWhatCandy from '../public/images/works/whatCandy.png';
import thumbnailSpotify from '../public/images/works/know-your-spotify.png';


const Projects = () => {
  return (
    <Container>
      <Heading as="h3" fontSize={20} mb={4}>
        Projects
      </Heading>
      <Layout title="works">
        <Section>
          <ProjectGridItem title="Lively" thumbnail={thumbnailLively} gitRepo="Lively">
			  설명 ~~~...
          </ProjectGridItem>
        </Section>
        <Section>
          <ProjectGridItem title="what candy" thumbnail={thumbnailWhatCandy} gitRepo="what-candy">
			  설명 ~~~...
          </ProjectGridItem>
        </Section>
        <Section>
          <ProjectGridItem title="know your Spotify" thumbnail={thumbnailSpotify} gitRepo="know-your-spotify">
			  설명 ~~~...
          </ProjectGridItem>
        </Section>
      </Layout>
    </Container>
  );
};
```
기존엔 이와 같이 컴포넌트에 props 값을 넘겨주고 렌더링 했습니다.

SSG를 적용하기 위해 해야할 것은
- type 정의 (typescript)
- 데이터 준비하기 (props값, 설명)
- 페이지 컴포넌트에서 `getStaticProps()`로 데이터 패칭
- `getStaticProps()`의 반환값으로 렌더링

입니다. 우선 데이터를 준비하겠습니다.

# 2. 타입 정의
types폴더를 만들어서 그 안에 타입을 정의 하겠습니다.
`ProjectGridItem`컴포넌트에 필요한 속성은
`title` : 프로젝트 제목
`thumbnail` : 프로젝트 썸네일
`description` : 프로젝트 설명 (썸네일밑에)
`gitRepo` : 클릭시 이동할 깃허브 주소

```ts
type GridItemProps = {
  title: string;
  thumbnail: StaticImageData;
  description: string;
};

export type WorkGridItemProps = GridItemProps & {
  gitRepo: string;
};
```

Career페이지도 있지만, Career페이지에는 `gitRepo`속성이 필요하지 않아서 `GridItemProps`를 선언하고 확장하는 방식으로 `WorkGridItemProps`타입을 정의했습니다.


# 3. 데이터 준비
원래 `Project`페이지에 넣어줬던 값들을 따로 빼서 배열의 형태로 관리하겠습니다.

```ts
import { WorkGridItemProps } from '../../types/types';

import thumbnailLively from '../../public/images/works/Lively_main.png';
import thumbnailWhatCandy from '../../public/images/works/whatCandy.png';
import thumbnailSpotify from '../../public/images/works/know-your-spotify.png';

export const projectGridItemData: WorkGridItemProps[] = [
  {
    title: 'Lively',
    thumbnail: thumbnailLively,
    gitRepo: 'Lively',
    description: 설명...
  },
  {
    title: 'what candy',
    thumbnail: thumbnailWhatCandy,
    gitRepo: 'what-candy',
    description: 설명...
  },
  {
    title: 'know your spotify',
    thumbnail: thumbnailSpotify,
    gitRepo: 'know-your-spotify',
    description: 설명...
  },
];
```
2번에서 정의한 `WorkGridItemProps`타입을 `import`하고 `WorkGridItemProps`의 배열 타입임을 선언합니다.

# 4. `getStaticProps()`로 데이터 패칭

타입도 정의했고 데이터도 준비했으니 `getStaticProps()`를 사용해 데이터를 받아오겠습니다.

```ts
import { projectGridItemData } from '../data/projects/projectData';
import { WorkGridItemProps } from '../types/types';

export const getStaticProps: GetStaticProps = () => {
  return {
    props: { projectGridItemData },
  };
};
```
`fetch`나 `axios`를 이용해 통신을 할 일은 없고, 폴더의 정적인 데이터를 가져오기 때문에 `import`를 하고, `getStaticProps()`의 반환값으로 넣어줍니다. 이렇게 하면 빌드시점에 `getStaticProps()`가 실행됩니다.


# 5. `getStaticProps()`의 반환값으로 렌더링

props로 받아왔으니 이제 렌더링 해주면 됩니다.

```ts
const Projects = ({ projectGridItemData }) => {
  return (
    <Container>
      <Heading as="h3" fontSize={20} mb={4}>
        Projects
      </Heading>
      <Layout title="works">
        {projectGridItemData.map((item: WorkGridItemProps, index: number) => (
          <Section key={index}>
            <ProjectGridItem
              title={item.title}
              thumbnail={item.thumbnail}
              gitRepo={item.gitRepo}
              description={item.description}
            />
          </Section>
        ))}
      </Layout>
    </Container>
  );
};
```
데이터를 객체를 담고 있는 배열로 선언했기 때문에 `Array.map()`함수를 사용해 렌더링 해주었습니다.
이제 Project페이지는 `getStaticProps()`를 이용한 SSG적용이 완료되었습니다.

# 6. 코드 전/후 비교
#### 전
```ts
const Projects = () => {
  return (
    <Container>
      <Heading as="h3" fontSize={20} mb={4}>
        Projects
      </Heading>
      <Layout title="works">
        <Section>
          <ProjectGridItem title="Lively" thumbnail={thumbnailLively} gitRepo="Lively">
            설명....
          </ProjectGridItem>
        </Section>
        <Section>
          <ProjectGridItem title="what candy" thumbnail={thumbnailWhatCandy} gitRepo="what-candy">
            설명...
          </ProjectGridItem>
        </Section>
        <Section>
          <ProjectGridItem title="know your Spotify" thumbnail={thumbnailSpotify} gitRepo="know-your-spotify">
            설명...
          </ProjectGridItem>
        </Section>
      </Layout>
    </Container>
  );
};
export default Projects;
```
#### 후
```ts
export const getStaticProps: GetStaticProps = () => {
  return {
    props: { projectGridItemData},
  };
};

const Projects = ({ projectGridItemData }) => {
  return (
    <Container>
      <Heading as="h3" fontSize={20} mb={4}>
        Projects
      </Heading>
      <Layout title="works">
        {projectGridItemData.map((item: WorkGridItemProps, index: number) => (
          <Section key={index}>
            <ProjectGridItem
              title={item.title}
              thumbnail={item.thumbnail}
              gitRepo={item.gitRepo}
              description={item.description}
            />
          </Section>
        ))}
      </Layout>
    </Container>
  );
};

export default Projects;
```


# 마무리
저의 경우에는 페이지에 넣어줄 컴포넌트가 3개밖에 없지만 만약 10개, 20개가 된다면 반복되는 코드가 많아질 것 입니다.
SSG를 적용해 빠른 로딩속도와 SEO, 그리고 데이터를 배열로 받아오기 때문에 `Array.map()`함수를 사용해 한번으로 줄일 수 있었습니다.
오늘의 포스팅은 여기서 마무리 하겠습니다. 감사합니다.
