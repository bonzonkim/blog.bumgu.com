---
title: "GitHub Actions Docker 빌드에서 사라지지 않는 디스크 사용량 줄이기"
author: Bumgu
draft: false
date: 2026-09-23T00:00:00+09:00
categories:
    - DevOps
    - CI/CD
tags:
    - GitHub Actions
    - Docker
    - Buildx
    - ECR
slug: "github-actions-buildx-disk"
---


## 문제

GitHub Actions에서 Docker 이미지를 빌드하다가 러너 디스크가 부족해지는 문제를 확인했습니다. 압축 크기 1.40GB인 이미지가 같은 공유 워크플로에서 `no space left on device`로 두 번 실패했는데, 두 번의 실패 지점이 서로 달랐습니다. 처음에는 Dockerfile이 너무 크거나 캐시가 오래 쌓였다고 생각했습니다. 하지만 이번 경우에는 한 번의 이미지 빌드 결과가 서로 다른 저장소에 여러 번 복제되는 구조가 원인이었습니다.

이 글에서는 기존 Buildx 빌드와 push 흐름에서 어떤 데이터가 어디에 저장되는지 정리하고, `load: true`를 제거해 로컬 Docker 데몬 사본을 없애는 방법을 설명합니다.

## 배경

서비스 저장소마다 CI/CD 워크플로를 따로 두지 않고, 공용 워크플로 저장소 하나에 재사용 워크플로(`workflow_call`)와 composite 액션을 모아 두고 있습니다. 각 서비스 저장소는 Dockerfile 경로, 이미지 이름, 배포 환경 같은 값만 넘기고 빌드와 push, manifests 갱신, ArgoCD 배포는 공용 워크플로가 수행합니다.

```yaml
# 서비스 저장소의 워크플로
jobs:
  deploy:
    uses: <org>/workflows/.github/workflows/deploy.yml@v2
    with:
      image-name: my-service
      docker-file: ./Dockerfile
      stage: staging
```

이 구조에서는 Docker 빌드 방식이 공용 저장소의 build 액션 한 곳에 정의되어 있습니다. 그래서 빌드 과정에서 생기는 디스크 사용 문제도 서비스마다 고치는 것이 아니라 공용 액션에서 한 번 고쳐야 하고, 반대로 공용 액션의 구조적인 문제는 큰 이미지를 가진 서비스에서 먼저 드러납니다.

## 기존 워크플로 구조

서비스 저장소는 재사용 워크플로를 호출합니다.

```text
서비스 저장소
  -> 공용 워크플로우 호출 (CI/CD 가 이뤄짐)
  -> ci job
  -> docker/build
  -> docker/push
  -> ECR
  -> 저장소 manifests 수정,커밋,푸쉬
  -> ArgoCD 배포
```

기존 공유 build 액션은 이미지를 로컬 Docker 데몬으로 적재하고, 별도 push 액션이 그 이미지를 ECR로 전송하는 방식입니다.

```yaml
- name: Build Docker image
  uses: docker/build-push-action@v7
  with:
    push: false
    load: true
    tags: ${{ inputs.tags }}
    cache-from: type=local,src=${{ runner.temp }}/.buildx-cache
    cache-to: type=local,dest=${{ runner.temp }}/.buildx-cache-new,mode=max

- name: Push Docker image
  run: docker push "$IMAGE_TAG"
```

`docker push` 명령은 로컬 Docker 데몬에서 태그를 찾습니다. 그래서 이 구조에서는 `load: true`가 필요했습니다. 문제는 push만을 위해 이미지를 한 번 더 로컬 데몬에 가져오는 비용입니다.

## 빌드 중 동시에 존재하는 데이터

캐시를 사용하는 빌드에서는 최종 이미지와 중간 레이어가 여러 위치에 공존할 수 있습니다.

![](/images/post/33-github-actions-buildx-disk/buildx_disk_copies_breakdown.png)

