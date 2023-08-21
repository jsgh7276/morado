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

## 4. 이미지 레이어
* 이미지에는 다음이 포함되어 있다.
  * 빌드 시 포함시킨 모든 파일
  * 메타데이터 및 빌드 이력
* `docker image history web-ping`: 한 줄마다 한 레이어 정보 출력
  * `CREATED BY` 칼럼은 인스트럭션을 나타낸다
  * 인스트럭션 : 이미지 레이어는 1대 1 관계이다
  * 즉, **인스트럭션 한 줄은 이미지 레이어 하나**에 대응된다
* 도커 이미지는 **이미지 레이어가 모인 논리적 대상**이다
  * 레이어는 도커 엔진에 물리적으로 저장된 파일이다
  * 기반인 이미지(위 예시의 경우 `diamol/node`)와 같은 레이어를 공유한다
* `docker image ls`로 용량 확인할 수 있다
  * `SIZE`는 논리적 용량이지 실제 용량이 아니다. (기반 이미지 레이어는 도커 엔진에서 캐시로 공유된다.)
  * 이는 `docker system df`로 확인 가능하다.
* 이미지 레이어는 읽기 전용이다 (수정 불가)
  * 다른 이미지에서 가져다 쓸 수 있으므로 읽기 전용이어야 한다.

## 5. 이미지 레이어 캐시 활용
* 파일 하나만 살짝 수정하고 위에서 빌드했던 `web-ping`을 다시 빌드하면
  * `docker image build -t web-ping:v2 .`
  * 이미 기존에 있던 이미지 레이어와 같다면 인스트럭션은 실행되지 않는다. (cache hit)
  * 변경사항이 있다면 해당 인스트럭션부터는 쭉 실행된다. (cache miss)
* 따라서, **잘 변경되지 않는 인스트럭션을 앞부분, 변경되는 것을 뒷부분**에 배치해야 바람직하다.
* 예시의 CMD 인스트럭션은 자주 수정되지 않으므로 앞으로 땡기는 것이 낫다
  ```dockerfile
  FROM diamol/node
  
  CMD ["node", "/web-ping/app.js"]
  
  ENV TARGET="blog.sixeyed.com" \
      METHOD="HEAD" \
      INTERVAL="3000"
  
  WORKDIR /web-ping
  COPY app.js .
  ```

## 6. 연습문제
* `docker image pull diamol/ch03-lab`
* `docker container run -it diamol/ch03-lab`
  * `echo jslee >> ch03.txt`
  * vi로 수정하는 것은 이력에 남지 않는듯 하다
* `docker container commit` 명령어로 컨테이너의 수정사항을 반영하여 이미지를 빌드할 수 있다
* `docker container commit 1b ch03-solution:v1`
* `docker image ls` 하면
  * `ch03-solution              v1         bc021698fc7d   4 seconds ago        6.89MB`와 같이 잘 나온다.
* `docker run ch03-solution cat ch03.txt` 하면 바뀐 내용이 출력된다.
