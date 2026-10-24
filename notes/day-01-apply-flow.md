## Day 1: Kubernetes control plane and the apply flow

### What happens when a Deployment is applied
1. `kubectl apply` goes to the API server, which validates it and stores the Deployment in etcd.
2. The Deployment controller (in the controller manager) sees the new Deployment and creates a ReplicaSet.
3. The ReplicaSet controller sees the ReplicaSet and creates 3 Pod objects, with no node assigned yet.
4. The scheduler sees the unassigned pods and picks a node for each one.
5. The kubelet on each chosen node sees a pod assigned to it, tells the container runtime to start the containers, and reports status back.
6. Once the readiness probe passes, the pod becomes part of the Service's endpoints. (Not tested yet, to verify in Week 2 with a Service and a failing readiness probe.)

Every component watches the API server and reacts to changes. They don't call each other directly.

### What I ran
```bash
k create deployment demo --image=nginx --replicas=3
k get events --sort-by=.metadata.creationTimestamp -o wide
k get pods -o wide
k delete pod <pod-name>
```

### What I saw (real output from my kind cluster)
```
deployment/demo           ScalingReplicaSet  Scaled up replica set demo-54659dcccf from 0 to 3
replicaset/demo-54659dcccf SuccessfulCreate  Created pod: demo-54659dcccf-z8jr2
pod/demo-54659dcccf-z8jr2  Scheduled         Successfully assigned default/demo-54659dcccf-z8jr2 to kind-worker3
pod/demo-54659dcccf-z8jr2  Pulling/Pulled    Successfully pulled image "nginx" in 1.762s
pod/demo-54659dcccf-z8jr2  Created, Started  Container created / Container started
```
- One pod landed on each worker (kind-worker, kind-worker2, kind-worker3), none on the control plane.
- The whole chain took about 2 seconds.
- Some events looked out of order (Pulling above Scheduled) because timestamps only have 1-second resolution.

### Where to see each step
- `k describe deployment <name>`: the Deployment controller scaling the ReplicaSet
- `k describe rs <name>`: the ReplicaSet controller creating pods
- `k describe pod <name>`: the scheduler (Scheduled) and the kubelet (Pulling, Created, Started)
- `k get events --sort-by=.metadata.creationTimestamp`: everything in one list

(`describe pod` only shows the later steps, because the pod doesn't exist until the ReplicaSet creates it.)

### Self-healing
When I deleted a pod, the ReplicaSet controller saw 2 pods when the desired count was 3 and created a replacement. This is the reconcile loop: Kubernetes keeps comparing desired state with actual state and fixes the difference.

### Who does what
| Thing | Handled by |
|---|---|
| Receiving requests, validating, storing state | API server (only component that talks to etcd) |
| Cluster state storage | etcd |
| Creating ReplicaSets and Pods, keeping counts right | Controller manager (Deployment and ReplicaSet controllers) |
| Choosing a node for a pod | Scheduler |
| Starting containers, running probes, reporting status | kubelet (on each node) |
| Pod IPs and pod-to-pod connectivity | CNI plugin (kindnet here, OVN-Kubernetes on OpenShift) |
| Service virtual IPs, routing to pods | kube-proxy |
| HTTP routing and TLS | Ingress controller (Routes on OpenShift) |
| type: LoadBalancer | Cloud provider or MetalLB |
| Network isolation | NetworkPolicy, enforced by the CNI plugin |

### Namespaces and kubeconfig
- Namespace = OpenShift project. `k get ns` = `oc get projects`.
- Kubeconfig (`~/.kube/config`) stores clusters, users, and contexts. A context is cluster + user + default namespace.
- `oc project dev` = `k config set-context --current --namespace=dev`.
- `oc login` adds or refreshes a context. The `server:` line in `k config view --minify` is the API server address.
- Deleting a namespace deletes everything inside it.
- Namespaces don't isolate the network by default. That needs NetworkPolicy (Week 5).

### What I got wrong earlier, and the correction
I thought the kubelet created the Deployment, the scheduler routed requests to the controller manager, kube-proxy assigned pod IPs, and CNI handled TLS routes and load balancers. In fact the controllers act first, the scheduler only assigns nodes to existing pods, the CNI gives pods their IPs, and kube-proxy only makes Services work.

### Open questions
- How does the scheduler score nodes?
- What changes in the events when a readiness probe fails?
