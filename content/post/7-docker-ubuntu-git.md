---
title: "Docker - Ubuntu에 git 설치: 이미지 만들기"
author: Bumgu
date: 2024-03-18
categories: 
    - DevOps
tags: 
    - Docker
slug: "docker-ubuntu-git-install"
---
Docker를 이용해 git 이 설치된 우분투 이미지를 만들어보겠습니다


# 1. Vanilia Ubuntu 실행
아무것도 설치되어 있지 않은 우분투 이미지를 실행하겠습니다.

`docker run -it --name velog-docker ubuntu:latest`  
`velog-docker`라는 이름으로 만들고, ubuntu레포지터리의 latest태그를 사용해 이미지를 이용해 컨테이너를 실행한다는 뜻 입니다.
```sh
docker run -it --name velog-docker ubuntu:latest
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
f4bb4e8dca02: Already exists
Digest: sha256:77906da86b60585ce12215807090eb327e7386c8fafb5402369e421f44eff17e
Status: Downloaded newer image for ubuntu:latest
root@edd8a22995f6:/#
```
저는 이전에 실행해본적이 있어서 `Already exists`가 떴습니다.
ubuntu에 접속해 bash 에 접속했습니다.
이후 `git --version` 명령어를 실행하면 `git`커맨드를 찾을 수 없다고 뜹니다. 아직 설치를 안했기 때문에 당연합니다.
![](/images/post/7-docker-ubuntu-git/1.png)

이제 순차적으로 git을 설치하겠습니다.

    1.`apt-get update`
    2.`apt-get install git -y`  
이 순서를 기억합니다.

다시 `git --version`을 실행하면 설치가 되었기 때문에 버전이 뜹니다.
![](/images/post/7-docker-ubuntu-git/2.png)


# 2. 현재 상태를 commit, 커스텀 이미지 생성
현재 git이 설치된 우분투의 상태를 commit해서 이미지로 이용할 수 있도록 하겠습니다.

`docker commit ${컨테이너 이름} ${reposetory}:${tag}`

```sh
➜ docker commit velog-docker ubuntu:velog
sha256:a2ef40a07b899a20064f511942355d3a8ecc3e1f6367d1d73feba77903fecebc

Desktop/velog/20. docker
➜ docker images
REPOSITORY                   TAG       IMAGE ID       CREATED         SIZE
ubuntu                       velog     a2ef40a07b89   3 seconds ago   188MB
```
아까 실행한 컨테이너의 이름인`velog-docker`란 이름으로 커밋을 하고, `velog`라는 태그를 달아줬습니다. 그리고 `docker images`명령어로 이미지를 확인하면 커밋되어 있는걸 확인 할 수 있습니다.

# 3. 커스텀 이미지를 이용해 컨테이너 실행

이제 방금 만든 이미지를 이용해 컨테이너를 실행하겠습니다.
방금 만든 `git`이 설치된 상태이기 때문에 컨테이너를 실행하면 `git`이 설치되어 있어야 합니다.
`docker run -it --name velog-docker2 ubuntu:velog`
이번엔 `velog-docker2`란 이름으로 컨테이너를 실행하고, 아까 생성한 이미지를 사용할 것이기때문에 `ubuntu:velog`를 사용합니다.
실행이되면 `git --version`을 실행해 `git`이 설치되어있는지 확인합니다.


![](/images/post/7-docker-ubuntu-git/3.png)


# 4. Dockerfile 로 build
이와같은 일련의 과정을 하나의 파일로 만들어 관리한다면 유지보수와 협업이 쉬워질 것입니다.
그래서 `Dockerfile`을 이용해 해보도록 하겠습니다.

먼저 `Dockerfile`파일을 생성하고 내용을 작성합니다.

```sh
FROM ubuntu:latest

RUN apt-get update
RUN apt-get install git -y
```
아까 1번에서 봤던 `git`설치 순서와 동일합니다.
명령줄로 입력했던것을 파일로 관리하는것 뿐입니다.

`docker build -t ubuntu:velog-git-docker .`

- `docker build`: Docker 이미지를 빌드하는 명령어입니다.
- `-t ubuntu:velog-git-docker`: 이 플래그는 빌드된 이미지에 태그를 지정합니다. 여기서는 ubuntu라는 리포지토리에 velog-git-docker라는 태그를 지정합니다.
- `.`: 현재 디렉토리를 빌드 컨텍스트로 사용합니다. Docker는 현재 디렉토리와 그 하위 파일 및 디렉토리를 기반으로 이미지를 빌드합니다.

따라서 이 명령은 현재 디렉토리의 Dockerfile을 사용하여 새로운 이미지를 빌드하고, 이 이미지를 ubuntu 리포지토리에 velog-git-docker라는 태그를 붙여 저장합니다.

이제 이미지를 build했으니 그 이미지를 이용해 컨테이너를 실행해보겠습니다

`docker run -it --name velog-docker3 ubuntu:velog-git-docker`
`velog-docker3`란 이름으로 컨테이너를 실행하고 사용할 이미지는 방금 `Dockerfile`로 빌드한 `ubuntu:velog-git-docker`입니다.

실행이되면 `git --version`을 실행해 `git`의 설치여부를 확인합니다.

![](/images/post/7-docker-ubuntu-git/4.png)


# 마무리
Docker를 공부중입니다. 잊지않기 위해 차근차근 포스팅 하겠습니다. 감사합니다.

