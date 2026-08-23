# WebGoat Reflected XSS — Shopping Cart

This project was a Reflected XSS assessment using OWASP WebGoat's shopping cart module. I used Burp Suite to intercept the requests, test the different input fields, and find which one was reflecting user input without proper encoding.

After finding the vulnerable field, I tested JavaScript execution and then created a payload that modified the shopping cart directly in the browser by changing the quantities and prices.

This was completed in an isolated lab environment using Kali Linux and OWASP WebGoat. No production systems were involved.

## Objective

The goal was to determine whether any of the shopping cart fields were vulnerable to reflected XSS.

Instead of only showing a basic `alert()` as proof, I wanted to show the actual impact by using JavaScript to modify the shopping cart in the browser.

## Methodology

### 1. Capture the request

I first used Burp Suite to capture the normal shopping cart requests so I could see exactly what parameters were being sent.

The request contained:

```text
QTY1=1&QTY2=1&QTY3=1&QTY4=1&field2=4128+3214+0002+1999&field1=111&SUBMIT=Purchase
```

There were seven parameters:

* `QTY1`–`QTY4` — item quantities
* `field2` — credit card number
* `field1` — access code
* `SUBMIT` — purchase action

### 2. Test each field

Instead of guessing which parameter was vulnerable, I tested each field individually with:

```html
'><script>alert('XSS')</script>
```

The goal was to see if the application would treat the input as normal text or actually interpret it as HTML/JavaScript.

The results were:

* `QTY1`–`QTY4`: reflected as text and did not execute
* `field2`: the input was modified by the field's formatting before being reflected
* `field1`: the payload executed successfully

The WebGoat success message appeared when testing `field1`, confirming that JavaScript was being executed.

## Key Finding

The vulnerable parameter was `field1`, the access code field.

The application reflected the value directly into the response without properly encoding it. Because characters such as `<`, `>` and `"` were not encoded, the browser interpreted the injected content as actual HTML and JavaScript.

I confirmed this with another payload:

```html
"><svg/onload=alert(document.cookie)>
```

This successfully executed and displayed the session cookie in the lab.

This confirmed that the injected JavaScript was running in the same context as the WebGoat application and could interact with the page's DOM and JavaScript-accessible data.

## Exploitation

After confirming the XSS, I wanted to demonstrate more than just an `alert()`.

I created a JavaScript payload that changed the shopping cart directly in the browser. The script:

1. Finds all of the quantity inputs.
2. Changes the quantities to `999`.
3. Finds the price and total cells.
4. Changes the displayed prices to `$0.00`.

The main part of the payload was:

```html
"><script>window.onload=function(){
  var inputs = document.getElementsByTagName('input');

  for (var a = 0; a < inputs.length; a++) {
    if (inputs[a].name && inputs[a].name.indexOf('QTY') === 0) {
      inputs[a].value = 999;
    }
  }

  var cells = document.getElementsByTagName('td');

  for (var b = 0; b < cells.length; b++) {
    var text = cells[b].innerHTML;

    if (text.indexOf('$') === 0) {
      cells[b].innerHTML = '$0.00';
    } else if (/^\d+\.\d\d$/.test(text)) {
      cells[b].innerHTML = '0.00';
    }
  }
}</script>
```

I used two separate loops because the quantity fields and table cells were easier to identify in different ways.

For the quantity fields, I could look for names starting with `QTY`.

For the price cells, I checked the text being displayed and looked for values matching the expected price format.

I also used `window.onload` so the script would wait until the page finished loading before trying to modify the shopping cart.

## Result

The quantities were changed to `999`, the displayed prices were changed to `$0.00`, and the displayed total was reduced to `$0.00`.

The changes were made entirely in the browser. This demonstrates that the injected JavaScript had access to and could modify the page's DOM.

## Why This Is Reflected XSS

Reflected XSS happens when an application takes user input and puts it back into the response without properly encoding it.

In this case:

1. I submitted the `field1` value to the application.
2. The server reflected that value back into the page.
3. Special characters such as `<` and `>` were not encoded.
4. The browser interpreted the injected HTML as part of the page.
5. The JavaScript executed in the context of the WebGoat application.

It is called reflected XSS because the malicious input isn't permanently stored in the application. It is reflected back as part of the response to the request.

## Impact

The lab showed that an attacker-controlled script could run inside the application and modify the page.

In a real application, the impact could include:

* Session theft if session cookies are accessible to JavaScript.
* Credential or payment information theft through fake forms or modified page content.
* DOM manipulation, such as changing prices, account information, or other displayed data.
* Phishing, since malicious content would appear inside the legitimate website.
* Business logic manipulation if the server trusts values that should be recalculated and validated server-side.

The cart manipulation in this lab was only a client-side demonstration. Changing what the browser displays does not automatically change what the server accepts. A properly designed application should always validate important values server-side.

## Remediation

### 1. Output encoding

The main fix is to properly encode user input before putting it into the response.

For example, HTML special characters such as `<`, `>` and `"` should be encoded so the browser treats them as data instead of HTML.

### 2. Content Security Policy

A strong Content Security Policy (CSP) can provide another layer of protection by restricting where JavaScript can run and blocking things like inline scripts.

### 3. HttpOnly cookies

Session cookies should be marked as HttpOnly so JavaScript cannot access them through `document.cookie`.

This would reduce the impact of XSS if another XSS vulnerability was found.

### 4. Server-side input validation

Fields that have a specific format should also be validated on the server.

For example, the access code should only accept the expected number and format of characters. This doesn't replace output encoding, but it adds another layer of protection.

## Skills & Tools

* Burp Suite — intercepted requests and tested parameters
* Kali Linux — used as the attacker environment
* OWASP WebGoat — intentionally vulnerable application used for the lab
* HTTP analysis — examined request parameters and server responses
* JavaScript / DOM manipulation — used to demonstrate the impact of the XSS
* Browser DevTools — used to confirm JavaScript execution and DOM changes
* XSS analysis — identified the vulnerable parameter, confirmed execution, and explained the remediation

## What I Took From This Project

The main thing I took from this lab was understanding the difference between simply finding XSS and actually showing its impact.

Using Burp Suite, I was able to test each parameter instead of just guessing which one was vulnerable. Once I found that `field1` was reflecting input without encoding, I could confirm JavaScript execution and then use the vulnerability to actually manipulate the page.

It also showed why output encoding is so important. The application wasn't necessarily doing anything complicated with the input — it was simply putting it back into the page without making sure the browser treated it as data.

---

*Performed for CCCS-455-784 (Intrusion Testing & Security Assessment), McGill University School of Continuing Studies, Summer 2026, as a group assignment. Completed in an isolated WebGoat lab environment.*