1. BuildKit builder 컨테이너의 content store, 스냅샷, 빌드 캐시입니다.
2. `load: true`가 만든 Docker image archive stream입니다. 이는 별도 파일로 저장하지 않는 한 영속 사본이 아닙니다.
3. Docker 데몬이 import한 이미지 레이어입니다.
4. `actions/cache`가 복원한 이전 `.buildx-cache` 디렉터리입니다.
5. `mode=max`로 생성하는 신규 `.buildx-cache-new` 디렉터리입니다.

특히 로컬 캐시는 기존 캐시를 지우기 전에 신규 캐시를 완성합니다.

```yaml
cache-from: type=local,src=${{ runner.temp }}/.buildx-cache
cache-to: type=local,dest=${{ runner.temp }}/.buildx-cache-new,mode=max

- name: Save Docker cache
  run: |
    rm -rf "${{ runner.temp }}/.buildx-cache"
    mv "${{ runner.temp }}/.buildx-cache-new" "${{ runner.temp }}/.buildx-cache"
```

따라서 cache export가 끝나기 전에는 이전 캐시와 신규 캐시가 함께 존재합니다. `mode=max`는 최종 레이어뿐 아니라 중간 레이어도 내보내므로 캐시 크기를 크게 만들 수 있습니다.

두 개의 이미지 태그는 같은 레이어를 공유하므로 레이어 전체가 두 배가 되지는 않습니다. 핵심 문제는 태그 수가 아니라 BuildKit, 로컬 Docker 데몬, 로컬 캐시가 각각 보유한 데이터입니다.

## 실제로 실패한 두 지점

위 목록의 3번과 5번이 각각 실제 실패로 나타났습니다. 같은 서비스에서 캐시를 끄고 빌드한 경우와 켜고 빌드한 경우가 서로 다른 지점에서 디스크를 소진했습니다.

캐시를 끈 빌드(`use-cache: false`, `no-cache: true`)는 빌드 자체는 끝났지만 Docker 데몬이 1.39GB 레이어를 import하는 중에 실패했습니다.

```text
#20 loading layer 063b05006375 1.39GB / 1.39GB 84.6s done
#19 exporting to docker image format
#19 sending tarball 119.6s done
#20 ERROR: write /app/data/<large-file>: no space left on device
 > importing to docker:
```

캐시를 켠 빌드는 `mode=max` 로컬 캐시를 `.buildx-cache-new`로 내보내는 중에 실패했습니다.

```text
#24 ERROR: error writing layer blob: failed to copy: failed to send write:
rpc error: code = Unknown desc = write /home/runner/work/_temp/.buildx-cache-new/ingest/6ccf6c85...: no space left on device
```

첫 번째 실패는 `load: true`가 만드는 데몬 사본 때문이고, 두 번째 실패는 로컬 캐시 export 때문입니다. 따라서 둘 중 하나만 고치면 다른 한쪽 경로는 그대로 실패합니다. 같은 Dockerfile을 `docker/build-push-action`의 `push: true`로 직접 push하도록 바꾼 서비스 저장소 자체 워크플로는 같은 1.40GB 이미지를 문제없이 빌드했으므로, 공유 액션도 같은 방향으로 바꾸기로 했습니다.

## 해결 방향: BuildKit에서 ECR로 직접 push하기

`docker/build-push-action`은 레지스트리로 직접 push할 수 있습니다. BuildKit이 ECR로 이미지를 전송하면 로컬 Docker 데몬에 이미지를 import할 이유가 없습니다.

최종적으로 적용한 공유 build 액션은 다음과 같습니다. 기존에는 캐시 사용 여부에 따라 빌드 스텝이 두 개로 나뉘어 있었고 no-cache 쪽 스텝에는 `id`가 없어 `digest` 출력이 비어 있었는데, 스텝을 하나로 합치면서 이 문제도 같이 정리했습니다.

