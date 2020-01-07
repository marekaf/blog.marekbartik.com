---
title: "Kubernetes in production — PodDisruptionBudget"
author: "Marek Bartík"
date: 2018-06-29T08:50:55.516Z
lastmod: 2020-01-07T09:02:32+01:00

description: ""

subtitle: "How to manage disruptions in Kubernetes? Setting a proper RollingUpdate strategy specs solves only one type of disruption."

image: "/posts/2018-06-29_kubernetes-in-production-poddisruptionbudget/images/1.png" 
images:
 - "/posts/2018-06-29_kubernetes-in-production-poddisruptionbudget/images/1.png" 


aliases:
    - "/kubernetes-in-production-poddisruptionbudget-1380009aaede"
---

How to manage disruptions in Kubernetes? Setting a proper RollingUpdate strategy specs solves only one type of disruption.

What about other disruptions?




![image](/posts/2018-06-29_kubernetes-in-production-poddisruptionbudget/images/1.png)

my baby’s on fire [source: google images]

### Voluntary and Involuntary Disruptions

Pods do not disappear until someone (a person or a controller) destroys them, or there is an unavoidable hardware or system software error.

We call these unavoidable cases _involuntary disruptions_ to an application. Examples are:

*   a hardware failure of the physical machine backing the node
*   cluster administrator deletes VM (instance) by mistake
*   cloud provider or hypervisor failure makes VM disappear
*   a kernel panic
*   the node disappears from the cluster due to cluster network partition
*   eviction of a pod due to the node being [out-of-resources](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/).

Except for the out-of-resources condition, all these conditions should be familiar to most users; they are not specific to Kubernetes.

We call other cases _voluntary disruptions_. These include both actions initiated by the application owner and those initiated by a Cluster Administrator. Typical application owner actions include:

*   deleting the deployment or other controller that manages the pod
*   updating a deployment’s pod template causing a restart
*   directly deleting a pod (e.g. by accident)

Cluster Administrator actions include:

