---
title: "Applying SSG to a Next.js Project"
author: Bumgu
date: 2024-02-11
categories: []
tags:
    - next.js
    - ssg
    - typescript
slug: "nextjs_ssg"
---
My [website](https://bumgu.com) is built with Next.js.
Although the `Project` and `Career` tabs here are small web pages, I wanted to try pre-rendering them with SSG, so I gave it a try.

![](/images/post/5-nextjs-ssg/1.png)
I'll apply it to this page.

- Development environment
Next.js 13
TypeScript

---
# 1. Existing code



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
              Description...
          </ProjectGridItem>
        </Section>
        <Section>
          <ProjectGridItem title="what candy" thumbnail={thumbnailWhatCandy} gitRepo="what-candy">
              Description...
          </ProjectGridItem>
        </Section>
        <Section>
          <ProjectGridItem title="know your Spotify" thumbnail={thumbnailSpotify} gitRepo="know-your-spotify">
              Description...
          </ProjectGridItem>
        </Section>
      </Layout>
    </Container>
  );
};
```
Previously, I passed prop values to components and rendered them as shown above.

To apply SSG, we need to:
- Define types (TypeScript)
- Prepare the data (prop values and descriptions)
- Fetch the data in the page component with `getStaticProps()`
- Render using the return value of `getStaticProps()`

First, let's prepare the data.

# 2. Define the types
I'll create a `types` folder and define the types in it.
The `ProjectGridItem` component needs these properties:
`title`: Project title
`thumbnail`: Project thumbnail
`description`: Project description (below the thumbnail)
`gitRepo`: The GitHub address to navigate to when clicked

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
There is also a Career page, but it doesn't need the `gitRepo` property, so I defined the `WorkGridItemProps` type by declaring and extending `GridItemProps`.


# 3. Prepare the data
I'll separate the values that were originally placed on the `Project` page and manage them as an array.

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
    description: 'Description...'
  },
  {
    title: 'what candy',
    thumbnail: thumbnailWhatCandy,
    gitRepo: 'what-candy',
    description: 'Description...'
  },
  {
    title: 'know your spotify',
    thumbnail: thumbnailSpotify,
    gitRepo: 'know-your-spotify',
    description: 'Description...'
  },
];
```
Import the `WorkGridItemProps` type defined in step 2 and declare that this is an array of `WorkGridItemProps`.

# 4. Fetch data with `getStaticProps()`
Now that the type is defined and the data is ready, let's use `getStaticProps()` to retrieve the data.

```ts
import { projectGridItemData } from '../data/projects/projectData';
import { WorkGridItemProps } from '../types/types';

export const getStaticProps: GetStaticProps = () => {
  return {
    props: { projectGridItemData },
  };
};
```
There's no need to communicate using `fetch` or `axios`; since we're retrieving static data from a folder, we import it and pass it as the return value of `getStaticProps()`. This runs `getStaticProps()` at build time.


# 5. Render using the return value of `getStaticProps()`
Now that we've received the data as props, we can render it.

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
Since the data is declared as an array of objects, I used `Array.map()` to render each project and reduce repeated component markup.
The Project page now has SSG applied using `getStaticProps()`.

# 6. Comparing the code before and after
#### Before
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
            Description...
          </ProjectGridItem>
        </Section>
        <Section>
          <ProjectGridItem title="what candy" thumbnail={thumbnailWhatCandy} gitRepo="what-candy">
            Description...
          </ProjectGridItem>
        </Section>
        <Section>
          <ProjectGridItem title="know your Spotify" thumbnail={thumbnailSpotify} gitRepo="know-your-spotify">
            Description...
          </ProjectGridItem>
        </Section>
      </Layout>
    </Container>
  );
};
export default Projects;
```
#### After
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


# Closing thoughts
In my case, I only have three components to place on the page, but if there were 10 or 20, there would be a lot of repeated code.
SSG provides pre-rendered HTML for fast initial loading and SEO, while keeping the data in an array lets us replace repeated component markup with a single `map()` call.
That's all for today's post. Thank you.
