## 9장 롤아웃과 롤백을 이용한 애플리케이션 릴리즈 관리

상황에 따라 업데이트 중지 또는 롤백이 가능한 롤링 업데이트를 적요해야 함

Rolling update 를 지원하는 Kubernetes Resources
- Deployment
- DemonSet
- StatefulSet

Kubernetes deployment strategy  

Application 중단 시간의 가장 큰 발생 원인은 Update 

### 9.1 쿠버네티스의 롤링 업데이트 

Deployment Rolling Update (Roll-out)

- 새로운 ReplicaSet 을 만들어 Replica 수를 지정된 수만큼 늘린다
- 기존 ReplicaSet 의 Replica 수를 0으로 낮춘다
- 업데이트 완료 후에도 빈 ReplicaSet 은 그대로 남는다


Roll-out 은 pod 의 정의가 변경될 때만 일어난다 

> Replica 수가 2 개인 간단한 Application 을 배치하고 Application 을 스케일링한 후 ReplicaSet 에 어떤 일이 일어나는지 관찰하라 

- Application
```
$ kubectl apply -f vweb/

//
service/vweb created
deployment.apps/vweb created
```

- ReplicaSet 
```
$ kubectl get rs -l app=vweb

//
NAME              DESIRED   CURRENT   READY   AGE
vweb-6ddb844d69   2         2         2       5m47s
```

- Application Scaling 
```
$ kubectl apply -f vweb/update/vweb-v1-scale.yaml

//
deployment.apps/vweb configured
```

- ReplicaSet
```
$ kubectl get rs -l app=vweb

//
NAME              DESIRED   CURRENT   READY   AGE
vweb-6ddb844d69   3         3         3       7m4s
```

- Deployment Roll-out history
```
$ kubectl rollout history deploy/vweb

//
deployment.apps/vweb 
REVISION  CHANGE-CAUSE
1         <none>
```

`kubectl rollout` : 롤아웃을 관리하고 정보를 확인하는 명령어

- Scaling update 가 ReplicaSet 숫자만 변경했기 때문에 두 번째 rollout 은 일어나지 않는다


> kubectl set 명령을 사용하여 Deployment 의 image version 을 변경하라. Pod 의 정의가 변경된 경우 rollout 이 발생한다

- Update
```
$ kubectl set image deployment/vweb web=kiamol/ch09-vweb:v2

//
deployment.apps/vweb image updated
```

- ReplicaSet
```
$ kubectl get rs -l app=vweb

//
NAME              DESIRED   CURRENT   READY   AGE
vweb-6ddb844d69   1         1         1       9m47s
vweb-7788cdb778   3         3         2       18s

//
NAME              DESIRED   CURRENT   READY   AGE
vweb-6ddb844d69   0         0         0       9m49s
vweb-7788cdb778   3         3         3       20s
```

- Rollout history
```
$ kubectl rollout history deploy/vweb

//

deployment.apps/vweb 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

`kubectl set` : Pod 의 image 나 환경 변수 또는 service 의 selector 등을 변경할 수 있음

Rollout 
- kubernetes 의 리소스는 아니지만, Application Release 관리에 중요한 도구
- Release history 를 관리하고 이전 릴리즈로 돌아갈 수 있음


### 9.2 롤아웃과 롤백을 이용한 디플로이먼트 업데이트 

ReplicaSet 에 version number (또는 git commit ID) 를 넣은 레이블을 정의하면 Application 의 update 내용을 추적하기 수월하다

> version 레이블을 수정하여 deployment 를 변경. Pod 정의 변경은 Rollout 이 발생
- Deployment (Record option)
```
$ kubectl apply -f vweb/update/vweb-v11.yaml --record

//
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/vweb configured
```

- ReplicaSet (Check lables)
```
$ kubectl get rs -l app=vweb --show-labels

