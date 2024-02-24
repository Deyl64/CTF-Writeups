# SQL INJECTION, COMMAND INJECTION, DIRECTORY TRAVERSAL
#### Directory Traversal
Directory Traversal is almost a 'path injection' attack. By controlling path construction, you're able to walk up the filesystem tree and control where files are being read/written.
##### Vulnerable PHP Example
	`<?php echo file_get_contents('/var/www/sandbox/uploads/' . $_GET['file']); ?>`
##### Consider:
`http://badsite.org/get_upload.php?file=../../../../etc/passwd`
*This would be read as:*
`/var/www/sandbox/uploads/../../etc/passwd`
`/var/www/sandbox/uploads/../etc/passwd`
`/var/www/sandbox/../../etc/passwd`
`...`
==`/etc/passwd`==
**Obviously this could be very dangerous!**
#### Command Injection
You're testing an appliance with a web interface for administration. As part of the interface, you have access to a ping function to test its ability to call out. You tell it to ping `google.com` and you see:
![[Pasted image 20240122144532.png]]
Looks an awful lot like the output that the actual ping command gives you on a UNIX-like system, right? What if they are running the equivalent of the following?
	`<?php echo shell_exec('ping -c 4 ' . $_GET['hostname']); ?>`
If this is the case, we can try using other commands. So you try to ping the hostname "`google.com;echo test`" and see that it silently gives back an empty page. Any use of `;` or `&` does so. How about trying to use backticks to utilize the output of a command? You discover that this also works. It is clear that it may take some work, but you could do anything at all, by embedding subcommands and watching how it affects the output of ping and the shell. From here, *you own the system*.

==**Read a full posting about an example of the bug from "Yoggie Pico Pro", including a proof of concept that swapped the root password for a new one and opened up SSHd on a port of my choosing (this was on a brand new security appliance, not something saddled with legacy code, etc. - this is by no means a rare case):**==
		http://seclists.org/fulldisclosure/2007/Jul/20
##### Mitigation
Expectedly, for most injection bugs, the safest route is just to never embed user data into a command line at all. If you must, then you should use shell escaping, e.g. `escapeshellcmd()` in PHP (The function escapes `#&;|*?~<>^()[]{}$\`, `\x0A`, `\xFF`, and any unbalanced quotes. However, it *doesn't prevent the use of spaces*, so if user input isn't quoted, it could very well add new arguments to the command!)
### SQL INJECTION
For the purpose of this, consider these 3 types of SQL queries:
- `SELECT some,columns,here FROM some_table WHERE some > columns AND here != 0;`
- `UPDATE some_table SET some=1, columns=2, here=3, WHERE id=5;`
- `INSERT INTO some_table (some, columns, here) VALUES (1, 2, 3);`
Applications build up SQL queries *as strings*, dispatch them to the database, then process the result set (if any).
Query building is where issues occur.

In this example, we're building a query directly using user input, which makes it a perfect target for SQLi (let's say Bobby entered his name in the school computer as `Robert'); DROP TABLE Students;--')` ):
![[Screenshot 2024-01-22 at 3.05.46 PM.png]]
Add Bobby's name, and you get the query:
	`SELECT age, grade, teacher FROM students WHERE (name='Robert'); DROP TABLE Students;--')`
	(Which is really three statements, a select, a drop, and a comment.)
	This is a very straightforward example of SQLi.
#### MITIGATION
SQLi mitigation is generally fairly easy, with a few options:
1. *Ensure all strings are properly quoted *and run through the appropriate escaping function, e.g.. `mysql_real_escape_string()` in PHP
2. Use *parameterized queries*
3. Use an **ORM** for data access instead of direct queries
Despite the fact that mitigation of SQLi bugs is quite easy, these bugs are still **everywhere**. Most web apps perform dozens -- if not hundreds or thousands -- of queries. Manually handled, the odds of getting every single instance correct is slim. And **a single bug is enough to destroy/exfiltrate data, or even take control of the system.** 
### DETECTION
The most common SQLi is *in the conditions of a `SELECT`*, so the simplest way to detect these bugs is via these payloads:
- `' OR 1='1` -- This returns all rows (constant true)
- `' AND 0='1` -- This returns no rows (constant false)
### EXFILTRATION
When the goal is to get data out of the system, there are many different ways to do so. The simplest way is generally using `UNION`. Look at the following query:
	`SELECT foo, bar, baz FROM some_table WHERE foo='some input';`
	(Returns 3 columns of data)
