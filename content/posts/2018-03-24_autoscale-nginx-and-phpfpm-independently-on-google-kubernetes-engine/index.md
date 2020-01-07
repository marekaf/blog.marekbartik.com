---
title: "Autoscale nginx and php-fpm independently on Google Kubernetes Engine"
author: "Marek Bartík"
date: 2018-03-24T15:43:47.222Z
lastmod: 2020-01-07T09:02:28+01:00

description: ""

subtitle: "Let’s say I have a stateless php app that needs to run 24/7 and automatically scale up to perform well and scale down to be cost-effective…"

image: "/posts/2018-03-24_autoscale-nginx-and-phpfpm-independently-on-google-kubernetes-engine/images/1.jpeg" 
images:
 - "/posts/2018-03-24_autoscale-nginx-and-phpfpm-independently-on-google-kubernetes-engine/images/1.jpeg" 
 - "/posts/2018-03-24_autoscale-nginx-and-phpfpm-independently-on-google-kubernetes-engine/images/2.png" 
 - "/posts/2018-03-24_autoscale-nginx-and-phpfpm-independently-on-google-kubernetes-engine/images/3.png" 


aliases:
    - "/autoscale-nginx-and-php-fpm-independently-on-google-kubernetes-engine-290aead64f7f"
---

Let’s say I have a stateless php app that needs to run 24/7 and automatically scale up to perform well and scale down to be cost-effective. Perfect use case for Kubernetes, right? (Let’s be honest here now — you would use Kubernetes anyway because all the cool kids do it these days).

_DISCLAIMER (8 months after using this in prod): We separated nginx and php-fpm to their own k8s deployments and it only brought us so many problems, complexity and inconsistencies that we rolled back every prod to using one deployment with 2 containers — nginx and php-fpm. The benefits described here it brings did not outweigh the complexity it brought FOR US. It might be beneficial for you, though. It is definitely something to think about when designing your architecture._

Cool then, let’s start the naive way —

### Solution 1:
> _FROM debian  
> RUN apt-get update &amp;&amp; apt-get -y install nginx php-fpm_



![image](/posts/2018-03-24_autoscale-nginx-and-phpfpm-independently-on-google-kubernetes-engine/images/1.jpeg)

I dare you!



Ok ok, let’s just forget about this and start a bit better, shall we?

### Solution 2:

Separate the nginx a php-fpm into two images
> _FROM nginx:1.13  
> COPY src/ /code/  
> COPY site.conf /etc/nginx/conf.d/default.conf_

notice the fastcgi_pass and try_files



Then build the php image
> _FROM php:7.1-fpm  
> COPY src/ /code/_

Now make a simple **kubernetes deployment** with two containers — nginx and php, the communicate via the pod’s shared network using tcp port 9000(fastcgi_pass localhost:9000)

_DISCLAIMER: although we are using tcp socket here, you should probably use unix socket instead (using shared volume between containers)_

Great, now **we can scale horizontally and vertically** with ease. Just set the number of replicas and create a horizontal pod autoscaler. Don’t forget to set the **containers’ cpu/mem quotas**.

But wait. What if my php app is very cpu intensive in peaks and I need to run a lot of php replicas but nginx, on the other hand, just handles a couple of requests and two/three replicas are more than enough just to satisfy the HA needs?

#### Problem 1:

**Horizontal scaling** scales both php and nginx, even if only one of them needs to be scaled out.

#### Problem 2:

What more, I need to build **two docker images with the same application code**. That just doesn’t seem right. Deployments/rollbacks need to change both image tags at the same time otherwise we have an undefined state. And god kills a kitten.

### Solution 3:

Nginx doesn’t know php. There is no need to have the __ **src/** in nginx image. Let’s use the standard nginx image and put the **source code in php image only.**

We had one pod with both nginx and php, now we want both of them have their own pod. Make **two kubernetes deployments** and kubernetes service for php pods.
> _FROM nginx:1.13  
> COPY site.conf /etc/nginx/conf.d/default.conf_

since the php script are not longer in nginx, delete the _try_files_ directive, next change the _fastcgi_pass_ to the kubernetes service’s local dns (I named it _php_):
> _#try_files $uri =404;  
> fastcgi_pass php:9000;_

The kubernetes service called _php_ loadbalances tcp traffic internally between pods labeled _php_. A local dns record _php_ is created automatically for that internal loadbalancer. Nginx now sends the requests to the balanced pods group.

Now we can scale nginx and php-fpm independently and the app code is in php image only.




![image](/posts/2018-03-24_autoscale-nginx-and-phpfpm-independently-on-google-kubernetes-engine/images/2.png)



check the php logs for loadbalancing:




![image](/posts/2018-03-24_autoscale-nginx-and-phpfpm-independently-on-google-kubernetes-engine/images/3.png)



### Solution 4:

Is there a better solution for GKE?

Is it a bad idea to get rid of the try_files in nginx.conf? Should I even use nginx at all? (let’s assume I’m serving the static assets from a different source)

How does this work? Why am I getting nginx’s IP address with this —
> _&lt;?php echo $_SERVER[‘SERVER_ADDR’]; ?&gt;_

Feel free to discuss this with me, I’m having a hard time finding anything useful for running php on GKE. Would be nice to get hands on some best practices! Comment below, write me a DM or submit an issue. Thanks!

[bartimar/gke-php-test](https://github.com/bartimar/gke-test)
