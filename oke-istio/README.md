
# Istio installation in Oracle Container Engine for Kubernetes
This recipe details how to install [Istio](https://istio.io/docs/concepts/what-is-istio/) in [OKE](https://cloud.oracle.com/containers/kubernetes-engine) with [Prometheus](https://prometheus.io/), [Grafana](https://grafana.com/), [Jaeger](https://www.jaegertracing.io/) and [Kiali](https://www.kiali.io/). The recipe also shows how to enable TLS security using [Secret Discovery](https://preliminary.istio.io/docs/tasks/traffic-management/secure-ingress/sds/).

> Note that the following Istio installation is based on [Helm](https://istio.io/docs/setup/kubernetes/install/helm/)
> using Mac.

> [Check this link](https://github.com/luisw19/oci-series/tree/master/oke-install) to provision a new Kubernetes Cluster in OCI if an instance is not already available.

> This installation has been tested with [Istio version 1.1](https://istio.io/about/notes/1.1/).

**1)** Grant the *Kubernetes RBAC cluster-admin clusterrole* to a OCI user based on
the user's *Oracle Cloud Identifier (OCID)*.

> To obtain the OICD open the OCI Console and from there
> click on the menu option *Identity > Users*.
> Then click on *show* under the username and take note of the OID.

```bash
kubectl create clusterrolebinding sttc_admin --clusterrole=cluster-admin --user=ocid1.user.oc1..aaaaaaaazhciwyt5kooopvnovupyao7v7a73imsvxoqrb2omojbcvcxpgvrq
```

**2)** Download Istio’s [latest release](https://istio.io/docs/setup/kubernetes/download/)

- First decide on which location you wish to download Istio then optionally create a folder for the download.

```bash
cd <location where Istio will be downloaded>
mkdir istio
cd istio
```

- Then download with the following command:

```bash
	curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.1.0 sh -
```

> This will download istio into the folder **./istio-1.1.0**

**3)** Set **$ISTIO_HOME** and add the **istioctl** to the path in case required.

- From the same location where Istio was downloaded, execute the following:

```bash
cd istio-1.1.0
export ISTIO_HOME=$PWD
export PATH=$PWD/bin:$PATH
```

> **istioctl** is used when manually injecting Envoy as a sidecar proxy.

**4)** [Helm installation](https://helm.sh/) and [Tiller](https://helm.sh/docs/glossary/#tiller) installation:

- Ensure **Tiller** is already installed on the server with the command:

> in Oracle Container Engine for Kubernetes Tiller can be installed
> during [service is provisioning](https://docs.cloud.oracle.com/iaas/Content/ContEng/Tasks/contengcreatingclusterusingoke.htm)

```bash
kubectl get pods -n kube-system | grep tiller
```

Result from the above command should be similar to:

```bash
$kubectl get pods -n kube-system | grep tiller
tiller-deploy-54885b67b5-pkjnk          1/1     Running   0          26
```

- Also make sure that that a service account with the cluster-admin role has been defined for Tiller.
This can be done by running the commands:

```bash
kubectl -n kube-system get serviceaccount tiller
kubectl -n kube-system get clusterrolebinding tiller
```

> If any of the responses contains **Error from server (NotFound)** it means the service account and cluster role
> has to be applied. This can be done by running: `kubectl apply -f $ISTIO_HOME/install/kubernetes/helm/helm-service-account.yaml`

- Ensure **Helm** is installed on the client. In Mac this can be easily done with brew

```bash
brew install kubernetes-helm
```

- Then **initialised** amd **upgrade** and ensure version corresponds to the server:

```bash
helm init --upgrade
```

Verity that both client and server versions are consistent by running:

```bash
helm version
```

Response should be similar to:

```bash
Client: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
```

**5)** Create the **istio-system** namespaces

```bash
kubectl create namespace istio-system
```

**6)** Create a secret for **Kiali**

> This is required as Kiali expects the secret to be present.

- Create a **base64** encoded **username** and **password**

```bash
$ echo -n 'user' | base64
dXNlcg==
$ echo -n 'sample' | base64
c2FtcGxl
 ```

 - Then use it to create a secret:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: kiali
  namespace: istio-system
  labels:
    app: kiali
type: Opaque
data:
  username: YWRtaW4=
  passphrase: V2VsY29tZTE=
EOF
```

**7)** Install the **istio-init** Helm chart to bootstrap all the Istio’s [Custom Resource Definitions (CRDs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions)

```bash
helm install $ISTIO_HOME/install/kubernetes/helm/istio-init --name istio-init --namespace istio-system \
--set certmanager.enabled=true
```

Verify that all **58** CRDs were created:

```bash
kubectl get crds | grep 'istio.io\|certmanager.k8s.io' | wc -l
```

> Note that it may take a few seconds to complete so you may need to
> run the command multiple times.

**8)** Install Istio with **Helm** the desired confirmation profile.

- To install with Istio with the
[Prometheus](https://istio.io/docs/tasks/telemetry/metrics/querying-metrics/),
[Jaeger](https://istio.io/docs/tasks/telemetry/distributed-tracing/jaeger/),
[Grafana](https://istio.io/docs/tasks/telemetry/using-istio-dashboard/) and
[Kiali](https://istio.io/docs/tasks/telemetry/kiali/) add-ons install as following:

```bash
helm install $ISTIO_HOME/install/kubernetes/helm/istio --name istio --namespace istio-system \
--set sds.enabled=true \
--set gateways.istio-ingressgateway.sds.enabled=true \
--set pilot.traceSampling=100.0 \
--set pilot.resources.requests.cpu=1800m \
--set telemetry-gateway.grafanaEnabled=true \
--set telemetry-gateway.prometheusEnabled=true \
--set grafana.enabled=true \
--set tracing.enabled=true \
--set tracing.jaeger.ingress.enabled=true \
--set kiali.enabled=true \
--set kiali.dashboard.jaegerURL='http://localhost:16686' \
--set kiali.dashboard.grafanaURL='http://grafana.istio-system:3000'
```

> [Click here](https://istio.io/docs/reference/config/installation-options/) for details of more options available.

> Note that Istio 1.1.10 also comes with an add-on for [LightStep [𝑥]PM](https://lightstep.com/). The configuration of this tool wasn't covered in this recipe as it goes beyond the main installing Istio installation also requiring for example the creation of a trial account in the Lightstep website, configuration of Satellite containers and others. [Click here for more info how how to set it up](https://istio.io/docs/tasks/telemetry/distributed-tracing/lightstep/).

- Verity all that **pods** are **running** run:

```bash
kubectl get pods -n istio-system
```

- Also Verify that  services were created and that an **EXTERNAL-IP** has been allocated to **istio-ingressgateway**:

```bash
kubectl get svc -n istio-system
```

> Repeat the above process until an **EXTERNAL-IP** is assigned.
> Note that this may take a few seconds as the controller is
> basically creating an **OCI load balancer** also visible from the
> OCI console itself under **Networking > Load Balancers**.

- To **uninstall** Istio:

```bash
helm delete --purge istio
helm delete --purge istio-init
```

To delete all running CRDs execute:

```bash
for i in $ISTIO_HOME/install/kubernetes/helm/istio-init/files/*crd*yaml; do kubectl delete -f $i; done
```

> note that a **(NotFound)** error will be shown after
> running the command. This is because Certificate Manager
> wasn't enabled during the installation so it can be ignored.

And finally the namespace:

```Bash
kubectl delete namespace istio-system
```

**9)** For Istio [automatic injection](https://istio.io/docs/setup/kubernetes/sidecar-injection/#automatic-sidecar-injection) of **Envoy side-cars** to work the label **istio-injection=enabled** has to be set to the **target namespaces** as following:

- Create a **namespaces**

```bash
kubectl create namespace httpbin-istio
```

- Set the **label**

```bash
kubectl label namespace httpbin-istio istio-injection=enabled
```

- To verity that the **label** was properly applied run:

```bash
kubectl get namespace -L istio-injection
```

Result should be something like:

```bash
NAME            STATUS   AGE    ISTIO-INJECTION
...
httpbin-istio   Active   25s   enabled
```

**10)** From the same **namespace** where **istio-injection** was enabled, deploy the **Pods** and verify that the **Istio side-cars** are being attached correctly:

- Based on [this Istio sample](https://istio.io/docs/tasks/traffic-management/ingress/), deploy the [HTTPBIN](https://httpbin.org) **manifest** as following:

First get the latest **manifest** file:

```bash
curl -O https://raw.githubusercontent.com/istio/istio/release-1.0/samples/httpbin/httpbin.yaml
```

Then apply the **manifest**:

```bash
kubectl apply -n httpbin-istio -f httpbin.yaml
```

- Verity that the **side-car** was attached to the pod:

```bash
kubectl describe pod $(kubectl get pods -n httpbin-istio -o jsonpath='{.items[0].metadata.name}') -n httpbin-istio | grep istio-proxy
```

The result should be something like:

```bash
Annotations:       sidecar.istio.io/status:
                  {"version":"887285bb7fa76191bf7f637f283183f0ba057323b078d44c3db45978346cbc1a","initContainers":["istio-init"],"containers":["istio-proxy"]...
```

**11)** Create the Istio [Ingress Gateway](https://istio.io/docs/tasks/traffic-management/ingress/)
and [Virtual Service](https://istio.io/docs/concepts/traffic-management/#virtual-services)

- First determine the **Istio Ingress LB external IP** and **ports** by running the following commands:

```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
```

You can then verify the values as following:

```bash
echo $INGRESS_HOST $INGRESS_PORT $SECURE_INGRESS_PORT
```

The result should be an **IP address** along with two **ports**.

- Create an **Istio Ingress Gateway** and **Virtual Service** as following:

```yaml
cat <<EOF | kubectl apply -n httpbin-istio -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-vts
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /httpbin
    rewrite:
        uri: /
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

- Test that the routes are working using Curl:

```bash
curl -I http://$INGRESS_HOST:$INGRESS_PORT/httpbin/headers
curl -I http://$INGRESS_HOST:$INGRESS_PORT/httpbin/status/200
curl -I http://$INGRESS_HOST:$INGRESS_PORT/httpbin/delay/5
```

In all cases the response should be a **HTTP/1.1 200 OK**

**12)** [Istio Ingress TLS configuration](https://preliminary.istio.io/docs/tasks/traffic-management/secure-ingress/#configure-a-tls-ingress-gateway-for-multiple-hosts) for adding HTTPS support (based on [Secret Discovery](https://preliminary.istio.io/docs/tasks/traffic-management/secure-ingress/sds/))

> Note that Istio provides 2 approaches to add TLS support in the Ingress Gateways.
> The first one is based on a [File Mount](https://preliminary.istio.io/docs/tasks/traffic-management/secure-ingress/mount/)
> and the second one using [Secret Discovery](https://preliminary.istio.io/docs/tasks/traffic-management/secure-ingress/sds/).
> Whereas both are approaches are fairly straight forward, when having to support multiple hosts (e.g. several subdomains) the
> **Secret Discovery** approach is simpler to implement as it won't require to re-deploy the istio-ingressgateway.

<!-- - Enable SDS at ingress gateway level and deploy the ingress gateway agent

```bash
helm template $ISTIO_HOME/install/kubernetes/helm/istio/ --name istio \
--namespace istio-system -x charts/gateways/templates/deployment.yaml \
--set gateways.istio-egressgateway.enabled=false \
--set gateways.istio-ingressgateway.sds.enabled=true > \
istio-ingressgateway.yaml
```

> This is required as this feature is disabled by default hence why the **istio-ingressgateway.sds.enabled** is required.

```bash
kubectl apply -f istio-ingressgateway.yaml
```

Note that the above step was commented as it was avoided by just adding --set gateways.istio-ingressgateway.sds.enabled=true as part of the istio installation.
-->

- Using the [openssl](https://www.openssl.org/) utility [generate a TLS certificate](https://istio.io/docs/tasks/traffic-management/secure-ingress/sds/) as following:

```bash
openssl req -new -newkey rsa:4096 -x509 -sha256 -days 365 -nodes \
-out httpbin.adomain.com.crt -keyout httpbin.adomain.com.key
```

When prompted enter further details as desired, for example:

```bash
Country Name (2 letter code) []:GB
State or Province Name (full name) []:Warwickshire
Locality Name (eg, city) []:Leamington
Organization Name (eg, company) []:STTC
Organizational Unit Name (eg, section) []:Orders
Common Name (eg, fully qualified host name) []:httpbin.adomain.com
Email Address []:me@adomain.com
```

Once completed this should generate the **httpbin.adomain.com.crt** and **httpbin.adomain.com.key** files.

> Note that **Common Name** is a domain name and should match
> the **Hosts** value in the Ingress Gateway.

- Create the kubernetes secret **httpbin-istio** to hold the key and cert:

```bash
kubectl create -n istio-system secret generic httpbin-istio-secret \
--from-file=key=httpbin.adomain.com.key \
--from-file=cert=httpbin.adomain.com.crt
```

> Note that **istio-system** must be the namespace for the secret containing the certificate.

- And re-apply an **Gateway** and **Virtual Service** with TLS support against domain **httpbin.adomain.com**:

```yaml
cat <<EOF | kubectl apply -n httpbin-istio -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use istio default ingress gateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: "httpbin-istio-secret" # must be the same as secret
    hosts:
    - "httpbin.adomain.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-vts
spec:
  hosts:
  - "httpbin.adomain.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /httpbin
    rewrite:
        uri: /
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

- Test that the route is working using Curl:

```bash
curl -v -HHost:httpbin.adomain.com \
--resolve httpbin.adomain.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
--cacert httpbin.adomain.com.crt \
https://httpbin.adomain.com:$SECURE_INGRESS_PORT/httpbin/status/418
```

> It may take a couple of minutes for the certificates to load. So you may
> need to retry a few times.

> Also notice the curl parameters **--insecure**, **-HHost** and **--resolve**. These were added
> to resolve the domain **httpbin.adomain.com** against the ingress IP and Port using the self-signed cert.
> Alternatively add an entry to the /etc/hosts file to match
> the Ingress IP to the domain used.

The responses should be:

```
...
HTTP/2 418
...
SSL certificate verify ok.
...
    -=[ teapot ]=-

       _...._
     .'  _ _ `.
    | ."` ^ `". _,
    \_;`"---"`|//
      |       ;/
      \_     _/
        `"""`
```

or

```bash
curl -I --insecure -HHost:httpbin.adomain.com \
--resolve httpbin.adomain.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
https://httpbin.adomain.com:$SECURE_INGRESS_PORT/httpbin/headers
```

Responses should be a **HTTP/2 200**

- To do a recursive test:

```bash
for ((i=1;i<=10;i++)); do curl -I --insecure -HHost:httpbin.adomain.com --resolve httpbin.adomain.com:$SECURE_INGRESS_PORT:$INGRESS_HOST https://httpbin.adomain.com:$SECURE_INGRESS_PORT/httpbin/headers; done
```

- To delete the entire sample run:

```bash
kubectl -n httpbin-istio delete service httpbin
kubectl -n httpbin-istio delete deployment httpbin
kubectl -n httpbin-istio delete gateway httpbin-gateway
kubectl -n httpbin-istio delete virtualservice httpbin-vts
kubectl -n istio-system delete secret httpbin-istio-secret
kubectl delete namespace httpbin-istio
```

**13)** Access the monitoring services via **port-forwarding** as following:

- For **Prometheus** run command:

```bash
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &
```

Then access the following link in browser: http://localhost:9090/graph

> Check [this page](https://istio.io/docs/tasks/telemetry/metrics/querying-metrics/) for more details.


- For **Grafana** run command:

```bash
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```

Then accessing the following link in the browser: http://localhost:3000

> Check [this page](https://istio.io/docs/tasks/telemetry/metrics/using-istio-dashboard/) for more details.

- For **Jaeger** run command:

```bash
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686 &
```

Then accessing the following link in the browser: http://localhost:16686

> Check [this page](https://istio.io/docs/tasks/telemetry/distributed-tracing/jaeger/) for more details.

- For **Kiali** run command:

```bash
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001 &
```

> Note that Kiali will try to access **Jaeger** in **http://localhost:16686** so ensure port-forwarding is enable for it.

Then accessing the following link in the browser: http://localhost:20001/kiali/

> Check [this page](https://istio.io/docs/tasks/telemetry/kiali/) for more details.

- For other monitoring options [check section 12 of think recipe](https://luisw19.github.io/oci-series/oke-istio/)

- To **kill** all `kubectl port-forward` processes run:

```bash
killall kubectl
```

**14)** Other useful tips:

- To read the **logs** of **Ingress Gateway** execute:

```bash
kubectl -n istio-system logs $(kubectl -n istio-system get pods -listio=ingressgateway -o jsonpath='{.items[0].metadata.name}') istio-proxy --tail=300
```

- To **bash** into a container (e.g. **Ingress Gateway**) execute:

```bash
kubectl -n istio-system exec -it $(kubectl -n istio-system get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}') bash
```

- To configure **Istio metrics** to collect **telemetry** from the mesh (so it can can later be access with Prometheus) run the following command:

> This is important as its not enabled by default.

```bash
kubectl apply -f new_metrics.yaml
```

> The Yaml [was obtained from here](https://istio.io/docs/tasks/telemetry/metrics/collecting-metrics/).

- For instructions on configuring an **Istio metric** to **collect** additional **TCP telemetry** [Click here](https://istio.io/docs/tasks/telemetry/metrics/tcp-metrics/).

- In some cases Kiali displays traffic originating from or destined to  **unknown**. This traffic could be generated by either a non-Istio Ingress that is receiving traffic from the outside, or by internal pods without the Envoy sidecar.

To fix this, you can add adjust the **match** clause of the Istio config rule **promhttp** as following:

```bash
kubectl edit rules.config.istio.io promhttp -n istio-system
```

Then look for the **match** clause (e.g. /match) and then replace it with the desire rule, e.g.:

```yaml
match: (context.protocol == "http" || context.protocol == "grpc") && match(request.useragent | "", "kube-probe*") == false
```

> [This article](https://itnext.io/where-does-the-unknown-taffic-in-istio-come-from-4a9a7e4454c3) describes well the concept.
