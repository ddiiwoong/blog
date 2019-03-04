---
layout: single
title: "Spinnaker on Kubernetes #2"
comments: true
description: "Spinnaker에 대해 알아봅니다 #2"
keywords: "Spinnaker, Kubernetes, k8s, CI/CD, Continuous Delivery, Continuous Deployment"
categories:
  - Kubernetes
tags:
  - CI/CD
  - Kubernetes
  - Spinnaker
  - Continuous Delivery
  - Continuous Deployment
  - Pipeline
---

# Spinnaker on Kubernetes

이전 포스팅 [Spinnaker on Kubernetes #1](https://ddiiwoong.github.io/2018/spinnaker-advanced-1/)에서 검토할때는 많이 개념을 이해하기 어려웠던것 같지만 어느정도 시간이 지났고 또 몇일 후에 발표도 있어서 다른 이야기를 해보고자 한다.  

지난 포스팅에 대충 집고 넘어간 용어들에 대한 정리를 다시 하고 기본적인 사상들을 정리해보고자 한다. 허접한 플랫폼 엔지니어 생각이니 언제든 다른 의견을 환영하는 바이다. 

## What is spinnaker? (+History)
최근 트렌드인 멀티 클라우드를 지향하는 오픈소스 플랫폼이다.  
2014년 Netflix의 Asgard로 시작되어 2015년에 오픈소스화 되었다.  
빠른 속도와 신뢰도있는 소프트웨어 릴리즈를 위해 만들어졌으며 대부분의 메이저 클라우드 프로바이더들을 지원한다.(AWS,GCP,Azure,openstack..)  
현재 Netflix, Google, MS, Veritas등이 Contribution을 하고 있다.

## 왜 Spinnaker를 써야하지?
여러가지 이유가 있겠지만
* Multi-Cloud용 Continuous Delivery/Deployment Platform 으로 대체가 가능
* 다양한 pipeline 형태로 배포가 가능하고 Rollback이 쉬움
* 빠른 배포가 가능하고 여러번 배포가 용이함
* 유연한 pipeline management system을 가지고 있음
* 다양한 배포전략을 가진다(Blue-Green, Rolling Red/Black, Canary)
* community 활동 활발 (github, slack) - 답은 잘 안해줌 ㅠㅠ
* VM과 Container 동시에 통합관리 가능
* CI통합 용이(Jenkins)
* CLI를 통한 설치 및 관리(halyard)
* VM, Helm Packaging 가능
* RBAC 지원
* Notification - Email, Slack, Hipchat등
* Safe Deployment - Judgement (승인기능)
* Chaos Monkey Built-in

이정도면 무조건 써야하지 않을까?

## Jenkins vs Spinnaker

|Jenkins|Spinnaker|
|:----------:|:----------:|
|강력한 빌드서버|	클라우드 자원의 1차 연동|
|완전한 deployment tool이 아님|vm & deployments 안에 빌드되어 있음|
|스크립팅이 많이 필요함|별도의 스크립팅이 많이 필요없음|
|기능들이 모두 플러그인 형태|CI tool이 아님(CI tools이 백엔드로)|

## Kubernetes vs Spinnaker

|Kubernetes|Spinnaker|
|:----------:|:----------:|
|리소스 사용 제한|정의한 퍼센트로 rollout|
|slow rollout|각 단계별 검증 가능|
|High rollback cost|Fast rollbacks|
|Linear rollouts|resource 사용량이 큼|
|검증단계가 없음||

## Deploy Pipeline

Spinnaker를 사용할때 기본적으로 아래와 같은 파이프라인으로 구성한다.  
수동으로 UI나 API로 트리거링할수 있고, 자동으로 Jenkins 등과 트리거 연동하여 빌드완료시 배포되도록 할수 있다.

![spinnaker-pipeline](/images/spinnaker-pipeline.png)

## Deployment Strategies
Spinnaker에서의 배포전략은 다음과 같이 제공된다.

![deployment-strategies](https://www.spinnaker.io/concepts/deployment-strategies.png)

### Red / Black (same as Blue / Green)
* 동일한 양의 instance로 이루어진 새로운 Server Group을 생성한다
* 신규 Server Group이 정상상태가 되면 LB는 신규 Server Group에 트래픽을 분산한다. 
  
### Rolling red/black
* 이전과 동일하지만 인스턴스별 또는 그룹별로 rolling
  
### Canary
* 가장 작은 개수의 인스턴스를 교체시키고
* 새로운 버전으로 트래픽을 분산시킨다 (1~5프로)
* 새로운 버전에 이슈가 없을때까지 테스트를 진행하고
* 특정시간까지 이슈가 없으면 배포를 늘려간다. 

## 용어정리 2탄

이전 [post](https://ddiiwoong.github.io/2018/spinnaker-advanced-1/)에서 정리한걸 다시 복기하고 추가적인 내용을들 적어봤다.

사용하면서 혼돈이 많이 생기는 부분이다 이게 GCE나 EC2를 쓰면 용어 매칭이 쉬운데 k8s를 위한 별도의 메뉴가 아닌 기능을 통합하다보니 용어가 조금 혼동스럽게 구성이 되었다.  
특히 Load Balancer 부분은 Service로 매핑되고 퍼블릭 k8s에서 제공하는 Type LoadBalancer는 미지원한다.  
그리고 모든 Resource들은 Deploy, Delete, Scale, Rollout(Undo, Pause, Resume)을 지원하며 Versioning이 지원된다.  Versioning은 [여기](https://www.spinnaker.io/reference/providers/kubernetes-v2/#strategy)에 설명된 대로 ```strategy.spinnaker.io/versioned``` annotation을 통해 manifest별로 재정의가 가능하다.

| Spinnaker | Kubernetes | 비고 |
|:----------:|:----------:|:-----------:|
| Server Group | [Workloads](https://www.spinnaker.io/reference/providers/kubernetes-v2/#workloads) | [CRD의 경우 별도 Build](https://www.spinnaker.io/guides/developer/crd-extensions/) |
| Clusters | Logical Server Group	 |  |
| Load Balancer | Services | LoadBalancer(k8s) 미지원 |
| Firewall | NetworkPolicies |  |

### Application Management
Spinnaker에서 Application 이란 배포하려는 서비스를 나타내는 구조라 생각하면 된다.  
* pipeline
* Clusters, Server Group의 집합이며, firewall과 loadbalancer를 포함한다.
* Canary Config

### Cluster
Kubernetes의 Cluster가 아니라 Spinnaker에서 Server Group의 논리적인 그룹

### Server Group
기본자원인 서버그룹은 배포할수 있는 artifacts(vm image, docker image, source)와 인스턴스(pod) 수, Auto-Scaling, metadata 등 기본 구성등을 가지고 있다.  
서버그룹은 LoadBalacer나 Firewall 도 선택적으로 연결되고, vm이나 pod 형태로 배포된 application의 집합체라 볼수 있다.

### Cloud Provider
* IaaS - AWS, GCP, Azure, Oracle, Openstack
* PaaS -  Google App Engine
* Orchestrator - K8s, DC/OS
* Docker v2 Registry

### Account
Cloud Provider에 인증하기 위한 Spinnaker에서만 사용하는 Account Name

### Pipeline
Pipeline은 주요 배포 관리도구로 사용된다. 
Stage라고하는 일련의 Action으로 구성되며 파이프라인을 따라 Stage간 매개변수 전달이 가능하다.  
수동으로 시작하거나, Jenkins 작업완료, Docker Registry 신규 Docker 이미지, Cron일정 또는 다른 Stage와 같은 이벤트에 의해 자동으로 트리거링되도록 구성할수 있다.  
Pipeline 실행중에(시작/완료/실패) mail, slack, hipchat(사라짐)을 통해 Alert가 가능하다.

![pipeline](https://www.spinnaker.io/concepts/pipelines.png)

### Stage (atomic building block)
Pipeline이 수행할 동작을 말한다.  
Deploy, Resize, Disable, Manual Judgement 등을 수행할수 있다.

![stage](https://www.spinnaker.io/concepts/pipelines/pipeline-tasks.png)

* Stage - Multiple steps
* Step - 진행되기전에 교정/폴링이 필요한 tasks
* Task - 특정 Cloud Platform으로 동시에 여러 API호출
* Operation - 단위 API

## 정리

용어나 개념은 어느정도 정리된듯 하고 다음 포스팅에서는 실제 multi cluster 환경에서 deploy하고 pipeline을 사용하는 내용을 적어볼 예정이다.