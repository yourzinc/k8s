## 10장 헬름을 이용한 애플리케이션 패키징 및 관리

Helm
- 여러 개의 YAML 정의 스크립트를 하나의 아티팩트로 묶어 public or private repository 에 공유
- repository 에 접근 권한이 있는 사용자만 application 설치 가능
- kubernetes resource 배치와 설정값 조정


### 10.1 헬름이 제공하는 기능

- 명령형 도구 형태로 repository server 와 상호 작용
- Application package 를 찾아 내려받고 kubernetes cluster 에 설치 및 관리

Helm package : kubernetes 의 manifest 묶음

> Install Helm

```
$ brew install helm
```

```
$ helm version

//
version.BuildInfo{Version:"v3.14.4", GitCommit:"81c902a123462fd4052bc5e9aa9c513c4c8fc142", GitTreeState:"clean", GoVersion:"go1.22.2"}
```

Helm repository
- Docker repository 같은 container image registry 와 유사

> Add Helm repository and search application after synchronizing index

- 원격 서버를 가리키는 이름으로 repository 추가
```
$ helm repo add kiamolo https://kiamol.net

//
"kiamol" has been added to your repositories
```

- Update local repository cache 
```
$ helm repo update

//
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "kiamol" chart repository
Update Complete. ⎈Happy Helming!⎈
```

- Search application with index cache
```
$ helm search repo vweb --versions

//
NAME       	CHART VERSION	APP VERSION	DESCRIPTION             
kiamol/vweb	2.0.0        	2.0.0      	Simple versioned web app
kiamol/vweb	1.0.0        	1.0.0      	Simple versioned web app
```

**Chart**
- Application package
- local 에서 만들어져 local computer 에 설치될 수도 있고, repository 로 배포될 수도 있음
- kubernetes YAML manifest 가 들어있음
- 압축 파일 형태
- Name, Version 
- Name = 압축 파일 내부에 directory 이름

- e.g
    - vweb-1.0.0.tgz

```
.
└── vweb
    └── Chart.yaml  # Chart name, version, description, metadata
    └── templates   # kubernetes manifest
        └── deployment.yaml
        └── service.yaml
└── values.yaml     # parameter default value used in kubernetes manifest
```

**Release**
- 설치한 chart
- release 에는 여러 이름을 붙일 수 있음 
- 하나의 cluster 에 같은 chart 를 여러 벌 설치 가능

> vweb chart version 1

- check the default value of parameter included in chart
```
$ helm show values kiamol/vweb --version 1.0.0

//
servicePort: 8090
replicaCount: 2
```

- Modify the default parameter value and install chart
```
$ helm install --set servicePort=8010 --set replicaCount=1 ch10-vweb kiamol/vweb --version 1.0.0

//
NAME: ch10-vweb
LAST DEPLOYED: Mon Apr 29 20:43:18 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

- Check the realase
```
$ helm ls

//
NAME     	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART     	APP VERSION
ch10-vweb	default  	1       	2024-04-29 20:43:18.169767 +0900 JST	deployed	vweb-1.0.0	1.0.0      
```

헬름을 이용하여 Application 을 설치하면 kubectl 명령을 사용하지 않아도 kubernetes resource 가 생성

resource 설치 후에는 이전과 같은 방식으로 `kubectl` 을 이용하여 kubernetes resource 를 다룰 수 있음

> Check the resources deployed by Helm using kubectl. Scale the deployment with Helm. Check the application running.

- Check deployment status
```
$ kubectl get deploy -l app.kubernetes.io/instance=ch10-vweb --show-labels

//
NAME        READY   UP-TO-DATE   AVAILABLE   AGE    LABELS
ch10-vweb   1/1     1            1           4m6s   app.kubernetes.io/instance=ch10-vweb,app.kubernetes.io/managed-by=Helm,app.kubernetes.io/name=vweb,kiamol=ch10
```

- Update the release by modifying replica number
```
$ helm upgrade --set servicePort=8010 --set replicaCount=3 ch10-vweb kiamol/vweb --version 1.0.0

//
Release "ch10-vweb" has been upgraded. Happy Helming!
NAME: ch10-vweb
LAST DEPLOYED: Mon Apr 29 20:48:38 2024
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
```
parameter 설정값을 별도로 지정하지 않으면 기본값 (default value) 가 적용되기 때문에 helm upgrade 명령에서 service port 를 지정함

- Check ReplicaSet status
```
$ kubectl get rs -l app.kubernetes.io/instance=ch10-vweb

