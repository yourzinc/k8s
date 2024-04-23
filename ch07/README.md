## 7장 멀티컨테이너 파드를 이용하여 애플리케이션 확장하기

여러 개의 컨테이너를 실행하는 멀티컨테이너 파드 

application container 와 helper container

한 pod 안에 있는 container 는 같은 가상 환경을 공유한다

### 7.1 파드와 컨테이너의 통신

Pod : 하나 이상의 컨테이너가 공유하는 네트워크 및 파일 시스템을 제공하는 가상 환경

Container : 별도의 환경 변수와 자신만의 프로세스를 가지는, 서로 다른 기술 스택으로 구성된 별개의 이미지를 사용할 수 있는 독립된 단위

같은 pod 안에 있는 container 는 네트워크를 공유한다
- 모든 container 는 같은 IP 주소를 가진다 
- 같은 pod 속 container 간 통신은 localhost 주소를 사용한다

Container 는 자신만의 파일 시스템을 가진다.
Container 는 pod 에서 제공하는 volume 을 마운트 할 수 있다.
volume 을 공유하는 방식으로 container 끼리 정보를 교환할 수 있다.

> 두 컨테이너를 가진 파드의 정의를 배치하라

- Pod 배치 
```
$ kubectl apply -f sleep/sleep-with-file-reader.yaml

//
deployment.apps/sleep created
```

- Pod 상세 정보 확인
```
$ kubectl get pod -l app=sleep -o wide

//
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE             NOMINATED NODE   READINESS GATES
sleep-85d7576d8d-x7dwx   2/2     Running   0          42s   10.1.0.105   docker-desktop   <none>           <none>

```

- Container 이름 출력
```
$ kubectl get pod -l app=sleep -o jsonpath='{.items[0].status.containerStatuses[*].name}'

//
file-reader sleep
```

- pod 의 log 확인 - 오류 발생
```
$ kubectl logs -l app=sleep

//
Defaulted container "sleep" out of: sleep, file-reader
```

- pod 속 container 의 log 확인 
```
$ kubectl logs -l app=sleep -c sleep   
```

> 한 컨테이너에는 쓰기 가능, 다른 컨테이너에는 읽기 전용으로 볼륨이 마운트되었다. 따라서 한쪽에서는 파일을 기록하고 다른 쪽에서는 기록된 파일을 읽을 수 있다

- 한쪽 컨테이너에서 공유 볼륨으로 파일을 기록
```
$ kubectl exec deploy/sleep -c sleep -- sh -c 'echo ${HOSTNAME} > /data-rw/hostname.txt'

//
```

- 같은 컨테이너에서 기록한 파일을 읽음
```
$ kubectl exec deploy/sleep -c sleep -- cat /data-rw/hostname.txt

//
sleep-85d7576d8d-x7dwx
```

- 다른 쪽 컨테이너에서 기록한 파일을 읽음
```
$ kubectl exec deploy/sleep -c file-reader -- cat /data-ro/hostname.txt

//
sleep-85d7576d8d-x7dwx
```

- 읽기 전용으로 볼륨을 마운트한 컨테이너에서 파일을 쓰려고 하면 오류 발생
```
$ kubectl exec deploy/sleep -c file-reader -- sh -c 'echo more >> /data-ro/hostname.txt'

//
sh: can't create /data-ro/hostname.txt: Read-only file system
command terminated with exit code 1
```

volume 은 pod 수준에서 정의되고 container 수준에서 마운트된다

volume, volume claim 도 여러 container 에 마운트될 수 있다

> sleep deployment 를 예제 7-2의 정의로 업데이트하라. 그리고 서버 컨테이너에 접근이 가능한지 확인하라

- pod 업데이트
```
$ kubectl apply -f sleep/sleep-with-server.yaml

//
deployment.apps/sleep configured
```

```
$ kubectl get pods -l app=sleep

//
NAME                     READY   STATUS    RESTARTS   AGE
sleep-5dfc8745b7-xhst2   2/2     Running   0          3m3s
```

- pod 속 컨테이너 이름 확인
```
$ kubectl get pod -l app=sleep -o jsonpath='{.items[0].status.containerStatuses[*].name}'

//
server sleep
```

- sleep container 에서 server container 로 통신
```
$ kubectl exec deploy/sleep -c sleep -- wget -q -O - localhost:8080

//
kiamol
```

- server container 의 log 확인
```
$ kubectl logs -l app=sleep -c server

//

GET / HTTP/1.1
Host: localhost:8080
User-Agent: Wget
Connection: close
```

