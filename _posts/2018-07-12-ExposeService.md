---
layout: post
title:  "TACO 서비스 외부로 오픈하기"
author: "문현선"
date:   2018-07-12 11:00:00 +0900
---

<p>by 문현선(<a href="mailto:hyunsun.moon@sk.com">hyunsun.moon@sk.com</a>)</p>

# TACO 서비스 외부로 오픈하기
이번 포스트에서는 TACO로 구축한 오픈스택을 외부에 서비스하는 방법에 대해 알아보겠습니다. TACO가 K8S를 이용해 오픈스택을 구축하고 운영하는 플랫폼이란 건 이제 다들 아시죠? 기본적으로 TACO는 오픈스택의 구성 요소들을 다양한 형태의 K8S 리소스로 구현하고 있는데요, 이렇게 K8S 클러스터에 배포한 응용 서비스는 기본적으로 클러스터 내에서만 접근이 가능합니다. 때문에 외부에서 접근이 필요한 오픈스택 서비스의 API 엔드포인트들은 부가적으로 K8S가 제공하는 다음의 리소스들을 이용해 클러스터 밖에서도 접근이 가능하도록 노출시켜줘야 합니다.

* Node Port
* Ingress

지금부터 K8S 설치가 완료된 TACO 개발 환경에 Keystone을 배포하면서 위 두 가지 방법에 대해 자세히 살펴보고, 여러분의 환경에는 어떤 방법과 설정이 적합한지에 대해서도 한번 고민해 보겠습니다. 참고로, 설명에 사용된 개발 환경의 K8S 버전은 `v1.10.4`이고 kube-proxy는 `iptables` 모드로 실행했습니다. 노드는 K8S 클러스터 8대 (master x 3, ctrl x 3, com x 2), 클러스터에 포함되지 않은 deployer 노드 1대로 구성했습니다.

