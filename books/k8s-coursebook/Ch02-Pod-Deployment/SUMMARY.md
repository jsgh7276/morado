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
  * 컨테이너가 죽자마자 파드가 대체 컨테이너를 생성하고 파드를 복원했다.
  * 이는 컨테이너를 파드로 추상화한 덕분이다.
    * 파드는 그대로 있으므로, 새로운 컨테이너를 추가하여 복원
  * 컨테이너 위에 파드, 파드 위에 디플로이먼트
* 포트 포워딩 설정하기
  * 노드에서 파드로 네트워크 트래픽을 전달하는 간편한 방법
  * `k port-forward pod/hello-kiamol 8080:80`
  * 로컬 브라우저에서 8080 포트로 접근이 가능해진다.
* 원시 타입 리소스이므로 직접 파드를 실행할 일은 잘 없다.
  * 보통 컨트롤러 객체가 함
