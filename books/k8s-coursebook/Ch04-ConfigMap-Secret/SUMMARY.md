# Ch04. ConfigMap, Secret

## 1. 설정 전달하기
* 컨테이너에 설정값을 주입하는 데 쓰이는 두 가지 리소스: ConfigMap, Secret
  * 데이터 포맷 제한 없다
  * 다른 리소스와 독립적인 장소에 보관됨
  * 매우 유연하다
  * 스스로 어떤 기능을 하진 않는다. 소량의 데이터를 저장할 뿐
* 환경 변수
  * 일단 출력해보기: `k exec deploy/sleep -- printenv HOSTNAME KIAMOL_CHAPTER`
    * HOSTNAME은 출력되지만 KIAMOL_CHAPTER라는 환경변수는 없으므로 에러 발생
  * 파드 정의에 다음과 같이 환경변수를 추가할 수 있다.
    ```yaml
    spec:
      containers:
        - name: sleep
          image: kiamol/ch03
          env:
          - name: KIAMOL_CHAPTER
            value: "04"
    ```
  * 환경 변수는 파드 실행 중 변경할 수 없다.
    * 수정하고 싶다면 파드 정의 수정 후 대체해야
  * 새로운 정의로 수정 후 배포하면 KIAMOL_CHAPTER 환경변수도 잘 출력된다.
    * 이미 같은 이름 리소스가 배포되어있는 상태에서도 k apply -f 할 수 있음
  * 간단한 설정은 위와 같이 해도 무방하나, 이보다 복잡한 경우 ConfigMap 사용
* ConfigMap
  * 파드에서 읽을 데이터를 저장하는 리소스
  * key-value, 텍스트, 바이너리 등 다양한 형태의 데이터를 저장할 수 있다.
  * 한 파드가 여러 ConfigMap 읽을 수도 있고, vice versa도 가능
  * ConfigMap은 read only이다 (파드에서 수정 불가)
  * yaml 예시
    ```yaml
      env:
      - name: KIAMOL_CHAPTER
        value: "04"               # 값을 직접 지정
      - name: KIAMOL_SECTION
        valueFrom:
          configMapKeyRef:        # ConfigMap에서 읽어오겠다.
            name: sleep-config-literal   # ConfigMap 이름
            key: kiamol.section          # ConfigMap 내부의 항목 이름
    ```
    * 즉 이 파드는 sleep-config-literal 이라는 ConfigMap이 있어야 배치(배포) 가능하다.
* ConfigMap 만들기
  * literal로 만들기: 간단한 경우 명령줄에서 바로 만들 수 있다
    ```shell
    k create configmap sleep-config-literal --from-literal=kiamol.section='4.1'
    k get cm sleep-config-literal
    k describe cm sleep-config-literal
    k apply -f sleep/sleep-with-configMap-env.yaml
    k exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'   # KIAMOL_SECTION 잘 입력됨
    ``` 
  * 파일로도 만들 수 있다.

## 2. yaml로 ConfigMap 만들기
* 파일로 ConfigMap 만들 수 있다.
* 파일의 내용
  ```
  KIAMOL_CHAPTER=ch04
  KIAMOL_SECTION=ch04-4.1
  KIAMOL_EXERCISE=try it now
  ```
* 만들기
  ```shell
  k create configmap sleep-config-env-file --from-env-file=sleep/ch04.env
  k exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"' 
  ```
  * 하지만 파일과 내용이 다른 변수가 있다. -> env가 envFrom보다 우선하기 때문
  * 다음 yaml을 보면 컨테이너의 configmap 읽는 설정 상에서 env 설정이 덮어씌워졌다.
  ```shell
    spec:
      containers:
        - name: sleep
          image: kiamol/ch03-sleep
          envFrom:
          - configMapRef:
              name: sleep-config-env-file
          env:
          - name: KIAMOL_CHAPTER
            value: "04"
          - name: KIAMOL_SECTION
            valueFrom:
              configMapKeyRef:              
                name: sleep-config-literal
                key: kiamol.section
  ```
* 설정 출처별로 우선순위가 다른 것을 잘 활용하자. 에러를 확인해야 할 때 임시로 컨테이너 env만 바꾼다던지..
  * 기본 설정값은 컨테이너 이미지에 포함시킨다.
  * 각 환경의 실제 설정값은 ConfigMap으로 설정
  * 변경이 필요한 값은 디플로이먼트 파드 정의에서 env를 바꿔서 적용
* 컨피그맵 파일로 저장하기
  * configmap yaml 정의
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: todo-web-config-dev
    data:
      config.json: |-  # 키가 파일 이름이 된다. 어떤 파일 내용이든 저장 가능하다
        {
          "ConfigController": {
            "Enabled": true
          }
        }
    ```
  * json 파일을 직접 만드는 것보다 yaml에 명시하는 게 apply 명령어로 바로 배포할 수 있다는 장점
  * json 파일을 직접 만들어놓으면 k create configmap... 먼저 해야 됨
  * 위의 컨피그맵을 띄우고 다시 localhost:8080/config 들어가면 정상 동작함을 알 수 있다.
