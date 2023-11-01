# Ch03. Service

## 1. 내부 라우팅
* 서비스: 파드를 드나드는 통신 트래픽 라우팅을 맡는다
  * k8s 클러스터 외부, 내부 모두
  * 시스템의 구성요소들을 결합할 수 있다
* 파드에도 IP가 부여되지만 활용하기에는 어렵다
  * 파드는 '쓰고 버리는' 용도이다
  * 매번 IP가 바뀌므로 활용이 어렵다
* (무식하게) IP를 알아내어 핑을 찍어보는 예제
  ```shell
  # deploy 2개를 만든다
  # apply 명령으로 한 번에 여러 개 파일을 전달할 수 있다.
  k apply -f sleep/sleep1.yaml -f sleep/sleep2.yaml
  
  # 파드 시작을 기다린다
  k wait --for=condition=Ready pod -l app=sleep-2
  
  # IP 확인
  k get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'
  
  # pod 1에서 pod 2 IP로 핑 보내기
  k exec deploy/sleep-1 -- ping -c 2 \
    $(k get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}')
  ```
* k8s 가상 네트워크는 클러스터 전체를 커버한다.
  * IP만 알면 다른 노드여도 가능
* 실제로 파드를 삭제해보면 deployment에 의해 새로 생성된 파드는 IP가 다른 것을 볼 수 있다.
* 이러한 문제는 이미 오래 전부터 있었다.
  * IP - 도메인을 연결해주는 DNS 서버는 도입된지 오래되었다
* k8s 클러스터에도 DNS 서버가 있다.
  * IP - 서비스 이름을 대응시켜준다
* 서비스
  * 파드의 네트워크 주소를 추상화한 것
  * 자신만의 IP 주소를 가지며, 삭제될 때까지 불변
  * 파드와는 디플로이먼트처럼 레이블 셀렉터로 느슨히 연결된다
  * 복수 파드와 연결 가능
* 서비스의 예시 yaml 정의
  ```yaml
  apiVersion: v1
  kind: Service
  
  metadata:
    name: sleep-2  # 도메인 네임으로 사용됨
  
  spec:
    selector:
      app: sleep-2  # 해당 레이블 가진 모든 파드
    ports:
      - port: 80  # 80번 포트 주시하다가 파드의 80번 포트로 넘겨준다
  ```
* 서비스를 배포하면 k8s DNS 서버에 해당 도메인 네임이 등록된다.
  ```shell
  k apply -f service.yaml
  k get svc sleep-2
  k exec deploy/sleep-1 -- ping -c 1 sleep-2
  ```
  * ping 요청은 k8s 서비스에서 미지원하는 프로토콜을 사용하므로 실패한다.

## 2. 파드 간 통신
* ClusterIP: 클러스터 내부 파드 - 파드 간 통신에 사용
  * 클러스터 외부에는 노출되지 않는다
* 웹 - API 예제
  * 웹에서는 http://numbers-api 라는 도메인에 접근하려 하지만, 서비스가 없으므로 DNS에 해당 도메인이 없다
  * 다음 yaml로 서비스 정의하여 DNS를 등록해주면 잘 동작한다.
    ```yaml
    apiVersion: v1
    kind: Service
  
    metadata:
      name: numbers-api
  
    spec:
      ports:
        - port: 80
      selector:
        app: numbers-api
      type: ClusterIP  # 기본값이므로 안 써도 동작은 같지만 명시하는 게 낫다
    ```
* 애플리케이션의 모든 것을 yaml로 정의할 수 있다. 컴포넌트, 통신, ...
  * 위 애플리케이션을 위해 2개의 디플로이먼트, 1개의 서비스를 정의하였다
  * 이 덕분에 자기수복성이 뛰어나다
  * 파드를 하나 삭제해도 곧 다시 생긴다.
  * ` k delete pod -l app=numbers-api ` 해도 문제 없음
