---
layout: single
title: "eksworkshop"
comments: true
classes: wide
description: "eskworkshop 튜토리얼을 실행하고 간단한 Microservice앱을 배포해보자"
keywords: "eksworkshop"
categories:
  - Kubernetes
tags:
  - eks
  - aws
  - Kubernetes
  - eskworkshop
  - kubectl
  - Managed Kubernetes
---

## Amazon EKS (Elastic Container Service for Kubernetes)

Kubernetes Managed Service인 Amazon EKS가 2018년 7월 출시되고 2019년 1월 정식으로 서울리전에 출시되었다. 개인적으로 완전관리형 Kubernetes 출시가 늦어져서 AWS 행보가 다소 늦다고 생각은 했으나 ALB와 VPC연동, 여러가지 기존 OpenSource와의 연결고리를 배제하고 자체 Managed서비스와 연동할것들이 많기 때문에 당연히 타사에 비해 늦어진걸로 보인다. 언제나 그랬지만 오픈소스를 받아들이는 느낌이 아니라 뭔가 완성된 제품을 쓰는 느낌(?)이다. 물론 불편한 부분과 감수해야할 내용들은 조금 있지만 기존 AWS 충성 User에게 호응을 얻을수 있기 때문이 아닐까라는 생각을 해보면서 포스팅을 시작하려고 한다.  

