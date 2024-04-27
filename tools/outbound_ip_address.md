## Get Outbound IP of Kubernetes Cluster

You can quickly retrieve the outbound IP address of a Kubernetes cluster using the following `kubectl` command:

```bash
kubectl run -it --rm curl --image nginx --restart=Never --command -- curl ifconfig.me
```
