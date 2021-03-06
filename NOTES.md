# Setup

## Create 3 Kubernetes clusters
- 1 Global control plane cluster
- 2 remote clusters

Set the following environment variables to point to your
clusters kube-context:

```
export KUMA_GLOBAL=<the global cluster's kube-context>
export KUMA_REMOTE_1=<the remote-1 cluster's kube-context>
export KUMA_REMOTE_2=<the remote-2 cluster's kube-context>
```
## Install Kuma in multi-zone configuration 

- Install Kuma
- Install ZoneIngress on the 2 zonal clusters
- MUST enable mTLS for cross-zone communication + traffic permission policy

## Verify Kuma Installation

Make sure the current context is the management cluster.

connect to the web UI:

```
kubectl port-forward --context $KUMA_GLOBAL -n kuma-system svc/kuma-control-plane 5681:5681
```

Then browse to http://localhost:5681/gui

You can also check various REST endpoints like zones:

```
$ curl http://localhost:5681/zones
{
 "total": 2,
 "items": [
  {
   "type": "Zone",
   "name": "remote-cluster-1",
   "creationTime": "2022-01-30T19:30:32Z",
   "modificationTime": "2022-01-30T19:30:32Z",
   "enabled": true
  },
  {
   "type": "Zone",
   "name": "remote-cluster-2",
   "creationTime": "2022-01-30T19:30:54Z",
   "modificationTime": "2022-01-30T19:30:54Z",
   "enabled": true
  }
 ],
 "next": null
}
```

## Install the social graph service and DB

- Install the service, by applying the social-graph.yaml manifest to the remote clusters.

Make sure to switch your kube context to a remote cluster.

Then port-forward to expose the service locally:

```
kubectl port-forward -n delinkcious svc/social-graph-manager 9090:9090
```

Now you can curl this endpoint (or any other name) and get an empty result:

```
$ curl http://localhost:9090/following/gigi
{"following":{},"err":""}
```

## Add the social graph service to Kuma

So far, Kuma is unaware of the social graph service.

The next step is to add the service to Kuma.

The trick is to add an annotation to the namespace and then automagically
Kuma will inject its data proxies to all the pods in the namespace

```
annotations:
    kuma.io/sidecar-injection: enabled
    kuma.io/mesh: default
```

# Enable connectivity between zones

Just injecting the data proxies is not enough for cross-zone connectivity.
Kuma requires that services will have strong identity via mTLS and specific traffic policy

The mTLS policy is defined on the Mesh resource:

Initially the default mesh has no policies:

```
$ kubectl get meshes.kuma.io default -o yaml | kubectl neat
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
```

Let's add mTLS across the board using the builtin certificate authority:

```
$ echo "apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
    - name: ca-1
      type: builtin" | kubectl apply -f -
```

There is a default allow-all traffic policy


# Add some data to each zone

Let's add a couple of follower/following relationships by invoking the /follow endpoint. I
will use the excellent HTTPie (https:/ / httpie. org/ ), which is a better curl in my honest
opinion. However, you can use curl if you prefer:

First zone one.

```
kubectl config set-context $KUMA_REMOTE_1
kubectl port-forward svc/social-graph-manager 9090:9090 

http POST http://localhost:9090/follow followed=gigi-1 follower=gigi-2
http POST http://localhost:9090/follow followed=gigi-1 follower=gigi-3
http POST http://localhost:9090/follow followed=gigi-1 follower=gigi-4
```

Let's verify it worked:

```
$ http http://localhost:9090/followers/gigi-1
HTTP/1.1 200 OK
Content-Length: 67
Content-Type: text/plain; charset=utf-8
Date: Sun, 06 Feb 2022 02:15:36 GMT

{
    "err": "",
    "followers": {
        "gigi-2": true,
        "gigi-3": true,
        "gigi-4": true
    }
}
```

Now. zone 2 with different followers:

```
kubectl config use-context $KUMA_REMOTE_2
kubectl port-forward -n delinkcious svc/social-graph-manager 9090:9090 

http POST http://localhost:9090/follow followed=gigi-1 follower=gigi-5
http POST http://localhost:9090/follow followed=gigi-1 follower=gigi-6
```

Let's verify

```
$ http http://localhost:9090/followers/gigi-1

HTTP/1.1 200 OK
Content-Length: 53
Content-Type: text/plain; charset=utf-8
Date: Sun, 06 Feb 2022 02:26:05 GMT

{
    "err": "",
    "followers": {
        "gigi-5": true,
        "gigi-6": true
    }
}
```

# Inside the mesh

