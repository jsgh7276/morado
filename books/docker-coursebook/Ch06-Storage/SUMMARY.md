# Ch06. Storage

## 1. 컨테이너 데이터가 사라지는 이유
* stateless 애플리케이션이면 필요없겠지만, 대부분은 스토리지가 필요함
* 모든 컨테이너는 독립된 파일 시스템을 가지며 서로 영향 주지 않는다
* 랜덤 넘버를 파일 시스템에 기록하는 컨테이너 2개
  ```dockerfile
    docker container run --name rn1 diamol/ch06-random-number
    docker container run --name rn2 diamol/ch06-random-number
  ```
  * 컨테이너 실행은 종료돼도 파일 시스템은 삭제되지 않는다
* `docker container cp`로 컨테이너, 로컬 컴퓨터간 파일 복사 가능
  ```dockerfile
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
    ```dockerfile
    docker container run --name f1 diamol/ch06-file-display
    echo “http://eltonstoneman.com” > url.txt
    docker container cp url.txt f1:/input.txt
    docker container start --attach f1
    ```
  * 마찬가지로 다른 컨테이너/이미지에는 영향 미치지 않는다
  * 해당 컨테이너 사라지면 변경도 사라짐
* 그렇다면 어떻게 stateful한 애플리케이션을 실행할 수 있는가?
  * 컨테이너는 밥 먹듯이 사라지는데
  * 도커 볼륨과 마운트 이용
  * 
