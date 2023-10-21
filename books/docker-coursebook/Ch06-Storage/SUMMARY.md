# Ch06. Storage

## 1. 컨테이너 데이터가 사라지는 이유
* stateless 애플리케이션이면 필요 없겠지만, 대부분의 애플리케이션은 stateful하며 스토리지가 필요함
* 모든 컨테이너는 독립된 파일 시스템을 가지며 서로 영향 주지 않는다
* 랜덤 넘버를 파일 시스템에 기록하는 컨테이너 2개
  ```shell
    docker container run --name rn1 diamol/ch06-random-number
    docker container run --name rn2 diamol/ch06-random-number
  ```
  * 같은 이미지로부터 실행되었으나 파일 시스템의 내용은 서로 다르다
  * 컨테이너 실행은 종료돼도 파일 시스템은 삭제되지 않는다
* `docker container cp`로 컨테이너, 로컬 컴퓨터간 파일 복사 가능
  ```shell
  docker container cp rn1:/random/number.txt number1.txt
  docker container cp rn2:/random/number.txt number2.txt
  ```
  * 두 파일 내용 다르다, 즉 독립적이다.
* 레이어의 종류
  * 이미지 레이어: 모든 컨테이너가 공유, 읽기 전용
  * 기록 가능 레이어: 컨테이너마다 다르다. 생애주기도 컨테이너와 동일
* 기록 가능 레이어에서 이미지 레이어의 파일을 수정할 수도 있다.
  * 기록 가능 레이어에 복사해온 이후 파일을 수정한다.
  * 수정하는 예제
    ```shell
    docker container run --name f1 diamol/ch06-file-display
    ## 위 커맨드에서는 이미지 레이어에 포함된 input.txt가 출력된다.
    echo "http://eltonstoneman.com" > url.txt
    docker container cp url.txt f1:/input.txt
    ## 컨테이너 레이어가 이미지 레이어의 input.txt를 덮어쓸 수 있다.
    docker container start --attach f1
    ## 바뀐 내용이 출력된다.
    ```
  * 마찬가지로 다른 컨테이너/이미지에는 영향 미치지 않는다
  * 해당 컨테이너 사라지면 변경도 사라짐
* 그렇다면 어떻게 stateful한 애플리케이션을 실행할 수 있는가?
  * 컨테이너는 밥 먹듯이 사라지는데
  * 도커 볼륨과 마운트를 이용하면 된다.

## 2. 볼륨
* 볼륨: 스토리지를 다루는 단위
  * 일종의 USB 메모리 비슷한 것
  * 별도로 존재하며 컨테이너와는 별도 생애주기
  * 컨테이너에 연결할 수 있다.
* 사용하는 방법
  * 수동으로 직접 생성
  * or Dockerfile에서 VOLUME 인스트럭션 사용
    * `VOLUME <target-directory>` 형식
* 볼륨으로 지정된 디렉토리는 볼륨에 영구적으로 저장된다.
  ```shell
  docker container run --name todo1 -d -p 8010:80 diamol/ch06-todo-list
  ## Dockerfile 스크립트에 의해 볼륨이 생성되고, 컨테이너에 연결된다
  docker container inspect --format '{{.Mounts}}' todo1
  ## 볼륨 ID와 호스트 컴퓨터상 경로, 컨테이너 파일 시스템상 경로가 출력됨
  docker volume ls
  ## 볼륨은 도커에서 이미지, 컨테이너와 동급인 요소이다.
  ```
* 해당 이미지로 컨테이너를 새로 생성하면 새로운 볼륨이 생성된다.
  * 하지만 기존의 볼륨을 연결시킬 수도 있다.
  ```shell
  # 이 컨테이너를 실행하면 볼륨을 생성한다
  docker container run --name todo2 -d diamol/ch06-todo-list
  docker container exec todo2 ls /data  ## 비어있다.
  
  # 이 컨테이너는 todo1의 볼륨을 공유한다
  docker container run -d --name t3 --volumes-from todo1 diamol/ch06-todo-list
  docker container exec t3 ls /data  ## DB가 확인된다.
  ```
* 위의 예시에서는 이미지의 스크립트에서 볼륨을 생성했지만, 따로 볼륨을 생성 후 명시적으로 관리하는 것이 여러 면에서 낫다.
  ```shell
  ## 볼륨 생성
  docker volume create todo-list
  
  ## 생성된 볼륨을 연결하여 실행
  docker container run -d -p 8011:80 -v todo-list:/data --name todo-v1 diamol/ch06-todo-list
  
  ## 기존 컨테이너 삭제
  docker container rm -f todo-v1
  
  ## 새로운 컨테이너 실행, 데이터가 유지되는 것을 볼 수 있다.
  docker container run -d -p 8011:80 -v todo-list:/data --name todo-v2 diamol/ch06-todo-list:v2
  ```
  * 볼륨은 생애주기가 별도라는 것을 확인할 수 있다.
  * 앱이 업데이트 되더라도 데이터 유지 가능
