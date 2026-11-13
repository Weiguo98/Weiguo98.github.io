---
title: "Kubernetes 探针与 Pod 生命周期"
date: "2023-01-25"
tags: ["kubernetes"]
---

- Readiness 和 Liveness 探针
  [Configure Liveness, Readiness and Startup Probes | Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#before-you-begin)
  - [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) 通过 liveness 探针判断什么时候需要重启容器。比如应用发生死锁时，进程还在运行，但已经无法继续工作，liveness 探针可以发现这种情况。重启处于这种状态的容器，可以让应用在有 bug 的情况下仍然尽量保持可用。
  - kubelet 通过 readiness 探针判断容器什么时候可以开始接收流量。只有 Pod 中所有容器都就绪时，Pod 才被认为是就绪的。这个信号的一个用途是控制哪些 Pod 可以作为 Service 的后端：Pod 未就绪时，会被从 Service 的负载均衡中移除。
  - readiness 检查会在 Pod 的整个生命周期中持续运行。
    > [!TIP]
    > 只要 Liveness 探针通过，Pod 状态就会变为 `running`。如果你想在 Pod 真正可以接收流量时再连接，最好检查 `ready` 字段。
- Pod 生命周期 [Pod Lifecycle | Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

  | 取值 | 说明 |
  |---|---|
  | `Pending` | Pod 已被 Kubernetes 集群接受，但有一个或多个容器还没有创建好、不能运行。包括 Pod 等待调度的时间，以及通过网络下载容器镜像的时间。 |
  | `Running` | Pod 已经绑定到某个节点，所有容器都已创建。至少有一个容器仍在运行，或者正在启动或重启。 |
  | `Succeeded` | Pod 中所有容器都已成功终止，并且不会再重启。 |
  | `Failed` | Pod 中所有容器都已终止，并且至少有一个容器是失败终止的，即容器以非 0 状态码退出，或者被系统终止。 |
  | `Unknown` | 因为某些原因无法获取 Pod 的状态。通常是因为与 Pod 所在节点通信出错。 |

 删除 Pod 时，某些 kubectl 命令会显示 `Terminating` 状态。`Terminating` 并不是 Pod 的阶段（phase）之一。Pod 有一段优雅终止的时间，默认是 30 秒。你可以用 `--force` 参数[强制终止 Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination-forced)。
