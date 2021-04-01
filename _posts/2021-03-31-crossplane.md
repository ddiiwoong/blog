---
layout: single
title: "crossplane 활용"
comments: true
classes: wide
description: "crossplane과 terraform 비교"
published: false
keywords: "Kubernetes, crossplane, terraform, iac, provisioning"
categories:
  - Kubernetes
tags:
  - Kubernetes
---

21년 첫번째 글. 제품에 필요한 provisioner를 고민중에 계속 terraform과 ansible을 만지작 거리다가 작년 KubeCon에서 봤던게 갑자기 생각나서 간단히 구성하고 테라폼과 비교해서 장단점을 살펴본다.  

## Crossplane

- Release 0.1 : Crossplane is an open source multicloud control plane.  
  ![0.1](https://crossplane.io/docs/v0.1/media/arch.png)

- Release 1.1 : Crossplane is an open source Kubernetes add-on that enables platform teams to assemble infrastructure from multiple vendors, and expose higher level self-service APIs for application teams to consume.  

2018년 0.1이 소개된 이후 현재 1.1 릴리스된 상태이고, 주 목적은 0.1 릴리스 노트에도 있듯이 쉽게 정의를 내리면 Multicloud control plane이다. 현재 릴리스 1.1 정의에도 있듯이 Kubectl을 사용한 멀티클라우드 인프라 및 서비스를 프로비저닝하고 관리하는 도구로 방향성이 명확해졌다. 

Ubbound라는 웹 기반 crossplane 서비스를 출시했고, 스타트업에 종사하다보니 당연히 펀딩 규모를 확인하게 되는데 2년만에 5명의 개발자가 시리즈 A로 9백만$를 유치할 정도면 어느정도 검증된 솔루션을 가지고 있다는 반증이기도 하다.  

## Crossplane Concepts

기본적으로 Kubernetes API를 활용한다. 그로 인해 추상화된 기능을 몇가지로 분리해서 Custom Resource([CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/))로 관리를 하고 있다.  

1. Package
  Package는 Kubernetes CRD이면서 [Custom Controller](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-controllers)로 관리 대상이 되는 Cloud Provider(GCP, AWS등)를 컨트롤 하기 위한 번들링 패키지라고 보면 된다. 기본적으로 OCI 호환 이미지로 Registry에 올려서 사용가능하고, AWSCLI를 업그레이드 하듯이 기존 인프라에 영향을 주지않고 버전 업그레이드나 버그 픽스가 쉽게 가능하다. 또한 의존성과 권한 관리를 담당한다고 한다.  
  Package는 Provider와 Configurtion으로 구성되고 아래에서 따로 언급한다.  

2. Provider
  Provider는 외부 클라우드 서비스에 자원을 프로비저닝을 가능하게 하는 CRD로 현재 보유중인 Kubernetes 클러스터에 등록되며 각 사전에 설치된 Package를 통해 Provider와 Configurtion이 구성되는 구조이다. Terraform의 Provider와 동일하다고 볼 수 있다.  
  현재 공식적으로 지원되는 Provider는 AWS, GCP, Azure, Rook, Alibaba 이 있고 [building template](https://github.com/crossplane/provider-template)도 제공한다.

3. Managed Resource
  Managed Resource는 crossplane이 자원을 관리할 때 사용되는 표현 방식으로 Terraform의 Resource와 거의 동일하다. 모든 API는 v1.1기준으로 [https://crossplane.io/docs/v1.1/api-docs/overview.html](https://crossplane.io/docs/v1.1/api-docs/overview.html)에서 확인이 가능하다. CNCF기반으로 파생된 프로젝트이기에 RDS, ECR, EKS, VPC 등 대부분 Managed Kubernetes 클러스터를 관리하는 API위주로 되어 있다.

4. Composing Infrastructure(XRs)
  기본적으로 선언형(Declarative) 컨피그 방식으로 관리가 되는데 예를 들면, 기본적으로 제공되는 AWS API이외에도 구성된 인프라를 관리하는 방식으로 RDS DB엔진 버전이라던지, 스토리지 크기등을 구성할 수 있도록 되어있다. 자세한 내용은 실제 구현할 때 살펴본다.

## Install & Configure

일단 클러스터를 준비하자. 아래 진행되는 내용은 Docker Desktop 로컬 클러스터에서 시작한다. 반드시 필요한 구성요소는 Kubernetes Cluster와 Helm 3.0이상 필요하다. 네임스페이스를 생성하고 차트 레포지토리 등록, 업데이트 그리고 crossplane를 설치한다. 

```sh
kubectl create namespace crossplane-system

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

설치후에 Crossplane의 상태를 확인한다.  

```sh
$ kubectl get all -n crossplane-system
NAME                                           READY   STATUS    RESTARTS   AGE
pod/crossplane-54dd944978-2vrks                1/1     Running   0          72s
pod/crossplane-rbac-manager-549c7c6dc9-f8x5x   1/1     Running   0          72s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/crossplane                1/1     1            1           72s
deployment.apps/crossplane-rbac-manager   1/1     1            1           72s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/crossplane-54dd944978                1         1         1       72s
replicaset.apps/crossplane-rbac-manager-549c7c6dc9   1         1         1       72s
```

위에서 언급했던 Package를 설치하는 방법이 kubectl을 사용하는 것이 기본이기 때문에 kubectl plugin 형태로 사용하기 위해 CLI를 설치한다.

```sh
$ curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
kubectl plugin downloaded successfully! Run the following commands to finish installing it:

sudo mv kubectl-crossplane /usr/local/bin
kubectl crossplane --help

Visit https://crossplane.io to get started. 🚀
Have a nice day! 👋

```

원하는 path로 옮기고 나면 `kubectl crossplane`을 실행할 수 있다.  

여기서 XRs(composite resources) 대한 내용을 잘 이해해야 하는데, Crossplane은 인프라를 CRD로 단순히 배포만 하는것이 아니라 우리가 흔히 이야기하는 배포를 진행하고 이후 ansible이나 puppet 같은 도구로 config를 원하는 대로 커스터마이징을 하게 되는데 이런 과정을 다시 추상화한 개념이라고 볼 수 있다.  

예를 들면 DB를 배포하고 비즈니스에 맞는 스키마를 별도의 리소스로 정의를 하고 이를 반영하는것을 XRs(composite resources)라고 말한다. 

### 

            