```yaml
inputs:
  use-cache:
    default: 'true'
  cache-scope:
    description: GitHub Actions cache scope; caching is disabled when empty
    default: ''
  push:
    description: Whether to push the built image or load it locally
    default: 'false'

- name: Build Docker image
  id: build
  uses: docker/build-push-action@v7
  with:
    context: ${{ inputs.context }}
    file: ${{ inputs.dockerfile }}
    tags: ${{ inputs.tags }}
    push: ${{ inputs.push == 'true' }}
    load: ${{ inputs.push != 'true' }}
    cache-from: ${{ inputs.use-cache == 'true' && inputs.cache-scope != '' && format('type=gha,scope={0}', inputs.cache-scope) || '' }}
    cache-to: ${{ inputs.use-cache == 'true' && inputs.cache-scope != '' && format('type=gha,mode=max,ignore-error=true,scope={0}', inputs.cache-scope) || '' }}
    no-cache: ${{ inputs.use-cache != 'true' || inputs.cache-scope == '' }}
    provenance: false
```

CI 워크플로는 build 액션에 `push: 'true'`와 이미지별 `cache-scope`를 전달합니다.

```yaml
- name: Build Docker image
  uses: <org>/workflows/.github/actions/docker/build@v2
  with:
    # 기존 context, Dockerfile, tags, build args를 전달합니다.
    use-cache: ${{ inputs.use-cache }}
    cache-scope: ${{ steps.vars.outputs.image-repository }}
    push: 'true'
```

별도 `docker push` 스텝은 두 재사용 워크플로에서 제거했습니다. 재사용 워크플로의 입력과 출력은 바뀌지 않습니다. 호출하는 서비스는 기존과 동일하게 이미지 repository와 tag를 받아 ArgoCD 배포 단계에 전달합니다. build 액션을 직접 호출하던 저장소는 `push` 기본값이 `false`라 기존처럼 로컬 데몬에 load하며, `cache-scope`를 지정하지 않으면 캐시 없이 빌드합니다.

`provenance: false`는 ECR에 기본 provenance attestation이 별도 매니페스트 목록으로 올라가는 것을 막기 위해 추가했습니다. SBOM이나 provenance를 ECR에서 관리해야 하는 환경이라면 이 값은 조직의 공급망 보안 정책에 맞춰 별도로 결정해야 합니다.

## 로컬에서 `load` 비용 측정하기

실제 GitHub-hosted runner의 디스크와 macOS의 Docker 환경은 다릅니다. 다만 `load: true`가 Docker Engine에 최종 이미지를 추가하는지, 그 이미지가 얼마나 큰지는 같은 Dockerfile로 확인할 수 있습니다.

테스트에는 작은 Go 애플리케이션을 사용했습니다. Buildx builder는 `disk-test-load`이고, `docker-container` driver로 실행했습니다.

### `--load`를 사용한 빌드

먼저 `--load`를 붙여 빌드했습니다.

```console
➜ docker buildx build \
  --builder disk-test-load \
  --file ./go-app/Dockerfile \
  --tag local/go-app-disk-test:with-load \
  --load \
  ./go-app

[+] Building 15.9s (18/18) FINISHED
=> exporting to oci image format
=> => sending tarball
=> importing to docker
```

마지막 두 줄이 중요합니다. BuildKit이 Docker image archive를 전송하고 Docker Engine이 이를 import했습니다. 이어서 Docker Engine에 생긴 이미지 크기를 확인했습니다.

```console
➜ docker image inspect local/go-app-disk-test:with-load \
  --format '{{.Size}} bytes'
8536017 bytes
```

이번 테스트에서 `--load`가 Docker Engine에 추가한 최종 이미지 크기는 `8,536,017 bytes`, 약 `8.14 MiB`였습니다.

### `--load` 없이 같은 빌드 실행

같은 builder, Dockerfile, build context로 `--load`만 빼고 실행했습니다.