트래픽을 파드의 특정 포트로 전달하는 서비스를 만들면 이 포트를 주시하는 컨테이너가 요청을 전달받는다

> kubectl 을 사용하여 파드의 포트를 개방한 후 (YAML 정의 없이도 빠르게 서비스를 만드는 방법), 파드 외부에서 HTTP 서버 컨테이너에 접근이 가능한지 확인하라

- server container 의 포트를 가리키는 service 를 생성
```
$ kubectl expose -f sleep/sleep-with-server.yaml --type LoadBalancer --port 8020 --target-port 8080

//
service/sleep exposed
```

- 서비스의 URL 을 출력
```
$ kubectl get svc sleep -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8020'

//
http://localhost:8020
```

- server container log 확인
```
$ kubectl logs -l app=sleep -c server

//
GET / HTTP/1.1
Host: localhost:8080
User-Agent: Wget
Connection: close
```

Pod 는 애플리케이션을 구성하는 한 단위이다.

Pod 는 애플리케이션의 단일 컴포넌트에 대응해야 한다.

한 Pod 안에 서로 다른 애플리케이션을 함께 넣으면, 각각의 컴포넌트를 독립적으로 업데이트 또는 스케일링하거나 관리할 수 없게 된다.


### 7.2 초기화 컨테이너를 이용한 애플리케이션 시작

Sidecar pattern
- 추가 컨테이너 (sidecar) 가 애플리케이션 컨테이너 (motorcycle) 를 지원하는 구도 

init container (초기화 컨테이너) : Application container 보다 먼저 실행하여 애플리케이션 실행 준비를 하는 container

Init container
- 파드 안에 여러 개 정의 가능
- 파드 정의에 기재된 순서대로 실행
- 각각의 초기화 컨테이너는 정해진 목표를 달성해야 다음 초기화 컨테이너를 실행 (init-1 -> init-2 -> init-3)
- 모든 초기화 컨테이너가 목표를 달성한 이후 application container 나 sidecar container 를 실행 

모든 Application container 의 준비가 끝나야 Pod 상태가 Ready 된다 

Init container 는 pod 가 시작하기 전 application container 에 필요한 환경을 준비하는데 유용하다
e.g. Init container 에서 공유된 볼륨에 git 저장소를 복제 -> application container 에서 복제된 저장소에 접근

> 초기화 컨테이너가 동작하는 과정 

- init container 배치 
```
$ kubectl apply -f sleep/sleep-with-html-server.yaml

//
deployment.apps/sleep configured
```

- pod container 확인
```
$ kubectl get pod -l app=sleep -o jsonpath='{.items[0].status.containerStatuses[*].name}'

//
server sleep
```

- init container 확인
```
$ kubectl get pod -l app=sleep -o jsonpath='{.items[0].status.initContainerStatuses[*].name}'

//
init-html
```

- init container 의 Log 확인 - Log 없음
```
$ kubectl logs -l app=sleep -c init-html

//
```

- sidecar container 에서 init container 가 생성한 파일에 접근 가능한지 확인
```
$ kubectl exec deploy/sleep -c server -- ls -l /data-ro

//
total 4
-rw-r--r--    1 root     root            62 Apr 22 14:20 index.html
```

> 애플리케이션의 기능은 몇 초마다 한 번씩 현재 시각의 타임스탬프를 로그 파일에 기록하는 것이다. 또한 구식 설정 방식을 취하고 있어 기존에 배운 애플리케이션 설정 방법을 쓸 수 없다. 

- 단일 설정 파일만 사용하는 application 실행
```
$ kubectl apply -f timecheck/timecheck.yaml

//
deployment.apps/timecheck created
```

- container log 확인 - log X
```
$ kubectl logs -l app=timecheck

//
```

- container 내부에서 log 확인 
```
$ kubectl exec deploy/timecheck -- cat /logs/timecheck.log

//
2024-04-22 14:29:49.718 +00:00 [INF] Environment: DEV; version: 1.0; time check: 14:29.49
2024-04-22 14:29:54.701 +00:00 [INF] Environment: DEV; version: 1.0; time check: 14:29.54
2024-04-22 14:29:59.702 +00:00 [INF] Environment: DEV; version: 1.0; time check: 14:29.59
2024-04-22 14:30:04.701 +00:00 [INF] Environment: DEV; version: 1.0; time check: 14:30.04
2024-04-22 14:30:09.701 +00:00 [INF] Environment: DEV; version: 1.0; time check: 14:30.09
```

