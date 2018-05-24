---
title: Open Infrastructure Project "Airship"
author: 안재석
author-email: jay.ahn@sk.com
description: SKT, AT&T-오픈스택 재단과 함께 손쉬운 클라우드 구축을 가속화 할 새로운 오픈 인프라 프로젝트 Airship 출범
---
# Open Infrastructure Project, Airship
For English Version, please visit [SKT Global Blog]. 
 
SKT는 AT&T등과 함께 새로운 Open Infrastructure Project인 "[Airship]"를 런칭하였습니다. 그리고, 앞으로 AT&T 및 다른 참여회사들, OpenStack Foundation과 협력하여 OpenStack의 Top-Level Project로 만들 수 있도록 노력할 계획입니다. 이번 블로그에서는 Airship 전체 프로젝트에 대해서 간략한 소개를 드리고, Airship에 포함된 여러개의 Sub-Project들 중에서 SKT가 참여하는 부분에 대해서 말씀드리도록 하겠습니다. 

![Companies participating Airship project]({{ site.baseurl }}{{ post.url }}/assets/img/airship/airship_companies_v2.png)
[![License: CC BY-NC-ND 4.0](https://licensebuttons.net/l/by-nc-nd/4.0/80x15.png)](https://creativecommons.org/licenses/by-nc-nd/4.0/)

## 프로젝트 개요   
SKT는 2017년 부터 ATT와의 협력을 통해서 컨테이너 기술과 오픈스택을 접목하여 클라우드 인프라를 쉽게 설치, 관리하기 위한 기술 개발을 해 왔으며, 같은 해 AT&T와 함께 [OpenStack] 산하에 [OpenStack-Helm] 프로젝트를 공식 런칭하였습니다. Airship은 OpenStack-Helm 프로젝트를 매개체로 하여 AT&T내에서 개발해오고 있던 다양한 오픈소스 기술들을 유기적으로 연결하여 만들어진 프로젝트로, 하드웨어 인프라 설치로부터 오픈스택 설치, 설정 변경 및 버전 업그레이드까지 클라우드 사이트 운영에 필요한 모든 단계를 관리할 수 있게 해 줍니다. 특히, 이 모든 작업은 완벽하게 컨테이너화된 클라우드 네이티브 플랫폼을 통해서 일관성 있게 수행됩니다. 

Airship을 사용하면 다양한 형상의 클라우드 사이트를 전보다 쉽게 구축 및 운영할 수 있습니다. 유/무선 통신 사업자, 제조사, IT회사, 개인 개발자 할 것 없이 모두가 예측 가능한 방법으로 클라우드 인프라를 만들고 관리할 수 있습니다. 

이는 Airship이 개발 방법의 주류로서 자리 잡아가고 있는 마이크로 서비스 기술과 클라우드 네이티브 원칙을 기반으로 만들어졌기 때문에 가능한 일입니다. Airship의 마이크로 서비스들은 클라우드를 만들고 관리하기 위해 필요한 특정 역할들을 각자 분담하여 성공적으로 수행하도록 만들어졌습니다. Airship의 궁극적인 목표는 클라우드 운영자들이 아무것도 없는 하드웨어들로부터 오픈스택 클라우드를 만들어 내고, 이를 문제없이 운영하기 위한 최상의 라이프 사이클 관리를 제공하는 것입니다. 

프로젝트 초기의 목표는 주로 [Kubernetes] 클러스터 위에 오픈스택을 설치하고, 그렇게 구축한 클라우드의 라이프 사이클을 규모와 신속성, 탄력성, 유연성, 특히 네트워크 클라우드에 필요한 운영상의 예측성을 가지고 관리할 수 있는 선언적인 플랫폼을 구현하는데 있습니다. 특히, "선언적"이라는 용어는 매우 큰 장점을 가지고 있는 단순한 개념입니다. 쉽게 설명하면, 구축할 클라우드의 모든 부분을 표준화된 Manifest에 정의함으로써 미세한 부분까지 유연하게 제어할 수 있게 해 주며, 선언적인 플랫폼은 이러한 Manifest에 정의된 내용이 실제로 실현 되고 상황이 변하더라도 일관성 있게 유지될 수 있도록 해줍니다. 

## AIRSHIP SUB-PROJECT 
Airship 프로젝트는 다음과 같은 서브 프로젝트들로 구성되어 있으며, SKT는 이 중에서 "Armada"의 개발에 참여하고 있습니다. 

- Armada - An orchestrator for deploying and upgrading a collection of Helm charts 
- Berth - A lightweight mechanism for managing VMs on top of Kubernetes via Helm
- Deckhand - A configuration management service with features to support managing large cluster configurations
- Diving Bell - A lightweight solution for bare metal configuration management
- Drydock - A declarative host provisioning system built initially to leverage MaaS for baremetal host deployment
- Pegleg - A tool to organize configuration of multiple Airship deployments
- Promenade - A deployment system for resilient, self-hosted Kubernetes
- Shipyard - A cluster lifecycle orchestrator for Airship

## SKT 참여 범위
위에서 말씀 드렸듯이, Airship의 서브 프로젝트들 중에서 SKT는 "Armada"의 개발에만 참여 하고 있습니다. 아래 그림과 같이 복잡한 마이크로 서비스 형태로 구성된 오픈스택 서비스들은 우선 OpenStack Kolla 혹은 LOCI 프로젝트를 통해서 컨테이너 이미지로 만들어집니다. 그리고, OpenStack-Helm 프로젝트를 기반으로 Kubernetes 상에 설치될 수 있는 형태로 패키징 되고, Armada는 이렇게 패키징 된 많은 서비스들을 위에서 설명한 것과 같이 선언적으로 관리하고 오케스트레이션 할 수 있도록 만들어 줍니다.

![Three Main Steps for OpenStack on Kubernetes]({{ site.baseurl }}{{ post.url }}/assets/img/airship/airship_sktflow.png)
[![License: CC BY-NC-ND 4.0](https://licensebuttons.net/l/by-nc-nd/4.0/80x15.png)](https://creativecommons.org/licenses/by-nc-nd/4.0/)

SKT는 오픈소스 소프트웨어 생태계를 통한 다양한 기술 협업을 기반으로 가상화 인프라 플랫폼인 TACO (SKT All Container OpenStack)를 개발하고 있으며, 사내외 여러 분야에 적용하기 위해 노력하고 있습니다. Airship의 Armada는 OpenStack-Helm과 함께 TACO의 기반이 되는 오픈소스 프로젝트이며, SKT는 계속해서 AT&T를 비롯한 다양한 글로벌 회사들과 협력하여 발전시켜 나갈 계획입니다.
또한, 저희 기술 블로그 포스팅중에서 "TACO All-In-One 설치"를 참조하셔서, Armada를 비롯한 관련 오픈 소스 기술들을 손쉽게 다루어 볼 수 있도록 TACO All-In-One 설치 코드를 오픈 해 놓았습니다.

## 관련 정보  
Airship과 관련된 내용은 프로젝트 홈페이지 (http://www.airshipit.org)를 통해서 더 상세히 알아보실 수 있습니다. 

[Airship]: http://www.aireshipit.org
[OpenStack-Helm]: https://github.com/openstack/openstack-helm
[OpenStack]: https://www.openstack.org/ 
[Kubernetes]: https://kubernetes.io/
[SKT Global Blog]: https://www.globalskt.com/home/info/2313
