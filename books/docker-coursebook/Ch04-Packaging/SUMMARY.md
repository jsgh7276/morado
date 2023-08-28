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
  * 멀티 스테이지 빌드
  * 각 빌드 단계는 `FROM` 인스트럭션으로 시작한다
  * `AS`로 alias 설정할 수 있다
  * 최종 단계의 산출물은 마지막 단계의 내용물을 담은 도커 이미지
  * 각 단계는 독립적이지만 앞 단계에서 파일 복사해올 수 있다
  * `RUN`: 빌드 중 컨테이너 안에서 명령 실행한 후 결과를 이미지 레이어에 저장
    * `FROM`으로 지정한 베이스 이미지에서 실행 가능해야
  * 각 단계는 격리되어 있다.
  * 마지막 산출물은 명시적으로 복사된 것만 포함할 수 있다
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
  * 포함시키고 싶으면 앞 단계에서 복사해와