//
NAME                   DESIRED   CURRENT   READY   AGE
ch10-vweb-56b76bcffb   3         3         3       6m12s
```

- Check the Application url
```
$ kubectl get svc ch10-vweb -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8010'

//
http://localhost:8010
```

헬름은 Application 을 관리하는 도구는 아니지만, 설정값을 update 할 수 있음

### 10.2 헬름으로 애플리케이션 패키징하기

- chart.yaml : 차트의 이름이나 버전 등 메타데이터를 기록한 파일
- values.yaml : 파라미터 값의 기본값을 기록한 파일
- templates/ : 템플릿 변수가 포함된 kubernetes manifest 파일을 담은 디렉터리

> Chart 를 만들 때는 압축하지 않아도 된다. Chart directory 에서 그대로 작업할 수 있다

```
$ cd ch10
```

```
$ tree .

//
.
├── Chart.yaml
├── templates
│   └── web-ping-deployment.yaml
└── values.yaml
```

- Chart 에 들어갈 파일의 유효성 검증
```
$ helm lint web-ping

//
==> Linting web-ping
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

- Chart directory 에서 release 설치
```
$ helm install wp1 web-ping/

//
NAME: wp1
LAST DEPLOYED: Mon Apr 29 21:37:27 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

- Check the release status
```
$ helm ls

//
NAME     	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART         	APP VERSION
ch10-vweb	default  	2       	2024-04-29 20:48:38.916742 +0900 JST	deployed	vweb-1.0.0    	1.0.0      
wp1      	default  	1       	2024-04-29 21:37:27.779278 +0900 JST	deployed	web-ping-0.1.0	1.0.0     
```

local 에 위치한 Chart 는 directory and 압축 파일 모두 가능
- local directory 로 된 Chart 는 repository 에 Chart 를 패키징하기 전 단계에서 Chart 를 개발할 때 사용

Helm 은 하나의 Chart 로 동일한 Application 을 여러 벌 실행할 수 있음
kubectl 은 리소스 이름이 동일하기 때문에 불가능

> web-ping Application 을 동일한 차트를 요청하되 요청 대상 URL 을 달리하여 새로운 Release 를 추가 배치

- Check the setting value in Chart
```
$ helm show values web-ping/ 

//
# targetUrl - URL of the website to ping
targetUrl: blog.sixeyed.com
# httpMethod - HTTP method to use for pings
httpMethod: HEAD
# pingIntervalMilliseconds - interval between pings in ms
pingIntervalMilliseconds: 30000
# chapter where this exercise is used
kiamolChapter: ch10
```

- Add another release with name wp2
```
$ helm install --set targetUrl=kiamol.net wp2 web-ping/

//
NAME: wp2
LAST DEPLOYED: Mon Apr 29 21:46:19 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

- Check log after 1 mintue
```
$ kubectl logs -l app=web-ping --tail 1

//
Got response status: 200 at 1714394817404; duration: 1226ms
Got response status: 200 at 1714394803507; duration: 1058ms
```

동일한 Chart 도 다른 이름으로 배치하면 완전히 별개의 Release 가 된다

2개의 Deployment pod 모두 app label 이 동일하다

Helm 을 사용하려면, 모든 환경에 Helm 을 도입해야 한다

개발 중에는 kubernetes manifest 를 사용하되, 개발 환경 외의 환경에는 Helm 을 이용할 수 있다

ChartMuseum : Repository server
- 비공개 Helm Repository 를 마련

> ChartMuseum 을 설치하고 local 환경에 전용 repository 운영
- 공식 Helm repository 추가
```
$ helm repo add stable https://charts.helm.sh/stable

//
"stable" has been added to your repositories
```

- Install ChartMuseum 
```
$ helm install --set service.type=LoadBalancer --set service.externalPort=8008 --set env.open.DISABLE_API=false repo stable/chartmuseum --version 2.13.0 --wait

//

NAME: repo
LAST DEPLOYED: Mon Apr 29 22:10:21 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
** Please be patient while the chart is being deployed **

Get the ChartMuseum URL by running:

** Please ensure an external IP is associated to the repo-chartmuseum service before proceeding **
** Watch the status using: kubectl get svc --namespace default -w repo-chartmuseum **

  export SERVICE_IP=$(kubectl get svc --namespace default repo-chartmuseum -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP:8008/

OR

  export SERVICE_HOST=$(kubectl get svc --namespace default repo-chartmuseum -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
  echo http://$SERVICE_HOST:8008/
```

