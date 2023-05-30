# Amazon EKS Network Policy Demo

In this walkthrough, we will use Nginx application to demo Network Policy functionality in Amazon EKS. It consists of a `demo-app`, `client-one`, and `client-two` applications.

## Overview

![k8s network policy overview](../images/k8s-net-pol-walkthrough.png)


## Walkthrough

Deploy the sample k8s deployments across two differnet namespaces

```bash
kubectl apply -f manifests/app-client.yaml
kubectl apply -f manifests/another-client.yaml
```

Verify the deployments

```bash
kubectl get all
```
```
NAME                                 READY    STATUS    RESTARTS         AGE
pod/client-one-5fc5ff8d84-qmxq4       1/1     Running   0                21m
pod/client-two-79c7489d5b-njzgg       1/1     Running   0                21m
pod/demo-app-c5cf4d4b8-ksvsh          1/1     Running   0                21m

NAME                             TYPE           CLUSTER-IP   EXTERNAL-IP                                                                    PORT(S)                     AGE
service/demo-svc                 ClusterIP      172.20.199.25    <none>                                                                         80/TCP                      3d12h
service/kubernetes               ClusterIP      172.20.0.1       <none>                                                                         443/TCP                     232d

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/client-one                1/1     1            1           21m
deployment.apps/client-two                1/1     1            1           21m
deployment.apps/demo-app                  1/1     1            1           21m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/client-one-5fc5ff8d84                1         1         1       21m
replicaset.apps/client-two-79c7489d5b                1         1         1       21m
replicaset.apps/demo-app-c5cf4d4b8                   1         1         1       21m
```

```bash
kubectl get all -n another-ns
```
```                                                                          
NAME                         			 READY   STATUS    RESTARTS   AGE
pod/another-client-one-f569c9cdd-4phfg   1/1     Running   0          40m
pod/another-client-two-78789c86b8-8z8lq  1/1     Running   0          40m

NAME                    			READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/another-client-one   1/1     1            1           40m
deployment.apps/another-client-two	 1/1     1            1           40m

NAME                               				DESIRED   CURRENT   READY   AGE
replicaset.apps/another-client-one-f569c9cdd    1         1         1       40m
replicaset.apps/another-client-two-78789c86b8   1         1         1       40m
```

## Verify connectivity

By default pods can communicate other pods seamlessely in a k8s cluster. Lets test the connectivity to `demo-app` application from with in the namespace and across namespaces.

Export the pod names
```bash
export DEMO_APP_POD=$(kubectl get pod --selector app=demo-app -o jsonpath='{.items[0].metadata.name}')
export CLIENT_ONE_POD=$(kubectl get pod --selector app=client-one -o jsonpath='{.items[0].metadata.name}')
export CLIENT_TWO_POD=$(kubectl get pod --selector app=client-two -o jsonpath='{.items[0].metadata.name}')
export XNS_CLIENT_ONE_POD=$(kubectl get pod -n another-ns --selector app=another-client-one -o jsonpath='{.items[0].metadata.name}')
export XNS_CLIENT_TWO_POD=$(kubectl get pod -n another-ns --selector app=another-client-two -o jsonpath='{.items[0].metadata.name}')
```

Test the connectivity from `client pods` with in & across the namespaces

```bash
kubectl exec -it $CLIENT_ONE_POD -- curl --max-time 10 demo-svc.default.svc.cluster.local
kubectl exec -it $CLIENT_TWO_POD -- curl --max-time 10 demo-svc.default.svc.cluster.local
kubectl exec -it $XNS_CLIENT_ONE_POD -n another-ns -- curl --max-time 10 demo-svc.default.svc.cluster.local
kubectl exec -it $XNS_CLIENT_TWO_POD -n another-ns -- curl --max-time 10 demo-svc.default.svc.cluster.local
```

You would see below response for each command, indicating successful API call
```html
<!DOCTYPE html>
<html>
  <head>
    <title>Welcome to Amazon EKS!</title>
    <style>
        html {color-scheme: light dark;}
        body {width: 35em; margin: 0 auto; font-family: Tahoma, Verdana, Arial, sans-serif;}
    </style>
  </head>
  <body>
    <h1>Welcome to Amazon EKS!</h1>
    <p>If you see this page, you are able successfully access the web application as the network policy allows.</p>
    <p>For online documentation and installation instructions please refer to
      <a href="https://docs.aws.amazon.com/eks/latest/userguide/eks-networking.html">Amazon EKS Networking</a>.<br/><br/>
      The migration guides are available at
      <a href="https://docs.aws.amazon.com/eks/latest/userguide/eks-networking.html">Amazon EKS Network Policy Migration</a>.
    </p>
    <p><em>Thank you for using Amazon EKS.</em></p>
</body>
</html>
```

Lets start applying the k8s Network policies to control traffic flow between different apps.

### Block all traffic to Demo app

```bash
kubectl apply -f policies/01-deny-all-ingress.yaml
networkpolicy.networking.k8s.io/demo-app-deny-all created
```
```bash
kubectl exec -it $CLIENT_ONE_POD -- curl --max-time 5 demo-svc.default.svc.cluster.local
kubectl exec -it $CLIENT_TWO_POD -- curl --max-time 5 demo-svc.default.svc.cluster.local
kubectl exec -it $XNS_CLIENT_ONE_POD -n another-ns -- curl --max-time 5 demo-svc.default.svc.cluster.local
kubectl exec -it $XNS_CLIENT_TWO_POD -n another-ns -- curl --max-time 5 demo-svc.default.svc.cluster.local
```
All the above calls would timeout 
```
curl: (28) Connection timed out after 5001 milliseconds
command terminated with exit code 28
```

