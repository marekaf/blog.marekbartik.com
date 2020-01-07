---
title: "Google Kubernetes Engine + Cloud Builder + GitHub for easy and quick CD pipeline"
author: "Marek Bartík"
date: 2018-02-26T09:58:14.425Z
lastmod: 2020-01-07T09:02:22+01:00

description: "I love Google Cloud Platform. I love Google Kubernetes Engine. And I love GitHub as well. But I need a quick CD pipeline for an easy automated app deployment. Setting up the whole Jenkins CI or using…"

subtitle: "I love Google Cloud Platform. I love Google Kubernetes Engine. And I love GitHub as well. But I need a quick CD pipeline for an easy…"

image: "/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/2.png" 
images:
 - "/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/1.png" 
 - "/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/2.png" 
 - "/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/3.png" 
 - "/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/4.png" 
 - "/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/5.png" 
 - "/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/6.png" 


aliases:
    - "/google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline-8aca663f1118"
---

I love Google Cloud Platform. I love Google Kubernetes Engine. And I love GitHub as well. But I need a quick CD pipeline for an easy automated app deployment. Setting up the whole Jenkins CI or using some paid Pipeline as a service takes too much time, energy and money. Here comes the Google Cloud Builder (currently beta).

## Let's start

I have an easy stateless php app running on nginx and php-fpm in Docker containers versioned in git on GitHub. And I need is to deploy it quickly and cheap to GKE.

Open up `Build triggers`. It should be in `Google console — Tools — Container Registry — Build triggers` (but, you know, they change the UI like every week, so it might be somewhere else)
![image](/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/1.png)

Link your GitHub account with GCP
![image](/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/2.png)

Import your repo
![image](/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/3.png)

Set the trigger
![image](/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/4.png)

Choose the cloudbuild.yaml where you can define your [CI/CD pipeline as a code.](https://cloud.google.com/container-builder/docs/create-custom-build-steps)

## Let's test it

It is ugly, I know. I needed a fast solution, not a pretty one.

I need to build two Docker images, from different Dockerfiles, push them to Google Container Registry, template kubernetes deployment manifests, deploy them on GKE (kubectl apply) and create an ingress resource for external http traffic.

After I push to the repo, the build is automatically started
![image](/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/5.png)

I see the git commit and action which caused the trigger, pipeline steps, time they took, total time of the build, link to images in GCR. And log:
![image](/posts/2018-02-26_google-kubernetes-engine-google-cloud-builder-github-for-easy-and-quick-cd-pipeline/images/6.png)

## Summary
All is immutable, I can easily rollback deployment to a tagged image with a commit id from git. I can Rebuild the build in Build history. [Ingress DNS records are pushed to CloudFlare automatically.](https://medium.com/@marekbartik/google-kubernetes-engine-with-external-dns-on-cloudflare-provider-24beb2a6b8fc)

I had to create my own deployment/service/ingress/whatever kubernetes manifests and had to template them with sed (which is super ugly and the syntax won’t always work). Google Cloud Builder is in beta and there is definitely a lot to improve. Templating and GKE integration would be nice (manual kubectl apply, srsly?) to have. As well as deployment strategy selection and easier provisioning maybe?

Check the source repo: [https://github.com/bartimar/gke-test](https://github.com/bartimar/gke-test)

EDIT: Check this amazing github repo of the one and only Kelsey Hightower with great tutorials on Google Cloud Builder [https://github.com/kelseyhightower/pipeline](https://github.com/kelseyhightower/pipeline)
