---
layout: post
title: Naver Cloud Container Registry Garbage Collection
author: theori
description:
categories: [ development, korean ]
tags: [ Naver Cloud ]
comments: true
image: assets/images/2020-10-19/01-ncr.png
---

Theori에서 개발하고 있는 사이버 보안 교육 플랫폼 [Dreamhack](https://dreamhack.io/)에서는 [NAVER CLOUD PLATFORM](https://www.ncloud.com/)을 이용해 서비스를 제공하고 있습니다.

<p align="center">
  <img src="/assets/images/2020-10-19/01-ncr.png">
</p>

이용하고 있는 여러 서비스 중 하나인 [Container Registry](https://www.ncloud.com/product/compute/containerRegistry)는 Amaozn Web Services의 S3와 유사한 [Object Storage](https://www.ncloud.com/product/storage/objectStorage)와 연계되어 Docker 컨테이너 이미지를 저장하고 관리하는 서비스입니다.

Docker 기반으로 이루어져 있는 Dreamhack 서비스 구조상 Container Registry를 CI/CD와 연동하여 빈번하게 사용하고, 그에 따라 Object Storage 사용량도 계속해서 증가하게 됐습니다.

<p align="center">
  <img src="/assets/images/2020-10-19/02-diagram.png">
</p>

예를 들어, Node.js를 기반으로 하는 한 이미지에서,

* `package.json` 파일 복사
* `npm install` 실행
* 소스 코드 복사
* 서비스 빌드

의 과정을 거치게 되면 네 개의 레이어가 생성되게 되는데, 소스 코드를 변경하여 새로운 이미지를 빌드하면 기존의 레이어 두 개 위에 두 개의 레이어가 새롭게 생성되고, 이전의 레이어 두 개는 어디에서도 사용되지 않는 채로 남게 됩니다. 이러한 과정이 여러 번 반복되면 사용되지 않는 레이어가 계속해서 쌓여만 가게 됩니다.

Dreamhack의 경우, 코드 베이스의 변경과 빌드 결과물 등이 쌓이고 쌓인 결과 1년간의 사용량이 20GB를 넘기는 상황에 도달하고 말았습니다. 다행히도 이 정도의 용량이 엄청난 금전적 비용을 발생시키지는 않지만, 사용하지도 않고 저장해둘 필요도 없는 데이터가 쌓이는 것이 좋은 상황은 아닙니다.

다행히도 Docker에서 제공하는 직접 호스팅 가능한 Registry 이미지에서는 이런 식으로 사용하지 않는 데이터를 삭제하는 [garbage collection](https://docs.docker.com/registry/garbage-collection/) 기능이 존재합니다. 하지만 네이버 클라우드의 Container Registry가 이 기능을 지원하는지에 대한 여부는 찾을 수 없었습니다. 네이버 클라우드에서 제공하는 문서에서도 언급되어 있지 않았고, 인터넷 검색을 통해서도 정보를 찾을 수가 없었습니다.

남들이 정리해둔 방법을 찾을 수 없다면 직접 부딪혀보고 해결책을 찾아나갈 수밖에 없습니다.


## 직접 삭제해보자

처음에 떠오른 방법은 Bucket에 저장된 레이어 목록을 어떻게든 가져온 뒤, Registry에 저장된 이미지들이 사용하는 레이어가 아닌 것들을 제외하고 전부 수동으로 삭제하는 방법이었습니다.

<p align="center">
  <img src="/assets/images/2020-10-19/03-bucket.png">
</p>

Bucket 안의 `docker/registry/v2` 디렉토리는 `blobs`와 `repositories` 두 개의 디렉토리로 구성되어 있습니다.

<p align="center">
  <img src="/assets/images/2020-10-19/04-bucket-repositories.png">
</p>

먼저 `repositories` 디렉토리 밑에는 Registry에 push 된 이미지들이 하위 디렉토리로 존재하고, 그 밑으로 이미지의 태그나 각 레이어의 reference를 저장하는 수많은 파일이 존재합니다.

<p align="center">
  <img src="/assets/images/2020-10-19/05-bucket-blobs.png">
</p>

그리고 `blobs` 디렉토리 밑에는 각 reference에 실제 데이터들이 저장되어 있습니다.

하나의 이미지를 구성하기 위해서는,

- `repositories/<image name>/_manifests/tags/<image tag>/current/link` 파일에 저장된 SHA256 해시를 가지고,
- `blobs/sha256/<hash[0:2]>/<hash>/data` 파일에서 해당 이미지의 manifest를 얻어온 뒤,
- manifest에 저장된 config나 layer들의 SHA256 해시를 가지고,
- `blobs/sha256/<hash[0:2]>/<hash>/data` 파일에서 실제 데이터를 받아오는...

여러 단계를 거쳐야 합니다.

<p align="center">
  <img src="/assets/images/2020-10-19/06-pexels-2696299.jpg">
</p>

뭔가 복잡합니다. 분명 더 쉬운 방법이 있을 것 같습니다.


## Registry API를 써보자

조금 더 검색해보니 Docker Registry에는 [HTTP API](https://docs.docker.com/registry/spec/api/)가 있습니다. 네이버 클라우드의 Container Registry가 일반적인 Docker에서 작동하려면 이 표준 API에 맞춰서 구현되어있을게 분명합니다.

```shell
$ curl --user [CENSORED] https://[CENSORED].kr.ncr.ntruss.com/v2/_catalog
{
  "repositories": [
    "service-alpha",
    "service-beta",
    "service-charlie",
    ...
  ]
}
```

이미지 목록을 받아올 수 있고,

```shell
$ curl --user [CENSORED] https://[CENSORED].kr.ncr.ntruss.com/v2/service-alpha/tags/list
{
  "name": "service-alpha",
  "tags": [
    "latest",
    ...
  ]
}
```

이미지의 태그도 받아올 수 있고,

```shell
$ curl --user [CENSORED] https://[CENSORED].kr.ncr.ntruss.com/v2/service-alpha/manifests/latest
{
   "schemaVersion": 1,
   "name": "service-alpha",
   "tag": "latest",
   "architecture": "amd64",
   "fsLayers": [
      {
         "blobSum": "sha256:a3ed95ca..."
      },
      {
         "blobSum": "sha256:f7537e82..."
      },
      {
         "blobSum": "sha256:bd3af235..."
      },
      ...
   ],
   "history": [ ... ],
   "signatures": [ ... ]
}
```

이미지의 레이어들도 가져올 수 있고,

```shell
$ curl --head --user [CENSORED] https://[CENSORED].kr.ncr.ntruss.com/v2/dreamhack_nuxt/blobs/sha256:a3ed95ca...
HTTP/2 307
location: https://kr.object.ncloudstorage.com/[CENSORED]/docker/registry/v2/blobs/sha256/a3/a3ed95ca.../data
docker-distribution-api-version: registry/2.0
```

각 레이어의 실제 데이터도 받아올 수 있습니다.

뭔가 나오기는 하지만... 이래서야 아까와 다른 게 없습니다. 분명 아주 간단하고 깔끔한 방법이 있을 것 같습니다.


## 직접 Registry를 돌려보자

이 시점에서 한가지 생각이 머리를 스쳤습니다.

<p align="center">
  <img src="/assets/images/2020-10-19/07-pexels-874242.jpg">
</p>

> 네이버 클라우드의 Container Registry가 Docker Registry HTTP API를 똑같이 지원한다면... 사실 Container Registry는 Docker에서 제공하는 Registry 바이너리를 똑같이 쓰고 있는 게 아닐까?

그리하여, [문서](https://docs.docker.com/registry/)에 나온 대로 일단 로컬에서 Registry를 실행 시켜 어떻게 돌아가는지 확인해봤습니다.

```shell
$ docker run -d -p 5000:5000 --name registry-test registry:2
Unable to find image 'registry:2' locally
[...]
6d731d373334

$ docker ps
CONTAINER ID    IMAGE         PORTS                     NAMES
6d731d373334    registry:2    0.0.0.0:5000->5000/tcp    registry-test

$ curl http://localhost:5000/v2/_catalog
{
  "repositories": []
}
```

특별한 설정 없이 바로 Registry가 실행되고, 똑같은 API를 사용할 수 있습니다. 이제 이미지 하나를 가져다가 여기다 push 해봤습니다.

```shell
$ docker image tag registry:2 localhost:5000/registry

$ docker push localhost:5000/registry
The push refers to repository [localhost:5000/registry]
[...]

$ curl http://localhost:5000/v2/_catalog
{
  "repositories": [
    "registry"
  ]
}
```

정상적으로 push가 되고, API를 통해서도 push 한 이미지를 확인할 수 있습니다. 그렇다면 이 이미지는 파일 시스템 내에서 어떻게 저장되어 있는 걸까요?

```shell
$ docker exec registry-test find /var/lib/registry -maxdepth 6 -type d
/var/lib/registry
/var/lib/registry/docker
/var/lib/registry/docker/registry
/var/lib/registry/docker/registry/v2
/var/lib/registry/docker/registry/v2/repositories
/var/lib/registry/docker/registry/v2/repositories/registry
/var/lib/registry/docker/registry/v2/repositories/registry/_layers
/var/lib/registry/docker/registry/v2/repositories/registry/_manifests
/var/lib/registry/docker/registry/v2/repositories/registry/_uploads
/var/lib/registry/docker/registry/v2/blobs
/var/lib/registry/docker/registry/v2/blobs/sha256
/var/lib/registry/docker/registry/v2/blobs/sha256/47
/var/lib/registry/docker/registry/v2/blobs/sha256/cb
/var/lib/registry/docker/registry/v2/blobs/sha256/e0
/var/lib/registry/docker/registry/v2/blobs/sha256/46
/var/lib/registry/docker/registry/v2/blobs/sha256/c1
/var/lib/registry/docker/registry/v2/blobs/sha256/3d
/var/lib/registry/docker/registry/v2/blobs/sha256/2d
```

낯이 익습니다. 아까 Bucket에서 봤던 디렉토리 구조 그대로입니다.

이 시점에서 네이버 클라우드의 Container Registry 서비스는 Docker에서 제공하는 Registry 바이너리를 사용하고 있다고 확신했습니다. 물론 진짜인지는 네이버 클라우드만이 알고 있겠지만, 이 정도면 Docker Registry의 garbage collect 기능을 사용해도 무방하다고 생각했습니다.

남은 것은 Registry의 storage backend를 네이버 클라우드의 Object Storage로 설정하여 로컬에서 Registry를 실행한 뒤, 로컬에서 garbage collection을 실행하는 것뿐입니다.


## Garbage Collection을 해보자

먼저, [설정 문서](https://docs.docker.com/registry/configuration/)에 맞춰서 Registry 설정 파일을 만들어야 합니다. 환경 변수를 이용해서 설정을 고칠 수도 있지만, 별도의 설정 파일을 만든 뒤 `docker run`의 `-v` 옵션을 통해 설정 파일을 통째로 바꾸기로 했습니다. 문서에서 제공하는 예제 설정 파일을 기반으로 `storage` 섹션을 고쳐줍니다.

```yaml
# config.yml

version: 0.1
log:
  fields:
    service: registry
storage:
  s3:
    accesskey: [NAVER CLOUD ACCESS KEY]
    secretkey: [NAVER CLOUD SECRET KEY]
    region: kr
    regionendpoint: https://kr.object.ncloudstorage.com
    bucket: [NAVER CLOUD OBJECT STORAGE BUCKET NAME]
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

Object Storage는 S3와 호환되는 API를 제공하고 있으니 S3 driver를 사용해도 무방합니다. [S3 driver 문서](https://github.com/docker/docker.github.io/blob/master/registry/storage-drivers/s3.md)에 맞춰서 `regionendpoint`만 적절히 설정해주면 `region`은 무시됩니다.

```shell
$ docker run -d -p 5000:5000 --name registry-test -v `pwd`/config.tml:/etc/docker/registry/config.yml registry:2
b5e180d33184

$ docker ps
CONTAINER ID    IMAGE         PORTS                     NAMES
b5e180d33184    registry:2    0.0.0.0:5000->5000/tcp    registry-test

$ curl http://localhost:5000/v2/_catalog
{
  "repositories": [
    "service-alpha",
    "service-beta",
    "service-charlie",
    ...
  ]
}
```

설정이 정상적으로 됐다면 별 탈 없이 로컬 Registry가 켜지고, Object Storage에 저장된 이미지들을 확인할 수 있습니다. 이젠 정말로 garbage collection만 실행하면 됩니다.

**주의: [Docker 문서](https://docs.docker.com/registry/garbage-collection/)에서는 Registry에 쓰기 작업이 이루어지는 도중에 garbage collection이 실행될 경우 문제가 생길 수 있음을 안내하고 있습니다. 아래의 내용을 따라 할 경우에도 동일하게 주의를 기울여주세요.**

<p align="center">
  <img src="/assets/images/2020-10-19/08-before.png">
</p>

현재 Dreamhack에서 사용하는 Container Registry는 이미 garbage collection을 한번 실행한 상태이기 때문에 위에서 언급된 20GB에 달하는 용량은 아니지만, 그 이후로도 시간이 흘렀기 때문에 한번 여기서 garbage collection을 실행해보도록 하겠습니다...만 먼저 시뮬레이션을 해보기로 했습니다.

```shell
$ docker exec registry-test bin/registry garbage-collect --dry-run /etc/docker/registry/config.yml
[...]
[...]
[...]

213 blobs marked, 1 blobs and 0 manifests eligible for deletion
blob eligible for deletion: sha256:a3ed95ca...
```

`--dry-run` 옵션을 사용하면 실제로 데이터가 삭제되지 않고 어떤 것들이 삭제되는지만 출력됩니다. 뭔가 하나밖에 없는 것이 이상하기는 하지만, 진짜로 실행해보고 얼마나 용량이 줄어드는지 확인해보겠습니다.

```shell
$ docker exec registry-test bin/registry garbage-collect /etc/docker/registry/config.yml
[...]
[...]
[...]

213 blobs marked, 1 blobs and 0 manifests eligible for deletion
blob eligible for deletion: sha256:a3ed95ca...
level=info msg="Deleting blob: /docker/registry/v2/blobs/sha256/a3/a3ed95ca..."
```

하나의 blob이 삭제됐고, 줄어든 용량은...

<p align="center">
  <img src="/assets/images/2020-10-19/08-before.png">
</p>

<p align="center">
  <img src="/assets/images/2020-10-19/09-pexels-3777553.jpg">
</p>

뭔가 이상합니다. ~~글에 사용된 이미지는 방금과 동일한 이미지이긴 하지만 실제로~~ 전혀 변화가 없습니다. 빠뜨린 옵션이 있는지 확인해봤습니다.

```shell
$ docker exec registry-test bin/registry garbage-collect --help
`garbage-collect` deletes layers not referenced by any manifests

Usage:
  registry garbage-collect <config> [flags]
Flags:
  -m, --delete-untagged=false: delete manifests that are not currently referenced via tag
  -d, --dry-run=false: do everything except remove the blobs
  -h, --help=false: help for garbage-collect
```

`--delete-untagged` 라는 옵션이 있었습니다. 문서에는 나와 있지 않지만 분명히 존재하는 옵션이고, 현재 Dreamhack에서 사용하는 패턴상 이 태그가 필요합니다. 이 옵션을 넣고 다시 한번 garbage collection을 실행해봤습니다.

```shell
# 똑같이 --dry-run을 사용할 수 있습니다.
$ docker exec registry-test bin/registry garbage-collect --delete-untagged /etc/docker/registry/config.yml
[...]
[...]
[...]

185 blobs marked, 28 blobs and 5 manifests eligible for deletion
blob eligible for deletion: sha256:fffa3942...
blob eligible for deletion: sha256:8771f681...
blob eligible for deletion: sha256:6d9d6122...
[...]
```

아까는 삭제 대상에 포함되지 않았던 데이터들이 대상에 포함되고, 실제로 삭제가 됐습니다.

<p align="center">
  <img src="/assets/images/2020-10-19/10-after.png">
</p>

콘솔에서 출력되는 사용량 또한 눈에 띄게 줄어들었습니다.


## 마치며

네이버 클라우드에서 제공하는 Container Registry 서비스, 그리고 Docker Registry 구조에 대해서 알아볼 수 있었습니다.

다른 곳에서 찾아볼 수 없는 내용이었기 때문에 빙 돌아가긴 했지만, 최종적으로는 원했던 것을 달성했고, 네이버 클라우드를 사용하는 다른 곳이나 비슷한 Container Registry 서비스를 이용하는 곳에서도 유용하게 사용될 수 있는 하나의 가이드를 만들게 됐습니다. 이 가이드와 팁들이 모두에게 유용할 수 있기를 바랍니다.

## References

* [NAVER CLOUD PLATFORM](https://www.ncloud.com/)
* [Docker Regsitry](https://docs.docker.com/registry/)
