---
author:
  name: Linode Community
  email: docs@linode.com
description: 'Learn frequently-used troubleshooting commands for Kubernetes and review common Kubernetes issues.'
keywords: ['kubernetes','cluster','troubleshooting','k8s','kubectl']
license: '[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0)'
published: 2019-07-29
modified_by:
  name: Linode
title: "Troubleshooting Kubernetes"
concentrations: ["Kubernetes"]
external_resources:
- '[Kubernetes Documentation - Troubleshooting](https://kubernetes.io/docs/tasks/debug-application-cluster/troubleshooting/)'
aliases: ['applications/containers/troubleshooting-kubernetes/']
---

<!-- EDITOR'S NOTE: This guide is split into two halves:

1. A reference for commands and options to use when debugging
2. A list of common problems

I can't really think of all the common problems that could might appear, so I've set up a few to start, and I'm imagining that the list could be expanded over time. We should consider advertising to Support that we would welcome JIRA tickets with recommendations to get addeed to the list, if they start to see certain problems more often. -->

Troubleshooting issues with Kubernetes can be complex, and it can be difficult to account for all the possible error conditions you may see. This guide tries to equip you with the core tools that can be useful when troubleshooting, and it introduces some situations that you may find yourself in.

{{< disclosure-note "Where to go for help outside this guide" >}}
If your issue is not covered by this guide, we also recommend researching and posting in the [Linode Community Questions](https://www.linode.com/community/questions/) site and in `#linode` on the [Kubernetes Slack](http://slack.k8s.io/), where other Linode users (and the Kubernetes community) can offer advice.

If you are running a cluster on Linode's managed LKE service, and you are experiencing an issue related to your master/control plane components, you can report these issues to Linode by [contacting Linode Support](/docs/platform/billing-and-support/support/). Examples in this category include:

-   Kubernetes' API server not running. If kubectl does not respond as expected, this can indicate problems with the API server.

-   The CCM, CSI, Calico, or kube-dns pods are not running.

-   Annotations on LoadBalancer services aren’t functioning.

-   PersistentVolumes are not re-attaching.

Please note that the kube-apiserver and etcd pods will not be visible for LKE clusters, and this is expected. Issues outside of the scope of Linode Support include:

-   Problems with the control plane of clusters not managed by LKE.

-   Problems with applications deployed on Kubernetes.
{{< /disclosure-note >}}

In this guide we will:

-   [Refer to the most-frequently used commands](#general-troubleshooting-strategies) when troubleshooting a cluster and find out where your Kubernetes logs are stored.
-   [Review some common issues](#troubleshooting-examples) that can appear on your cluster.


## General Troubleshooting Strategies

To troubleshoot issues with the applications running on your cluster, you can rely on the [`kubectl` command](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) to gather debugging information. `kubectl` includes a set of subcommands that can be used to research issues with your cluster, and this guide will highlight four of them: [get](#kubectl-get), [describe](#kubectl-describe), [logs](#kubectl-logs), and [exec](#kubectl-exec).

To troubleshoot issues with your cluster, you may need to [directly view the logs](#viewing-master-and-worker-logs) that are generated by Kubernetes' components.

### kubectl get

Use the [`get` command](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get) to list different kinds of resources in your cluster (`nodes`, `pods`, `services`, etc). The output will show the status for each resource returned. For example, this output shows that a pod is in the `CrashLoopBackOff` status, which means it should be investigated further:

    kubectl get pods
    NAME              READY     STATUS             RESTARTS   AGE
    ghost-0           0/1       CrashLoopBackOff   34         2h
    mariadb-0         1/1       Running            0          2h

-   Use the `--namespace` flag to show resources in a certain namespace:

        # Show pods in the `kube-system` namespace
        kubectl get pods --namespace kube-system

    {{< note >}}
If you've set up Kubernetes using automated solutions like Linode's Kubernetes Engine, [k8s-alpha CLI](/docs/applications/containers/how-to-deploy-kubernetes-on-linode-with-k8s-alpha-cli/), or [Rancher](/docs/applications/containers/how-to-deploy-kubernetes-on-linode-with-rancher-2-2/), you'll see [csi-linode](https://github.com/linode/linode-blockstorage-csi-driver) and [ccm-linode](https://github.com/linode/linode-cloud-controller-manager) pods in the `kube-system` namespace. This is normal as long as they're in the Running status.
{{< /note >}}


-   Use the [`-o` flag](https://kubernetes.io/docs/reference/kubectl/overview/#output-options) to return the resources as YAML or JSON. The Kubernetes API's complete description for the returned resources will be shown:

        # Get pods as YAML API objects
        kubectl get pods -o yaml

-   Sort the returned resources with the `--sort-by` flag:

        # Sort by name
        kubectl get pods --sort-by=.metadata.name

-   Use the `--selector` or `-l` flag to get resources that match a label. This is useful for finding all pods for a given service:

        # Get pods which match the app=ghost selector
        kubectl get pods -l app=ghost

-   Use the [`--field-selector` flag](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/) to return resources which match different resource fields:

        # Get all pods that are Pending
        kubectl get pods --field-selector status.phase=Pending

        # Get all pods that are not in the kube-system namespace
        kubectl get pods --field-selector metadata.namespace!=kube-system


### kubectl describe

Use the [`describe` command](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#describe) to return a detailed report of the state of one or more resources in your cluster. Pass a resource type to the `describe` command to get a report for each of those resources:

    kubectl describe nodes

Pass the name of a resource to get a report for just that object:

    kubectl describe pods ghost-0

You can also use the `--selector` (`-l`) flag to filter the returned resources, as with the `get` command.

### kubectl logs

Use the [`logs` command](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs) to print logs collected by a pod:

    kubectl logs mariadb-0

-   Use the `--selector` (`-l`) flag to print logs from all pods that match a selector:

        kubectl logs -l app=ghost

-   If a pod's container was killed and restarted, you can view the previous container's logs with the `--previous` or `-p` flag:

        kubectl logs -p ghost-0

### kubectl exec

You can run arbitrary commands on a pod's container by passing them to kubectl's [`exec` command](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#exec):

    kubectl exec mariadb-0 -- ps aux

The full syntax for the command is:

    kubectl exec ${POD_NAME} -c ${CONTAINER_NAME} -- ${CMD} ${ARG1} ${ARG2} ... ${ARGN}

{{< note >}}
The `-c` flag is optional, and is only needed when the specified pod is running more than one container.
{{< /note >}}

It is possible to run an interactive shell on an existing pod/container. Pass the `-it` flags to `exec` and run the shell:

    kubectl exec -it mariadb-0 -- /bin/bash

Enter `exit` to later leave this shell.

### Viewing Master and Worker Logs

If the Kubernetes API server isn't working normally, then you may not be able to use `kubectl` to troubleshoot. When this happens, or if you are experiencing other more fundamental issues with your cluster, you can instead log directly into your nodes and view the logs present on your filesystem.

#### Non-systemd systems

If your nodes do not run [systemd](/docs/quick-answers/linux-essentials/what-is-systemd/), the location of logs on your master nodes should be:

- `/var/log/kube-apiserver.log` - [API server](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
- `/var/log/kube-scheduler.log` - [Scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/ )
- `/var/log/kube-controller-manager.log` - [Replication controller manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)

    {{< note >}}
You will not be able to directly access the master nodes of your cluster if you are using Linode's LKE service.
{{< /note >}}

On your worker nodes:

- `/var/log/kubelet.log` - [Kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
- `/var/log/kube-proxy.log` - [Kube proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)

#### systemd systems

If your nodes run systemd, you can access the logs that kubelet generates with journalctl:

    journalctl --unit kubelet

Logs for your other Kubernetes software components can be found through your container runtime. When using Docker, you can use the `docker ps` and `docker logs` commands to investigate. For example, to find the container running your API server:

    docker ps | grep apiserver

The output will display a list of information separated by tabs:

    2f4e6242e1a2    cfdda15fbce2    "kube-apiserver --au…"  2 days ago  Up 2 days   k8s_kube-apiserver_kube-apiserver-k8s-trouble-1-master-1_kube-system_085b2ab3bd6d908bde1af92bd25e5aaa_0

The first entry (in this example: `2f4e6242e1a2`) will be an alphanumeric string, and it is the ID for the container. Copy this string and pass it to `docker logs` to view the logs for your API server:

    docker logs ${CONTAINER_ID}

## Troubleshooting Examples

### Viewing the Wrong Cluster

If your `kubectl` commands are not returning the resources and information you expect, then your client may be assigned to the wrong cluster context. To view all of the cluster contexts on your system, run:

    kubectl config get-contexts

An asterisk will appear next to the active context:

    CURRENT   NAME                                        CLUSTER            AUTHINFO
              my-cluster-kayciZBRO5s@my-cluster           my-cluster         my-cluster-kayciZBRO5s
    *         other-cluster-kaUzJOMWJ3c@other-cluster     other-cluster      other-cluster-kaUzJOMWJ3c

To switch to another context, run:

    kubectl config use-context ${CLUSTER_NAME}

E.g.:

    kubectl config use-context my-cluster-kayciZBRO5s@my-cluster

### Can't Provision Cluster Nodes

If you are not able to create new nodes in your cluster, you may see an error message similar to:

{{< output >}}
Error creating a Linode Instance: [400] Account Limit reached. Please open a support ticket.
{{< /output >}}

This is a reference to the total number of Linode resources that can exist on your account. To create new Linode instances for your cluster, you will need to either remove other instances on your account, or request a limit increase. To request a limit increase, [contact Linode Support](/docs/platform/billing-and-support/support/#contacting-linode-support).

### Insufficient CPU or Memory

If one of your pods requests more memory or CPU than is available on your worker nodes, then one of these scenarios may happen:

-  The pod will remain in the Pending state, because the scheduler cannot find a node to run it on. This will be visible when running `kubectl get pods`.

    If you run the `kubectl describe` command on your pod, the Events section may list a `FailedScheduling` event, along with a message like `Failed for reason PodExceedsFreeCPU and possibly others`. You can run `kubectl describe nodes` to view [information about the allocated resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#my-pods-are-pending-with-event-message-failedscheduling) for each node.

-   The pod may continually crash. For example, the [Ghost](https://ghost.org) pod specified by [Ghost's Helm chart](https://github.com/helm/charts/tree/master/stable/ghost) will show the following error in its logs when not enough memory is available:

        kubectl logs ghost --tail=5
        1) SystemError

        Message: You are recommended to have at least 150 MB of memory available for smooth operation. It looks like you have ~58 MB available.

If your cluster has insufficient resources for a new pod, you will need to:

-   Reduce the number of other pods/deployments/applications running on your cluster,
-   [Resize the Linode instances](/docs/platform/disk-images/resizing-a-linode/) that represent your worker nodes to a higher-tier plan, or
-   Add a new worker node to your cluster.