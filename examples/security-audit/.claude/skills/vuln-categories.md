---
name: vuln-categories
type: domain
version: v1.0
description: "8 vulnerability categories with scan patterns, CWE mappings, and false positive indicators"
---

# Vulnerability Categories

## Purpose

Define the 8 scanning categories for security audits. Each category includes grep/glob patterns, vulnerable code examples, false positive indicators, and CWE mappings.

## Categories

### 1. `injection` — Injection (OWASP A03:2021)

**What to scan for:** SQL/NoSQL/OS command/template injection, string concatenation in queries, unsanitized user input in exec/eval.

**Grep Patterns:**
```
# SQL injection
"SELECT.*\+.*req\."
"query\(.*\+.*\)"
"execute\(.*\+.*\)"
"\.raw\(.*\$\{"
"\.raw\(.*\+.*\)"
"sequelize\.query\(.*\+.*\)"

# NoSQL injection
"\$where.*req\."
"find\(\{.*req\.(body|query|params)"
"findOne\(\{.*req\."

# OS command injection
"exec\(.*req\."
"spawn\(.*req\."
"child_process.*req\."
"eval\(.*req\."
"Function\(.*req\."

# Template injection
"render\(.*req\.(body|query)"
"template.*\+.*req\."
```

**Glob Patterns:**
```
**/*.js
**/*.ts
**/*.py
**/*.rb
**/*.php
**/routes/**
**/controllers/**
**/handlers/**
```

**Vulnerable Code Examples:**
```javascript
// SQL injection via string concatenation
const result = await db.query("SELECT * FROM users WHERE id = " + req.params.id);

// NoSQL injection via unsanitized input
const user = await User.findOne({ username: req.body.username });

// OS command injection
const output = exec("ping " + req.query.host);
```

**False Positive Indicators:**
- Pattern found in test files (`**/*.test.js`, `**/*.spec.ts`)
- Pattern found in comments or documentation
- Pattern uses parameterized queries (`$1`, `?`, named params)
- Input is validated/sanitized before use

**CWE IDs:** CWE-89 (SQL Injection), CWE-943 (NoSQL Injection), CWE-78 (OS Command Injection), CWE-94 (Code Injection), CWE-1336 (Template Injection)

---

### 2. `auth` — Authentication & Session (OWASP A07:2021)

**What to scan for:** JWT without expiry, hardcoded secrets, insecure cookie flags, weak password requirements, session fixation.

**Grep Patterns:**
```
# JWT issues
"jwt\.sign\("
"expiresIn"
"JWT_SECRET.*=.*['\"]"
"jsonwebtoken"

# Hardcoded secrets
"secret.*=.*['\"][a-zA-Z0-9]"
"password.*=.*['\"][a-zA-Z0-9]"
"apiKey.*=.*['\"][a-zA-Z0-9]"
"API_KEY.*=.*['\"][a-zA-Z0-9]"

# Cookie flags
"cookie.*secure.*false"
"httpOnly.*false"
"sameSite.*none"

# Weak password
"minlength.*[1-5][^0-9]"
"password.*length.*<\s*[1-8][^0-9]"
```

**Glob Patterns:**
```
**/auth/**
**/middleware/**
**/config/**
**/*.env*
**/passport*
**/session*
```

**Vulnerable Code Examples:**
```javascript
// JWT without expiry
const token = jwt.sign({ userId: user.id }, "my-secret-key");

// Hardcoded JWT secret
const JWT_SECRET = "super-secret-key-123";

// Insecure cookie
res.cookie("session", token, { httpOnly: false, secure: false });
```

**False Positive Indicators:**
- Secret loaded from environment variable (`process.env.`)
- Token has `expiresIn` option set
- Found in test fixtures or mock data
- Cookie flags set correctly in production config

**CWE IDs:** CWE-287 (Improper Authentication), CWE-798 (Hardcoded Credentials), CWE-613 (Insufficient Session Expiration), CWE-614 (Sensitive Cookie Without Secure Flag), CWE-521 (Weak Password Requirements)

---

### 3. `xss` — Cross-Site Scripting (OWASP A03:2021)

**What to scan for:** Reflected/stored/DOM XSS, dangerouslySetInnerHTML, missing CSP, unescaped template output.

**Grep Patterns:**
```
# React XSS
"dangerouslySetInnerHTML"
"__html.*req\."
"innerHTML.*="

# Template XSS
"\{\{\{.*\}\}\}"          # Handlebars unescaped
"<%[-=].*req\."           # EJS unescaped
"v-html"                  # Vue unescaped
"\| safe"                 # Jinja2 unescaped

# DOM XSS
"document\.write\("
"\.innerHTML\s*="
"\.outerHTML\s*="
"eval\(.*location"

# Missing CSP
"helmet"
"Content-Security-Policy"
```

**Glob Patterns:**
```
**/*.jsx
**/*.tsx
**/*.vue
**/*.ejs
**/*.hbs
**/*.html
**/views/**
**/templates/**
**/components/**
```