- Application 설정 확인
```
$ kubectl exec deploy/timecheck -- cat /config/appsettings.json

//
{
  "Application": {
    "Version": "1.0",
    "Environment": "DEV"
  },
  "Timer": {
    "IntervalSeconds": "5"
  },
  "Metrics": {
    "Enabled": false,
    "Port" : 8080
  }
}
```

Init container 가 설정값 (configMap, Secret, Env) 를 읽어 설정을 구성하고, 구성된 설정 파일을 application 의 설정 파일 경로에 생성할 수 있다.

- init container 에서 실행되는 명령어 : configMap volume mount 에서 설정을 읽은 후 환경 변수의 설정값을 병합하여 구성한 설정을 emptyDir volume mount 에 파일로 기록

- container 는 각자의 환경 변수를 공유하지 않는다

- container 는 역할에 따라 필요한 volume 만 mount 한다 

> timecheck application 이 여러 출처의 설정값을 사용하도록 init container 설정 추가 
- configMap, deployment 정의 
```
$ kubectl apply -f timecheck/timecheck-configMap.yaml -f timecheck/timecheck-with-config.yaml

//
configmap/timecheck-config created
deployment.apps/timecheck configured
```

- container 준비될 때까지 대기
```
$ kubectl wait --for=condition=ContainersReady pod -l app=timecheck,version=v2

//
pod/timecheck-8585b6f98b-5l6qd condition met
```

- 새로운 application container log 확인
```
$ kubectl exec deploy/timecheck -- cat /logs/timecheck.log

//
Defaulted container "timecheck" out of: timecheck, init-config (init)
2024-04-22 14:43:59.024 +00:00 [INF] Environment: TEST; version: 1.1; time check: 14:43.59
2024-04-22 14:44:06.044 +00:00 [INF] Environment: TEST; version: 1.1; time check: 14:44.06
2024-04-22 14:44:13.027 +00:00 [INF] Environment: TEST; version: 1.1; time check: 14:44.13
2024-04-22 14:44:20.012 +00:00 [INF] Environment: TEST; version: 1.1; time check: 14:44.20
2024-04-22 14:44:27.010 +00:00 [INF] Environment: TEST; version: 1.1; time check: 14:44.27
```

- init container 가 생성한 설정 파일을 확인
```
$ kubectl exec deploy/timecheck -- cat /config/appsettings.json

//
Defaulted container "timecheck" out of: timecheck, init-config (init)
{
  "Application": {
    "Version": "1.1",
    "Environment": "TEST"
  },
  "Timer": {
    "IntervalSeconds": "7"
  }
}
```

### 7.3 어댑터 컨테이너를 이용한 일관성 있는 애플리케이션 관리

Sidecar container 를 adapter 역할로 사용 (Application 과 container platform 사이를 중재)
e.g. logging 

Node.js, .Net Core - 표준 출력 스트림에 로그를 출력
Docker, k8s - 표준 출력의 container 로그를 수집

파일에 직접 로그를 남기거나, container log 가 수집될 수 없는 채널을 이용해 로그를 남기는 경우 pod 의 로그를 볼 수 없다
-> sidercar container 로 해결한다

> Sidecar 로 logging 설정을 적용한 후 application log 확인
- sidecar container 추가
```
$ kubectl apply -f timecheck/timecheck-with-logging.yaml

//
deployment.apps/timecheck configured
```

- container 준비될 때까지 wait
```
$ kubectl wait --for=condition=ContainersReady pod -l app=timecheck,version=v3

//
pod/timecheck-6795bf78fb-5w8ws condition met
```

- pod 상태 확인
```
$ kubectl get pods -l app=timecheck

//
NAME                         READY   STATUS    RESTARTS   AGE
timecheck-6795bf78fb-5w8ws   2/2     Running   0          71s
```

- pod 속 container 의 상태 확인
```
$ kubectl get pod -l app=timecheck -o jsonpath={'.items[0].status.containerStatuses[*].name}'

//
logger timecheck
```

- Pod 의 Log 를 확인할 수 있다
```
$ kubectl logs -l app=timecheck -c logger

//
2024-04-22 15:05:32.441 +00:00 [INF] Environment: TEST; version: 1.1; time check: 15:05.32
2024-04-22 15:05:39.441 +00:00 [INF] Environment: TEST; version: 1.1; time check: 15:05.39
2024-04-22 15:05:46.442 +00:00 [INF] Environment: TEST; version: 1.1; time check: 15:05.46
2024-04-22 15:05:53.441 +00:00 [INF] Environment: TEST; version: 1.1; time check: 15:05.53
2024-04-22 15:06:00.442 +00:00 [INF] Environment: TEST; version: 1.1; time check: 15:06.00
```

