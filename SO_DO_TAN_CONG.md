# SƠ ĐỒ TẤN CÔNG CHI TIẾT - ASCII ART

## 1. TOPOLOGY MẠNG NỘI BỘ

```
╔═══════════════════════════════════════════════════════════════════════════╗
║                          INTERNAL CORPORATE NETWORK                        ║
║                          (Isolated - No Internet)                          ║
║                                                                            ║
║   ┌─────────────────┐         ┌──────────────────┐      ┌──────────────┐ ║
║   │   ATTACKER #1   │         │                  │      │              │ ║
║   │ 192.168.116.146 │────┐    │  CACHING PROXY   │      │ APPLICATION  │ ║
║   │   (Nmap Scan)   │    │    │   SERVER         │      │   SERVER     │ ║
║   └─────────────────┘    │    │                  │      │              │ ║
║                          ├───►│  - Squid/Apache  │─────►│ - Apache     │ ║
║   ┌─────────────────┐    │    │  - Caching       │ HTTP │ - PHP 5.x    │ ║
║   │   ATTACKER #2   │    │    │  - Filtering?    │ Fwd  │ - MySQL      │ ║
║   │ 192.168.10.105  │────┤    │  - Load Balance  │      │ - DVWA       │ ║
║   │   (Nmap Scan)   │    │    │                  │      │ - PHPMyAdmin │ ║
║   └─────────────────┘    │    └──────────────────┘      │ - TWiki      │ ║
║                          │             │                 └──────────────┘ ║
║   ┌─────────────────┐    │             │                        │         ║
║   │   ATTACKER #3   │    │             │                        │         ║
║   │ 192.168.174.150 │────┤             └────────────────────────┘         ║
║   │ (Manual Test)   │    │           Logs to:                             ║
║   └─────────────────┘    │           /var/log/apache2/access.log          ║
║                          │                                                 ║
║   ┌─────────────────┐    │                                                 ║
║   │   ATTACKER #4   │    │                                                 ║
║   │ 192.168.174.157 │────┘                                                 ║
║   │(SQLMap/Havij)   │                                                      ║
║   └─────────────────┘                                                      ║
║                                                                            ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

---

## 2. ATTACK FLOW DIAGRAM

```
┌──────────────┐
│   ATTACKER   │
│  Workstation │
└──────┬───────┘
       │
       │ Phase 1: RECONNAISSANCE
       ▼
┌──────────────────────────────────────┐
│  Nmap Port Scan                      │
│  - Port 80 (HTTP) - OPEN             │
│  - Port 443 (HTTPS) - CLOSED         │
│  - Port 22 (SSH) - FILTERED          │
│  - Port 3306 (MySQL) - FILTERED      │
└──────┬───────────────────────────────┘
       │
       │ Phase 2: ENUMERATION
       ▼
┌──────────────────────────────────────┐
│  Web Directory Scanning              │
│  ✓ /phpMyAdmin/     → 200 OK         │
│  ✓ /dvwa/           → 302 Redirect   │
│  ✓ /twiki/          → 200 OK         │
│  ✓ /dav/            → 200 OK         │
│  ✓ /phpinfo.php     → 200 OK         │
│  ✗ /admin/          → 404 Not Found  │
└──────┬───────────────────────────────┘
       │
       │ Phase 3: VULNERABILITY IDENTIFICATION
       ▼
┌──────────────────────────────────────┐
│  Information Gathering               │
│  - PHP Version: 5.3.10               │
│  - MySQL Version: 5.5.x              │
│  - Server: Apache/2.2.22 (Ubuntu)    │
│  - Vulnerable apps detected          │
└──────┬───────────────────────────────┘
       │
       │ Phase 4: EXPLOITATION ATTEMPTS
       ▼
┌──────────────────────────────────────────────────┐
│  Attack Vector 1: PHP CGI Injection              │
│  Exploit: CVE-2012-1823                          │
│  Payload: /?-d allow_url_include=1...            │
│  Result: 500 Internal Server Error ✗ FAILED     │
└──────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────┐
│  Attack Vector 2: PHPMyAdmin Login Bruteforce   │
│  Method: Dictionary attack                       │
│  Attempts: admin/admin, admin/password, root/... │
│  Result: Access Denied ✗ FAILED                 │
└──────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────┐
│  Attack Vector 3: DVWA SQL Injection            │
│  Method: Automated (SQLMap) + Manual            │
│  Target: /dvwa/vulnerabilities/sqli/             │
│  Result: ✓ SUCCESS (Low security level)         │
└──────┬───────────────────────────────────────────┘
       │
       │ Phase 5: POST-EXPLOITATION
       ▼
