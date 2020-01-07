---
title: "Google Kubernetes Engine — HorizontalPodAutoscaler with external metrics from PubSub"
author: "Marek Bartík"
date: 2019-04-05T12:31:58.932Z
lastmod: 2020-01-07T09:02:45+01:00

description: ""

subtitle: "There are certain use cases where scaling horizontally based on cpu usage does not really work well. Let’s say you have a consumer worker…"

image: "/posts/2019-04-05_google-kubernetes-enginehorizontalpodautoscaler-with-external-metrics-from-pubsub/images/1.png" 
images:
 - "/posts/2019-04-05_google-kubernetes-enginehorizontalpodautoscaler-with-external-metrics-from-pubsub/images/1.png" 
 - "/posts/2019-04-05_google-kubernetes-enginehorizontalpodautoscaler-with-external-metrics-from-pubsub/images/2.png" 
 - "/posts/2019-04-05_google-kubernetes-enginehorizontalpodautoscaler-with-external-metrics-from-pubsub/images/3.png" 


aliases:
    - "/google-kubernetes-engine-horizontalpodautoscaler-with-external-metrics-from-pubsub-28780c300305"
---

There are certain use cases where scaling horizontally based on cpu usage does not really work well. Let’s say you have a consumer worker pool running in Kubernetes. The consumers are pulling messages from a PubSub topic. When the queue is filling up we want more workers to process the messages quickly. On the other hand, when the queue is empty, we don’t want to pay for a big worker pool that sits idle. With PubSub Stackdriver metrics adapter running on GKE we can easily autoscale our worker pool for minimum latency and maximum cost-effectivity.### Autoscaling Deployments with External Metrics

This tutorial demonstrates how to automatically scale your GKE workloads based on metrics available in [Stackdriver](https://cloud.google.com/stackdriver/).

If you want to autoscale based on metric exported by your Kubernetes workload or a metric attached to Kubernetes object such as Pod or Node visit [Autoscaling Deployments with Custom Metrics](https://cloud.google.com/kubernetes-engine/docs/tutorials/custom-metrics-autoscaling) instead.

This example shows autoscaling based on number of undelivered messages in a [Cloud Pub/Sub](https://cloud.google.com/pubsub/) subscription, but the instructions can be applied to any metric available in Stackdriver.




![image](/posts/2019-04-05_google-kubernetes-enginehorizontalpodautoscaler-with-external-metrics-from-pubsub/images/1.png)

Stackdriver Cloud Pub/Sub Monitoring



### Provision GCP resources

We’ll be using terraform here to provision all necessary GCP resources. The cluster and nodepool’s definition is in file [main.tf](https://raw.githubusercontent.com/marekaf/gke-hpa-stackdriver-pubsub/master/main.tf). Make sure to follow all the steps in README to create a service account for terraform with all necessary permissions to create all the resources.

Then run:
``terraform init  
terraform plan -out planfile  
terraform apply planfile``

The PubSub topic will be named “echo”, the subscription to it “echo-read”. If you’ve run terraform apply successfully, this is provisioned already.
`resource &#34;google_pubsub_topic&#34; &#34;echo&#34; {  
  name = &#34;echo&#34;  
}  

resource &#34;google_pubsub_subscription&#34; &#34;echo&#34; {  
  name  = &#34;echo-read&#34;  
  topic = &#34;${google_pubsub_topic.echo.name}&#34;  

  ack_deadline_seconds = 20  
}`

### Deploy Stackdriver metrics adapter

Make sure you have kubectl installed and you can [access the cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl).

Deploy the stackdriver adapter:
`kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/production/adapter.yaml`

### Deploy the HPA and deployment

Here’s how the HPA’s definition look like:
`apiVersion: autoscaling/v2beta1  
kind: HorizontalPodAutoscaler  
metadata:  
  name: pubsub  
spec:  
  minReplicas: 1  
  maxReplicas: 5  
  metrics:  
  - external:  
      metricName: pubsub.googleapis.com|subscription|num_undelivered_messages  
      metricSelector:  
        matchLabels:  
          resource.labels.subscription_id: echo-read  
      targetAverageValue: &#34;2&#34;  
    type: External  
  scaleTargetRef:  
    apiVersion: apps/v1  
    kind: Deployment  
    name: pubsub`

We’ll autoscale between 1–5 replicas, based on external metric _pubsub.googleapis.com|subscription|num_undelivered_messages_ from our _echo-read_ subscription.

Target value is 2 undelivered messages. What does it actually mean though?

Example: let’s say my deployment is currently running **3** replicas and my queue grows from 6 to 8 undelivered messages.  
I have 8/**3**=2.6 undelivered messages per replica.  
That hits the threshold and triggers a scale-out to **4** replicas, which will have 8/**4**=2 undelivered messages per replica and that fits the desired targetAverageValue.  
If I had 50 undelivered messages, I will have 5 replicas as it’s my maximum.  
If I had 0 undelivered messages, I will have 1 replica as it’s my minimum.

The _scaleTargetRef_ is a reference of the resource that I’m autoscaling. It’s a deployment that is defined in file [pubsub-deployment.yaml](https://raw.githubusercontent.com/marekaf/gke-hpa-stackdriver-pubsub/master/pubsub-deployment.yaml).

Deploy the HPA with the deployment that is going to be autoscaled:
``kubectl apply -f  pubsub-hpa.yaml  
kubectl apply -f  pubsub-deployment.yaml``

### Test it!

Publish some messages to the topic
`for i in {1..200}; do   
  gcloud pubsub topics publish echo --message=”Autoscaling #${i}”  
done`

And watch the cluster’s resources doing its magic
`watch &#39;kubectl get pods; echo ; kubectl get hpa&#39;`



![image](/posts/2019-04-05_google-kubernetes-enginehorizontalpodautoscaler-with-external-metrics-from-pubsub/images/2.png)

scale-out on saturated queue





![image](/posts/2019-04-05_google-kubernetes-enginehorizontalpodautoscaler-with-external-metrics-from-pubsub/images/3.png)

scale-in on empty qeue



### Summary

With this simple setup you have a pretty decent setup for horizontal autoscaling. The ugly thing is running the stackdriver adapter yourself, at least the HPA controller is part of GKE and is fully managed for you.   
The other cool thing about HPA is that [you can use multiple metrics](https://cloud.google.com/blog/products/gcp/beyond-cpu-horizontal-pod-autoscaling-comes-to-google-kubernetes-engine) (even a combination of custom/external/cpu) in the same HPA resource and your deployment is going to be scaled based on either of them hitting a threshold.

### Links

[https://github.com/marekaf/gke-hpa-stackdriver-pubsub](https://github.com/marekaf/gke-hpa-stackdriver-pubsub)

[https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform](https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform)

[https://cloud.google.com/kubernetes-engine/docs/tutorials/external-metrics-autoscaling](https://cloud.google.com/kubernetes-engine/docs/tutorials/external-metrics-autoscaling)
