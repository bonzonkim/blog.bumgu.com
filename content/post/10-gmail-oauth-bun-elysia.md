---
title: "gmail OAuth2 토큰 발급(bun, elysia)"
author: Bumgu
date: 2024-06-19
categories: 
    - Developer
tags: 
    - bun
    - elysia
    - oauth2
    - gmail
    - typescript
slug: "gmail_oauth_with_bun_elysia"
---
# 1. bun이란?
`bun`은 `node.js`와 같은 자바스크립트 런타임입니다. 자세한 내용과 설치는 [bun.sh](https://bun.sh/) 에서 확인 하실 수 있습니다. `node.js`의 `express`와 같은 `elysia`라는 프레임워크를 사용해서 google gmail OAuth2토큰을 발급 받는 과정까지 포스팅 하도록 하겠습니다.


# 2. elysia 앱 생성
`elysia`앱을 생성해보겠습니다 생성은
`bun create elysia ${app name}` 
명령어를 통해 생성 할 수 있습니다.

![](https://velog.velcdn.com/images/kellyb9/post/6196cf35-57fc-4c79-ab03-cbb5648e4fb8/image.png)

`src/index.ts`파일을 보면

``` typescript
import { Elysia } from "elysia";

const app = new Elysia().get("/", () => "Hello Elysia").listen(3000);

console.log(
  `🦊 Elysia is running at ${app.server?.hostname}:${app.server?.port}`
);
```
3000번포트로 리스닝 중인것을 확인 할 수 있습니다. 
`bun run dev`명령어로 실행하고 `localhost:3000`을 접속해보면
![](https://velog.velcdn.com/images/kellyb9/post/4851b823-393e-42eb-a4bb-2a5f4ea91ab1/image.png)
이와 같이 작동중인것을 확인 할 수 있습니다.

# 3. Google 프로젝트 생성
1. [google developer console](https://console.cloud.google.com/apis/dashboard?project=gmail-ts-425902)에 접속해서 기존 프로젝트에서 api를 추가해주거나 새로운 프로젝트를 생성합니다.


2. 왼쪽탭에서 Library를 클릭하고 gmail API를 검색후 클릭 => ENABLE 선택합니다.
![](/images/post/10-gmail-oauth-bun-elysia/1.png)

3. OAuth consent screen 을 선택후 External타입으로 CREATE 해줍니다.
![](/images/post/10-gmail-oauth-bun-elysia/2.png)
4. 다음 화면에서 App name, user support email, Developer contanct information을 채워준후 save and continue 버튼을 누릅니다.

5. scope를 추가합니다 `https://www.googleapis.com/auth/gmail.readonly` => ADD TO TABLE => UPDATE => Save and continue
![](/images/post/10-gmail-oauth-bun-elysia/3.png)

6. 이후 나오는 화면들은 특별히 건드릴 것은 없습니다.


7. 다음 Credentials 탭에서 CREATE CREDENTIALS => OAuth client ID를 누릅니다
![](/images/post/10-gmail-oauth-bun-elysia/4.png)

 
8. 내용을 채워줍니다
![](/images/post/10-gmail-oauth-bun-elysia/5.png)

현재 `elysia`앱이 3000번포트를 사용하고있기때문에 `localhost:3000` 이고 redirect URI는 아무거나 상관없지만 저는 `/auth`로 했습니다. 이후 CREATE를 눌러줍니다.

9. 누른뒤 나오는 Client ID와 Client Secret을 메모해둡니다.







# 4. 코드 작성
이제 준비가 완료되었습니다. 코드를 작성하러 가보겠습니다.


### 1. 시크릿 준비
`Client ID`와 `Client Secret`는 랜덤한 문자이기 때문에 외워서 칠 수도 없고, 복잡하기 때문에 시크릿 파일을 만들어두고 필요할때 import해서 사용하도록 하겠습니다.

```typescript
export const CLIENT_ID = "your client id"
export const CLIENT_SECRET = "your client secret"
export const SCOPES = ['https://www.googleapis.com/auth/gmail.readonly']
export const REDIRECT_URI = 'http://localhost:3000/auth'
export const AUTH_URI = 'https://accounts.google.com/o/oauth2/auth'
export const TOKEN_URI = 'https://oauth2.googleapis.com/token'
export const MESSAGES_END_POINT = 'https://gmail.googleapis.com/gmail/v1/users/me/messages'
export const GET_MESSAGE_END_POIND = '/gmail/v1/users/me/messages/'
```
필요한 변수들을 준비했습니다.

### 2. code 발급
OAuth2에서는 리디렉션 페이지에서 쿼리스트링으로 제공되는 Code를 발급받고 그 코드로 AccessToken을 발급받습니다. 그 Code를 발급받기위한 코드를 작성하겠습니다.

```typescript
import { Elysia } from "elysia";
import { 
  CLIENT_ID,
  AUTH_URI,
  REDIRECT_URI,
  SCOPES } from '../secrets';

const app = new Elysia()
  .get("/", () => "Hello Elysia")
  .get("/test", ({ redirect }) => {
    return redirect(`${AUTH_URI}?scope=${SCOPES}&access_type=offline&include_granted_scopes=true&response_type=code&prompt=consent&redirect_uri=${REDIRECT_URI}&client_id=${CLIENT_ID}`) 
})
  .listen(3000);

console.log(
  `🦊 Elysia is running at ${app.server?.hostname}:${app.server?.port}`
);
```

이후 `localhost:3000/auth`에 접속하면
![](https://velog.velcdn.com/images/kellyb9/post/cd4d8e01-bf4e-436f-aa9e-75c6f77742ee/image.png)
이렇게 계정을 선택하는 화면이 뜹니다.

계정을 선택하고 주소창을 보면 쿼리스트링에 `/auth?code=코드값`의 형태로 나와있습니다. 이 코드값을 가져오면됩니다.

```typescript
const app = new Elysia()
  .get("/", () => "Hello Elysia")
  .get("/test", ({ redirect }) => {
    return redirect(`${AUTH_URI}?scope=${SCOPES}&access_type=offline&include_granted_scopes=true&response_type=code&prompt=consent&redirect_uri=${REDIRECT_URI}&client_id=${CLIENT_ID}`) 
  })
  .get("/auth", async ({ query }) => {
    const code: string | undefined = query.code
    if (code) {
    try {
		// code를 이용해 token을 발급받는 함수를 불러옴
    } catch (err) {
      console.error(err);
    }
  }
})
  .listen(3000);

console.log(
  `🦊 Elysia is running at ${app.server?.hostname}:${app.server?.port}`
);
```

이제 맨 위에 token의 type과 tokenResponse라는 변수를 선언합니다
```typescript
type tokenType = {
	accessToken: string
	expiresIn: number
	scope: string
	token_type: string
}
let tokenResponse: tokenType | null = null;
```

다시 밑으로 내려와서

```typescript
import { Elysia } from "elysia";
import { 
  CLIENT_ID,
  AUTH_URI,
  REDIRECT_URI,
  SCOPES } from '../secrets';
import { issueToken } from 'gmail' ;

type tokenType = {
	accessToken: string
	expiresIn: number
	scope: string
	token_type: string
}
let tokenResponse: tokenType | null = null; 
const app = new Elysia()
  .get("/", () => "Hello Elysia")
  .get("/test", ({ redirect }) => {
    return redirect(`${AUTH_URI}?scope=${SCOPES}&access_type=offline&include_granted_scopes=true&response_type=code&prompt=consent&redirect_uri=${REDIRECT_URI}&client_id=${CLIENT_ID}`) 
  })
  .get("/auth", async ({ query, set}) => {
    const code: string | undefined = query.code
    if (code) {
    try {
      tokenResponse = await issueToken(code);
      if (!tokenResponse) {
        set.status = 401;
        return "No toke available";
      }
    } catch (err) {
      console.error(err);
    }
  }
})
  .listen(3000);

console.log(
  `🦊 Elysia is running at ${app.server?.hostname}:${app.server?.port}`
);
```

##### issueToken
```typescript
import axios from 'axios';
import {
	TOKEN_URI,
	CLIENT_ID,
	CLIENT_SECRET,
	REDIRECT_URI,
} from '../../secrets';

async function issueToken(code: string) {
	const searchParams = new URLSearchParams()
	searchParams.append('code', code)
	searchParams.append('client_id', CLIENT_ID)
	searchParams.append('client_secret', CLIENT_SECRET)
	searchParams.append('redirect_uri', REDIRECT_URI)
	searchParams.append('grant_type', 'authorization_code')

	try {
		const response = await axios.post(TOKEN_URI, searchParams, {
			headers: {
				'Content-Type': 'application/x-www-form-urlencoded'
			}
		})
		console.log(response.data)
		return response.data
	} catch (err) {
		console.error(err)
	}
	return null
}

export { issueToken }
```

여기까지 하면 Token을 발급받을 수 있습니다.


![](https://velog.velcdn.com/images/kellyb9/post/919183cc-0a56-49bd-ac83-47a39812a5ab/image.png)


# 5. 마무리

`elysia`에서는 라우팅을
```typescript
import { Elysia } from 'elysia'

// ❌ don't
const app1 = new Elysia()

app1.get('/', () => 'hello')

app1.post('/', () => 'world')

// ✅ do
const app = new Elysia()
    .get('/', () => 'hello')
    .post('/', () => 'world')
```
와 같이 하기를 권장하고 있습니다. 1번처럼 해도 동작은 하지만 `elysia`의 장점인 간단한 문법을 지키기 위해서 2번을 사용하는 것을 권장합니다. [elysia 공식문서](https://elysiajs.com/essential/route.html)

토큰 받아오는 것까지 했는데 다음엔 토큰을 이용해 gmail 메일 리스트를 보여주고 화면에 보여주는걸 목표로 포스팅 해보겠습니다 감사합니다.
