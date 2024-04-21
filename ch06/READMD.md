## 6장 컨트롤러 리소스를 이용한 애플리케이션의 스케일링

Replica
- 동일한 애플리케이션이 돌아가는 pod


### 6.1 쿠버네티스는 어떻게 애플리케이션을 스케일링하는가

Deployment -> ReplicaSet
ReplicaSet -> Pod 
Pod -> Container

Deployment 는 ReplicaSet 을 관리한다.
ReplicaSet 은 Pod 를 관리한다.
Pod 는 Container 를 관리한다. 

> ReplicaSet 을 배치. LoadBalancer 서비스를 함께 배치. pod 에 트래픽이 전달되도록 두 리소스의 label Selector 를 동일하게 한다. 

- ReplicaSet 과 Service 배치
```
$ kubectl apply -f whoami/
```

- 배치된 리소스 확인
```
$ kubectl get replicaset whoami-web

//
NAME         DESIRED   CURRENT   READY   AGE
whoami-web   1         1         1       16s
```

- 서비스로 HTTP GET 요청을 전달
```
$ curl $(kubectl get svc whoami-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8088') -UseBasicParsing

//
"I'm whoami-web-wtgv7 running on Linux 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022"
```

- 파드를 모두 삭제
```
$ kubectl delete pods -l app=whoami-web

//
pod "whoami-web-wtgv7" deleted
```

- HTTP GET 요청을 다시 한 번 전달
```
$ curl $(kubectl get svc whoami-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080')

// 
"I'm whoami-web-vc97v running on Linux 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022"
```

- ReplicaSet 의 정보 확인
```
$ kubectl describe rs whoami-web

//
Name:         whoami-web
Namespace:    default
Selector:     app=whoami-web
Labels:       <none>
Annotations:  <none>
Replicas:     1 current / 1 desired
Pods Status:  1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=whoami-web
  Containers:
   web:
    Image:        kiamol/ch02-whoami
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  6m11s  replicaset-controller  Created pod: whoami-web-wtgv7
  Normal  SuccessfulCreate  73s    replicaset-controller  Created pod: whoami-web-vc97v

```

ReplicaSet 은 항상 제어 루프를 돌며 관리 중인 리소스 수와 필요한 리소스 수를 확인하기 때문에 즉각 삭제된 파드를 대체할 수 있다.

Application scaling 도 같은 원리로 진행된다.

> Replica 수가 3개로 지정된 ReplicaSet 의 정의를 반영하여 Application 을 스케일링하라.

- Replica 수가 변경된 정의를 배치
```
$ kubectl apply -f whoami/update/whoami-replicas-3.yaml

//
replicaset.apps/whoami-web configured
```
- pod 목록 확인
```
$ kubectl get pods -l app=whoami-web

//
NAME               READY   STATUS    RESTARTS   AGE
whoami-web-n7hhc   1/1     Running   0          57s
whoami-web-vc97v   1/1     Running   0          6m49s
whoami-web-zhdkl   1/1     Running   0          57s
```

- 모든 pod 삭제
```
$ kubectl delete pods -l app=whoami-web

//
pod "whoami-web-n7hhc" deleted
pod "whoami-web-vc97v" deleted
pod "whoami-web-zhdkl" deleted
```

- pod 목록 다시 확인
```
$ kubectl get pods -l app=whoami-web

//
NAME               READY   STATUS    RESTARTS   AGE
whoami-web-9ll44   1/1     Running   0          32s
whoami-web-pgvjq   1/1     Running   0          32s
whoami-web-qdjqq   1/1     Running   0          32s
```

- HTTP 요청을 몇 번 더 반복
```
$ curl $(kubectl get svc whoami-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080')

//
"I'm whoami-web-pgvjq running on Linux 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022"
"I'm whoami-web-qdjqq running on Linux 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022"
"I'm whoami-web-9ll44 running on Linux 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022"
```

Q. Application Scaling 이 빠른 이유
A. 단일 노드 클러스터의 경우 Application 의 이미지가 이미 존재하므로, 이미지를 내려받는 시간이 추가로 소요되지 않음
다중 노드 클러스터의 경우, 이미지 내려 받는 시간이 소요되므로 -> 이미지 최적화가 필요함

필요한 만큼 파드를 실행하되, 이들을 모두 같은 서비스 아래 연결한다.

서비스에 요청이 들어오면 서비스 아래의 파드에 고르게 요청이 분배된다. 

