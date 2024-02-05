# Micro-CMS v1 Write-up (Hacker101 CTF)
---
> [!TIP]
> If you are new to Capture-The-Flag challenges as I was when I began this series of challenges, for this challenge it will help to check out the [XSS and Authorization](https://www.hacker101.com/sessions/xss) session from the Hacker101 course material.
---
## Flag0
I first explore a little bit to see what I have to work with. The **Create a New Page** link certainly seems like the place I might find a vulnerability. On the page that appears, there is a box for a **Title** and an area for a **Body** of text.
To see if either of the fields are vulnerable to XSS, I try out a simple method for identifying an XSS vulnerability by trying entering the following HTML into the fields and pressing **Create**:
```
<script>alert(1);</script>
```
Surprisingly, this is all that is required for this flag -- the flag appears within the alert box.

## Flag1
Now that I've gotten an attempt to check for an XSS vulnerability within a page input out of the way, I decide test the page URL itself for an XSS vulnerability from the `/page/edit/1` page. I accomplish this simply by adding a single quote to the end of the URL like below:
```
https://<MACHINE_ID>.ctf.hacker101.com/page/edit/1'
```
This leads directly to the second flag.

## Flag2
After experimenting by creating a couple of additional pages, I notice that after creating pages `1` and `2`, the subsequent page to be created is page `10` -- very suspect! I start to try opening each page from `page/edit/4` to `page/edit/9`, and eventually a *secret* is revealed within the **Body** field of the page -- and Flag2 is found!

## Flag3
I notice that one of the pre-existing pages has a `Some button` that doesn't respond to a click. After opening the page in the editor, I modify the button's code within the HTML to the following:
```
<button onclick=alert(1);>Some button</button>
```
The button now results in an alert box containing "1"; however, upon further analysis, I realize that the HTML for the button's code within my browser's inspector contains the value for the fourth and final flag!
