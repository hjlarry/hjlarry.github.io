---
title: "Web Security"
draft: false
---

# Web Security

## Same Origin Policy

Two pages from different sources should not be allowed to interface with each other! This is the fundamental security model of the web. The browser is analogous to an OS kernel, an origin is analogous to an OS process, so site rely on the browser to enfoce all the system security rules.

### Definition
The origin means `protocal-host-port` tuple, this can be described as:
{{< highlight python>}}
def isSameOrigin(url1, url2):
    return url1.protocol == url2.protocal and url1.hostname == url2.hostname and url1.port == url2.port
{{< /highlight >}}

If two sites do not comply with same origin policy, they can't:
  - Read each other's Cookie,LocalStorage,IndexdDB
  - Get each other's DOM
  - Send to each other AJAX request

But they can embed static resources from another origin, like Images,Scripts,Styles.

Sometimes policy is too narrow, it's difficult to get `login.stanford.edu` and `axess.stanford.edu` to exchange data; but sometimes it's too broad, there is no way to isolate `https://web.stanford.edu/class/cs106a/` and `https://web.stanford.edu/class/cs253/`. We need a way around Same Origin Policy to allow two different origins to communicate.

### Cookie
If two sites have a top-level domain, different sub domain, e.g.`login.stanford.edu` and `axess.stanford.edu`, then they can define:
{{< highlight js>}}
document.domain = 'stanford.edu';
document.cookie = "userid=test1";
{{< /highlight >}}
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
{{< highlight js>}}
document.getElementById("myIFrame").contentWindow.document
// Uncaught DOMException: Blocked a frame from accessing a cross-origin frame.
{{< /highlight >}}
There are 3 ways to resolve this situation.

#### fragment identifier
For a url `http://example.com/x.html#fragment`, the content after `#` is fragment, when you change the fragment, the page does not refresh.

So the parent window can write data to child window's fragment like this:
{{< highlight js>}}
var src = originURL + '#' + data;
document.getElementById('myIFrame').src = src;
{{< /highlight >}}
Then the child window can monitor an event named `hashchange` to get the notice:
{{< highlight js>}}
window.onhashchange = checkMessage;
function checkMessage() {
  var message = window.location.hash;
  // ...
{{< /highlight >}}
The child window also can change parent window's fragment:
{{< highlight js>}}
parent.location.href= target + "#" + hash;
{{< /highlight >}}

#### window.name
Each browser window has a `window.name` property, this property has a quality, even if the page jump to a new location, the `window.name` will still be retained.

For example, the `http://parent.com/a.html` has a iframe src to `http://child.com`. When the child page loaded it can set it's `window.name=data`, and then use `location.href='http://parent.com/empty'`to redirect the same origin page as parent. Now the parent can get the data simple use `$('iframe').contentWindow.name`.

This property can exchange serveral MB data once, but monitor the child window name change is a performance loss.

#### window.postMessage
The above methods are hack method, the `window.postMessage` is a new cross-document messaging API design to resolve this situation. It can be used for complicated objects, handle cycles, read/write localstorage etc.

For example, when the parent window want to send a message to child:
{{< highlight js>}}
var popup = window.open('http://child.com', 'title');
popup.postMessage('Hello World!', 'http://child.com');
{{< /highlight >}}
The first parameter of `postMessage` method is data, the second is origin(`protocal-host-port` tuple), also can use `*` represent all window.  

The child also can send message to parent:
{{< highlight js>}}
window.opener.postMessage('Nice to see you', 'http://parent.com');
{{< /highlight >}}

They both can monitor message use a `message` event:
{{< highlight js>}}
window.addEventListener('message', function(event) {
  console.log(event.data);
},false);
{{< /highlight >}}

`event` object has 3 property, `event.data` is the content of this message, `event.source` means which window send this message, `event.origin` means which origin send this message. So you can filter the message which not send to you and send back directly which send to you.
{{< highlight js>}}
window.addEventListener('message', receiveMessage);
function receiveMessage(event) {
  if (event.origin !== 'http://parent.com') return;
  if (event.data === 'Hello World') {
      event.source.postMessage('Hello back', event.origin);
  } else {
    console.log(event.data);
  }
}
{{< /highlight >}}

### AJAX

## Session Attack

### Session and Cookie
The Http protocol is statusless, we need the server keeps a set of data related to a user's current browsing session, like Logins,Shopping carts,User tracking etc.The most common way is Cookies. Session desired these properties:

 - Browser remember user(so user doesn't need to repeatedly log in)
 - User cannot modify session cookie to login as another user
 - Session cookies are not valid forever
 - Session can be deleted on the server-side
 - Session should expire after some time, e.g. 30 days

When the server Set-Cookie, cookie have some basic attributes:

 - Expires, Specifies expiration date. If no date, then lasts for "browser session". If set to `Thu, 01 Jan 1970 00:00:00 GMT` will delete this cookie name and its value
 - Path, Scope the "Cookie" header to a particular request path prefix, e.g. `Path=/docs` will match `/docs` and `/docs/web`
 - Domain, Allows the cookie to be scoped to a "broader domain"(within the same registrable domain), e.g. `sub.a.com` can set cookies for `a.com`

The Cookies setting don't obey Same Origin Policy, because cookies were created before Same Origin Policy, they have different security model. Sometimes Cookies are more restrictive than Same Origin Policy, like `Path` partions cookies by path, but is ineffective, because pages on same origin can access each other's DOMs, run code in each other's contexts;The other time Cookies are less restrictive than Same Origin Policy, like Pages with same hostname can share the cookies, the protocol and port are ignored, and different origins(e.g. subdomain) can mess with each others cookies.

