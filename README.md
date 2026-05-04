# Report - Juice Shop Vulnerabilities and Tracking

## Project

Target: OWASP Juice Shop  
URL: `http://localhost:3000`  
Tester: Max  
Purpose: Find, test, exploit, and record Juice Shop vulnerabilities for training.

## How To Use This File

Use this file as your working report. For every step, paste screenshots, copied terminal output, Burp requests, Burp responses, browser notes, and short explanations of what you found.

Recommended evidence format:

```md
### Evidence

Screenshot:
Paste image here.

Request:
Paste Burp/curl request here.

Response:
Paste response here.

Notes:
Write what happened and why it matters.
```

## Setup Checklist

- [ ] Docker installed
- [ ] Juice Shop image pulled
- [ ] Juice Shop container running
- [ ] Browser opened at `http://localhost:3000`
- [ ] Burp Suite installed
- [ ] Browser proxy configured for Burp
- [ ] Burp certificate installed in test browser, if HTTPS testing is needed later
- [ ] Screenshots folder created for evidence

## Testing Steps

### 1. Start Juice Shop

Run:

```bash
docker pull bkimminich/juice-shop
docker run -d -p 3000:3000 bkimminich/juice-shop
```

Open:

```text
http://localhost:3000
```

Evidence:

- Screenshot of Juice Shop running:
- Docker command/output:
- Notes:

### 2. Nmap

Use Nmap to confirm what is exposed locally.

Example:

```bash
nmap -sV -p 3000 localhost
```

Record:

- Open ports:
- Service/version details:
- Interesting findings:
- Screenshot or terminal output:

### 3. Browser Walkthrough

Use the site normally before attacking it.

Check:

- Login page
- Register page
- Products
- Search
- Basket/cart
- Checkout flow
- Contact/support forms
- Error messages

Record:

- Pages visited:
- Inputs/forms found:
- Visible errors:
- Interesting URLs:
- Screenshots:

### 4. Browser DevTools

Open DevTools with `F12`.

Check:

- Network tab
- Console tab
- Application tab
- Sources tab

Record:

- API endpoints seen:
- Cookies:
- localStorage/sessionStorage values:
- JavaScript files with useful names:
- Console errors:
- Screenshots:

### 5. Burp Suite - Proxy Intercept On

Start Burp Suite and enable proxy interception.

Browser proxy:

```text
HTTP proxy: 127.0.0.1
Port: 8080
```

In Burp:

- Proxy -> Intercept -> Intercept is on
- Visit Juice Shop in the browser
- Send interesting requests to Repeater

Record:

- Screenshot of Burp intercepting traffic:
- Important requests:
- Important responses:
- Notes:

### 6. Burp Suite - Site Map

Browse the full application while Burp records traffic.

In Burp:

- Target -> Site map
- Review discovered paths
- Mark interesting endpoints

Record:

- Discovered endpoints:
- API routes:
- Hidden or unusual paths:
- Screenshots:

### 7. Authentication Testing

Test login and registration behavior.

Try:

- Normal registration
- Normal login
- Wrong password
- Unknown email
- SQL injection style input
- Password reset flow

Record:

- Test account used:
- Payloads tested:
- Result:
- Burp request/response:
- Screenshots:

### 8. Input Validation Testing

Test forms and parameters.

Look at:

- Search box
- Login form
- Register form
- Contact form
- Product reviews
- Basket quantities

Record:

- Input tested:
- Payload:
- Result:
- Vulnerability suspected:
- Evidence:

### 9. SQL Injection Testing

Test login, search, and API parameters carefully.

Example payloads to record if used:

```text
'
' OR '1'='1
admin@juice-sh.op'--
```

Record:

- Target field/endpoint:
- Payload:
- Result:
- Request:
- Response:
- Screenshot:
- Explanation:

### 10. Cross-Site Scripting Testing

Test reflected or stored input locations.

Example payloads to record if used:

```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
```

Record:

- Target field:
- Payload:
- Did JavaScript execute:
- Screenshot:
- Request/response:
- Explanation:

### 11. Access Control Testing

Check whether normal users can access admin or other users' data.

Look for:

- User IDs in URLs or JSON
- Admin endpoints
- Basket IDs
- Order IDs
- Profile data

Record:

- Endpoint:
- User account used:
- What was accessed:
- Why it is unauthorized:
- Evidence:

### 12. Sensitive Data Exposure

Look for secrets or leaked information.

Check:

- JavaScript source files
- API responses
- Error messages
- Cookies/storage
- Public files

Record:

- Location:
- Data found:
- Risk:
- Evidence:

### 13. Directory and Endpoint Discovery

Use tools like Gobuster or ffuf if available.

Example:

```bash
gobuster dir -u http://localhost:3000 -w /usr/share/wordlists/dirb/common.txt
```

Record:

- Tool used:
- Wordlist:
- Interesting paths:
- Output:
- Screenshots:

### 14. Confirm Each Vulnerability

Before writing a final finding, confirm it works more than once.

For each confirmed issue, record:

- Vulnerability name:
- OWASP category:
- Affected endpoint/page:
- Steps to reproduce:
- Impact:
- Evidence:
- Possible fix:

### 15. Write Final Findings

Use the template below for every vulnerability.

## Vulnerability Findings

### Finding 1 - Title

Status: Draft / Confirmed  
Severity: Low / Medium / High / Critical  
OWASP Category:  
Affected URL or endpoint:

#### Summary

Write a short explanation of the vulnerability.

#### Steps To Reproduce

1. 
2. 
3. 

#### Evidence

Screenshot:

Request:

```http

```

Response:

```http

```

Notes:

#### Impact

Explain what an attacker could do.

#### Recommendation

Explain how this should be fixed.

### Finding 2 - Title

Status: Draft / Confirmed  
Severity: Low / Medium / High / Critical  
OWASP Category:  
Affected URL or endpoint:

#### Summary

#### Steps To Reproduce

1. 
2. 
3. 

#### Evidence

Screenshot:

Request:

```http

```

Response:

```http

```

Notes:

#### Impact

#### Recommendation

## Evidence Log

Use this section to track screenshots, pasted images, terminal output, and Burp evidence.

| Evidence ID | Date | Step/Finding | Type | Description | File or Paste Location |
| --- | --- | --- | --- | --- | --- |
| EVID-001 |  |  | Screenshot |  |  |
| EVID-002 |  |  | Burp request/response |  |  |
| EVID-003 |  |  | Terminal output |  |  |

## Notes

Use this section for rough notes while testing.

- 

