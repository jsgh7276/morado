# Ch04. Packaging

## 1. 빌드 서버를 대체하는 Dockerfile
* 빌드는 절차가 귀찮고 버전이 안맞으면 실패하는 등 성가신 부분이 많다
* 도커를 사용하면 빌드 툴 체인을 한 번에 패키징할 수 있다
* 모든 도구를 Dockerfile을 통해 배포하고 컴파일하면 편하다.
* 예제
    ```dockerfile
    FROM diamol/base AS build-stage
    RUN echo 'Building...' > /build.txt
    
    FROM diamol/base AS test-stage
    COPY --from=build-stage /build.txt /build.txt
    RUN echo 'Testing...' >> /build.txt
    
    FROM diamol/base
    COPY --from=test-stage /build.txt /build.txt
    CMD cat /build.txt
    ```
  * 위 도커파일은 멀티 스테이지 빌드이다.
  * 각 빌드 단계는 `FROM` 인스트럭션으로 시작한다
  * `AS`로 alias 설정할 수 있다
  * 최종 단계의 산출물은 마지막 단계의 내용물을 담은 도커 이미지
  * 각 단계는 기본적으로 독립적이지만, 앞 단계에서 파일 복사해올 수 있다
  * `RUN`: 빌드 중 컨테이너 안에서 명령 실행한 후 결과를 이미지 레이어에 저장
    * `FROM`으로 지정한 베이스 이미지에서 실행 가능해야
  * 각 단계는 격리되어 있다.
  * 마지막 산출물에는 명시적으로 복사된 것만 포함된다.
  * 하나라도 실패하면 전체 빌드 실패
* 이러한 방식으로 자바 maven 빌드, 빌드 결과물 복사 및 테스트 수행 등을 한 번에 할 수 있다
* 즉 도커만 갖춰지면 어디서든 애플리케이션을 빌드/실행할 수 있다
  * 대부분의 유명 애플리케이션의 경우 이미 도커 허브에 공식 이미지가 제공되고 있다

## 2. 빌드 실전 예제
* 예제
    ```dockerfile
    FROM diamol/maven AS builder
    
    WORKDIR /usr/src/iotd
    COPY pom.xml .
    RUN mvn -B dependency:go-offline
    
    COPY . .
    RUN mvn package
    
    # app
    FROM diamol/openjdk
    
    WORKDIR /app
    COPY --from=builder /usr/src/iotd/target/iotd-service-0.1.0.jar .
    
    EXPOSE 80
    ENTRYPOINT ["java", "-jar", "/app/iotd-service-0.1.0.jar"]
    ```
  * 이미지에 작업 디렉토리(iotd)를 만든 후 로컬 pom.xml를 이미지로 복사한다
  * builder 단계
    * `RUN mvn ...` 명령은 pom.xml 파일이 변경된 경우에만 실행된다 (아니면 캐시 이용)
    * `COPY . .`: 현재 작업중인 로컬 디렉토리의 파일 & 디렉토리를 모두 이미지의 작업 디렉토리로 복사
    * `mvn package`로 jar 파일 출력됨
    * 이 단계가 끝나면 컴파일된 애플리케이션이 해당 단계의 파일 시스템에 만들어짐
  * 마지막(openjdk) 단계
    * 이미지 작업 디렉토리를 지정하고 앞 단계의 jar 파일을 복사해준다
    * `EXPOSE`로 포트를 외부로 공개한다
    * `ENTRYPOINT`는 `CMD`와 같은 역할
* `docker network create nat`
  * 컨테이너 간 통신에 사용되는 도커 네트워크 생성
* `docker container run --name
  iotd -d -p 800:80 --network nat image-of-the-day`
  * --network 로 네트워크 지정할 수 있다
* 최종 생성된 이미지에 빌드 도구는 포함되지 않는다 -> 효율적
  * 마지막 단계 콘텐츠만 이미지에 포함된다.
  * 포함시키고 싶으면 앞 단계에서 복사해와야 한다.

## 3. Node.js 빌드 예제
* Node.js는 인터프리터 언어라서 자바와 달리 별도의 컴파일 절차 필요가 없다
* 자바스크립트로 구현됨
* 예제
  ```dockerfile
  FROM diamol/node AS builder
  
  WORKDIR /src
  COPY src/package.json .
  
  RUN npm install
  
  # app
  FROM diamol/node
  
  EXPOSE 80
  CMD ["node", "server.js"]
  
  WORKDIR /app
  COPY --from=builder /src/node_modules/ /app/node_modules/
  COPY src/ .
  ```
  * `diamol/node`는 npm과 node가 설치된 이미지
  * 의문: node server.js 커맨드가 왜 실행되지
* `docker image build -t access-log .`
* `docker container run --name accesslog -d -p 801:80 --network nat access-log`
* node 10.16으로 구동되지만, 이러한 사실에 신경쓸 필요가 없다는 것이 주목할 지점이다.

## 4. Go 예제
* Go: native binary로 컴파일되는 크로스 플랫폼 언어
  * 별도 런타임이 필요하지 않다는 장점
  * 도커도 Go로 구현됨
* 예제
  ```dockerfile
  FROM diamol/golang AS builder
  
  COPY main.go .
  RUN go build -o /server
  
  # app
  FROM diamol/base
  
  ENV IMAGE_API_URL="http://iotd/image" \
  ACCESS_API_URL="http://accesslog/access-log"
  CMD ["/web/server"]
  
  WORKDIR web
  COPY index.html .
  COPY --from=builder /server .
  RUN chmod +x server
  ```
  * 빌드 단계: go 컴파일
  * 애플리케이션 단계: 바이너리와 html 파일을 복사
* `docker image build -t image-gallery`
* `docker image ls -f reference=diamol/golang -f reference=image-gallery`
  * `-f` 태그로 결과에 필터 걸 수 있다
  * diamol/golang 이미지보다 빌드된 go 애플리케이션 포함한 이미지가 크기가 더 작다
* `docker container run -d -p 802:80 --network nat image-gallery`
* 지금까지 한 것
  * Go로 구현된 웹 애플리케이션이 Java API를 호출하여 이미지를 얻어온 다음 Node.js API에 로그를 남긴다
  * 이때까지 각 프로그램의 소스코드와 도커만 필요했으며 다른 도구는 필요없었다

## 5. 멀티 스테이지 도커파일
* 장점
  * 표준화: os, 환경과 상관없이 빌드에 성공할 수 있음
  * 성능 향상: 레이어 캐시로 인해 소요 시간 단축
  * 공간 효율: 빌드 도구가 최종 산출물에 포함되지 않게 하여 용량 줄일 수 있음 (curl 등)

## 6. 연습 문제
다음 Dockerfile을 최적화하라.
```dockerfile
FROM diamol/golang

WORKDIR web
COPY index.html .
COPY main.go .

RUN go build -o /web/server
RUN chmod +x /web/server

CMD ["/web/server"]
ENV USER=sixeyed
EXPOSE 80
```

* `docker image build -t ch04-before` -> 736MB
* 최적화 후:


```dockerfile
FROM diamol/golang AS builder
COPY main.go .
RUN go build -o /server

FROM diamol/base
ENV USER=sixeyed
WORKDIR web
COPY --from=builder /server .
CMD ["/web/server"]
EXPOSE 80
RUN chmod +x /web/server

COPY index.html .
```
* 멀티 스테이지로 나누어서 앞 단계에서 빌드를 모두 진행하고, 빌드 도구는 최종 산출물에 포함되지 않도록 한다
* `COPY index.html . ` 부분을 맨 밑으로 내려서 수정해도 한 단계만 재실행되도록 한다


