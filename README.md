# httpbin deployment

This is a kubernetes Deployment from [kennethreitz/httpbin](https://github.com/kennethreitz/httpbin)
following best practices.

This repository follows [Semantic Commit Messages](https://www.conventionalcommits.org/en/v1.0.0/).

## Setup

### Requirements

In order to test, you need in your system

- `Docker`
- `kind`
- `kubectl` (bear in mind [skew policy](https://kubernetes.io/releases/version-skew-policy/). 

Once everything is ready, create the cluster and apply the deployment.

```
$ kubectl apply -f kind-cluster-config.yaml
$ kubectl apply -f httpbin-deployment.yaml
```

## Deployment best practices

- Define `securityContext` you can either set it at Pod spec level or at container level. It depends
if you want to apply different options to different containers or the same to all of them. In 
this example set un unprivilege UID/GID.

- Set `requests` and memory `limits`, leave out CPU to avoid throttling. This sets a QoS `burstable`
but is a better approach for scheduling, the requests are guaranteed.

- Define `livenessProbe` and `readinessProbe`. The first one restarts the pod in the event of failure,
the second prevents from sending traffic when the pod is not ready. In a production scenario `readinessProbe`
should check dependencies (i.e: connection to DB, Kafka connection is ready).

- Define `NetworkPolicy` (what pods from what namespaces can talk to each other). (**NotImplemented**)

- Define `PodBudgetDisruption` to protect against disruptions. Better to use `maxUnavailable` as it
automatically responds to `changes` in the number of replicas of the corresponding controller. (**NoImplemented**)

## Topology spread constraints

Using spread topology constraints we can evently distribute our pods, depending on the 
`whenUnsatisfiable` we achieve different behaviour. More on [topology spread constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/).

- `maxSkew` is tricky, you need to have in consideration rolling rollouts. If we have
3 nodes and `maxSkew: 1` and we try to rollout a  new version the pod will be in `pending state`. 

This example allows to have up to 2 pods per node. This means we could have 6 pods and then 
we wouldn't be able to perform a rolling restart due to `maxSkew`.

This requires planification especially if you plan to integrate with `HPA`.

```
$ kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE                
httpbin-548d89c647-2q58m   1/1     Running   0          20m   10.244.3.10   httpbin-k8s-worker2 
httpbin-548d89c647-5lvkt   1/1     Running   0          62s   10.244.3.11   httpbin-k8s-worker2 
httpbin-548d89c647-btps2   1/1     Running   0          16m   10.244.2.12   httpbin-k8s-worker3 
httpbin-548d89c647-kjf7l   1/1     Running   0          20m   10.244.2.11   httpbin-k8s-worker3 
httpbin-548d89c647-7lgqc   1/1     Running   0          20m   10.244.1.9    httpbin-k8s-worker  
httpbin-548d89c647-hmnxp   1/1     Running   0          60s   10.244.1.10   httpbin-k8s-worker  
httpbin-548d89c647-lvfg2   0/1     Pending   0          42s   <none>        <none>              
```

If topology constraints are updated 

```yaml
whenUnsatisfiable: ScheduleAnyway
```
This triggers a rolling restart.


## Testing

Just verify the the service is working:

```
$ kubectl expose deploy/httpbin --target-port=80 --port=8080
$ kubectl exec -it pod/debug -- curl -s  httpbin.default.svc.cluster.local:8080/anything/hello
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.default.svc.cluster.local:8080",
    "User-Agent": "curl/8.14.1"
  },
  "json": null,
  "method": "GET",
  "origin": "10.244.1.7",
  "url": "http://httpbin.default.svc.cluster.local:8080/anything/hello"
}

```
