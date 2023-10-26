# Ch01. Overview

## 1. 이해하기
* 쿠버네티스
  * 컨테이너를 실행하는 플랫폼
  * 인프라 수준 관심사였던 로드밸런싱, 네트워크, 스토리지, 컴퓨팅을 애플리케이션 설정 영역으로 데려왔다.
* 두 가지 핵심 개념: API, 클러스터
  * 클러스터는 도커와 같은 컨테이너가 동작하는 여러 대의 서버 노드가 모인 논리적 단위
  * 노드: 서버이다. 노드 추가/제외/업데이트가 가능. 평소에는 신경쓸 일 없음
  * 노드 중 일부는 k8s api를 실행하고 나머지는 컨테이너 애플리케이션을 실행
  * 쿠버네티스 API는 리눅스 노드상의 컨테이너에서 동작하지만, 클러스터에 다른 플랫폼의 노드도 포함될 수 있다.
* 앱 실행 절차
  * yaml 파일에 애플리케이션을 기술하고 k8s api에 전달
  * k8s가 앱 현상태와 yaml을 비교하고 맞추는 작업을 한다
* 노드 또는 컨테이너에 이상이 생겨도 k8s가 대응해준다.
* 분산 DB, 스토리지, 민감정보 저장 공간도 제공하며, 트래픽도 관리해준다.
* 앱 형식, 플랫폼과 상관없이 모두 똑같은 방식으로 기술하고 배포 및 관리할 수 있다.
* 용어
  * 애플리케이션 매니페스트: 애플리케이션을 기술한 yaml 파일 (모든 컴포넌트 목록이 기록됨)
  * 리소스: 애플리케이션을 구성하는 컴포넌트
    * 서비스, 디플로이먼트, 컨피그맵, 볼륨, 레플리카셋, 파드 등

## 2. 대상 독자
* 쿠버네티스는 방대한 주제이다.
* CKAD의 80%, CKA의 50%를 커버한다
* 컨테이너 기술 지식이 어느 정도 있는 k8s 초보

## 3. 환경 세팅
* 먼저 도커 데스크톱 설치
* 설정 - Kubernetes - Enable Kubernetes 체크한 후 재시작한다.
  * 도커 데스크톱이 리눅스 가성 머신 위에서 쿠버네티스를 실행해준다
  * `Reset Kubernetes Cluster` 버튼으로 리셋할 수 있다.
* 설정은 `~/kube`에 저장되는듯. 뭔가 꼬였다면 kubernetes cluster를 docker desktop 상에서 재시작해주자
  * 사내 클러스터용 설정파일은 `~/.kube-backup`에 설정해주었음
  * `Reset Kubernetes Cluster` 버튼 누르면 `~/.kube`가 새로 생긴다.
  * 디폴트 설정파일
    ```
    > k config view
    apiVersion: v1
    clusters:
    - cluster:
      certificate-authority-data: DATA+OMITTED
      server: https://127.0.0.1:6443
      name: docker-desktop
      contexts:
    - context:
      cluster: docker-desktop
      user: docker-desktop
      name: docker-desktop
      current-context: docker-desktop
      kind: Config
      preferences: {}
      users:
    - name: docker-desktop
      user:
      client-certificate-data: REDACTED
      client-key-data: REDACTED
    ```
  * 다음 커맨드로 클러스터 정상 동작 확인. 아래와 비슷하게 나오면 정상이다.
    ```
    > k get nodes
    NAME             STATUS   ROLES           AGE     VERSION
    docker-desktop   Ready    control-plane   7m56s   v1.27.2
    ```
  * k get pod, k run 등의 커맨드 정상 실행 확인
