---
title: "MCP(Model Context Protocol)를 사용해보자"
author: Bumgu
date: 2025-03-01
categories: []
tags: 
    - Claude
    - MCP
slug: "try_mcp"
---
2월 27일 열린 Docker Community Day를 다녀왔습니다.
연사자분중 당근마켓 SRE 염근철님의 발표주제였는데요, 재밌는 주제인것 같아 간단하게 사용을 해보았습니다.

---

#### MCP란?
>Model Context Protocol(MCP)은 LLM(Large Language Model) 애플리케이션과 외부 데이터 소스 및 도구들 간의 원활한 통합을 가능하게 하는 개방형 프로토콜입니다. AI 기반 IDE를 구축하든, 채팅 인터페이스를 개선하든, 혹은 커스텀 AI 워크플로우를 만들든 관계없이, MCP는 LLM이 필요로 하는 컨텍스트와 연결하기 위한 표준화된 방법을 제공합니다.

#### MCP 구조
MCP follows a client-server architecture where:
* Hosts: LLM 애플리케이션 (Claude 데스크탑 혹은 IDE)
* Clients: 호스트 애플리케이션 안에서 서버와 1:1 연결을 유지합니다.
* Servers: 클라이언트에게 context, tools, prompts를 제공합니다.

![](/images/post/20-MCP/1.png)

설명을 봐서는 무슨말인지 감이 오지 않습니다..
사용을해보고 확인해보겠습니다.


### 1. Claude 한테 질문을 해보자
우선 Claude Desktop 애플리케이션을 설치하고, 간단한 부탁을 해보겠습니다.
![](/images/post/20-MCP/2.png)

제 깃허브에 접근해서 스크린샷 하나 찍어달라고 했는데, 외부 웹사이트에 접근할 수 없다고 합니다. 생각해보면 당연하죠.
이제 Claude가 브라우저를 사용할 수 있게 MCP서버를 구동해보겠습니다.