## Node Port
NodePort는 이름처럼 클러스터에 속한 노드의 IP와 지정된 port를 통해 클러스터 위에 배포한 응용 서비스에 접근할 수 있게 해 주는 `서비스 리소스`의 한 종류입니다. 직접 사용해 보는 것이 가장 이해가 빠르겠죠? 그러기 위해서 먼저 아래처럼 OpenStack Helm 차트를 이용해 rabbitmq, memcached, mariadb를 초간단 모드로 배포하고, 마지막으로 keystone도 배포합니다. (TACO 설치에 대해서는 이전 포스트를 참고하시기 바랍니다.)
```
$ helm install openstack-helm/rabbitmq \
  --namespace=openstack \
  --name=rabbitmq \
  --set volume.enabled=false \
  --set pod.replicas.server=1

$ helm install openstack-helm/memcached \
  --namespace=openstack \
  --name=memcached

$ helm install openstack-helm/mariadb \
  --namespace=openstack \
  --name=mariadb \
  --set volume.enabled=false

$ helm install openstack-helm/keystone \
  --namespace=openstack \
  --name=keystone \
  --set network.api.ingress.public=false \
  --set pod.replicas.api=3
```
배포가 완료된 후 pod 목록을 조회해 보면 `keystone-api pod`가 3대의 컨트롤러 노드에 하나씩 생성되었고, pod마다 IP를 하나씩 할당 받은 것을 확인할 수 있습니다. 이 pod IP는 클러스터 내에서는 어디서든 접근이 가능합니다. 
```
[centos@hyunsun-deployer ~]$ kubectl get po -n openstack -o wide
NAME                                           READY     STATUS    RESTARTS   AGE       IP               NODE
keystone-api-7d9759db58-jxwzs                  1/1       Running   0          1m        10.233.78.154    ctrl-2
keystone-api-7d9759db58-l6vvx                  1/1       Running   0          1m        10.233.108.235   ctrl-1
keystone-api-7d9759db58-xs6qh                  1/1       Running   0          1m        10.233.109.149   ctrl-3

[centos@ctrl-1 ~]$ curl -I http://10.233.78.154/v3
HTTP/1.1 200 OK
```
다음으로 서비스 목록을 조회해 보면 `keystone-api`라는 이름을 가진 `ClusterIP` 타입의 서비스가 생성된 것을 확인할 수 있습니다. ClusterIP는 이름처럼 클러스터 내에서만 접근 가능한 IP로, keystone-api pod들을 대표하는 IP라고 생각하면 됩니다. 즉, 클러스터에 속한 한 노드에서 `<cluster IP>:80` 혹은 `<cluster IP>:35357`로 요청을 보내면 3개의 keystone-api pod 중 하나로 전달됩니다.
```
[centos@hyunsun-deployer ~]$ kubectl get svc -n openstack
NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
keystone-api                  ClusterIP   10.233.36.178   <none>        80/TCP,35357/TCP               10m

[centos@ctrl-1 ~]$ curl -I http://10.233.11.46/v3
HTTP/1.1 200 OK
```
ClusterIP는 뜯어보면 `NAT`로 구현되어 있습니다. 클러스터 내에서 `10.233.36.178:80`를 목적지로 패킷을 보내면, 이 패킷은 `OUTPUT`, `KUBE-SERVICES`, `KUBE-SVC-BXJIHEYR7GKHDJOX` 체인을 거치면서 최종적으로는 3개의 keystone-api pod 중 `<랜덤하게 선택된 pod의 IP>:80`으로 DNAT 됩니다. `KUBE-SVC-BXJIHEYR7GKHDJOX` 체인은 NodePort에서도 사용되니 눈여겨 보시기 바랍니다. 참고로, 아래는 실제 iptables 룰 조회 결과의 일부만 발췌한 것입니다.
```
[centos@ctrl-1 ~]$ sudo iptables -t nat -L OUTPUT
Chain OUTPUT (policy ACCEPT)
target         prot opt source               destination
KUBE-SERVICES  all  --  anywhere             anywhere     /* kubernetes service portals */

[centos@ctrl-1 ~]$ sudo iptables -t nat -L KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
target                     prot opt source   destination
KUBE-SVC-BXJIHEYR7GKHDJOX  tcp  --  anywhere 10.233.36.178 /* openstack/keystone-api:ks-pub cluster IP */ tcp dpt:http
KUBE-SVC-AS5YV7NDYZAGVTDH  tcp  --  anywhere 10.233.36.178 /* openstack/keystone-api:ks-adm cluster IP */ tcp dpt:35357

[centos@ctrl-1 ~]$ sudo iptables -t nat -L KUBE-SVC-BXJIHEYR7GKHDJOX
Chain KUBE-SVC-BXJIHEYR7GKHDJOX (1 references)
target                     prot opt source   destination
KUBE-SEP-ZQRJ43QLNKNAZLRB  all  --  anywhere anywhere     /* openstack/keystone-api:ks-pub */ statistic mode random probability 0.33332999982
KUBE-SEP-KDJJX6BHT75K2QRU  all  --  anywhere anywhere     /* openstack/keystone-api:ks-pub */ statistic mode random probability 0.50000000000
KUBE-SEP-L7QHG36UK5BTPDQX  all  --  anywhere anywhere     /* openstack/keystone-api:ks-pub */

[centos@ctrl-1 ~]$ sudo iptables -t nat -L KUBE-SEP-ZQRJ43QLNKNAZLRB
Chain KUBE-SEP-ZQRJ43QLNKNAZLRB (1 references)
target                     prot opt source   destination
DNAT                       tcp  --  anywhere anywhere     /* openstack/keystone-api:ks-pub */ tcp to:10.233.108.235:80

[centos@ctrl-1 ~]$ sudo iptables -t nat -L KUBE-SEP-KDJJX6BHT75K2QRU
Chain KUBE-SEP-KDJJX6BHT75K2QRU (1 references)
target                     prot opt source   destination
DNAT                       tcp  --  anywhere anywhere    /* openstack/keystone-api:ks-pub */ tcp to:10.233.109.149:80

[centos@ctrl-1 ~]$ sudo iptables -t nat -L KUBE-SEP-L7QHG36UK5BTPDQX
Chain KUBE-SEP-L7QHG36UK5BTPDQX (1 references)
target                     prot opt source   destination
DNAT                       tcp  --  anywhere anywhere    /* openstack/keystone-api:ks-pub */ tcp to:10.233.78.154:80
```
참고로 노드의 iptables 상태를 업데이트 해 주는 것은 `kube-proxy`의 역할입니다. kube-proxy는 모든 노드에 실행되며 클러스터 내의 리소스, 특히 서비스 리소스의 생성, 업데이트, 삭제에 반응해 위와 같은 netfilter 룰을 추가 혹은 삭제합니다.
```
[centos@hyunsun-deployer ~]$ kubectl logs kube-proxy-ctrl-1 -n kube-system
(생략)
I0709 13:11:59.984818       7 service.go:310]  Adding new service port "openstack/keystone-api:ks-pub" at 10.233.36.178:80/TCP
I0709 13:11:59.984870       7 service.go:310]  Adding new service port "openstack/keystone-api:ks-adm" at 10.233.36.178:35357/TCP
I0710 02:16:38.699216       7 service.go:312]  Updating existing service port "openstack/keystone-api:ks-pub" at 10.233.36.178:80/TCP
I0710 02:16:38.699301       7 service.go:312]  Updating existing service port "openstack/keystone-api:ks-adm" at 10.233.36.178:35357/TCP
I0710 02:16:38.741055       7 proxier.go:1372] Opened local port "nodePort for openstack/keystone-api:ks-pub" (:30500/tcp)
I0710 02:16:38.741301       7 proxier.go:1372] Opened local port "nodePort for openstack/keystone-api:ks-adm" (:31302/tcp)
```
다시 본론으로 돌아와서, 어쨌든 ClusterIP는 클러스터 밖에서는 접근할 방법이 없기 때문에 외부에 TACO를 서비스 할 때는 사용이 불가능합니다. 그럼 이제 NodePort를 한번 사용해 볼까요? 우선 NodePort를 사용하도록 keystone을 업그레이드 하겠습니다.
```
$ helm upgrade keystone openstack-helm/keystone \
  --namespace=openstack \
  --set network.api.ingress.public=false \
  --set pod.replicas.api=3 \
  --set network.api.node_port.enabled=true
```
업그레이드가 완료된 후 서비스 목록을 다시 조회해 보면 keystone-api 서비스의 타입이 NodePort로 바뀌었고, `PORT(S)` 아래의 값 역시 80 포트는 `30500`, 35357 포트는 `31302` 포트에 각각 매핑이 된 것으로 보입니다. NodePort가 성공적으로 할당 된 것 같네요. 이제 클러스터 밖에서 `<Node IP>:30500` 혹은 `<Node IP>:31302`로 keystone-api에 접근할 수 있습니다. 
```
[centos@hyunsun-deployer ~]$ kubectl get svc -n openstack
NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
keystone-api                  NodePort    10.233.36.178   <none>        80:30500/TCP,35357:31302/TCP   13h

[centos@hyunsun-deployer ~]$ curl -I http://ctrl-1:30500/v3
HTTP/1.1 200 OK

[centos@hyunsun-deployer ~]$ curl -I http://ctrl-1:31302/v3
HTTP/1.1 200 OK
```
NodePort 역시 NAT로 구현되어 있습니다. 관련된 iptables 룰을 한번 살펴보겠습니다. 클러스터 밖에서 `<Node IP>:30500`으로 패킷을 보내면 이번에는 `KUBE-SERVICES` 체인의 마지막 룰에 해당하는 `KUBE-NODEPORTS` 체인을 타게 됩니다. `KUBE-NODEPORTS` 체인의 룰을 살펴보면, 목적지의 포트가 30500인 경우 앞서 ClusterIP에서 보았던 `KUBE-SVC-BXJIHEYR7GKHDJOX` 체인으로 이어지는 것을 확인할 수 있습니다. 즉, 3개의 kube-api pod 중 `<랜덤하게 선택된 pod의 IP>:80`으로 목적지의 주소가 바뀌는 것이죠.
```
[centos@ctrl-1 ~]$ sudo iptables -t nat -L PREROUTING
Chain PREROUTING (policy ACCEPT)
target                     prot opt source      destination
KUBE-SERVICES              all  --  anywhere    anywhere             /* kubernetes service portals */

[centos@ctrl-1 ~]$ sudo iptables -t nat -L KUBE-SERVICES
Chain KUBE-SERVICES (2 references)
target                     prot opt source      destination
KUBE-SVC-BXJIHEYR7GKHDJOX  tcp  --  anywhere    10.233.36.178        /* openstack/keystone-api:ks-pub cluster IP */ tcp dpt:http
KUBE-SVC-AS5YV7NDYZAGVTDH  tcp  --  anywhere    10.233.36.178        /* openstack/keystone-api:ks-adm cluster IP */ tcp dpt:3537
KUBE-NODEPORTS             all  --  anywhere    anywhere             /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL

[centos@ctrl-1 ~]$ sudo iptables -t nat -L KUBE-NODEPORTS
Chain KUBE-NODEPORTS (1 references)
target                     prot opt source      destination
KUBE-SVC-BXJIHEYR7GKHDJOX  tcp  --  anywhere    anywhere             /* openstack/keystone-api:ks-pub */ tcp dpt:30500
KUBE-SVC-AS5YV7NDYZAGVTDH  tcp  --  anywhere    anywhere             /* openstack/keystone-api:ks-adm */ tcp dpt:31302

[centos@ctrl-1 ~]$ sudo iptables -t nat -L KUBE-SVC-BXJIHEYR7GKHDJOX
Chain KUBE-SVC-BXJIHEYR7GKHDJOX (1 references)
target                     prot opt source     destination
KUBE-SEP-ZQRJ43QLNKNAZLRB  all  --  anywhere   anywhere             /* openstack/keystone-api:ks-pub */ statistic mode random probability 0.33332999982
KUBE-SEP-KDJJX6BHT75K2QRU  all  --  anywhere   anywhere             /* openstack/keystone-api:ks-pub */ statistic mode random probability 0.50000000000
KUBE-SEP-L7QHG36UK5BTPDQX  all  --  anywhere   anywhere             /* openstack/keystone-api:ks-pub */
```
그러면 과연 NodePort로 TACO를 외부에 서비스 할 수 있을까요? 그러기 위해서는 몇 가지 고려해야 할 점이 남아 있습니다. 단순히 특정 노드에 퍼블릭 IP를 할당한 다음 NodePort로 외부에 서비스를 한다면, 그 노드에 장애가 발생하는 순간부터 서비스가 단절될 것입니다. 때문에 노드와 서비스의 상태를 모니터링해서 외부 사용자의 트래픽을 정상 노드로 포워딩 해 줄 `로드 발란서`가 앞 단에 필요합니다.

