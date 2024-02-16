# Petshop Pro Write-up (Hacker101 CTF)

> [!IMPORTANT]
> If these concepts are new to you, I would highly recommend watching the _Hacker101_ **Session Videos**[^1] or check out their **Getting Started**[^2] page.

---

#### TASK LIST
- [x] **Locate and submit `Flag0`**
- [ ] **Locate and submit `Flag1`**
- [ ] **Locate and submit `Flag2`**

#### HINTS
| Flag | Hint |
| --- | :--- |
| `Flag0` | Something looks out of place with the checkout.<br>It's always nice to get free stuff. |
| `Flag1` | There must be a way to administer the app.<br>Tools may help you find the entrypoint.<br>Tools are also great for finding credentials. |
| `Flag2` | Always test every input.<br>Bugs don't always appear in a place where data is entered. |

---
# `Flag0`
After spending a short time navigating the shop and looking at how the shop is coded, it became very clear that the way the shopping cart functions is very flawed and contains a *Reflected XSS vulnerability*. The vulnerability affects all values assigned to a particular product (most notably the _price_) when the order information is relayed upon clicking *Check Out* on the *Shopping Cart* page. Product information is actually implemented on the page as a *hidden form* which can be exploited by changing the price of an item and then checking out.

##### Proof of Concept
Any product values (name, description, price, etc.) on the site can be modified because products were implemented to exist within a "hidden" form in the HTML code. When the information is submitted using the `Check Out` button, a new set of values is submitted (the actual product information winds up simply acting as the default in this case). In the picture below, the `price` value was changed from `8.95` to `0.00`:

<img width="662" alt="Screenshot 2024-01-29 at 10 40 54 AM" src="https://github.com/Deyl64/CTF-Writeups/assets/33075179/1944a83e-69d1-404c-bedb-4c9e1b0d7f81">

The result upon clicking the **`Check Out`** button is as shown below:
<img width="662" src="https://github.com/Deyl64/CTF-Writeups/assets/33075179/a8630a1e-d551-415c-b723-a2e4ea6379fa">

The variable "`cart`" appears to be a concatenation of all parameters belonging to each product. It can be seen in the HTML for the page within a hidden form.

---
# Flag1 -- ==Not Found==
### Admin Login
I found the *Admin Login* `/login` page simply by guessing the URL for the page (although you could do this with a web fuzzer as well). I automated attempts at guessing the username by using a _web fuzzer_ called `ffuf` and filtering the results containing "Invalid username". After an evening of being repeatedly infuriated by getting results that do not function as they should within the actual web form -- I realize that I made an error that had been causing all of my troubles. When I added the `Content-Type` shown within the `POST` request for the login form, I did not see correctly. For an entire evening, I had been erroneously using "**`application/www-form-urlencoded`**" instead of "**`application/x-www-form-urlencoded`**". A lesson painfully learned - **review every option and parameter for your commands individually if it is not operating as it should**!

Here is the **properly composed** command I used in the terminal:
  `ffuf -w $usernames -u https://<MACHINE_ID>.ctf.hacker101.com/login -d "username=FUZZ&&password=x"  -X POST -H "Content-Type: application/x-www-form-urlencoded" -fr "Invalid username"`
Which resulted in the following output:
  ```BASH
  :: Method           : POST
   :: URL              : https://<MACHINE_ID>.ctf.hacker101.com/login
   :: Wordlist         : FUZZ: /Users/deyl64/Repositories/SecLists/Usernames/Names/names.txt
   :: Header           : Content-Type: application/x-www-form-urlencoded
   :: Data             : username=FUZZ&&password=x
   :: Follow redirects : false
   :: Calibration      : false
   :: Timeout          : 10
   :: Threads          : 40
   :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
   :: Filter           : Regexp: Invalid username
  ________________________________________________
  
  enrichetta              [Status: 200, Size: 378, Words: 19, Lines: 15, Duration: 59ms]
  :: Progress: [10177/10177] :: Job [1/1] :: 533 req/sec :: Duration: [0:00:27] :: Errors: 0 ::
  ```
It is now clear that I have the username I need -- **`enrichetta`**. Next, I prepare to brute force the password using **`hydra`** with the following command:
  `hydra -l enrichetta -P ~/Repositories/SecLists/Passwords/Leaked-Databases/rockyou.txt -o cracked-credentials.txt -F -M 54.148.133.24:443 https-post-form "/login:username=^USER^&password=^PASS^:Invalid password"`

> [!NOTE]
> The best collection of word lists I have found is the GitHub repository [**danielmiessler/SecLists**](https://github.com/danielmiessler/SecLists) (_danielmiessler_ has word lists and other resources that are worth checking out in his other repositories as well). 

For some reason my `hydra` command does not produce the credentials, which has me puzzled. I try modifying the command several times, to no avail. I tried using several different word lists which also did not lead to the goal. Occasionally, a username and password would be printed to `stdout` (because the response did not contain "_Invalid password_"); however, I realized shortly thereafter that they did not work and resulted only because the request led to some other erroneous result such as a time-out or the like. Time for a new strategy...
Next, I explore something that I overlooked because I mistakenly thought that _Burp Suite - Community Edition_ would not allow the use of extensions (thanks to a Google query about password cracking). This leads me to install a _Burp Suite Extension/Add-On_ called _**Turbo Intruder**_. *Turbo Intruder* is a Burp Suite extension that is definitely worth grabbing. You can install it directly through the _Extensions_ tab within Burp Suite, but can alternatively be found within _PortSwigger's_ GitHub repository [PortSwigger/turbo-intruder](https://github.com/PortSwigger/turbo-intruder). I found a detailed article on how to use Turbo Intruder called [*Turbo Intruder – hacking at light speed*](https://security.packt.com/turbo-intruder/) which I recommend for anyone who is new to the tool like me.

I tried using some different word lists within the SecLists repository (`rockyou.txt`, `rockyou-65.txt`, `10-million-password-list-top-100000.txt`, and `top-passwords-shortlist.txt`), and then I cloned another repository from _SecLists_ and tried using `10k-most-common.txt`. Despite this, I am still working on this.

  

---
# 

---
[^1]: [Videos - Hacker101](https://www.hacker101.com/videos)
[^2]: [Getting Started - Hacker101](https://www.hacker101.com/start-here)


---
