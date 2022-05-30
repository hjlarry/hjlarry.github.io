---
title: "Web Security"
draft: false
---

# Web Security

## Same Origin Policy

Two pages from different sources should not be allowed to interface with each other! This is the fundamental security model of the web. The browser is analogous to an OS kernel, an origin is analogous to an OS process, so site rely on the browser to enfoce all the system security rules.

The origin means `protocal-host-port` tuple, this can be described as:
```
def isSameOrigin(url1, url2):
    return url1.protocol == url2.protocal and url1.hostname == url2.hostname and url1.port == url2.port
```
