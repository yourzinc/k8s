## 4장 : 컨피그맵과 비밀값으로 애플리케이션 설정하기

쿠버네티스에서 컨테이너에 설정값을 주입하는데 사용하는 리소스: ConfigMap, Secret 

ConfigMap, Secret 모두 포맷 제한 없이 데이터 보유 

클러스터 속에서 다른 리소스와 구분되어 독립된 장소에 보관됨

### 4.1 쿠버네티스에서 애플리케이션에 설정이 전달되는 과정

ConfigMap, Secret 은 적은 양의 데이터를 저장하는 것이 목적

컨테이너는 ConfigMap, Secret 에 저장된 데이터를 읽을 수 있다.

c.f 환경 변수

> 환경 변수는 컴퓨터 단위로 설정되며 모든 애플리케이션이 이 값을 읽을 수 있다. 모든 컨테이너에서 쿠버네티스나 컨테이너 속 운영체제가 한두 가지 값을 설정한다.

- 설정값 없이 sleep 이미지로 파드 실행
```
$ kubectl apply -f sleep/sleep.yaml

//
deployment.apps/sleep created
```

- pod 가 준비될 때까지 대기
```
$ kubectl wait --for=condition=Ready pod -l app=sleep

//
pod/sleep-7df75f8569-nljfb condition met
```

- 파드 속 컨테이너에 설정된 몇 가지 환경 변수의 값을 확인
```
$ kubectl exec deploy/sleep -- printenv HOSTNAME KIAMOL_CHAPTER

//
sleep-7df75f8569-nljfb
command terminated with exit code 1
```

쿠버네티스에서도 설정값을 주입하는 가장 간단한 방법은 파드 정의에 환경 변수를 추가하는 것이다.

환경 변수는 파드의 생애 주기 내내 변하지 않는다. 파드가 실행되는 중에는 환경 변수의 값을 수정할 수 없다. 

> sleep deployment 의 pod 정의를 수정하여 새로운 환경 변수를 추가하라

- deployment 업데이트
```
$ kubectl apply -f sleep/sleep-with-env.yaml

//
deployment.apps/sleep configured
```

- 환경 변수 값 확인
```
$ kubectl exec deploy/sleep -- printenv HOSTNAME KIAMOL_CHAPTER

//
sleep-74fbb9d7cd-fjgfr
04
```

애플리케이션의 설정값이 복잡할 때는 환경 변수가 아닌 ConfigMap 을 사용한다

**ConfigMap**
파드에서 읽어 들이는 데이터를 저장하는 리소스
데이터 형태는 key-value, text, binary file 등 다양하다

파드 하나에 여러 개의 컨피그맵을 전달할 수 있고, 하나의 컨피그맵을 여러 파드에 전달할 수 있다.

ConfigMap 은 읽기 전용이다. 파드에서 ConfigMap 의 내용은 수정할 수 없다.

> 명령행 도구로 데이터를 입력하여 ConfigMap 을 생성하라. 그리고 데이터를 확인한 후 이 컨피그맵을 사용하도록 수정된 sleep 애플리케이션을 배치하라.

- 명령행 도구를 사용하여 컨피그맵 생성
```
$ kubectl create configmap sleep-config-literal --from-literal=kiamol.section='4.1'

//
configmap/sleep-config-literal created
```

- 컨피그맵에 들어 있는 데이터 확인
```
$ kubectl get cm sleep-config-literal

//
NAME                   DATA   AGE
sleep-config-literal   1      46s
```

- 컨피그맵의 상세 정보를 보기 좋게 출력
```
$ kubectl describe cm sleep-config-literal

//
Name:         sleep-config-literal
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
kiamol.section:
----
4.1

BinaryData
====

Events:  <none>
```

- 파드 정의 수정 update 
```
$ kubectl apply -f sleep/sleep-with-configMap-env.yaml

//
deployment.apps/sleep configured
```

- 파드 속 환경 변수가 적용되었는지 확인
```
$ kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'

//
KIAMOL_SECTION=4.1
KIAMOL_CHAPTER=04
```

설정이 몇 가지 없는 경우에는 리터럴로 컨피그맵을 생성해도 충분하다.


### 4.2 컨피그맵에 저장한 설정 파일 사용하기

> ch04.env 로 ConfigMap 을 생성하고, 이 설정을 사용하도록 sleep application 을 업데이트해라.

- env 파일의 내용으로 컨피그맵 생성
```
$ kubectl create configmap sleep-config-env-file --from-env-file=sleep/ch04.env

//
configmap/sleep-config-env-file created
```

- ConfigMap 의 상세 정보 확인
```
$ kubectl get cm sleep-config-env-file

//
NAME                    DATA   AGE
sleep-config-env-file   3      41s
```

```
$ kubectl describe cm sleep-config-env-file

//
Name:         sleep-config-env-file
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
KIAMOL_CHAPTER:
----
ch04
KIAMOL_EXERCISE:
----
try it now
KIAMOL_SECTION:
----
ch04-4.1

BinaryData
====

Events:  <none>
```

- 새로운 configMap 의 설정을 적용하여 파드 업데이트
```
$ kubectl apply -f sleep/sleep-with-configMap-env-file.yaml

//
deployment.apps/sleep configured
```

- 컨테이너에 적용된 환경 변수의 값 확인
```
$ kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'

//
KIAMOL_EXERCISE=try it now
KIAMOL_SECTION=4.1
KIAMOL_CHAPTER=04
```

환경 변수 이름이 중복되는 경우 
-> env 항목에서 정의된 값 > envFrom 항목에서 정의된 값

### 4.3 컨피그맵에 담긴 설정값 데이터 주입하기

### 4.4 비밀값을 이용하여 민감한 정보가 담긴 설정값 다루기

### 4.5 쿠버네티스의 애플리케이션 설정 관리

### 연습 문제