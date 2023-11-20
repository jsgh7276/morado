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

## 3. ConfigMap 설정값 주입하기
* ConfigMap의 파일을 컨테이너 파일시스템 상으로 주입할 수 있다.
  * 위의 예시의 경우 config.json 파일을 주입할 수 있음
* volume: 컨피그맵 데이터를 파드로 전달
* volume mount: 볼륨을 파드 컨테이너의 특정 경로에 위치
  ```yaml
  spec:
    containers:
      - name: web
        image: kiamol/ch04-todo-list
        volumeMounts:                   # 컨테이너에 볼륨을 마운트한다
          - name: config                # 마운트할 볼륨 이름
            mountPath: "/app/config"    # 볼륨이 마운트될 경로
            readOnly: true
    volumes:                            # 볼륨은 파드 수준에서 정의된다
      - name: config                    # 이 이름이 볼륨 마운트의 이름과 일치해야 한다
        configMap:                      # 볼륨의 원본은 컨피그맵이다
          name: todo-web-config-dev     # 내용을 읽어 올 컨피그맵 이름
  ```
  * 컨피그맵은 디렉터리로 취급된다
  * 컨피그맵의 각 항목이 컨테이너 파일 시스템에 들어간다.
* 컨테이너 입장에서는 그냥 하나의 파일 시스템이지만, origin을 따지면 다르다.
  ```sh
  > k exec deploy/todo-web -- sh -c 'ls -l /app/app*.json'
  -rw-r--r--    1 root     root   /app/appsettings.json  # 이미지에서 온 파일
  > k exec deploy/todo-web -- sh -c 'ls -l /app/config/*.json'
  lrwxrwxrwx    1 root     root   /app/config/config.json -> ..data/config.json  # 컨피그맵에서 온 파일
  # 해당 파일 lrwx로 권한 나오지만, 실제로 수정하려고 해보면 실패한다. 읽기 전용 파일을 가리키는 링크이므로
  ```
* 여러 개의 설정 파일도 하나의 컨피그맵으로 관리 가능
  ```yaml
  data:
    config.json: |-
      {
        "ConfigController": {
          "Enabled" : true
        }
      }
    logging.json: |-
      {
        "Logging": {
          "LogLevel": {
            "ToDoList.Pages" : "Debug"
          }
        }
      }
  ```
* 파드 동작 중 설정을 업데이트할 수도 있다. 다만 애플리케이션의 구현에 따라 업데이트 여부가 갈린다.
  * 다음 명령어로 로깅 레벨을 조정
  * `k apply -f configMaps/todo-web-config-dev-with-logging.yaml`
  * 컨테이너 내에 새로운 json (loggin.json) 파일 생긴 것을 알 수 있다.
* 주의사항: 볼륨 마운트가 컨테이너 이미지 상의 경로를 덮어쓸 수 있다.
  * 이미지 상의 경로와 겹치지 않도록 해야 함
  * 잘못된 마운트로 바이너리를 다 날린 경우 (`todo-web-dev-broken.yaml`)
    * 3번 재시도 후 새로운 파드가 CrashLoopBackOff 상태가 됨
    * 기존 파드가 남아있어서 애플리케이션은 동작하긴 함
  * 새로운 파드가 정상 시작하지 않으면 기존 파드는 제거되지 않는다.
* 컨피그맵 데이터 중 단일 항목만 전달하기
  ```yaml
  volumes:
    - name: config
      configMap:
        name: todo-web-config-dev  # 컨피그맵 지정
        items:                     # 전달할 데이터 항목 지정
          - key: config.json
            path: config.json
  ```
* 컨피그맵을 통해 환경 변수부터 볼륨 마운트까지 필요한 설정을 유연하게 관리 적용할 수 있다.
* 애플리케이션과 설정을 분리하여 유연성을 확보할 수 있음