> Pod 를 수동으로 배치하고 이 파드에서 whoami-web 서비스의 ClusterIP 를 통해 해당 서비스를 호출하라

- Sleep pod 실행
```
$ kubectl apply -f sleep.yaml

//
deployment.apps/sleep created
```

- whoami-web 서비스의 상세 정보 확인
```
$ kubectl get svc whoami-web

//
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
whoami-web   LoadBalancer   10.98.193.232   localhost     8080:32226/TCP   21m
```

- sleep pod 에서 service 이름으로 DNS 조회
```
$ kubectl exec deploy/sleep -- sh -c 'nslookup whoami-web | grep "^[^*]"'

//
Server:         10.96.0.10
Address:        10.96.0.10:53
Name:   whoami-web.default.svc.cluster.local
Address: 10.98.193.232
```

- whoami-web 서비스로 HTTP 요청
```
$ kubectl exec deploy/sleep -- sh -c 'for i in 1 2 3; do curl -w \\n -s http://whoami-web:8080; done;'

//
"I'm whoami-web-qdjqq running on Linux 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022"
"I'm whoami-web-9ll44 running on Linux 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022"
"I'm whoami-web-qdjqq running on Linux 5.15.49-linuxkit #1 SMP PREEMPT Tue Sep 13 07:51:32 UTC 2022"
```

클러스터IP 서비스를 통해 접근해도, 외부로부터의 트래픽과 마찬가지로 로드밸런싱이 적용된다.

클러스터 내 트래픽도 서비스를 거치며 로드밸런싱이 적용된다

클러스터 내 어떤 노드에서 요청이 오더라도 동일한 네트워크 계층에서 여러 파드에 고르게 트래픽을 분배할 수 있다.


### 6.2 디플로이먼트와 레플리카셋을 이용한 부하 스케일링

> 원주율 웹 애플리케이션을 배치하는 deployment 와 service 를 생성하라. 그리고 application 을 업데이트하고 replicaset 이 어떻게 반응하는지 살펴보아라.

- 원주율 웹 애플리케이션을 배치
```
$ kubectl apply -f pi/web/

//
service/pi-web unchanged
deployment.apps/pi-web created
```

- ReplicaSet 의 상태 확인
```
$ kubectl get rs -l app=pi-web

//
NAME               DESIRED   CURRENT   READY   AGE
pi-web-7c778878b   2         2         2       40s
```

- Replica 를 늘려 스케일링을 적용
```
$ kubectl apply -f pi/web/update/web-replicas-3.yaml

//
deployment.apps/pi-web configured
```

- ReplicaSet 의 상태 확인
```
$ kubectl get rs -l app=pi-web

//
NAME               DESIRED   CURRENT   READY   AGE
pi-web-7c778878b   3         3         3       6m51s
```

- log 설정이 추가된 새로운 파드의 정의를 적용
```
$ kubectl apply -f pi/web/update/web-logging-level.yaml

//
deployment.apps/pi-web configured
```

- ReplicaSet 의 상태를 다시 확인
```
$ kubectl get rs -l app=pi-web

//
NAME                DESIRED   CURRENT   READY   AGE
pi-web-75987c7485   3         3         3       31s
pi-web-7c778878b    0         0         0       8m31s
```

pod 의 정의가 변경된 경우, 기존 ReplicaSet 에 속한 pod 는 필요하지 않게 된다. Deployment 는 새로운 ReplicaSet 을 생성하고, 새 파드가 준비됨에 따라 기존 파드를 삭제한다.

> pi 웹 애플리케이션을 kubectl scale 명령을 사용하여 스케일링하라. 그리고 ReplicaSet 동작이 전체 애플리케이션 재배치와 비교해서 어떻게 달라지는지 살펴보아라.

- pi Application 을 신속하게 스케일링
```
$ kubectl scale --replicas=4 deploy/pi-web 

//
deployment.apps/pi-web scaled
```

- ReplicaSet 변경 확인
```
$ kubectl get rs -l app=pi-web

//
NAME                DESIRED   CURRENT   READY   AGE
pi-web-75987c7485   4         4         4       6m8s
pi-web-7c778878b    0         0         0       14m
```

- log 수준을 원래대로 되돌리기
```
$ kubectl apply -f pi/web/update/web-replicas-3.yaml

//
deployment.apps/pi-web configured
```

- ReplicaSet 변경 확인 (수동 스케일링 원복됨)
```
$ kubectl get rs -l app=pi-web     

//
NAME                DESIRED   CURRENT   READY   AGE
pi-web-75987c7485   0         0         0       7m3s
pi-web-7c778878b    3         3         3       15m
```