```console
➜ docker buildx build \
  --builder disk-test-load \
  --file ./go-app/Dockerfile \
  --tag local/go-app-disk-test:without-load \
  ./go-app

[+] Building 0.4s (14/14) FINISHED
WARNING: No output specified with docker-container driver. Build result will only remain in the build cache. To push result image into registry use --push or to load image into docker use --load
```

`--tag`는 결과에 붙일 이름일 뿐입니다. `docker-container` builder에서는 `--load`, `--push`, `--output` 중 하나가 있어야 BuildKit 밖으로 결과를 내보냅니다. 이 실행은 export를 요청하지 않았으므로 결과가 BuildKit cache에만 남았습니다.

```console
➜ docker image inspect local/go-app-disk-test:without-load \
  --format '{{.Size}} bytes'

Error response from daemon: No such image: local/go-app-disk-test:without-load
```

이 오류는 빌드 실패가 아닙니다. `docker image inspect`는 Docker Engine image store만 조회하는데, 두 번째 빌드는 그곳에 이미지를 import하지 않았기 때문에 발생했습니다.

### BuildKit에 남은 용량

`--load`를 빼도 BuildKit의 content store와 snapshot cache는 남습니다. 다음 명령으로 확인했습니다.

```console
➜ docker buildx du --builder disk-test-load --verbose

95.56MB   mount / from exec /bin/sh -c go build -o main .
308.8MB   pulled from docker.io/library/golang:1.25-alpine
13.5MB    pulled from docker.io/library/alpine:latest
12.23MB   [stage-1 3/3] COPY --from=builder /app/main .

Reclaimable:  446.4MB
Total:        446.4MB
```

이 `446.4MB`는 BuildKit builder에 남은 cache와 snapshot의 총량입니다. `Reclaimable`은 BuildKit garbage collection의 대상이라는 뜻이지, 이미 디스크에서 사라졌다는 뜻은 아닙니다.

두 번의 빌드는 같은 builder cache를 재사용했습니다. 따라서 이 숫자는 `--load`만의 비용이 아니라 두 실행이 공유한 BuildKit 저장 영역입니다. `--load`의 직접 비용은 Docker Engine에 추가된 `8,536,017 bytes`입니다.

| 저장 영역 | 관측값 | `--load`를 빼면 |
| --- | ---: | --- |
| BuildKit content store와 snapshot cache | 446.4MB | 유지됩니다 |
| Docker Engine 최종 이미지 | 8,536,017 bytes | 생성되지 않습니다 |

## `type=gha`로 캐시 복사본 줄이기

`load` 제거는 Docker 데몬 사본을 없앱니다. 로컬 캐시를 GitHub Actions cache exporter로 옮기면 이전 캐시와 신규 캐시가 runner filesystem에서 공존하는 문제도 없앨 수 있습니다.

```yaml
cache-from: type=gha,scope=<image repository>
cache-to: type=gha,mode=max,ignore-error=true,scope=<image repository>
```

이 설정은 GitHub Actions runtime이 제공하는 cache API를 사용합니다. 따라서 로컬 Docker Desktop에서는 `type=gha`의 동작을 검증할 수 없습니다. GitHub-hosted runner에서 실제 push 경로까지 포함해 측정한 결과는 아래에 정리했습니다. BuildKit 0.20 이상이 필요한데, `docker/setup-buildx-action`이 기본으로 내려받는 BuildKit 이미지가 이 조건을 만족합니다.

`scope`는 반드시 이미지마다 다르게 주어야 합니다. 기본 scope는 `buildkit` 하나라서, 한 저장소에서 Dockerfile 두 개를 빌드하면 서로의 캐시를 덮어씁니다. 실패했던 서비스도 데이터용 Dockerfile과 서버용 Dockerfile이 한 저장소에 있어 ECR repository 이름을 scope로 썼습니다. `ignore-error=true`는 cache export가 GitHub cache API의 rate limit 등으로 실패해도 이미지 push까지 끝난 빌드를 실패로 만들지 않기 위한 설정입니다.