- Local ChartMuseum URL
```
$ kubectl get svc repo-chartmuseum -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8008'

//
http://localhost:8008
```

- 설치된 ChartMuseum 을 local 이라는 이름으로 repository 등록
```
$ helm repo add local $(kubectl get svc repo-chartmuseum -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8008')

//
"local" has been added to your repositories
```

Chart 를 repository 에 배포하려면 먼저 packaging 을 거쳐야 한다

1. Chart 를 zip 압축 파일로 압축 
2. 압축된 파일을 server 에 업로드
3. Repository index 에 새로운 chart 정보 추가 (by ChartMuseum)

> Helm 을 이용하여 Chart 를 압축하고, curl 을 사용하여 local ChartMuseum Repository 에 업로드. Repository 를 확인하여 index 에 새로 만든 Chart 가 추가되었는지 확인

- Chart packaging
```
$ helm package web-ping

//
Successfully packaged chart and saved it to: /Users/zinc/downloads/kiamol-master/ch10/web-ping-0.1.0.tgz
```

- Upload the zip file to repository 
```
$ curl --data-binary "@web-ping-0.1.0.tgz" $(kubectl get svc repo-chartmuseum -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8008/api/charts')

//
{"saved":true}
```

- Check the chart in ChartMuseum
```
$ curl $(kubectl get svc repo-chartmuseum -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8008/index.yaml')

//
apiVersion: v1
entries:
  web-ping:
  - apiVersion: v2
    appVersion: 1.0.0
    created: "2024-04-29T13:17:58.027486002Z"
    description: A simple web pinger
    digest: af98ce36c085dad0f983bf8ac92dcf778a594332ccf17cb5758f72028f7476fa
    name: web-ping
    type: application
    urls:
    - charts/web-ping-0.1.0.tgz
    version: 0.1.0
generated: "2024-04-29T13:18:23Z"
serverInfo: {}
```

Image 는 Release 가 설치될 때 Docker Hub 또는 그 외 Registry 에서 내려받는다

ChartMuseum 또는 Repository server 를 사용하여 조직 내 Applicaion 을 공유하거나, public repository 에 내보낼 Release 후보를 정하는 CI 의 일부로 Chart 를 푸시

> Install another version of web-ping application. Use the chart from the local repository and the setting configuration files. 

- Update the repository cache
```
$ helm repo update

//
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "kiamol" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
```

- Check the chart uploaded previously 
```
$ helm search repo web-ping

//
NAME          	CHART VERSION	APP VERSION	DESCRIPTION        
local/web-ping	0.1.0        	1.0.0      	A simple web pinger
```

- Install the Chart using setting files
```
$ helm install -f web-ping-values.yaml wp3 local/web-ping

//
NAME: wp3
LAST DEPLOYED: Mon Apr 29 22:25:38 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

- Check the pods configuring the web-ping application
```
$ kubectl get pod -l app=web-ping -o custom-columns='NAME:.metadata.name,ENV:.spec.containers[0].env[*].value'

//
NAME                   ENV
wp1-68ff7f56f4-664pw   blog.sixeyed.com,HEAD,30000
wp2-5b45f644f7-dgfmt   kiamol.net,HEAD,30000
wp3-6bb4d4b5bf-m8tz8   blog.sixeyed.com,HEAD,15000
```

Helm 설치할 때는 환경별로 설정값을 따로 저장 가능
- set parameter 대신 설정값 파일을 사용 (Recommended)
- 설정값은 pod container 에 환경 변수로 전달

### 10.3 차트 간 의존 관계 모델링하기

Reverse proxy 에 cahce 가 필요한 web application 을 배치할 때, proxy 를 먼저 배치해야 한다 (의존 관계)

```
# chart.yaml

apiVersion: v2
name: pi
version: 0.1.0
dependencies:
  - name: vweb
    version: 2.0.0
    repository: https://kiamol.net
    condition: vweb.enabled         # 필요할 때만 설치
  - name: proxy
    version: 0.1.0
    repository: file://../proxy
    conition: proxy.enabled         # 필요할 때만 설치