오픈소스가 아닌 Managed서비스에 대한 리뷰는 처음이지만 4월 10일 [AWS판교소모임](https://www.meetup.com/ko-KR/awskrug/events/260024327/) 발표도 있고, 실제 고려중인 아키텍처에 포함이 되어야 하는 EKS에 대해서 살펴보고 eskworkshop 튜토리얼을 실행해보면서 다른 관리형 Kubernetes 서비스들에 비해 어떤 사항을 좀더 고민해야 하는지 정리해보고 넘어가면 좋을듯 하다.

## eksworkshop.com

[https://eksworkshop.com/](https://eksworkshop.com/)  
Kubernete를 처음접하는 유저를 위한 기본 개념과 아키텍처, 그리고 VPC, ALB를 활용하여 EKS에 대한 설치, 구성, 데모앱 배포 등을 해볼수 있는 튜토리얼 사이트이다.  

[AWSKRUG](https://www.facebook.com/groups/awskrug/)에서 한글화 작업도 진행중이다.  [한글화 링크](https://awskrug.github.io/eks-workshop/deploy/)

## eksworkshop 따라하기전 사전 준비사항

eksworkshop에서는 기본적으로 workshop이라는 신규 IAM 계정을 생성하고 Cloud9 Workspace 와 몇가지 설정들을 진행하지만 [AWS판교소모임](https://www.meetup.com/ko-KR/awskrug/events/260024327/)을 위해 최대한 비용이 드는 구성요소를 배제하고 작성하고자 한다.

### AWS account
Free Tier는 EKS를 자유롭게 활용할수 없다.  
관련 issue - [https://github.com/aws/containers-roadmap/issues/78](https://github.com/aws/containers-roadmap/issues/78)  

실제 사용중인 계정이나 Credit이 확보된 계정이 필요하다.  

### IAM 설정 (JSON template)
EKSworkshop에서는 Full administrator 계정을 필요로 하지만 eksctl로 배포를 진행하므로 그 기준으로 IAM설정을 진행한다.  

[terraform eks iam 설정](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/examples/eks_test_fixture/README.md)을 참고하려고 했지만 eksctl과 terraform과의 약간 다른 방식의 배포로 인해 어쩔수없이 EKS Full Access권한을 할당하였다.  
(다른 유경험자의 도움이 필요한 상황 ㅠㅠ)

자세한 JSON 내용은 [링크](https://github.com/ddiiwoong/eksworkshop/blob/master/iam_for_eksworkshop.json)를 참고한다.  

### kubectl, aws-iam-authenticator
* kubectl : kubernetes CLI
* aws-iam-authenticator : AWS IAM Authenticator CLI

#### kubectl config를 저장하기 위해 .kube directory를 생성

```bash
$ mkdir -p ~/.kube
``` 

#### kubectl 설치 (linux)
```bash
$ sudo curl --silent --location -o /usr/local/bin/kubectl "https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/linux/amd64/kubectl"
$ sudo chmod +x /usr/local/bin/kubectl
```

#### kubectl 설치 (MacOS Homebrew)
MacOS는 brew로 설치하였다.  
[https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-with-homebrew-on-macos](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-with-homebrew-on-macos)
```bash
$ brew install kubernetes-cli
```

#### kubectl 설치 (windows PowerShell)
[https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-kubectl.html](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-kubectl.html)

```
curl -o kubectl.exe https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/windows/amd64/kubectl.exe
```


#### kubectl 설치 확인

현재 MacOS에서는 1.11.7 버전이 설치되어 있다. (다른경로로 설치됨)
```bash
$ kubectl version --short --client
Client Version: v1.11.7-dispatcher
```

#### aws-iam-authenticator 설치
Amazon EKS는 IAM을 사용하여 Kubernetes용 AWS IAM Authenticator를 통해 Kubernetes 클러스터에 인증을 제공한다.  Go(버전 1.7이상)가 설치되어 있으면 아래와 같이 진행하면 된다. 
```bash
$ go get -u -v github.com/kubernetes-sigs/aws-iam-authenticator/cmd/aws-iam-authenticator
$ sudo mv ~/go/bin/aws-iam-authenticator /usr/local/bin/aws-iam-authenticator
```

만약 Go설치가 안되어 있다면 다음 링크를 통해 설치 진행할수 있다.  
[https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-aws-iam-authenticator.html](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-aws-iam-authenticator.html)

#### aws-iam-authenticator binary 다운로드

Linux: [https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/aws-iam-authenticator](https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/aws-iam-authenticator)  
MacOS: [https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/darwin/amd64/aws-iam-authenticator](https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/darwin/amd64/aws-iam-authenticator)  
Windows: [https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/windows/amd64/aws-iam-authenticator.exe](https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/windows/amd64/aws-iam-authenticator.exe)

##### MacOS의 경우 
```bash
$ curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/darwin/amd64/aws-iam-authenticator
$ chmod +x ./aws-iam-authenticator
$ cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH
```

##### MacOS bash 환경변수 추가
```bash
$ echo 'export PATH=$HOME/bin:$PATH' >> ~/.bash_profile
```

##### MacOS zsh 환경변수 추가
```bash
$ echo 'export PATH=$HOME/bin:$PATH' >> ~/.zshrc
```

#### aws-iam-authenticator binary 확인
```bash
$ aws-iam-authenticator help
```

### eksctl

![mascot](https://github.com/weaveworks/eksctl/blob/master/logo/eksctl.png?raw=true)

eksctl은 weaveworks에서 contribute하고 있는 오픈소스로 EKS 클러스터를 생성하는 간단한 CLI 도구이다. Go로 작성되어 있고 CloudFormation을 기본으로 동작한다.  

[https://eksctl.io/](https://eksctl.io/)  
[https://github.com/weaveworks/eksctl](https://github.com/weaveworks/eksctl)

#### eksctl binary 다운로드
```bash
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
$ sudo mv -v /tmp/eksctl /usr/local/bin
```

#### MacOS Homebrew 설치
```bash
$ brew tap weaveworks/tap
$ brew install weaveworks/tap/eksctl
```

#### Windows 설치 (PowerShell, chocolatey)
```
chocolatey install eksctl
```

#### eksctl 동작확인
```bash
$ eksctl version
```

#### CLI API 자격증명 구성

사전설정한 aws configure가 있는지 확인. 기존 Terraform이나 kops 사용한적이 있으면 건너뛰어도 된다.  
```bash
$ ls  ~/.aws
```
없을경우 aws 자격증명을 생성한다.

#### ~/.aws/credentials
```
[default]
aws_access_key_id={EXAMPLE}
aws_secret_access_key={EXAMPLEKEY}
```

#### ~/.aws/config
```
[default]
region=ap-northeast-2
output=json
```

## EKS 배포
kubectl, aws-iam-authenticator, eksctl, AWS 자격증명 환경까지 구성되어 있으면 바로 배포가 가능하다.  

```bash
$ eksctl create cluster --name=eksworkshop-eksctl --nodes=3 --node-ami=auto
[ℹ]  using region ap-northeast-2
[ℹ]  setting availability zones to [ap-northeast-2a ap-northeast-2c ap-northeast-2a]
[ℹ]  subnets for ap-northeast-2a - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for ap-northeast-2c - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for ap-northeast-2a - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  nodegroup "ng-cfb3cb01" will use "ami-0c7972077aa002104" [AmazonLinux2/1.11]
[ℹ]  creating EKS cluster "eksworkshop-eksctl" in "ap-northeast-2" region
[ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial nodegroup
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-northeast-2 --name=eksworkshop-eksctl'
[ℹ]  creating cluster stack "eksctl-eksworkshop-eksctl-cluster"
[ℹ]  creating nodegroup stack "eksctl-eksworkshop-eksctl-nodegroup-ng-cfb3cb01"
[ℹ]  --nodes-min=2 was set automatically
[ℹ]  --nodes-max=2 was set automatically
[✔]  all EKS cluster resource for "eksworkshop-eksctl" had been created
[✔]  saved kubeconfig as "/Users/ddii/.kube/config"
[ℹ]  nodegroup "ng-cfb3cb01" has 0 node(s)
[ℹ]  waiting for at least 2 node(s) to become ready in "ng-cfb3cb01"
[ℹ]  nodegroup "ng-cfb3cb01" has 2 node(s)
[ℹ]  node "ip-192-168-42-42.ap-northeast-2.compute.internal" is ready
[ℹ]  node "ip-192-168-66-165.ap-northeast-2.compute.internal" is ready
[ℹ]  kubectl command should work with "/Users/ddii/.kube/config", try 'kubectl get nodes'
[✔]  EKS cluster "eksworkshop-eksctl" in "ap-northeast-2" region is ready
```

서울리전(ap-northeast-2)기준 약 15-20분이 소요되며 `eksctl` 기본 설정값은 다음과 같다.

* 클러스터명은 자동생성, --name 옵션으로 지정가능 (eksworkshop-eksctl)
* CloudFormation : eksctl-{$Cluster_Name}-cluster
* m5.large * 2 instances ([EKS Instance Type](https://github.com/awslabs/amazon-eks-ami/blob/7f6c8cb3597e17f6e5f7df96d12bccf5604dc909/amazon-eks-nodegroup.yaml) NodeInstanceType.AllowedValues 참고)
* Default AMI : AWS EKS AMI (custom EKS AMI 가능 - Packer활용)
* Default Region : us-west-2
* dedicated VPC : 192.168.0.0/16 
* kubernetes version : 1.11.x ([EKS Version](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/platform-versions.html) 참고)
* StorageClass : gp2 ([AWS EBS](https://kubernetes.io/docs/concepts/storage/storage-classes/#aws-ebs) 참고)
* CNI : [Amazon VPC](https://github.com/aws/amazon-vpc-cni-k8s)
* Node Autoscaler : --node-min, --node-max Auto Scaling 설정
* 기본 Pod 개수 : [참고](https://github.com/awslabs/amazon-eks-ami/blob/7f6c8cb3597e17f6e5f7df96d12bccf5604dc909/files/eni-max-pods.txt)
* nodegroup : worker가 포함되는 group 
* kubeconfig : ~/.kube/config 로 통합됨
 
### Config File 사용

[https://github.com/weaveworks/eksctl/tree/master/examples](https://github.com/weaveworks/eksctl/tree/master/examples)를 참고하여 `YAML`형태로 작성하여 배포가능하다. 

```bash
$ eksctl create cluster -f example.yaml
```

기존에 관리하는 VPC subnet정보 및 AutoScaling, AZ(availabilityZones)설정, nodegroup 관리, node Instance에 preBootstrapCommand등을 아래 예시와 같이 미리 작성하면 GitOps측면에서 활용도가 더욱 높아질수 있다.

#### 05-advanced-nodegroups.yaml
```yaml
# An advanced example of ClusterConfig object with customised nodegroups:
--- 
apiVersion: eksctl.io/v1alpha4
kind: ClusterConfig

metadata:
  name: cluster-5
  region: eu-west-2

nodeGroups:
  - name: ng1-public
    instanceType: m5.xlarge
    minSize: 2
    maxSize: 8
    labels:
      nodegroup-type: frontend-workloads
    iam:
      withAddonPolicies:
        autoScaler: true

  - name: ng2-private-a
    instanceType: m5.large
    desiredCapacity: 2
    labels:
      nodegroup-type: backend-cluster-addons
    privateNetworking: true
    preBootsrapCommand:
      # allow docker registries to be deployed as cluster service 
      - 'echo {\"insecure-registries\": [\"172.20.0.0/16\",\"10.100.0.0/16\"]} > /etc/docker/daemon.json'
      - "systemctl restart docker"

  - name: ng3-private-b
    instanceType: c3.8xlarge
    desiredCapacity: 4
    labels:
      nodegroup-type: very-special-science-workloads
    privateNetworking: true
    availabilityZones: ["eu-west-2a"] # use single AZ to optimise data transfer between isntances
    preBootstrapCommand:
      # disable hyperthreading
      - "for n in $(cat /sys/devices/system/cpu/cpu*/topology/thread_siblings_list | cut -s -d, -f2- | tr ',' '\n' | sort -un); do echo 0 > /sys/devices/system/cpu/cpu${n}/online; done"

# cluster AZs must be set explicitly for single AZ nodegroup example to work
availabilityZones: ["eu-west-2a", "eu-west-2b", "eu-west-2c"]
```

## Kubernetes 대시보드 배포

Kubernetes 공식 대시보드는 기본으로 배포되지 않기 때문에 수동으로 배포해야한다. 설치 방법은 [공식 문서](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)에서 확인가능하다.  

위에서 구성된 클러스터에서 Kubernetes 대시보드를 배포한다.
```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

접속을 위해 kube-proxy 기능을 활용하여 8080포트로 expose를 진행한다. 모든 인터페이스에서 필터링없이 접속이 가능하도록 옵션을 지정한다. 
아래 명령은 터미널에서 백그라운드로 계속 동작한다.  
```bash
$ kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &
W0328 16:39:09.061754    9100 proxy.go:139] Request filter disabled, your proxy is vulnerable to XSRF attacks, please be cautious
```

TOKEN으로 접속하기 위해 aws-iam-authenticator를 통해 해당 클러스터 token을 확인한다.  
```bash
$ aws-iam-authenticator token -i eksworkshop-eksctl --token-only
k8s-aws-v1.aHR0cHM6Ly9zdHMuYXAtbm9ydGhlYXN0LTIuYW1hem9uYXdzLmNvbS8_QWN0aW9uPUdldENhbGxlcklkZW50aXR5JlZlcnNpb249MjAxMS0wNi0xNSZYLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFKMjVNVjJSVVZQNlRWTURBJTJGMjAxOTAzMjglMkZhcC1ub3J0aGVhc3QtMiUyRnN0cyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMTkwMzI4VDA3NDEwNVomWC1BbXotRXhwaXJlcz0wJlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCUzQngtazhzLWF3cy1pZCZYLUFtei1TaWduYXR1cmU9Yjc0NzkzYzUwOTU5NDYwMzMxMjY2YjExYWY4ODBkM2Q2OWQ5MWRhYzFhZWY1NjZmZTAwNTNlNWY2MTM0NGFlZQ
```

그냥 localhost:8080 으로만 접속하면 구성된 클러스터 kube API list를 확인되므로 아래 경로로 접속한다.  
위에서 출력된 TOKEN값으로 로그인한다. Token 세션 Timeout이 있으니 세션만료시 `aws-iam-authenticator` 명령을 통해 갱신값을 입력하면 된다.  
```
http://localhost:8080/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login
```
![dashboard](/images/k8s_dashboard.png)


## Microservice app 배포
가장 기본적인 [Sock Shop](https://microservices-demo.github.io/)을 배포하기 위해 Git Clone 수행
```bash
$ git clone https://github.com/microservices-demo/microservices-demo.git sockshop
$ cd sockshop/deploy/kubernetes
```

`NodePort`로 되어있는 Front-End Service를 `LoadBalancer`로 수정한다.
```bash
$ sed -i 's/NodePort/LoadBalancer/g' complete-demo.yaml
$ cat complete-demo.yaml | grep LoadBalancer
 type: LoadBalancer
```


namespace 생성 및 sock-shop 배포
```bash
$ kubectl create namespace sock-shop
$ kubectl apply -f complete-demo.yaml
$ kubectl get all
```

서비스 접속확인을 위해서는 ALB배포시간(DNS전파)이 일정 소요되므로 잠시후 접속을 시도한다.
```bash
$ kubectl get svc
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)        AGE
carts          ClusterIP      10.100.84.58     <none>                                                                      80/TCP         1h
carts-db       ClusterIP      10.100.227.156   <none>                                                                      27017/TCP      1h
catalogue      ClusterIP      10.100.206.52    <none>                                                                      80/TCP         1h
catalogue-db   ClusterIP      10.100.119.66    <none>                                                                      3306/TCP       1h
front-end      LoadBalancer   10.100.249.164   a4c42cfe951be11e9bb8c0a8cd8a2e5d-8156023.ap-northeast-2.elb.amazonaws.com   80:30001/TCP   1h
orders         ClusterIP      10.100.160.17    <none>                                                                      80/TCP         1h
orders-db      ClusterIP      10.100.70.203    <none>                                                                      27017/TCP      1h
payment        ClusterIP      10.100.57.233    <none>                                                                      80/TCP         1h
queue-master   ClusterIP      10.100.146.109   <none>                                                                      80/TCP         1h
rabbitmq       ClusterIP      10.100.32.115    <none>                                                                      5672/TCP       1h
shipping       ClusterIP      10.100.180.174   <none>                                                                      80/TCP         1h
user           ClusterIP      10.100.211.41    <none>                                                                      80/TCP         1h
user-db        ClusterIP      10.100.87.142    <none>                                                                      27017/TCP      1h
```
  

브라우저에서 ALB주소 `a4c42cfe951be11e9bb8c0a8cd8a2e5d-8156023.ap-northeast-2.elb.amazonaws.com` 로 접속하여 sock-shop Demo Web을 확인할수 있다.  

![sockshop](/images/sock-shop.png)


## EKS 클러스터 및 삭제

위에서 외부접속을 위해 LoadBalancer를 수동으로 설정하였으므로 EC2 - Load Balancer서 프로비저닝된 ALB를 삭제하고 진행해야 한다.  

클러스터에서 실행 중인 모든 서비스를 다시 확인하고 EXTERNAL-IP값과 연결된 모든 서비스를 삭제한다.  
```bash
$ kubectl get svc --all-namespaces
$ kubectl delete svc front-end
```

ALB와 VPC 삭제가 완료된것을 확인하고 클러스터를 삭제하자.  

```bash
$ eksctl delete cluster --name=eksworkshop-eksctl
[ℹ]  deleting EKS cluster "eksworkshop-eksctl"
[ℹ]  will delete stack "eksctl-eksworkshop-eksctl-nodegroup-ng-3af535b7"
[ℹ]  waiting for stack "eksctl-eksworkshop-eksctl-nodegroup-ng-3af535b7" to get deleted
[ℹ]  will delete stack "eksctl-eksworkshop-eksctl-cluster"
[✔]  kubeconfig has been updated
[✔]  the following EKS cluster resource(s) for "eksworkshop-eksctl" will be deleted: cluster. If in doubt, check CloudFormation console
```

## 정리
서울 리전(ap-northeast-2)에 EKS가 오픈되고 Production환경을 EKS로 넘어가는 기업들이 많아지고 있는건 아주 긍정적인 현상이다.  
또한 엄청난 속도의 기술개발과 다양한 툴들로 Kubernetes Cluster를 구성하거나 Microservice형태의 App을 배포하는것은 점점 대중화 되어가고 있다.  
실제 Production에서 구현을 하기 위해서는 보안 및 성능을 동시에 고려한 네트워크(CNI), Ingress 설정이나 전체 클러스터 퍼포먼스 측면에서의 파라미터 튜닝이 더욱 더 중요해지고 관련된 DevOps(SRE) 인력들의 중요도가 높아질것은 분명해 보인다.  

EKS를 오픈소스 측면에서 고려했을때에는 예전에 페이스북이나 다른곳에서 종종 이야기 했던것처럼 AWS 고유의 Lock-in 전략을 엄청나게 고민하고 발표한듯한 생각이 한 느낌을 지울수는 없는건 사실이지만 훌륭한 제품인건 확실하고 단기간에 서비스 성장속도를 낼 수 있는 서비스라 생각한다.  

마지막으로 Managed Kubernetes의 선택은 엔지니어의 몫으로 보고 각 벤더별 비교한 아래 링크를 참조해서 선정 기준을 잡으면 더욱 좋을것 같다.  
[https://docs.google.com/spreadsheets/d/1U0x4-NQegEPGM7eVTKJemhkPy18LWuHW5vX8uZzqzYo/edit#gid=0](https://docs.google.com/spreadsheets/d/1U0x4-NQegEPGM7eVTKJemhkPy18LWuHW5vX8uZzqzYo/edit#gid=0)