- pod 상태 확인
```
$ kubectl get pods -l app=pi-web

//
NAME                     READY   STATUS    RESTARTS   AGE
pi-web-7c778878b-4zfzh   1/1     Running   0          93s
pi-web-7c778878b-bhf4w   1/1     Running   0          97s
pi-web-7c778878b-q8pps   1/1     Running   0          90s
```

Deployment 가 생성한 파드의 무작위 문자열은 Deployment 에 포함된 템플릿 속 pod 정의의 해시값
파드 템플릿의 해시값은 레이블에 포함되어 있다

> pi web application 을 구성하는 pod 와 ReplicaSet 의 레이블에서 template Hash 값을 확인하라

- ReplicaSet 과 Label 확인
```
$ kubectl get rs -l app=pi-web --show-labels

//
NAME                DESIRED   CURRENT   READY   AGE   LABELS
pi-web-75987c7485   0         0         0       11m   app=pi-web,pod-template-hash=75987c7485
pi-web-7c778878b    3         3         3       19m   app=pi-web,pod-template-hash=7c778878b
```

- Pod 와 Label 확인
```
$ kubectl get po -l app=pi-web --show-labels

//
NAME                     READY   STATUS    RESTARTS   AGE     LABELS
pi-web-7c778878b-4zfzh   1/1     Running   0          5m11s   app=pi-web,pod-template-hash=7c778878b
pi-web-7c778878b-bhf4w   1/1     Running   0          5m15s   app=pi-web,pod-template-hash=7c778878b
pi-web-7c778878b-q8pps   1/1     Running   0          5m8s    app=pi-web,pod-template-hash=7c778878b
```

> Replica 수가 2개인 Proxy deployment 를 생성하라. 이와 함께 proxy 를 원주율 애플리케이션과 통합해 줄 service 와 configMap 도 함께 배치하라

- proxy deployment 생성
```
$ kubectl apply -f pi/proxy/

//
configmap/pi-proxy-configmap created
service/pi-proxy created
deployment.apps/pi-proxy created
```

- proxy 가 적용된 application 의 URL 확인
```
$ kubectl get svc whoami-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080/?dp=10000'


//
http://localhost:8080/?dp=10000
```

모든 Proxy pod 가 cache volume 을 공유하면 지연 시간에 대한 문제를 해결할 수 있다

> Proxy pod 를 공디렉터리 볼륨 대신 호스트경로 볼륨을 사용하도록 수정된 정의로 업데이트하라. 같은 노드에서 실행된 모든 노드는 같은 볼륨을 사용하므로 볼륨을 공유하는 효과를 볼 수 있다.

- Proxy pod 업데이트
```
$ kubectl apply -f pi/proxy/update/nginx-hostPath.yaml

//
deployment.apps/pi-proxy configured
```

- pod 수 확인 - 새 정의의 Replica 수는 3개 
```
$ kubectl get po -l app=pi-proxy

//
NAME                        READY   STATUS    RESTARTS   AGE
pi-proxy-7d97789c6f-55nwf   1/1     Running   0          39s
pi-proxy-7d97789c6f-m24wf   1/1     Running   0          38s
pi-proxy-7d97789c6f-xbdkd   1/1     Running   0          37s
```

- pi application 에서 새로고침

- Proxy log 확인
```
$ kubectl logs -l app=pi-proxy --tail 1

//
192.168.65.4 - - [21/Apr/2024:02:50:56 +0000] "GET /?dp=10000.1 HTTP/1.1" 200 1019 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123.0.0.0 Safari/537.36"
192.168.65.4 - - [21/Apr/2024:02:50:52 +0000] "GET /?dp=1000000 HTTP/1.1" 499 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123.0.0.0 Safari/537.36"
```

Nginx 는 Cache Directory 를 다른 nginx 인스턴스와 공유하더라도 별다른 문제를 일으키지 않는다.

DemonSet 과 StatefulSet 은 모두 pod 를 관리하는 controller 이다.
Deployment 에 비해 사용빈도는 낮다.

### 6.3 데몬셋을 이용한 스케일링으로 고가용성 확보하기

DaemonSet : 클러스터 내 모든 node 또는 selector 와 일치하는 일부 노드에서 단일 replica 또는 pod 로 동작하는 리소스 

- 각 노드에서 정보를 수집하여 중앙의 수집 모듈에 전달
- 각 노드마다 파드가 하나씩 동작하면서 해당 노드의 데이터를 수집
- 파드의 로그와 노드의 활성 지표를 수집하는데 사용