```

최대한 일반적으로 활용할 수 있는 차트를 하위 차트로 배치 
c.f 애플리케이션 차트에 직접 포함시키기

> Install the proxy chart 
- Install with local chart 
```
$ helm install --set upstreamToProxy=ch10-vweb:8010 vweb-proxy proxy/

//
NAME: vweb-proxy
LAST DEPLOYED: Mon Apr 29 22:37:27 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

- Check the proxy service URL 
```
$ kubectl get svc vweb-proxy-proxy -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080'

//
http://localhost:8080
```

하위 Chart proxy 는 어떤 Application 이든 proxing 할 수 있기 때문에 그 자체로 유용하다

> Build the pi Chart's dependencies Chart. 

- Build the child Chart
```
$ helm dependency build pi

//
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "kiamol" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 2 charts
Downloading vweb from repo https://kiamol.net
Deleting outdated charts
```

- Check the child Chart downloaded
```
$ ls ./pi/charts

//
proxy-0.1.0.tgz	vweb-2.0.0.tgz
```

조건부 의존 관계
e.g. PostgreSQL
- Dev : Install PostgreSQL locally
- Test : Use StatefulSet
- Prod : Use Managed Database Service

> Deploy the pi applicaation with child Proxy Chart. Run dry-run command to check the error in default settings. Modify the setting values in installing.

- Validate (Default setting values)
```
$ helm install pi1 ./pi --dry-run

//
NAME: pi1
LAST DEPLOYED: Mon Apr 29 22:50:32 2024
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: pi/templates/web-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: pi1-web
  labels:
    kiamol: ch10
spec:
  ports:
    - port: 80
      name: http
  selector:
    app: pi1
    component: web
  type: LoadBalancer
---
# Source: pi/templates/web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pi1-web
  labels:
    kiamol: ch10
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pi1
      component: web
  template:
    metadata:
      labels:
        app: pi1
        component: web
    spec:
      containers:
        - image: kiamol/ch05-pi
          command: ["dotnet", "Pi.Web.dll", "-m", "web"]
          name: web
          ports:
            - containerPort: 80
              name: http

```

- Install the application with modified settings value with proxy
```
$ helm install --set serviceType=ClusterIP --set proxy.enabled=true pi2 ./pi

//
NAME: pi2
LAST DEPLOYED: Mon Apr 29 22:51:09 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

- Check the application URL 
```
$ kubectl get svc pi2-proxy -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8030'

//
http://localhost:8030
```

`dry-run`
- template 변수를 모두 치환하여 YAML 파일을 완성
- 실제 application 배치 X

Proxy child Chart 를 사용하고 pi service 를 내부용으로 돌리면, application 은 proxy 를 거쳐서만 접근 가능


### 10.4 헬름으로 설치한 릴리스의 업그레이드와 롤백

Cluster 에 application 을 한 벌 더 설치하여 새 버전의 테스트를 해볼 수 있다 

> Install the vweb application version 2. 

- Helm Release list
```
$ helm ls -q

//
ch10-vweb
pi2
repo
vweb-proxy
wp1
wp2
wp3
```

```
$ helm ls 

//
NAME      	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART             	APP VERSION
ch10-vweb 	default  	2       	2024-04-29 20:48:38.916742 +0900 JST	deployed	vweb-1.0.0        	1.0.0      
pi2       	default  	1       	2024-04-29 22:51:09.82577 +0900 JST 	deployed	pi-0.1.0          	           
repo      	default  	1       	2024-04-29 22:10:21.310431 +0900 JST	deployed	chartmuseum-2.13.0	0.12.0     
vweb-proxy	default  	1       	2024-04-29 22:37:27.89988 +0900 JST 	deployed	proxy-0.1.0       	1.16.0     
wp1       	default  	1       	2024-04-29 21:37:27.779278 +0900 JST	deployed	web-ping-0.1.0    	1.0.0      
wp2       	default  	1       	2024-04-29 21:46:19.796301 +0900 JST	deployed	web-ping-0.1.0    	1.0.0      
wp3       	default  	1       	2024-04-29 22:25:38.323117 +0900 JST	deployed	web-ping-0.1.0    	1.0.0    
```

- Check the setting values of new version Chart 
```
$ helm show values kiamol/vweb --version 2.0.0

