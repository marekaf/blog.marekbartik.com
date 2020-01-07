---
title: "GCP Stackdriver Logging export to bucket and extract textPayload from json with Cloud Functions"
author: "Marek Bartík"
date: 2019-01-26T16:20:44.986Z
lastmod: 2020-01-07T09:02:42+01:00

description: ""

subtitle: "Stackdriver Logging can get expensive. Sometimes you don’t need to query/store all your application logs in Stackdriver, especially dev…"

image: "/posts/2019-01-26_gcp-stackdriver-logging-export-to-bucket-and-extract-textpayload-from-json-with-cloud-functions/images/1.png" 
images:
 - "/posts/2019-01-26_gcp-stackdriver-logging-export-to-bucket-and-extract-textpayload-from-json-with-cloud-functions/images/1.png" 
 - "/posts/2019-01-26_gcp-stackdriver-logging-export-to-bucket-and-extract-textpayload-from-json-with-cloud-functions/images/2.png" 


aliases:
    - "/gcp-stackdriver-logging-export-to-bucket-and-extract-textpayload-from-json-with-cloud-functions-4696798df803"
---

Stackdriver Logging can get expensive. Sometimes you don’t need to query/store all your application logs in Stackdriver, especially dev logs. Or you simply don’t like Stackdriver (wink wink). On GCP you can export your logs to Google Storage Bucket automatically. Every hour there is a json file containing your logs and its metadata. What if you need just the raw logs though?With Stackdriver Logging you pay for log ingestion. There’s also [log retention of 30 days only](https://cloud.google.com/logging/quotas#logs_retention_periods)! What if I want to store my logs for longer time? What if I don’t want to pay so much for my dev logs? What if I’m using a different tools for log analysis?

Stackdriver Logging has two additional services for this: Export and Exlusions. See the diagram below:




![image](/posts/2019-01-26_gcp-stackdriver-logging-export-to-bucket-and-extract-textpayload-from-json-with-cloud-functions/images/1.png)

Google Cloud Platform Logging Architecture, source: [https://cloud.google.com/logging/docs/export/](https://cloud.google.com/logging/docs/export/)



### Stackdriver Logging Exclusions

You can exclude logs based on their resource type or write more complex exclusion queries like:
> resource.type=”container”   
> AND labels.”container.googleapis.com/namespace_name”=”devel”   
> AND severity &gt; INFO

this excludes all GKE container logs from k8s namespace called “devel” that have severity greater than INFO (basically here just everything from stderr)

And you can set to exclude just 99% of those logs to be discarded! (or any other number from 0–100%).

### Stackdriver Logging Exports

The great thing about this architecture is, that Exports are independent of Exclusions. If I want to Export some logs to BigQuery or Bucket I can let them to be ingested by Stackdriver at the same time or Exclude them to save some money. So creating an Exclusion does not affect my Exports and that’s pretty dope.

Exports are either batch or streamed. Export to bucket is batched (every hour one file named with the timestamp is created) and this can’t be changed afaik. Exports to BigQuery and Pub/Sub are streamed. You will pay for the streamed volume to BigQuery or Pub/Sub pricing accordingly. In Either case you will pay for the storage as well.

### Export logs to Google Storage Bucket

When you create an automatic export of your logs to a bucket, you can expect two things:

Delay: around 10:10am you’ll get logs collected in time window of 9:00–10:00am and there’s nothing to guarantee that delivery time, sometimes it gets delayed even more. Not suitable for (near-)realtime log processing.

Log format: each line looks like this
> {“insertId”:”9pxrl4ffvybrn”,”labels”:{“compute.googleapis.com/resource_name”:”fluentd-gcp-v3.1.0–2bmxq”,”container.googleapis.com/namespace_name”:”prod”,”container.googleapis.com/pod_name”:”my-app-6b495f4d8b-9l2gb”,”container.googleapis.com/stream”:”stdout”},”logName”:”projects/my-project/logs/my-app”,”receiveTimestamp”:”2019–01–26T00:00:02.198052458Z”,”resource”:{“labels”:{“cluster_name”:”prod-cluster”,”container_name”:”myapp”,”instance_id”:”4164682295655914646&#34;,”namespace_id”:”prod”,”pod_id”:”my-app-6b495f4d8b-9l2gb”,”project_id”:”my-project”,”zone”:”europe-west1-b”},”type”:”container”},”severity”:”INFO”,”textPayload”:”2019–01–26 01:00:00.788 INFO [:] DispatcherServlet:102 — Request in 3155 ms: /my/endpoint/ : 1.2.3.4 : Mozilla/5.0 (compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm)\n&#34;,&#34;timestamp&#34;:&#34;2019-01-26T00:00:00Z&#34;}

You cannot do anything neither about the delay nor about the log format…

So what do we do if we want to parse out the textPayload of each json line and put that into a .txt file?

### Processing the exported logs with Cloud Functions

You can use [Cloud Functions](https://cloud.google.com/functions/) to process the exported log file in the bucket.

Use the` --trigger-event google.storage.object.finalize` to execute the Cloud Function when this event on a particular bucket happens. This basically means that changing any file in some bucket will create this event and Cloud Function will consume that event and fire up a function to process it. For other [supported event sources see the documentation](https://cloud.google.com/functions/features/).

We will use nodejs-6 runtime for the Cloud Function.
`&#39;use strict&#39;;  

const path = require(&#39;path&#39;);  
const fs = require(&#39;fs&#39;);  
const Storage = require(&#39;@google-cloud/storage&#39;);  
const readline = require(&#39;readline&#39;);  

const DEST_BUCKET_NAME = process.env.DEST_BUCKET_NAME;  

// Instantiates a client  
const storage = Storage();`

We initialized some global vars and imported libs to work with Google Cloud Storage

Next, we will define some helper functions:
`function getFileStream (file) {  
  if (!file.bucket) {  
    throw new Error(&#39;Bucket not provided. Make sure you have a &#34;bucket&#34; property in your request&#39;);  
  }  
  if (!file.name) {  
    throw new Error(&#39;Filename not provided. Make sure you have a &#34;name&#34; property in your request&#39;);  
  }  

  return storage.bucket(file.bucket).file(file.name).createReadStream();  
}`

Finally the main function:
`exports.logTransformer = (event, callback) =&gt; {  
  const file = event.data;  

  const filePath = file.name; // File path in the bucket.  
  const contentType = file.contentType; // File content type.  

  const destBucket = storage.bucket(DEST_BUCKET_NAME);  

  // Get the file name.  
  const fileName = path.basename(filePath);  

  const options = {  
    input: getFileStream(file)  
  };  

  // the output is no longer application/json but text/plain  
  const metadata = {  
    contentType: &#34;text/plain&#34;,  
  };  

  const logStream = fs.createWriteStream(&#39;/tmp/log.txt&#39;, {flags: &#39;w&#39;, encoding: &#39;utf-8&#39;});  

  logStream.on(&#39;open&#39;, function() {  

    readline.createInterface(options)  
    .on(&#39;line&#39;, (line) =&gt; {  

        const clonedData = JSON.parse(line.replace(/\r?\n|\r/g, &#34; &#34;));  
        logStream.write(clonedData.textPayload.toString());   

    })  
    .on(&#39;close&#39;, () =&gt; {  
      logStream.end(function () { console.log(&#39;logstream end&#39;); });  

    });  
  }).on(&#39;finish&#39;, () =&gt; {  

     // We replace the .json suffix with .txt   
     const thumbFileName = fileName.replace(&#39;.json&#39;,&#39;.txt&#39;);  
     const thumbFilePath = path.join(path.dirname(filePath), thumbFileName);  

    // Uploading the thumbnail.  
    destBucket.upload(&#39;/tmp/log.txt&#39;, {  
      destination: thumbFilePath,  
      metadata: metadata,  
      resumable: false  
    });  

    callback();  
  });  
};`

And that’s about it! Save that to index.js a don’t forget to include dependencies in package.json.

### Deploy it

Simply deploy it via gcloud CLI
``SOURCE_BUCKET_NAME=my-bucket-exported-logs  
DEST_BUCKET_NAME=my-bucket-exported-logs-nojson  

gcloud beta functions deploy logTransformer \   
   --set-env-vars DEST_BUCKET_NAME=&#34;$DEST_BUCKET_NAME&#34; \  
   --runtime nodejs6 \  
   --region europe-west1 \  
   --memory 128MB  \  
   --trigger-resource &#34;$SOURCE_BUCKET_NAME&#34; \  
   --trigger-event google.storage.object.finalize``



![image](/posts/2019-01-26_gcp-stackdriver-logging-export-to-bucket-and-extract-textpayload-from-json-with-cloud-functions/images/2.png)

Cloud Functions come with handy out-of-box monitoring dashboard — Execution Time here



Don’t forget to check the ‘View Logs’ (if you didn’t exclude them of course) for troubleshooting.

This month we’ve processed 14GiB with it so far and it’s still covered by the free tier Cloud Functions usage ❤

Source code: [https://github.com/marekaf/gcp-stackdriver-logging-extract-textpayload](https://github.com/marekaf/gcp-stackdriver-logging-extract-textpayload/blob/master/README.md)
