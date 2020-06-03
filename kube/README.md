# kube

## outside of cluster

### cluster access

 - [local proxy](https://kubernetes.io/docs/tasks/access-kubernetes-api/http-proxy-access-api/)
`kubectl proxy –port 8080` pod can then be accessed on http://localhost:8080/api/v1/namespaces/default/pods
 - [forward port](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)
`kubectl port-forward <pod> <local-port>:<pod-port>`
 - run "debug" pod, exec inside and run relevant debug commands

### monitor cluster

 - **cluster info** `kubectl cluster-info` quick check if cluster is accessible
 - **master nodes** `watch 'kubectl get nodes --sort-by=.metadata.creationTimestamp --selector="node-role.kubernetes.io/master"'`
 - **worker nodes** `watch 'kubectl get nodes --sort-by=.metadata.creationTimestamp -l "kubernetes.io/role=node"'`
 - **top 10 nodes with highest CPU** `kubectl top nodes --selector="node-role.kubernetes.io/node" | sort --reverse --key 3 --numeric | head -10`

### monitor pods

 - **failed pods** `watch 'kubectl get pods -A --field-selector status.phase!=Running,status.phase!=Succeeded'`
 - **not-running pods** `watch 'kubectl get pods -A --field-selector status.phase!=Running'`
 - **running pods** `watch 'kubectl get pods -A --field-selector status.phase=Running'`

### debug
 - **check pods start time** `kubectl get pod -n <namespace> -o jsonpath='{.items[*].status.startTime}' | tr -s '[[:space:]]' '\n' | sort`

### inspect pods

 - `kubectl get pods -n <namespace>`
 - `kubectl describe pod -n <namespace> <pod>`
 - `kubectl logs -n <namespace> <pod>`
 - `kubectl exec -it -n <namespace> <pod> [sh|bash]`

### cluster autoscaler

 - **check status** `kubectl get deployments -n kube-system cluster-autoscaler`
 - **enable** `kubectl scale deployment.v1.apps/cluster-autoscaler --replicas=1 -n kube-system`
 - **disable** `kubectl scale deployment.v1.apps/cluster-autoscaler --replicas=0 -n kube-system`

### restart deployment/daemonset

`kubectl rollout restart -n <namespace> [daemonset|deployment]/<name>`

## inside cluster (on the node)

### setup kubectl

```
kubectl () {
    /usr/bin/docker run -i --rm --net=host k8s.gcr.io/hyperkube-amd64:<version> /hyperkube kubectl --request-timeout=1s "$@"
}
```

### monitor

Main systemd units to check are `docker` and `kubelet`:
 - status `systemctl status`
 - logs `journalctl -fu <unit>`
 - unit file `systemctl cat <unit>`

Kubelet uses `/etc/kubernetes/manifests` as staticPodPath (which can be seen from `systemctl status kubelet` command).
staticPodPath is directory where kubelet will read every yaml file and start pod. To troubleshoot scheduler, put yaml file there.

`/etc/kubernetes/manifests/...` can be edited and kubelet restarted `systemctl restart kubelet.service`. After restart just monitor `journalctl -fu kubelet`

#### api server logs

All info is in `/etc/kubernetes/manifests/kube-apiserver.yaml` (kubelet manifests directory). If pod is not running we can check docker logs:
 - `docker ps | grep kube-apiserver` - get api server container id
 - `docker logs -f <id>` - check logs
 - or view log directly `sudo find / -name "*apiserver*log"`

if/when api server pod is running, the logs (exactly the same) can be viewed via kubectl:
 - `kubectl get po -n kube-system`
 - `kubectl logs -n kube-system kube-apiserver-...`

Same can be done for `kube-dns`, `kube-flannel`, and `kube-proxy`

more reading:
 - https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/
 - https://kubernetes.io/docs/tasks/debug-application-cluster/determine-reason-pod-failure/

#### cloud formation signal (AWS)

if node does not come up, the problem can be in `cfn-signal`:
 - `journalctl -fu cfn-signal`
 - `systemctl status cfn*`