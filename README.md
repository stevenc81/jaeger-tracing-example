# When Istio Meets Jaeger - An Example of End-to-end Distributed Tracing

Kubernetes is great! It helps many engineering teams to realize the dream of SOA (Service Oriented Architecture). For the longest time, we build our applications around the concept of monolith mindset, which is essentially having a large computational instance running all services provided in an application. Things like account management, billing, report generation are all running from a shared resource. This worked pretty well until SOA came along and promised us a much brighter future. By breaking down applications to smaller components, and having them to talk to each other using REST or gRPC. We ~~hope~~ expect things will only get better from there but only to realize a new set of challenges awaits. How about cross services communication? How about observability between microservices such as logging or tracing? This post demonstrates how to set up OpenTracing inside a Kubernetes cluster that enables end-to-end tracing between services, and inside the service itself with a right instrumentation.

# Kubernetes Cluster Setup
First thing first, we will need to have a working Kubernetes cluster to play with. I use [kops](https://github.com/kubernetes/kops) on AWS because it offers an array of automated cluster operations like upgrade, scaling up/down and multiple instance groups. In addition to the handy cluster opertations, the kops team has been tailing latest Kubernetes version closely to provide up-to-date cluster experience. I found that very cool :D

To get kops up and running there are [some steps](https://github.com/kubernetes/kops/blob/master/docs/aws.md) to follow.

## Create Cluster
Creating a Kubernetes cluster with kops on AWS could be as simple as an one [cluster create command](https://github.com/kubernetes/kops/blob/master/docs/cli/kops_create_cluster.md).

```
kops create cluster \
--name steven.buffer-k8s.com \
--cloud aws \
--master-size t2.medium \
--master-zones=us-east-1b \
--node-size m5.large \
--zones=us-east-1a,us-east-1b,us-east-1c,us-east-1d,us-east-1e,us-east-1f \
--node-count=2 \
--kubernetes-version=1.8.6 \
--vpc=vpc-1234567a \
--network-cidr=10.0.0.0/16 \
--networking=flannel \
--authorization=RBAC \
--ssh-public-key="~/.ssh/kube_aws_rsa.pub" \
--yes
```

This command tells AWS to create a Kubernetes cluster with 1 master node and 2 minion (worker) nodes in `us-east-1` on VPC `1234567a` using CIDR `10.0.0.0/16`. It will take around 10 minutes for the cluster to spin up and ready to use. In the meantime you could use `watch kubectl get nodes` to monitor the progress.

Once completed, we should be ready to install [Istio](https://istio.io/) on the new Kubernetes cluster. It's a service mesh that manages traffic between services running on the same cluster. Because of this nature it makes Istio a perfect candidate to trace requests across services. 

# Install Istio
Download Istio from their [GitHub repo](https://github.com/istio/istio/releases/tag/0.4.0).

From the downloaded Istio directory you could install Istio to the Kubernetes cluster with this command

`kubectl apply -f install/kubernetes/istio.yaml`

Now Istio should be up and running on the cluster. It aslo creates a Nginx Ingress Controller that takes external requests. We will include how to set ip up later.

# Install Jaeger
Jaeger and Instio work side by side to provide tracing across services. You could insall Jaeger with this command

`kubectl create -n istio-system -f https://raw.githubusercontent.com/jaegertracing/jaeger-kubernetes/master/all-in-one/jaeger-all-in-one-template.yml`

After completed, you should be able to access the Jaeger UI. Viola!

![Jaeger](https://dha4w82d62smt.cloudfront.net/items/3v2P1z1S1R243e2o2y3f/Image%202018-01-27%20at%2012.39.25%20PM.png?X-CloudApp-Visitor-Id=2456693)

# Instrument Code
After installing Jaeger and Istio you will be able to see cross services traces automajically! This is because Envoy sidecars injected by Istio handle inter-service traffic, while the deployed application only talks to the assigned sidecar.

You can find my [GitHub repo](https://github.com/stevenc81/jaeger-tracing-example) with a small sample application.

`main.go` looks like this

```
package main

import (
	"fmt"
	"log"
	"net/http"
	"runtime"
	"time"

	"github.com/opentracing-contrib/go-stdlib/nethttp"
	opentracing "github.com/opentracing/opentracing-go"
	jaeger "github.com/uber/jaeger-client-go"
	"github.com/uber/jaeger-client-go/zipkin"
)

func indexHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "hello world, I'm running on %s with an %s CPU ", runtime.GOOS, runtime.GOARCH)
}

func getTimeHandler(w http.ResponseWriter, r *http.Request) {
	log.Print("Received getTime request")
	t := time.Now()
	ts := t.Format("Mon Jan _2 15:04:05 2006")
	fmt.Fprintf(w, "The time is %s", ts)
}

func main() {
	zipkinPropagator := zipkin.NewZipkinB3HTTPHeaderPropagator()
	injector := jaeger.TracerOptions.Injector(opentracing.HTTPHeaders, zipkinPropagator)
	extractor := jaeger.TracerOptions.Extractor(opentracing.HTTPHeaders, zipkinPropagator)

	// Zipkin shares span ID between client and server spans; it must be enabled via the following option.
	zipkinSharedRPCSpan := jaeger.TracerOptions.ZipkinSharedRPCSpan(true)

	sender, _ := jaeger.NewUDPTransport("jaeger-agent.istio-system:5775", 0)
	tracer, closer := jaeger.NewTracer(
		"myhelloworld",
		jaeger.NewConstSampler(true),
		jaeger.NewRemoteReporter(
			sender,
			jaeger.ReporterOptions.BufferFlushInterval(1*time.Second)),
		injector,
		extractor,
		zipkinSharedRPCSpan,
	)
	defer closer.Close()

	http.HandleFunc("/", indexHandler)
	http.HandleFunc("/gettime", getTimeHandler)
	http.ListenAndServe(
		":8080",
		nethttp.Middleware(tracer, http.DefaultServeMux))
}

```

From line 28 - 30 we create a Zipkin propagator to tell Jaeger to capture OpenZipkin context from request headers. You might ask how these headers get to a request in the first place? Remember when I said Istio sidecar handles service communication and you application only talks to it? Yeah, you might have guessed it already. In order for Istio to trace a request between services, a set of headers are injected by Istio's Ingress Controller when a request enters the cluster. It then gets prapagated arond Envoy sidecars and each one reports the associated span to Jaeger. This helps connecting the spans to a single trace. Our application code takes advantage of these headers to collapse the inter-service with inner-service spans.

Here is a list of OpenZipkin headers injected by Istio Ingress Controller

```
x-request-id
x-b3-traceid
x-b3-spanid
x-b3-parentspanid
x-b3-sampled
x-b3-flags
x-ot-span-context
```

To deploy the sample applciation you could use this yaml file

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: myhelloworld
  name: myhelloworld
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: myhelloworld
    spec:
      containers:
      - image: stevenc81/jaeger-tracing-example:0.1
        imagePullPolicy: Always
        name: myhelloworld
        ports:
        - containerPort: 8080
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: myhelloworld
  labels:
    app: myhelloworld
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    name: http
  selector:
    app: myhelloworld
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myhelloworld
  annotations:
    kubernetes.io/ingress.class: "istio"
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: myhelloworld
          servicePort: 80
      - path: /gettime
        backend:
          serviceName: myhelloworld
          servicePort: 80
```

# See the Traces
Time to reap profit! When we send a request to the Istio Ingress Controller it will be traced between services as well as inside the application. From the screenshot we could see 3 spans reported from different places

* Ingress Controller
* Envoy sidecar for application
* Application code

![Jaeger](https://dha4w82d62smt.cloudfront.net/items/0c0w373p3J3V1g3Q2B3J/Image%202018-01-27%20at%2012.48.23%20PM.png?X-CloudApp-Visitor-Id=2456693)

Expand the trace (to show 3 spans) and see the end-to-end tracing we shall :D
![Trace](https://dha4w82d62smt.cloudfront.net/items/1W03470s3T402V0d3Q3G/Image%202018-01-27%20at%2012.50.15%20PM.png?X-CloudApp-Visitor-Id=2456693)


# Closing Words
* SOA brings whole set of new issues, especially around service observability
* Istio + Jager integration solves it on the service to service level
* Using OpenZipkin prapagator with Jaeger allows for true end-to-end tracing
