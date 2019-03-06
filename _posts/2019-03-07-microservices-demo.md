---
layout: single
title: "Cloud-Native Microservices Demo Application with OpenCensus"
comments: true
description: "Hipster Shop: Cloud-Native Microservices Demo Application 를 살펴보고 안에 들어있는 여러가지 오픈소스들을 알아본다"
keywords: "Kubernetes, Istio, Stackdriver, gRPC, OpenCensus"
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Istio
  - Stackdriver
  - Prometheus
  - gRPC
  - OpenCensus
  - skaffold
---

# Hipster Shop: Cloud-Native Microservices Demo Application
[https://github.com/GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo)에서 소개된 Demo Application인 `Hipster Shop`을 Kubernetes 기반으로 배포하고 관련 오픈소스들을 더 알아보고자 한다. 

위 링크에 가서 보면 알수 있지만 `Hipster Shop` 아래 그림처럼 10개의 Microservice로 구성되어 있고 상품을 검색 및 구매할 수있는 웹 기반 이커머스 Application으로 구성 되어있다.

![arch](https://github.com/GoogleCloudPlatform/microservices-demo/raw/master/docs/img/architecture-diagram.png)

각각의 서비스는 **gRPC** 로 통신하고 외부 사용자만 HTTP로 접근한다. 모든 서비스는 서로 다른 언어(Go, C#, Node.js, Python, Java)로 구성되어 있고 대부분의 Microservice들은 `Istio` service mesh 형태로 구성할수 있도록 되어있다. `Skaffold`를 통해 배포하고 `OpenCensus`라고 하는 gRPC/HTTP기반 Tracing tool을 활용하여 Google `Stackdriver`로 보내도록 되어있지만  Prometheus에 통합하는 방향으로 작성하기 위해서 Prometheus 기반으로 Metric을 수집하는 Fork된 데모 Application을 검색을 통해 찾을수 있었다.  
[https://github.com/census-ecosystem/opencensus-microservices-demo](https://github.com/census-ecosystem/opencensus-microservices-demo)  

## OpenCensus

[OpenCensus(OC)](https://opencensus.io/)는 Microservice 및 Monolith 서비스를 Tracing 및 Metric Monitoring 할 수 있는 라이브러리를 제공하는 오픈소스이다.

아래와 같이 다양한 언어를 지원 한다.
* Java , Go, Python, C++, Nodejs, Erlang, Ruby, PHP, C#

또한 수집된 데이터를 Prometheus, Stackdriver Tracing and Monitoring, DataDog, Graphite, Zipkin, Jaeger, Azure Insights 등 과 같은 백엔드 Application으로 내보낼 수 있기 때문에 개발자, 운영자 측면에서 좋은 선택사항이 될 수 있다.  

Microservice와 같은 분산 시스템에서 개발자/운영자 관점의 가장 중요한 미션은 각각의 수행되는 서비스들의 실행 시간을 확인하고 상호 API간 통신이 얼마나 걸리는지를 확인하고 Span(아래 그림참조)에서 가장 지연이 발생하는 서비스를 빨리 찾아내 확인하고 조치하는 것이라 할 수 있다.

![span](https://www.jaegertracing.io/img/spans-traces.png)

OpenCensus는 주로 두가지 기능으로 활용된다.  
첫번째는 Metric 수집이고 두번째는 Tracing 기능이다.  
Log같은 경우 현재 미지원이지만 다음 메이저 릴리즈에 추가될 예정이라고 하니 조금더 지켜보면 좋을것 같다.

* Metrics
  * 데이터베이스 및 API의 응답시간, 요청 content length, open file descriptor 수와 같이 추적하고자하는 정량 데이터를 말한다. Metric 을 시각화해서 보면 응용 프로그램 및 서비스의 성능과 품질을 측정하는 데 도움이 될 수 있다. 예를 들면 요청 응답시간의 평균값이나 cache hit/miss 수와 같은 것들이 될 수 있다. 

* Traces
  * 서비스 요청에 대한 애플리케이션 또는 서비스 구조를 확인할수 있고 모든 서비스 간 데이터 흐름을 시각화하여 아키텍처상의 병목 현상을 파악하는 데 도움이 된다.

## Hipster Shop Demo

위에서 언급했던 내용처럼 GCP에서 작성한 [Hipster Shop Demo](https://github.com/GoogleCloudPlatform/microservices-demo)는 minikube 및 GCP 데모로 되어있고 코드안에 기본 Metric 설정이 Stackdriver으로 되어있어 Prometheus Exporter 적용을 하려면 코드 수정이 필요하기 때문에 Prometheus기반으로 작성된 [Forked Repo](https://github.com/census-ecosystem/opencensus-microservices-demo)를 살펴보기로 하였다.


### Requirement
현재 가지고 있는 MacOS 환경에서 구동하였다. 최소 스펙은 따로 기재하지 않았으나 K8s 1.11 이상을 권장한다.
* kubectl : v1.10.7
* Minikube : v0.34.1
* Kubernetes : v1.13.3
* [skaffold](https://github.com/GoogleContainerTools/skaffold/#installation) : v0.24

### minikube 기동
최소 3 CPU, 6Gb Memory가 필요하다. 그냥 minikube를 구동시기면 4 CPU, 8Gb 로 구동이 되기 때문에 별다른 옵션 없이 default로 구동하면 된다. 
```
$ minikube start
$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     master    6h        v1.13.3
```

### Repository Clone
```
$ git clone https://github.com/census-ecosystem/opencensus-microservices-demo.git
$ cd opencensus-microservices-demo
```

내부 구조를 살펴보면 기본적으로 [skaffold](https://github.com/GoogleContainerTools/skaffold)를 활용하여 배포를 진행을 하는 것을 알수있다.  
`skaffold`는 로컬에서 Kubernetes 기반 어플리케이션 개발과 배포(CD)를 빠르게 도와주는 CLI tool이다. 소스코드의 변화를 감지하여 build, registry push/tagging, deploy까지 자동으로 할 수 있는 로컬 기반 도구이다.

`skaffold dev`는 로컬 환경의 반복적인 개발에 활용하고 실제 배포는 CI Process에서 `skaffold run`을 통해 배포를 진행할 수 있다.  

![skaffold demo](https://github.com/GoogleContainerTools/skaffold/raw/master/docs/static/images/intro.gif)

Kubernetes 배포툴에 대해 비교한 글은 [블로그 링크](https://blog.hasura.io/draft-vs-gitkube-vs-helm-vs-ksonnet-vs-metaparticle-vs-skaffold-f5aa9561f948/)를 통해 확인할 수 있다.

아래에서는 `skaffold`에 대한 자세한 내용은 미뤄두고 배포하는 도구로서만 설명한다.  

기본적으로 구성을 하고자 하는 내용은 helm처럼 template 파일을 사용하게 되는데 프로젝트 root에 `skaffold.yaml` 에 build를 위한 image name, tag, src 위치등 기본적인 내용을 기재한다. 파일내용을 살펴보면 build에 관련된 내용들을 작성하고 deploy할 manifests의 위치까지 지정하도록 되어있다. 로컬환경에서 확인을 위해 grafana, prometheus, jaeger가 추가된 것을 확인할 수 있다.

```
apiVersion: skaffold/v1alpha2
kind: Config
build:
  tagPolicy:
    gitCommit: {}
  artifacts:
  - imageName: gcr.io/opencensus-microservices-demo/emailservice
    workspace: src/emailservice
  - imageName: gcr.io/opencensus-microservices-demo/productcatalogservice
    workspace: src/productcatalogservice
  - imageName: gcr.io/opencensus-microservices-demo/recommendationservice
    workspace: src/recommendationservice
  - imageName: gcr.io/opencensus-microservices-demo/shippingservice
    workspace: src/shippingservice
  - imageName: gcr.io/opencensus-microservices-demo/checkoutservice
    workspace: src/checkoutservice
  - imageName: gcr.io/opencensus-microservices-demo/paymentservice
    workspace: src/paymentservice
  - imageName: gcr.io/opencensus-microservices-demo/currencyservice
    workspace: src/currencyservice
  - imageName: gcr.io/opencensus-microservices-demo/cartservice
    workspace: src/cartservice
  - imageName: gcr.io/opencensus-microservices-demo/frontend
    workspace: src/frontend
  - imageName: gcr.io/opencensus-microservices-demo/loadgenerator
    workspace: src/loadgenerator
  - imageName: gcr.io/opencensus-microservices-demo/adservice
    workspace: src/adservice
  - imageName: gcr.io/opencensus-microservices-demo/grafana
    workspace: src/grafana
  - imageName: gcr.io/opencensus-microservices-demo/prometheus
    workspace: src/prometheus
  - imageName: gcr.io/opencensus-microservices-demo/jaeger
    workspace: src/jaeger
deploy:
  kubectl:
    manifests:
    - ./kubernetes-manifests/**.yaml
```

Go로 작성된 Frontend microservice을 살펴보자. [**./src/frontend/main.go**]

### library 추가 및 http handler 초기화

Go기반 exporter 패키지(jaeger,prometheus)를 추가적으로 import 하고 http handler를 위한 [ochttp 패키지](https://godoc.org/go.opencensus.io/plugin/ochttp)를 추가하였다. 
```
import (
...
        "go.opencensus.io/exporter/jaeger"
        "go.opencensus.io/exporter/prometheus"
        "go.opencensus.io/plugin/ochttp"
        "go.opencensus.io/plugin/ochttp/propagation/b3"
...
)
func main() {
...
        var handler http.Handler = r
        handler = &logHandler{log: log, next: handler}
        handler = ensureSessionID(handler)      
        // add opencensus instrumentation
        handler = &ochttp.Handler{ 
                Handler:     handler,
                Propagation: &b3.HTTPFormat{}}
        log.Infof("starting server on " + addr + ":" + srvPort)
        log.Fatal(http.ListenAndServe(addr+":"+srvPort, handler))
}
```

### exporter 등록(Jaeger Tracing 및 Prometheus exporter)

예시처럼 각각의 서비스에 jaeger와 prometheus exporter Endpoint를 쉽게 등록할수 있다.  
또한 initTracing() 에서는 데모를 위해 trace.AlwaysSample()을 사용하였다. 실제 운영환경에서는 [다음 링크](https://github.com/census-instrumentation/opencensus-specs/blob/master/trace/Sampling.md)를 참고해서 사용하는 것을 권고하고 있다. 

```
...
func initJaegerTracing(log logrus.FieldLogger) {
        // Register the Jaeger exporter to be able to retrieve
        // the collected spans.
        exporter, err := jaeger.NewExporter(jaeger.Options{
                Endpoint: "http://jaeger:14268",
                Process: jaeger.Process{
                        ServiceName: "frontend",
                },
        })
        if err != nil {
                log.Fatal(err)
        }
        trace.RegisterExporter(exporter)
}

func initTracing(log logrus.FieldLogger) {
        // This is a demo app with low QPS. trace.AlwaysSample() is used here
        // to make sure traces are available for observation and analysis.
        // In a production environment or high QPS setup please use
        // trace.ProbabilitySampler set at the desired probability.
        trace.ApplyConfig(trace.Config{
                DefaultSampler: trace.AlwaysSample(),
        })
        initJaegerTracing(log)
}

func initPrometheusStatsExporter(log logrus.FieldLogger) *prometheus.Exporter {
	exporter, err := prometheus.NewExporter(prometheus.Options{})

	if err != nil {
		log.Fatal("error registering prometheus exporter")
		return nil
	}

	view.RegisterExporter(exporter)
	return exporter
}
func startPrometheusExporter(log logrus.FieldLogger, exporter *prometheus.Exporter) {
	addr := ":9090"
	log.Infof("starting prometheus server at %s", addr)
	http.Handle("/metrics", exporter)
	log.Fatal(http.ListenAndServe(addr, nil))
}
...
```

### Demo Application 배포

minikube에 Hipster Shop Demo를 배포한다. 단순하게 `skaffold run` 명령으로 진행하면 된다.

```
$ skaffold run
```

현재 사용중인 2018 Macbook Pro(3.1 GHz Intel Core i7, 16GB) 상의 Docker기반 minikube 환경으로도 배포를 하였는데 시간이 꽤 소요되었다.(20분이상)  
코드를 실시간으로 수정하고 빌드, 배포되는 것은 `skaffold dev` 명령으로 확인할 수 있다. 진행되는 과정을 보면 [draft.sh](https://draft.sh/) 프로젝트와도 꽤 유사하다고 볼 수 있다.  

에러없이 run이 실행되고 난후 minikube에 배포된 pod와 service를 확인한다. 중간에 loadgenerator가 init인 이유는 minikube 자원이 부족해서 발생하는 현상이다.

```
$ kubectl get pod
NAME                                     READY     STATUS     RESTARTS   AGE
adservice-7c7d687dcb-xzr4m               1/1       Running    1          4h
cartservice-f54bcb9ff-tcfgn              1/1       Running    4          4h
checkoutservice-576446687b-95bwj         1/1       Running    1          4h
currencyservice-5bd99bf97d-28mtz         1/1       Running    1          4h
emailservice-6cb587798d-wwzdh            1/1       Running    1          4h
frontend-6bf9796f7b-ch9pl                1/1       Running    4          4h
grafana-6678c5c5d9-2qx75                 1/1       Running    1          4h
jaeger-5b66bdf7f7-csdzx                  1/1       Running    2          4h
loadgenerator-7c4f446774-68djg           0/1       Init:0/1   1          4h
paymentservice-fc4c8589-wrfg7            1/1       Running    1          4h
productcatalogservice-84878c8b9c-jhgnw   1/1       Running    1          4h
prometheus-58d98b7578-87td6              1/1       Running    1          4h
recommendationservice-8564f9d894-smlpf   1/1       Running    1          4h
redis-cart-798bc66d58-xn6ff              1/1       Running    1          4h
shippingservice-789656f6bc-rgzrp         1/1       Running    1          4h

$ kubectl get svc
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                            AGE
adservice               ClusterIP      10.107.196.115   <none>        9555/TCP,9090/TCP                                                  4h
cartservice             ClusterIP      10.98.151.164    <none>        7070/TCP                                                           4h
checkoutservice         ClusterIP      10.97.9.197      <none>        5050/TCP                                                           4h
currencyservice         ClusterIP      10.103.112.225   <none>        7000/TCP                                                           4h
emailservice            ClusterIP      10.97.184.174    <none>        5000/TCP                                                           4h
frontend                ClusterIP      10.103.40.138    <none>        80/TCP,9090/TCP                                                    4h
frontend-external       LoadBalancer   10.108.170.241   <pending>     80:31944/TCP                                                       4h
grafana                 ClusterIP      10.104.141.254   <none>        3000/TCP                                                           4h
grafana-external        LoadBalancer   10.102.166.138   <pending>     3000:30459/TCP                                                     4h
jaeger                  ClusterIP      10.98.71.173     <none>        9411/TCP,5775/UDP,6831/UDP,6832/UDP,5778/TCP,16686/TCP,14268/TCP   4h
jaeger-external         LoadBalancer   10.96.164.126    <pending>     16686:32362/TCP                                                    4h
kubernetes              ClusterIP      10.96.0.1        <none>        443/TCP                                                            6h
paymentservice          ClusterIP      10.109.31.241    <none>        50051/TCP                                                          4h
productcatalogservice   ClusterIP      10.101.124.106   <none>        3550/TCP                                                           4h
prometheus              ClusterIP      10.103.107.213   <none>        9090/TCP                                                           4h
recommendationservice   ClusterIP      10.104.225.28    <none>        8080/TCP                                                           4h
redis-cart              ClusterIP      10.101.24.157    <none>        6379/TCP                                                           4h
shippingservice         ClusterIP      10.104.224.18    <none>        50051/TCP                                                          4h
```

### 서비스 접속 및 Metric/Tracing 확인
로컬 minikube환경이기 때문에 external service가 pending이므로 service를 minikube NAT IP로 expose 시킨다.

```
$ minikube service frontend-external
$ minikube service grafana-external
$ minikube service jaeger-external
$ minikube service list
|-------------|-----------------------|-----------------------------|
|  NAMESPACE  |         NAME          |             URL             |
|-------------|-----------------------|-----------------------------|
| default     | adservice             | No node port                |
| default     | cartservice           | No node port                |
| default     | checkoutservice       | No node port                |
| default     | currencyservice       | No node port                |
| default     | emailservice          | No node port                |
| default     | frontend              | No node port                |
| default     | frontend-external     | http://192.168.99.101:31944 |
| default     | grafana               | No node port                |
| default     | grafana-external      | http://192.168.99.101:30459 |
| default     | jaeger                | No node port                |
| default     | jaeger-external       | http://192.168.99.101:32362 |
| default     | kubernetes            | No node port                |
| default     | paymentservice        | No node port                |
| default     | productcatalogservice | No node port                |
| default     | prometheus            | No node port                |
| default     | recommendationservice | No node port                |
| default     | redis-cart            | No node port                |
| default     | shippingservice       | No node port                |
| kube-system | kube-dns              | No node port                |
|-------------|-----------------------|-----------------------------|
```

3개의 서비스로 각각 접속이 되는것을 확인할수 있다.
Grafana 대시보드로 들어가면 현재 수집되는 prometheus source(http://prometheus:9090)를 통해 OpenCensus기반 Application Metric을 확인할 수 있다.

![hipster_grafana](/images/hipster_grafana.png)

그림에서 보면 전체 서비스 응답에 대한 99% 백분위 지연시간이 944ms 인것을 확인할 수 있다. 

또한 Jaeger를 통해 DAG(Directed acyclic graph) 및 서비스간 Tracing 을 확인할 수 있다.

![DAG](/images/dag.png)

![tracing](/images/tracing.png)

## 정리
OpenCensus 기반으로 개발자가 코드를 작성하고 Microservice를 minikube에서 배포하고 Prometheus, Jaeger Exporter 연동을 통해 시스템뿐만이 아닌 Application기반 Metrics/Stats을 수집하고 개발자가 작성한 코드를 직접 Tracing하는 간단한 데모를 진행하였다. (Istio를 포함해서 Public환경에 배포해봐도 좋은 공부가 될 것 같다)  

향후 [OpenMetric](https://openmetrics.io/)과 [Opencensus](https://opencensus.io/)가 실제 개발자 기반으로 활성화되고 적용이 된다면 Telemetric 측면에서 많은 Use-Case가 도출될 수 있을것 같다.  

위에서 언급했듯이 Prometheus기반 Kubernetes 클러스터를 운영하고 있는 팀의 경우 개발자의 작성 코드를 최소화할 수 있는 도구로서 충분히 활용될 수 있어 보인다.  

꼭 Cloud Native 기반 Web 개발이 아니더라도 기존 공장, 금융, 병원 등 의 IoT나 센서/설비를 위한 비즈니스에도 Backend로서 확장성있는 도구로서 활용이 될 수 있을것 같다.