Knowing there are 3 columns of return data, you could make a payload such to create a query like so:
	`SELECT foo, bar, baz FROM some_table WHERE foo='1'`
	`UNION SELECT 1, 2, 3; --';`
This will return an extra row containing the values `(1, 2, 3);`
We can use this technique to select data from other tables too, as long as we match column counts.
### EXAMPLE CASE
The result of entering "`' OR 1='1`" within the `Username` field demonstrates a vulnerability to SQL injection. Only the admin account can create or modify pages (clearly stated on the page). The admin account can be accessed by modification of both `Username` and `Password` by injecting a `UNION` call:
##### USERNAME FIELD:
	username=admin' UNION SELECT 'YourChosenPassword' AS password FROM admins WHERE '1' = '1
##### PASSWORD FIELD (as designated in the SQLi above):
	YourChosenPassword
### BLIND SQL INJECTION
Blind SQLi is when your data is inserted into a query but you can't directly see the results (for instance, a login page might contain blind SQLi, in that you only get back whether or not a login has succeeded).

There are two types of blind SQLi:
1. **Oracles** -- Where you get back a binary condition; the query succeeded/returned results or not.
2. **Truly blind** -- You see no difference whether the query failed or not.

**Truly blind SQL*i* is fairly rare, but **understanding how to exploit it is critical**. A good example is a facility that logs web requests to the database as you interact with it. In this case, you'll never see the results of the query whatsoever, nor will its failure impact the use of the application.

**Exploiting oracles** - Oracles allow you to answer a question with a true or false. Meaning that you can exfiltrate data one bit at a time. For example you could read the administrator password for a site, bit by bit, to reconstruct it. You'll generally write a script to do this, rather than attacking by hand.

**Truly Blind -> Oracle** -- We can  see at least one side-effect of the query: the amount of time it takes to execute. In the first SQL INJECTION example above, the `INSERT` call(s) would most likely be instantaneous in most cases; however, *if it's an MSSQL server, we can introduce a delay*:
**By introducing a conditional delay, we can make this blind SQLi into an oracle:** *10 second delay if 1, no delay if 0*.
Then it can be exploited identically to a standard oracle type, watching time instead of differences on a page.
### DETECTING DATABASE
SQL syntax and/or functionality differs dramatically across different database engines. For instance, *MSSQL* has `WAITFOR DELAY` as mentioned, but with *MySQL* you;d use the `BENCHMARK()` function to slow down the query and induce a delay.
**Being able to identify which DB the application is using is critical to easy exploitation.**
### ==TRICKS==
- `/*! comment here */` -- This looks like a normal comment to most DBs, but MySQL will include the contents of the comment inline, if it has an exclamation point at the beginning.
- `WAITFOR DELAY` will work on *MSSQL* and fail elsewhere.
- `ULT_INADDR.get_host_address('google.com')` will do a DNS request on Oracle.

### DETECTING MYSQL COLUMN/TABLE NAMES
To determine the length of the table name (# of characters), test various different numbers in place of "`X`". If the length is *correct*, the `SLEEP` command will execute, and the response time of the page will demonstrate this:
	`2’ AND LENGTH(DATABASE())=X AND SLEEP(5)#`

### CRACKING A USERNAME/PASSWORD WITH SQLMAP

##### Cracking the Username/Password with SQLMap
You can use the following command to find the credentials.
	`sqlmap -u "https://https://43617af3cd3c06f6b48172781fd62ae8.ctf.hacker101.com/login" --data "username=abc&password=xyz" -p username --dbms=mysql --dump`

The ==`-p`== option is for whichever parameter `sqlmap` should test. We know `username` is the vulnerable one from our previous SQLi tests with the field.
Note the format of the form input: ==`username=abc&password=xyz`== 
### WRAP-UP: SQLi, Command Injection, and Directory Traversal 
##### Directory traversal
- User input is used to build paths
- Leads to arbitrary file reads/writes in many cases
- The directory `..` is always the parent
##### Command injection
- User input gets injected into a command line that's being executed
- Often allows complete system compromise
- Backticks, semicolons, pipes, and ampersands are your friends here.
##### SQL Injection
- User data in put into SQL queries unescaped
- Can allow destruction and exfiltration of data
	- In rare cases, it can even allow filesystem access or code execution
- Most typical is that you see the results of your queries
- When you can't see them, you have blind SQLi
##### Blind SQLi
- You can get data out one bit at a time
- Often slow but always dangerous
# SESSION FIXATION & CLICKJACKING
