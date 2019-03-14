---
layout: single
title: "Cert-manager"
comments: true
classes: wide
description: "Cert-manager에 대해 알아봅니다"
keywords: "Cert-manager, Kubernetes, k8s, TLS, Manage TLS Certificates, Security, HTTPS, Secure HTTP"
categories:
  - Kubernetes
tags:
  - Cert-manager
  - Kubernetes
  - TLS
  - Manage TLS Certificates
  - Security
  - HTTPS
---

## TLS 인증서

제한된 예산과 시간, 인력으로 서비스를 만들다 보니 어려운 점들이 참 많다. 특히나 TLS 인증서 같은 경우 관리하는것이 은근히 귀찮다. 인증서 만료일 관리하는것 부터 어느 페이지까지를 평문으로만 두어야 하는지도... 그런데 바보같은 이슈를 저지르고 말았다. 도메인을 신청하고 fanout, Name based virtual hosting 방식중에 고민하다가 설계를 virtual host로 하였다.  

그런데 문제는 어이없는곳에서 나왔다. *.xxxxx.io 라고 wildcard SSL 인증서를 신청하였는데 virtual host방식으로 2차 subdomin (예, api.stg.xxxxx.io)으로 ingress를 마구잡이로 생성하고 인증서를 적용을 했다. 하지만 인증서 오류 ㅋㅋ

인터넷 비지니스를 거의 해보지 않아서 인지, 이런 기본적인 내용도 간과하고 서비스를 구상한 내가 모든걸 책임져야하는 상황이 와버렸다. 

설계를 이상하게 해서 wildcard 도메인을 몇십개를 신규로 계약해야하는 쓸데없는 비용을 들어야하는 처지가 되었다. 하지만 그냥 죽으란 법은 없는게 사람사는 세상이고 이런 고민들을 분명 다른사람들도 했을거야라고 생각하면서 떠오른게 let's encrypt 였고 찾아보니 이걸 또 kubernetes에서 발급부터 갱신까지 해주는 오픈소스를 찾아 적용까지 하고 이렇게 포스팅을 한다. 


### Cert-manager

인증서 없이 구성할수도 있지만 기본적으로 kubernetes 는 secret과 ingress의 간단한 설정만으로 TLS를 구성할 수 있다.  


* Cert-manager : K8s 클러스터내에서 TLS인증서를 자동으로 프로비저닝 및 관리하는 오픈소스
* Let’s Encrypt :  자율적으로 작동하는 개방된 CA(Certificate authority-인증기관)으로 공공성(공공의 이익)을 위해서 운영되는 오픈소스



