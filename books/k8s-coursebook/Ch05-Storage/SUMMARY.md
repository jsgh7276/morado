# Ch05. Storage

## 1. 컨테이너의 파일 시스템
* 컨테이너의 파일시스템은 여러 출처를 합쳐 구성됨
  * 컨테이너 이미지 (read only)
  * 컨테이너가 기록 가능한 레이어 (writable layer)
    * 컨테이너에서 파일 수정/추가 작업이 일어남
  * 출처 다양하지만 애플리케이션 입장에서는 그냥 하나의 파일 시스템으로 보인다.
  * 파드 재시작하거나 교체될 때 데이터 손실에 주의하여야 한다.
* 데이터 손실되는 것 확인하기
  ```shell
  k exec deploy/sleep -- sh -c 'echo ch05 > /file.txt; ls /*.txt'  # 파일 생성
  k exec -it deploy/sleep -- killall5  # 모든 프로세스 종료하여 파드가 재시작되도록 함
  k get pod -l app=sleep -o jsonpath='{.items[0].status.containerStatuses[0].containerID}'  # 컨테이너 아이디 확인 (전후에 바뀜)
  k exec deploy/sleep -- ls /*.txt  # 파일 사라진 것 확인 가능
  ```
* ConfigMap, Secret의 경우 읽기 전용 스토리지로 볼륨 마운트 방식으로 연결 가능하다.
* EmptyDir 볼륨 정의하여 마운트시키기
  ```yaml
  spec:
    containers:
      - name: sleep
        image: kiamol/ch03-sleep
        volumeMounts:
          - name: data
            mountPath: /data
    volumes:
      - name: data
        emptyDir: {}
  ```
  * 파드와 생애주기가 같다. 컨테이너가 교체되어도 그대로 사용 가능
  * 데이터 유지 여부를 위와 같은 방식으로 확인했을 때, 컨테이너 ID가 교체되어도 그대로 유지되는 것 확인됨
  * EmptyDir는 로컬 캐시에 적합하다.
  * 주의사항: 파드와 생애주기가 같으므로 파드가 교체되면 없어짐

## 2. 노드에 데이터 저장
* 제일 상상하기 쉬운 방법 중 하나는 볼륨이 노드의 특정 디렉토리를 가리키게 하는 것
  * 하지만 파드가 새로 뜰 때 기존과 동일한 노드에 배치되어야 한다는 단점이 있다.
