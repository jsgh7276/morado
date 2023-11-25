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
  * 보안 취약점 해결한 정의
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