### Allow traffic from Same namespace (default)
```bash
kubectl apply -f policies/02-allow-ingress-from-samens.yaml
networkpolicy.networking.k8s.io/demo-app-allow-samens created
```
```bash
kubectl exec -it $CLIENT_ONE_POD -- curl --max-time 5 demo-svc.default.svc.cluster.local
kubectl exec -it $CLIENT_TWO_POD -- curl --max-time 5 demo-svc.default.svc.cluster.local
kubectl exec -it $XNS_CLIENT_ONE_POD -n another-ns -- curl --max-time 5 demo-svc.default.svc.cluster.local
kubectl exec -it $XNS_CLIENT_TWO_POD -n another-ns -- curl --max-time 5 demo-svc.default.svc.cluster.local
```
First two commands succeed, as the network policy is allowing the ingress traffic only with in the `default` namespace.

#### Clean up
```bash
kubectl delete -f policies/02-allow-ingress-from-samens.yaml
networkpolicy.networking.k8s.io "demo-app-allow-samens" deleted
```

### Allow traffic from `client-one` app in same namespace
```bash
kubectl apply -f policies/03-allow-ingress-from-samens-client-one.yaml
networkpolicy.networking.k8s.io/demo-app-allow-samens-client-one created
```
```bash
kubectl exec -it $CLIENT_ONE_POD -- curl --max-time 5 demo-svc.default.svc.cluster.local
kubectl exec -it $CLIENT_TWO_POD -- curl --max-time 5 demo-svc.default.svc.cluster.local
kubectl exec -it $XNS_CLIENT_ONE_POD -n another-ns -- curl --max-time 5 demo-svc.default.svc.cluster.local
kubectl exec -it $XNS_CLIENT_TWO_POD -n another-ns -- curl --max-time 5 demo-svc.default.svc.cluster.local
```
Only the first command works, and the other three timeout given the network policy allow access only from `client-one` pod in `default` namespace.

### Allow traffic from `another-ns` namespace
```bash
kubectl apply -f policies/04-allow-ingress-from-xns.yaml
networkpolicy.networking.k8s.io/demo-app-allow-another-ns created
```
```bash
kubectl exec -it $XNS_CLIENT_ONE_POD -n another-ns -- curl --max-time 5 demo-svc.default.svc.cluster.local
kubectl exec -it $XNS_CLIENT_TWO_POD -n another-ns -- curl --max-time 5 demo-svc.default.svc.cluster.local
```
Now, traffic is allowed from all pods in the `another-ns` namespace.

#### Clean up
```bash
kubectl delete -f policies/04-allow-ingress-from-xns.yaml
networkpolicy.networking.k8s.io "demo-app-allow-another-ns" deleted
```

### Allow traffic from `another-client-one` in `another-ns` namespace
```bash
kubectl apply -f policies/05-allow-ingress-from-xns-client-one.yaml
networkpolicy.networking.k8s.io/demo-app-allow-another-client created
```
```bash
kubectl exec -it $XNS_CLIENT_ONE_POD -n another-ns -- curl --max-time 5 demo-svc.default.svc.cluster.local
kubectl exec -it $XNS_CLIENT_TWO_POD -n another-ns -- curl --max-time 5 demo-svc.default.svc.cluster.local
```
Now, traffic is allowed from only `another-client-one` pod in the `another-ns` namespace, and not from `another-client-two` pod.

## Ingress Cleanup

```bash
kubectl delete -f policies/01-deny-all-ingress.yaml
kubectl delete -f policies/02-allow-ingress-from-samens.yaml
kubectl delete -f policies/03-allow-ingress-from-samens-client-one.yaml
kubectl delete -f policies/04-allow-ingress-from-xns.yaml
kubectl delete -f policies/05-allow-ingress-from-xns-client-one.yaml
```

## Egress Example Walkthrough

### Deny all egress from `client-one` pod
```bash
kubectl apply -f policies/06-deny-egress-from-client-one.yaml
networkpolicy.networking.k8s.io/client-one-deny-egress created
```
```bash
kubectl exec -it $CLIENT_ONE_POD -- curl --max-time 5 demo-svc.default.svc.cluster.local
```
```
curl: (28) Resolving timed out after 5000 milliseconds
command terminated with exit code 28
```
It fails with timeout error, as `client-one` pod is not able to lookup/resolve the `demo-svc` service ip address.

### Allow egress to a specific `port(53)` on `coredns` from `client-one` pod 
```bash
kubectl apply -f policies/07-allow-egress-to-coredns.yaml
networkpolicy.networking.k8s.io/client-one-allow-egress-coredns created
```
```bash
kubectl exec -it $CLIENT_ONE_POD -- curl --max-time 5 -v demo-svc.default.svc.cluster.local
```
```
*   Trying 172.20.20.101:80...
* Connection timed out after 5000 milliseconds
* Closing connection 0
curl: (28) Connection timed out after 5000 milliseconds
command terminated with exit code 28
```
Now, `client-one` is able to communicate with `coredns` to resolve the service ip of `demo-svc`, but failed to connect to `demo-app` due to missing egress rule.

### Allow egress to multiple apps and ports from `client-one` pod
```bash
kubectl apply -f policies/08-allow-egress-to-demo-app.yaml
networkpolicy.networking.k8s.io/client-one-allow-egress-demo-app created
```
```bash
kubectl exec -it $CLIENT_ONE_POD -- curl --max-time 5 demo-svc.default.svc.cluster.local
```
This time, `client-one` is able to resolve the ip address and connect to the `demo-app` on port `80` successfully.

YaY!!

## Cleanup

```bash
./cleanup.sh
```
## Security

See [CONTRIBUTING](../CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](../LICENSE) file.
