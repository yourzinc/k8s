## 3장 : 네트워크를 통해 서비스에 파드 연결하기 

Service : pod 에서 들고나는 통신 트래픽의 라우팅을 맡는 유연한 리소스

### 3.1 쿠버네티스 내부의 네트워크 트래픽 라우팅

> 파드는 IP 주수로 서로 통신할 수 있다.

- 각각 pod 하나를 실행하는 두 개의 deployment 를 생성
```
$ kubectl apply -f sleep/sleep1.yaml -f sleep/sleep2.yaml

//
deployment.apps/sleep-1 created
deployment.apps/sleep-2 created
```

- 파드가 완전히 시작될 때까지 기다린다
```
$ kubectl wait --for=condition=Ready pod -l app=sleep-2

//
pod/sleep-2-6695c7cd68-mg6xp condition met
```

- 두 번째 pod 의 IP 주소를 확인한다
```
$ kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'

//
10.1.0.24
```

- 같은 주소를 사용하여 첫 번째 pod 에서 두 번째 pod 로 ping 을 보낸다
```
$ kubectl exec deploy/sleep-1 -- ping -c 2 $(kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}')

//
PING 10.1.0.24 (10.1.0.24): 56 data bytes
64 bytes from 10.1.0.24: seq=0 ttl=64 time=0.156 ms
64 bytes from 10.1.0.24: seq=1 ttl=64 time=0.217 ms

--- 10.1.0.24 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.156/0.186/0.217 ms

```

_파드가 대체될 때마다 IP 주소가 바뀌기 때문에, 일반적으로 위 방법은 사용하지 않는다._

> pod 는 controller 객체인 deployment 로 관리된다. 두 번째 파드를 수동으로 삭제하면, 이를 관장하는 deployment 가 다른 IP 를 가진 새로운 pod 를 생성한다.

- pod 의 현재 IP 확인
```
$ kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'

//
10.1.0.24
```

- deployment 가 새 pod 를 만들도록 현재 pod 를 삭제한다
```
$ kubectl delete pods -l app=sleep-2

//
pod "sleep-2-6695c7cd68-mg6xp" deleted
```

- 새로 대체된 pod 의 IP 를 확인한다
```
$ kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'

//
10.1.0.27
```

쿠버네티스 클러스터에는 전용 DNS 서버가 있다. 이 서버는 서비스 이름과 IP 주소를 대응시켜준다. 서비스를 경유해서 파드는 서로 고정된 domain name 으로 통신할 수 있다.

> YAML 파일과 kubectl apply 명령을 사용하여 정의된 서비스를 배포하라. pod 로 네트워크 트래픽이 잘 연결되는지 확인하라.

- service 배포
```
$ kubectl apply -f ch03/sleep2-service.yaml

//
service/sleep-2 created
```

- service 상세 정보 출력
```
$ kubectl get svc sleep-2

//
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
sleep-2   ClusterIP   10.108.192.173   <none>        80/TCP    47s
```

- pod 와 통신이 잘 되는지 확인 
```
$ kubectl exec deploy/sleep-1 -- ping -c 1 sleep-2

//
PING sleep-2 (10.108.192.173): 56 data bytes

--- sleep-2 ping statistics ---
1 packets transmitted, 0 packets received, 100% packet loss
command terminated with exit code 1
```

_쿠버네티스 서비스에서는 ping (ICMP) 를 지원하지 않는다. 서비스 리소스는 표준 TCP/UDP 프로토콜을 지원한다._

서비스를 배포하면 쿠버네티스 내부 DNS 서버에 고정 IP 주소에 대한 도메인 네임이 등록된다.

**Serivce discovery** 

서비스 리소스를 배포하면, 서비스 이름을 도메인 네임으로 사용하여 다른 컴포넌트와 통신할 수 있다.

### 3.2 파드와 파드 간 통신

**ClusterIP** 

클러스터 내에서만 유효한 IP로, 파드와 파드 간 통신에 사용
내부에서는 접근이 가능하되 외부의 접근을 차단해야 하는 분산 시스템의 컴포넌트에 적합 

>  두 개의 deployment 를 실행하라. 하나는 웹 애플리케이션, 다른 하는 API 역할을 담당한다. 이 application 에는 아직 service 가 없다. web application 이 API 에 접근하지 못해 application 이 제대로 동작하지 않은 상태.

- Web site, API 를 담당하는 두 개의 deployment 를 실행
```
$ kubectl apply -f numbers/api.yaml -f numbers/web.yaml

// 
deployment.apps/numbers-api created
deployment.apps/numbers-web created
```

