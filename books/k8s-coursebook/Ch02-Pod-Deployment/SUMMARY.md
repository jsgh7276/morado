# Ch02. Pod and Deployment

## 1. 쿠버네티스와 컨테이너
* 컨테이너: 애플리케이션을 실행하는 가상화된 환경
* 파드: 컨테이너를 감싸는 또다른 가상 환경
  * 컴퓨팅의 최소 단위이다
  * 클러스터의 노드 중 하나에서 실행된다
  * 자신만의 가상 IP를 가지며 다른 파드와 통신 가능
  * 컨테이너 하나 or 여러 개를 포함
  * 한 파드 내부에서는 네트워크를 공유한다. (localhost로 통신 가능)
  ```shell
  # 컨테이너 하나를 담은 파드 실행
  k run hello-kiamol --image=kiamol/ch02-hello-kiamol
  
  # 파드 준비 대기
  k wait --for=condition=Ready pod hello-kiamol
  
  # 클러스터 내 모든 파드 출력
  k get pods
  
  # 파드 상세정보
  k describe pod hello-kiamol
  ```
  * 보통의 경우에는 컨테이너 하나만 실행한다.
  * "쿠버네티스가 컨테이너를 실행하는 수단"
* k8s가 직접 컨테이너 실행하지는 않는다
  * 컨테이너 런타임에 맡긴다.
  * 도커 또는 다른 것일 수 있다.
  * 다양한 방법으로 파드 설정 보기
    ```shell
    # 기본 정보
    > k get pod hello-kiamol
    
    # 특정 정보 지정 출력 (호스트 IP, 파드 가상 IP)
    > kubectl get pod hello-kiamol --output custom-columns=NAME:metadata.name,NODE_IP:status.hostIP,POD_IP:status.podIP
    NAME           NODE_IP        POD_IP
    hello-kiamol   192.168.65.4   10.1.0.15
    
    # JSON Path로 첫 번쩨 컨테이너의 id 출력 (go 템플릿)
    # 컨테이너 런타임 이름이 맨 앞에 붙는다.
    > kubectl get pod hello-kiamol -o jsonpath='{.status.containerStatuses[0].containerID}'
    docker://ff5e215ee0d24dfc6b29b1bcd08d25
    ```
  * 출력 내용 지정이 간편하다.
  * 컨테이너 식별자는 단순 다른 시스템에 대한 참조일 뿐이다. (직접 실행 안 하므로)
* CRI: 컨테이너 런타임 인터페이스
  * 컨테이너 실행할 책임은 노드에게 있음
  * 이는 CRI를 통해 구현됨
  * 컨테이너 런타임이 도커든 뭐든 간에, CRI를 통해서 관리할 수 있음
  * 표준 API로 제공된다.
* 직접 컨테이너 런타임(도커)으로 컨테이너를 삭제하면
  ```shell
  docker container rm -f $(docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol)
  ```
  * 해당 커맨드 실행 앞뒤로 `docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol`를 실행하여 ID를 확인하면 바뀌었다.
  * 컨테이너가 죽자마자 파드가 대체 컨테이너를 생성하고 파드를 복원했다. (자기수복성)
  * 이는 컨테이너를 파드로 추상화한 덕분이다.
    * 파드는 그대로 있으므로, 새로운 컨테이너를 추가하여 복원
  * 컨테이너 위에 파드, 파드 위에 디플로이먼트
* 포트 포워딩 설정하기
  * 노드에서 파드로 네트워크 트래픽을 전달하는 간편한 방법
  * `k port-forward pod/hello-kiamol 8080:80`
  * 로컬 브라우저에서 8080 포트로 접근이 가능해진다.
* 원시 타입 리소스이므로 직접 파드를 실행할 일은 잘 없다.
  * 보통 컨트롤러 객체가 함

## 2. 디플로이먼트
* 컨트롤러 객체: 다른 객체를 다시 추상화한 것. 즉 다른 객체를 관리한다
* 디플로이먼트: 주로 파드를 관리하는 컨트롤러 객체
  * 파드는 하나의 노드에서만 실행된다는 단점이 있는데, 이를 보완한다.
  * 유실된 파드 대체, 스케일링 기능
  * 디플로이먼트는 파드를, 파드는 컨테이너를 관리한다
