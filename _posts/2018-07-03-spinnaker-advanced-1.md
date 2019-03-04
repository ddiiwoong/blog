---
layout: single
title: "Spinnaker on Kubernetes #1"
comments: true
description: "Spinnaker에 대해 알아봅니다 #1"
categories:
  - Kubernetes
tags:
  - CI/CD
  - Kubernetes
  - Spinnaker
---

# Spinnaker on Kubernetes 

지난번 [OpenInfraDay발표](https://www.slideshare.net/openstack_kr/openinfra-days-korea-2018-track-4-provisioning-dedicated-game-server-on-kubernetes-cluster)때 질문을 해주셨는데 요즘 Spinnaker를 많이들 쓰거나 검토를 많이 하는것으로 알고 있다.  

Spinnaker를 설치하는 내용은 많이 있으니 아래 halyard로 설치하는 포스트를 참고하면 된다.

[윤상준의 기술 블로그 - Spinnaker 설치하기](https://yunsangjun.github.io/blog/spinnaker/2018/06/03/installing-spinnaker.html)

이번에 이야기하고자 하는 부분은 실제 Spinnaker를 설치하고 난 후 운영상에 고려해야할 부분들과 팁들을 공유해보려고 한다.

사실 Spinnaker를 구성하면서 가장 어려웠던 부분들은 용어, halyard config 관리와 custom resourse 인식부분 이였다. 나머지들은 뭐 튜토리얼을 따라가면 별로 어렵진 않으니 아래 내용을 차근차근 따라가면서 이해하면 될것 같다. 

## 용어들

사용하면서 혼돈이 많이 생기는 부분이다 이게 GCE나 EC2를 쓰면 용어 매칭이 쉬운데 k8s를 위한 별도의 메뉴가 아닌 기능을 통합하다보니 용어가 조금 혼동스럽게 구성이 되었다.  
특히 Load Balancer 부분은 Service로 매핑되고 퍼블릭 k8s에서 제공하는 Type LoadBalancer는 미지원한다.  
그리고 모든 Resource들은 Deploy, Delete, Scale, Rollout(Undo, Pause, Resume)을 지원하며 Versioning이 지원된다.  Versioning은 [여기](https://www.spinnaker.io/reference/providers/kubernetes-v2/#strategy)에 설명된 대로 ```strategy.spinnaker.io/versioned``` annotation을 통해 manifest별로 재정의가 가능하다.

| Spinnaker | Kubernetes | 비고 |
|:----------:|:----------:|:-----------:|
| Server Group | [Workloads](https://www.spinnaker.io/reference/providers/kubernetes-v2/#workloads) | [CRD의 경우 별도 Build](https://www.spinnaker.io/guides/developer/crd-extensions/) |
| Clusters | Logical Server Group	 |  |
| Load Balancer | Services | LoadBalancer(k8s) 미지원 |
| Firewall | NetworkPolicies |  |


Spinnaker Server Group으로 분류 된 항목은 모두 Spinnaker의 클러스터 탭에 표시가 된다. 가능한 경우 모든 포드가 표기되지만, 해당 [Workloads](https://www.spinnaker.io/reference/providers/kubernetes-v2/#workloads) 이외의 CRD(Custom Resource Definition)는 halconfig에서 아래와 같이 customResources config를 추가하면 deploy는 가능하나 Spinnaker UI에서 보이지는 않는다.  

```
kubernetes:
      enabled: true
      accounts:
      - name: my-k8s-account
        customResources:
        - kubernetesKind: GameServer
```

이유는 바로 다음 링크처럼 [(CRD의 경우 별도 Build필요)](https://www.spinnaker.io/guides/developer/crd-extensions/) 
별도로 Java를 빌드해야 한다. Spinnaker Slack에 문의를 몇번했는데 질문하는 사람만 있고 답은 아무도 안해준다는...  
https://spinnakerteam.slack.com/



## Jenkins 연동

[Spinnaker, Jenkins integration 상세내용 공식문서 ](https://www.spinnaker.io/setup/ci/jenkins/)

Jenkins와 연동하면서 가장 어이없이 헤맨부분은 아래 그림처럼 되어있어야 하는데 Security Realm을 Jenkins Default admin 계정만을 가지고 integration 하려다가 계속 실패하였다. Delegate to servlet container 말고 Jenkins 자체 사용자 DB로 별도 계정을 생성하고 아래 그림처럼 설정을 해야한다.

![Jenkins Config](/images/jenkins_config.png)

위 설정 이후 아래와 같이 Spinnaker UI에서 Jenkins API연동이 가능하다.  

![Spinnaker Jenkins](/images/spin_jenkins.png)


오늘은 여기까지만 하고 다음글에서는 배포전략이나 Network Policy 연동등을 상세히 적어볼 예정이다.