//
NAME              DESIRED   CURRENT   READY   AGE     LABELS
vweb-5f9bd55d54   3         3         3       46s     app=vweb,pod-template-hash=5f9bd55d54,version=v1.1
vweb-6ddb844d69   0         0         0       18m     app=vweb,pod-template-hash=6ddb844d69,version=v1
vweb-7788cdb778   0         0         0       8m34s   app=vweb,pod-template-hash=7788cdb778,version=v1
```

- Rollout 
```
$ kubectl rollout status deploy/vweb

//
deployment "vweb" successfully rolled out
```

- Rollout history 
```
$ kubectl rollout history deploy/vweb

//
deployment.apps/vweb 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         kubectl apply --filename=vweb/update/vweb-v11.yaml --record=true
```

- ReplicaSet Rollout revision 
```
$ kubectl get rs -l app=vweb -o=custom-columns=NAME:.metadata.name,REPLICAS:.status.replicas,REVISION:.metadata.annotations.deployment\.kubernetes\.io/revision


//
NAME              REPLICAS   REVISION
vweb-5f9bd55d54   3          <none>
vweb-6ddb844d69   0          <none>
vweb-7788cdb778   0          <none>
```

`record` : Rollout 을 실행시킨 kubectl 명령을 저장하는 기능

> Web Application 에 요청을 보내고 응답을 확인하라. 그다음 Application 을 업데이트하고 Application 의 응답을 확인하라

- URL 
```
$ kubectl get svc vweb -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8090/v.txt' > url.txt 
```

- HTTP Request 
```
$ curl $(cat url.txt) -UseBasicParsing

//
v1
```

- Update v2
```
$ kubectl apply -f vweb/update/vweb-v2.yaml --record

//
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/vweb configured
```

- HTTP Request 
```
$ curl $(cat url.txt) -UseBasicParsing

// 
v2
```

- ReplicaSet 
```
$ kubectl get rs -l app=vweb --show-labels

//
NAME              DESIRED   CURRENT   READY   AGE   LABELS
vweb-5f9bd55d54   0         0         0       13m   app=vweb,pod-template-hash=5f9bd55d54,version=v1.1
vweb-65d7d8f885   3         3         3       63s   app=vweb,pod-template-hash=65d7d8f885,version=v2
vweb-6ddb844d69   0         0         0       30m   app=vweb,pod-template-hash=6ddb844d69,version=v1
vweb-7788cdb778   0         0         0       21m   app=vweb,pod-template-hash=7788cdb778,version=v1
```

kubernetes 의 rollout 관리 기능이 있지만, 식별에 필요한 label 을 따로 추가하는 편이 좋다.

> Rollout history 를 확인하고 Application 을 버전 v1 로 rollback
- Revision history 
```
$ kubectl rollout history deploy/vweb

//

deployment.apps/vweb 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         kubectl apply --filename=vweb/update/vweb-v11.yaml --record=true
4         kubectl apply --filename=vweb/update/vweb-v2.yaml --record=true
```

- ReplicaSet Revison
```
$ kubectl get rs -l app=vweb -o=custom-columns=NAME:.metadata.name,REPLICAS:.status.replicas,VERSION:.metadata.labels.version,REVISION:.metadata.annotations.deployment\.kubernetes\.io/revision

//

NAME              REPLICAS   VERSION   REVISION
vweb-5f9bd55d54   0          v1.1      <none>
vweb-65d7d8f885   3          v2        <none>
vweb-6ddb844d69   0          v1        <none>
vweb-7788cdb778   0          v1        <none>
```

- Rollback 예측 결과 확인
```
$ kubectl rollout undo deploy/vweb --dry-run=client

//
deployment.apps/vweb Pod Template:
  Labels:	app=vweb
	pod-template-hash=5f9bd55d54
	version=v1.1
  Containers:
   web:
    Image:	kiamol/ch09-vweb:v1
    Port:	80/TCP
    Host Port:	0/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
 (dry run)
```

- Revision v2 rollback
```
$ kubectl rollout undo deploy/vweb --to-revision=2

//
deployment.apps/vweb rolled back
```

- Application 상태 확인 - 깜짝 놀랄 결과
```
$ curl $(cat url.txt) 

