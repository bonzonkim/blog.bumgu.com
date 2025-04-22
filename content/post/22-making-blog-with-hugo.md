---
title: "Hugo를 사용해 블로그 만들기"
author: Bumgu
date: 2025-04-22T20:42:28+09:00
categories: []
tags: 
    - Hugo
    - Github
slug: "making_blog_with_hugo"
---

마크다운이 익숙해서 [Velog](https://velog.io/@kellyb9/posts)를 블로그로 사용하다가 **Hugo + GitHub Pages** 사용해 블로그를 호스팅했습니다.  
배포했던 과정을 글로 남깁니다.


# 1. Hugo?
[Hugo](https://gohugo.io/)는 Go로 작성된 오픈소스 SSG(Static Site Generator) 입니다.  
JS/TS 기반의 [Astro](https://astro.build/)를 사용할까 하다가 가벼운 Hugo를 선택했습니다.  
Hugo는 내부적으로 `theme-override`를 사용해 css와 html 커스텀이 가능하고, `GoTemplate`를 사용해 `if`문과 `for`문같은 구현이 가능합니다.


# 2. Hugo 설치
[설치 공식문서](https://gohugo.io/installation/)  
* mac: `brew install hugo`
* Linux: `sudo snap install hugo`
* Windows  
    * chocolately: `choco install hugo-extended`  
    * scoop: `scoop install hugo-extended`
    * winget: `winget install Hugo.Hugo.Extended`  

Go로 설치도 가능합니다.
* Go: `go install github.com/gohugoio/hugo@latest`

# 3. 프로젝트 생성
설치가 끝나면 프로젝트를 생성합니다.  
`hugo new site <프로젝트명>`  
프로젝트가 생성되면 기본 구조는 이렇습니다.

```sh
.
├── archetypes # 새로운 포스트를 작성할때 지정해서 사용할 수 있는 템플릿
├── assets # Hugo Pipes를 통한 리소스 처리(SCSS, JS 등)
├── content # 포스트가 저장되는 경로
├── data # 사이트에서 사용할 구조화된 데이터 (YAML, TOML, JSON)
├── hugo.toml # 설정파일
├── i18n # internalization, 다국어 문자열 저장
├── layouts # 레이아웃 커스텀
├── static # css, images
└── themes # 사용할 Hugo 테마
```

# 4. Hugo 테마
## 4-1 테마 찾기
[Themes](https://themes.gohugo.io/)에서 맘에 드는 테마를 하나 고릅니다.  
저는 [Hugo-Classic](https://themes.gohugo.io/themes/hugo-classic/)을 사용했습니다.  
친구들이 별로라했지만 미니멀한것도 마음에 들고 테마 Author의 감성도 마음에 들었기 때문에 사용했습니다.  

## 4-2 테마 적용
**테마는 각 테마마다 조금씩 다릅니다** 이글은 `Hugo-Classic`기준입니다. 각 테마 깃허브를 참고하세요  
1. 마음에드는 테마를 찾았으면 테마 git저장소를 아까 생성한 프로젝트의 `themes/`경로에 클론받습니다.  
2. `themes/클론받은테마/exampleSite/. ../`명령어를 실행합니다.
3. `cd .. && hugo server` 를 실행하면 `localhost:1313`으로 접근이 가능합니다.

# 5. 사용법
* 글(콘텐츠) 생성  
`hugo new post/good-to-greet.md`: `content/post`경로에 `goot-to-greet.md`라는 파일이 생깁니다. 이 파일에 글을 쓰면 사이트에도 나타납니다.
* layout 커스텀  
Hugo는 `theme-override`를 사용해 커스텀을 지원합니다. `프로젝트/layout/partials/header.html`을 생성해 커스텀하면 됩니다.  
* css 커스텀  
`프로젝트/css/theme-override.css`를 사용해 커스텀하면 됩니다.  
* 라우팅 추가  
`hugo.toml`의  
```toml
[[menu.main]]
name = "Posts"
url = "/post/"
[[menu.main]]
name = "someRoute"
url = "/some/"
```
를 추가하면 라우팅을 추가할 수 있습니다.


# 6. 배포 (Github Pages)
만든 프로젝트를 깃허브 저장소로 init하고 원격에 푸쉬합니다.  
푸쉬가 되었다면 원격 저장소에서  
settings -> pages 로 들어가 Build and deployment의 Source 드랍다운에서 `Github Actions`를 선택합니다.  
그럼 CI Workflow를 생성하는 창이 나오는데,  아래의 Workflow를 사용했습니다.
```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches:
      - main
  # Actions 탭에서 수동 실행 허용
  workflow_dispatch:

# GitHub Pages에 배포할 수 있도록 권한 설정
permissions:
  contents: read
  pages: write
  id-token: write

# 한 번에 하나의 배포만 허용하며, 실행 중인 작업과 최신 대기 작업 사이에 있는 작업은 건너뜀
# 단, 실행 중인 작업은 취소하지 않음 (배포 완료를 허용하기 위함)
concurrency:
  group: "pages"
  cancel-in-progress: false

# 기본 셸을 bash로 설정
defaults:
  run:
    shell: bash

jobs:
  # 빌드 작업
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.140.1
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb
      - name: Install Dart Sass
        run: sudo snap install dart-sass
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Install Node.js
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"
      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
          TZ: Asia/Seoul
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  # 배포 작업
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```
깃허브 저장소 main브랜치에 푸쉬하면 빌드를 거쳐 Github Pages에 배포됩니다.

# 7. 도메인 연결
Github Pages로 배포하면, `<github username>.github.io`형식의 도메인으로 배포가 됩니다.  
저는 도메인 DNS를 Cloudflare로 관리하기 때문에, Cloudflare에서 DNS CNAME 레코드를 생성해줬습니다.  
제 도메인은 `bumgu.com`이고, 블로그 도메인은 `blog.bumgu.com`으로 하고싶으니,
```
Type: CNAME  
Name: blog  
Content: <github username>.github.io
```
위와 같이 DNS 레코드를 생성하고, [doggo](https://github.com/mr-karan/doggo)로 DNS조회 한번 때려줍니다.  
```sh
➜ doggo blog.bumgu.com
NAME                    TYPE    CLASS   TTL     ADDRESS                 NAMESERVER
blog.bumgu.com.         CNAME   IN      217s    bonzonkim.github.io.    8.8.8.8:53
bonzonkim.github.io.    A       IN      3517s   185.199.109.153         8.8.8.8:53
bonzonkim.github.io.    A       IN      3517s   185.199.110.153         8.8.8.8:53
bonzonkim.github.io.    A       IN      3517s   185.199.108.153         8.8.8.8:53
bonzonkim.github.io.    A       IN      3517s   185.199.111.153         8.8.8.8:53
blog.bumgu.com.         CNAME   IN      300s    bonzonkim.github.io.    8.8.4.4:53
bonzonkim.github.io.    A       IN      3600s   185.199.111.153         8.8.4.4:53
bonzonkim.github.io.    A       IN      3600s   185.199.110.153         8.8.4.4:53
bonzonkim.github.io.    A       IN      3600s   185.199.108.153         8.8.4.4:53
bonzonkim.github.io.    A       IN      3600s   185.199.109.153         8.8.4.4:53
```
정상적으로 나오면 준비끝 입니다.
# 8. 완성
모든 준비가 끝났으니 Github원격 저장소에 푸쉬합니다.  
푸쉬하고, CI workflow가 끝나길 기다리면...  

![Image](/images/post/22-making-blog-with-hugo/1.png)

간단하게 블로그 배포가 완료되었습니다.  
포스트 작성이 마크다운 기반이며 가볍고 빠른게 큰 장점인것 같습니다.

감사합니다.

---
### 참고한글
[Create a static blog with Hugo and GitHub Pages](https://www.testingwithmarie.com/posts/20241126-create-a-static-blog-with-hugo/)
