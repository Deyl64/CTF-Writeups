# XSS AND AUTHORIZATION (CTF)
#### **Types of XSS:*
- **Reflected XSS** - Input from a user is directly returned to the browser, permitting injection of arbitrary content. Usually paired with CSRF bugs.
- **Stored XSS** (**aka Persistent XSS**) - User input is stored on the server (often in a database) and returned later without proper escaping.
- **DOM XSS** - User input is inserted into the page's DOM without proper handling, enabling insertion of *arbitrary nodes*.
### XSS Vulnerability Check-list (for each input)
1. **Find out where the input goes**: Does it get embedded in a *tag attribute?* Does it get embedded into a *string in a script tag*?
2. **Discover any special handling**: Do URLs get turned into links, like posts in Hacker101 CTF Level 1?
3. **Figure out how special characters are handled**: A good way is to input something like: `<>:;"` 

 From these steps, you will know whether a given input is vulnerable to XSS. At this point one of the differences between *Stored XSS* and *Reflected XSS* becomes apparent:
 - ***rXSS* vulnerabilities are inherently dependant on *CSRF* vulnerabilities to be exploitable***, in the case of *POSTs*. If your only rXSS exists in a GET, you're fine, but you're dependant on CSRF otherwise.
 - If you're in the attribute of a tag and you can type a double quote, you almost certainly have an XSS vulnerability. If you can't, you're stuck in the attribute and it is probably secure.
 - If you're in a script or you can break out of the string? If not, it is probably secure.
 - Sometimes special characters will sometimes not go through if **not** as part of a tag. In cases like this, see if you can break out of an attribute and use a DOM event like OnMouseOver. Not all HTML tags support DOM events, but the vast majority do. One of the exceptions to DOM events in reflected XSS when DOM goes into a hidden input field, so it isn't possible to use `onmouseover` and the like.
 - One of the differences between reflected XSS and stored XSS comes into play If it's triggered on POST and it's reflected - proper CSRF mitigations will prevent this from being exploitable. If a POST is required for the attacker on a stored XSS though, for instance when changing text on a page, then CSRF mitigations don't matter at all.

**Simple testing for XSS / SQL injection vulnerabilities:**
- `"><h1>test</h1>`
- `'+alert(1)+'`
- `"onmouseover="alert(1)`
- `http://"onmouseover="alert(1)`
==If you can get any special characters in, you're probably going to win.==

Note on *attributes*: Attributes are only considered closed after their ending quote. *You can lead right into the next attribute immediately after an end quote* and you still have valid HTML.
### Exploitation Case 1
Angle brackets cannot be used in `a href` tags, but in this case DOM events work well. For example, a good one is `onmouseover` (such as `http://"onmouseover="alert(1);`) - which gives:
		`<a href="http://"onmouseover="alert(1);">...`
		Once the victim hovers over that link, you now have Javascript executing.
### Exploitation Case 2
If your input is being reflected in a script tag, there are several ways this can go wrong. For example:
	`<script>var token = 'user input here';</script>`
Normal HTML encoding does not properly mitigate this case for two reasons:
	1. HTML entities won't be parsed in Javascript, meaning the input will simply be wrong.
	2. Single quotes are rarely encoded as HTML entities.

Examples of specific payloads under HTML encoding and simple string escaping:
- HTML encoded payload: `'; alert(1); '`
	- Gives us a final script of: `<script>var token = "; alert(1);";</script>`
	- Meaning we have complete control over execution.
- JS string-escaped payload: `</script><script>alert(1);</script>`
	- Gives us a final script of: `<script>var token = '</script><script>alert(1);</script>';</script>`
	- Again, giving us complete control.
### Mitigation
It's enough to string escape angle brackets in addition to quotes and backslashes and replace them with their appropriate HEX escape; however, there are a multitude of cases where this will not be enough (e.g. when you are passing an integer value into a DOM event attribute of a variable in a script tag).
	i.e. Replace `<` with `\x3c` and `>` with `\x3e
Unless there is NO OTHER OPTION, user-controlled input should not end up in a script tag or inside of an. attribute for a DOM event. While it is possible to mitigate it as discussed, the likelihood of proper mitigation is next to nil.
### DOM XSS
- Differs from rXSS/sXSS in that *it doesn't depend on a server-side flaw to get attacker input into a page.* This means that through vulnerable Javascript on the client side, it's possible for an attacker to inject arbitrary content.

Here is an example of DOM XSS:
![[Pasted image 20240122141619.png]]

- The core problem with DOM XSS is that there are effectively an infinite number of ways which it can come about, each of which requiring different mitigations:
	- Embedding attacker data into `eval`/`setTimeout`/`setInterval` requires string escaping/filtering
	- Embedding attacker data into tags and attributes requires HTML encoding
	- Same goes for innerHTML
	
	Don't put user-controlled data on the page! Whitelist very specific things, such as a list of valid locales for the flag. If user data must be put into a page, escape/encode for the specific context.
### Forced Browsing/Improper Authorization
In both cases there is a failure to properly authorize access to a resource, e.g. an admin area is left unprotected, or you're able to directly enumerate values in a request to access other users' data.
**Forced Browsing** is the term used when you're talking about enumerable values such as post IDs and other parts of the site that are not ordinarily available to you from your privilege level - though some people combine the terms for both to "*authorization bugs*" (or *auth-z*, to differentiate from *auth-n*, authentication).

Changing a post ID # which leads to seeing another users post is an example of this (i.e. `http://somewebsite/directory/1/post?id=666`).

Another way to find auth-z bugs is to perform every action you can as the highest-privileged user, then switch to a lower-privileged user and relay those requests, changing session IDs/CSRF token as needed.