**Vulnerable Code Examples:**
```javascript
// React dangerouslySetInnerHTML with user input
<div dangerouslySetInnerHTML={{ __html: userComment }} />

// Direct innerHTML assignment
document.getElementById("output").innerHTML = req.query.message;

// EJS unescaped output
<p><%- user.bio %></p>
```

**False Positive Indicators:**
- Content is sanitized before rendering (DOMPurify, sanitize-html)
- CSP headers configured via helmet or manually
- innerHTML set with static/trusted content
- Found in server-side rendering with auto-escaping

**CWE IDs:** CWE-79 (Cross-Site Scripting), CWE-80 (Basic XSS), CWE-116 (Improper Encoding)

---

### 4. `sensitive-data` — Sensitive Data Exposure (OWASP A02:2021)

**What to scan for:** Hardcoded API keys, PII in logs, HTTP instead of HTTPS, verbose error messages, .env in repo.

**Grep Patterns:**
```
# Hardcoded keys
"AKIA[0-9A-Z]{16}"          # AWS access key
"['\"]sk-[a-zA-Z0-9]{20,}"  # OpenAI/Stripe key pattern
"api[_-]?key.*=.*['\"][a-zA-Z0-9]"
"token.*=.*['\"][a-zA-Z0-9]{20,}"

# PII in logs
"console\.log.*password"
"console\.log.*token"
"console\.log.*email"
"logger\.(info|debug|error).*password"
"console\.log.*req\.body"

# HTTP
"http://(?!localhost|127\.0\.0)"

# Verbose errors
"stack.*trace"
"err\.stack"
"console\.log.*err"
"res\.(send|json)\(.*err"
```

**Glob Patterns:**
```
**/*.js
**/*.ts
**/*.env*
**/*.json
**/config/**
**/.env*
```

**Vulnerable Code Examples:**
```javascript
// Hardcoded API key
const STRIPE_KEY = "sk_live_abc123def456ghi789";

// PII in logs
console.log("Login attempt:", { email: user.email, password: req.body.password });

// Verbose error to client
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message, stack: err.stack });
});
```

**False Positive Indicators:**
- Keys are placeholder/example values ("YOUR_KEY_HERE", "xxx")
- Logging in test environment only
- Error details filtered before sending to client
- `.env.example` files with placeholder values

**CWE IDs:** CWE-200 (Information Exposure), CWE-532 (Information Exposure Through Log Files), CWE-798 (Hardcoded Credentials), CWE-209 (Information Exposure Through Error Messages)

---

### 5. `access-control` — Broken Access Control (OWASP A01:2021)

**What to scan for:** Missing auth middleware, IDOR patterns, CORS wildcard, directory traversal, privilege escalation.

**Grep Patterns:**
```
# Missing auth middleware
"app\.(get|post|put|delete|patch)\("
"router\.(get|post|put|delete|patch)\("

# IDOR patterns
"req\.params\.id"
"req\.params\.userId"
"findById\(req\.params"
"deleteOne\(\{.*_id.*req\.params"

# CORS
"origin.*['\"]\\*['\"]"
"cors\(\)"
"Access-Control-Allow-Origin.*\\*"

# Directory traversal
"path\.join\(.*req\."
"readFile\(.*req\."
"\.\./"
"req\.(query|params|body).*path"

# Privilege escalation
"role.*=.*req\.body"
"isAdmin.*=.*req\.body"
"\.update\(.*role"
```

**Glob Patterns:**
```
**/routes/**
**/controllers/**
**/middleware/**
**/api/**
**/server.*
**/app.*
```

**Vulnerable Code Examples:**
```javascript
// Missing auth on admin route
app.delete("/api/admin/users/:id", async (req, res) => {
  await User.deleteOne({ _id: req.params.id });
});

// IDOR — no ownership check
app.get("/api/orders/:id", auth, async (req, res) => {
  const order = await Order.findById(req.params.id);
  res.json(order);
});

// CORS wildcard
app.use(cors({ origin: "*" }));
```

**False Positive Indicators:**
- Route has auth middleware in the chain
- Ownership check performed after findById
- CORS restricted to specific origins in production
- Path input sanitized (path.normalize, no `..` allowed)

**CWE IDs:** CWE-862 (Missing Authorization), CWE-639 (Authorization Bypass Through User-Controlled Key), CWE-942 (Overly Permissive CORS), CWE-22 (Path Traversal), CWE-269 (Improper Privilege Management)

---

### 6. `misconfig` — Security Misconfiguration (OWASP A05:2021)

**What to scan for:** Debug mode in production, default credentials, missing security headers, Docker running as root, open ports.