//
v2
```

Release 절차
- Deployment 는 ReplicaSet 을 생성 or 재활용, Pod 를 필요에 따라 조정
- ReplicaSet 에 변동이 생기면 rollout 으로 기록

> 기존 Deployment 를 제거해서 Revision history 를 정리한 후 ConfigMap 을 사용하는 새로운 버전의 Deployment 를 배치한다. 그리고 CongifMap 을 업데이트한 후 결과를 확인

- Delete deploy
```
$ kubectl delete deploy vweb

//
deployment.apps "vweb" deleted
```

- ConfigMap 
```
$ kubectl apply -f vweb/update/vweb-v3-with-configMap.yaml --record

//
Flag --record has been deprecated, --record will be removed in the future
configmap/vweb-config created
deployment.apps/vweb created
```

- curl 
```
$ curl $(cat url.txt)

//
v3-from-config
```

- ConfigMap update
```
$ kubectl apply -f vweb/update/vweb-configMap-v31.yaml --record
$ sleep 120

//
Flag --record has been deprecated, --record will be removed in the future
configmap/vweb-config configured
```

- curl
```
$ curl $(cat url.txt)

//
v3.1
```

- Rollout history
```
$ kubectl rollout history deploy/vweb

//
deployment.apps/vweb 
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=vweb/update/vweb-v3-with-configMap.yaml --record=true

```

**Hot reload** 
기존 Pod와 Container 가 그대로 남아 있어 서비스 중단의 위험이 없다.
e.g. 설정값만 업데이트하면 Application 동작을 변화시킬 수 있지만, rollout 기록이 남지 않는다 

c.f ConfigMap, Secret 을 불변적 요소로 간주하여, 새로운 객체를 참조하도록 Deployment 를 업데이트 하는 방식

> 불변적 설정값 객체를 사용하는 새로운 버전의 Application 을 배치하고 Release 절차를 비교 
- delete
```
$ kubectl delete deploy vweb

//
deployment.apps "vweb" deleted
```

- 불변적 설정값 객체 Deployment 
```
$ kubectl apply -f vweb/update/vweb-v4-with-configMap.yaml --record

//
Flag --record has been deprecated, --record will be removed in the future
configmap/vweb-config-v4 created
deployment.apps/vweb created
```

- Application 응답 확인
```
$ curl $(cat url.txt) 

//
v4-from-config
```

- 새로운 ConfigMap 배치, Deploymlent update
```
$ kubectl apply -f vweb/update/vweb-v41-with-configMap.yaml --record

//
Flag --record has been deprecated, --record will be removed in the future
configmap/vweb-config-v41 created
deployment.apps/vweb configured
```

- Application 응답 확인
```
$ curl $(cat url.txt) 

//
v4.1-from-config
```

- Rollout
```
$ kubectl rollout history deploy/vweb

//
deployment.apps/vweb 
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=vweb/update/vweb-v4-with-configMap.yaml --record=true
2         kubectl apply --filename=vweb/update/vweb-v41-with-configMap.yaml --record=true
```

- Rollback 
```
$ kubectl rollout undo deploy/vweb

//
deployment.apps/vweb rolled back
```
```
$ kubectl rollout history deploy/vweb

//
zinc@zincs-MacBook-Air ch09 % kubectl rollout history deploy/vweb
deployment.apps/vweb 
REVISION  CHANGE-CAUSE
2         kubectl apply --filename=vweb/update/vweb-v41-with-configMap.yaml --record=true
3         kubectl apply --filename=vweb/update/vweb-v4-with-configMap.yaml --record=true
```

- Application 응답 확인
```
$ curl $(cat url.txt) 

// 
v4-from-config
```

불변적 설정값 객체
- 설정값 변경만으로도 rollout history 를 남길 수 있음
- 설정값 변경할 때마다 rollout 이 일어남 


### 9.3 디플로이먼트의 롤링 업데이트 설정

**Deployment Strategy**
- Rollingupdate
- Recreate

**Recreate**
- 기존 ReplicaSet 의 Pod 수가 0까지 감소한 후 새로운 ReplicaSet 의 Pod 수가 증가한다 


```
# vweb-recreate-v2.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: vweb
spec:
  replicas: 3
  strategy:
    type: Recreate 