> 기존 컨트롤러 리소스의 유형을 바꿀 수는 없지만, 애플리케이션을 망가뜨리지 않으면서 deployment 를 데몬셋으로 교체할 수는 있다.

- DaemonSet 배치
```
$ kubectl apply -f pi/proxy/daemonset/nginx-ds.yaml

//
daemonset.apps/pi-proxy created
```

- Proxy service 에 등록된 Endpoint 확인
```
$ kubectl get endpoints pi-proxy

// 
NAME       ENDPOINTS                                            AGE
pi-proxy   10.1.0.90:80,10.1.0.92:80,10.1.0.93:80 + 1 more...   4h50m
```

- Deployment 삭제
```
$ kubectl delete deploy pi-proxy

//
deployment.apps "pi-proxy" deleted
```

- DaemonSet 의 상세 정보 확인
```
$ kubectl get daemonset pi-proxy

//
NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
pi-proxy   1         1         1       1            1           <none>          3m55s
```

- pod 의 상태를 확인
```
$ kubectl get po -l app=pi-proxy

//
NAME             READY   STATUS    RESTARTS   AGE
pi-proxy-6jljk   1/1     Running   0          6m35s
```

Deployment 를 삭제하면 관리하던 pod 가 삭제된다
하지만, Daemonset 이 생성한 pod 가 요청을 처리할 수 있다

> 수동으로 proxy pod 를 삭제하라. DaemonSet 이 대체 파드를 실행한다

- DaemonSet 상태 확인
```
$ kubectl get ds pi-proxy

//
NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
pi-proxy   1         1         1       1            1           <none>          11m
```

- proxy pod 를 수동으로 삭제
```
$ kubectl delete po -l app=pi-proxy

//
pod "pi-proxy-6jljk" deleted
```

- pod 의 목록을 확인
```
$ kubectl get po -l app=pi-proxy

//
NAME             READY   STATUS    RESTARTS   AGE
pi-proxy-mpxkw   1/1     Running   0          6s
```

DaemonSet 이 Replica 를 관리하는 규칙은 Deployment 와는 다르다.

하지만 DaemonSet 역시 파드를 관리하는 컨트롤러이므로 파드가 유실되면 대체 pod 를 생성한다.

> nodeSelector 가 추가된 정의대로 DaemonSet 을 업데이트하라. 레이블이 일치하는 노드가 없으므로 기존 파드가 제거된다.
그 다음 노드에 레이블을 부여하여 대상 노드로 만들면 새 파드가 생성된다.

- DaemonSet 업데이트
```
$ kubectl apply -f pi/proxy/daemonset/nginx-ds-nodeSelector.yaml

//
daemonset.apps/pi-proxy configured
```

- DaemonSet 의 상태를 확인
```
$ kubectl get ds pi-proxy

//
NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
pi-proxy   0         0         0       0            0           kiamol=ch06     16m
```

- Pod 의 상태를 확인
```
$ kubectl get po -l app=pi-proxy

//
No resources found in default namespace.
```

- Selecttor 와 일치하는 레이블을 노드에 부여
```
$ kubectl label node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') kiamol=ch06 --overwrite

//
node/docker-desktop labeled
```

- Pod 의 상태를 확인
```
$ kubectl get ds pi-proxy

//
NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
pi-proxy   1         1         1       1            1           kiamol=ch06     18m
```

노드에 레이블을 부여하니 셀렉터와 일치하는 노드가 생겼다. 이에 따라 데몬셋의 지정된 레플리카 수도 1이 된다.

> kubectl delete 명령 옵션 (cascade) : controller 의 관리 대상 리소스는 유지하고 controller 의 리소스만 삭제할 수 있다. 남겨진 관리 대상 리소스는 selector 가 일치하는 새로운 controller 리소스가 생성되면, 다시 그 리소스의 관리 대상으로 들어간다.

- 관리 대상 파드는 남기고 DaemonSet 을 삭제
```
$ kubectl delete ds pi-proxy --cascade=false

//
warning: --cascade=false is deprecated (boolean value) and can be replaced with --cascade=orphan.
daemonset.apps "pi-proxy" deleted
```

- Pod 상태 확인
```
$ kubectl get po -l app=pi-proxy

//
NAME             READY   STATUS    RESTARTS   AGE
pi-proxy-b2dhv   1/1     Running   0          5m43s
```

