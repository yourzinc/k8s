# k8s

---

## 1장 : 시작하기 전에

```
$ kubectl get nodes
```
---

## 2장 : pod 와 deployment 로 container 실행하기

### 2.1 쿠버네티스는 어떻게 컨테이너를 실행하고 관리하는가
- 컨테이너 하나를 담은 파드를 실행한다
```
$ kubectl run hello-kiamol --image=kiamol/ch02-hello-kiamol

//
pod/hello-kiamol created
```

- 파드가 준비 상태가 될 때까지 기다린다
```
$ kubectl wait --for=condition=Ready pod hello-kiamol

//
pod/hello-kiamol condition met
```

- 클러스터에 있는 모든 파드의 목록을 출력한다
```
$ kubectl get pods

// 
NAME           READY   STATUS    RESTARTS   AGE
hello-kiamol   1/1     Running   0          2m17s
```

- 파드의 상세 정보를 확인한다
```
$ kubectl describe pod hello-kiamol

//
Name:             hello-kiamol
Namespace:        default
Priority:         0
Service Account:  default
Node:             docker-desktop/192.168.65.4
Start Time:       Mon, 08 Apr 2024 23:50:30 +0900
Labels:           run=hello-kiamol
Annotations:      <none>
Status:           Running
IP:               10.1.0.6
IPs:
  IP:  10.1.0.6
Containers:
  hello-kiamol:
    Container ID:   docker://4398243f242b649c6b3e021300ca5230c3b7033affc3c7cf6c3ffe5be112f754
    Image:          kiamol/ch02-hello-kiamol
    Image ID:       docker-pullable://kiamol/ch02-hello-kiamol@sha256:8a27476444b4c79b445f24eeb5709066a9da895b871ed9115e81eb5effeb5496
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 08 Apr 2024 23:50:40 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-hf55k (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-hf55k:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  3m9s   default-scheduler  Successfully assigned default/hello-kiamol to docker-desktop
  Normal  Pulling    3m8s   kubelet            Pulling image "kiamol/ch02-hello-kiamol"
  Normal  Pulled     2m59s  kubelet            Successfully pulled image "kiamol/ch02-hello-kiamol" in 8.572875629s
  Normal  Created    2m59s  kubelet            Created container hello-kiamol
  Normal  Started    2m59s  kubelet            Started container hello-kiamol
```

- 파드에 대한 기본적인 정보를 확인한다
```
$ kubectl get pod hello-kiamol

//
NAME           READY   STATUS    RESTARTS   AGE
hello-kiamol   1/1     Running   0          5m56s
```

- 네트워크 상세 정보 중 특정한 항목을 따로 지정해서 출력한다
```
$ kubectl get pod hello-kiamol --output custom-columns=NAME:metadata.name,NODE_IP:status.hostIP,POD_IP:status.podIP

//
NAME           NODE_IP        POD_IP
hello-kiamol   192.168.65.4   10.1.0.6
```

- JSONPath로 복잡한 출력을 구성한다 (pod 의 첫 번째 컨테이너의 컨테이너 식별자만 출력한다)
```
$ kubectl get pod hello-kiamol -o jsonpath='{.status.containerStatuses[0].containerID}'

//
docker://4398243f242b649c6b3e021300ca5230c3b7033affc3c7cf6c3ffe5be112f754
```

- 파드에 포함된 컨테이너 찾기
```
$ docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol

//
4398243f242b
```

- 해당 컨테이너 삭제하기
```
$ docker container rm -f $(docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol)

//
4398243f242b
```

- 파드 상태 확인
```
$ kubectl get pod hello-kiamol

//
NAME           READY   STATUS    RESTARTS   AGE
hello-kiamol   1/1     Running   1          16m
```

- 이전 컨테이너 다시 찾아보기
```
$ docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol

//
37a7817bb226
```

- 로컬 컴퓨터의 8080번 포트를 주시하다가 이 포트로 들어오는 트래픽을 파드의 80번 포트로 전달한다
```
$ kubectl port-forward pod/hello-kiamol 8080:80

//
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80

Handling connection for 8080
```
### 2.2 컨트롤러 객체와 함께 파드 실행하기

- 웹 애플리케이션을 실행하는 디폴로이먼트 'hello-kiamol-2' 생성
```
$ kubectl create deployment hello-kiamol-2 --image=kiamol/ch02-hello-kiamol

//
deployment.apps/hello-kiamol-2 created
```