...
```

> recreate 를 사용하는 Deployment 
- delete
```
$ kubectl delete deploy vweb

//
deployment.apps "vweb" deleted
```

- apply Deployment (recreate)
```
$ kubectl apply -f vweb-strategies/vweb-recreate-v2.yaml

//
deployment.apps/vweb created
```

- ReplicaSet 
```
$ kubectl get rs -l app=vweb

//
NAME              DESIRED   CURRENT   READY   AGE
vweb-65d7d8f885   3         3         3       6s
```

- Test
```
$ curl $(cat url.txt) 

//
v2
```

- Deployment check
```
$ kubectl describe deploy vweb

//
Name:               vweb
Namespace:          default
CreationTimestamp:  Sun, 28 Apr 2024 21:52:21 +0900
Labels:             kiamol=ch09
Annotations:        deployment.kubernetes.io/revision: 1
Selector:           app=vweb
Replicas:           3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:       Recreate
MinReadySeconds:    0
Pod Template:
  Labels:  app=vweb
           version=v2
  Containers:
   web:
    Image:        kiamol/ch09-vweb:v2
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   vweb-65d7d8f885 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set vweb-65d7d8f885 to 3
```
> 오류가 있는 버전의 Application 을 배치

- update Deployment
```
$ kubectl apply -f vweb-strategies/vweb-recreate-v3.yaml

//
deployment.apps/vweb configured
```

- Rollout
```
$ kubectl rollout status deploy/vweb --timeout=2s

//
Waiting for deployment "vweb" rollout to finish: 0 of 3 updated replicas are available...
error: timed out waiting for the condition
```

- ReplicaSet
```
$ kubectl get rs -l app=vweb

//
NAME              DESIRED   CURRENT   READY   AGE
vweb-65d7d8f885   0         0         0       5m10s
vweb-698f584b8f   3         3         0       43s
```

- Application test (failed)
```
$ curl $(cat url.txt)

//
curl: (52) Empty reply from server
```

Rolling update 정의에서 기존/신규 ReplicaSet 의 Pod 스케일링 속도 조절 인자
- maxUnavailable
  - 기존 ReplicaSet 의 속도를 조절하는 값
  - 업데이트 동안 사용할 수 없는 pod 의 최대 수 
  - 기존 ReplicaSet 에서 동시에 종료되는 pod 수 
- maxSurge
  - 새로운 ReplicaSet 의 속도를 조절하는 값
  - 업데이트 동안 생기는 잉여 파드의 최개 개수 
  - 새 ReplicaSet 에서 동시에 함께 시작되는 Pod 수 

> Rolling update 전략으로 maxSurge=1, maxUnavailable=0 으로 설정하고 version 2 이미지를 사용하도록 Rollback
- v2 image deployment
```
$ kubectl apply -f vweb-strategies/vweb-rollingUpdate-v2.yaml

//
deployment.apps/vweb configured
```

- Application pod 
```
$ kubectl get po -l app=vweb

//
NAME                    READY   STATUS    RESTARTS   AGE
vweb-65d7d8f885-hr7wm   1/1     Running   0          9s
vweb-65d7d8f885-jmlv6   1/1     Running   0          7s
vweb-65d7d8f885-nwhdq   1/1     Running   0          8s
```

- Rollout
```
$ kubectl rollout status deploy/vweb

//
deployment "vweb" successfully rolled out
```
- ReplicaSet 
```
$ kubectl get rs -l app=vweb

//
NAME              DESIRED   CURRENT   READY   AGE
vweb-65d7d8f885   3         3         3       17m
vweb-698f584b8f   0         0         0       12m
```

- Application test
```
$ curl $(cat url.txt) 

//
v2
```

Rolling update 를 사용하여 Deployment 를 안전하게 유지하며 Application 을 보호하는 방법
- maxUnavailable=1, maxSurge=1

> Version 3 update. RollingUpdate 를 사용하므로 Application 전체가 고장나지 않는다

- Update with invalid image
```
$ kubectl apply -f vweb-strategies/vweb-rollingUpdate-v3.yaml

