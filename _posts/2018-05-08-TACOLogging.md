# TACO Logging
<p>by 임성일(<a href="mailto:si.im@sk.com">si.im@sk.com</a>)</p>
## Overview
우리는 로그를 사용하여 실행 중인 애플리케이션이나 시스템에의 내부 상황을 파악합니다. 특히, 로그의 사용은 문제를 디버깅하고 작업을 모니터링하는 데 유용합니다. 최신의 응용 소프트웨어들은 다양한 로깅 메커니즘을 제공하고 있으며 TACO의 기반 소프트웨어인 오픈소스들도 각각의 로깅 기능을 제공합니다. TACO의 기반 컨테이너 엔진인 docker-engine의 경우 가장 간단하면서 일반적으로 사용하는 logging방식은 표준출력과 표준에러를 사용하는 로깅 기능을 제공하고 있습니다. 차상위 시스템인 kubernetes 또한 그 나름의 로깅 기능을 제공합니다.

하지만 docker나 kubernetes에서 제공하는 로깅 기능으로는 TACO의 상황을 효율적으로 감시할 수 없습니다. TACO의 로그는 그 구성요소인 pod나 컨테이너등의 문제로 인해 재기동 되거나 삭제 되었을때에도 관련 로그를 유지하고 있어야 하며 이를 위해 별도의 저장소와 유지주기 등을 관리해야 합니다. 이러한 기능을 수행하는 것을 TACO Logging이라고 합니다.