## 4.4 Secret
* Secret: 컨피그맵과 비슷한 별개의 리소스. 노출이 최소화된다는 차이점
  * 클러스터에서 별도 관리됨
  * 해당 노드에만 전달됨
  * 노드는 해당 값을 파일에 저장하지 않고, 메모리에만 저장한다.
  * 암호화가 적용된다
  * 권한이 있으면 읽을 수 있지만, 평소에는 Base64 인코딩된 상태임
* 시크릿 생성 및 값 확인하기
  ```sh
  k create secret generic sleep-secret-literal --from-literal=secret=shh...
  k describe secret sleep-secret-literal  # 해도 안나온다.
  k get secret sleep-secret-literal -o jsonpath='{.data.secret}'  # 암호화된 값이 나온다
  k get secret sleep-secret-literal -o jsonpath='{.data.secret}' | base64 -d  # 정상 출력
  ```
* 파드에 시크릿 주입하기
  ```yaml
  spec:
    containers:
      - name: sleep
        image: kiamol/ch03-sleep
        env:
        - name: KIAMOL_SECRET             # 컨테이너에서의 환경 변수 이름
          valueFrom:
            secretKeyRef:
              name: sleep-secret-literal  # 시크릿 이름
              key: secret                 # 시크릿 항목 이름
  ```
  * 파드 컨테이너에 전달되면, 해당 파드는 시크릿 평문을 이용할 수 있다.
  * `k apply -f sleep/sleep-with-secret.yaml`
  * `k exec deploy/sleep -- printenv KIAMOL_SECRET` -> 평문 출력됨
  * 환경변수는 모든 앱이 접근 가능하다는 취약점이 있음. 파일로 넘기는 것이 대안이 될 수 있다.
* 이미지에 시크릿으로 DB 로그인 정보 넘기기
  * secret yaml
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: todo-db-secret-test
    type: Opaque  # 임의의 텍스트를 저장
    stringData:
      POSTGRES_PASSWORD: "kiamol-2*2*"  # key-value 형태로 저장
    ```
    * yaml에 저장하는 것은 앱 배치가 간편하지만 민감정보가 git에 노출된다는 단점, 실제 운영에서는 이렇게 하면 안 된다
  * 해당 시크릿을 `k apply`로 배치한다.
    * `k get secret todo-db-secret-test -o jsonpath='{.data.POSTGRES_PASSWORD}'` -> base64 출력됨
    * `k get secret todo-db-secret-test -o jsonpath='{.metadata.annotations}'` -> 여기선 평문 볼 수 있음
* 파일로 전달하고, 해당 파일의 경로를 환경변수로 지정해보자
  * 앞서 본 환경 변수로 값을 직접 전달하는 방법보다 안전하다.
    * 파일 권한은 파드 정의에서 설정 가능하기 떄문이다.
  * 컨피그맵과 마찬가지로, 파드 정의에 볼륨 마운트를 해야 한다.
  ```yaml
  spec:
    containers:
      - name: db
        image: postgres:11.6-alpine
        env:
        - name: POSTGRES_PASSWORD_FILE
          value: /secrets/postgres_password
        volumeMounts:
          - name: secret
            mountPath: "/secrets"
    volumes:
      - name: secret
        secret:
          secretName: todo-db-secret-test   # 볼륨을 만들 시크릿 이름
          defaultMode: 0400                 # 파일 권한 설정
          items:                            # 시크릿 중 특정 항목 지정 가능
          - key: POSTGRES_PASSWORD
            path: postgres_password
  ```
  * 해당 파드를 배치하고 다음 명령어로 권한을 확인: `k exec deploy/todo-db -- sh -c 'ls -l $(readlink -f /secrets/postgres_password)'`
    * 400 권한인 것을 확인할 수 있음 (컨테이너 유저만 read 권한)
* 컨피그맵 - 시크릿 - 디플로이먼트/서비스 순서대로 배치하면 파드 속에 시크릿 데이터가 담긴 것을 확인 가능

