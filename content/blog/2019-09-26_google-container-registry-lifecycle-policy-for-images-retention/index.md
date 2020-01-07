---
title: "Google Container Registry lifecycle policy for images retention"
author: "Marek Bartík"
date: 2019-09-26T09:32:41.706Z
lastmod: 2020-01-07T09:02:49+01:00

description: ""

subtitle: "Is your Google Container Registry filling up, taking up storage and becoming expensive? How to handle images retention as a service?"

image: "/posts/2019-09-26_google-container-registry-lifecycle-policy-for-images-retention/images/1.png" 
images:
 - "/posts/2019-09-26_google-container-registry-lifecycle-policy-for-images-retention/images/1.png" 


aliases:
    - "/google-container-registry-lifecycle-policy-for-images-retention-e0bc6bb88246"
---

Is your Google Container Registry filling up, taking up storage and becoming expensive? How to handle images retention as a service?

Amazon’s Elastic Container Registry has a feature called [Lifecycle Policies](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html) to handle images retention. Google doesn’t have this feature. There is a [feature request in their tracker](https://issuetracker.google.com/issues/113559510) since Aug 2018 and there is not ETA for it so far…

There is a popular [bash script from Ahmet](https://gist.github.com/ahmetb/7ce6d741bd5baa194a3fac6b1fec8bb7) and [Go in CloudRun from Seth](https://github.com/sethvargo/gcr-cleaner) but none of them solve the requirements I needed. What exactly do I need?

I wanna scan my whole GCR and delete the digests that are:

*   older than X days
*   not being used in my kubernetes cluster
*   not the most recent Y digests (I wanna keep, say, 20 most recent tagged digests)

When I check these requirements, I want to apply these lifecycle policies to all the images.

Say I have few images in GCR with certain prefixes:
`eu.gcr.io/my-project/foo/bar/my-service:123`

_eu.gcr.io_ is the docker registry endpoint  
_my-project_ is ID of my GCP project  
_foo/bar_ is the prefix (“repo”)  
_my-service_ is an image name  
_123_ is a tag

_my-service:123_ is an image with a tag, but wait, [what is the digest](https://windsock.io/explaining-docker-image-ids/)?




![image](/posts/2019-09-26_google-container-registry-lifecycle-policy-for-images-retention/images/1.png)

Image vs Layers, taken from [https://windsock.io/explaining-docker-image-ids/](https://windsock.io/explaining-docker-image-ids/)



A docker image digest is an ID (hashing algorithm used and the hash computed). The digest can look like this:

_@sha256:296e2378f7a14695b2f53101a3bd443f656f823c46d13bf6406b91e9e9950ef0_

You can tag a digest with several tags, even zero tags = untagged image.

Let’s say build an image _my-service_ and push it to docker registry. When pushing, I tag it with _:123._ The new produced digest has two tags,_:123_ and_:latest._The digest that was tagged _:latest_ before I pushed this image, got the _:latest_ tag removed.

If I remove a tag from an image in GCR, I simply remove a tag from the digest, I don’t delete the digest though.

What I can delete, in order to save some space, is the digest, like this:
`gcloud container images delete -q — force-delete-tags eu.gcr.io/my-project/foo/bar/my-service@sha256:296e2378f7a14695b2f53101a3bd443f656f823c46d13bf6406b91e9e9950ef0`

Then, what do I need to do?

*   Recursively scan _gcr.io_ for all image prefixes (eu.gcr.io/my-project/foo/bar/my-service)
*   For each prefix — list all its digests, delete the ones that don’t match my rules

How to check if they match the rules:

*   sort them, preserve the most recent Y digests
*   fetch pods and replicaSets’ image:tags (all of them, even the ones scaled to zero, we’d need these images in case of a rollback) from the k8s cluster, then go through the digests (that belong to that image name) and check if ANY of their tags contain a tag that is used in the cluster, preserve those
*   check the rest of the digests, if they are older than X days, delete them

you could use standard kubectl to fetch the data:
`kubectl get rs,po --all-namespaces -o jsonpath={..image} | tr &#39; &#39; &#39;\n&#39;`

the gcr.io is exposing a docker v2 API, you can use a standard docker client or just curl using the gcloud token
`ACCESS_TOKEN=$(gcloud auth print-access-token)  
curl --silent --show-error -u_token:&#34;$ACCESS_TOKEN&#34; -X GET &#34;https://eu.gcr.io/v2/_catalog&#34;`

I implemented all of this using _bash/jq_ (yep, that wasn’t a smart idea) and published it to github:

[marekaf/gcr-lifecycle-policy](https://github.com/marekaf/gcr-lifecycle-policy)


Right now I’m running this in Gitlab-CI pipeline on a cron schedule (once a day) to evaluate it’s dry-run logs for production GCP projects.

I’m planning on rewriting this to python (py-kubeng and docker-py) if Google will not to come up with ETA for this feature :(




![image](https://cdn-images-1.medium.com/max/800/0*Piks8Tu6xUYpF4DU)



**Follow us on** [**Twitter**](https://twitter.com/joinfaun) **** 🐦 **and** [**Facebook**](https://www.facebook.com/faun.dev/) **** 👥 **and join our** [**Facebook Group**](https://www.facebook.com/groups/364904580892967/) **** 💬**.**

**To join our community Slack** 🗣️ **and read our weekly Faun topics** 🗞️, **click here⬇**



[![image](https://cdn-images-1.medium.com/max/2560/0*oSdFkACJxs5iy1oR)](https://www.faun.dev/join/?utm_source=medium.com%2Ffaun&amp;utm_medium=medium&amp;utm_campaign=faunmediumbanner)

#### If this post was helpful, please click the clap 👏 button below a few times to show your support for the author! ⬇
