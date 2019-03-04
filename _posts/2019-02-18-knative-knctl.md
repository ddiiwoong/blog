---
layout: single
title: "Knative CLI - knctl"
comments: true
description: "Knative CLI tool knctl에 대해 알아보자"
keywords: "knative, Kubernetes, k8s, knative 설치, knative 구성, knctl"
categories:
  - Kubernetes
tags:
  - Knative
  - Kubernetes
  - FaaS
  - Serverless
  - CRDs
  - CloudEvents
  - Mesh
  - knctl
---

# Knative CLI - knctl

## knctl
`knctl` 은 Knative CLI 툴로 간단하게 knative cluster를 만들고 Knative를 추상화해서 앱까지 배포할 수 있는 오픈소스이다. 

#### 참고
* https://github.com/cppforlife/knctl  
* https://developer.ibm.com/blogs/2018/11/12/knctl-a-simpler-way-to-work-with-knative/  
* https://starkandwayne.com/blog/public-traffic-into-knative-on-gke/

### Knative 다시 살펴보기

[앞선 포스팅에서도 이야기](https://ddiiwoong.github.io/2018/knative/) 했지만 기존 FaaS(AWS Lambda, Google Cloud Funtions, Azure Function) 과는 다른 Serverless 개념으로 받아들어야 한다.

다시 한번 특징을 나열해보면 아래와 같다.

- Serverless Container의 신속한 배치 
- Automatic scaling up and down to zero
- Istio를 백엔드로 활용하여 Routing 구현
- 배포 된 코드 및 config의 특정 시점 스냅 샷

그리고 다음과 같은 CRDs(Custom Resource Definitions)로 구성된 오브젝트들로 정의된다.

- `Route`는 사용자 서비스에 대한 HTTP endpoint와 Routing을 제공한다.
- `Revisions`은 code(function)와 config로 구성된 불변의 스냅샷. Route를 통해 endpoint를 할당받지 못한 Revision은 자동으로 kubernetes resource에서 삭제됨
- `Configuration`은 요구되는 Revision 최신 상태를 기록하고 생성하고 추적할수 있음. 소스 패키지(git repo나 archive)를 컨테이너로 변환하기 위한 내용이나 메타데이터등을 포함시킬수 있음. 
- `Service`는 `Routes`와 `Configurations` 리소스의 추상화된 집합체. 모든 워크로드의 lifecycle을 관리함. 트래픽을 항상 최신의 Revision으로 route되도록 정의할수 있음

하나씩 조금 자세히 이야기 하면 아래처럼 정리 할수 있다.

### Route
**Route**는 사용자 서비스(Code와 Configuration의 Revision정보)의 네트워크 Endpoint를 제공한다. kubernetes namespace는 여러개의 `Route`를 가질수 있다. `Route`는 하나 이상의 `Revisions`을 가지면서 수명이 길고 안정적인 HTTP Endpoint를 제공한다. 기본구성은 `Route` 객체가 `Configuration`에 의해 생성된 최신의 Revision으로 트래픽을 자동으로 지정한다. 조금더 복잡한 경우로는 istio의 기능을 활용하여 트래픽을 백분율 기준으로 `Route`를 지정할 수 있다. 

### Revision
**Revision**은 Code와 Configuration의 불변의 스냅샷이다. 하나의 `Revision`은 컨테이너 이미지를 참조하고 선택적으로 `Build`를 참조할 수 있다. `Revision`은 **Configuration**이 업데이트 시 생성된다.  
`Route`를 통해 http주소 지정이 불가능한 `Revision`은 폐기 되고 관련된 kubernetes 리소스가 삭제가 된다. 시간이 지남에 따라 `Configuration`이 생성한 `Revision` 히스토리가 제공되고 사용자는 이전 `Revision`로 쉽게 롤백 할 수 있다.  

### Configuration
`Configuration`은 최신의 `Revision`상태를 설명하고, 생성하고, 원하는 상태가 갱신될때 `Revision`의 상태를 추적한다. `Configuration`은 `Build`를 참조하여 소스(git repo 또는 archive)를 컨테이너로 변환하는 방법에 대한 가이드가 포함되어 있거나 단순히 컨테이너 이미지 및 수정에서 필요한 메타 데이터 만 참조 할 수 있다. 

### Product Integration

2019년 2월 현재 0.3이 릴리스되고 있고 벌써 여러 제품에 통합이 되고 있다. 

최근 IBMthink 2019에서 Managed Knative (Experimental)를 내놓기도 하였다.  
https://www.ibm.com/blogs/bluemix/2019/02/announcing-managed-knative-on-ibm-cloud-kubernetes-service-experimental/ 

Istio를 포함한 Knative 마저도 품는 모습으로 managed kubernetes 영역에서 글로벌 플레이어들 모두 서로 치고나가는 모습들을 볼수 있다. 

작년 11월에는 Gitlab 제품안에 serverless라는 extension형태의 서비스가 추가 되기도 하였고,  
https://about.gitlab.com/press/releases/2018-12-11-gitlab-and-triggermesh-announce-gitlab-serverless.html

triggermesh 라는 곳에서는 serverless management platform이라는 이름으로 knative 기반 멀티 서버리스 플랫폼을 출시하기도 하였다.  
https://triggermesh.com/serverless_management_platform/

Pivotal Function Service (PFS), Google GKE SERVERLESS ADD-ON 등은 아직 early access 신청만 받고 있는 상태이다.

오늘은 간단하게 배포할수 있는 툴 knctl과 관련 use-case를 소개하고자 한다.

### Kubernetes Cluster 생성

일단 GKE Free tier에서 Cluster를 하나 생성하자.

```
gcloud container clusters create knative \
  --region=asia-east1-a \
  --cluster-version=latest \
  --machine-type=n1-standard-2 \
  --enable-autoscaling --min-nodes=1 --max-nodes=5 \
  --enable-autorepair \
  --scopes=service-control,service-management,compute-rw,storage-ro,cloud-platform,logging-write,monitoring-write,pubsub,datastore \  
  --num-nodes=3
```

cluster 생성된것을 확인하고 cluster-admin 권한을 할당한다. 
```
$ kubectl get nodes
NAME                                     STATUS    ROLES     AGE       VERSION
gke-knative-default-pool-d1a39347-5m5t   Ready     <none>    1m        v1.11.7-gke.4
gke-knative-default-pool-d1a39347-l6zh   Ready     <none>    1m        v1.11.7-gke.4
gke-knative-default-pool-d1a39347-qv5r   Ready     <none>    1m        v1.11.7-gke.4

$ kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=$(gcloud config get-value core/account)
Your active configuration is: [cloudshell-4728]
clusterrolebinding.rbac.authorization.k8s.io "cluster-admin-binding" created
```

## knctl 설치

이번 포스팅에서는 Mac OS 설치 기준으로 작성하였다.

#### homebrew 설치
```
brew install starkandwayne/kubernetes/knctl
```

#### binary
https://github.com/cppforlife/knctl/releases 
```
$ wget https://github.com/cppforlife/knctl/releases/download/v0.1.0/knctl-darwin-amd64
$ mv knctl-* /usr/local/bin/knctl
$ chmod +x /usr/local/bin/knctl
```

## knctl 로 Knative 배포

설치한 knctl로 Knative 배포를 진행한다. 설치되는 내용을 지켜보고 있으면 `istio`를 먼저 배포하고 그다음에 `Knative`를 설치하는 것을 확인할 수 있다. 배포되는 모듈들의 상태를 하나하나 체크해서 배포하기 때문에 설치상에 과정들을 확인할 수 있다. 

```
$ knctl install --exclude-monitoring
```

테스트를 위한 namespace `hello-test`를 생성한다.
```
$ kubectl create namespace hello-test
namespace "hello-test" created
```

knctl deploy 명령으로 최초 revision을 배포한다.  
아래 결과를 보면 hello-00001 이라고 하는 최초의 revision을 작성하기 때문에 `latest` tag를 달고 배포를 하게 된다. 
```
$ knctl deploy \
      --namespace hello-test \
      --service hello \
      --image gcr.io/knative-samples/helloworld-go \
      --env TARGET=Rev1

Name  hello

Waiting for new revision to be created...

Tagging new revision 'hello-00001' as 'latest'

Tagging new revision 'hello-00001' as 'previous'

Annotating new revision 'hello-00001'

Waiting for new revision 'hello-00001' to be ready for up to 5m0s (logs below)...

hello-00001 > hello-00001-deployment-5cdbfc9bc9-hks6t | 2019/02/17 22:27:50 Hello world sample started.

Revision 'hello-00001' became ready

Continuing to watch logs for 5s before exiting

Succeeded
```

```
kubectl get svc knative-ingressgateway -n istio-system
NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                                                                                                   AGE
knative-ingressgateway   LoadBalancer   10.63.253.209   34.***.***.248   80:32380/TCP,443:32390/TCP,31400:32400/TCP,15011:30082/TCP,8060:31125/TCP,853:32009/TCP,15030:31102/TCP,15031:31631/TCP   6h
```

위처럼 Knative가 프로비저닝 되면서 ingress-gateway가 하나 생성이 되어있는 것을 확인할 수 있고 knctl로도 ingress를 확인이 가능하다.
```
$ knctl ingress list
Ingresses

Name                    Addresses     Ports  Age
knative-ingressgateway  34.***.***.248  80     6h
                                        443
                                        31400
                                        15011
                                        8060
                                        853
                                        15030
                                        15031

1 ingresses

Succeeded
```

## Knative custom domain 연결

Domain이 별도로 없기 때문에 Knative는 내부적으로 example.com이라고 하는 기본 domain을 사용한다. 그래서 실제 `knctl curl` 명령은 내부적으로 `hello.hello-test.example.com`으로 curl을 실행하게 되고 해당 결과를 아래와 같이 확인할 수 있다.

```
$ knctl curl --service hello -n hello-test
Running: curl '-sS' '-H' 'Host: hello.hello-test.example.com' 'http://34.***.***.248:80'

Hello Rev1!

Succeeded
```

kubernetes node가 3개이므로 3개의 pod가 생성된 것을 확인할 수 있다. 일정시간(default:5분)이 지나면 `zero to scale` 관점에서 pod가 종료되므로 다시 확인할때는 다시 curl 명령을 날리게 되면 다시 pod가 올라오게 된다. 해당 개념은 FaaS또는 AWS Lambda에서 Cold-Start와 동일한 것이라 볼 수 있다.  

AWS Cold Start 참고 : https://novemberde.github.io/aws/2018/02/02/Lambda_coldStart.html
```
$ kubectl get po
NAME                                      READY     STATUS    RESTARTS   AGE
hello-00001-deployment-5cdbfc9bc9-hks6t   3/3       Running   0          4m
```

가지고 있는 도메인이 있다면 위에서 나온 `34.***.***.248` IP를 domain에 매핑해보자.
아래에서는 기존 보유중인 skcloud.io 도메인을 연결하였다.
```
$ dig knative.skcloud.io
;; ANSWER SECTION:
knative.skcloud.io.     603     IN      A       34.***.***.248
```

knctl domain을 이용하여 default domain을 `knative.skcloud.io`로 변경한다.
```
$ knctl domain create -d knative.skcloud.io --default
Succeeded
```

knctl routes 명령으로 해당 hello-test app의 Endpoint 정보를 확인할 수 있다.
```
$ knctl routes list -n hello-test
Routes in namespace 'hello-test'

Name   Domain                               Traffic        Annotations  Conditions  Age
hello  hello.hello-test.knative.skcloud.io  100% -> hello  -            3 OK / 3    1h

1 routes

Succeeded
```

5분이상 기다렸다가 curl로 확인하면 Cold-Start 되는 시간(몇초) 지연이 발생하고 결과를 확인할 수 있다.
이후에는 바로 응답을 확인할 수 있다. 
```
$ curl http://hello.hello-test.knative.skcloud.io/
Hello Rev1!
```

## revision 추가

이번에는 revision을 추가해보자. TARGET environment 변수를 `Rev2`로 수정하고 배포를 한다.
기존 hello-00002 revision이 최신 revision으로 갱신되어 배포가 되는것을 확인할 수 있다.
```
$ knctl deploy --service hello \
    --image gcr.io/knative-samples/helloworld-go \
    --env TARGET=Rev2
Name  hello

Waiting for new revision (after revision 'hello-00001') to be created...

Tagging new revision 'hello-00002' as 'latest'

Tagging older revision 'hello-00001' as 'previous'

Annotating new revision 'hello-00002'

Waiting for new revision 'hello-00002' to be ready for up to 5m0s (logs below)...

hello-00002 > hello-00002-deployment-6cf86bbfc7-sblz9 | 2019/02/17 23:25:43 Hello world sample started.

Revision 'hello-00002' became ready

Continuing to watch logs for 5s before exiting

Succeeded
```

신규 `revision` 서비스를 추가된것을 확인할 수 있다. 마찬가지로 몇초간의 Cold-Start delay가 발생할 수도 있다. 
```
$ curl http://hello.hello-test.knative.skcloud.io/
Hello Rev2!

$ knctl curl --service hello
Running: curl '-sS' '-H' 'Host: hello.hello-test.knative.skcloud.io' 'http://34.***.***.248:80'

Hello Rev2!

Succeeded
```

revision list를 확인해보면 현재 latest, previous TAG정보를 확인할 수 있다.
```
$ knctl revision list --service hello
Revisions for service 'hello'

Name         Tags      Annotations  Conditions  Age  Traffic
hello-00002  latest    -            1 OK / 4    14m  100% -> hello.hello-test.knative.skcloud.io
hello-00001  previous  -            1 OK / 4    4h   -

2 revisions

Succeeded
```

## Blue/Green 배포

Blue/Green Deploy는 knctl rollout 명령으로 수행할수 있다.  
rollout 할때 `--managed-route=false` 옵션을 줘야 특정 비율로 routing이 가능하다.  
아래 예시는 TARGET environment 변수를 `blue`, `green`으로 바꿔가면서 배포를 진행하였다.
```
$ knctl deploy --service hello \
    --image gcr.io/knative-samples/helloworld-go \
    --env TARGET=blue \
    --managed-route=false
Name  hello

Waiting for new revision (after revision 'hello-00002') to be created...

Tagging new revision 'hello-00003' as 'latest'

Tagging older revision 'hello-00002' as 'previous'

Annotating new revision 'hello-00003'

Waiting for new revision 'hello-00003' to be ready for up to 5m0s (logs below)...

Revision 'hello-00003' became ready

Continuing to watch logs for 5s before exiting

hello-00003 > hello-00003-deployment-99478dcc5-jf267 | 2019/02/17 23:48:20 Hello world sample started.

Succeeded
```

revision list를 확인하면 아래와 같이 `latest`로 Traffic 전체가 routing 되는 것을 확인할 수 있다.
```
$ knctl revision list --service hello
Revisions for service 'hello'

Name         Tags      Annotations  Conditions  Age  Traffic
hello-00005  latest    -            4 OK / 4    44s  100% -> hello.hello-test.knative.skcloud.io
hello-00004  previous  -            4 OK / 4    2m   -
hello-00003  -         -            1 OK / 4    5m   -
hello-00002  -         -            1 OK / 4    28m  -
hello-00001  -         -            1 OK / 4    4h   -

5 revisions

Succeeded
```

이후에 rollout을 통해 previous로 90%, latest로 10%로 변경을 하면 즉시 반영이 되고 실제 트래픽도 분산되어 들어온다. %가 낮은 쪽으로 routing이 될 경우 Cold-Start가 발생하게 되면 delay는 발생하게 된다.
```
$ knctl rollout --route hello -p hello:latest=10% -p hello:previous=90%
Succeeded

$ knctl revision list
Revisions

Service  Name         Tags      Annotations  Conditions  Age  Traffic
hello    hello-00005  latest    -            2 OK / 4    1h   10% -> hello.hello-test.knative.skcloud.io
~        hello-00004  previous  -            2 OK / 4    1h   90% -> hello.hello-test.knative.skcloud.io
~        hello-00003  -         -            1 OK / 4    1h   -
~        hello-00002  -         -            1 OK / 4    1h   -
~        hello-00001  -         -            1 OK / 4    5h   -

5 revisions

Succeeded
```

간단하게 curl 반복문을 작성하여 돌려보자. 
```
#!/bin/sh
while true
do
curl -sS --max-time 3 http://hello.hello-test.knative.skcloud.io/
done
```

간단하게 위 sh을 돌리면 아래와 같이 Cold-Start delay가 발생할때 time out이 발생하고 이후 green revision으로 접속이 되는것을 볼 수 있다. 
```
$ ./test.sh
curl: (28) Operation timed out after 3002 milliseconds with 0 bytes received
Hello blue!
Hello blue!
Hello blue!
Hello blue!
Hello blue!
curl: (28) Operation timed out after 3003 milliseconds with 0 bytes received
Hello blue!
Hello blue!
Hello blue!
Hello blue!
Hello blue!
Hello blue!
Hello green!
Hello blue!
Hello blue!
Hello blue!
```

## 정리
지금까지 `knctl`을 사용하여 간단하게 knative를 배포하고 custom domain을 연결하여 blue-green 배포까지 해봤다. 
이외에도 Knative Build를 활용하여 Docker image 작업을 하거나 서비스 카탈로그 등을 연동하여 외부 DBaaS를 연동하는 use-case등을 찾아볼수 있다.

아직 초기 단계이지만 Knative는 istio와 함께 IBM, Google, Pivotal등 global player들의 차세대 오픈소스로 부상하고 있다고 볼 수 있다.  

`Zero to scale` 이라는 슬로건아래 Serverless, FaaS 사상을 기반으로 build, serving, event, routing이라고 하는 Cloud Computing 추상화의 끝판으로 진화하고 있다. 앞으로 어떤 모습으로 진화될지 궁금하고 다음번에는 MQ나 Pub/sub를 연동하거나 멀티 클라우드 환경에서 동작하는 모습을 보여주는 것도 좋을것 같다. 희망사항이지만 올해 OpenInfraDay나 Kubernetes Day Korea 행사에서 Hands-on을 진행해보는것도 좋지 않을까?