* 도커파일 인스트럭션으로 사용할 경우 식별자가 무작위이므로 유의해야 함 (기억해놓아야 함)
  * 볼륨을 따로 명시적으로 생성하는 것이 좋다
* 컨테이너 실행시 --volume 플래그로 명시하면 이미지에 볼륨이 정의되어 있더라도 생성되지는 않는다.


## 3. 파일 시스템 마운트
* 볼륨의 장점: 컨테이너와 생애주기를 분리하면서도 도커 사용 방식대로 스토리지를 다룰 수 있다
* 바인드 마운트: 보다 직접적으로 호스트 스토리지를 연결
  * 파일 시스템 디렉토리를 컨테이너 파일 시스템의 디렉토리로 만든다
  * 호스트 <-> 컨테이너 간 파일 직접 접근이 가능해진다
  * `docker container run --mount type=bind,source=./dockerstudy-db,target=/data
    -d -p 8012:80 diamol/ch06-todo-list`
  * /dockerstudy-db 호스트 디렉토리가 컨테이너에 /data 로 매핑되었다
* 바인드 마운트는 양방향이다.
* 호스트 파일에 접근해야 하므로 도커파일 상에서 권한을 상승시켜주어야 한다.
* 마운트된 디렉토리를 통해 앱 설정을 조정하는 예시
  ```shell
  docker container run --name todo-configured -d -p 8013:80 --mount \
    type=bind,source=./config,target=/app/config,readonly diamol/ch06-todo-list
  curl http://localhost:8013
  
  ## 로그 확인하면 디버그 로그가 다량 출력된 것을 볼 수 있음 (설정이 바뀜)
  docker container logs todo-configured
  ```
* 마찬가지 방법으로 네트워크 드라이브도 마운트 가능하다

## 4. 마운트의 한계
* 이미지 레이어에 이미 존재하는 경로의 파일을 똑같이 마운트하는 경우, 이미지 레이어 파일에는 접근 불가능해진다
  ```shell
  ## 아래 컨테이너는 /init 경로의 파일명들을 출력한다.
  docker container run diamol/ch06-bind-mount
  
  ## 마운트한 경우 호스트의 디렉토리로 대체되며 기존 (이미지 레이어에 있던) 파일에는 접근할 수 없다
  docker container run --mount type=bind,source=./new,target=/init diamol/ch06-bind-mount
  ```
* 호스트 컴퓨터의 단일 파일을 컨테이너에 이미 있는 디렉토리로 마운트하는 경우, 윈도우 / 리눅스 동작이 다르다
  * 리눅스에서는 둘 다 보이지만, 윈도우에서는 그렇지 않다
  ```shell
  docker container run --mount type=bind,source=./new/123.txt,target=/init/123.txt diamol/ch06-bind-mount
  ## 리눅스에서는 컨테이너 파일, 마운트한 파일이 모두 보인다.
  ## 윈도우에서는 단일 파일 마운트를 지원하지 않는다.
  ```
* 분산 파일 시스템을 마운트하는 경우 
  * 해당 시스템에서 지원하지 않는 기능이 있을 수 있다.
  * 예를 들어 azure files를 마운트한 경우, 파일 링크 생성을 지원하지 않으므로 에러 발생
  * 느려질 우려도 높다

## 5. 컨테이너 파일 시스템 동작원리
* 유니언 파일 시스템: 도커가 다양한 출처로부터 모아 만든 단일 가상 디스크 파일 시스템
  * 여러 출처 디렉터리를 단일 디스크 사용하듯 접근 가능
* 스토리지별 특징
  * 기록 가능 레이어: 컨테이너별 독립적인 공간이지만, 종료시 유실된다. 데이터 캐싱 등에 적합
  * 로컬 바인드 마운트: 호스트 컴퓨터 <-> 컨테이너 간 공유. 이미지 빌드 없이 즉각 전달될 수 있다
  * 분산 바인드 마운트: 네트워크 스토리지 <-> 컨테이너 간 공유. 여러 컴퓨터/컨테이너에서 동시 접근이 용이하지만 성능 및 안정성이 다소 떨어짐
  * 볼륨 마운트: 컨테이너 <-> 도커의 볼륨 간 공유. 영속성 구현에 좋음
  * 이미지 레이어: 컨테이너의 초기 파일 시스템 구성

## 6. 연습 문제
* 이미 TODO list가 입력된 상태의 컨테이너를, 빈 디렉토리를 마운트로 덮어씌워 리스트 비우기
  ```shell
  docker container exec c9 ls / ## init-data가 컨테이너 상에서 db 디렉토리인 것을 확인
  mkdir dockerstudy-lab-db
  docker container run -d -p 8011:80 --name todo-lab --mount \
   type=bind,source=./dockerstudy-lab-db,target=/init-data diamol/ch06-lab
  ```
* 참고: 모든 컨테이너 삭제는 `docker rm-f $(docker ps -aq)`
* 책에서는 /app/config를 바꿔야 한다고 한다