본 포스트에서는 TACO Logging의 구조와 동작방식을 설명하여 독자들의 이해도를 높히고자 합니다. 이를 통해 TACO Logging을 사용자 편의에따라 다양하게 변경하여 프로젝트에 적용할 수 있도록 하고 사용자 고유의 필요한 로그를 수집하도록 하는 방법또한 설명합니다.
### logging in docker
TACO의 기반 컨테이너 엔진인 docker는 자체의 로깅 기능을 제공하고 있습니다. 로깅 드라이버(logging driver)를 사용하여 표준출력과 표준에러에 대한 처리를 수행합니다. 기본설정에서 docker는 표준출력과 표준에러애서 발생하는 모든 로그를 json 형태로 변환하여 각 컨테이너별 로그파일로 저장합니다. 로깅 드라이버는 컨테이너마다 지정할 수 있으며 이에 대한 설정등은 다음의 페이지를 참조할 수 있습니다.
* 로깅 드라이버 설정 (https://docs.docker.com/config/containers/logging/configure/)
* 드라이버에 plugin 적용 (https://docs.docker.com/config/containers/logging/plugins/)
* output 조절하기 (https://docs.docker.com/config/containers/logging/log_tags/)
### logging in kubernetes
Kubernetes에서는 App과 system의 log를 제공하여 클러스터 내부를 사용자가 이해할수 있도록 돕습니다. 간단하게 다음과 같은 명령으로 클러스터내 존재하는 POD (컨테이너)에 대한 로그를 얻어올 수 있습니다.
```bash
kubectl logs -n [namespace] [pod_name]
```
```bash 
taco@master:~$ kubectl logs elasticsearch-data-2
+ COMMAND=start
+ start
+ ulimit -l unlimited
+ exec /docker-entrypoint.sh elasticsearch
[2018-04-13T05:07:27,992][INFO ][o.e.n.Node ] [elasticsearch-data-2] initializing ...
[2018-04-13T05:07:28,193][INFO ][o.e.e.NodeEnvironment ] [elasticsearch-data-2] using [1] data paths, mounts [[/usr/share/elasticsearch/data (/dev/rbd4)]], net usable_space [933.2gb], net total_space [984.1gb], spins? [no], types [ext4]
[2018-04-13T05:07:28,194][INFO ][o.e.e.NodeEnvironment ] [elasticsearch-data-2] heap size [1.9gb], compressed ordinary object pointers [true]
[2018-04-13T05:07:28,298][INFO ][o.e.n.Node ] [elasticsearch-data-2] node name [elasticsearch-data-2], node ID [-B45XC7NR5qqvab3zfqgdg]
[2018-04-13T05:07:28,298][INFO ][o.e.n.Node ] [elasticsearch-data-2] version[5.6.4], pid[41], build[8bbedf5/2017-10-31T18:55:38.105Z], OS[Linux/4.4.0-116-generic/amd64], JVM[Oracle Corporation/OpenJDK 64-Bit Server VM/1.8.0_151/25.151-b12]
[2018-04-13T05:07:28,298][INFO ][o.e.n.Node ] [elasticsearch-data-2] JVM arguments [-Xms2g, -Xmx2g, -XX:+UseConcMarkSweepGC, -XX:CMSInitiatingOccupancyFraction=75, -XX:+UseCMSInitiatingOccupancyOnly, -XX:+AlwaysPreTouch, -Xss1m, -Djava.awt.headless=true, -Dfile.encoding=UTF-8, -Djna.nosys=true, -Djdk.io.permissionsUseCanonicalPath=true, -Dio.netty.noUnsafe=true, -Dio.netty.noKeySetOptimization=true, -Dio.netty.recycler.maxCapacityPerThread=0, -Dlog4j.shutdownHookEnabled=false, -Dlog4j2.disable.jmx=true, -Dlog4j.skipJansi=true, -XX:+HeapDumpOnOutOfMemoryError, -Xms2g, -Xmx2g, -Des.path.home=/usr/share/elasticsearch]
[2018-04-13T05:07:29,169][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] loaded module [aggs-matrix-stats]
[2018-04-13T05:07:29,170][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] loaded module [ingest-common]
[2018-04-13T05:07:29,170][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] loaded module [lang-expression]
[2018-04-13T05:07:29,170][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] loaded module [lang-groovy]
[2018-04-13T05:07:29,170][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] loaded module [lang-mustache]
[2018-04-13T05:07:29,170][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] loaded module [lang-painless]
[2018-04-13T05:07:29,170][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] loaded module [parent-join]
[2018-04-13T05:07:29,170][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] loaded module [percolator]
[2018-04-13T05:07:29,170][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] loaded module [reindex]
[2018-04-13T05:07:29,170][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] loaded module [transport-netty3]
[2018-04-13T05:07:29,171][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] loaded module [transport-netty4]
[2018-04-13T05:07:29,171][INFO ][o.e.p.PluginsService ] [elasticsearch-data-2] no plugins loaded
[2018-04-13T05:07:30,393][INFO ][o.e.d.DiscoveryModule ] [elasticsearch-data-2] using discovery type [zen]
[2018-04-13T05:07:31,121][INFO ][o.e.n.Node ] [elasticsearch-data-2] initialized
[2018-04-13T05:07:31,121][INFO ][o.e.n.Node ] [elasticsearch-data-2] starting ...
[2018-04-13T05:07:31,274][INFO ][o.e.t.TransportService ] [elasticsearch-data-2] publish_address {172.16.119.179:9300}, bound_addresses {[::]:9300}
[2018-04-13T05:07:31,283][INFO ][o.e.b.BootstrapChecks ] [elasticsearch-data-2] bound or publishing to a non-loopback or non-link-local address, enforcing bootstrap checks
```
![Logging at the node level](https://d33wubrfki0l68.cloudfront.net/59b1aae2adcfe4f06270b99a2789012ed64bec1f/4d0ad/images/docs/user-guide/logging/logging-node-level.png) 

그림출처: https://kubernetes.io

kubernetes는 노드수준에서 로그의 보존, 로테이션 기능을 제공합니다. 또한 POD의 타 노드로 이동(migraion)시 컨테이너 뿐 아니라 해당 로그를 함께 옮겨주는 기능도 수행하고 있다. 하지만, 공식적으로 클러스터 수준에서 로깅기능을 제공하는 솔루션은 없습니다.
## fluent-logging (Central logging on k8s)
### Cluster logging in k8s (reference architecture)
kubernetes에서는 공식적인 솔루션을 제시하지 않지만 다음과 같은 일반적인 방식을 고려할 수 있습니다.
* Use a node-level logging agent that runs on every node. (**fluent-logging approach**)
* Include a dedicated sidecar container for logging in an application pod.
* Push logs directly to a backend from within an application.

공식 페이지에서는 아래와 같이 클러스터수준의 아키텍쳐를 제시합니다. [https://kubernetes.io](https://kubernetes.io/docs/concepts/cluster-administration/logging/#cluster-level-logging-architectures)
| Using a node logging agent | Streaming sidecar container |
| :-------------: |:-------------:|
| ![Using a node logging agent ](https://d33wubrfki0l68.cloudfront.net/2585cf9757d316b9030cf36d6a4e6b8ea7eedf5a/1509f/images/docs/user-guide/logging/logging-with-node-agent.png ) | ![Streaming sidecar container ](https://d33wubrfki0l68.cloudfront.net/c51467e219320fdd46ab1acb40867b79a58d37af/b5414/images/docs/user-guide/logging/logging-with-streaming-sidecar.png ) |

| Sidecar container with a logging agent | Exposing logs directly from the application |
| :-------------: |:-------------:|
| ![Sidecar container with a logging agent ](https://d33wubrfki0l68.cloudfront.net/d55c404912a21223392e7d1a5a1741bda283f3df/c0397/images/docs/user-guide/logging/logging-with-sidecar-agent.png ) |![Exposing logs directly from the application](https://d33wubrfki0l68.cloudfront.net/0b4444914e56a3049a54c16b44f1a6619c0b198e/260e4/images/docs/user-guide/logging/logging-from-application.png) |
그림출처: https://kubernetes.io

### Architecture
앞에서 언급한 것처럼 kubernetes는 몇가지 클러스터 수준의 로깅기능을 제시하고 있다. 제시된 가지 모델 중 첫번째 방식(Using a node logging agent)를 제외한 나머지 방법은 해당 애플리케이션(POD) 내부에 해당 기능을 제공하거나 

### Agent Selection






