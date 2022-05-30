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

If two sites do not comply with same origin policy, they can't:
  - Read each other's Cookie,LocalStorage,IndexdDB
  - Get each other's DOM
  - Send to each other AJAX request

But they can embed static resources from another origin, like Images,Scripts,Styles.

Sometimes policy is too narrow, it's difficult to get `login.standford.edu` and `axess.standford.edu` to exchange data; but sometimes it's too broad, there is no way to isolate `https://web.stanford.edu/class/cs106a/` and `https://web.stanford.edu/class/cs253/`. We need a way around Same Origin Policy to allow two different origins to communicate.
