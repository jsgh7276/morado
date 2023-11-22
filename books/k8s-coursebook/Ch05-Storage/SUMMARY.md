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
