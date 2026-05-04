# MISSION 00 — JUICE SHOP
### BeCode Corp | Internal Training Exercise
---

> No theory will be handed to you.
> Figure it out. That is the point.

---

## CONTEXT

Before we touch any client environment, every analyst at BeCode Corp goes
through the same first exercise: OWASP Juice Shop.

Juice Shop is a deliberately vulnerable web application built by OWASP —
the Open Web Application Security Project. It simulates a realistic
e-commerce platform riddled with intentional security flaws covering the
OWASP Top 10 and beyond.

Your job this week: find as many vulnerabilities as you can, exploit them,
understand why they work, and write it up.

No walkthrough. No step-by-step guide. You have the application, the
internet, and your brain.

---

## SETUP

### 1. Install Docker (if not already installed)

Juice Shop runs best via Docker. If you don't have Docker:

```
https://docs.docker.com/get-docker/
```

Verify installation:
```bash
docker --version
```

### 2. Pull and run Juice Shop

```bash
docker pull bkimminich/juice-shop
docker run -d -p 3000:3000 bkimminich/juice-shop
```

Open your browser and go to:
```
http://localhost:3000
```

You should see the Juice Shop storefront. If you do, you're ready.

> Note: port 3000 must be free on your machine.
> If it is in use, remap it: `-p 8080:3000` and open http://localhost:8080

### 3. Stop / restart the container

```bash
# List running containers
docker ps

# Stop Juice Shop
docker stop <container_id>

# Restart it
docker start <container_id>
```

---

## TOOLS

You are not limited to these. But these are your starting point.

### Burp Suite Community Edition

Your main weapon for web application testing. Burp Suite sits between your
browser and the application, letting you intercept, inspect, and modify
every HTTP request and response.

**Download:**
```
https://portswigger.net/burp/communitydownload
```

### Firefox / Chromium

Use a dedicated browser profile for testing. Keep your personal browser
separate. This avoids cookie contamination and keeps your traffic clean.

### Browser DevTools (F12)

Before reaching for any external tool, open DevTools:
- **Network tab** — watch every request the app makes
- **Console tab** — JavaScript errors and logs, often reveal internal logic
- **Application tab** — cookies, localStorage, sessionStorage
- **Sources tab** — JavaScript source files, sometimes with comments

### curl

For quick manual request crafting from the terminal:
```bash
curl -v http://localhost:3000/api/Users
curl -X POST http://localhost:3000/rest/user/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"test"}'
```

### Gobuster / ffuf (optional)

For directory and endpoint discovery:
```bash
gobuster dir -u http://localhost:3000 -w /usr/share/wordlists/dirb/common.txt
```

---

We will begin the Lab training on next Monday!

---

*BeCode Corp | Internal Training Exercise | BCC-2026*
*Classification: Internal — Not for distribution*