- pod 의 준비가 끝날 때가지 기다린다
```
$ kubectl wait --for=condition=Ready pod -l app=numbers-web

//
pod/numbers-web-c45b49bc8-rmhq6 condition met
```

- web application 에 port forwarding 을 적용한다
```
$ kubectl port-forward deploy/numbers-web 8080:80

//
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
```

> API 에 접근하기 위해 도메인 조회가 가능하도록 서비스를 배포하라. 트래픽이 실제로 API 파드에 전달되는지 확인하라.

- Service 배포
```
$ kubectl apply -f numbers/api-service.yaml

//
service/numbers-api created
```

- Service 의 상세 정보를 출력
```
$ kubectl get svc numbers-api

//
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
numbers-api   ClusterIP   10.109.219.78   <none>        80/TCP    29s
```

- Web application 에 접근할 수 있도록 port forwarding 적용
```
$ kubectl port-forward deploy/numbers-web 8080:80
```

_간단한 애플리케이션도 현재 상태처럼 동작하기 위해 2 개의 deployment, 1 개의 service 를 정의한다._

> API pod 는 deployment 가 관리한다. 수동으로 pod 를 지우더라도 대체 pod 가 생성된다. 새로 생성된 pod 역시 API 파드에 정의된 label selector 와 일치하므로 새로운 pod 에도 트래픽이 연결된다.

- API pod 이름과 IP 주소 확인
```
$ kubectl get pod -l app=numbers-api -o custom-columns=NAME:metadata.name,POD_IP:status.podIP

//
NAME                           POD_IP
numbers-api-7949c64648-kzrmm   10.1.0.29
```

- API pod 삭제 
```
$ kubectl delete pod -l app=numbers-api

//
pod "numbers-api-7949c64648-kzrmm" deleted
```

- 새로 생성된 pod 이름과 IP 주소 확인
```
$ kubectl get pod -l app=nubmers-api -o custom-columns=NAME:metadata.name,POD_IP:status.podIP

//
NAME                           POD_IP
numbers-api-7949c64648-8flgg   10.1.0.30
```

- Web application 에 port forwarding 적용
```
$ kubectl port-forward deploy/numbers-web 8080:80
```

_새로운 pod 는 이전과 다른 IP 를 할당 받지만, label 이 일치하고, 서비스의 IP 주소는 변경되지 않았기 때문에 Web application 는 전과 같은 주소로 API 파드와 통신할 수 있다._

서비스는 웹 애필리케이션 파드와 API 파드의 결합을 분리하므로 API 파드가 대체되어도 웹 애플리케이션 파드는 영향을 받지 않는다.

-> 서비스가 제공하는 추상화가 있으면, 지속적인 파드 교체에도 애플리케이션이 계속 서로 통신할 수 있다.

### 3.3 외부 트래픽을 파드로 전달하기

**LoadBalancer Service**
- 외부에서 또는 다른 노드에서 들어오는 트래픽을 대상 파드로 전달할 수 있다.
- 클러스터로 트래픽을 전달해주는 외부 로드밸런서와 함께 동작하며, label selector 와 일치하는 pod 로 traffic 을 전달한다.

> Loadbalancer service 를 클러스터에 배치하고, kubectl 을 사용하여 해당 서비스의 IP 주소 확인
- Loadbalancer service 배치
```
$ kubectl apply -f numbers/web-service.yaml

//
service/numbers-web created
```

- Serivce 상세 정보 확인
```
$ kubectl get svc numbers-web

//
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
numbers-web   LoadBalancer   10.103.253.130   localhost     8080:30111/TCP   30s
```

- Application 의 URL 을 EXTERNAL-IP 필드로 출력한다
```
$ kubectl get svc numbers-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080'

//
http://localhost:8080Z
```

_로드밸런서 서비스는 실제 IP 주소를 부여받는다. AKS, EKS 클러스터에서 한다면, 클라우드에서 제공되는 공인 IP 주소를 부여받는다._

**Node Port**
- 외부 트래픽을 파드로 전달
- 별도의 로드밸런서가 필요없다
- 모든 노드가 서비스에 설정된 포트를 주시한다.
- 서비스에서 설정된 포트가 **모든 노드** 에서 개방되어 있어야 하기 때문에 유연하지 않다.

### 3.4 쿠버네티스 클러스터 외부로 트래픽 전달하기

- ExternalName 서비스를 사용하면 클러스터 외부의 컴포넌트를 로컬 주소 또는 로컬 도메인 네임으로 참조할 수 있다
  - ExternalName 서비스는 도메인 네임의 별명을 만든다.
  - pod가 로컬 클러스터 네임 db-service 를 사용하면, 쿠버네티스 DNS 서버에서 이 도메인 네임을 외부 도메인 app.mydatabase.io 로 해소한다 

