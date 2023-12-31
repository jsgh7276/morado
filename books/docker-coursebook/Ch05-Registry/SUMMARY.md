# Ch05. Registry

## 1. 레지스트리, 레포지토리, 이미지 태그
* 도커의 핵심은 공유의 용이성
  * 환경 차이가 사라진다
* 도커 허브: 가장 유명한 레지스트리
  * 매우 많은 이미지들이 있다
  * 도커 엔진 디폴트 레지스트리
* 이미지 이름의 형식
  * docker.io/diamol/golang:latest
  * 순서대로 레지스트리 / 작성자 / 레포(앱) 이름 / 이미지 태그
  * 해당 이미지는 모두 내려받을 수 있지만 diamol 단체 소속만 push할 수 있다
  * 대기업에서는 별도의 레지스트리를 가지고 있음
  * 태그를 붙여서 버전간 구별할 수 있도록 할것

## 2. 푸시하기
* 절차
  * 레지스트리에 로그인
  * 이미지에 계정명을 포함하는 이미지 참조 붙이기
* 로그인: `docker login --username $dockerId`
* 참조 붙이기: `docker image tag image-gallery $dockerId/image-gallery:v1`
* 한 이미지에 참조는 여러 개일 수 있다.
    ```
    > docker image ls --filter reference=image-gallery --filter reference='*/image-gallery'
    REPOSITORY               TAG       IMAGE ID       CREATED       SIZE
    image-gallery            latest    de60324cb39a   3 weeks ago   26.2MB
    jsgh7276/image-gallery   v1        de60324cb39a   3 weeks ago   26.2MB
    ```
* 이미지 푸시하기: `docker image push $dockerId/image-gallery:v1`
  * 레지스트리에 존재하는 이미지는 가져다 씀. 최적화가 중요하다
  * 확인: https://hub.docker.com/r/jsgh7276/image-gallery/tags 접속

## 3. 레지스트리 운영
* 로컬 레지스트리가 있으면 여러 장점이 있다.
* `docker container run -d -p 5000:5000 --restart always diamol/registry`
  * `--restart`: 도커 재시작시 컨테이너 재시작
* 이제 `localhost:5000`이 레지스트리 도메인 네임이 되었다.
* `docker image tag image-gallery localhost:5000/gallery/ui:v1`
* 소규모 팀에서는 충분히 유용함
* 로컬 레지스트리를 허용하려면 insecure-registries를 설정에 추가해주어야 한다
* `docker image push localhost:5000/gallery/ui:v1`

## 4. 효율적인 이미지 태그
* 기본적으로 `major.minor.patch` 형식을 따른다
  * `docker image tag image-gallery localhost:5000/gallery/ui:2.1`
* 태그는 여러 개가 연결 가능하다. 즉 별명 붙이기 가능하다
  * `latest` 태그는 릴리스에 따라 계속 옮겨다닌다
  * `3` `3.0` 등은 `3.0.12`의 별칭

## 5. 골든 이미지
* 검증된 퍼블리셔: MS, 오라클, IBM 등 대기업
* 공식 이미지: 오픈소스 소프트웨어. 도커와 개발주체가 함께 관리
* 골든 이미지: 공식 이미지 기반으로 커스터마이즈한 이미지. 신뢰도와 커스터마이징 두 마리 토끼
* 골든 이미지를 만드는 도커파일 예시
  ```dockerfile
  FROM mcr.microsoft.com/dotnet/core/sdk:3.0.100
  
  LABEL framework="dotnet"
  LABEL version="3.0"
  LABEL description=".NET Core 3.0 SDK"
  LABEL owner="golden-images@sixeyed.com"
  
  WORKDIR src
  COPY global.json .
  ```
  * `LABEL` 인스트럭션으로 이미지 메타데이터 관리






