---
layout: post
title:  "Chaos Engineering in TACO"
date:   2018-02-26 11:00:00 +0900
author: "이재상"
---

<p>by 이재상(<a href="mailto:jaesang_lee@sk.com">jaesang_lee@sk.com</a>)</p>

## Overview
 보통 서비스 및 소프트웨어를 개발하는 경우, 최초에 기능 검증을 하거나 빠르게 개발하기 위해 하나의 서버에 모든 기능을 구현합니다. 그러나 서비스 규모가 증가한다던가 혹은 서비스 요청이 무척 많아질 것을 대비하려면 서버 한대로는 원하는 것을 이룰 수 없겠죠. 이에 서비스를 분산 배포하기 시작합니다. 예를 들어 소규모 서비스일 때는 DB서버 1대만 구축하지만, 서비스 리퀘스트가 증가한다면 복수의 DB서버로 클러스터링을 구성하는 식입니다. 이렇게 증가되는 서비스는 DB뿐만이 아닙니다. TACO의 경우 OpenStack 서비스별 API, Scheduler가 다수 실행되며 RabbitMQ, Ingress, Etcd, Open vSwitch, Memcached 등 수 많은 서비스가 분산 배포됩니다.

 이런 서비스를 개발하면서 그에 대한 안정성 테스트를 생각하게 됩니다. 분산 서비스를 하는 이유는 속도 개선도 있지만 서버 한대가 이상이 생겨도 나머지 서버들이 서비스를 안전하게 동작시키기 위함입니다. 때문에 전통적인 기능, 성능 테스트뿐 아니라 장애 테스트를 추가하게됩니다.

 장애 테스트를 하려면 다음을 규정해야 합니다.