- 파드의 목록을 출력
```
$ kubectl get pods

//
NAME                              READY   STATUS    RESTARTS   AGE
hello-kiamol                      1/1     Running   1          26m
hello-kiamol-2-7cb44d9bdd-xfgns   1/1     Running   0          28s
```

- deployment 가 부여한 pod 의 레이블 출력
```
$ kubectl get deploy hello-kiamol-2 -o jsonpath='{.spec.template.metadata.labels}'

//
{"app":"hello-kiamol-2"}
```

- 앞서 출력한 레이블을 가진 파드의 목록 출력
```
$ kubectl get pods -l app=hello-kiamol-2

//
NAME                              READY   STATUS    RESTARTS   AGE
hello-kiamol-2-7cb44d9bdd-xfgns   1/1     Running   0          5m6s
```

- 모든 파드 이름과 레이블 확인
```
$ kubectl get pods -o custom-columns=NAME:metadata.name,LABELS:metadata.labels

//
NAME                              LABELS
hello-kiamol                      map[run:hello-kiamol]
hello-kiamol-2-7cb44d9bdd-xfgns   map[app:hello-kiamol-2 pod-template-hash:7cb44d9bdd]
```

- deployment 가 생성한 pod 의 'app' 레이블 수정
```
$ kubectl label pods -l app=hello-kiamol-2 --overwrite app=hello-kiamol-x

//
pod/hello-kiamol-2-7cb44d9bdd-5pfp7 labeled
```

- pod 가 또 하나 생성되었다
```
$ kubectl get pods -o custom-columns=NAME:metadata.name,LABELS:metadata.labels

//
NAME                              LABELS
hello-kiamol                      map[run:hello-kiamol]
hello-kiamol-2-7cb44d9bdd-5pfp7   map[app:hello-kiamol-x pod-template-hash:7cb44d9bdd]
hello-kiamol-2-7cb44d9bdd-mvzf7   map[app:hello-kiamol-2 pod-template-hash:7cb44d9bdd]
hello-kiamol-2-7cb44d9bdd-xfgns   map[app:hello-kiamol-x pod-template-hash:7cb44d9bdd]
```

- 'app' 이라는 레이블이 부여된 모든 파드의 이름과 레이블 출력
```
$ kubectl get pods -l app -o custom-columns=NAME:metadata.name,LABELS:metadata.labels

//
NAME                              LABELS
hello-kiamol-2-7cb44d9bdd-5pfp7   map[app:hello-kiamol-x pod-template-hash:7cb44d9bdd]
hello-kiamol-2-7cb44d9bdd-mvzf7   map[app:hello-kiamol-2 pod-template-hash:7cb44d9bdd]
hello-kiamol-2-7cb44d9bdd-xfgns   map[app:hello-kiamol-x pod-template-hash:7cb44d9bdd]
```

- deployment 의 관리를 벗어난 파드의 'app' 레이블을 원래대로 수정
```
$ kubectl label pods -l app=hello-kiamol-x --overwrite app=hello-kiamol-2

//
pod/hello-kiamol-2-7cb44d9bdd-5pfp7 labeled
pod/hello-kiamol-2-7cb44d9bdd-xfgns labeled
```

- pod 목록을 다시 한 번 확인
```
$ kubectl get pods -l app -o custom-columns=NAME:metadata.name,LABELS:metadata.labels

//
NAME                              LABELS
hello-kiamol-2-7cb44d9bdd-xfgns   map[app:hello-kiamol-2 pod-template-hash:7cb44d9bdd]
```

- local 에서 deployment 로 Port-forwarding 설정
```
$ kubectl port-forward deploy/hello-kiamol-2 8080:80

//
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
Handling connection for 8080
```

### 2.3 Application Manifesto 에 배포 정의하기

- manifesto 파일로 application 배포
```
$ kubectl apply -f pod.yaml

//
pod/hello-kiamol-3 created
```

- 실행 중인 파드 목록 확인
```
$ kubectl get pods

//
NAME                              READY   STATUS    RESTARTS        AGE
hello-kiamol                      1/1     Running   2 (2m33s ago)   20h
hello-kiamol-2-7cb44d9bdd-xfgns   1/1     Running   1 (2m33s ago)   20h
hello-kiamol-3                    1/1     Running   0               6s
```

