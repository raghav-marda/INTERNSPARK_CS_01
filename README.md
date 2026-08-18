# InternSpark Cybersecurity Internship — Task 1: Reconnaissance & Vulnerability Scanning

## Internship / Task Information
**Internship:** InternSpark Cybersecurity Internship  
**Task:** Task 1 — Reconnaissance & Vulnerability Scanning  
**Target:** Smart Expense Tracker  
**Target URL:** https://smart-expense-tracker-0pon.onrender.com/login  
**Application:** Python / Flask / SQLite  
**Deployment:** Render  
**Authorization:** This is my own deployed application and I was authorized to perform the security testing.

## Objective
The objective was to perform authorized reconnaissance and vulnerability scanning against the deployed Smart Expense Tracker application using Nmap 7.99.1, identify observable network and HTTP security characteristics, and document findings without treating infrastructure-level observations or inconclusive scan results as confirmed application vulnerabilities.

## Scope
- TCP port reconnaissance
- HTTP/HTTPS service and version detection
- Nmap vulnerability scripts on ports 80 and 443
- HTTP title and header inspection
- HTTP security-header inspection
- Documentation of scan limitations and failed enumeration attempts

No destructive Slowloris or denial-of-service testing was performed.

## Tools Used
- Nmap 7.99.1
- Windows Command Prompt

**Nikto was not used.** The command was attempted, but Nikto was not installed. WSL was also checked and was not installed.  
**OpenVAS was not used.**

## Methodology and Commands

### 1. Basic Port Scan
```text
nmap smart-expense-tracker-0pon.onrender.com
```

Observed:
- Host was up.
- 991 TCP ports were reported as filtered.
- 25/tcp smtp
- 80/tcp http
- 110/tcp pop3
- 113/tcp closed ident
- 143/tcp imap
- 443/tcp https
- 8010/tcp xmpp
- 8080/tcp http-proxy
- 8443/tcp https-alt

Because the target is deployed on Render and is behind Cloudflare, these ports should not automatically be interpreted as services running directly inside the Smart Expense Tracker Flask application.

Evidence: `screenshots/01_nmap_port_scan.png`

### 2. Service / Version Detection
```text
nmap -sV -p 80,443 smart-expense-tracker-0pon.onrender.com
```

Observed:
```text
80/tcp   open  http      Cloudflare http proxy
443/tcp  open  ssl/http  Cloudflare http proxy
```

Nmap primarily detected the Cloudflare proxy/edge layer rather than directly identifying the underlying Flask application server.

Evidence: `screenshots/02_service_version_detection.png`

### 3. Vulnerability Scan
```text
nmap --script vuln -p 80,443 smart-expense-tracker-0pon.onrender.com
```

Observed:
- CSRF: Couldn't find any CSRF vulnerabilities.
- DOM-based XSS: Couldn't find any DOM based XSS.
- Stored XSS: Couldn't find any stored XSS vulnerabilities.
- `http-vuln-cve2014-3704`: script execution failed.
- `ssl-ccs-injection`: no reply from server (TIMEOUT).
- `http-fileupload-exploiter`: couldn't find a file-type field.
- Nmap's Slowloris check reported **LIKELY VULNERABLE**.

The Slowloris result is treated only as a **potential/likely exposure requiring further validation**, not as a confirmed vulnerability. No destructive Slowloris or DoS attack was performed.

Evidence: `screenshots/03_vulnerability_scan.png`

### 4. HTTP Title and Headers
```text
nmap --script http-title,http-headers -p 80,443 smart-expense-tracker-0pon.onrender.com
```

Observed:
- HTTP redirected to HTTPS.
- HTTPS redirected toward `/login`.
- `Server: cloudflare`
- `x-render-origin-server: Werkzeug/3.1.8 Python/3.14.3`
- `cf-cache-status: DYNAMIC`
- `alt-svc: h3=":443"; ma=86400`

The `x-render-origin-server` header exposes backend technology/version information: Werkzeug/3.1.8 and Python/3.14.3. This is documented as potential low-severity technology/version information disclosure, not proof of an exploitable vulnerability.