* k8s가 디플로이먼트를 생성하면 디플로이먼트는 파드를 생성한다
* `k create deployment hello-kiamol-2 --image=kiamol/ch02-hello-kiamol`
  * k get pods 하면 파드가 생성된 것을 볼 수 있다
    * `hello-kiamol-2-5dbf59b864-cmklk` -> 지정한 이름에 무작위 문자열이 붙는다
  * deployment가 파드가 없는 것을 확인하고 생성해준 것
* 리소스 추적은 key-value 형태의 레이블로 이루어진다
  * 디플로이먼트는 자신이 관리하는 파드에 레이블을 부여한다.
  * 파드의 레이블 출력: `k get deploy hello-kiamol-2 -o jsonpath='{.spec.template.metadata.labels}'`
    * `{"app":"hello-kiamol-2"}%`
    * app이라는 label에 hello-kiamol-2라는 값을 달았다.
  * 해당 레이블 가진 파드 출력: `k get pods -l app=hello-kiamol-2`
  * 쿠버네티스는 이러한 레이블을 이용하여 리소스 관계를 파악하는 패턴이 자주 활용된다.
  * 따라서 레이블 정보를 함부로 수정하지 않게 주의할 것
* 레이블을 수정하는 예시
  * `k label pods -l app=hello-kiamol-2 --overwrite app=hello-kiamol-x`
  * 파드가 또 하나 생성되었다.
  * deployment 입장에서는 자신이 관리하는 파드가 사라졌기 떄문에 새로 만든 것
    ```shell
    k get pods -o custom-columns=NAME:metadata.name,LABELS:metadata.labels
    NAME                              LABELS
    hello-kiamol                      map[run:hello-kiamol]
    hello-kiamol-2-5dbf59b864-cmklk   map[app:hello-kiamol-x pod-template-hash:5dbf59b864]
    hello-kiamol-2-5dbf59b864-vqmb7   map[app:hello-kiamol-2 pod-template-hash:5dbf59b864]
    ```
  * 레이블을 다시 복원하면 2개가 됐던 파드 하나가 삭제된다.
* 포트 포워딩
  * 디플로이먼트로 생성된 파드는 랜덤 문자열이 붙으므로 포트 포워딩을 직접 붙이기 어렵다
  * 디플로이먼트 리소스 정의에 직접 설정해주면 됨
  * `k port-forward deploy/hello-kiamol-2 8080:80`

## 3. 애플리케이션 매니페스트
* 애플리케이션 매니페스트: 애플리케이션을 속속들이 기술하는 yaml 파일
  * yaml 또는 json이지만, yaml은 가독성이 더 뛰어나고 주석을 달 수 있다는 장점
* 파드의 yaml 정의 예시
  ```yaml
  # 시작은 항상 아래와 같다
  apiVersion: v1
  kind: Pod
  
  # metadata의 name은 필수, label은 비필수이다.
  metadata:
    name: hello-kiamol-3
  
  # spec은 리소스의 실제 정의 내용, 파드는 컨테이너를 지정해야 한다.
  # 컨테이너는 이미지와 이름으로 정의된다.
  spec:
    containers:
      - name: web
        image: kiamol/ch02-hello-kiamol
  ```
* 다음 커맨드를 통해 매니페스트 파일로 애플리케이션을 배포할 수 있다.
  * `k apply -f pod.yaml`
* 매니페스트의 장점
  * 정의 공유가 용이하다
  * 똑같은 배포를 반복 가능하다
* 매니페스트 파일은 꼭 로컬에 있을 필요는 없다
  * 즉 `k apply -f https://raw.githubusercontent.com/sixeyed/kiamol/master/ch02/pod.yaml` 도 가능
  * 위 커맨드 실행하면 바뀌진 않는다. 이미 해당 파드가 있기 때문
* deployment의 yaml 정의
```yaml
# deployment는 API v1에 속한다.
apiVersion: apps/v1
kind: Deployment

# deployment의 이름은 필수
metadata:
  name: hello-kiamol-4

# 자신의 관리 대상을 결정하는 레이블 셀렉터
spec:
  selector:
    matchLabels:
      app: hello-kiamol-4

  # 디플로이먼트가 파드를 만들 때 아래 템플릿이 사용됨
  template:
    metadata:
      labels:
        app: hello-kiamol-4

    # 파드의 정의에는 컨테이너 이름, 이미지 이름을 정한다.
    spec:
      containers:
        - name: web
          image: kiamol/ch02-hello-kiamol
```
* `k get pods`로 확인해보면 k create로 생성했을 때와 같음을 알 수 있다.