//
# port for the Service to listen on
servicePort: 8090
# type of the Service:
serviceType: LoadBalancer
# number of replicas for the web Pod
replicaCount: 2
```

- Install the Release using internal service
```
$ helm install --set servicePort=8020 --set replicaCount=1 --set serviceType=ClusterIP ch10-vweb-v2 kiamol/vweb --version 2.0.0

//
NAME: ch10-vweb-v2
LAST DEPLOYED: Mon Apr 29 23:06:01 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

- Set the Port forwarding for application test
```
$ kubectl port-forward svc/ch10-vweb-v2 8020:8020

//
Forwarding from 127.0.0.1:8020 -> 80
Forwarding from [::1]:8020 -> 80
Handling connection for 8020
Handling connection for 8020
```

서비스를 포함하는 Chart 는 보통 서비스의 유형을 선택할 수 있어 Application 을 외부로 노출시키지 않을 수 있다 

> 임시로 설치한 Version 2 Release 를 제거한 후, 기존 설정값을 그대로 재사용하여 남아있는 Version 1 Release 를 Version 2 로 upgrade 하라 

- Remove the Release version 2
```
$ helm uninstall ch10-vweb-v2

//
release "ch10-vweb-v2" uninstalled
```

- Check the setting value of version 1 release 
```
$ helm get values ch10-vweb

//
USER-SUPPLIED VALUES:
replicaCount: 3
servicePort: 8010
```

- Upgrade the version 2 using the default setting values
```
$ helm upgrade --reuse-values --atomic ch10-vweb kiamol/vweb --version 2.0.0

//
Release "ch10-vweb" has been upgraded. Happy Helming!
NAME: ch10-vweb
LAST DEPLOYED: Mon Apr 29 23:17:02 2024
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

- Helm History
```
$ helm history ch10-vweb

//
REVISION	UPDATED                 	STATUS    	CHART     	APP VERSION	DESCRIPTION     
1       	Mon Apr 29 23:16:14 2024	superseded	vweb-1.0.0	1.0.0      	Install complete
2       	Mon Apr 29 23:17:02 2024	deployed  	vweb-2.0.0	2.0.0      	Upgrade complete
```

> Retry the upgrade version 2.

- Save the settings value in the current Release
```
$ helm get values ch10-vweb -o yaml > vweb-values.yaml

//
replicaCount: 3
servicePort: 8010
```

- Upgrade using the flag (--atomic) and settings file
```
$ helm upgrade -f vweb-values.yaml --atomic ch10-vweb kiamol/vweb --version 2.0.0

//
Release "ch10-vweb" has been upgraded. Happy Helming!
NAME: ch10-vweb
LAST DEPLOYED: Mon Apr 29 23:23:48 2024
NAMESPACE: default
STATUS: deployed
REVISION: 4
TEST SUITE: None
```

- Check the service and ReplicaSet 
```
$ kubectl get svc,rs -l app.kubernetes.io/instance=ch10-vweb

//
NAME                TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/ch10-vweb   LoadBalancer   10.101.89.214   localhost     8010:32018/TCP   7m48s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/ch10-vweb-56b76bcffb   0         0         0       7m48s
replicaset.apps/ch10-vweb-c7f86bc7f    3         3         3       7m
```

> Rollback to V2 (Version 1.0.0. Replica=3) 

- Check the setting values in revision 2
```
$ helm get values ch10-vweb --revision 2

//
USER-SUPPLIED VALUES:
replicaCount: 3
servicePort: 8010
```

- Rollback to revision 2
```
$ helm rollback ch10-vweb 2

//
Rollback was a success! Happy Helming!
```

- Check the recent revision 
```
$ helm history ch10-vweb --max 2 -o yaml

//
- app_version: 2.0.0
  chart: vweb-2.0.0
  description: Upgrade complete
  revision: 4
  status: superseded
  updated: "2024-04-29T23:23:48.804742+09:00"
- app_version: 1.0.0
  chart: vweb-1.0.0
  description: Rollback to 2
  revision: 5
  status: deployed
  updated: "2024-04-29T23:27:13.782628+09:00"
```

Helm Release 는 Application 을 추상화

### 10.5 헬름은 어떤 상황에 적합한가

> Remove the all Release
```
$ helm uninstall $(helm ls -q)

//
release "ch10-vweb" uninstalled
release "pi2" uninstalled
release "repo" uninstalled
release "vweb-proxy" uninstalled
release "wp1" uninstalled
release "wp2" uninstalled
release "wp3" uninstalled
```
