# UNCHECKED REDIRECTS
An unchecked redirect is when a web application performs a redirect to an arbitrary URL outside the application. This may seem entirely useless at first. But what if you have a page that is using referrer checks for authorization (never do this), for instance?
#### Simple Scenario
An attacker creates an identical clone of your site, but instead of authenticating against your database, she just dumps login credentials to a file, then redirects back to your site.
With an unchecked redirect, an attacker could send victims a link to *your* site, which then sends them to the evil site to steal their credentials.
Unless the victims look at the URL *after* the redirect, they'll never notice the problem.
#### Detection
Any time you see a redirect, look for the origin of the destination. Often, they come from a user session which is not exploitable. If you find that it's coming from the browser in an unsafe way -- for instance, as part of a CSRFable POST request -- you probably have an exploitable case.
#### Mitigation
Mitigation of this is simple but easy to get wrong.
One way to fix this is to not allow protocol specification in the destination. That is, remove instances of '`http://`' and the like. This will mean that -- at worst -- a redirect can only cause a `404`.
Another common mitigation is to do away with the redirect destination entirely, by constructing it on the server side from data the client sends. This is always the much safer option.
# PASSWORD STORAGE