![](https://raw.githubusercontent.com/sktelecom-oslab/Virtualization-Software-Lab/master/assets/img/july/july-img-01.jpeg)

그런데 이 경우 로드 발란서가 외부로부터 유입되는 트래픽을 매우 세련된 방식으로 분산시켜 줄 것이기 때문에 노드에서 다시 3개의 keystone-api pod 중 하나를 랜덤하게 선택하는 기초적인 로드 분산 작업은 불필요해 보입니다. 예를 들어, 로드 발란서에 의해 1차적으로 `ctrl-1:30500`으로 전달 된 패킷의 2/3 가량은 다시 ctrl-2나 ctrl-3에 있는 keystone-api pod로 포워딩될 것입니다. 로컬에 keystone-api pod가 있는데도 불구하고 말이죠. 이런 불필요한 홉을 제거하기 위해서 NodePort로 들어온 패킷에 대해 무조건 로컬에 실행 중인 pod로 포워딩하는 `external_policy_local` 옵션을 활성해 해 주는 것이 효과적입니다.
```
$ helm upgrade keystone openstack-helm/keystone \
  --namespace=openstack \
  --set network.api.ingress.public=false \
  --set pod.replicas.api=3 \
  --set network.api.node_port.enabled=true \
  --set network.api.external_policy_local=true
```
다시 업데이트 된 iptables를 한번 확인해 보겠습니다. `KUBE-NODEPORTS` 체인의 룰이 달라진 것에 주목해 주시기 바랍니다. 목적지 포트가 `30500`인 경우 기존의 3개의 keystone-api pod 중 하나를 선택하는 체인으로 보내는 것이 아니라 로컬에 존재하는 keystone-api pod로 보내는 새로운 체인으로 보냅니다.
```
[centos@ctrl-1 ~]$ sudo iptables -t nat -L KUBE-NODEPORTS
Chain KUBE-NODEPORTS (1 references)
target                     prot opt source          destination
KUBE-XLB-BXJIHEYR7GKHDJOX  tcp  --  anywhere        anywhere             /* openstack/keystone-api:ks-pub */ tcp dpt:30500
KUBE-XLB-AS5YV7NDYZAGVTDH  tcp  --  anywhere        anywhere             /* openstack/keystone-api:ks-adm */ tcp dpt:31302

[centos@ctrl-1 ~]$ sudo iptables -t nat -L KUBE-XLB-BXJIHEYR7GKHDJOX
Chain KUBE-XLB-BXJIHEYR7GKHDJOX (1 references)
target                     prot opt source          destination
KUBE-SVC-BXJIHEYR7GKHDJOX  all  --  10.233.64.0/18  anywhere             /* Redirect pods trying to reach external loadbalancer VIP to clusterIP */
KUBE-SEP-ZQRJ43QLNKNAZLRB  all  --  anywhere        anywhere             /* Balancing rule 0 for openstack/keystone-api:ks-pub */

[centos@ctrl-1 ~]$ sudo iptables -t nat -L KUBE-SEP-ZQRJ43QLNKNAZLRB
Chain KUBE-SEP-ZQRJ43QLNKNAZLRB (2 references)
target                     prot opt source          destination    
DNAT                       tcp  --  anywhere        anywhere             /* openstack/keystone-api:ks-pub */ tcp to:10.233.108.235:80

[centos@ctrl-2 ~]$ sudo iptables -t nat -L KUBE-XLB-BXJIHEYR7GKHDJOX
Chain KUBE-XLB-BXJIHEYR7GKHDJOX (1 references)
target                     prot opt source          destination
KUBE-SVC-BXJIHEYR7GKHDJOX  all  --  10.233.64.0/18  anywhere             /* Redirect pods trying to reach external loadbalancer VIP to clusterIP */
KUBE-SEP-L7QHG36UK5BTPDQX  all  --  anywhere        anywhere             /* Balancing rule 0 for openstack/keystone-api:ks-pub */

[centos@ctrl-2 ~]$ sudo iptables -t nat -L KUBE-SEP-L7QHG36UK5BTPDQX
Chain KUBE-SEP-L7QHG36UK5BTPDQX (2 references)
target                     prot opt source          destination
DNAT                       tcp  --  anywhere        anywhere             /* openstack/keystone-api:ks-pub */ tcp to:10.233.78.154:80

[centos@ctrl-3 ~]$ sudo iptables -t nat -L KUBE-XLB-BXJIHEYR7GKHDJOX
Chain KUBE-XLB-BXJIHEYR7GKHDJOX (1 references)
target                     prot opt source          destination
KUBE-SVC-BXJIHEYR7GKHDJOX  all  --  10.233.64.0/18  anywhere             /* Redirect pods trying to reach external loadbalancer VIP to clusterIP */
KUBE-SEP-KDJJX6BHT75K2QRU  all  --  anywhere        anywhere             /* Balancing rule 0 for openstack/keystone-api:ks-pub */

[centos@ctrl-3 ~]$ sudo iptables -t nat -L KUBE-SEP-KDJJX6BHT75K2QRU
Chain KUBE-SEP-KDJJX6BHT75K2QRU (2 references)
target                     prot opt source          destination
KUBE-MARK-MASQ             all  --  10.233.109.149  anywhere             /* openstack/keystone-api:ks-pub */
DNAT                       tcp  --  anywhere        anywhere             /* openstack/keystone-api:ks-pub */ tcp to:10.233.109.149:80
```
주의할 점은 keystone-api pod가 떠 있지 않은 노드를 통해서는 NodePort로 접근이 불가능해 집니다. 따라서 로드 발란서에서는 백엔드 서버로 keystone-api pod가 떠 있는 컨트롤러 노드만을 지정해야 합니다. 아래는 keystone-api가 떠 있지 않은 컴퓨트 노드의 IP를 iptables 룰입니다. 외부에서 NodePort로 유입된 패킷은 DROP 시켜버립니다.
```
[centos@com-1 ~]$ sudo iptables -t nat -L KUBE-NODEPORTS
Chain KUBE-NODEPORTS (1 references)
target                     prot opt source               destination
KUBE-XLB-AS5YV7NDYZAGVTDH  tcp  --  anywhere             anywhere        /* openstack/keystone-api:ks-adm */ tcp dpt:31302
KUBE-XLB-BXJIHEYR7GKHDJOX  tcp  --  anywhere             anywhere        /* openstack/keystone-api:ks-pub */ tcp dpt:30500

[centos@com-1 ~]$ sudo iptables -t nat -L KUBE-XLB-BXJIHEYR7GKHDJOX
Chain KUBE-XLB-BXJIHEYR7GKHDJOX (1 references)
target                     prot opt source               destination
KUBE-SVC-BXJIHEYR7GKHDJOX  all  --  10.233.64.0/18       anywhere        /* Redirect pods trying to reach external loadbalancer VIP to clusterIP */
KUBE-MARK-DROP             all  --  anywhere             anywhere        /* openstack/keystone-api:ks-pub has no local endpoints */
```
지금까지 NodePort의 동작 원리를 살펴보고 NodePort를 이용해 외부에 TACO를 서비스 할 때 고려해야 할 점에 대해서도 살펴 봤습니다. NodePort는 설정이 간단하기는 하지만, 클러스터 밖에 존재하는 로드 발란서의 도움이 필요하다는 점에 유의해서 사용해야 한다는 점 잊지 마세요.
## Ingress 
Ingress는 외부에서 클러스터로 유입되는 트래픽을 일차적으로 처리해 적절한 서버로 proxy하는 K8S 리소스 타입으로 로드 밸런싱, SSL, 네임 기반의 호스팅을 기능도 제공합니다. Ingress를 사용하기 위해서는 먼저 Ingress 컨트롤러가 필요합니다. 아래처럼 OpenStack Helm 차트를 이용해 배포해 보겠습니다. 참고로 OpenStack Helm에 포함된 Ingress 차트는 `nginx` 기반의 Ingress 컨트롤러를 사용하고 있습니다. 클러스터 밖에서 노드 IP로 nginx에 접근하기 위해서는 `host_namespace` 설정을 `true`로 지정해 줘야 합니다. 
```
$ helm install openstack-helm/ingress \
  --namespace=openstack \
  --name=ingress \
  --set monitoring.prometheus.enabled=false \
  --set network.host_namespace=true
```
배포가 끝난 후 pod 목록을 조회해 보면 Ingress pod가 컨트롤러 노드 중 하나에 생성된 것을 볼 수 있습니다. 해당 노드에 접속해서 `docker top`으로 Ingress 컨테이너에서 실행된 프로세스 목록을 한 번 확인해 보겠습니다. `nginx-ingress-controller`와 `nginx` 프로세스들이 보이네요. (`dumb-init`은 `TERM` 시그널을 차일드 프로세스들에 포워딩하는 역할을 하는 단순한 init process로, pod 삭제 시 좀비 프로세스가 남지 않게 하기 위해 실행하는 것입니다.) `nginx-ingress-controller`는 클러스터 내의 Ingress 리소스 생성, 업데이트, 삭제를 모니터링하고 있다가 필요 시 nginx 설정 파일을 업데이트한 다음 nginx를 리로드하는 역할을 하고, `nginx`는 nginx-ingress-controller가 만들어 준 설정에 따라 외부에서 유입된 트래픽을 처리하는 역할을 합니다. `netstat`으로 확인해 보면 실제로 nginx 프로세스가 로컬의 모든 IP(0.0.0.0)에 대해 80 포트와 443 포트를 listen하고 있군요. `ingress-error-pages` pod는 존재하지 않는 리소스에 대한 요청이 들어왔을 때 에러 메시지를 응답해 줄 default backend 서비스입니다. 
```
[centos@hyunsun-deployer ~]$ kubectl get po -n openstack -o wide
NAME                                           READY     STATUS    RESTARTS   AGE       IP               NODE
ingress-error-pages-7cb55fcbdd-ndv2h           1/1       Running   0          2m        10.233.78.182    ctrl-2
ingress-pqqmw                                  1/1       Running   0          2m        10.10.10.18      ctrl-1

[centos@ctrl-1 ~]$ sudo docker top [ingress-container-docker-id]
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                16116               16097               0                   Jul10               ?                   00:00:00            /usr/bin/dumb-init /nginx-ingress-controller --watch-namespace openstack --http-port=80 --https-port=443 --election-id=ingress --ingress-class=nginx --default-backend-service=openstack/ingress-error-pages --configmap=openstack/ingress-conf --tcp-services-configmap=openstack/ingress-services-tcp --udp-services-configmap=openstack/ingress-services-udp
root                16129               16116               0                   Jul10               ?                   00:02:17            /nginx-ingress-controller --watch-namespace openstack --http-port=80 --https-port=443 --election-id=ingress --ingress-class=nginx --default-backend-service=openstack/ingress-error-pages --configmap=openstack/ingress-conf --tcp-services-configmap=openstack/ingress-services-tcp --udp-services-configmap=openstack/ingress-services-udp
root                16156               16129               0                   Jul10               ?                   00:00:00            nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
nfsnobo+            8574                16156               0                   Jul10               ?                   00:00:01            nginx: worker process

[centos@ctrl-1 ~]$ sudo netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      8574/nginx: worker
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      8574/nginx: worker
```
Ingress 컨트롤러를 배포했으니, 이제 기존에 배포했던 keystone을 Ingress를 사용하는 설정으로 재배포하겠습니다.
```
$ helm delete --purge keystone && helm install openstack-helm/keystone \
  --namespace=openstack \
  --name=keystone \
  --set network.api.ingress.public=true \
  --set pod.replicas.api=3
```
Ingress 리소스 목록을 한번 조회 해 보겠습니다. Ingress 리소스는 ingress 컨트롤러가 서비스를 제공할 대상에 해당합니다. `keystone` 그리고 `openstack-ingress`라는 이름의 Ingress가 생성되었네요. openstack-ingress는 마지막에 멀티 Ingress 부분에서 설명하기로 하고, 먼저 keystone을 한번 살펴보겠습니다. 상세 내용을 보면 호스트명이 `keystone`, `keystone.openstack`, 또는 `keystone.openstack.svc.cluster.local`로 들어오는 모든 요청을 `keystone-api` 서비스의 80 포트(ks-pub)로 보내라는 룰이 설정되어 있습니다. (참고로, Ingress 룰의 백엔드는 K8S 서비스 리소스여야 합니다.) `keystone-api` 서비스는 앞서 살펴 봤던 ClusterIP 타입의 서비스입니다, 기억 나시죠? 이 룰들이 그대로 nginx의 설정이 됩니다.
```
[centos@hyunsun-deployer openstack-helm]$ kubectl get ingress -n openstack
NAME                  HOSTS                                                                                               ADDRESS   PORTS     AGE
keystone              keystone,keystone.openstack,keystone.openstack.svc.cluster.local                                              80        30m
openstack-ingress     *.openstack.svc.cluster.local

[centos@hyunsun-deployer ~]$ kubectl describe ingress -n openstack keystone
Name:             keystone
Namespace:        openstack
Address:
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host                                  Path  Backends
  ----                                  ----  --------
  keystone                              /     keystone-api:ks-pub (<none>)
  keystone.openstack                    /     keystone-api:ks-pub (<none>)
  keystone.openstack.svc.cluster.local  /     keystone-api:ks-pub (<none>)
Annotations:
Events:  <none>

[centos@hyunsun-deployer ~]$ kubectl get svc -n openstack
NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
keystone-api                  ClusterIP   10.233.17.27    <none>        80/TCP,35357/TCP               18h

[centos@hyunsun-deployer ~]$ kubectl -n openstack exec ingress-8kvrd cat /etc/nginx/nginx.conf
(관심 부분만 발췌함)
    upstream openstack-keystone-api-ks-pub {
        # Load balance algorithm; empty for round robin, which is the default
        least_conn;
        keepalive 32;

        server 10.233.78.184:80 max_fails=0 fail_timeout=0;
        server 10.233.109.163:80 max_fails=0 fail_timeout=0;
        server 10.233.108.219:80 max_fails=0 fail_timeout=0;
    }

    ## start server keystone
    server {
        server_name keystone ;
        (생략)
    ## end server keystone

    ## start server keystone.openstack
    server {
        server_name keystone.openstack ;
        (생략)
    ## end server keystone.openstack

    ## start server keystone.openstack.svc.cluster.local
    server {
        server_name keystone.openstack.svc.cluster.local ;
        (생략)
    ## end server keystone.openstack.svc.cluster.local
```
추가로 horizon도 배포해 보겠습니다. horizon이라는 이름의 ingress 리소스가 생겼고, nginx 설정도 horizon 서비스를 포함하게끔 업데이트 된 것을 확인할 수 있습니다.
```
$ helm install openstack-helm/horizon \
  --namespace=openstack \
  --name=horizon \
  --set network.dashboard.ingress.public=true

[centos@hyunsun-deployer openstack-helm]$ kubectl get ingress -n openstack
NAME                  HOSTS                                                                                               ADDRESS   PORTS     AGE
horizon               horizon,horizon.openstack,horizon.openstack.svc.cluster.local                                                 80        1m
keystone              keystone,keystone.openstack,keystone.openstack.svc.cluster.local                                              80        30m
openstack-ingress     *.openstack.svc.cluster.local

[centos@hyunsun-deployer ~]$ kubectl describe ingress -n openstack horizon
Name:             horizon
Namespace:        openstack
Address:
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host                                 Path  Backends
  ----                                 ----  --------
  horizon                              /     horizon-int:web (<none>)
  horizon.openstack                    /     horizon-int:web (<none>)
  horizon.openstack.svc.cluster.local  /     horizon-int:web (<none>)
Annotations:
Events:  <none>

[centos@hyunsun-deployer ~]$ kubectl -n openstack exec ingress-8kvrd cat /etc/nginx/nginx.conf
(관심 부분만 발췌함)
    upstream openstack-horizon-int-web {
        # Load balance algorithm; empty for round robin, which is the default
        least_conn;
        keepalive 32;

        server 10.233.108.254:80 max_fails=0 fail_timeout=0;
    }
    ## start server horizon
    server {
        server_name horizon ;
    ## end server horizon

    ## start server horizon.openstack
    server {
        server_name horizon.openstack ;
    ## end server horizon.openstack

    ## start server horizon.openstack.svc.cluster.local
    server {
        server_name horizon.openstack.svc.cluster.local ;
    ## end server horizon.openstack.svc.cluster.local
```
이제 Ingress가 잘 동작하는지 한번 확인해 보겠습니다. NodePort 시험과 비슷하게 클러스터 밖에서 Ingress pod가 실행 중인 컨트롤러 노드의 80 포트로 요청을 보내 보면 됩니다. 단, 헤더에 원하는 서비스의 호스트명을 꼭 포함시켜 줘야 nginx에서 올바른 서버로 요청을 전달할 수 있습니다.
```
[centos@hyunsun-deployer ~]$ curl -I http://ctrl-1/v3 -H "Host: keystone"
HTTP/1.1 200 OK

[centos@hyunsun-deployer ~]$ curl -I http://ctrl-1/auth/login/?next=/ -H "Host: horizon"
HTTP/1.1 200 OK
```
잘 동작하는 군요. 그럼 Ingress로 외부에 TACO 서비스가 가능할까요? 우선, NodePort와 같은 이유로 노드 IP로 서비스하는 것은 위험합니다. 그렇다고 Ingress 앞단에 중복으로 또 로드 발란서를 둘 순 없는 노릇입니다. 이를 해결하기 위한 방법으로 OpenStack Helm의 Ingress 차트에서는 Ingress 컨트롤러 pod에 VIP 제공 기능을 내장시켰습니다. 한 번, 사용해 볼까요? 
```
$ helm upgrade ingress openstack-helm/ingress \
  --namespace=openstack \
  --set monitoring.prometheus.enabled=false \
  --set network.host_namespace=true \
  --set network.vip.manage=true \
  --set network.vip.interface=eth0 \
  --set network.vip.addr=10.10.10.254/32
```
업그레이드가 끝난 후 Ingress pod가 생성된 노드에서 VIP 인터페이스로 지정한 인터페이스의 IP를 조회해 보면 VIP가 추가된 것을 볼 수 있습니다. nginx의 바인드 주소 또한 0.0.0.0에서 VIP로 바뀌어 보안상 좀 더 나아 보입니다. 클러스터 밖에서 VIP로 keystone 서비스에 접근하는 것도 잘 되는 군요. 
```
[centos@hyunsun-deployer ~]$ kubectl get po -n openstack -o wide
NAME                                           READY     STATUS    RESTARTS   AGE       IP               NODE
ingress-8585b5b86b-rmblg                       2/2       Running   0          2m        10.10.10.20      ctrl-3
ingress-error-pages-7cb55fcbdd-k8tbj           1/1       Running   0          2m        10.233.78.133    ctrl-2

[centos@ctrl-3 ~]$ sudo ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:d1:bb:22 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.20/24 brd 10.10.10.255 scope global dynamic eth0
       valid_lft 56782sec preferred_lft 56782sec
    inet 10.10.10.254/32 scope global eth0
       valid_lft forever preferred_lft forever

[centos@ctrl-3 ~]$ sudo netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 10.10.10.254:80         0.0.0.0:*               LISTEN      32103/nginx: master
tcp        0      0 10.10.10.254:443        0.0.0.0:*               LISTEN      32103/nginx: master

[centos@hyunsun-deployer ~]$ curl -I http://10.10.10.254/v3 -H "Host: keystone"
HTTP/1.1 200 OK
```
이번엔 VIP가 잘 이동하는지 시험해 보기 위해서 ctrl-3에 있던 Ingress pod를 삭제해 다른 노드에 새로 생성되게 했습니다. ctrl-1에 생성 됐군요. VIP도 ctrl-3의 VIP 인터페이스에서 ctrl-1의 VIP 인터페이스로 잘 옮겨갔습니다. 어떻게 구현 된 걸까요? Ingress pod의 상세 정보를 보면 답을 알 수 있습니다. Ingress pod는 `ingress-vip-init`이라는 이름의 특별한 init 컨테이너를 포함하고 있는데요, 이 컨테이너는 Ingress 컨트롤러 컨테이너가 생성되기 전에 먼저 VIP 인터페이스에 VIP를 할당하는 스크립트를 실행하는 역할을 합니다. 그리고 죽습니다. `Containers` 아래 부분을 잘 보면 Ingress pod는 `ingress`와 `ingress-vip`라는 두 개의 컨테이너로 이뤄져 있는 것을 알 수 있습니다. `ingress`는 지금껏 살펴 본 Ingress 컨트롤러구요, `ingress-vip` 컨테이너의 역할은 무한히 sleep하고 있다가 pod가 삭제 될 때 `preStop` 훅으로 VIP를 VIP 인터페이스에서 제거하는 스크립트를 실행하는 것입니다. 이렇게 해서 VIP가 Ingress pod를 따라 다닐 수 있게 된 것입니다. 알고 보면 간단하죠?
```
[centos@hyunsun-deployer ~]$ kubectl delete po -n openstack ingress-8585b5b86b-rmblg
[centos@hyunsun-deployer ~]$ kubectl get po -n openstack -o wide
NAME                                           READY     STATUS        RESTARTS   AGE       IP               NODE
ingress-8585b5b86b-2282p                       2/2       Running       0          26s       10.10.10.18      ctrl-1
ingress-8585b5b86b-rmblg                       0/2       Terminating   0          11m       10.10.10.20      ctrl-3
ingress-error-pages-7cb55fcbdd-k8tbj           1/1       Running       0          12m       10.233.78.133    ctrl-2

[centos@ctrl-3 ~]$ sudo ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:d1:bb:22 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.20/24 brd 10.10.10.255 scope global dynamic eth0
       valid_lft 56524sec preferred_lft 56524sec

[centos@ctrl-1 ~]$ sudo ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:6c:5f:9d brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.18/24 brd 10.10.10.255 scope global dynamic eth0
       valid_lft 72636sec preferred_lft 72636sec
    inet 10.10.10.254/32 scope global eth0
       valid_lft forever preferred_lft forever

[centos@hyunsun-deployer ~]$ curl -I http://10.10.10.254/v3 -H "Host: keystone"
HTTP/1.1 200 OK
```
그런데 pod에 문제가 생긴 것이 아니라 노드에 장애가 생긴 경우에는 어떨까요? 이 경우에도 Ingress pod가 다른 노드로 금방 옮겨 갈까요? 안타깝게도 그렇지 않습니다. K8S는 `--node-monitor-grace-period` 동안 상태 업데이트가 없는 경우에 노드가 다운됐다고 판단하고, 다시 `--pod-eviction-timeout` 동안 기다렸다가 장애가 난 노드에 존재하는 pod들을 정상적인 다른 노드로 이동시킵니다. 이 시간은 기본 설정으로 약 6분에 가깝기 때문에 서비스 장애로 이어질 수 있습니다. 그렇다고 이 시간을 짧게 단축시키면 다른 부작용이 발생할 수 있습니다. 때문에 이 VIP 기능을 제대로 사용하기 위해서는 `calico` CNI를 사용하는 경우 Ingress pod를 `DaemonSet`으로 하나 이상 띄운 다음 그림처럼 노드에서 BGP peering을 맺고 있는 앞 단의 라우터에게 BGP를 이용해 VIP 라우팅 정보를 제공하고 라우터에서 VIP에 대해 ECMP로 멀티 패스를 제공하는 것입니다. 이 때, VIP는 라우팅을 통해서만 접근이 가능해야 하므로 실제 물리 링크와는 연결되지 않은 dummy 인터페이스를 만들어 사용하게 됩니다.
```
$ helm upgrade ingress openstack-helm/ingress \
  --namespace=openstack \
  --set monitoring.prometheus.enabled=false \
  --set network.host_namespace=true \
  --set network.vip.manage=true \
  --set network.vip.interface=dummy-ingress-iface \
  --set network.vip.addr=10.10.10.254/32 \
  --set deployment.type=DaemonSet
```

![](https://raw.githubusercontent.com/sktelecom-oslab/Virtualization-Software-Lab/master/assets/img/july/july-img-02.jpeg)

위와 같은 라우팅 설정이 어려운 환경을 위해 `keepalived` 컨테이너를 Ingress pod에 추가해 VRRP로 VIP를 관리하는 패치가 리뷰 중이니 곧 사용 가능해 질 것으로 기대하고 있습니다.

![](https://raw.githubusercontent.com/sktelecom-oslab/Virtualization-Software-Lab/master/assets/img/july/july-img-03.jpeg)

마지막으로, 초반에 그냥 넘어갔던 `openstack-ingress`라는 이름의 Ingress 리소스를 살펴볼 차례입니다. 우리가 띄운 Ingress 컨트롤러의 실행 옵션을 자세히 보면 `--watch-namespace openstack --ingress-class=nginx` 설정이 포함되어 있습니다. 첫 번째 옵션은 openstack 네임스페이스에 생성되는 Ingress만 서비스하라는 뜻이고, 두 번째 옵션은 Ingress class가 nginx인 Ingress만 서비스하라는 뜻입니다. 다시 말해, 우리가 띄운 Ingress 컨트롤러는 openstack 네임스페이스에 생성된 Ingress class가 nginx인 Ingress만 처리하고 그 외의 Ingress에 대해서는 무시합니다.
```
[centos@ctrl-1 ~]$ sudo docker top [ingress-container-docker-id]
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                16129               16116               0                   Jul10               ?                   00:02:17            /nginx-ingress-controller --watch-namespace openstack --http-port=80 --https-port=443 --election-id=ingress --ingress-class=nginx --default-backend-service=openstack/ingress-error-pages --configmap=openstack/ingress-conf --tcp-services-configmap=openstack/ingress-services-tcp --udp-services-configmap=openstack/ingress-services-udp
```
앞서 시험했던 keystone이나 horizon Ingress는 확인해 보면 위 두 가지 조건을 모두 충족하는 것을 알 수 있습니다.
```
[centos@hyunsun-deployer ~]$ kubectl get ing -o yaml keystone -n openstack
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  namespace: openstack
(이하 생략)
```
그런데 `openstack-ingress`는 네임스페이스는 openstack이긴 하지만 Ingress class가 nginx가 아니라 `nginx-cluster` 군요. 때문에 우리가 띄운 Ingress 컨트롤러는 이 Ingress를 서비스하지 않습니다. 그럼 언제 사용하는 걸까요?
```
[centos@hyunsun-deployer ~]$ kubectl get ing -o yaml openstack-ingress -n openstack
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx-cluster
  namespace: openstack
spec:
  rules:
  - host: '*.openstack.svc.cluster.local'
    http:
      paths:
      - backend:
          serviceName: ingress
          servicePort: 80
        path: /
(이하 생략)
```
`openstack-ingress`는 아래 그림처럼 Ingress 컨트롤러를 계층적으로 구성할 때 오픈스택 서비스로 들어오는 트래픽(호스트가 `*.openstack.svc.cluster.local`)을 openstack 네임스페이스 전용 nginx로 보내기 위해 만들어 둔 것입니다. 이 경우 외부로부터 유입되는 트래픽을 일차적으로 수용하는 것은 가장 상위의 Ingress 컨트롤러가 제어하는 nginx기 때문에 VIP는 여기에 붙여주는 것이 좋겠죠. 오픈스택을 포함한 특정 네임스페이스 전용 Ingress 컨트롤러는 더 이상 외부에서 직접 노드 IP를 통해 접근할 필요가 없기 때문에 `hostNetwork` 설정 또한 비활성화 시켜줍니다. 이런 구성은 클러스터 내에 오픈스택 외에 다른 서비스도 혼재 할 때 유용해 보입니다.

![](https://raw.githubusercontent.com/sktelecom-oslab/Virtualization-Software-Lab/master/assets/img/july/july-img-04.jpeg)

간단히 테스트 해 보겠습니다. 기존에 설치했던 Ingress는 깔끔하게 삭제하고, 아래처럼 Ingress 컨트롤러를 `cluster` 모드로 하나 `namespace` 모드로 하나 총 두 개 띄우겠습니다. VIP에 대한 라우팅 설정이 불가능한 환경에서는 `deployment.type=DaemonSet` 부분을 빼고, `vip.interface`를 물리 인터페이스로 대체하여 시험하시기 바랍니다. 배포가 완료되면 아래처럼 클러스터 밖에서 VIP로 접근이 가능한지 확인해 봅니다. 이 때, 호스트 명은 기존처럼 `keystone`만 써서는 안 되고 `keystone.openstack.svc.cluster.local`로 써 줘야 `openstack-ingress` Ingress에서 설정한 룰에 매칭이 되어 openstack 네임스페이스 전용 nginx로 요청이 전달될 수 있습니다.
```
$ helm install openstack-helm/ingress \
  --namespace=kube-system \
  --name=ingress-cluster \
  --set monitoring.prometheus.enabled=false \
  --set network.host_namespace=true \
  --set network.vip.manage=true \
  --set network.vip.interface=dummy-ingress-iface \
  --set network.vip.addr=10.10.10.254/32 \
  --set deployment.type=DaemonSet \
  --set deployment.mode=cluster

$ helm install openstack-helm/ingress \
  --namespace=openstack \
  --name=ingress \
  --set monitoring.prometheus.enabled=false \
  --set network.host_namespace=false \
  --set deployment.mode=namespace

[centos@hyunsun-deployer ~]$ helm list
NAME               REVISION    UPDATED                     STATUS      CHART              NAMESPACE
ingress            1           Wed Jul 11 10:59:54 2018    DEPLOYED    ingress-0.1.0      openstack
ingress-cluster    1           Wed Jul 11 10:41:43 2018    DEPLOYED    ingress-0.1.0      kube-system

[centos@hyunsun-deployer ~]$ curl -I http://10.10.10.254/v3 -H "Host: keystone.openstack.svc.cluster.local"
HTTP/1.1 200 OK
```
지금까지 Ingress의 동작 원리와 Ingress를 활용해서 외부에 TACO를 서비스하는 방법을 살펴봤습니다. Ingress는 조금 복잡하게 느껴질 수 있지만, 별도의 로드 발란서가 필요 없다는 점과 네임 기반의 서비스, SSL 등 상용 서비스 시에 유용한 부가적인 기능들을 제공한다는 점에서 NodePort 보다는 좀 더 활용도가 높은 방법으로 여겨 집니다.
