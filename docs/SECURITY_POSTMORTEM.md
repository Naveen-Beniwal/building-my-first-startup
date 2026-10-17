# Production Security Postmortem & Hardening Report

A transparent technical postmortem detailing two real production security incidents encountered during CVInsight's operation, the root causes, immediate mitigations, and architectural lessons learned.

---

## Executive Summary

Operating a live web application used by thousands of students exposes software to real-world threats that never surface in local development. 

Between May and July 2026, CVInsight encountered two distinct operational security challenges:
1. **May 2026:** An automated bot registration burst that inflated accounts from 800 to 2,100+, rapidly consuming API token reserves.
2. **July 2026:** An authentication and routing edge-case vulnerability involving institutional/academic email domain normalization.

Both incidents were contained without data breaches, and the remediations led to a significantly more resilient platform.

---

## Incident 01: The Bot Registration Burst & Token Scraping Attack (May 2026)

### Timeline & Impact
* **T+00:00 (Saturday, 02:15 AM):** Server telemetry indicated an abnormal spike in MongoDB connection counts and outbound API calls to LLM providers.
* **T+01:30:** Registered accounts surged from ~800 to over 2,100 within a few hours. Token quotas were being consumed at 8x normal weekend velocity.
* **T+02:00:** Legitimate users attempting to audit resumes began receiving `HTTP 429 Too Many Requests` due to upstream provider rate limits.
* **T+03:15:** The maintainer intervened, temporarily locking the public registration endpoint to halt the attack.

### Root Cause Analysis (RCA)
1. **Unprotected Public Endpoints:** The user registration and free resume audit endpoints lacked automated bot detection, relying solely on client-side form validation.
2. **Absence of Velocity Checks:** Automated headless browser scripts submitted registration payloads in parallel without human-like interaction delays.
3. **Disposable Email Domains:** The registration pipeline accepted arbitrary email domains without checking against known disposable email providers.

### Technical Remediation
```
+-------------------------------------------------------------+
| Incoming Request Payload                                    |
+-------------------------------------------------------------+
                              |
                              v
    [Honeypot Check] ---- Populated? ----> [Silent 400 Drop]
                              |
                            Clean
                              v
    [Velocity Check] ---- Delta < 1.2s? -> [Reject as Bot]
                              |
                            Human
                              v
    [Rate Limiter] ------ Exceeded? -----> [HTTP 429 Backoff]
                              |
                            Valid
                              v
    [Domain Verification] Disposable? ---> [Reject Email]
                              |
                            Clean
                              v
             Proceed to Database & Application Logic
```

1. **Honeypot Input Injection:** Added hidden, dynamically rendered form fields with CSS obfuscation rules and negative tab indices. Legitimate users never see or interact with these fields; automated headless form-fillers populate them automatically, triggering silent rejection.
2. **Submission Velocity Analysis:** Form initialization timestamps are cryptographically signed into ephemeral submission tokens. Submissions completed with superhuman velocity (sub-second bursts) are flagged and dropped.
3. **Sliding-Window IP Rate Limiting:** Enforced dynamic per-IP rate limiting on authentication and AI generation routes (`express-rate-limit` backed by in-memory stores) to suppress brute-force and credential stuffing attempts.
4. **Database Sanitization:** Wrote custom migration scripts to detect and safely purge **1,300+ scripted ghost accounts**, reclaiming database index memory and ensuring analytics integrity.

---

## Incident 02: Institutional Email Routing & Verification Logic Flaw (July 2026)

### Context
To support university campus placement cells and verified college cohorts (e.g., NIT Kurukshetra, IITs, IIITs), CVInsight provides institutional email verification allowing automatic enrollment into university T&P cohorts.

### The Vulnerability
During internal testing of multi-tenant academic routing, a logic flaw was discovered in the regex matching and normalization of institutional domains:

```typescript
// VULNERABLE IMPLEMENTATION (Pre-Patch)
const isAcademic = email.endsWith('.ac.in') || email.endsWith('.edu');
```

1. **Subdomain Manipulation:** An email address such as `user@attacker-domain.edu.attacker.com` would pass naïve suffix checks if string splitting and boundary matching were not strictly anchored to the host domain.
2. **Privilege Confusion:** The system evaluated student versus recruiter dashboard routes using domain flags before secondary token validation was finalized, potentially allowing access to empty cohort state views.

### Immediate Containment
* Maintenance mode was invoked for 90 minutes during an off-peak window.
* All active JWT session tokens were revoked, forcing re-authentication.
* The routing logic was audited and rewritten from scratch.

### The Patch
```typescript
// HARDENED IMPLEMENTATION (Post-Patch)
import { parse } from 'psl'; // Public Suffix List parser

export function validateInstitutionalDomain(email: string, allowedDomains: Set<string>): boolean {
  if (!email || typeof email !== 'string') return false;
  
  // RFC 5322 Compliant Email Split
  const parts = email.trim().toLowerCase().split('@');
  if (parts.length !== 2) return false;
  
  const domain = parts[1];
  
  // Strict exact domain or verified subdomain matching
  return allowedDomains.has(domain) || Array.from(allowedDomains).some(allowed => domain.endsWith('.' + allowed));
}
```

* **Public Suffix Parsing:** Email domains are parsed using the Public Suffix List to prevent top-level domain manipulation.
* **Strict Middleware Boundary:** Domain-level attributes are never used for authorization. Role-Based Access Control (RBAC) is derived strictly from verified cryptographic claims signed inside the JWT payload.
* **Regression Test Suite:** Added unit tests covering 50+ malicious and malformed email inputs to guarantee immunity against boundary bypasses.

---

## Ongoing Security Architecture & Hardening Measures

Following these incidents, CVInsight implemented a defense-in-depth policy across the entire stack:

### 1. Hardened Session Security
* Authentication tokens are stored exclusively in **`HttpOnly`**, **`Secure`**, **`SameSite=Lax`** cookies.
* JavaScript running in the browser cannot read session tokens, eliminating token theft via Cross-Site Scripting (XSS).

### 2. NoSQL Injection Defense
* All user-supplied inputs pass through `mongo-sanitize`, stripping reserved MongoDB operators (`$gt`, `$ne`, `$where`, etc.) from query payloads.

### 3. Strict Cross-Origin Resource Sharing (CORS)
* Access-Control-Allow-Origin is strictly locked to `https://www.cvinsight.me` and authenticated Manifest V3 extension IDs in production, rejecting untrusted origins.

### 4. Zero Credentials in Repository Policy
* All API keys, database connection strings, and webhook secrets are loaded via environment variables at runtime.
* Git hooks enforce pre-commit pattern matching to prevent credentials from ever being committed to version control.

---

## Lessons for Student & Indie Builders

1. **Assume Automated Bots Will Find You:** The moment your app is indexed on Google or shared on LinkedIn/Reddit, bots will discover every open POST endpoint. Build anti-bot defenses on Day 1.
2. **Never Rely on Client-Side Validation Alone:** Attackers bypass UI validation by sending raw HTTP requests. Every check on the frontend must have an equal or stricter counterpart on the backend.
3. **Design for Rate-Limit Failures:** Upstream APIs will fail or rate-limit you. Building graceful fallback routing is not just a convenience — it is fundamental operational security.