`type=local`과 `actions/cache` 조합은 저장소의 Actions cache 쿼터에도 부담이 컸습니다. 캐시 key에 커밋 SHA가 들어가므로 커밋마다 전체 캐시 tar를 새로 저장합니다. 한 저장소에서는 650~816MB짜리 항목 15개가 10.78GB를 차지해 10GB 기본 쿼터를 넘긴 채 eviction이 돌고 있었고, 다른 저장소는 199MB짜리 항목 216개가 5.25GB를 차지했습니다. `type=gha`는 레이어 blob 단위로 저장하고 변하지 않은 blob은 다시 올리지 않으므로, 전환 뒤 같은 두 저장소는 각각 135개 항목 4.16GB, 59개 항목 0.98GB로 줄었습니다.

`mode=max`는 캐시 적중률에 유리하지만 intermediate layer까지 저장합니다. 커밋마다 tar 전체를 저장하던 구조에서는 `mode=max`가 쿼터를 빠르게 소진했지만, blob 단위로 중복 제거되는 `type=gha`에서는 부담이 크게 줄었으므로 `mode=max`를 유지했습니다.

## GitHub-hosted runner에서 변경 전후 비교

캐시만 비교하거나 `type=cacheonly`로 빌드하면 ECR push와 Docker 데몬 적재 비용을 검증할 수 없습니다. 따라서 기존 워크플로의 **local cache export + `load: true` + 별도 `docker push`**와 변경 워크플로의 **GHA cache export + `load: false` + BuildKit 직접 push**를 각각 새 runner에서 실행했습니다. 두 실행 모두 기존 테스트 ECR 저장소에 서로 다른 고유 tag로 이미지를 push했고, 기존 경로의 local cache 저장과 변경 경로의 GHA cache export까지 각각 측정했습니다.

같은 소스 커밋과 Dockerfile에서 고정된 seed로 압축하기 어려운 1,024MiB payload를 생성했습니다. 두 실행의 payload SHA-256은 모두 `10ab0772c72b48db208f59c1e1d6fb915f7888727d8304c4141c2375b33c8657`였고, ECR에서 확인한 이미지 크기는 모두 `1,074,070,130 bytes`였습니다. 두 runner의 이미지 버전은 `ubuntu24`의 `20260920.314.1`로 같았습니다. cache scope는 run마다 달리했고, 이전 빌드 단계의 cache hit 없이 payload를 생성한 cold build만 비교했습니다.

Buildx 설정 전부터 build, push, cache export 및 저장이 끝날 때까지 root filesystem의 `df -B1` 사용량을 약 1초 간격으로 기록했습니다. 다음의 증가는 각 runner에서 측정한 **최대 사용량에서 시작 시 사용량을 뺀 값**입니다.

| 측정값 | 변경 전: local cache + Docker load | 변경 후: GHA cache + 직접 push |
| --- | ---: | ---: |
| 시작 시 root 사용량 | 62,568,120,320 bytes | 62,568,054,784 bytes |
| 샘플링된 root 최대 사용량 | 72,590,290,944 bytes | 67,183,652,864 bytes |
| **최대 증가량** | **10,022,170,624 bytes** | **4,615,598,080 bytes** |
| 최대 사용량 시 남은 공간 | 4,280,086,528 bytes | 9,686,724,608 bytes |
| runner의 local cache 디렉터리 할당량 | 2,166,247,424 bytes | 0 bytes |
| Docker 데몬의 테스트 이미지 | 있음, 논리 크기 1,073,741,824 bytes | 없음 |
| BuildKit backing volume 할당량 | 4,371,144,704 bytes | 4,370,817,024 bytes |

