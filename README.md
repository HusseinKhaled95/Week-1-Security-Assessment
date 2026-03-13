# 🔐 Cybersecurity Internship — Week 1: Security Assessment

**DevelopersHub Cybersecurity Internship Program**  
**Task:** Strengthen Security Measures for a Web Application  
**Week:** 1 — Security Assessment  
**Application Tested:** [OWASP NodeGoat v1.3.0](https://github.com/OWASP/NodeGoat)

---

## 📋 Overview

This repository documents the **Week 1 Security Assessment** of a vulnerable Node.js web application (OWASP NodeGoat). The goal was to:

1. Set up a vulnerable web application
2. Perform a vulnerability assessment using manual testing and static code analysis
3. Document all findings with severity ratings, evidence, and remediation steps

---

## 🏗️ Application Setup

```bash
# Clone the vulnerable application
git clone https://github.com/OWASP/NodeGoat.git
cd NodeGoat

# Install dependencies
npm install

# Start the application
npm start
# App runs at http://localhost:4000
```

**Tech Stack:**
- Runtime: Node.js / Express.js
- Database: MongoDB
- Templating: Swig
- Authentication: Session-based (express-session)

---

## 🧪 Testing Methodology

### Tools Used

| Tool | Purpose |
|------|---------|
| **OWASP ZAP** | Automated web vulnerability scanner |
| **Browser DevTools** | Inspect headers, cookies, DOM |
| **Manual Code Review** | Static analysis of all route/DAO files |
| **npm audit** | Dependency CVE scanning |

### Manual Test Payloads

**XSS Test:**
```html
<script>alert('XSS');</script>
```
→ Entered in memo text field — payload stored and executed on page load ✅ VULNERABLE

**NoSQL Injection Test:**
```
Username: admin' OR '1'='1
Password: anything
```
→ MongoDB operator injection tested against login form

**SSRF Test:**
```
GET /research?url=http://169.254.169.254/latest/meta-data/&symbol=iam
```
→ Server makes outbound request to attacker-controlled URL ✅ VULNERABLE

---

## 🚨 Vulnerabilities Found

### Summary

| ID | Vulnerability | Severity | OWASP Category |
|----|--------------|----------|----------------|
| V-01 | Plaintext Password Storage | 🔴 Critical | A07: Auth Failures |
| V-02 | Server-Side Request Forgery (SSRF) | 🔴 Critical | A10: SSRF |
| V-03 | Stored XSS via Memo Input | 🟠 High | A03: Injection |
| V-04 | Missing Security Headers (Helmet disabled) | 🟠 High | A05: Misconfiguration |
| V-05 | CSRF Protection Disabled | 🟠 High | A01: Broken Access |
| V-06 | Session Fixation on Login | 🟠 High | A07: Auth Failures |
| V-07 | Sensitive PII Stored Unencrypted | 🟠 High | A02: Crypto Failures |
| V-08 | ReDoS via Regex Backtracking | 🟡 Medium | A06: Vulnerable Components |
| V-09 | Log/CRLF Injection | 🟡 Medium | A09: Logging Failures |
| V-10 | Hardcoded Secrets in Config | 🟡 Medium | A05: Misconfiguration |

**npm audit:** 141 known CVEs in dependencies (36 Critical, 63 High, 33 Moderate, 9 Low)

---

### V-01 — Plaintext Password Storage 🔴 Critical

**File:** `app/data/user-dao.js`

**Evidence:**
```javascript
// VULNERABLE — password stored as plaintext
const user = {
    userName,
    password  // received directly from req.body
};

// Fix EXISTS but is commented out:
// password: bcrypt.hashSync(password, bcrypt.genSaltSync())
```

**Impact:** Complete credential exposure if database is breached.

**Fix:** Uncomment `bcrypt.hashSync()` in `addUser()` and replace string comparison with `bcrypt.compareSync()` in `validateLogin()`.

---

### V-02 — Server-Side Request Forgery (SSRF) 🔴 Critical

**File:** `app/routes/research.js`

**Evidence:**
```javascript
// VULNERABLE — user-controlled URL used directly
const url = req.query.url + req.query.symbol;
needle.get(url, (error, newResponse, body) => { ... });
```

**Impact:** Attacker can probe internal services, read cloud metadata, pivot to internal network.

**Fix:** Implement strict URL allowlist; block private IP ranges; validate URL scheme.

---

### V-03 — Stored XSS via Memo Input 🟠 High

**File:** `app/routes/memos.js`

**Evidence:**
```javascript
// VULNERABLE — no sanitization before insert
memosDAO.insert(req.body.memo, (err, docs) => { ... });
```

Payload `<script>alert('XSS')</script>` stored and executed for all users.

**Fix:** Sanitize with `sanitize-html` before storage; enable CSP via Helmet.

---

### V-04 — Missing Security Headers 🟠 High

**File:** `server.js`

**Evidence:**
```javascript
// const helmet = require("helmet");  ← COMMENTED OUT
// app.use(helmet());                 ← COMMENTED OUT
```

Response headers missing: `X-Frame-Options`, `Content-Security-Policy`, `X-Content-Type-Options`, `Strict-Transport-Security`.
`X-Powered-By: Express` header exposed.

**Fix:** Uncomment Helmet.js configuration.

---

### V-05 — CSRF Protection Disabled 🟠 High

**Evidence in `server.js`:**
```javascript
// const csrf = require('csurf');  ← COMMENTED OUT
```

All POST endpoints (profile update, memo creation) are vulnerable to CSRF attacks.

---

### V-06 — Session Fixation 🟠 High

**File:** `app/routes/session.js`

**Evidence:**
```javascript
// Session NOT regenerated — fixation vulnerability
req.session.userId = user._id;
// Fix: wrap in req.session.regenerate(() => { ... })
```

---

### V-07 — Unencrypted PII Storage 🟠 High

**File:** `app/data/profile-dao.js`

SSN, date of birth, bank account numbers stored as plaintext in MongoDB. Encryption code exists but is commented out under `/* Fix for A6 - Sensitive Data Exposure */`.

---

### V-08 — ReDoS via Regex 🟡 Medium

**File:** `app/routes/profile.js`

```javascript
// VULNERABLE — exponential backtracking
const regexPattern = /([0-9]+)+\#/;

// SAFE alternative (commented out):
// const regexPattern = /([0-9]+)\#/;
```

---

### V-09 — Log/CRLF Injection 🟡 Medium

**File:** `app/routes/session.js`

```javascript
// Raw user input in log — CRLF injection possible
console.log("Error: attempt to login with invalid user: ", userName);
```

---

### V-10 — Hardcoded Secrets 🟡 Medium

**File:** `config/env/all.js`

```javascript
cookieSecret: "session_cookie_secret_key_here",  // HARDCODED
cryptoKey: "a_secure_key_for_crypto_here",        // HARDCODED
```

---

## 📁 Repository Structure

```
├── README.md                          ← This file
├── Week1_Security_Assessment_Report.docx  ← Full report document
└── screenshots/                       ← Evidence screenshots
    ├── xss-payload-memo.png
    ├── missing-headers-curl.png
    └── ssrf-test.png
```

---

## 📹 Video Walkthrough

> **[Link to recorded screen demo]** — covers:
> - Application setup and login walkthrough  
> - Live XSS demo in memo field  
> - Browser DevTools header inspection  
> - Code walkthrough of critical vulnerabilities  
> - Summary of findings and Week 2 plan

---

## 🗓️ Week 2 Plan

Based on Week 1 findings, the following fixes will be implemented in Week 2:

- [ ] Enable bcrypt password hashing (`user-dao.js`)
- [ ] Enable Helmet.js security headers (`server.js`)
- [ ] Sanitize memo inputs with `sanitize-html`
- [ ] Enable CSRF protection via `csurf`
- [ ] Regenerate session on login
- [ ] Add JWT-based token authentication
- [ ] Move secrets to environment variables (`.env`)

---

## 📚 References

- [OWASP Top 10 (2021)](https://owasp.org/www-project-top-ten/)
- [OWASP ZAP](https://www.zaproxy.org/)
- [Helmet.js Docs](https://helmetjs.github.io/)
- [OWASP XSS Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP SSRF Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html)

---

*Submitted as part of DevelopersHub Cybersecurity Internship — Deadline: March 23, 2026*