[Cert-manager](http://docs.cert-manager.io/en/latest/)는 AWS의 Certificate Manager와 유사하게 인증서를 발급하고 설정할수 있다. 3개월 유효기간을 가지는 [let's encrypt](https://letsencrypt.org/)를 사용하기에 인증서 만료시 문제가 될거라 생각했지만 이것도 자동으로 갱신을 해주는 관리 app.을 같이 배포하기 때문에 인증서 관리에 대한 부담을 덜수 있다.  

기본적으로 Cert-manager는 [let's encrypt](https://letsencrypt.org/), [Vault](https://www.vaultproject.io/) 같은 것을들 사용하여 인증서 발급을 할 수 있다. 인증서 발급후에도 만료가 되기 전에 바로 갱신을 자동으로 한다. 


### 구성
![certmanager](http://docs.cert-manager.io/en/latest/_images/high-level-overview.png)

### 사전준비사항
* [helm](https://docs.helm.sh/using_helm/#installing-helm-client)
* k8s cluster (1.7+) - CRD 사용이 가능해야함

### 설치
```
$ helm install --name cert-manager --version v0.3.1 \
    --namespace kube-system stable/cert-manager

$ helm status cert-manager
LAST DEPLOYED: Fri Jun 15 10:02:45 2018
NAMESPACE: kube-system
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                                        READY  STATUS   RESTARTS  AGE
cert-manager-cert-manager-5f76b676b4-5tdh8  1/1    Running  0         2d

==> v1/ServiceAccount
NAME                       SECRETS  AGE
cert-manager-cert-manager  1        28d

==> v1beta1/CustomResourceDefinition
NAME                               AGE
certificates.certmanager.k8s.io    28d
clusterissuers.certmanager.k8s.io  28d
issuers.certmanager.k8s.io         28d

==> v1beta1/ClusterRole
cert-manager-cert-manager  28d

==> v1beta1/ClusterRoleBinding
NAME                       AGE
cert-manager-cert-manager  28d

==> v1beta1/Deployment
NAME                       DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
cert-manager-cert-manager  1        1        1           1          28d


NOTES:
cert-manager has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.readthedocs.io/en/latest/reference/issuers.html

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.readthedocs.io/en/latest/reference/ingress-shim.html
```

### 인증서 설정 및 배포

인증서 등록 및 만료시 noti를 받기 위한 변수 설정을 한다.
```
$ export EMAIL=zaction@sk.com
```
그리고 issuer manifest를 배포한다.  
(letsencrypt-issuer.yaml)
```
# Copyright 2018 Google Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: ''
    privateKeySecretRef:
      name: letsencrypt-staging
    http01: {}
---
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ''
    privateKeySecretRef:
      name: letsencrypt-prod
    http01: {}
```

두개의 endpoint를 생성한다. (Staging, Prod)

```
$ sed -e "s/email: ''/email: $EMAIL/g" letsencrypt-issuer.yaml | \
    kubectl apply -f-
clusterissuer "letsencrypt-staging" created
clusterissuer "letsencrypt-prod" created
```

Web App을 HTTP Ingress 형태로 배포한다. TLS발급전에 미리 HTTP(80) Ingress를 배포해야 한다. 이후 Let's encrypt TLS배포 후 TLS spec을 추가하고 재배포를 해야한다. 

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: auth-ingress
spec:
  rules:
  - host: auth.dev.action.cloudz.co.kr
    http:
      paths:
      - backend:
          serviceName: faas-auth-service
          servicePort: 8081
        path: /
``` 

### TLS 인증서 발급
인증서를 발급할때는 위에서 만든 Ingress로 생성하고namespace별 생성이 필요하다.

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: auth-dev-tls
  namespace: auth #### namespace별 생성
spec:
  secretName: auth-dev-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: auth.dev.action.cloudz.co.kr
  dnsNames:
  - auth.dev.action.cloudz.co.kr
  acme:
    config:
    - http01:
        ingress: auth-ingress ### 기존에 만든 HTTP ingress 명
      domains:
      - auth.dev.action.cloudz.co.kr
```

```
$ kubectl apply -f auth-dev-tls.yaml
certificate "auth-dev-tls" created
```

이후 TLS 생성 상태를 확인한다. 이때 Message에서 ```Normal   CeritifcateIssued     Certificated issued successfully``` 문구가 확인되면 인증서 발급이 정상적으로 되었다고 볼 수 있다.  
이때 발급되는 시간이 네트워크 환경에 따라 1~5분정도 소요될때로 있는것으로 보인다. 
``` 
$ kubectl describe -f auth-dev-tls.yaml
...
Type     Reason                Message
----     ------                -------
Warning  ErrorCheckCertificate Error checking existing TLS certificate: secret "auth-dev-tls" not found
Normal   PrepareCertificate    Preparing certificate with issuer
Normal   PresentChallenge      Presenting http-01 challenge for domain foo.kubernetes.tips
Normal   SelfCheck             Performing self-check for domain auth.dev.action.cloudz.co.kr
Normal   ObtainAuthorization   Obtained authorization for domain auth.dev.action.cloudz.co.kr
Normal   IssueCertificate      Issuing certificate...
Normal   CeritifcateIssued     Certificated issued successfully
```

secret이 생성된것을 확인할 수 있다. 

```
$ kubectl get secret | grep tls
auth-dev-tls                 kubernetes.io/tls                     2         3h
```

이후 위에서 만든 Ingress에 TLS를 적용한다.
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: auth-ingress
  annotations:
      ingress.bluemix.net/hsts: enabled=true maxAge=<31536000> includeSubdomains=true
      ingress.bluemix.net/redirect-to-https: "True"
spec:
  rules:
  - host: auth.dev.action.cloudz.co.kr
    http:
      paths:
      - backend:
          serviceName: faas-auth-service
          servicePort: 8081
        path: /
  tls:
  - hosts:
    - auth.dev.action.cloudz.co.kr
    secretName: auth-dev-tls
```

위 설정을 적용하고 상태를 확인한다.
```
$ kubectl apply -f auth-tls-ingress.yaml

$ kubectl get ing
NAME           HOSTS                          ADDRESS         PORTS     AGE
auth-ingress   auth.dev.action.cloudz.co.kr   10.1.1.182   80, 443   14h
```

![cert-tls](/images/cert-tls.png)

## 정리
설계의 실수로 시작되었지만 정리하다보니 인증서 갱신이 자동을 된다는 점과 또한 Kubernetes환경에서 TLS관리가 간편하다는것을 알게 되었다. 인증서 비용을 아낀것도 하나의 성과이기도 하다. 인증서는 써야하겠고 정말 많은 애플리케이션들을 생성/삭제를 상시 해야하는 환경이라면 한번 적용해볼만한 가치가 있는것 같다.