It seems that port-forward to a pod circumvents the mesh. To test 
east-west traffic including  cross-zone communication let's send
some requests from inside the cluster.

First, run the g1g1/py-kube:0.3 image and start an interactive bash session:

```
kubectl run -it trouble --image g1g1/py-kube:0.3 -- bash
```

Then, lets get the followers of gigi-1 a couple of times:

```
root@trouble:/# http http://social-graph-manager:9090/followers/gigi-1
HTTP/1.1 200 OK
Content-Length: 67
Content-Type: text/plain; charset=utf-8
Date: Sun, 06 Feb 2022 10:17:07 GMT

{
    "err": "",
    "followers": {
        "gigi-2": true,
        "gigi-3": true,
        "gigi-4": true
    }
}

root@trouble:/# http http://social-graph-manager:9090/followers/gigi-1
HTTP/1.1 200 OK
Content-Length: 53
Content-Type: text/plain; charset=utf-8
Date: Sun, 06 Feb 2022 10:17:16 GMT

{
    "err": "",
    "followers": {
        "gigi-5": true,
        "gigi-6": true
    }
}
```

We can see that we get results from both remote clusters.

The DNS name of the service inside the cluster is:

```
dig +short social-graph-manager.delinkcious.svc.cluster.local
172.16.252.48
```

By the magic of core-dns it can be shortened to any of the following:

- social-graph-manager.delinkcious.svc.cluster
- social-graph-manager.delinkcious.svc
- social-graph-manager.delinkcious
- social-graph-manager 

The last one - just `social-graph-manager` works as long as there is
no other `social-graph-manager` service in another namespace.

# Observability

Let's add prometheus and Grafana to the mix. 
It's super-easy with Kuma. Just run:

```
kumactl install metrics | kubectl apply -f -
```

It installs Prometheus and GRafana into the kuma-metrics namespace 
on the management cluster.

Then we can expose Grafana locally using:

```
kubectl port-forward --context $KUMA_GLOBAL -n kuma-metrics svc/grafana 3000:80```
```

Now, browse to http://localhost:3000/dashboards and enter `admin/admin` for username/password. 

There several built-in dashboards you can explore.

# Changing load balancing algorithm

Kuma defines a default routing policy of round robin.

The k8s/traffic_policy file has a different policy that use 
the least request algorithm. This algorithm is useful when the duration of 
different requests varies widely. For example, some requests return immediately,
while other requests can take 30 seconds or several minutes. In these cases, 
some unfortunate service instance may be hit with multiple long requests
and exhaust its resources. With the least request algorithm, when a request comes
in several healthy servers are checked and the one with the least number of active
requests gets to handle it. 

To apply this policy delete the default traffic routing policy:

```
kubectl delete trafficroute route-all-default
```

Then apply the least request policy

```
kubectl apply -f k8s/traffic_route.yaml
```

# Adding a health check policy

The least request policy relies on healthy service instances. 
How does Kuma know if a servie instance is healthy or not. The passive way
is to discover it when it tries to send a request to unhealthy service instance
and the request fails and then needs to sent to another instance. Kuma has a 
CircuitBreaker policy that implements passive health checking.  

But, Kuma also supports active health checks that ping service instances
periodically and detect out of band if a service instance is healthy or not.

Let's cause the social graph service in th remote-cluster-2 to fail by
shutting down its DB

```
$ kubectl scale deploy social-graph-db --replicas 0 -n delinkcious --context $KUMA_REMOTE_2
deployment.apps/social-graph-db scaled
```

Now, with no DB requests to the social-graph service in the remote-cluster-2
return the following error:

```
curl http://social-graph-manager:9090/followers/gigi-1
{"followers":{},"err":"read tcp 10.0.128.30:43064-\u003e172.16.62.35:5432: read: connection reset by peer"}
```

Now, we can add a HealthCheck policy and see how Kuma deals with failure of some of 
the service instances.

```
kubectl apply --context $KUMA_GLOBAL -f k8s/health_check.yaml 
```

# What happens when health check fails









# Reference

https://konghq.com/blog/zerolb/
https://thenewstack.io/zerolb-a-new-decentralized-pattern-for-load-balancing/
https://github.com/svenwal/kuma-multi-zone-k3d/blob/main/start.sh
https://jetzlstorfer.medium.com/running-k3d-and-istio-locally-32adc5c41a63

https://kuma.io/docs/1.4.x/deployments/stand-alone/
https://kuma.io/docs/1.4.x/networking/networking/

https://www.cncf.io/online-programs/multi-cluster-multi-cloud-service-mesh-with-cncfs-kuma-and-envoy/
https://konghq.com/webinars/success/service-mesh-zerolb/