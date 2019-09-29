# ConfigMap과 Secret: 애플리케이션 설정

앞서 진행하였던 6장까지의 예제는 어떤 설정 옵션(설정값, 변수, 환경변수, etc.)의 데이터를 전달할 필요가 없었다.
하지만, 우리가 개발하고 운영하는 대부분의 모든 애플리케이션에는 설정 옵션이 필요하므로 쿠버네티스에는 어떤 방식으로 전달할 수 있는지에 대하여 알아보기로 하자!

## ConfigMap과 Secret은 뭘까요?
- 쿠버네티스에서는 설정 옵션 자체를 변수로 관리해서 Pod가 생성될 때 참조할 수 있도록 하는 기능이 있는데 이것을 바로 우리는 **ConfigMap, Secret** 이라고 부른다. 

## ConfigMap과 Secret은 왜 사용하게 되었을까?   
- 컨테이너 안에 있는 설정 파일을 사용하는 것은 꽤나 까다롭다. 설정 파일을 컨테이너 이미지 자체에 구울 경우도 있고, 파일이 들어있는 볼륨을 컨테이너에 마운트 해야할 상황이 다가오기도 하는데, 이 이야기는 설정 옵션을 변경할 때 마다 이미지를 다시 만들어야한다는 뜻과 같다. ~~(귀찮아... 행봇님 대신 해주세요....)~~
- 조금 더 와닿는 예를 들어보자면, 여러가지 이유로 인하여 사용자 입장에서는 컨테이너 이미지 내부에 API 키, 패스워드, 인증 토큰과 같은 개인 정보를 저장하고 싶지 않은 니즈가 있다. ~~(나만 그런가...)~~

## ConfigMap 톺아보기
### 1. ConfigMap 소개 
- ConfigMap은 앞서 설명한 것과 같이 설정 옵션을 Key/Value 형태로 저장해놓는 일종의 저장소다.
- 애플리케이션은 ConfigMap을 직접 읽거나 존재 유무에 대하여 알 필요가 없고, 환경 변수 또는 볼륨의 파일로 컨테이너에 전달된다. (Kubernetes in Action 그림 7.2 참고)
### 2. ConfigMap 사용 방법
A. YAML file을 통한 생성
```
apiVersion:v1
kind: Configmap
metadata:
  name: example-config
data:
  log.level=err
```
B. kubectl create configmap 커맨드를 통한 생성
```
kubectl create configmap example-config --from-literal=log.level=err
```
### 3. ConfigMap 관리
A. 최초로 ```kubectl create configmap``` 커맨드를 통하여 ConfigMap을 생성한 후에는 다른 ConfigMap으로 덮어쓰는 것은 불가능하다. 다른 쿠버네티스 리소스와 같이 별도의 파일을 관리하고 변경이 있을 경우 ```kubectl apply ``` 커맨드를 통하여 원하는 Configuration을 관리할 수 있다.

B. A가 가능하려면 먼저 YAML 파일 형식으로 내보내야 하는데 방법은 아래와 같다. 
```
kubectl get configmap example-config -o yaml -export > deploy/example-config.yml
```

C. B를 통하여 생성된 example-config.yml를 확인해보면 아래와 같다.
data apiVersion, kind, metadata 키는 매우 중요한 요소이지만 metadata 하위에 있는 metadata.creationTimestamp와 metadata.selfLink 와 같은 서브키들은 필수적이지 않기 때문에 삭제해도 무방하다.
```
apiVersion:v1
kind: Configmap
data:
  log.level=err
metadata:
  creationTimestamp: null
  name: example-config
  selfLink: /api/v1/namespaces/default/configmaps/example-config
```

D. B를 통하여 생성된 example-config.yml의 변경이 있을 경우, A에서 언급한 것과 같이 ```kubectl apply ``` 커맨드를 실행하여야 하는데 동작하지 않고 아마 아래와 같은 경고 메세지가 출력될 것이다. 이럴 때는 ```--save-config``` 옵션을 함께 사용하면 ```kubectl apply ``` 커맨드 실행 시 필요로 하는 어노테이션이 자동으로 설정되기 때문에 더이상 경고 메세지가 출력되지 않을 것이다.
```
kubectl apply -f deploy/example-config.yml

Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply configmap "example-conifg" configured
```

### 4. ConfigMap 컨테이너 이미지에 노출 및 전달하기
A. 환경변수
Pod 사양을 정의할 때 필수적으로 포함되어야 할 name과 image 외에도 env를 개별적으로 정의할 수 있다. 
```
env:
  - name: LOG_LEVEL_KEY
    valueForm:
      configMapKeyRef:
        name: example-config
        Key: log. level
```

개별적으로 정의하는 것 대신 envFrom을 사용해 ConfigMap을 지정하는 것 또한 가능하며, 모든 Key/Value 환경변수로 사용된다.
```
envFrom:
    - configMapRef:
          name: example-config
```
B. 파일
어노테이션과 라벨을 컨테이너 내부에 파일로 노출시키는 방법과 유사한 점을 가지고 있는데, 아래의 예제에서 나타내는 바는 이렇다. (첫번째 부분은 볼륨의 이름과 마운트될 위치를 포함해 컨테이너 볼륨을 정의. 두번째는 볼륨에 대한 설명과 값을 가져올 ConfigMap 나열)
```
volumeMounts:
    - name: config
      mountPath: /etc/kconfig
      readOnly: true
```
```
volumes:
    - name: config
      configMap:
      name: example-config
```
### 5. ConfigMap 장점 아닌 장점 같은 장점
A. ConfigMap을 분리된 독립 실행형 객체 형태로 설정한다면, 동일한 이름의 ConfigMap에 대하여 각각 다른 매니패스트 (개발, 테스트, QA, 프로덕션)의 유지가 가능하진다. Pod는 이름으로 ConfigMap을 참조하기 때문에 각기 다른 환경에서 서로 다른 설정을 사용할 수 있다. (그림 7.3 참고)

B. ConfigMap은 외부 설정 파일을 통하여 설정 옵션을 저장할 수도 있다. ~~(어떻게? 감이 안온다면 예제를 보자.)~~
```
kubectl create configmap my-config --from-file = config-file.conf
```

C. ConfigMap은 디렉토리에 있는 파일로부터 설정 옵션을 저장할 수도 있다. (예제를 보자.)
```
kubectl create configmap my-config --from-file =/path/to/dir
```

D. ConfigMap은 B+C and etc. 를 활용하여 옵션을 조합하고 결합할 수 있다. (그림 7.5 참고)
```
kubectl create configmap my-config 
  --from-file =foo.json
  --from-file =bar=foobar.conf
  --from-file =config-opts/
  --from-file =some=thing
```



참조:
https://bcho.tistory.com/1267
Kubernets in Action 도서
개발자를 위한 쿠버네티스 도서
