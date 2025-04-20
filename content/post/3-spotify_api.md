---
title: "Spotify API 사용 (Node.js, express, React)"
author: Bumgu
date: 2024-01-31
categories: []
tags:
  - React
  - express
  - node.js
  - Spotify
slug: Spotify_api
---

[깃허브 링크](https://github.com/bonzonkim/spotify-ranking-app)

원래 Next.js로 구현해보려 했으나 OAuth2.0기반 로그인도 처음 구현해보는데, Next.js의 숙련도도 부족해 Node.js + Express + React로 구현했습니다.

# 1. Spotify의 로그인 방식
Spotify 는 OAuth2.0 프레임워크를 사용합니다.

![](/images/post/3-spotify_api/0.png)
먼저 API를 사용하기위해 필요한건
- `code` : `access_token`, `refresh_token` 과 교환하기위함
- `access_token` : 실제 정보를 받기 위해 필요한 토큰
- `refresh_token` : `access_token`이 만료되면 다시 받기위한 토큰
- `client_id` : 스포티파이 디벨로퍼 앱의 식별번호 (개발자에게 부여)
- `client_secret` : 스포티파이 디벨로퍼 앱의 시크릿 번호 (개발자에게 부여)

흐름은
1.`client_secret`, `client_id` (스포티파이 개발자 대쉬보드에서 발급)
2.`code`발급
3.`access_token`,`refresh_token` 발급
4.`access_token`을 이용해 API사용 가능
5.`access_token`만료 시 `refresh_token`을 이용해 다시 발급
이후 다시 4번 > 5번 > 4번 > 5번 ... 이 됩니다.

# 2. 스포티파이 개발자 대쉬보드
https://developer.spotify.com 로 접속하고 로그인합니다.
이후 오른쪽 위 이름을 클릭해 대쉬보드로 이동하고 `create app`버튼을 클릭합니다.

![](/images/post/3-spotify_api/1.png)

`App name` : 앱의 이름
`App description` : 앱의 설명 (필수 x)
`Website` : 웹사이트의 주소 (필수 x)
`Redirect URI` : **필수!** 코드 발급 시 리다이렉트될 콜백 페이지입니다. 보통 개발환경에서는 `http://localhost:3000/callback` 등으로 설정합니다.

그리고 본인이 사용 할 API를 클릭합니다. 이후에 수정이 가능합니다.
이후 대쉬보드에서 만든 앱을 클릭하고 setting에 들어가면 `clientId`, `client secret` 확인이 가능합니다.

![](/images/post/3-spotify_api/2.png)

# 3. 프로젝트 세팅

### 3-1. Express서버

리액트 앱을 만들고 그 안에 Express서버를 세팅하겠습니다.
`yarn create react-app appname`
이후 react 폴더에
`yarn add express` 명령어로 express를 설치하고
`server`라는 폴더를 만들고 `index.js`에 express서버를 만듭니다.
```ts
const express = require('express');
const path = require('path');
const app = express();

app.use(express.static(path.join(__dirname, '..', 'build')));
app.use(express.json());


app.get('/', function (req, res) {
  res.sendFile(path.join(__dirname, '..', 'build', 'index.html'));
});

app.listen(9000,() => {
  console.log('SERVER LISTENING ON 9000')
}) ;
```
```ts
app.get('/', function (req, res) {
  res.sendFile(path.join(__dirname, '..', 'build', 'index.html'));
});
```
는 `build`폴더의 `index.html`을 렌더링합니다. 아직 `build`폴더가 없기 때문에 `yarn build`명령어를 통해 빌드해줍니다.


Express는 9000, 리액트는 3000에서 실행됩니다. 하지만 저는 하나의 포트에서 관리하고 싶으므로 `React Proxy`세팅을 해주도록 하겠습니다.

### 3-2. React Proxy

[공식문서](https://create-react-app.dev/docs/proxying-api-requests-in-development/)
`src/`경로에 `setUpProxy.js`를 만듭니다.
```ts
const { createProxyMiddleware } = require('http-proxy-middleware');


module.exports = function(app) {
  app.use(
    '/api',
    createProxyMiddleware({
      target: 'http://localhost:9000',
      changeOrigin: true,
    })
  )
}
```
저는 백엔드 엔드포인트를 `/api`경로에 둘 것이기 때문에 `/api`로 했고, `target`은 Express 실행 포트인`localhost:9000` 로 했습니다. 프록시 설정을 하고나면 리액트 실행포트인 3000포트로 (`localhost:3000/api/~~~`)으로 요청해도 Express서버의 실행포트은 9000포트로 바꿔줍니다. 즉 하나의 포트만 사용하듯이 사용할 수 있습니다.

# 4. Spotify Token 요청
### 4-1. code 발급
우선 코드발급의 흐름은 `https://acounts.spotify.com/authorize`경로에 쿼리스트링으로 데이터들을 넣어서 요청하면 대쉬보드에서 설정한 Redirect_uri(콜백페이지)로 리디렉션되며 리디렉션된 콜백페이지의 쿼리스트링으로 담겨옵니다.

| URL | mehtod | 필요한 데이터 |
| --- | --- | --- |
| https://accounts.spotify.com/authorize | GET | response_type, client_id, scope, redirect_uri |

`https://accounts.spotify.com/authorize?response_type=code&client_id=$CLINET_ID&scope=${SCOPE}&redirect_uri=${REDIRECT_URI}` 의 모습을 띕니다.

`response_type` : code를 발급받기에 code를 입력합니다.
`client_id` : 스포티파이 대쉬보드에서 부여받은 번호
`scope` : scope에 따라 접근할 수 있는 데이터의 범위가 달라집니다 [scope list](https://developer.spotify.com/documentation/web-api/concepts/scopes)
`redirect_uri` : 스포티파이 대쉬보드에서 설정한 콜백페이지 주소

이제 코드로 써보겠습니다.

- server/routes/api.js
```ts
const express = require('express');
const { SPOTIFY_CLIENT_ID, SCOPE, REDIRECT_URI } = require('../config');

const apiRouter = express.Router();


apiRouter.get('/login', (req, res) => {
  res.redirect(`https://accounts.spotify.com/authorize?response_type=code&client_id=${SPOTIFY_CLIENT_ID}&scope=${SCOPE}&redirect_uri=${REDIRECT_URI}`);
});
```

`/api`경로로 들어오는 요청을 수행하는 파일이기 때문에 라우터 설정을 했습니다.
필요한 환경변수들은 `dotenv`를 사용해 `config.js`파일에 설정했고, import 해왔습니다.
이 경로로 필요한데이터를 입력하고 요청을하게되면 설정한 `redirect_uri`로 리디렉션되며 주소창을 보면 `/callback?code=.....`의 형식을 띄고있는데 여기서 `code`를 가져오면 됩니다.

`http://localhost:3000/api/callback?code=AQAYxRYFF7hAkwLNG-Q3qUJ9K7Wew3Dk6NKr-6bwk0POYHiSQg2QQLpukCOM-uA2YCkysqv1I...`
의 형식을 가지고 있습니다.

### 4-2. Token 발급
이제 발급받은 `code`로 토큰을 발급해야 합니다.

![](/images/post/3-spotify_api/3.png)
본인이 스포티파이 대쉬보드에 설정했던 `redirect_uri`로 리디렉션이 되므로 그에 맞는 엔드포인트를 만듭니다. 저의 경우에는 `api/callback`였습니다.




- src/routes/api.js
```ts
apiRouter.get('/callback', (req, res) => {
  const searchParams = new URLSearchParams(req.query);
  const code = searchParams.get('code');

  const tokenResponse = await getSpotifyRefreshToken(code);// 스포티파이 토큰을 받아오는 함수를 tokenResponse에 저장

  if (tokenResponse) { // tokenResponse에 값이 있을 경우 쿠키에 저장
    res.cookie('refresh_token', tokenResponse.refresh_token, {
      httpOnly: true,
    })

  }
  res.redirect('http://localhost:3000'); // 이후 메인페이지로 리디렉션
});
```
우선 주소창의 `code` 값을 가져오고, 토큰과 교환하는 함수의 인자에 넣어줍니다

| URL | method | header |필요한 데이터 |
| --- | --- | --- | --- |
| https://accounts.spotify.com/api/token | POST | Content-Type: application/x-www-url-encoded, Authorization: Basic ${basicToken} | grant_type, redirect_uri, code |

- `grant_type` : 인가타입입니다. 'authorization_code'입니다.
- `redirect_uri` : 스포티파이 대쉬보드에서 설정한 콜백페이지 입니다.
- `basicToken` : base64 방식으로 인코딩된 `client_id : client_secret`입니다.
- `code` : 위 단계에서 발급받은 코드입니다.

- lib/spotify.js
```ts
async function getSpotifyRefreshToken(code) {
  const searchParams = new URLSearchParams();
  searchParams.append('grant_type', 'authorization_code');
  searchParams.append('redirect_uri', REDIRECT_URI);
  searchParams.append('code', code);

 const basicToken = new Buffer.from(SPOTIFY_CLIENT_ID + ':' + SPOTIFY_CLIENT_SECRET).toString('base64');

  try {
   const response = await axios.post(SPOTIFY_TOKEN_ENDPOINT, searchParams, {
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Authorization': `Basic ${basicToken}`,
      },
    });

    return response.data;
  } catch (err) {
    console.error(err);
  }
  return null;
}
```
필요한 데이터들을 `URLSearchParams`객체에 넣어줍니다.
`basicToken`은 base64 방식으로 인코딩된 `client_id : client_secret`입니다.
axios를 이용해 post방식으로 토큰을 받는 스포티파이 엔드포인트https://accounts.spotify.com/api/token 에 데이터를 넣어 요청합니다.
이후 JSON데이터로 반환이되고 반환 값은
```json
{
"access_token":"BQD-8jTKGyYS7W-LfaMvwj_vgGxRoFiWtVKfwZPovf15_tjYbOXGF9sg_p9KS1VRNIqeox2TpiiFAhncntC...."
,"token_type":"Bearer"
,"expires_in":3600,
"refresh_token":"AQCIF3IBVLc_R560C-sB_YtOfophiZjrw...",
"scope":"user-top-read"
}
```
의 형식을 가지고 있습니다.
다시 `api/callback` 의 코드를 보면
```ts
const tokenResponse = await getSpotifyRefreshToken(code);
if (tokenResponse) { // tokenResponse에 값이 있을 경우 쿠키에 저장
    res.cookie('refresh_token', tokenResponse.refresh_token, {
      httpOnly: true,
    })
```
반환받은 response의 `refresh_token`값을 쿠키에 넣어줍니다. 메인페이지 접속시 쿠키의 `refresh_token`의 값을 확인하고, 있다면 데이터를 렌더링하고, 없다면 로그인 버튼을 렌더링하기 위해 쿠키에 저장했습니다.

이제
>`/api/login`으로 요청 >
스포티파이 로그인창 >
`/api/callback`(토큰발급 후 쿠키에 저장) >
메인페이지로 리디렉션

의 흐름이 완성되었습니다.
`/api/login`으로 접속하고 스포티파이 로그인을 하고 개발자탭에서 Application탭을 확인해보면
![](/images/post/3-spotify_api/4.png)

쿠키가 저장되는 모습을 확인할 수 있습니다.

이제 이 토큰을 가지고 실제 스포티파이 유저 데이터를 받아오고, 화면에 보여주면 됩니다.


# 5. API로 사용자 정보 요청

토큰까지 받아서 쿠키에 저장이 되었습니다.
제가 하려는건
>1. 메인페이지 접속 자동으로 쿠키의 토큰값을 검사
2. 토큰이 있으면 사용자의 최근6개월간  많이 들은 트랙들을 렌더링
3. 아닐경우 로그인 버튼을 렌더링

의 흐름입니다.

백엔드에서 세팅해준 `refresh_token`은 `httponly`로 설정되어서 브라우저에서 값을 확인 할 수 없습니다. 그렇기때문에 쿠키의 값을 확인하고, 그 값으로 사용자 정보를 요청하는 백엔드 엔드포인트를 만들어주겠습니다.


### 5-1. 토큰확인 백엔드 엔드포인트
- server/api.js
```ts
apiRouter.get('/token', async (req, res) => {
    const refreshToken = req.cookies.refresh_token; //쿠키에 저장된 토큰값

    const tokenResponse = await getSpotifyAccessTokenByRefreshToken(refreshToken);
    //토큰을 기반으로 액세스토큰 발급

    if(tokenResponse) {
    //액세스 토큰으로 유저프로필과 탑트랙데이터를 받아옴
      const topTrackData = await getTopTrackData(tokenResponse);
      const userData = await getUsersProfile(tokenResponse);
      res.status(200).json({topTrackData: topTrackData, userData: userData});
      // 데이터를 담은 JSON과함께 응답
    }
});
```
브라우저에서는 `useEffect`를 사용해 첫 렌더시 `api/token`으로 요청을 하고 쿠키의 토큰값에 따라 다음 행동이 결정됩니다.

이제
`getSpotifyAccessTokenByRefreshToken()` : 쿠키에 저장된 `refresh_token`의 값으로 `access_token`발급
`getTopTrackData()` : 발급받은 `access_token`으로 유저의 탑트랙 데이터 요청
`getUsersProfile()` : 발급받은 `access_token`으로 유저 프로필 데이터 요청

이 3개의 함수들을 작성하겠습니다.


### 5-2. 데이터 요청 함수

- server/lib/spotify.js

#### access_token 발급
```ts
async function getSpotifyAccessTokenByRefreshToken(refreshToken) {
  const searchParams = new URLSearchParams();
  searchParams.append('grant_type', 'refresh_token');
  searchParams.append('refresh_token', refreshToken);
  searchParams.append('client_id', SPOTIFY_CLIENT_ID);
  const basicToken = new Buffer.from(SPOTIFY_CLIENT_ID + ':' + SPOTIFY_CLIENT_SECRET).toString('base64');

  try {
    const response = await axios.post(SPOTIFY_TOKEN_ENDPOINT, searchParams, {
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Authorization': `Basic ${basicToken}`,
      },
    });

    return response.data.access_token;
  } catch (err) {
    console.error(err);
  }
  return null;
}
```
| URL | method | headers | 필요한 데이터 |
| --- | --- | --- | --- |
| https://accounts.spotify.com/api/token | POST | Content-Type: application/x-www-form-urlencoded, Authorization : Basic ${basicToken} | grant_type, refresh_token, client_id |

`grant_type` : 'refresh_token'
`refresh_token` : 쿠키에 저장된 `refresh_token`의 값
`client_id` 스포티파이 대쉬보드에서 부여받은 번호

`URLSearchParams`객체에 필요한데이터 들을 넣어주고, axios를 통해`POST`방식으로 요청합니다.
요청 후 응답 JSON객체에서 `access_token`을 반환합니다.

#### 유저 탑 트랙 데이터 요청
```ts
async function getTopTrackData(accessToken) {
  try {
     const response = await axios.get(SPOTIFY_TOP_TRACK_ENDPOINT, {
            headers: {
                'Authorization': `Bearer ${accessToken}`
            }
        })
    return response.data;
  } catch (err) {
    console.error(err);
  }
  return null;
}
```
| URL | method | headers | 필요한 데이터 |
| --- | --- | --- | --- |
| https://api.spotify.com/v1/me/top/tracks | GET | Authorization: Bearer ${access_token} | - |

유저의 최근 많이들은 트랙들을 받아오는 함수입니다. 이전의 Token발급과정과는 달리 단순합니다.

`TopTrackData`의 반환 JSON은 굉장히 많은 데이터를 포함하고 있습니다. 클라이언트에서 골라 사용할 것이기 때문에 `response.data`를 반환합니다.

#### 유저 프로필 데이터 요청
```ts
async function getUsersProfile(accessToken) {
  try {
     const response = await axios.get(SPOTIFY_USER_DATA_ENDPOINT, {
            headers: {
                'Authorization': `Bearer ${accessToken}`
            }
        })
    return response.data;
  } catch (err) {
    console.error(err);
  }
  return null;
}
```
| URL | method | headers | 필요한 데이터 |
| --- | --- | --- | --- |
| https://api.spotify.com/v1/me/ | GET | Authorization: Bearer ${access_token} | - |
유저의 탑트랙 데이터와 다른점은 `URL`밖에 없습니다. 이것 역시 JSON객체를 응답해주기 때문에 `response.data`를 반환합니다.

# 6. 프론트에서 데이터 받기
이제 백엔드에서의 작업은 끝났습니다.
프론트에서 보여주기만 하면 됩니다.

### 6-1. 데이터 패칭
- src/App.js
```ts
  const [token, setToken] = useState(false);
  const [topTrackData, setTopTrackData] = useState([]);
  const [userData, setUserData] = useState('');
```
- [token, setToken] : 토큰값에 따라 조건부 렌더링 위함 값은 boolean
- [topTrackData, setTopTrackData] : 받아온 탑트랙 데이터를 세팅해줄 변수, 값은 배열
- [userData, setUserData] : 유저 프로필 데이터를 세팅할 변수, 유저의 이름만 사용할것이기에 값은 string

```ts
  useEffect(() => {
    async function fetchToken() {
      try{
        // cookie에 저장돼있는 토큰값을 확인하고 데이터 받아옴
        const response = await axios.get('/api/token');
        if (response) {
          setToken(true);
          setUserData(response.data.userData.display_name);
          setTopTrackData(response.data.topTrackData.items);
        }
      } catch (err) {
        console.error('Error fetching token :', err);
      }
    }
    fetchToken();
  }, [])
```
값을 세팅해줄 변수들은 만들었으니 이제 `useEffect`내부에
`async function`을 작성합니다.
- 1. 첫 렌더링시 `/api/token`으로 요청해 쿠키의 토큰값을 확인,
- 2. `/api/token`은 위에서 작성했듯이, 실제 데이터를 받아옵니다.
- 3. 요청 결과값을 `response`에 저장
- 4. `response`의 값 유무에 따라 `if`문을 따라갑니다.
	1. `token`에 `true` 할당
    	2.받아온 유저 프로필데이터의 `display_name`키를 가진 값을 `userData`에할당
        	3.받아온 탑트랙데이터의 `items`(배열)을 `topTrackData`에 할당

- 5. 만약 `response`의 값이 없을 경우 에러를 출력.


### 6-2. 받아온 데이터 렌더링
이제 데이터 준비까지 끝났습니다. 이제 `return`문안에 데이터를 렌더링하겠습니다.
```ts
return (
  <div className="App">
    {token ? ( // token이 true일 때
      <>
        <h3>{userData}'s Top Track for last 6 months</h3>
        <div>
            <TrackList topTrackData={topTrackData} />
        </div>
      </>
    ) : ( // token이 false일 때
      <button onClick={() => window.location.href='/api/login'}>Login to Spotify</button>
    )}
  </div>
);
```
`token`값은 `/api/token`으로 요청후 응답이 있는 경우에 `true`가 됩니다. 즉 로그인 하기 전에는 `false`입니다. 그렇기 때문에 삼항연산자를 이용해 조건부렌더링을 해줍니다.
- `token`이 `true` 일 때는 데이터를 보여주고
- `token`이 `false`일 때는 로그인 버튼을 보여줍니다.

유저의 이름은 이미 `response.data.display_name`으로 할당했기에 바로 유저의 이름이 할당됩니다.
유저의 탑트랙 데이터는 설정한 조건에 따라 개수가 다르지만, 기본값은 최근 6개월간의 20개를 순서대로 보여줍니다. `TrackList`라는 컴포넌트를 만들어서 `topTrackData`를 props로 넘겨주겠습니다.

- src/components/TrackList.jsx
```ts
function TrackList({ topTrackData }) {
  return (
  <div>
      {topTrackData.map((track, index) => (
      <div key={index}>
          <h3>{track.album.name}</h3>
          <h3>{track.album.artists[0].name}</h3>
          <img src={track.album.images[1].url} alt={'artist image'}/>
        </div>
      ))}
    </div>
  )
}

export default TrackList;
```
스포티파이에서 받아온 `topTrackData`는 배열로 반환이 됩니다. `.map`함수를 사용해 차례대로 렌더링 해줍니다.
차례대로 앨범명, 가수이름, 앨범이미지 입니다.
가수이름은 가수가 1명인 경우가있고, 여러명인 경우가 있기에 `artists`라는 배열안에 값이 들어있습니다. 여기서 `.map`함수를 한번 더 사용해 모두 보여주도록 하겠습니다.

```ts
return (
  <div>
      {topTrackData.map((track, index) => (
      <div key={index}>
          <h3>{track.album.name}</h3>
          {track.album.artists.map((artist, index, arr) => (
          <span>{artist.name} {index < arr.length -1 && ' ,'}</span>
          ))}
          <div></div>
          <img src={track.album.images[1].url} alt={'artist image'}/>
        </div>
      ))}
    </div>
  )
```
`track.album.artists`는 위에서 말했듯이 배열입니다. `.map`함수를 사용해 `artists`배열을 순회하며 아티스트를 모두 보여주고, 여러명인 경우 `{index < arr.length -1}`를 통해 배열안의 마지막 아티스트가 아닌지를 확인하고 `&&`연산자를 통해 앞의 조건이 `true`일 경우 ` ,`를 추가합니다.
줄바꿈을 위해 block요소인 빈`<div>`를 추가했습니다.



# 7. 결과 확인

![](/images/post/3-spotify_api/5.gif)
이렇게 Spotify API를 사용해 유저의 최근 많이들은 노래들과, 유저 프로필의 이름을 가져와 보여줍니다. 포스팅은 여기서 마치겠습니다. 이후 계속 발전시킬 생각이라 따라했는데 잘 안되거나 코드가 궁금하신분은
[깃허브 링크](https://github.com/bonzonkim/spotify-ranking-app)에서 확인 하시면 됩니다. 감사합니다.
