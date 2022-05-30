---
title: "Web Security"
draft: false
---

# Web Security

## Same Origin Policy

Two pages from different sources should not be allowed to interface with each other! This is the fundamental security model of the web. The browser is analogous to an OS kernel, an origin is analogous to an OS process, so site rely on the browser to enfoce all the system security rules.

### Definition
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

Sometimes policy is too narrow, it's difficult to get `login.stanford.edu` and `axess.stanford.edu` to exchange data; but sometimes it's too broad, there is no way to isolate `https://web.stanford.edu/class/cs106a/` and `https://web.stanford.edu/class/cs253/`. We need a way around Same Origin Policy to allow two different origins to communicate.

### Cookie
If two sites have a top-level domain, different sub domain, e.g.`login.stanford.edu` and `axess.stanford.edu`, then they can define:
```
document.domain = 'stanford.edu';
document.cookie = "userid=test1";
```
and the axess site can read the cookie. More examples:
|Originating URL|document.domain|Accessed URL|document.domain|Allowed|
|----|----|----|----|----|
|http://www.a.com/|a.com|http://pay.a.com/|a.com|YES|
|http://www.a.com/|a.com|https://pay.a.com/|a.com|NO|
|http://pay.a.com/|a.com|http://a.com/|(not set)|NO|
|http://www.a.com/|(not set)|http://www.a.com/|a.com|NO|

Another way to do this is to use server response:
```
Set-Cookie: userid=test1; domain=.stanford.edu; path=/
```
Both resolution can't resolve the `attack.stanford.edu` access content from `axess.stanford.edu`, so it's bad!

### iframe
If two sites not match Same Origin Policy, they can't get each other's DOM. For example, when you use `iframe` or `window.open` to open a new child window, they can not communicate with the parent window. 
```
document.getElementById("myIFrame").contentWindow.document
// Uncaught DOMException: Blocked a frame from accessing a cross-origin frame.
```
There are 3 ways to resolve this situation.

#### fragment identifier
For a url `http://example.com/x.html#fragment`, the content after `#` is fragment, when you change the fragment, the page does not refresh.

So the parent window can write data to child window's fragment like this:
```
var src = originURL + '#' + data;
document.getElementById('myIFrame').src = src;
```
Then the child window can monitor an event named `hashchange` to get the notice:
```
window.onhashchange = checkMessage;
function checkMessage() {
  var message = window.location.hash;
  // ...
}
```
The child window also can change parent window's fragment:
```
parent.location.href= target + "#" + hash;
```


### AJAX