//
deployment.apps/vweb configured
```

- Pod
```
$ kubectl get po -l app=vweb

//
NAME                    READY   STATUS             RESTARTS     AGE
vweb-65d7d8f885-jmlv6   1/1     Running            0            9m8s
vweb-65d7d8f885-nwhdq   1/1     Running            0            9m9s
vweb-698f584b8f-2ldrx   0/1     CrashLoopBackOff   1 (7s ago)   8s
vweb-698f584b8f-rv9vl   0/1     CrashLoopBackOff   1 (7s ago)   8s


//
NAME                    READY   STATUS    RESTARTS      AGE
vweb-65d7d8f885-jmlv6   1/1     Running   0             9m18s
vweb-65d7d8f885-nwhdq   1/1     Running   0             9m19s
vweb-698f584b8f-2ldrx   0/1     Error     2 (17s ago)   18s
vweb-698f584b8f-rv9vl   0/1     Error     2 (17s ago)   18s
```

- ReplicaSet scaling 
```
$ kubectl get rs -l app=vweb

//
NAME              DESIRED   CURRENT   READY   AGE
vweb-65d7d8f885   2         2         2       26m
vweb-698f584b8f   2         2         0       21m
```

- Application test
```
$ curl $(cat url.txt) 

//
v2
```

Kubenetes 는 pod 가 실패하면 새로운 pod 를 만들어 계속 재시도하게 되는데, 이때 node 의 CPU 자원이 바닥나지 않도록 재시작 사이에 일정 시간 간격을 둔다

### 9.4 데몬셋과 스테이트풀셋의 롤링 업데이트

DemonSet, StratefulSet 의 업데이트 전략
- RollingUpdate
- OnDelete

OnDelete : 각 Pod 의 업데이트 시점을 직접 제어해야 할 때 사용하는 전략 
1. Controller 가 기존 Pod 를 종료하지 않고 그대로 둔 채 pod 를 주시
2. 다른 process 가 Pod 를 삭제하면 새로운 정의를 따른 대체 pod 를 생성하는 방식

OnDelete 전략을 사용하면 Pod 의 제거 시점은 사용자가 직접 통제하되 삭제된 pod 를 쿠버네티스가 자동으로 대체하게끔 할 수 있다.

> to-do Application 은 6개의 pod 로 구성. 기존 App 을 제거하여 클러스터 용량을 확보한 후 Application 을 실행
- delete
```
$ kubectl delete all -l kiamol=ch09

//
service "vweb" deleted
deployment.apps "vweb" deleted
```

- to-do Application, DB, reverse-proxy
```
$ kubectl apply -f todo-list/db/ -f todo-list/web/ -f todo-list/proxy/

//
configmap/todo-db-config created
configmap/todo-db-env created
configmap/todo-db-scripts created
secret/todo-db-secret created
service/todo-db created
statefulset.apps/todo-db created
configmap/todo-web-config created
secret/todo-web-secret created
service/todo-web created
deployment.apps/todo-web created
configmap/todo-proxy-configmap created
service/todo-proxy created
daemonset.apps/todo-proxy created
```

- Application URL
```
$ kubectl get svc todo-proxy -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8091'

//
http://localhost:8091
```

> DemonSet Rolling update

- DemonSet update
```
$ kubectl apply -f todo-list/proxy/update/nginx-rollingUpdate.yaml

//
daemonset.apps/todo-proxy configured
```

- Pod update check
```
$ kubectl get po -l app=todo-proxy --watch

