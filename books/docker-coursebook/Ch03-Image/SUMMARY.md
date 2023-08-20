# Ch03. 이미지

## 1. 도커 허브 이미지 사용하기
* 이미지 내려받기: `docker image pull {이미지명}`
  * 내려받는 과정에서 여러 개의 파일을 받음 (이미지 레이어)
* 레지스트리: 이미지 제공 저장소
  * 도커 허브
* `docker container run -d --name web-ping diamol/ch03-web-ping`
  * `--name` 태그로 컨테이너에 이름 붙일 수 있음
  * `-d`는 detach 
* `docker container logs web-ping`
* `--env` 태그로 환경변수를 설정할 수 있다
  * `docker container run --env TARGET=google.com diamol/ch03-web-ping`
  * 환경변수로 컨테이너의 동작을 유연하게 바꿀 수 있도록 이미지를 작성하는 것이 바람직하다
  * 호스트 컴퓨터의 환경변수와는 독립적이다

## 2. Dockerfile
* 애플리케이션을 패키징하기 위한 스크립트
* 인스트럭션들로 구성되어 있으며 인스트럭션들의 실행 결과로 이미지가 만들어진다
* 예제
  ```dockerfile
    FROM diamol/node
    
    ENV TARGET="blog.sixeyed.com"
    ENV METHOD="HEAD"
    ENV INTERVAL="3000"
    
    WORKDIR /web-ping
    COPY app.js .
    
    CMD ["node", "/web-ping/app.js"]
  ```
  * `FROM`: 모든 이미지는 다른 이미지로부터 출발한다. 시작점 이미지 지정
  * `ENV`: 환경 변수 설정
  * `WORKDIR`: 디렉토리를 만들고 작업 디렉토리로 설정
  * `COPY`: source / dest 순으로 지정
  * `CMD`: 컨테이너 실행했을 때 실행할 명령 지정

## 3. 이미지 빌드
* 모든 필요한 파일이 갖춰진 디렉토리에서 다음을 실행
  * `docker image build --tag {이미지명} {컨텍스트}`
  * 컨텍스트는 포함할 파일의 경로를 뜻한다
* `docker image build --tag web-ping .`
* `docker image ls 'w*'`
* `docker container run -e TARGET=docker.com -e INTERVAL=5000 web-ping`