> 쿠버네티스 버전에 따라 이미 배포된 서비스 리소스의 유형을 변경할 수 없는 경우가 있기 때문에 원래 있던 클러스터IP 서비스를 삭제하고, ExternalName 서비스로 대체한다.

- 클러스터IP 서비스 삭제
```
$ kubectl delete svc numbers-api

//
service "numbers-api" deleted
```

- ExternalName 서비스를 새로 배포
```
$ kubectl apply -f numbers-services/api-service-externalName.yaml

//
service/numbers-api created
```

- 서비스 상세 정보 확인
```
$ kubectl get svc numbers-api

//
NAME          TYPE           CLUSTER-IP   EXTERNAL-IP                 PORT(S)   AGE
numbers-api   ExternalName   <none>       raw.githubusercontent.com   <none>    34s
```

> 서비스 역시 클러스터 전체를 커버하는 쿠버네티스 가상 네트워크의 일부이다. 모든 파드는 서비스를 사용할 수 있다. API 서비스의 도메인 네임을 조회하라.

- nslookup 명령으로 서비스의 도메인 네임을 조회
```
$ kubectl exec deploy/sleep-1 -- sh -c 'nslookup numbers-api | tail -n 5'

//
Name:   raw.githubusercontent.com
Address: 185.199.110.133
Name:   raw.githubusercontent.com
Address: 185.199.108.133
```

**Headless Service**

> ExternalName Service 를 Headless Service 로 대체하면, API 도메인이 접근할 수 없는 IP 주소로 해소되기 때문에 오류가 발생한다.

- 기존 서비스 제거
```
$ kubectl delete svc numbers-api

//
service "numbers-api" deleted
```

- Headless service 배포
```
$ kubectl apply -f numbers-services/api-service-headless.yaml

//
service/numbers-api unchanged
endpoints/numbers-api created
```

- 서비스 상세 정보 확인
```
$ kubectl get svc numbers-api

//
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
numbers-api   ClusterIP   10.110.206.4   <none>        80/TCP    3m10s
```

- Endpoint 상세 정보 확인
```
$ kubectl get endpoints numbers-api

//
NAME          ENDPOINTS            AGE
numbers-api   192.168.123.234:80   79s
```

- DNS 조회 결과 확인
```
$ kubectl exec deploy/sleep-1 -- sh -c 'nslookup numbers-api | grep "^[^*]"'

//
Server:         10.96.0.10
Address:        10.96.0.10:53
Name:   numbers-api.default.svc.cluster.local
Address: 10.110.206.4
```

_도메인 네임이 내부 클러스터IP 로 해소되는데, 이 IP 주소가 엔드포인트에서 실재하지 않는 주소로 연결되기 때문에 요청이 실패한다_

**Q1. DNS 조회 결과가 왜 엔드포인트의 IP 주소가 아니라 클러스터IP 주소를 가리킬까?**

**Q2. 도메인 네임은 왜 .default.svc.cluster.local 과 같이 끝날까?**

### 3.5 쿠버네티스 서비스의 해소 과정

#### Q1

파드는 각 노드마다 동작하는 네트워크 프록시를 경유하여 네트워크에 접근한다.

서비스 리소스는 삭제될 때까지 IP 주소가 변경되지 않는다.

서비스의 컨트롤러는 파드가 변경될 때마다 엔드포인트의 목록을 최신으로 업데이트한다. 

> 파드의 변경이 일어난 시점을 전후로 서비스의 엔드포인트 목록을 출력해 보면 즉각 엔드포인트가 업데이트되는 것을 확인할 수 있다. 엔드포인트의 이름은 서비스와 같으므로 kubectl 을 사용하여 엔드포인트의 상세 정보를 볼 수 있다

- sleep-2 서비스의 엔드포인트 목록 출력
```
$ kubectl get endpoints sleep-2

//
NAME      ENDPOINTS      AGE
sleep-2   10.1.0.27:80   47h
```

- 파드 삭제
```
$ kubectl delete pods -l app=sleep-2

//
pod "sleep-2-6695c7cd68-p2tx2" deleted
```

- 엔드포인트가 새로운 파드의 주소로 업데이트 되었는지 확인
```
$ kubectl get endpoints sleep-2

//
NAME      ENDPOINTS      AGE
sleep-2   10.1.0.31:80   47h
```

- deployment 삭제
```
$ kubectl delete deploy sleep-2

//
deployment.apps "sleep-2" deleted
```