Application 자체를 수정하지 않고도, k8s 를 통해 Application log 를 수집할 수 있다
Application container 의 파일 시스템에 기록된 로그가 sidecar container 를 통해 파드에서 수집된다. 
-> Sidecar container 가 adapter 역할을 한다. 

Platform 을 통해 설정을 주입하고, Platform 에 Log 를 출력하는 것은 container platform 으로 기본적인 사항이다 

Sidecar 가 유용한 경우
- Container HealthCheck
- Application 상태나 성능 파악하는 지표 수집 

> Sidecar (HealthChecker, Monitoring) 를 추가하고 H/C API 와 성능 지표 API 를 확인
- pod update
```
$ kubectl apply -f timecheck/timecheck-good-citizen.yaml

//
service/timecheck unchanged
deployment.apps/timecheck configured
```

- container 준비될 때까지 wait
```
$ kubectl wait --for=condition=ContainersReady pod -l app=timecheck,version=v3

//
pod/timecheck-5486dc68f6-qgsbk condition met
```

- container 상태 확인
```
$ kubectl get pod -l app=timecheck -o jsonpath='{.items[0].status.containerStatuses[*].name}'

//
healthz logger metrics timecheck
```

- sleep container 에서 timecheck application 의 health check API 사용
```
$ kubectl exec deploy/sleep -c sleep -- wget -q -O - http://timecheck:8080

//
{"status": "OK"}
```

- sleep container 에서 성능 지표 API 
```
$ kubectl exec deploy/sleep -c sleep -- wget -q -O - http://timecheck:8081

//
# HELP timechecks_total The total number timechecks.
# TYPE timechecks_total counter
timechecks_total 6
```

Adaptor 역할을 하는 sidecar 는 Overhead 로 작용한다

Sidecar container 를 이용하여 Application 의 network 통신을 세세하게 제어할 수 있다
- Application container 에서 외부를 향하는 트래픽을 관리하는 proxy container 를 둔다

### 7.4 외부와의 통신을 추상화하기: 앰배서더 컨테이너

Ambassador container
- Application 과 외부와의 통신을 제어하고 단순화하는 역할 

e.g. Application 이 localhost 주소로 요청을 전달하면, Ambassador container 가 받아 처리하는 형태이다

1. Application 전체가 사용하는 일반 Ambassador container
2. 특정 application component 의 통신만 처리하는 ambassador container

Proxy container 활용
- service discovery
- load balancing
- connection retry
- encryption 

> 무작위 숫자 생성 application 을 실행하고, web application container 에서 아무 주소나 통신을 보낼 수 있는지 확인

- Application, Service 배치
```
$ kubectl apply -f numbers/

//
service/numbers-api created
deployment.apps/numbers-api created
service/numbers-web created
deployment.apps/numbers-web created
```

- Application 에 접근할 URL 확인
```
$ kubectl get svc numbers-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8090'

//
http://localhost:8090
```

- Application 에 접근하여 무작위 숫자 생성

- Web application 에서 다른 endpoint 에 접근 가능한지 확인
```
$ kubectl exec deploy/numbers-web -c web -- wget -q -O - http://timecheck:8080

//
{"status": "OK"}
```

Web application container 는 어떤 주소에도 제한 없이 접근할 수 있다.
공격자가 이 점을 악용하면 cluster 에 동작하는 다른 application 이 무엇인지 알아낼 수 있다.

> 무작위 숫자 애플리케이션에서 proxy container 를 추가하여 접근 가능한 주소에 제한이 생겼는지 확인하라

- update
```
$ kubectl apply -f numbers/update/web-with-proxy.yaml

//
deployment.apps/numbers-web configured
```

- web page 새로고침 후 숫자 생성

- Proxy container log 확인
```
$ kubectl logs -l app=numbers-web -c proxy

//
** Logging proxy listening on port: 1080 **
```

- timecheck application 의 H/C endpoint 에 접근
```
$ kubectl exec deploy/numbers-web -c web -- wget -q -O - http://timecheck:8080

//

```

- Proxy container log 확인
```
$ kubectl logs -l app=numbers-web -c proxy

//
** Logging proxy listening on port: 1080 **
```
### 7.5 파드 환경 이해하기

Pod 는 컴퓨팅의 기본 단위

Pod 안에 있는 모든 container 준비가 끝나야 pod 도 준비 상태가 된다

Service 는 준비된 pod 에 트래픽을 전달한다