* 예시: 캐시를 사용하는 pi 프로그램
  * EmptyDir가 적절하다.
  * 어차피 캐시이고, 데이터의 중요도가 높지 않음
  * 파드가 대체될 때 캐시가 유실되지만, 그렇다고 동작을 못하는 것은 아님
  * 실제로 파드를 삭제(`k delete pod -l app=pi-proxy)해보면 다시 응답시간이 느려짐
* HostPath 볼륨: 노드의 디스크를 가리키는 볼륨
  * EmptyDir보다 오래 간다.
  * 노드의 디스크에 기록됨
  * 한계점: 모든 노드에 복제본을 만들어주지는 않는다.
  * 다음과 같은 yaml로 볼륨을 정의할 수 있다.
    ```yaml
    volumes:
      - name: cache-volume
        hostPath:
          path: /volumes/nginx/cache   # 노드 디렉토리
          type: DirectoryOrCreate      # 디렉토리 없으면 생성할 것
    ```
  * 볼륨의 생애주기가 노드 디스크와 같아진다. 단일 노드인 경우 이 점이 보장됨
  * pi 프로그램을 hostPath로 교체하고 실습하면, 파드 날려도 캐시 남아있는 것을 확인할 수 있음
  * 단점
    * 노드 두 개 이상인 경우 문제
    * 항상 같은 노드에서 실행해야 하면 자기수복성이 제한됨
    * 노드가 고장난 경우 파드도 복구가 어려움
    * 보안에 취약함 (`/` 경로를 마운트하면 시스템 전체 접근 가능해짐)
    ```yaml
    volumes:
    - name: node-root
      hostPath:
      path: /
      type: Directory      # 경로에 디렉터리가 존재해야 함
    ```
  * 보안 취약점 해결하기
    * 볼륨은 루트로 하더라도, 볼륨 하위 디렉토리를 마운트하면 안전하다
    ```yaml
    spec:
    containers:
      - name: sleep
        image: kiamol/ch03-sleep
        volumeMounts:
          - name: node-root
            mountPath: /pod-logs       # 컨테이너 경로
            subPath: var/log/pods      # 해당 볼륨 내 경로
          - name: node-root
            mountPath: /container-logs
            subPath: var/log/containers
    volumes:
      - name: node-root
        hostPath:
          path: /
          type: Directory
    ```
    ```shell
    k apply -f sleep/sleep-with-hostPath-subPath.yaml
    k exec deploy/sleep -- sh -c 'ls /pod-logs | grep _pi-'
    k exec deploy/sleep -- sh -c 'ls /container-logs | grep nginx'
    ```
    * 이처럼 해당 로그 경로 밖은 컨테이너에서 볼 수 없다.
  * stateful application 도입할 때 hostPath 사용을 검토해볼 수 있다.

## 3. PersistentVolume, Claim
* 분산 스토리지: 파드가 어떤 노드에서 실행중이어도 접근할 수 있음
  * NFS, 애저 파일스 등
  * 다만 플랫폼에서 해당 스토리지를 지원해야 한다
* 파드는 컴퓨팅 계층의 추상, 서비스는 네트워크 계층의 추상
  * 스토리지 추상: PersistentVolume, PersistentVolumeClaim
* PersistentVolume: 사용가능한 스토리지 조각
  * 정의 예시
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv01            # 볼륨 이름
    spec:
      capacity:
        storage: 50Mi       # 용량
      accessModes:
        - ReadWriteOnce     # 파드 하나에서만 사용가능
      nfs:
        server: nfs.my.network         # NFS 도메인 네임
        path: "/kubernetes-volumes"    # 스토리지 경로
    ```
    * 이 정의는 nfs가 실제로 구축되어 있어야 돌아간다.
  * 로컬 스토리지 사용하는 예시
    ```shell
    # 첫 번째 노드에 레이블 부여
    k label node $(k get nodes -o jsonpath='{.items[0].metadata.name}') kiamol=ch05
    k get nodes -l kiamol=ch05
    k apply -f todo-list/persistentVolume.yaml
    k get pv
    ```
    ```yaml
    spec:
      capacity:
        storage: 50Mi
      accessModes:
        - ReadWriteOnce
      local:
        path: /volumes/pv01 
      nodeAffinity:
        required:
          nodeSelectorTerms:
            - matchExpressions:
              - key: kiamol
                operator: In
                values:
                  - ch05
    ```
    * 분산 스토리지가 없으므로 노드에 레이블을 부여해야 한다.
      * 모든 노드에서 접근 가능한 분산 스토리지는 상관 없지만, 노드의 로컬 볼륨은 한 노드에서만 접근 가능하기 때문
    * pv STATUS가 `Available`이면 아직 요청되지 않은 볼륨이다.
* PersistentVolumeClaim
  * 파드는 PV를 직접 사용할 수는 없다. PVC로 먼저 사용을 요청해야 한다.
  * 파드가 사용하는 스토리지의 추상
  * 요구 조건이 일치해야 한다.
  * 정의에는 스토리지의 용량과 유형, 접근 유형을 지정한다.
  * 스토리지 유형 지정하지 않으면 k8s가 요구사항과 일치하는 pv를 알아서 찾아줌 (밑의 예시)
  * pvc와 pv는 일대일 관계이고 이미 연결된 pv는 다른 pvc와 연결될 수 없다.
  * 정의 예시
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: postgres-pvc
    spec:
      accessModes:            # required
        - ReadWriteOnce
      resources:
        requests:
          storage: 40Mi       # 요청하는 스토리지 용량
      storageClassName: ""    # 유형 지정 x
    ```
    * 이 pvc를 배치하면 위에서 미리 배치해둔 pv가 바로 연결된다.
    * pv STATUS를 보면 `Bound`로 바뀌었다.
  * 프로비저닝 방식: pv를 명시적으로 생성하는 방식
    * pvc만 생성하고 요구사항을 만족하는 pv가 없는 경우: pvc가 생성은 되지만 스토리지 사용 불가, 대기 상태로 남는다
    * pvc를 하나 더 생성해보면 pv를 찾지 못해서 STATUS가 `Pending`이다.
    * 100MB 요구사항을 만족하는 pv가 나타날 때까지 대기한다.
    * pvc가 대기 상태이면 해당 pvc 이용하는 파드도 정상 시작되지 못한다.
  * pvc를 스토리지로 사용하는 파드 정의 예시
    ```yaml
    spec:
      containers:
        - name: db
          image: postgres:11.6-alpine
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-pvc
    ```
  * 노드에 로그인 권한을 부여하기 위해 다음과 같이 우회: `kubectl exec deploy/sleep -- mkdir -p /node-root/volumes/pv01`
  * postgres db 배치 시 정상 동작하는 것을 확인할 수 있다.
    * `kubectl exec deploy/sleep -- mkdir -p /node-root/volumes/pv01`
  * 파드 교체해도 데이터가 남아있음

## 4. 동적 볼륨 프로비저닝
* 정적 볼륨 프로비저닝: 명시적으로 pv와 pvc를 연결
  * 모든 k8s 클러스터에서 사용 가능
  * 스토리지 접근 제약이 큰 경우 선호
* 동적 볼륨 프로비저닝: pvc만 생성하면 요구사항에 맞는 pv를 클러스터가 동적으로 생성해줌
  * 스토리지 유형만 설정하면 된다.
  * 따로 지정하지 않으면 기본 유형이 제공됨
  * 정의 예시
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: postgres-pvc-dynamic
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 100Mi
    ```
    * storageClassName 필드가 없으므로 기본 StorageClass가 사용됨
    * 해당 pvc를 배치하면 pv가 자동으로 생성되어 연결된 것을 볼 수 있다.
    * 삭제하면 같이 삭제된다.
  * k8s 플랫폼에 따라 동작이 다를 수 있다.
    * 도커 데스크탑은 hostPath가 기본 StorageClass이다. aks는 애저 파일스
* StorageClass 정의하기
  * provisioner: pv 필요할 때 만드는 주체
  * reclaimPolicy: pvc 삭제되었을 때 pv를 어떻게 할 것인가에 대한 설정
  * volumeBindingMode: pvc 생성 직후 pv를 생성해서 연결할지, 해당 pvc 사용하는 파드가 생길 때 pv를 생성할지
  * 스크립트를 실행하여 기본 StorageClass를 복제하였다.
    * `k get storageclass`
  * 스토리지클래스는 스토리지의 추상화이다.
  * 스토리지클래스 사용하는 pvc 정의 예시
    ```yaml
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: kiamol
      resources:
        requests:
          storage: 100Mi
    ```
  * 해당 볼륨을 배치하고 todo-db 파드와 연결하면, 기존 todo 목록은 사라진다.
    * 하지만 다시 기존 볼륨을 연결하면 복구할 수 있다.

## 5. 적절한 스토리지 선택
* DB 등의 stateful 애플리케이션을 꼭 k8s에서 실행하는 것이 제일 좋은 방법은 아닐 수 있다.

## 6. 연습 문제
* url 찾기
  * `kubectl get svc todo-proxy-lab -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8082'`
* 파드 정의에 pvc 추가
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: todo-proxy-lab-pvc
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 100Mi
  ```
  ```yaml
  volumes:
    - name: cache-volume
      persistentVolumeClaim:
        claimName: todo-proxy-lab-pvc
  ```