- DaemonSet 을 다시 생성
```
$ kubectl apply -f pi/proxy/daemonset/nginx-ds-nodeSelector.yaml

//
daemonset.apps/pi-proxy created
```

- DaemonSet 과 pod 의 상태 확인
```
$ kubectl get ds pi-proxy

//
NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
pi-proxy   1         1         1       1            1           kiamol=ch06     26s
```

```
$ kubectl get po -l app=pi-proxy

//
NAME             READY   STATUS    RESTARTS   AGE
pi-proxy-b2dhv   1/1     Running   0          7m19s
```

- cascade 옵션 없이 DaemonSet 을 삭제
```
$ kubectl delete ds pi-proxy

//
daemonset.apps "pi-proxy" deleted
```

- pod 의 상태를 확인
```
$ kubectl get po -l app=pi-proxy

//
No resources found in default namespace.
```

DeamonSet 은 `각각의 인스터스가 독립적인 데이터 저장소를 가져도 괜찮은 애플리케이션` 에 적합하다.
StatefulSet 은 `고가용성이 필요하지만, 여러 인스턴스가 데이터 저장소를 공유할 때`에 적합하다. 

### 6.4 쿠버네티스의 객체 간 오너십

Controller : label selector 를 이용하여 관리 대상 리소스를 결정
Resource : metadata.field 에 자신을 관리하는 리소스 정보를 기록

Garbage collector : 관리 주체가 사라진 객체를 찾아 제거

e.g. Pod -> ReplicaSet -> Deployment 

> 모든 파드와 레플리카셋의 메타데이터에서 관리 주체 리소스 정보를 확인하라

- 각 pod 의 관리 주체 리소스를 확인
```
$ kubectl get po -o custom-columns=NAME:'{.metadata.name}',OWNER:'{.metadata.ownerReferences[0].name}',OWNER_KIND:'{.metadata.ownerReferences[0].kind}'

//
NAME                     OWNER              OWNER_KIND
pi-web-7c778878b-65fqm   pi-web-7c778878b   ReplicaSet
pi-web-7c778878b-d7hws   pi-web-7c778878b   ReplicaSet
pi-web-7c778878b-r6pzg   pi-web-7c778878b   ReplicaSet
whoami-web-vwlqs         whoami-web         ReplicaSet
```

- 각 ReplicaSet 의 관리 주체 리소스를 확인
```
$ kubectl get rs -o custom-columns=NAME:'{.metadata.name}',OWNER:'{.metadata.ownerReferences[0
].name}',OWNER_KIND:'{.metadata.ownerReferences[0].kind}'

//
NAME                OWNER    OWNER_KIND
pi-web-75987c7485   pi-web   Deployment
pi-web-7c778878b    pi-web   Deployment
whoami-web          <none>   <none>
```

> 이 장에서 다룬 객체 중 최상위 객체에는 kiamol 레이블이 부여되었다. 리소스를 삭제하면, 해당 리소스의 관리 대상 리소스도 함께 삭제 된다

- 모든 controller 리소스 및 service 를 삭제
```
$ kubectl delete all -l kiamol=ch06

//
service "pi-proxy" deleted
service "pi-web" deleted
service "whoami-web" deleted
deployment.apps "pi-web" deleted
```

### 6.5 연습 문제

```
$ kubectl apply -f lab/numbers/

//
service/numbers-api created
pod/numbers-api created
service/numbers-web created
replicationcontroller/numbers-web created
```

```
$ curl http://localhost:8086/
```

```
$  kubectl label node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') rng=hw --overwrite

//
node/docker-desktop labeled
```

```
$ kubectl get all -l kiamol=ch06-lab

//
NAME              READY   STATUS    RESTARTS   AGE
pod/numbers-api   1/1     Running   0          17m

NAME                                DESIRED   CURRENT   READY   AGE
replicationcontroller/numbers-web   2         2         2       17m

NAME                  TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/numbers-api   ClusterIP      10.97.88.72   <none>        80/TCP           17m
service/numbers-web   LoadBalancer   10.100.68.0   localhost     8086:30470/TCP   17m

NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/numbers-api   1         1         1       1            1           rng=hw          3m45s

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/numbers-web   2/2     2            2           3m45s
```

```
$ kubectl delete all -l kiamol=ch06-lab

//
pod "numbers-api" deleted
replicationcontroller "numbers-web" deleted
service "numbers-api" deleted
service "numbers-web" deleted
daemonset.apps "numbers-api" deleted
deployment.apps "numbers-web" deleted
```