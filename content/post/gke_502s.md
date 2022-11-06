+++
aliases = ["posts","articles","blog","showcase"]
title = "GKE and the mysterious 502s"
author = "Aarushi Kansal"
+++

In the world of distributed systems, a mysterious 502 can easily become one of the most frustrating errors to debug. It's a generic error telling you something, somewhere's gone wrong. A single request goes through multiple hops, connections and other systems; a 502 tells you, your application encountered an error ANYWHERE in this web of connections and it's up to you to figure out where. 

In this post I want to share a very specific, issue I faced using Google Kubernetes Engine (GKE) and how I fixed it. I had a web app deployed on GKE, health checks passing, 99% of traffic going through with no issues. Every now and then I started getting complaints from other engineers and integrations testers about this mysterious "502" that was stopping them from using the app. 

Looking further into my application logs - I saw no errors, meaning these requests weren't even hitting the app. Next, I check the load balancer logs on stackdriver: 

```
resource.type="http_load_balancer"
httpRequest.status=502

```

This showed me the following error: 

``
backend_connection_closed_before_data_sent_to_client 
``

This error confused me, a lot 😕 - it indicated that my backend closed the connection, but all the requests were less than 10 seconds, and my backend services on Google Cloud Platform (GCP) were all set to a 30 second timeout. All of the pods CPU and memory usage stayed below 10%, so definitely not a resource issue.

Did some googling, found a LOT of results, none of them quite solved my exact issue. All the results did indicate there was definitely a timeout SOMEWHERE. I just needed to figure out where and why. 

Spent quite a few days (🤭 embarrassingly long time) on this, looking at different points of failures, including the NAT I was using. This investigation was only made more difficult by the fact that it was such an intermittent error and impossible to figure how to reliably replicate the problem. 

So, I used tshark and a bunch of load tests to actually observe the traffic and finally found out of about every ~100 or so requests, one would result in an ``RST`` from my actual application. 

Digging further into Google's documentation, it turns out the HTTP load balancer has a default timeout of 10 minutes (not configurable), but the http library I was using had a timeout of 10 seconds - basically this mismatch was resulting in a race condition. The load balancer would try and reuse a connection, but my application had already decided it was done with this connection, hence the RSTs(!!).

In the end, I increased the timeout on my application to 620 seconds, just more than the load balancer's timeout. This ensures that the server never closes the connection before the load balancer. 

And since then, haven't seen any more scary 502s. 🎉 
