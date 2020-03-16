---
layout: post
title: AWS Application Load Balancer caveats
categories:
author: amit
---


Recently, i was testing an api, which is deployed on our staging machine.
The postman request looked something like this.
![alt text](https://i.imgur.com/YpzgFI0.png)

So after sending the request the respose body said.

```
{
    "message": "Method not allowed on this route"
}
With Status: 405 Method Not Allowed
```
So I thought I made a mistake in my code and the request method should be POST instead of any other, but it wasn't the case.
then i checked the logs on the staging machine which is deployed on aws.

The logs looked something like this.
```
INFO -- : Started GET "/api/v5/trip_events"
```
The logs were showing that a GET request on this url was sent, but i was sending a POST request.
Then i tested the api with ```https``` instead of ```http``` in the url and the request returned expected response.


So there was something happening between postman sending the request and rails server receiving the request..
We were using AWS Application load balancer for proxying and performing HTTP(80) to HTTPS(443) redirects.
So when a request is made via HTTP the proxy returns ```302``` redirect to HTTPS.

While reading about HTTP status Codes on https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.3.3

I found a note for status `302 found`.

```
Note: RFC 1945 and RFC 2068 specify that the client is not allowed
      to change the method on the redirected request.  However, most
      existing user agent implementations `treat 302 as if it were a 303
      response, performing a GET on the Location field-value regardless
      of the original request method`. The status codes 303 and 307 have
      been added for servers that wish to make unambiguously clear which
      kind of reaction is expected of the client.
```

So, what was happening with the request ?
1. a POST request is sent via HTTP
2. gets redirected to HTTPS with 302, it gets treated as `303`
3. and changes its request method sent via HTTPS to GET.

The Solution..
returning 307 instead of 302 on the application load balancer.

Unfortunately AWS ALB does not support the proper HTTP status code which we need to redirect POST to POST, which is `307`.

Share your thoughts about this in the comments section below.
