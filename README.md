# mci-multi-backend-service-demo
Demo of exposing multiple backend services through GCPs [multi-cluster ingress](https://cloud.google.com/kubernetes-engine/docs/concepts/multi-cluster-ingress) &amp; Anthos Service Mesh.

These instructions assume you already have a pair of GKE clusters up and running with ASM installed and exposed as a multi-cluster service using MCI.

There are 3 different services that will get deployed across both clusters (2 pods per cluster X 2 clusters = 4 pods per service):

- at the root path (`/`), [whereami](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/master/whereami) is listening
- at `/app1`, [nginx](https://hub.docker.com/_/nginx) is listening
- at `/app2`, [httpd](https://hub.docker.com/_/httpd) is listening

##

First, export the kube context names of cluster 1 and cluster 2 to environment variables:

```
# replace these with your own values!!!
export CLUSTER_1_CTX=${CLUSTER_1_CTX}
export CLUSTER_2_CTX=${CLUSTER_2_CTX}
```

Create your namespaces for each app and enable sidecar injection:
```
for NAMESPACE in app0 app1 app2
do 
    kubectl --context=${CLUSTER_1_CTX} create ns $NAMESPACE
    kubectl label --context=${CLUSTER_1_CTX} ns $NAMESPACE istio.io/rev=${ASM_LABEL}

    kubectl --context=${CLUSTER_2_CTX} create ns $NAMESPACE
    kubectl label --context=${CLUSTER_2_CTX} ns $NAMESPACE istio.io/rev=${ASM_LABEL}
done
```

"app0" is `whereami`, "app1" is `nginx`, and "app2" is `httpd`.

In the `k8s` subfolder, inspect each app's virtual service resource spec. Notice how each has a unique URI pattern match. Apply the services:

```
kubectl --context=${CLUSTER_1_CTX} apply -f k8s/
kubectl --context=${CLUSTER_2_CTX} apply -f k8s/
```

Capture the IP address of the MCI resource from your [MCI config cluster](https://cloud.google.com/kubernetes-engine/docs/concepts/multi-cluster-ingress#config_cluster_design) (in my case, it is Cluster 1):

```
export MCI_VIP=$(kubectl --context ${CLUSTER_1_CTX} -n istio-system get multiclusteringress -o jsonpath="{.items[].status.VIP}")
```

And finally, `curl` the various paths:
```
$ curl http://$MCI_VIP

{
  "cluster_name": "gke-central1-1", 
  "host_header": "34.117.211.56", 
  "metadata": "frontend", 
  "node_name": "gke-gke-central1-1-default-pool-6e287e8f-3f5f.c.am01-networking.internal", 
  "pod_ip": "10.0.1.9", 
  "pod_name": "app0-deployment-798b764bf6-dj958", 
  "pod_name_emoji": "🙅🏽‍♀", 
  "pod_namespace": "app0", 
  "pod_service_account": "app0-ksa", 
  "project_id": "am01-networking", 
  "timestamp": "2021-01-29T19:33:51", 
  "zone": "us-central1-a"
}

$ curl http://$MCI_VIP/app1
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>


$ curl http://$MCI_VIP/app2
<html><body><h1>It works!</h1></body></html>
```

