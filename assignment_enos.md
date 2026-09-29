# Debug Operations in Kubernetes

You can use the `kubectl` command-line tool to debug Kubernetes clusters. The `kubectl` tool uses the Kubernetes API to interact with the Kubernetes control plane. This topic provides examples of `kubectl` commands that are used in debug operations, in the following order:

- [`kubectl get`](#kubectl-get)
- [`kubectl describe`](#kubectl-describe)
- [`kubectl logs`](#kubectl-logs)
- [`kubectl debug`](#kubectl-debug)
- [`kubectl exec`](#kubectl-exec) 

For `kubectl` installation and detailed reference information, see [Command line tool (kubectl)](https://kubernetes.io/docs/reference/kubectl/) in the Kubernetes.io [reference](https://kubernetes.io/docs/reference/).

The following examples assume the use of the `default` cluster namespace.

## kubectl get

Use `kubectl get` to list basic information about cluster resources. This command allows you to quickly check for pod names, readiness, and lifecycle phase.

```
$ kubectl get pods
```
This instruction lists the pod resources that exist in the current namespace. This instruction does not use the `-n` argument to specify a namespace and is therefore scoped to the current `default` namespace. 

```
NAME         READY   STATUS    RESTARTS        AGE
api-server   2/2     Running   4 (6h31m ago)   24h
```

The `api-server` pod includes two containers, both of which are ready. The lifecycle phase of the pod resource is `Running`. There have been four restarts since the resource was created about 24 hours ago. The last restart occurred about 6 hours and 31 minutes ago. The time values shown in the **RESTARTS** and **AGE** columns are estimates.

To view exhaustive configuration and status information for a resource, use the `kubectl get` command with the `-o yaml` argument. 

```
$ kubectl get pod flying-pod -o yaml
```
This instruction reports information from the `etcd` cluster database about the `flying-pod` pod resource. Note the status blocks that appear in the following partial output.

```
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"flying-pod","namespace":"default"},"spec":{"containers":[{"command":["sh","-c","echo Flying pod started; sleep 10"],"image":"busybox:1.36.1","name":"flying-pod"}],"restartPolicy":"Never"}}
  creationTimestamp: "2026-09-23T20:06:24Z"
  generation: 1
  name: flying-pod
  namespace: default
  resourceVersion: "139232"
  uid: 255c3e92-2a38-4a4b-8e84-6d19362046d7
spec:
  containers:
  ...
  ...
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2026-09-23T20:06:40Z"
    message: pod sandbox is not ready
    observedGeneration: 1
    reason: PodSandboxNotReady
    status: "False"
    type: PodReadyToStartContainers
  - lastProbeTime: null
    lastTransitionTime: "2026-09-23T20:06:24Z"
    observedGeneration: 1
    reason: PodCompleted
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2026-09-23T20:06:39Z"
    observedGeneration: 1
    reason: PodCompleted
    status: "False"
    type: Ready
    ...
    ...
  containerStatuses:
  - containerID: containerd://3cc36ceda5767b3654664931c03979857069996c9dcb56b408dfbe6db1045531
    image: docker.io/library/busybox:1.36
    imageID: docker.io/library/busybox@sha256:73aaf090f3d85aa34ee199857f03fa3a95c8ede2ffd4cc2cdb5b94e566b11662
    lastState: {}
    name: flying-pod
    ready: false
    resources: {}
    restartCount: 0
    started: false
    state:
      terminated:
        containerID: containerd://3cc36ceda5767b3654664931c03979857069996c9dcb56b408dfbe6db1045531
        exitCode: 0
        finishedAt: "2026-09-23T20:06:39Z"
        reason: Completed
        startedAt: "2026-09-23T20:06:29Z"
    ...
    ...
```

You can also use the `kubectl get` command to retrieve information about service type resources. 

```
$ kubectl get services
```

This instruction lists the services that exist in the current namespace&mdash;in this case, the `default` namespace. 

```
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
api-service   LoadBalancer   10.104.180.102   10.104.180.102   80:31058/TCP   25h
kubernetes    ClusterIP      10.96.0.1        <none>           443/TCP        3d21h
```

The output displays information about a `kubernetes` service of type `ClusterIP`. This built-in service provides access to the Kubernetes API. In this case, `api-service` is a user-defined service of type `LoadBalancer`. The service exposes an external IP address that is used to route requests to the containerized API server in the `api-server` pod.

For more information about the `kubectl get` command, see [`kubectl get`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/).

## kubectl describe

Use the `kubectl describe` command to obtain a comprehensive description of the state of a Kubernetes resource. This command provides aggregated information about resource function within the context of the cluster. In contrast to the `kubectl get <myPod> -o yaml` instruction, `kubectl describe` aggregates and delivers information that includes event history, configuration, health metrics, and relationships with network, memory, and storage components.

```
$ kubectl describe pod api-server
```
This instruction retrieves a comprehensive description of the `api-server` pod resource. 

Output from the `kubectl describe` command can be separated into information specific to the pod wrapper and information specific to its containers. 

The first fields describe the `api-server` pod. These include diagnostic fields such as `Priority`, `Labels`, `Status` and `IP`.

```
Name:             api-server
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.64.2
Start Time:       Mon, 21 Sep 2026 17:32:42 -0700
Labels:           app=api-server
Annotations:      <none>
Status:           Running
IP:               10.244.0.30
IPs:
  IP:  10.244.0.30
```

Next, the `Containers` object encloses child objects, each of which describe one of the pod's two containers:

- `node-api`, an API server application
- `debug-sidecar`, BusyBox troubleshooting tools

Each child object describes a container in the pod. Inspect the child objects for information about the image that was used to create the containerized application, as well as the port, the current and last state, and exit codes.  

```
Containers:
  node-api:
    Container ID:  containerd://3fa7359df695d173b97b3c2332e2e471c7ee23432a7f06499328c0d9f7879542
    Image:         node:20-alpine
    Image ID:      docker.io/library/node@sha256:fb4cd12c85ee03686f6af5362a0b0d56d50c58a04632e6c0fb8363f609372293
    Port:          8080/TCP (http)
    Host Port:     0/TCP (http)
    Command:
      node
      -e
      const http = require('http');
      
      http.createServer((req, res) => {
        res.writeHead(200, { 'Content-Type': 'text/plain' });
        res.end('API OK');
      }).listen(8080, '0.0.0.0');
      
    State:          Running
      Started:      Tue, 22 Sep 2026 11:30:40 -0700
    Last State:     Terminated
      Reason:       Unknown
      Exit Code:    255
      Started:      Tue, 22 Sep 2026 10:49:43 -0700
      Finished:     Tue, 22 Sep 2026 11:30:27 -0700
    Ready:          True
    Restart Count:  2
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-nqh7s (ro)

  debug-sidecar:
    Container ID:  containerd://4a7fe2081c77e6bc9f1d308a86662ea0ffd266c0232273c37df0c56fc310e378
    Image:         busybox:1.36
    Image ID:      docker.io/library/busybox@sha256:73aaf090f3d85aa34ee199857f03fa3a95c8ede2ffd4cc2cdb5b94e566b11662
    Port:          <none>
    Host Port:     <none>
    Command:
      sh
      -c
      while true; do sleep 3600; done
    State:          Running
      Started:      Tue, 22 Sep 2026 11:30:40 -0700
    Last State:     Terminated
      Reason:       Unknown
      Exit Code:    255
      Started:      Tue, 22 Sep 2026 10:49:43 -0700
      Finished:     Tue, 22 Sep 2026 11:30:27 -0700
    Ready:          True
    Restart Count:  2
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-nqh7s (ro)
```

The final group of fields contain additional information about the pod. Be sure to check the `Conditions` and `Events` fields for diagnostic information.

```
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-nqh7s:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
```

For more information about the `kubectl describe` command, see [`kubectl describe`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/).

## kubectl logs

Use the `kubectl logs` command to inspect resource logs. This command is specific to the resource and, unlike `get describe`, does not provide information about cluster interactions.

The following instruction prints log information about the `flying-pod` pod resource to the command line.

```
$ kubectl logs --timestamps flying-pod
```
This command uses the `--timestamps` argument to include date-time information with the event.
`
```
2026-09-23T20:06:29.023518788Z Flying pod started
```

The following instruction prints heartbeat information for the `api-server` pod's `node-api` container.

```
$ kubectl logs -f api-server -c node-api
```
The command output shows the resource heartbeat with date-time information:

```
API server started on port 8080
Debug heartbeat: 2026-09-23T21:21:47.399Z
Debug heartbeat: 2026-09-23T21:21:57.409Z
Debug heartbeat: 2026-09-23T21:22:07.411Z
Debug heartbeat: 2026-09-23T21:22:17.421Z
Debug heartbeat: 2026-09-23T21:22:27.429Z
Debug heartbeat: 2026-09-23T21:22:37.430Z
Debug heartbeat: 2026-09-23T21:22:47.434Z
Debug heartbeat: 2026-09-23T21:22:57.445Z
Debug heartbeat: 2026-09-23T21:23:07.445Z
Debug heartbeat: 2026-09-23T21:23:17.457Z
Debug heartbeat: 2026-09-23T21:23:27.467Z
Debug heartbeat: 2026-09-23T21:23:37.468Z
Debug heartbeat: 2026-09-23T21:23:47.478Z
Debug heartbeat: 2026-09-23T21:23:57.486Z
Debug heartbeat: 2026-09-23T21:24:07.488Z
Debug heartbeat: 2026-09-23T21:24:17.498Z
Debug heartbeat: 2026-09-23T21:24:27.506Z
```

For more information about the `kubectl logs` command, see [`kubectl logs`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/).

## kubectl debug

Use the `kubectl debug` command to clone a pod or to create an ephemeral container in an existing pod for debug purposes. For example, you can use `kubectl debug` with the `--copy-to` argument to clone a resource for debug purposes.

```
$ kubectl debug api-server --copy-to=api-server-debug --image=busybox:1.36
Defaulting debug container name to debugger-mpqlw.
```
This command clones the `api-server` pod to the `api-server-debug` copy and uses the `--image` argument to apply BusyBox image version 1.36. Kubernetes creates the container name with a disambiguator, in this case, `mpqlw`.

The `kubectl get` command is used to verify creation of the `api-server-debug` clone.

```
$ kubectl get pods
```

The `api-server-debug` clone is verified in the following output.

```
NAME               READY   STATUS      RESTARTS      AGE
api-server         2/2     Running     0             68m
api-server-debug   2/3     NotReady    3 (47s ago)   61s
flying-pod         0/1     Completed   0             143m
```

Use a debug copy to isolate debug operations from production. Creating a debug copy is a useful strategy in testing.

For more information about the `kubectl debug` command, see [`kubectl debug`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_debug/).


## kubectl exec

Use the `kubectl exec` command to execute commands inside a container. This command requires the specification of the container name or annotation; otherwise, the command is executed against the first container that is defined in the pod specification (`spec`). 

```
$ kubectl exec -it api-server-debug -c debugger -- sh
```
This command requests a connection to an interactive TTY session in the `api-server-debug` pod's `debugger` container. The `debugger` container is running the BusyBox collection of command-line tools.

```
/ #
```

The following instruction uses the `wget` utility to send an `HTTP GET` request to the API server.

```
/ # 
/ # wget -qO- http://localhost:8080
```

The API server responds with the success message:

```
API OK
```

For more information about the `kubectl exec` command, see [`kubectl exec`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_exec/).


# For More Information

- [kubectl reference](https://kubernetes.io/docs/reference/kubectl/generated/)

- [Kubernetes Overview](https://kubernetes.io/docs/concepts/overview/)