변경 후 샘플링된 최대 증가량은 **5,406,572,544 bytes(약 5.04GiB), 기존 대비 약 54% 낮았습니다**. BuildKit backing volume은 양쪽 모두 약 4.37GB로 남았습니다. 반면 변경 후에는 runner의 local cache 디렉터리가 없었고, `docker image inspect`로 조회되는 테스트 이미지도 Docker 데몬에 없었습니다. 기존 경로에서는 local cache export와 `actions/cache` 저장, 변경 경로에서는 GHA cache export 완료를 각각 확인했습니다.

다만 표의 디렉터리 할당량과 Docker image의 논리 크기는 서로 다른 방식으로 측정했고 저장 영역이 겹칠 수 있습니다. 이 수치를 더해 5.04GiB 절감량의 항목별 원인으로 계산할 수는 없습니다. 약 1초 간격의 샘플링도 순간적인 실제 최고치를 보장하지 않습니다. 변경 후 GHA cache 전송에는 `217.6초`가 걸렸으므로, 디스크 사용량 감소가 cache export 시간 감소를 의미하지도 않습니다.

이번 결과는 **1,024MiB fixture를 사용한 cold build와 실제 ECR push**의 비교입니다. 실제 서비스 이미지의 성공과 warm-cache 성능까지 입증한 결과는 아니며, 실패했던 1.40GB 이미지의 결과는 다음 절에 정리했습니다.

## 적용 결과

변경은 다음 순서로 적용했습니다.

1. 로컬에서 `--load`와 `--push` 실험을 같은 Dockerfile과 소스 커밋으로 비교했습니다.
2. 공유 build 액션에 `push`와 `cache-scope` 입력을 추가하고, 두 개로 나뉘어 있던 빌드 스텝을 하나로 합쳤습니다.
3. 두 재사용 워크플로에서 별도 `docker push` 액션 호출을 제거하고 `push: 'true'`와 이미지별 `cache-scope`를 전달했습니다.
4. local cache restore 및 export를 제거하고 `type=gha`로 전환했습니다.
5. 테스트 저장소 브랜치에서 GitHub-hosted runner의 디스크, ECR 이미지 tag와 digest, scope가 없는 직접 호출의 무캐시 빌드를 확인했습니다.
6. 재사용 워크플로의 입력과 출력이 유지되는지 확인한 뒤 `v2` 태그를 갱신했습니다.

태그 갱신 당일 실패했던 서비스는 자체 워크플로를 버리고 다시 공유 워크플로(`use-cache: "true"`)로 돌아왔고, 같은 1.40GB 이미지가 성공했습니다. 빌드 로그의 액션 입력은 다음과 같았고, `importing to docker` 단계 없이 `pushing layers 115.5s done`으로 끝났습니다.

```text
push: true
load: false
cache-from: type=gha,scope=<registry>/<image>
cache-to: type=gha,mode=max,ignore-error=true,scope=<registry>/<image>
no-cache: false
provenance: false
```

## 정리

Docker 빌드의 디스크 부족은 단순히 이미지가 큰 문제만은 아닙니다. 이미지와 캐시가 빌드 과정에서 몇 번 복제되는지 확인해야 합니다.

이번 구조에서는 `load: true`를 제거하고 BuildKit이 ECR에 직접 push하도록 바꾸면 Docker 데몬 사본을 없앨 수 있습니다. 이어서 `type=gha`를 사용하면 로컬 cache 교체 과정에서 생기던 두 개의 cache 디렉터리도 runner filesystem에서 제거할 수 있습니다. 두 실패가 서로 다른 지점에서 났던 만큼, 둘 중 하나만 고쳤다면 나머지 한 경로는 계속 실패했을 것입니다.

측정값을 남기면 이미지 크기만 보는 대신, BuildKit과 Docker 데몬, 캐시 exporter가 실제로 차지하는 용량을 기준으로 CI runner 크기와 cache 정책을 결정할 수 있습니다.