┌──────────────────────────────────────┐
│  Data Extraction via SQLi            │
│  - Database enumeration              │
│  - Table listing                     │
│  - User credentials dump             │
│  - Privilege escalation attempts     │
└──────────────────────────────────────┘
```

---

## 3. ATTACK TIMELINE (Chi tiết)

```
═════════════════════════════════════════════════════════════════════════════
                            TIMELINE VISUALIZATION
═════════════════════════════════════════════════════════════════════════════

2018-04-22  ▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
            │
            └──► 14:45:12 - Nmap scan from 192.168.116.146
                 - GET /nmaplowercheck1524422717 (404)
                 - POST /sdk (404) - VMware vuln check
                 - GET /HNAP1 (404) - Router vuln check
                 
═════════════════════════════════════════════════════════════════════════════

2018-05-14  ░░░░░░░░░░░░░░░░▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                            │
                            └──► 16:02:26 - Nmap scan from 192.168.10.105
                                 - Same pattern as previous scan
                                 - Automated reconnaissance

═════════════════════════════════════════════════════════════════════════════

2018-12-30  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
06:43:45    │                                              │
            └──► Initial web enumeration (192.168.174.150) │
                 GET / → 200 OK                            │
                 GET /favicon.ico → 404                    │
                                                            │
06:43:54    Access PHPMyAdmin                              │
            GET /phpMyAdmin/ → 200 OK                      │
            POST /phpMyAdmin/index.php → 302 (login fail)  │
                                                            │
06:44:17    Explore TWiki                                  │
            GET /twiki/bin/view/Main/WebHome → 200 OK      │
                                                            │
06:51:24    Second attacker IP appears (192.168.174.157)   │
            GET /phpMyAdmin/ → 200 OK                      │
            POST /phpMyAdmin/index.php → 302 (login fail)  │
                                                            │
06:58:52    DVWA Discovery                                 │
            GET /dvwa/ → 302 → GET /dvwa/login.php → 200   │
            POST /dvwa/login.php → 302 (login fail)        │
                                                            │
07:01:23    Information Disclosure                         │
            GET /phpinfo.php → 200 OK                      │
            !! PHP 5.3.10 detected - Vulnerable version !! │
                                                            │
07:02:35    ▓▓▓ PHP CGI INJECTION ATTEMPT #1 ▓▓▓          │
            POST /?-d+allow_url_include=1+-d+safe_mode=Off │
            Response: 200 0 bytes (no execution)           │
                                                            │
07:05:36    ▓▓▓ PHP CGI INJECTION ATTEMPT #2 ▓▓▓          │
            POST /?--define+allow_url_include=On...         │
            Response: 200 0 bytes (still failed)           │
                                                            │
07:12:33    ✓✓✓ DVWA LOGIN SUCCESS ✓✓✓                    │
            POST /dvwa/login.php                           │
            Credentials: admin/password (default)          │
            → GET /dvwa/index.php → 200 OK                 │
                                                            │
07:13:26    ▓▓▓▓▓▓▓▓ AUTOMATED SQLi ATTACK ▓▓▓▓▓▓▓▓       │
            Source: 192.168.174.157 (SQLMap/Havij)         │
            Target: /dvwa/vulnerabilities/sqli/?id=        │
            Duration: 9 seconds                            │
            Requests: 200+                                 │
            Techniques tested:                             │
              - Error-based SQLi                           │
              - Union-based SQLi                           │
              - Boolean-based Blind SQLi                   │
              - Time-based Blind SQLi                      │
              - Stacked queries                            │
            Response: 302 Redirect (Medium security)       │
                                                            │
07:22:25    Security Level Changed                         │
            POST /dvwa/security.php                        │
            Changed from Medium to... (Low/High?)          │
                                                            │
07:28:13    ▓▓▓▓▓▓▓▓ MANUAL + AUTO SQLi ▓▓▓▓▓▓▓▓          │
            Source: 192.168.174.150                        │
            200+ requests in 28 seconds                    │
            Testing with different payloads                │
            Mix of manual and automated                    │
                                                            │
07:28:41    Attack sequence ends                           │
                                                            │
═════════════════════════════════════════════════════════════════════════════

