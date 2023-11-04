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

## 3. 외부 트래픽 파드로 전달
* 클러스터 외부에서 들어오는 트래픽은 LoadBalancer 유형의 서비스가 처리할 수 있다.
* 클러스터 전체를 커버한다. 즉 다른 노드의 파드에도 트래픽 전달이 가능
* 로드밸런서 서비스 정의 예시
  ```yaml
  apiVersion: v1
  kind: Service
  
  metadata:
    name: numbers-web
  
  spec:
    ports:
      - port: 8080  # 주시하는 포트
        targetPort: 80  # 트래픽이 전달될 파드의 포트
    selector:
      app: numbers-web
    type: LoadBalancer
  ```
  * 8080 포트를 주시하다가, 트래픽을 파드의 80 포트로 전달한다.
  * 따로 `k port-forward`를 통해 포트 포워딩을 따로 설정할 필요 없다.
* 적용하기
  ```shell
  k apply -f numbers/web-service.yaml
  k get svc numbers-web
  k get svc numbers-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080'
  ```
  * 웹 브라우저에서 8080 포트로 접근이 가능하게 되었다.
  * 원래는 포트 포워딩 설정을 해야 했다.
* 도커 데스크탑을 사용하는 경우 localhost가 hostname이다.
  * managed k8s (AKS, EKS) 사용하는 경우는 localhost가 아닌 다른 hostname이 부여될 수 있다.
* 노드포트 NodePort
  * LoadBalancer과 함께 외부 트래픽 파드로 전달하는 역할
  * 노드가 지정된 포트를 주시하도록 한다.
  * 노드에 요청이 직접 인입된다.
  * 모든 노드의 해당 포트(nodePort)가 다 열려있어야 하므로 유연하지 않다.

## 4. 외부로 트래픽 전달
* DB 등의 외부 동작 소프트웨어로 트래픽을 전달해야 할 때가 있다.
* ExternalName: 특정 도메인 네임에 대한 별명
  * k8s DNS 서버가 이를 외부 도메인에 매핑해준다.
  * 파드 입장에서는 외부 컴포넌트와 통신하는 것을 모른다. 로컬 도메인 네임이기 때문
  ```shell
  k delete svc numbers-api
  k apply -f api-service-externalName.yaml
  k get svc numbers-api
  ```
  * 이제 numbers-api에 요청 시 raw.githubusercontent.com으로 k8s DNS 서버가 바꿔준다.
  * 해당 yaml파일 내용
    ```yaml
    apiVersion: v1
    kind: Service
  
    metadata:
      name: nubmers-api  # 로컬 도메인 네임
  
    spec:
      type: ExternalName
      externalName: raw.githubusercontent.com  # 로컬 도메인 네임에 매핑할 외부 도메인
    ```
  * 모든 파드에서 서비스의 도메인 네임을 조회할 수 있다.
    * `k exec deploy/sleep-1 -- sh -c 'nslookup numbers-api | tail -n 50'` -> 외부 도메인 정보가 출력된다.
  * 한계: 주소를 치환해줄 뿐 요청의 내용을 바꿀 수는 없다.
    * HTTP 요청의 경우 헤더의 호스트명이 응답과 다르므로 에러가 발생한다.
* headless service
  * ExternalName과 비슷하지만 도메인 네임 대신 IP로 대체해준다.
  * type: ClusterIP이지만 레이블 셀렉터가 없으므로 대상 파드가 없다.
  * 자신이 제공해야 할 IP가 기록된 endpoint와 함께 배포된다.
  * yaml 예시
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: numbers-api
  spec:
    type: clusterIP
    ports:
      - port: 80
  ---  # 리소스를 구분할 때는 하이픈 3개를 사용한다
  kind: Endpoints
  apiVersion: v1
  metadata:
    name: numbers-api
  subsets:
    - addresses:
      - ip: 192.168.123.234
    ports:
      - ports: 80
  ```
  * DNS 조회 결과 엔드포인트 IP 주소가 아닌 ClusterIP를 가리킨다.