#### 1-1. MCP 클론
[MCP 공식 깃허브](https://github.com/modelcontextprotocol/servers/)
를 클론받습니다.

```bash
# 클론받은 디렉토리로 이동
cd servers

# puppeteer 이미지 빌드
docker build -t mcp/puppeteer -f src/puppeteer/Dockerfile .
```
공식 지원 MCP중 하나인 `puppeteer`를 컨테이너 이미지로 빌드합니다.

#### 1-2. 빌드한 이미지를 Claude MCP-Server로 사용
빌드한 MCP-Server를 Claude가 사용할 수 있도록 설정에 넣어줍니다.

* Mac / Linux
`~/Library/Application\ Support/Claude/claude_desktop_config.json`
* Windows
`$env:AppData\Claude\claude_desktop_config.json`

를 열고
```json
{
  "mcpServers": {
    "puppeteer": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "--init", "-e", "DOCKER_CONTAINER=true", "mcp/puppeteer"]
    }
  }
}
```
이렇게 작성해줍니다.

`puppeteer`를 사용할때
아까 빌드한 컨테이너 이미지를 사용해 MCP 서버를 구동 -> Claude가 puppeteer를 사용

이제 Claude는 브라우저(puppeteer)를 사용할 수 있습니다.
한번 테스트 해보겠습니다.

우선 `claude_desktop_config.json`을 수정했으면 Claude Desktop 애플리케이션을 껐다 킵니다.

![](/images/post/20-MCP/3.png)

빨간네모칸 안에 망치버튼이 생겼습니다.

클릭해보면
![](/images/post/20-MCP/4.png)

현재 사용가능한 MCP Tools 목록이 보입니다.

아까 클론받은 폴더에서
`servers/src/puppeteer/index.ts`를 열어보면

```typescript
const TOOLS: Tool[] = [
  {
    name: "puppeteer_navigate",
    description: "Navigate to a URL",
    inputSchema: {
      type: "object",
      properties: {
        url: { type: "string" },
      },
      required: ["url"],
    },
  },
  {
    name: "puppeteer_screenshot",
    description: "Take a screenshot of the current page or a specific element",
    inputSchema: {
      type: "object",
      properties: {
        name: { type: "string", description: "Name for the screenshot" },
        selector: { type: "string", description: "CSS selector for element to screenshot" },
        width: { type: "number", description: "Width in pixels (default: 800)" },
        height: { type: "number", description: "Height in pixels (default: 600)" },
      },
      required: ["name"],
    },
  },
  {
    name: "puppeteer_click",
    description: "Click an element on the page",
    inputSchema: {
      type: "object",
      properties: {
        selector: { type: "string", description: "CSS selector for element to click" },
      },
      required: ["selector"],
    },
  },
  {
    name: "puppeteer_fill",
    description: "Fill out an input field",
    inputSchema: {
      type: "object",
      properties: {
        selector: { type: "string", description: "CSS selector for input field" },
        value: { type: "string", description: "Value to fill" },
      },
      required: ["selector", "value"],
    },
  },
  {
    name: "puppeteer_select",
    description: "Select an element on the page with Select tag",
    inputSchema: {
      type: "object",
      properties: {
        selector: { type: "string", description: "CSS selector for element to select" },
        value: { type: "string", description: "Value to select" },
      },
      required: ["selector", "value"],
    },
  },
  {
    name: "puppeteer_hover",
    description: "Hover an element on the page",
    inputSchema: {
      type: "object",
      properties: {
        selector: { type: "string", description: "CSS selector for element to hover" },
      },
      required: ["selector"],
    },
  },
  {
    name: "puppeteer_evaluate",
    description: "Execute JavaScript in the browser console",
    inputSchema: {
      type: "object",
      properties: {
        script: { type: "string", description: "JavaScript code to execute" },
      },
      required: ["script"],
    },
  },
];
```
이렇게 정의되어있는 Tool들이 MCP로 인해 Claude가 사용할 수 있게 되는겁니다.
이제 같은 질문을 다시 해보겠습니다!


#### 1-3. MCP를 사용해 Claude에게 다시 질문

![](/images/post/20-MCP/5.png)

같은 질문을 했더니 Github에 접근하겠다고 하면서 권한을 물어봅니다. `Allow for This Chat`을 클릭해서 허용해주도록 하겠습니다.

![](/images/post/20-MCP/6.png)

Claude는 `puppeteer_navigate`, `puppeteer_screenshot`를 사용하여
깃허브 접근 -> 스크린샷을 했습니다.

하나만 더 시켜보겠습니다. 제 레포지토리 목록중 `Go`언어 레포지터리만 골라오라고 시켜보겠습니다.

```
Let me check the repositories and find the ones written in Go language.

Let me try a different selector for the Repositories tab:

Now let me take a screenshot of the repositories page to identify Go repositories:

I can see one Go repository on this page: "minio-cleanup" (marked with the Go language tag). Let me scroll down to check if there are more Go repositories:

I can see more Go repositories in this view. Let me continue scrolling to check if there are additional ones:

Let me continue scrolling to see if there are more repositories:

Let me continue checking for more repositories:

Let me continue scrolling to check for more repositories:

Let me look at the final page of repositories to make sure I find all Go repositories:


I'll check if there are more pages of repositories by clicking the "Next" button:

Let me continue checking for more Go repositories:

Let me check for more pages:

Let me try another selector for the next page:

Let me try one more approach:

Based on the screenshots I've gathered, I can now provide a list of all the Go language repositories from bonzonkim's GitHub profile:

1. **minio-cleanup** - Updated last week
2. **url-shortener** - Updated last month
3. **go-crud** - Updated on Jan 19
4. **spotify-go** - Updated on Jan 1
5. **distributed-minio-logs** - Updated on Dec 21, 2024
6. **jaeger-trace** - Updated on Dec 5, 2024

These are all the repositories written in Go language that are visible in bonzonkim's GitHub profile. Each one is marked with the Go language tag (blue dot with "Go" label).
```
![](/images/post/20-MCP/7.png)

~~소름..~~
정확하게 가져왔습니다.

Claude의 대답 내용을 보면, 처음에 
![](/images/post/20-MCP/8.png)


1. 처음 클릭을 하려고 했지만 잘 안됐는지 다른 Selector를 선택해본다고 합니다.
이후 Repository 탭에서 스크린샷을 한장 찍고
![](/images/post/20-MCP/9.png)

2. 현재 페이지에서는 Go 레포지토리가 하나밖에 안 보여서 스크롤을 내려보겠다고 합니다.


3. 스크롤을 내리다가 맨 밑에 내려오니까, Next Page 버튼이 있는지 확인하고 Next Page 버튼을 클릭합니다.
![](/images/post/20-MCP/10.png)

결과:

![](/images/post/20-MCP/11.png)

다시 페이지 맨 밑에 도달한후, Next Page가 있는지 확인하고, Next Page가 없으니 끝이라고 판단하고 지금까지 확인한 레포지토리들의 목록을 정리해서 알려줍니다.

신기했던 점은, 이 글을 작성하기전 테스트해봤을때는
**지금과 다른 방식으로 목록을 찾았습니다**
테스트할때는 깃허브의 Repositories 탭에서 Language Drop down을 선택하고, Go를 선택하고 목록을 만들었습니다.

이로 미루어 보았을때, LLM(Claude)는 구동한 MCP 서버에 정의된 도구를 이용해 **직접 생각하고** 일을 처리하고 답변을 줍니다.

### 2. Postgres에 써보자!
#### 2-1. Postgres MCP 이미지 빌드
아까 클론받은 `servers`경로에서

`docker build -t mcp/postgres -f src/postgres/Dockerfile . `


빌드가 완료되었다면 `puppeteer`를 작성했던것처럼 postgres mcp도 `claude_desktop_config.json`에 작성합니다.

```json
{
  "mcpServers": {
    "puppeteer": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "--init", "-e", "DOCKER_CONTAINER=true", "mcp/puppeteer"]
    },
    "postgres": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "--init", "-e", "DOCKER_CONTAINER=true", "mcp/postgres"]
    }
  }
}
```
`docker run -d -e POSTGRES_PASSWORD=1234 -p5432:5432 --rm --name testpostgres postgres:latest` 명령어로 postgres 하나 후다닥 띄웁니다.

```bash
create database test;

\c test

CREATE TABLE users (
     id SERIAL PRIMARY KEY,
     name VARCHAR(100) NOT NULL,
     password VARCHAR(100) NOT NULL
 );
```
test 데이터베이스를 만들고, user 테이블을 하나 생성합니다.

![](/images/post/20-MCP/12.png)

보다시피 읽기전용 작업을 할 수 있습니다. 테스트 데이터 하나 넣어주고, 바로 질문을 해보도록하죠

```
postgres@127.0.0.1:test> INSERT INTO users (name, password) VALUES
     ('Alice', 'password123'),
     ('Bob', 'securepass'),
     ('Charlie', 'mypassword'),
     ('David', 'pass1234'),
     ('Eve', 'eve_secret');

INSERT 0 5
Time: 0.022s
postgres@127.0.0.1:test> select * from users;
+----+---------+-------------+
| id | name    | password    |
|----+---------+-------------|
| 1  | Alice   | password123 |
| 2  | Bob     | securepass  |
| 3  | Charlie | mypassword  |
| 4  | David   | pass1234    |
| 5  | Eve     | eve_secret  |
+----+---------+-------------+
SELECT 5
Time: 0.014s
```
5개를 넣어줬습니다.

이제 Claude에게 질문을 해봅니다.

![](/images/post/20-MCP/13.png)

이제 Claude는 제 로컬 postgres에 접근해서 쿼리를 날리고, 결과를 저에게 말해줍니다.

---


### 마무리
목요일에 처음듣고 2일뒤인 오늘 사용을 해봤는데, 생각보다 사용법이 간단하고 또한 성능이 확실해서 놀랐습니다.

특히, 자기가 스스로 생각한 후 작동을 하는것이 놀라운 부분이었습니다.

아직은 Claude가 MCP에 대해 제일 많은 기능을 지원하지만, 조만간 다른 LLM애플리케이션도 지원을 하리라 생각합니다.

간단한 MCP를 직접 생성하는 것도 한번해봐야겠네요. 읽어주셔서 감사하고 꼭 한번 사용해보시기 바랍니다.