//
NAME               READY   STATUS    RESTARTS   AGE
todo-proxy-cnwhq   1/1     Running   0          115s
```

DemonSet 의 업데이트는 기존 pod 를 먼저 제거한 후 대체 pod 가 생성되는 방식이다

StatefulSet 의 업데이트는 동시에 업데이트되는 파드 수는 항상 하나다. partiton 값을 사용하여 전체 Pod 중 업데이트 해야하는 Pod 의 비율은 설정할 수 있다.

> StatefulSet 에 partition 설정이 적용된 업데이트를 배치하라. 업데이트 결과는 파드 1만 업데이트되며 파드 0 은 업데이트되지 않는다.
- update
```
$ kubectl apply -f todo-list/db/update/todo-db-rollingUpdate-partition.yaml

//
statefulset.apps/todo-db configured
```

- rollout
```
$ kubectl rollout status statefulset/todo-db

//
partitioned roll out complete: 1 new pods have been updated...
```

- pod 의 실행 시각과 이미지 이름 확인 
```
$ kubectl get pods -l app=todo-db '-o=custom-columns=NAME:metadata.name,IMAGE:.spec.containers[0].image,START_TIME:.status.startTime'


//
NAME        IMAGE                  START_TIME
todo-db-0   postgres:11.6-alpine   2024-04-28T14:36:14Z
todo-db-1   postgres:11.8-alpine   2024-04-28T15:11:52Z
```

- Web application 을 읽기 전용 모드로 전환하여 데이터베이스 부 인스턴스만 사용하도록 한다
```
$ kubectl apply -f todo-list/web/update/todo-web-readonly.yaml

//
deployment.apps/todo-web configured
```

> DB 의 주 인스턴스를 업데이트. Pod 정의에서 partition 설정값만 제거함
- update
```
$ kubectl apply -f todo-list/db/update/todo-db-rollingUpdate.yaml

//
statefulset.apps/todo-db configured
```

- update check
```
$ kubectl rollout status statefulset/todo-db

//
waiting for statefulset rolling update to complete 1 pods at revision todo-db-d77d4cdc5...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
statefulset rolling update complete 2 pods at revision todo-db-d77d4cdc5...
```

- pod 정의 확인
```
$ kubectl get pods -l app=todo-db '-o=custom-columns=NAME:metadata.name,IMAGE:.spec.containers[0].image,START_TIME:.status.startTime'

//
NAME        IMAGE                  START_TIME
todo-db-0   postgres:11.8-alpine   2024-04-28T15:33:51Z
todo-db-1   postgres:11.8-alpine   2024-04-28T15:11:52Z
```

- Web application : readOnly 에서 원래 모드로 변경
```
$ kubectl apply -f todo-list/web/todo-web.yaml

//
deployment.apps/todo-web configured
```

Deployment, DemonSet, StatefulSet 모두 기본적으로 Rolling Update 전략을 사용

기존 파드를 점진적으로 변경된 정의의 파드로 대체

### 9.5 릴리스 전략 이해하기 

Blue-Green Deployment
- 구 버전과 신 버전 Application 을 동시에 배치하되, 한쪽만 활성화시킨다
- e.g 쿠버네티스 : 서비스에서 트래픽 전달 대상 파드를 결정하는 레이블 셀럭터를 수정
  - 클러스터 용량이 완전한 애플리케이션 두 벌을 동시에 실행할 수 있어야 함
  - Pod 가 준비된 상태에서 트래픽을 전달받기 때문에 실질적으로는 한순간에 업데이트 


Resource 정리 
```
$ kubectl delete all -l kiamol=ch09

//
service "todo-db" deleted
service "todo-proxy" deleted
service "todo-web" deleted
daemonset.apps "todo-proxy" deleted
deployment.apps "todo-web" deleted
statefulset.apps "todo-db" deleted
```

```
$ kubectl delete cm -l kiamol=ch09

//
configmap "todo-db-config" deleted
configmap "todo-db-env" deleted
configmap "todo-db-scripts" deleted
configmap "todo-proxy-configmap" deleted
configmap "todo-web-config" deleted
configmap "vweb-config" deleted
configmap "vweb-config-v4" deleted
configmap "vweb-config-v41" deleted
```

```
$ kubectl delete pvc -l kiamol=ch09

//
persistentvolumeclaim "data-todo-db-0" deleted
persistentvolumeclaim "data-todo-db-1" deleted
```