Sidecar container, init container 는 Application 의 안전 모드를 추가하는 것과 같다

> init container 가 실패하면 application 은 고장난다. init container 의 설정 오류로 update 가 실패하는 것을 확인

- update
```
$ kubectl apply -f numbers/update/web-v2-broken-init-container.yaml

//
deployment.apps/numbers-web configured
```

- pod 확인
```
$ kubectl get po -l app=numbers-web,version=v2

//
NAME                           READY   STATUS                  RESTARTS      AGE
numbers-web-6d95474d7b-ds5vm   0/2     Init:CrashLoopBackOff   3 (46s ago)   97s

```

- 새로 생성된 init container 의 log 확인
```
$ kubectl logs -l app=numbers-web,version=v2 -c init-version

//
sh: can't create /config-out/version.txt: Read-only file system
```

- deployment 확인
```
$ kubectl get deploy numbers-web

//
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
numbers-web   1/1     1            1           2m29s
```

- replicaSet 상태 확인
```
$ kubectl get rs -l app=numbers-web

//
NAME                     DESIRED   CURRENT   READY   AGE
numbers-web-5ffbc8bf8c   1         1         1       2m7s
numbers-web-6d95474d7b   1         1         0       50s
numbers-web-c45b49bc8    0         0         0       2m37s
```

새 ReplicaSet 이 준비 상태가 되지 않았으므로 기존 ReplicaSet 이 아직 동작 중이다.

Pod 의 재시작 조건
- init container 를 가진 pod 가 대체되는 경우
    - new pod 는 init container 를 모두 실행한다 -> init logic 은 반복적으로 실행 가능한 것이어야 한다

- init container 의 이미지를 변경하는 경우, pod 자체를 재시작한다
    - init container 는 다시 실행되며, application container 도 교체된다

- pod 정의에서 application container 이미지를 변경한 경우
    - application container 가 대체 된다
    - init container 는 재시작 X

- Application container 가 종료된 경우 
    - pod 가 application container 를 재생성한다 
    - 모든 container 가 준비될 때까지 service 에서 트래픽을 전달받지 못한다 

Pod 속 container 는 network 주소와 file system 일부를 공유할 수 있지만, process 에는 접근할 수 없다.

Container 가 Application process 에 접근해야 하는 경우
- process 간 통신
- application process metrics 수집

-> `shareProcessNamespace: true`
Pod 의 모든 컨테이너가 컴퓨팅 공간을 공유하며 서로의 process 를 본다 

> Conatiner 가 computing 공간을 공유하도록 sleep pod 를 업데이트하고, 다른 container 의 process 에 접근이 가능한지 확인

- 현재 container 의 process 확인 
```
$ kubectl exec deploy/sleep -c sleep -- ps

//
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh -c trap : TERM INT; (while true; do sleep 1000; done) & wait
    7 root      0:00 /bin/sh -c trap : TERM INT; (while true; do sleep 1000; done) & wait
    8 root      0:00 sleep 1000
   33 root      0:00 ps

```

- Update pod
```
$ kubectl apply -f sleep/sleep-with-server-shared.yaml

//
deployment.apps/sleep configured
```

- container ready
```
$ kubectl wait --for=condition=ContainersReady pod -l app=sleep,version=shared

//
pod/sleep-67946966c5-4sp65 condition met
```

- Check process 
```
$ kubectl exec deploy/sleep -c sleep -- ps 

//
PID   USER     TIME  COMMAND
    1 65535     0:00 /pause
    7 root      0:00 /bin/sh -c trap : TERM INT; (while true; do sleep 1000; done) & wait
   13 root      0:00 /bin/sh -c trap : TERM INT; (while true; do sleep 1000; done) & wait
   14 root      0:00 sleep 1000
   15 root      0:00 sh -c while true; do echo -e 'HTTP/1.1 200 OK Content-Type: text/plain Content-Length: 7  kiamol' | nc -l -p 8080; done
   22 root      0:00 nc -l -p 8080
   23 root      0:00 ps
```

sleep container 에서 server container 의 process 를 모두 볼 수 있다.

모든 process 를 종료시켜 pod 를 망가뜨릴 수도 있다.

> Resource 정리
```
$ kubectl delete all -l kiamol=ch07

//
service "numbers-api" deleted
service "numbers-web" deleted
service "sleep" deleted
service "timecheck" deleted
deployment.apps "numbers-api" deleted
deployment.apps "numbers-web" deleted
deployment.apps "sleep" deleted
deployment.apps "timecheck" deleted
```