**Grep Patterns:**
```
# Debug mode
"DEBUG.*=.*true"
"NODE_ENV.*development"
"app\.debug.*=.*True"

# Default credentials
"admin.*admin"
"root.*root"
"password.*password"
"default.*password"

# Missing security headers
"helmet"
"X-Frame-Options"
"X-Content-Type-Options"
"Strict-Transport-Security"

# Docker issues
"FROM.*:latest"
"USER root"
"--privileged"

# Open ports
"0\.0\.0\.0"
"EXPOSE"
"listen\(.*0\.0\.0\.0"
```

**Glob Patterns:**
```
**/Dockerfile*
**/docker-compose*
**/.env*
**/config/**
**/server.*
**/app.*
**/nginx*
```

**Vulnerable Code Examples:**
```javascript
// Debug mode in config
const config = { debug: true, verbose: true };

// Missing security headers (no helmet)
const app = express();
// No app.use(helmet()) anywhere

// CORS wildcard
app.use(cors({ origin: "*", credentials: true }));
```

**False Positive Indicators:**
- Debug mode in development config only (not production)
- Default credentials in test fixtures
- Security headers set via reverse proxy (nginx)
- Docker USER set to non-root after initial setup

**CWE IDs:** CWE-16 (Configuration), CWE-1188 (Insecure Default Initialization), CWE-250 (Execution with Unnecessary Privileges), CWE-693 (Protection Mechanism Failure)

---

### 7. `dependencies` — Vulnerable Dependencies (OWASP A06:2021)

**What to scan for:** Known CVE patterns, outdated packages, unpinned versions, unused dependencies.

**Grep Patterns:**
```
# Unpinned versions
"['\"]\\^"
"['\"]~"
"['\"]\\*['\"]"
"latest"

# Known vulnerable patterns (check package names)
"express.*['\"]3\."
"lodash.*['\"]4\.[0-9]\."
"moment"
"request['\"]"
"node-uuid"
```

**Glob Patterns:**
```
**/package.json
**/package-lock.json
**/yarn.lock
**/requirements.txt
**/Pipfile
**/go.mod
**/Cargo.toml
**/pom.xml
**/Gemfile
```

**Vulnerable Code Examples:**
```json
{
  "dependencies": {
    "express": "^3.0.0",
    "lodash": "*",
    "moment": "2.18.0"
  }
}
```

**False Positive Indicators:**
- Versions are pinned to latest stable
- Lock file present with exact versions
- Dependency used in devDependencies only
- Package has been patched (check lock file version)

**CWE IDs:** CWE-1104 (Use of Unmaintained Third-Party Components), CWE-829 (Inclusion of Functionality from Untrusted Control Sphere)

---

### 8. `crypto` — Cryptographic Failures (OWASP A02:2021)

**What to scan for:** MD5/SHA1 for security purposes, hardcoded encryption keys, weak random, custom crypto implementations.

**Grep Patterns:**
```
# Weak hashing
"createHash\(['\"]md5"
"createHash\(['\"]sha1"
"MD5\("
"hashlib\.md5"
"hashlib\.sha1"

# Weak random
"Math\.random"
"random\.random"
"rand\(\)"

# Hardcoded keys
"createCipher\("
"createCipheriv\(.*['\"][a-fA-F0-9]{32}"
"encryption[_-]?key.*=.*['\"]"
"ENCRYPTION_KEY.*=.*['\"]"

# Custom crypto
"xor.*encrypt"
"base64.*encrypt"
"btoa.*secret"
"atob.*secret"
```

**Glob Patterns:**
```
**/*.js
**/*.ts
**/*.py
**/*.rb
**/crypto/**
**/utils/hash*
**/utils/encrypt*
**/auth/**
```

**Vulnerable Code Examples:**
```javascript
// MD5 for password hashing
const hash = crypto.createHash("md5").update(password).digest("hex");

// Math.random for tokens
const resetToken = Math.random().toString(36).substring(2);

// Hardcoded encryption key
const key = "0123456789abcdef0123456789abcdef";
const cipher = crypto.createCipheriv("aes-256-cbc", key, iv);
```

**False Positive Indicators:**
- MD5/SHA1 used for non-security purposes (checksums, cache keys)
- Math.random used for non-security purposes (UI, shuffle)
- Encryption key loaded from environment variable
- Using established library (bcrypt, argon2, scrypt)

**CWE IDs:** CWE-327 (Use of a Broken Crypto Algorithm), CWE-330 (Use of Insufficiently Random Values), CWE-321 (Use of Hard-coded Cryptographic Key), CWE-916 (Use of Password Hash With Insufficient Computational Effort)

---

## Category Selection

Not all categories apply to every project. During reconnaissance:

| Project Type | Skip | Focus |
|-------------|------|-------|
| CLI tool | xss | injection, sensitive-data, crypto |
| Static site | injection, auth, access-control | xss, misconfig, dependencies |
| REST API | xss (if no templates) | injection, auth, access-control, sensitive-data |
| Full-stack web app | None | All 8 categories |
| Library/SDK | misconfig | injection, crypto, dependencies |
