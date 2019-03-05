---
layout: single
title: "2018 Retrospective"
comments: true
description: "18년도를 돌아보며..."
keywords: "Retrospective, Planning"
categories:
  - Retrospective
tags:
  - Retrospective
  - 2018
  - Planning
  - Change
---

한해가 또 훅 가버렸다. 18년도 Retrospective 시작한다.

* Work
  * 17년까지 팀 막내였지만 새로운 부서로 이동하여 서비스개발 파트장을 맞게 되었음. 17년도에 개발이라고는 python를 사용하여 vmware vm을 openstack instance로 migration하는 프로젝트 및 kubernetes 기반 tensorflow 플랫폼을 개발하는 초보개발자 수준이였던 내가 인프라와 클라우드 플랫폼(openstack, vsphere, kubernetes) 경력이 인정되어 서비스 개발 파트를 맡게 되었다. 인생에서 가장 큰 변혁중 하나였다고 생각하고 항상 감사하는 마음으로 일을 하고 있다.  
    
  * CNCF OpenSource들을 활용하여 FaaS(Function as a Service), IaC(Infra as Code) 플랫폼 개발을 진행하였다. 서비스 기획 및 두개 프로젝트의 Scrum Master 역할을 수행하였고 메인 업무는 SRE역할로 Kubernetes 설계 구축, 클러스터 및 스토리지 관리, LB, 인증서 등 클라우드 인프라 자원관리 정도였던것 같다. 

* Tech.
  * Public Cloud : AWS, DigitalOcean, IBM Cloud, GCP, Azure, nCloud
  * Provisioning : Harbor, Ansible, Portus, Clair, Kubicorn, Terraform, Vault
  * Runtime : Rook, Minio, OpenEBS, Calico, REX-ray, WeaveNet, Cilium
  * Orchestration : Kubernetes, Envoy, etcd, gRPC, Kong, HAProxy, Istio, Kong, NGINX, Traefik
  * App : NATS, Helm, Kafka, Docker-compose, Draft, Gitlab, minikube, Operator, Jenkins, JenkisX, Packer, RabbitMQ, Spinnaker
  * Serverless : Dispatch, Fission, Knative, Nuclio, OpenWhisk, OpenFaas
  * Monitoring, Logging : Prometheus, Fluentd, Elastic, Grafana, influx, WeaveScope
  * 기타 : Cert-Manager, Dex, Agones

* Speak
  쓰다보니 사내외 발표를 10번을 넘게 한거 같다.
  * 4월 사내 Cloud Tech : 컨테이너 기반 SaaS 개발을 위한 고려사항
  * 5월 사내 DT 솔루션 행사, 클라우드사업부 세션 : Advanced Cloud (Road to Serverless)
  * 5월 사내 통신사업부문 Tech세션 : Road to Serverless (Functions as Applications)
  * 6월 Open Infra Days Korea 2018 : [Provisioning Dedicated Game Server on Kubernetes Cluster](https://www.slideshare.net/JinwoongKim8/provisioning-dedicated-game-server-on-kubernetes-cluster)
  * 8월 SK DNA 2018 : [Cloud Z 의 오픈소스 서비스 소개 및 Serverless로 게임 개발하기](https://www.slideshare.net/JinwoongKim8/cloud-z-serverless-118143924)
  * 10월 DT Labs Meetup : [Continuous Delivery with Spinnaker on K8s(kubernetes) Cluster](https://www.slideshare.net/JinwoongKim8/continuous-delivery-with-spinnaker-on-k8skubernetes-cluster-118140930)
  * 10월 NCsoft IO : [Cloud Native 오픈소스 서비스 소개 및 Serverless로 실제 게임 개발하기](https://www.slideshare.net/JinwoongKim8/cloud-native-serverless/JinwoongKim8/cloud-native-serverless)
  * 11월 IBM Developer Day : [Continuous Delivery using Spinnaker on Kubernetes Multi Cluster](http://public.dhe.ibm.com/software/kr/TrackB/B3.pdf)
  * 11월 "Kubernetes Meetup" 1Day : [Spinnaker on Kubernetes](https://www.slideshare.net/JinwoongKim8/spinnaker-on-kubernetes-123752186)


쓰고나서 보니 2018년도 회고라기 보다 거의 개발에 필요한 잡일(DevOps라고 두둔하면서)과 발표만 한거 같네...
19년도 부터는 다른업무를 하게 될거 같은데 초심을 잃지 말아야 하겠다.

최근 화두가 되었던 `노력중독` 이였던것 같은 18년은 털어버리고
내가 진짜로 좋아하고 하고 싶은 공부와 업무, 그리고 가족을 위해서 로드밸런싱 하면서 지내야 하겠다. 

2006년에 쓴 미래의 내 모습과 어느정도 일치하는거 같지만 목표를 좀더 높이 잡아야할듯 하다.

번역에 참여하거나 책을 써본다던지 커뮤니티 활동을 좀더 적극적으로 해보고
회사에서 시키는 밋업이 아닌 자발적으로 참여하는 외부 강연이나 밋업을 위주로 해봐야겠다. 

또한 점점 개발영역과 멀어지는거 같은 내모습이 나이가 들면서 더 심화될거 같아서 지인들과 별도의 개발 프로젝트를 진행해보려고 한다.

내 13년간 직장 생활중 가장 바쁘고 다이내믹하면서 바빴던 한해이자 많은 경험과 공부가 되었던 일년이였던것 같다.

나를 땡겨주셨던 `June`, 항상 분위기를 리드했던 `Joonbum`, 언제나 열정가득하고 최고의 DevOps 플레이어 `Jeongho`, 든든한 Backender `Donghun`, Excellent/Free/Kind(a.k.a EFK)한 `Minyoung`, 내가 인정하는 Frontier `Sanghun`, Best Rookie `Minsoo`, sorry because the hy `HyunSang`, Moontoring `Jinsoo`, Forever Mentor `Jaemoon`, Guru of CNI `Youngchae` 모두에게 감사의 말씀을 전하고 싶다. (무슨 수상소감도 아니고 ㅋㅋ)

19년에도 남들에게 존경을 받는 사람보다는 무슨일을 하든지 대중에게 욕먹지 않고 필요할때 도움을 요청을 할수 있는 그런 편한 사람이 되어야 겠다.