- 원격 URL 에서 제공되는 manifesto 파일로 애플리케이션을 배포하라
```
$ kubectl apply -f http://raw.githubusercontent.com/sixeyed/kiamol/master/ch02/pod.yaml

//
pod/hello-kiamol-3 unchanged
```

- deployment manifesto 로 application 실행
```
$ kubectl apply -f deployment.yaml

//
deployment.apps/hello-kiamol-4 created
```

- 새로운 deployment 가 만든 pod 찾기 
```
$ kubectl get pods -l app=hello-kiamol-4

//
NAME                              READY   STATUS    RESTARTS   AGE
hello-kiamol-4-66874cd94b-cv2nl   1/1     Running   0          71s
```

### 2.4 Pod 에서 실행 중인 application 에 접근하기

- 처음 실행한 파드의 내부 IP 주소 확인
```
$ kubectl get pod hello-kiamol -o custom-columns=NAME:metadata.name,POD_IP:status.podIP

//
NAME           POD_IP
hello-kiamol   10.1.0.12
```

- Pod 내부와 연결할 대화형 shell 실행
```
$ kubectl exec -it hello-kiamol sh 

```

- Pod 안에서 IP 주소 확인
```
/ # hostname -i

//
10.1.0.12
```

- Web application 의 동작 확인
```
$ wget -O - http://localhost | head -n 4

//
Connecting to localhost (127.0.0.1:80)
writing to stdout
<html>
  <body>
    <h1>
      Hello from Chapter 2!
    </h1>
    <h2>
      This is
      <a
        href="https://www.manning.com/books/learn-kubernetes-in-a-month-of-lunches"
        >Learn Kubernetes in a Month of Lunches</a
      >.
    </h2>
    <h3>By <a href="https://blog.sixeyed.com">Elton Stoneman</a>.</h3>
  </body>
</html>
-                    100% |*********************************************************************************************************************************************************************************************|   353  0:00:00 ETA
written to stdout

```

> 쿠버네티스는 컨테이너 런타임을 경유해서 애플리케이션 로그를 불러온다. 애플리케이션 로그를 확인하고, 컨테이너에 직접 점속하여 (컨테이너 런타임이 허용한다면) 실제 컨테이너 로그와 애플리케이션 로그가 일치하는지 확인하라.

- 쿠버네티스를 통해 컨테이너의 최근 로그를 출력
```
$ kubectl logs --tail=2 hello-kiamol

//
127.0.0.1 - - [09/Apr/2024:11:48:43 +0000] "GET / HTTP/1.1" 200 353 "-" "Wget" "-"
127.0.0.1 - - [09/Apr/2024:11:49:01 +0000] "GET / HTTP/1.1" 200 353 "-" "Wget" "-"
```

```
$ kubectl logs hello-kiamol

//
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2024/04/09 11:26:27 [notice] 1#1: using the "epoll" event method
2024/04/09 11:26:27 [notice] 1#1: nginx/1.21.6
2024/04/09 11:26:27 [notice] 1#1: built by gcc 10.3.1 20211027 (Alpine 10.3.1_git20211027) 
2024/04/09 11:26:27 [notice] 1#1: OS: Linux 5.15.49-linuxkit
2024/04/09 11:26:27 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2024/04/09 11:26:27 [notice] 1#1: start worker processes
2024/04/09 11:26:27 [notice] 1#1: start worker process 32
2024/04/09 11:26:27 [notice] 1#1: start worker process 33
2024/04/09 11:26:27 [notice] 1#1: start worker process 34
2024/04/09 11:26:27 [notice] 1#1: start worker process 35
127.0.0.1 - - [09/Apr/2024:11:48:43 +0000] "GET / HTTP/1.1" 200 353 "-" "Wget" "-"
127.0.0.1 - - [09/Apr/2024:11:49:01 +0000] "GET / HTTP/1.1" 200 353 "-" "Wget" "-"

```

- 도커를 통해 컨테이너에 접속해서 실제 로그와 동일한지 확인
```
$ docker container logs --tail=2 $(docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol)

//
127.0.0.1 - - [09/Apr/2024:11:48:43 +0000] "GET / HTTP/1.1" 200 353 "-" "Wget" "-"
127.0.0.1 - - [09/Apr/2024:11:49:01 +0000] "GET / HTTP/1.1" 200 353 "-" "Wget" "-"
```