LEGEND:
▓ = Active attack period
░ = Idle period
─ = Timeline flow
```

---

## 4. SQL INJECTION ATTACK PATTERNS

```
┌───────────────────────────────────────────────────────────────────────┐
│                    SQL INJECTION TECHNIQUE TREE                        │
└───────────────────────────────────────────────────────────────────────┘

                         ┌─── Error-based SQLi
                         │    └─► UNION SELECT with invalid syntax
                         │        └─► Force MySQL error with data extraction
                         │            GET /sqli/?id=12' AND (SELECT ... FROM
                         │              (SELECT COUNT(*),CONCAT(0x71766b7171,
                         │              (SELECT (ELT(4481=4481,1))),...)
                         │
                         ├─── Union-based SQLi
                         │    └─► Column count detection
                         │        ├─► id=12 UNION ALL SELECT NULL--
                         │        ├─► id=12 UNION ALL SELECT NULL,NULL--
                         │        └─► ... up to 10 columns tested
┌─────────────┐          │
│  SQL INJECTION │────────┤
│  TECHNIQUES    │         ├─── Boolean-based Blind SQLi
└─────────────┘          │    └─► True/False condition testing
                         │        ├─► id=12 AND 1=1  (True → Normal response)
                         │        └─► id=12 AND 1=2  (False → Different response)
                         │
                         ├─── Time-based Blind SQLi
                         │    └─► Delay injection
                         │        ├─► MySQL:  id=12 AND SLEEP(5)
                         │        ├─► MSSQL:  id=12 WAITFOR DELAY '0:0:5'
                         │        ├─► PgSQL:  id=12 AND pg_sleep(5)
                         │        └─► Oracle: id=12;SELECT DBMS_PIPE.RECEIVE_MESSAGE(...)
                         │
                         └─── Stacked Queries
                              └─► Multiple statements
                                  └─► id=12; DROP TABLE users--
                                      id=12'; SELECT version()--

═══════════════════════════════════════════════════════════════════════════

PAYLOAD ENCODING:
═══════════════════════════════════════════════════════════════════════════

Original:   id=12' UNION SELECT username,password FROM users--
            ↓
URL Encode: id=12%27%20UNION%20SELECT%20username%2Cpassword%20FROM%20users--%20
            ↓
Double:     id=12%2527%2520UNION%2520SELECT%2520username%252C...
            ↓
Hex:        id=12' UNION SELECT 0x61646d696e,0x70617373776f7264...
            ↓
Unicode:    id=12\u0027 UNION SELECT username,password...
            ↓
Base64:     id=MTInIFVOSU9OIFNFTEVDVCB1c2VybmFtZSxwYXNzd29yZA==

═══════════════════════════════════════════════════════════════════════════
```

---

## 5. DATA FLOW DIAGRAM

```
╔════════════════════════════════════════════════════════════════════════╗
║                        DATA FLOW ANALYSIS                               ║
╚════════════════════════════════════════════════════════════════════════╝

┌───────────────┐
│   ATTACKER    │
│   Browser/    │
│   Tool        │
└───────┬───────┘
        │
        │ 1. HTTP GET Request
        │    GET /dvwa/vulnerabilities/sqli/?id=12' UNION SELECT 1,2--
        │    Cookie: PHPSESSID=abc123; security=low
        ▼
┌─────────────────────────────────────────────────────────────────────┐
│   CACHING PROXY                                                     │
│   - Check cache: MISS (dynamic content)                             │
│   - Log request                                                     │
│   - Add X-Forwarded-For header                                      │
│   - Forward to application server                                   │
└────────┬────────────────────────────────────────────────────────────┘
         │
         │ 2. Forwarded HTTP Request
         │    + X-Forwarded-For: 192.168.174.157
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│   APPLICATION SERVER (Apache)                                        │
│   - Receive request                                                 │
│   - Parse URL parameters                                            │
│   - Load PHP interpreter                                            │
│   - Execute DVWA code                                               │
└────────┬────────────────────────────────────────────────────────────┘
         │
         │ 3. SQL Query Construction
         │    $query = "SELECT * FROM users WHERE id='$id'";
         │    → Becomes: SELECT * FROM users WHERE id='12' UNION SELECT 1,2--'
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│   MySQL DATABASE                                                     │
│   - Execute malicious query                                         │
│   - Return data:                                                    │
│     Row 1: Legitimate user data                                     │
│     Row 2: Injected values (1, 2)                                   │
└────────┬────────────────────────────────────────────────────────────┘
         │
         │ 4. Database Response
         │    Results: id=12, name="John" + id=1, name=2
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│   PHP Processing                                                     │
│   - Render HTML with database results                               │
│   - Include injected data in output                                 │
│   - Send HTTP response                                              │
└────────┬────────────────────────────────────────────────────────────┘
         │
         │ 5. HTTP Response
         │    200 OK
         │    Content: <html>...John...1...2...</html>
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│   CACHING PROXY                                                     │
│   - Receive response                                                │
│   - Log response                                                    │
│   - Do NOT cache (Set-Cookie present)                               │
│   - Forward to attacker                                             │
└────────┬────────────────────────────────────────────────────────────┘
         │
         │ 6. Final Response
         │    200 OK + Extracted data
         ▼
┌───────────────┐
│   ATTACKER    │
│   Receives:   │
│   ✓ User data │
│   ✓ Evidence  │
│     of SQLi   │
└───────────────┘
```

---

## 6. INDICATORS OF COMPROMISE (IOCs)

```
╔═══════════════════════════════════════════════════════════════════════╗
║                   INDICATORS OF COMPROMISE                             ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                        ║
║  [NETWORK IOCs]                                                        ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │ • Source IPs:                                                  │  ║
║  │   - 192.168.116.146  (Reconnaissance)                         │  ║
║  │   - 192.168.10.105   (Reconnaissance)                         │  ║
║  │   - 192.168.174.150  (Main attacker)                          │  ║
║  │   - 192.168.174.157  (Automated tools)                        │  ║
║  │                                                                │  ║
║  │ • Unusual traffic patterns:                                   │  ║
║  │   - 200+ requests in < 10 seconds                             │  ║
║  │   - Requests to /dvwa/vulnerabilities/sqli/                   │  ║
║  │   - Multiple 302 redirects (security blocking)                │  ║
║  │   - Requests with %27 (') in URL                              │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
║                                                                        ║
║  [WEB APPLICATION IOCs]                                                ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │ • SQL Keywords in URLs:                                        │  ║
║  │   - UNION                                                      │  ║
║  │   - SELECT                                                     │  ║
║  │   - CONCAT                                                     │  ║
║  │   - 0x[hex]                                                    │  ║
║  │   - SLEEP()                                                    │  ║
║  │   - WAITFOR                                                    │  ║
║  │                                                                │  ║
║  │ • PHP Injection patterns:                                      │  ║
║  │   - /?-d+allow_url_include=1                                  │  ║
║  │   - safe_mode=Off                                             │  ║
║  │   - disable_functions=""                                      │  ║
║  │   - php://input                                               │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
║                                                                        ║
║  [BEHAVIORAL IOCs]                                                     ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │ • Failed login attempts:                                       │  ║
║  │   - PHPMyAdmin: Multiple POST to /phpMyAdmin/index.php        │  ║
║  │   - DVWA: Failed then successful                              │  ║
║  │                                                                │  ║
║  │ • Information gathering:                                       │  ║
║  │   - Access to /phpinfo.php                                    │  ║
║  │   - Directory listing requests                                │  ║
║  │   - Nmap fingerprinting                                       │  ║
║  │                                                                │  ║
║  │ • Persistence attempts:                                        │  ║
║  │   - Multiple sessions                                         │  ║
║  │   - Different IP addresses                                    │  ║
║  │   - Tools rotation (manual → automated)                       │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
║                                                                        ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

## 7. KILL CHAIN MAPPING

```
╔═══════════════════════════════════════════════════════════════════════╗
║            CYBER KILL CHAIN - Lockheed Martin Model                   ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                        ║
║  1. RECONNAISSANCE  ✓ COMPLETED                                       ║
║     ├─ Port scanning (Nmap)                                          ║
║     ├─ Service enumeration                                           ║
║     └─ OS fingerprinting                                             ║
║                                                                        ║
║  2. WEAPONIZATION   ✓ COMPLETED                                       ║
║     ├─ SQLMap/Havij configured                                       ║
║     ├─ PHP CGI exploit prepared                                      ║
║     └─ Wordlists for bruteforce                                      ║
║                                                                        ║
║  3. DELIVERY        ✓ COMPLETED                                       ║
║     ├─ HTTP GET/POST requests                                        ║
║     ├─ Direct web access                                             ║
║     └─ Through caching proxy                                         ║
║                                                                        ║
║  4. EXPLOITATION    ⚠ PARTIAL                                         ║
║     ├─ SQL Injection: ✓ SUCCESS (DVWA Low)                           ║
║     ├─ PHP CGI Injection: ✗ FAILED (500 error)                      ║
║     └─ Login bruteforce: ✗ FAILED (except DVWA default)             ║
║                                                                        ║
║  5. INSTALLATION    ✗ NOT REACHED                                     ║
║     └─ No backdoor/webshell uploaded                                 ║
║                                                                        ║
║  6. COMMAND & CONTROL  ✗ NOT REACHED                                  ║
║     └─ No C2 communication established                               ║
║                                                                        ║
║  7. ACTIONS ON OBJECTIVES  ⚠ ATTEMPTED                                ║
║     ├─ Data exfiltration via SQLi (possible)                         ║
║     ├─ Privilege escalation (attempted)                              ║
║     └─ Lateral movement (not observed)                               ║
║                                                                        ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

**Tài liệu:** Sơ đồ tấn công và phân tích chi tiết
**Mục đích:** Học tập và nghiên cứu Digital Forensics
**Tạo bởi:** Investigation Team
**Ngày:** 3 December 2025
