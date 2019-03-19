---
layout: single
title: "VScode Server"
comments: true
classes: wide
description: "code-server를 사용하여 로컬 VScode 설치없이 원격 환경에서 사용해보자."
keywords: "vscode"
categories:
  - Tools
tags:
  - vscode
  - code-server
  - kubernetes
  - online coding
  - GoormIDE
  - Cloud9
---

IntelliJ IDEA를 사용하다가 개발업무를 거의 손놓다시피 하다보니 라이센스를 연장하지 못하게 되었고 주로 Local에서 VScode를 사용하고 있다. 현재 부서와 업무도 바뀌었고 워낙 고가다보니 IDE를 부서비로 구매하도 어렵다. 뭐 얼마나 한다고 말할수도 있겠지만 업무특성상 개발툴을 사달라고 할수 없고 최근 크롬북이나 아이패드 프로같은 태블릿을 가지고 다니시는 분들도 많고 단순 필기나 메모가 아닌 온라인 환경에서 블로깅 포스팅 정도는 할수 있다는 가정으로 Web IDE를 구성하는 포스팅을 시작해본다. 

## Web IDE
이번 포스팅을 쓰기 시작하면서 2017년 AWS re;invent에선가 Cloud9 제품이 출시되어서 Lambda에서 사용했던 장면이 갑자기 떠올랐다. Cloud 9은 [https://github.com/c9/core](https://github.com/c9/core)를 기반으로 instance를 띄는것을 기본으로 한다. Lambda의 기본 에디터도 나쁘진 않지만 Cloud9에서 강조하는 부분은 코드 협업이다. 

![cloud9](https://d1.awsstatic.com/product-marketing/Tulip/C9-Collab-Image@3x.e03a65d9488633c154358430540ab363dd1e8f45.png)

AWS에서 서비스로 출시되기 이전에도 Dockerhub에서도 종종 확인할수 있었지만 현재는 찾아보기 어렵니다. 

국내에는 Cloud 형태로 [GoormIDE](https://ide.goorm.io/) 제품이 있다.  
Free 에디션도 있으니 따로 확인해보면 된다. (응원합니다!)

이외에도 [https://github.com/theia-ide/theia](https://github.com/theia-ide/theia), [https://github.com/codercom/code-server](https://github.com/codercom/code-server)와 같은 Web기반 오픈소스가 존재하고 둘다 상용으로 제공되거나 Beta형태로 준비중이다. 

## code-server

[code-server](https://github.com/codercom/code-server) 는 원격서버 형태로 동작하는 브라우저 기반 [VSCode](https://github.com/Microsoft/vscode) IDE이다.

그런데 왜 구지 VScode 설치형을 냅두고 Server로 구동하느냐? 아래와 같이 설명하고 있다.
* Chromebook, Table(IPAD) 에서 Coding 가능
* 저사양 Client에서도 Cloud 기반 Server의 이점 사용가능
* Battery! (제일 중요포인트) 

구동방식은 여러방식이 존재한다. 
* Hosted : [coder](https://coder.com/) - Enterprise 
* Docker
    ```
    $ docker run -t -p 127.0.0.1:8443:8443 -v "${PWD}:/root/project" codercom/code-server code-server --allow-http --no-auth
    ```
* Bianry
    ```
    $ code-server <initial directory to open>
    ```

## code-server kubernetes에 배포

공식 [Image](https://hub.docker.com/r/codercom/code-server)와 [Dockerfile](https://github.com/codercom/code-server/blob/master/Dockerfile)는 해당 링크를 참조한다.
살짝 변경해서 재빌드하려고 했는데 맥북 메모리가 부족한지 `yarn build`할때 `Serve fails with Error Code 137` 가 발생하였다. 이문제는 나중에 해결하기로 하고 일단 배포하자.  

로컬 minikube에서 테스트하기 위해  
[https://github.com/codercom/code-server/blob/master/deployment/deployment.yaml](https://github.com/codercom/code-server/blob/master/deployment/deployment.yaml) 에서 ClusterIP를  NodePort로 수정하고 패스워드를 로그에서 확인하지 않고 사용할수 있도록 `1234`로 설정하고 배포한다. 

### deployment.yaml
```
apiVersion: v1
kind: Namespace
metadata:
  name: code-server
---
apiVersion: v1
kind: Service
metadata:
 name: code-server
 namespace: code-server
spec:
 ports:
 - port: 8443
   name: https
   protocol: TCP
 selector:
   app: code-server
 type: NodePort
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: code-server
  name: code-server
  namespace: code-server
spec:
  selector:
    matchLabels:
      app: code-server
  replicas: 1
  template:
    metadata:
      labels:
        app: code-server
    spec:
      containers:
      - image: codercom/code-server
        imagePullPolicy: Always
        name: code-server
        ports:
        - containerPort: 8443
          name: https
        args:
         - "--password=1234"
```
```
$ kubectl create -f deployment.yaml
```

이후 서비스 확인 및 minikube service로 expose 시킨다.

```
$ kubectl get svc
NAME          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
code-server   NodePort   10.111.206.64   <none>        8443:32665/TCP   15m

$ minikube service code-server

$ minikube service list
|-------------|---------------|-----------------------------|
|  NAMESPACE  |     NAME      |             URL             |
|-------------|---------------|-----------------------------|
| code-server | code-server   | http://192.168.99.102:32665 |
| default     | git-sync-demo | http://192.168.99.102:31595 |
| default     | kubernetes    | No node port                |
| kube-system | kube-dns      | No node port                |
|-------------|---------------|-----------------------------|
```

해당 URL로 접속하여 패스워드 `1234`를 입력하면 vscode가 원격으로 실행된 것을 확인할수 있다.
![vscode](/images/vscode.png)

## 정리
실제 사용을 해보면 아직 버그가 많다. 애드온이나 플러그인 설치시 제대로 동작을 하지 않는 경우도 있었고 
[git-sync](https://github.com/kubernetes/git-sync) 프로젝트와 연동을 통해 실제 gitOPS를 구현해보고자 하였으나 배포후에 mount된 repository volume에 변경사항이 발생시 갑자기 UI가 먹통이 되는 경우가 발생하기도 하였다. Kubernetes플랫폼을 개발자에게 PaaS형태로 제공하는 경우 webIDE의 좋은 옵션이 될수 있을것 같다. 