- 엔드포인트는 여전히 있지만, 가리키는 IP 주소가 없음
```
$ kubectl get endpoints sleep-2

//
NAME      ENDPOINTS   AGE
sleep-2   <none>      47h
```

쿠버네티스 DNS 서버는 엔드포인트 IP 주소가 아닌 클러스터의 IP 주소를 반환한다. 엔드포인트가 가리키는 IP 주소는 계속 변화하기 때문이다.

서비스의 클러스터IP 주소는 변하지 않지만, 서비스가 가리키는 엔드포인트 주소의 목록은 지속적으로 변화한다.

#### Q2

네임스페이스는 쿠버네티스 클러스터를 논리적 파티션으로 분할하는 역할을 하지만, 서비스는 자신이 속한 네임스페이스 외의 네임스페이스로도 접근이 가능하다.

클러스터는 여러 개의 네임스페이스로 나뉠 수 있다. default 네임스페이스가 항상 존재한다.

네임스페이스 안에서는 도메인 네임을 이용하여 서비스에 접귾나다.

네임스페이스를 포함하는 완전한 도메인 네임으로도 서비스에 접근할 수 있다.

DNS 서버나 쿠버네티스 API 같은 쿠버네티스 내장 컴포넌트는 kube-system 네임스페이스에 속한 파드에서 동작한다.

> kubectl 에서 --namespace 플래그를 사용하면 default 가 아닌 다른 네임스페이스를 대상으로 지정할 수 있다

- default 네임스페이스의 서비스 리소스 목록 확인
```
$ kubectl get svc --namespace default

//
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes    ClusterIP      10.96.0.1        <none>        443/TCP          6d22h
numbers-api   ClusterIP      10.110.206.4     <none>        80/TCP           44h
numbers-web   LoadBalancer   10.103.253.130   localhost     8080:30111/TCP   46h
sleep-2       ClusterIP      10.108.192.173   <none>        80/TCP           47h
```

- 쿠버네티스 시스템 네임스페이스의 서비스 리소스 목록 확인
```
$ kubectl get svc -n kube-system

//
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   6d22h
```

- 완전한 도메인 네임으로 DNS 조회하기
```
$ kubectl exec deploy/sleep-1 -- sh -c 'nslookup numbers-api.default.svc.cluster.local | grep "^[^*]"'

//
Server:         10.96.0.10
Address:        10.96.0.10:53
Name:   numbers-api.default.svc.cluster.local
Address: 10.110.206.4

```

- 쿠버네티스 시스템 네임스페이스의 완전한 도메인 네임으로 DNS 조회하기
```
$ kubectl exec deploy/sleep-1 -- sh -c 'nslookup kube-dns.kube-system.svc.cluster.local | grep "^[^*]"'

//
Server:         10.96.0.10
Address:        10.96.0.10:53
Name:   kube-dns.kube-system.svc.cluster.local
Address: 10.96.0.10
```

로컬 도메인 네임은 네임스페이스를 포함하는 완전한 도메인 네임의 별명이다.

네임스페이스는 클러스터를 분할하여 보안을 해치지 않고도 클러스터 활용도를 높일 수 있는 강력한 수단이다. 

> 디플로이먼트를 삭제하면 디폴로이먼트가 관리하는 파드도 모두 삭제된다. 하지만 서비스를 삭제할 때는 서비스의 대상 파드나 디폴로이먼트가 삭제되지 않는다. 서비스와 디플로이먼트는 따로따로 삭제해야 한다.

- 모든 deployment 삭제
  - deployment 를 삭제하면 파드도 함께 삭제된다. (default 네임스페이스에 속한 리소스가 삭제) 
```
$ kubectl delete deploy --all

//
deployment.apps "numbers-api" deleted
deployment.apps "numbers-web" deleted
deployment.apps "sleep-1" deleted
```



- 모든 서비스 삭제
  - default 네임스페이스에 있는 쿠버네티스 API 까지 삭제
```
$ kubectl delete svc --all

//
service "kubernetes" deleted
service "numbers-api" deleted
service "numbers-web" deleted
service "sleep-2" deleted
```

- 남아 있는 리소스 확인
  - kube-system 네임스페이스에 있는 컨트롤러 객체가 쿠버네티스 API 를 복구 

```
$ kubectl get all 

//
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   19s
```
### 3.6 연습 문제 

Q. 사용자 인터페이스가 개선된 무작위 숫자 생성 애플리케이션의 새 버전을 서비스로 배포

A.
```
kubectl apply -f lab/deployments.yaml

kubectl apply -f lab/api-service.yaml -f lab/web-service.yaml
```