*   [Draining a node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) for repair or upgrade.
*   Draining a node from a cluster to scale the cluster down (learn about [Cluster Autoscaling](https://kubernetes.io/docs/tasks/administer-cluster/cluster-management/#cluster-autoscaler) ).
*   Removing a pod from a node to permit something else to fit on that node.### Dealing with Disruptions

Here are some ways to mitigate involuntary disruptions:

*   Ensure your pod [requests the resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-ram-container) it needs.
*   Replicate your application if you need higher availability. (Learn about running replicated [stateless](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/) and [stateful](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/) applications.)
*   For even higher availability when running replicated applications, spread applications across racks (using [anti-affinity](https://kubernetes.io/docs/user-guide/node-selection/#inter-pod-affinity-and-anti-affinity-beta-feature)) or across zones (if using a [multi-zone cluster](https://kubernetes.io/docs/setup/multiple-zones).)

### PodDisruptionBudget

An Application Owner can create a `**PodDisruptionBudget**` object (PDB) for each application. A PDB limits the number pods of a replicated application that are down simultaneously from voluntary disruptions. For example, a quorum-based application would like to ensure that the number of replicas running is never brought below the number needed for a quorum. A web front end might want to ensure that the number of replicas serving load never falls below a certain percentage of the total.

Cluster managers and hosting providers should use tools which respect Pod Disruption Budgets by calling the [Eviction API](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/#the-eviction-api) instead of directly deleting pods. Examples are the `**kubectl drain**` command and the Kubernetes-on-GCE cluster upgrade script (`**cluster/gce/upgrade.sh**`).

When a cluster administrator wants to drain a node they use the `**kubectl drain**` command. That tool tries to evict all the pods on the machine. The eviction request may be temporarily rejected, and the tool periodically retries all failed requests until all pods are terminated, or until a configurable timeout is reached.

Example PDB Using minAvailable:
``**apiVersion: policy/v1beta1  
kind: PodDisruptionBudget  
metadata:  
  name: zk-pdb  
spec:  
  minAvailable: 2  
  selector:  
    matchLabels:  
      app: zookeeper**``

Example PDB Using maxUnavailable (Kubernetes 1.7 or higher):
``**apiVersion: policy/v1beta1  
kind: PodDisruptionBudget  
metadata:  
  name: zk-pdb  
spec:  
  maxUnavailable: 1  
  selector:  
    matchLabels:  
      app: zookeeper**``### Helm

use this in your Chart!

templates/pdb.yaml:
`**{{- if .Values.budget.minAvailable -}}**  
apiVersion: policy/v1beta1  
kind: PodDisruptionBudget  
metadata:  
  name: {{ template &#34;app.fullname&#34; . }}  
  namespace: {{ .Values.namespace }}  
  labels:  
    app: {{ template &#34;app.name&#34; . }}  
    chart: {{ .Chart.Name }}-{{ .Chart.Version | replace &#34;+&#34; &#34;_&#34; }}  
    release: {{ .Release.Name }}  
    heritage: {{ .Release.Service }}  
spec:  
  selector:  
    matchLabels:  
      app: {{ template &#34;app.name&#34; . }}  
      env: {{ .Values.env.name }}  
  minAvailable: **{{ .Values.budget.minAvailable }}**  
**{{- end -}}**`

Imagine you have a service with 2 replicas and you need at least 1 to be available even during node upgrades and other ops tasks.

install / upgrade your release:
`helm upgrade --install --debug &#34;$RELEASE_NAME&#34; -f helm/values.yaml \ **--set replicas=2,budget.minAvailable=1** myrepo/mychart`

kubectl describe pdb “$RELEASE_NAME”
`Name: mysvc-prod  
Namespace: prod  
Min available: 1  
Selector: app=myservice,env=prod  
Status:  
 **Allowed disruptions: 1  
 Current: 2  
 Desired: 1  
 Total: 2**  
Events: &lt;none&gt;`

drain a node with one of your pods running:
`kubectl drain --delete-local-data --force --ignore-daemonsets gke-mycluster-prod-pool-2fca4c85-k6g5``node &#34;gke-mycluster-prod-pool-2fca4c85-k6g5&#34; already cordoned  
WARNING: Deleting pods with local storage: sqlproxy-67f695889d-t778w; Ignoring DaemonSet-managed pods: fluentd-gcp-v3.0.0-llp5s; Deleting pods not managed by ReplicationController, ReplicaSet, Job, DaemonSet or StatefulSet: kube-proxy-gke-testing-dev-pool-2fca4c85-k6g5  
pod &#34;tiller-deploy-7b7b795779-rcvkd&#34; evicted  
**pod &#34;mysvc-prod-6856d59f9b-lzrtf&#34; evicted**  
node &#34;gke-mycluster-prod-pool-2fca4c85-k6g5&#34; drained`

again: kubectl describe pdb “$RELEASE_NAME”
`Name: mysvc-prod  
Namespace: prod  
Min available: 1  
Selector: app=myservice,env=prod  
Status:  
 **Allowed disruptions: 0  
 Current: 1  
 Desired: 1  
 Total: 2**  
Events: &lt;none&gt;`

Tadaaa! We drained a node without any disruptions of our service.

### PDB with 1 replica only?

If we had 1 replica only, the [**kubectl drain would get stuck**](https://github.com/kubernetes/kubernetes/issues/48307) **always**. Node drains / upgrades would need to be solved manually.

You might expect the eviction API would try to surge a replica to comply with the minAvailable condition, instead the drain gets stuck and it is your responsibility to solve this situation yourself. Is it a **bug or feature**? The Kubernetes community says you shouldn’t use 1 replica in production at all if you want HA, which is fair :)

It does what is expected, though.

If you don’t want your kubectl drains to get stuck, you might want to use PDB for deployments with more than 1 replica.

Edit your templates/pdb.yaml:
`{{- if .Values.budget.minAvailable -}}  
**{{- if gt .Values.replicaCount 1 -}}**  
apiVersion: policy/v1beta1  
kind: PodDisruptionBudget  
metadata:  
...  
**{{- end -}}**  
{{- end -}}`### How to perform Disruptive Actions on your Cluster

If you are a Cluster Administrator, and you need to perform a disruptive action on all the nodes in your cluster, such as a node or system software upgrade, here are some options:

#### Accept downtime during the upgrade.

#### Fail over to another complete replica cluster.

*   No downtime, but may be costly both for the duplicated nodes, and for human effort to orchestrate the switchover.

#### Write disruption tolerant applications and use PDBs.

*   No downtime.
*   Minimal resource duplication.
*   Allows more automation of cluster administration.
*   Writing disruption-tolerant applications is tricky, but the work to tolerate voluntary disruptions largely overlaps with work to support autoscaling and tolerating involuntary disruptions.Sources:  
docs [https://kubernetes.io/docs/concepts/workloads/pods/disruptions/](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)  
magic