Evidence: `screenshots/04_http_headers.png`

### 5. HTTP Security Headers
```text
nmap --script http-security-headers -p 443 smart-expense-tracker-0pon.onrender.com
```

Result:
```text
Strict_Transport_Security:
HSTS not configured in HTTPS Server
```

HSTS does not appear to be configured at the scanned HTTPS endpoint.

Potential impact: browsers are not instructed to always access the website over HTTPS, which can increase exposure to downgrade/SSL-stripping scenarios under certain conditions.

Evidence: `screenshots/05_security_headers.png`

## Findings

### Finding 1 — HSTS Not Configured
**Severity: Low**

Nmap reported that HSTS was not configured at the scanned HTTPS endpoint.

**Recommendation:** Configure an appropriate `Strict-Transport-Security` header after confirming that the application is intended to operate exclusively over HTTPS.

### Finding 2 — Backend Technology / Version Information Disclosure
**Severity: Low / Informational**

The `x-render-origin-server` header exposed Werkzeug/3.1.8 and Python/3.14.3. This provides technology/version information that may assist fingerprinting, but is not evidence of an exploitable vulnerability by itself.

**Recommendation:** Review whether backend technology/version information needs to be exposed through response headers and minimize unnecessary version disclosure where practical.

### Finding 3 — Potential / Likely Slowloris Exposure
**Severity: Further validation required**

Nmap reported **LIKELY VULNERABLE** through its Slowloris check. This is not a confirmed vulnerability. No destructive Slowloris/DoS test was performed.

**Recommendation:** Validate the result using an authorized, controlled test. If confirmed, review connection handling, request timeouts, and applicable protective controls.

## Security Observations
- Cloudflare HTTP proxy was detected on HTTP/HTTPS.
- Multiple ports were observed, but their ownership should not be attributed directly to the Flask application because of Cloudflare/Render infrastructure.
- Automated Nmap checks did not identify CSRF, DOM-based XSS, or stored XSS.
- Negative automated scan results do not prove that the application is completely secure.
- Nmap errors/timeouts were treated as scan limitations rather than vulnerabilities.

## Limitations
- The assessment relied on Nmap 7.99.1 and the available scan responses.
- Cloudflare/Render limited direct visibility into the underlying application infrastructure.
- Some Nmap vulnerability scripts produced errors/timeouts.
- The `http-enum` attempt failed because of DNS resolution issues and was not treated as a successful enumeration result.
- Nikto was not available and therefore was not used.
- OpenVAS was not used.
- No destructive Slowloris or denial-of-service testing was performed.
- Automated XSS/CSRF checks are not equivalent to a complete manual application security assessment.

## Remediation Recommendations
1. Configure HSTS after confirming HTTPS-only operation.
2. Review and minimize unnecessary backend technology/version disclosure in response headers.
3. Perform controlled validation of the Nmap Slowloris result before treating it as a confirmed vulnerability.
4. If confirmed, review connection handling, request timeouts, and protective controls.
5. Repeat relevant scans after remediation.

## Screenshot Evidence
- `screenshots/01_nmap_port_scan.png` — Basic Nmap Port Scan
- `screenshots/02_service_version_detection.png` — Service / Version Detection
- `screenshots/03_vulnerability_scan.png` — Vulnerability Scan
- `screenshots/04_http_headers.png` — HTTP Title & Headers
- `screenshots/05_security_headers.png` — HTTP Security Headers

The failed HTTP enumeration attempt is not included as successful evidence.

## Conclusion
The authorized assessment identified several observable security characteristics of the deployed Smart Expense Tracker endpoint. The most actionable findings were the absence of HSTS and the exposure of backend technology/version information through response headers. Nmap also flagged a potential/likely Slowloris exposure, but this remains unconfirmed and requires further validation.

The observed ports and Cloudflare proxy responses were documented as infrastructure-level observations and were not attributed directly to the Flask application. Automated checks did not identify CSRF, DOM-based XSS, or stored XSS, while script errors and timeouts were treated as limitations. This assessment therefore provides useful reconnaissance and vulnerability-scanning evidence without overstating what the automated testing established.