1. 정상 상태에 대한 규정 - 정상이라고 판단하기 위해 동작해야 하는 기능 리스트
2. 장애 발생 변수에 대한 규정 - 서비스의 구성요소(하드웨어, 네트워크, 스토리지 장애 및 OS 장애 등등)

 위 두가지를 정의했다면 이제 정상 상태의 서비스에 장애를 발생시키면서 서비스가 정상 작동하는지 검증해야합니다. 이는 실제로 일어날 수 있는 상황들을 가정하기 때문에 운영을 하기 위해 꼭 필요한 테스트입니다. 서비스 운영 시 발생하는 장애 요인을 미리 제거하는 이러한 활동을 카오스 엔지니어링<sup name="a1">[1](#f1)</sup>이라고 합니다.


![TACO CI/CD]({{ site.baseurl }}{{ post.url }}/assets/img/cookiemonster/taco-cicd.png)
[![License: CC BY-NC-ND 4.0](https://licensebuttons.net/l/by-nc-nd/4.0/80x15.png)](https://creativecommons.org/licenses/by-nc-nd/4.0/)

 위 그림은 내부에 구축한 CI/CD Pipeline에 대한 구성도입니다. OpenStack Container 빌드 혹은 Helm Chart 빌드 때마다 그림에 표시된 "Integration Test"단계에서 아래에서 설명할 장애테스트를 수행하여 TACO 서비스의 안정성을 검증하고 있습니다. 이 글에서는 TACO에서 사용하는 장애 발생 툴인 Cookiemonster와 장애 테스트에 대해 설명합니다. TACO CI/CD에 대해서는 따로 포스트를 작성할 예정입니다.

## Kubernetes 구조
 TACO는 Kubernetes위에 구축된 OpenStack 서비스로 TACO를 알기 위해서는 Kubernetes의 이해가 필요합니다. Kubernetes의 대한 내용은 [이전 글]({{ site.baseurl }}{{ post.url }}/Taco/#kubernetes)을 참고하시기 바랍니다.

## 장애 발생 툴
 Kubernetes환경에서 사용하는 장애 발생 툴은 여러가지가 있습니다. 대부분 넷플릭스의 카오스몽키의 영감을 갖고 만들어졌으며, 각자 Kubernetes 환경에 대한 장애를 발생시킵니다.

- kube-monkey
- chaoskube
- powerfulseal

| Tools         | Repo                                      | 실행방법          |
| ------------- |------------------------------------------ | ----------------- |
| kube-monkey   | https://github.com/asobti/kube-monkey     | Install template  |
| chaoskube     | https://github.com/linki/chaoskube        | Command Line      |
| powerfulseal  | https://github.com/bloomberg/powerfulseal | Command Line      |

## Cookiemonster
 Cookiemonster는 SKT에서 개발한 Apache 2.0 라이센스의 오픈소스 프로젝트로 TACO의 CI/CD에서 사용하고 있는 Kubernetes용 장애발생툴입니다. 또한, 구축된 TACO에 대한 장애테스트 용도로도 사용할 수 있습니다.
   - Project Code: <https://github.com/sktelecom-oslab/cookiemonster>

 아래 구성도에 표시된 것 처럼 Cookiemonster pod와 worker pod으로 구성되며 다음 역할을 수행합니다.
![Cookiemonster 구성]({{ site.baseurl }}{{ post.url }}/assets/img/cookiemonster/cookiemonster.png)
[![License: CC BY-NC-ND 4.0](https://licensebuttons.net/l/by-nc-nd/4.0/80x15.png)](https://creativecommons.org/licenses/by-nc-nd/4.0/)
- Cookiemonster pod: REST API 서버, 외부로부터 요청을 받고 worker pod과 통신
- worker pod: cookiemonster pod으로부터 받은 요청 수행, 실질적인 장애 발생

### 개발이유
 기존 장애 유발 툴들은 테스트를 백그라운드가 아닌 포어그라운드에서 수행합니다. 때문에 장애를 내고 있는 화면이 아닌 다른 곳에서 검증을 수행해야하는데 이는 저희가 사용하는 Jenkins에 적합하지 않았습니다. (Jenkins에서 parallel 작업을 지원하지만 깔끔하게 적용하기가 어려웠습니다.) 만약에 테스트가 백그라운드로 수행된다면, Jenkins 내에서 장애를 발생시킨 후 바로 검증 작업을 수행할 수 있을 것입니다. Cookiemonster는 이러한 필요에 의해 REST API로 서비스 요청을 수행하며 백그라운드로 장애발생을 시작/중단하여 손쉽게 Jenkins Job으로 적용할 수 있습니다.  

### 설치방법
 github 저장소에서 Cookiemonster를 다운로드 받습니다.

```bash
taco@ctrl01-stg:~$ git clone https://github.com/sktelecom-oslab/cookiemonster.git
taco@ctrl01-stg:~$ cd cookiemonster
```
 kubectl로 k8s폴더에 있는 템플릿을 설치합니다.

```bash
taco@ctrl01-stg:~/cookiemonster$ kubectl create ns cookiemonster
taco@ctrl01-stg:~/cookiemonster$ kubectl create -f ./k8s
taco@ctrl01-stg:~/cookiemonster$ kubectl get po -n cookiemonster
NAME                             READY     STATUS    RESTARTS   AGE
cookiemonster-7dfdf5d77d-blscj   1/1       Running   0          2s
cookiemonsterworker-7tk4f        1/1       Running   0          3s
cookiemonsterworker-gd924        1/1       Running   0          3s
cookiemonsterworker-k7vrc        1/1       Running   0          3s
cookiemonsterworker-k8lzz        1/1       Running   0          3s
cookiemonsterworker-kccdm        1/1       Running   0          3s
cookiemonsterworker-lmzbc        1/1       Running   0          3s
cookiemonsterworker-lwtcv        1/1       Running   0          3s
cookiemonsterworker-rhnsj        1/1       Running   0          3s
```

 cookiemonster pod이 제대로 배포되지 않았다면 ```kubectl create -f ./k8s```을 한번 더 수행합니다.

### 사용방법
 cookiemonster는 30003 포트로 API 요청을 처리합니다. 포트 정보는 아래 명령으로 확인합니다.

```bash
taco@ctrl01-stg:~/cookiemonster$ kubectl get svc -n cookiemonster
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
cookiemonster   NodePort   10.96.34.201   <none>        8080:30003/TCP   30s
```

 kubernetes 노드 중 한 곳의 IP:30003로 API를 요청합니다.

```bash
taco@ctrl01-stg:~/cookiemonster$ curl localhost:30003/list
killpod
nodeexec

taco@ctrl01-stg:~$ curl localhost:30003/list/killpod
openstack-random-daemonset
openstack-random-deployment
openstack-random-statefulset
```

 현재 cookiemonster는 killpod API만 제공하며 1분에 한번씩 pod을 random하게 파괴시킵니다.  openstack으로 시작하는 API 내용은 cookiemonster/cookies.d/killpod 폴더에서 확인할 수 있습니다.

```bash
taco@ctrl01-stg:~/cookiemonster$ cat cookies.d/killpod/openstack-random-deployment.json
{
  "kind": "deployment",
  "namespace": "openstack",
  "target": 1,
  "interval": 30,
  "duration": 600,
  "slack": false
}
```

 openstack-random-deployment.json의 경우 openstack 네임스페이스에 있는 deployment를 30초에 한번씩 랜덤하게 파괴시킵니다. json에 명시된 값은 모두 변경 가능합니다.

#### 장애 발생
 이제 장애 발생 API를 호출해보도록 하겠습니다.

```bash
taco@ctrl01-stg:~$ curl localhost:30003/start/killpod/openstack-random-deployment
start initiated
```

 cookiemonster pod의 로그를 확인합니다. 30초마다 랜덤하게 deployment를 파괴시키고 있습니다.

~~~
taco@ctrl01-stg:~$ kubectl logs cookiemonster-7dfdf5d77d-blscj -n cookiemonster -f
2018/02/28 04:27:49 checking directory:  /cookies.d
2018/02/28 04:27:49 checking directory:  /etc/cookies.d
2018/02/28 04:27:49 loading  /etc/cookies.d/killpod/openstack-random-deployment.json
2018/02/28 04:27:49 Cookie Time!!! Random feast starting on deployment in namespace openstack
2018/02/28 04:28:19 Running in Kubernetes cluster
2018/02/28 04:28:19 deployment etcd in namespace openstack has 1 pods defined, 1 available and 0 unavailable
2018/02/28 04:28:19 Only one pod available, doing nothing
2018/02/28 04:28:49 Running in Kubernetes cluster
2018/02/28 04:28:49 deployment horizon in namespace openstack has 3 pods defined, 3 available and 0 unavailable
2018/02/28 04:28:49 Found deployment horizon in namespace openstack
2018/02/28 04:28:49 Eating pod horizon-796f7b5f5-fn7mn NOM NOM NOM!!!!
2018/02/28 04:29:19 Running in Kubernetes cluster
2018/02/28 04:29:19 deployment glance-api in namespace openstack has 3 pods defined, 3 available and 0 unavailable
2018/02/28 04:29:19 Found deployment glance-api in namespace openstack
2018/02/28 04:29:19 Eating pod glance-api-59ff964c7b-p6k8d NOM NOM NOM!!!!
2018/02/28 04:29:49 Running in Kubernetes cluster
2018/02/28 04:29:49 deployment cinder-backup in namespace openstack has 1 pods defined, 1 available and 0 unavailable
2018/02/28 04:29:49 Only one pod available, doing nothing
~~~

#### 장애 중단
 장애 중단 API는 다음과 같습니다.

```bash
taco@ctrl01-stg:~$ curl localhost:30003/stop/killpod/openstack-random-deployment
stop initiated
```

 cookiemonster pod의 로그를 확인합니다. 장애 발생이 중단된 것을 볼 수 있습니다.

```bash
taco@ctrl01-stg:~$ kubectl logs cookiemonster-7dfdf5d77d-blscj -n cookiemonster -f
2018/02/28 04:30:37 checking directory:  /cookies.d
2018/02/28 04:30:37 checking directory:  /etc/cookies.d
2018/02/28 04:30:37 loading  /etc/cookies.d/killpod/openstack-random-deployment.json
2018/02/28 04:30:37 Done snacking on openstack-random-deployment
```

## 장애테스트 방법
 TACO CI/CD의 장애테스트는 다음의 구조로 실행됩니다. Cookiemonster를 통해 장애를 발생시키고 Rally를 통해 검증하는 방식입니다.

* 배포 - OpenStack 서비스를 분산 구조로 배포
* Cookiemonster 설치/실행 - OpenStack 서비스 장애 발생
* Rally 테스트 실행 - OpenStack 서비스의 정상작동 확인
* Cookiemonster 중단 - OpenStack 서비스 장애 중단

![장애테스트 Pipeline]({{ site.baseurl }}{{ post.url }}/assets/img/cookiemonster/hatest.png)
[![License: CC BY-NC-ND 4.0](https://licensebuttons.net/l/by-nc-nd/4.0/80x15.png)](https://creativecommons.org/licenses/by-nc-nd/4.0/)

## 마무리

 장애테스트는 서비스 안정성을 검증하기 위해 꼭 수행해야 하는 테스트입니다. Kubernetes를 구축하고 이 위에 서비스를 배포하였다면 장애 유발 테스트 툴을 사용하여 장애 환경을 시뮬레이션할 수 있습니다. 여러 장애 유발 테스트툴을 사용해보고 가장 적합한 테스트 툴을 찾아야할 것입니다. 마지막으로 Cookiemonster에도 관심이 있으신 분은 github을 통해서 개발에 참여하거나 이슈를 남겨주시면 좋겠습니다.



참고
<b><a name="f1">[1](#a1)</a></b> 카오스 엔지니어링: http://principlesofchaos.org http://channy.creation.net/blog/1173
