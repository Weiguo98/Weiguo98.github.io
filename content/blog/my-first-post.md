---
title: "Kubernetes probes and pod lifecycle"
date: "2023-01-25"
# draft: true
---

- Readiness and Liveness probe
  [Configure Liveness, Readiness and Startup Probes | Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#before-you-begin)
	- The [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) uses liveness probes to know when to restart a container. For example, liveness probes could catch a deadlock, where an application is running, but unable to make progress. Restarting a container in such a state can help to make the application more available despite bugs.
	- The kubelet uses readiness probes to know when a container is ready to start accepting traffic. A Pod is considered ready when all of its containers are ready. One use of this signal is to control which Pods are used as backends for Services. When a Pod is not ready, it is removed from Service load balancers.
	- The readiness check will continuously run in the pod lifecycle.
	  > As long as Liveness Probe passed, the pod status changed to `running`. If you want to connect to a pod when it is ready for traffic, it is better to check the `ready` keywords.
- Pod lifecycle [Pod Lifecycle | Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
  
  | Value | Description |
  |---|---|
  | `Pending` | The Pod has been accepted by the Kubernetes cluster, but one or more of the containers has not been set up and made ready to run. This includes time a Pod spends waiting to be scheduled as well as the time spent downloading container images over the network. |
  | `Running` | The Pod has been bound to a node, and all of the containers have been created. At least one container is still running, or is in the process of starting or restarting. |
  | `Succeeded` | All containers in the Pod have terminated in success, and will not be restarted. |
  | `Failed` | All containers in the Pod have terminated, and at least one container has terminated in failure. That is, the container either exited with non-zero status or was terminated by the system. |
  | `Unknown` | For some reason the state of the Pod could not be obtained. This phase typically occurs due to an error in communicating with the node where the Pod should be running. |
	- When a Pod is being deleted, it is shown as `Terminating` by some kubectl commands. This `Terminating` status is not one of the Pod phases. A Pod is granted a term to terminate gracefully, which defaults to 30 seconds. You can use the flag `--force` to [terminate a Pod by force](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced).
<!-- ## Introduction -->

<!-- This is **bold** text, and this is *emphasized* text.

Visit the [Hugo](https://gohugo.io) website! -->