> 파드 이름을 직접 알지 못해도 deployment 가 관리하는 pod 에서 명령을 실행할 수 있다. label selector 와 일치하는 모든 pod 의 로그를 열람

- YAML 정의에 따라 생성한 deployment 가 만든 Pod 안에 들어 있는 컨테이너에서 웹 애플리케이션 호출
```
$ kubectl exec deploy/hello-kiamol-4 -- sh -c 'wget -O - http://localhost > /dev/null'

//
Connecting to localhost (127.0.0.1:80)
writing to stdout
-                    100% |********************************|   353  0:00:00 ETA
written to stdout
```

- 해당 pod 의 log 열람
```
$ kubectl logs --tail=1 -l app=hello-kiamol-4

//
127.0.0.1 - - [09/Apr/2024:11:58:50 +0000] "GET / HTTP/1.1" 200 353 "-" "Wget" "-"
```

> 로컬 컴퓨터에 임시 directory 를 만들고 pod 속 컨테이너에서 이 directory 로 파일을 복사하라

- local 에 임시 directory 생성
```
$ mkdir -p /tmp/kiamol/ch02
```

- Pod 속에서 웹 페이지를 local 로 복사
```
$ kubectl cp hello-kiamol:/usr/share/nginx/html/index.html /tmp/kiamol/ch02/index.html
```

- local 에서 파일 내용 확인
```
$ cat /tmp/kiamol/ch02/index.html

//
<html>
  <body>
    <h1>
      Hello from Chapter 2!
    </h1>
    <h2>
      This is
      <a
        href="https://www.manning.com/books/learn-kubernetes-in-a-month-of-lunches"
        >Learn Kubernetes in a Month of Lunches</a
      >.
    </h2>
    <h3>By <a href="https://blog.sixeyed.com">Elton Stoneman</a>.</h3>
  </body>
</html>
```

### 2.5 쿠버네티스의 리소스 관리 이해하기

> kubectl 의 delete 명령을 사용하여 모든 파드를 삭제한 후 정말로 모든 파드가 삭제되었는지 확인하라

- 실행 중인 모든 파드의 목록 출력
```
$ kubectl get pods

// 
NAME                              READY   STATUS    RESTARTS      AGE
hello-kiamol                      1/1     Running   2 (81m ago)   21h
hello-kiamol-2-7cb44d9bdd-xfgns   1/1     Running   1 (81m ago)   21h
hello-kiamol-3                    1/1     Running   0             78m
hello-kiamol-4-66874cd94b-cv2nl   1/1     Running   0             66m
```

- 모든 파드 삭제
```
$ kubectl delete pods --all

//
pod "hello-kiamol" deleted
pod "hello-kiamol-2-7cb44d9bdd-xfgns" deleted
pod "hello-kiamol-3" deleted
pod "hello-kiamol-4-66874cd94b-cv2nl" deleted
```

- 모든 파드가 삭제되었는지 확인
```
$ kubectl get pods

//
NAME                              READY   STATUS    RESTARTS   AGE
hello-kiamol-2-7cb44d9bdd-t52r5   1/1     Running   0          21s
hello-kiamol-4-66874cd94b-p4dtf   1/1     Running   0          21s
```

> 실행 중인 deployment 의 목록을 확인하고 deployment 를 삭제하라. deployment 를 삭제한 후 남아 있던 pod 도 함께 삭제되었는지 확인

- deployment 목록 확인
```
$ kubectl get deploy

//
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hello-kiamol-2   1/1     1            1           21h
hello-kiamol-4   1/1     1            1           70m 
```

- deployment 모두 삭제 
```
$ kubectl delete deploy --all

//
deployment.apps "hello-kiamol-2" deleted
deployment.apps "hello-kiamol-4" deleted
```

- pod 목록 확인
```
$ kubectl get pods

// 
No resources found in default namespace.
```

- 모든 리소스 목록 확인
```
$ kubectl get all 

//
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   23h
```

### 2.6 연습 문제

```
kubectl apply -f lab/deployment.yaml
```

```
kubectl port-forward deploy/whoami 8080:80
```

```
curl http://localhost:8080
```

```
kubectl get pods -o custom-columns=NAME:metadata.name
```

```
kubectl exec deploy/whoami -- sh -c 'hostname'
```