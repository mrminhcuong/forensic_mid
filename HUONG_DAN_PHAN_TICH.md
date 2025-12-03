# HƯỚNG DẪN PHÂN TÍCH CHI TIẾT

## PHẦN 1: PHÂN TÍCH LOG VỚI APACHE LOG VIEWER (Windows)

### Bước 1: Import Log File
1. Mở Apache Log Viewer
2. File → Open → Chọn `AppServer.csv`
3. Format: **CSV** (hoặc tự động detect)

### Bước 2: Lọc theo IP Address
```
Filter by IP: 192.168.174.157
→ Kết quả: Attacker sử dụng automated tools
→ Pattern: 200+ requests trong 9 giây
```

```
Filter by IP: 192.168.174.150
→ Kết quả: Manual testing, DVWA login
→ Pattern: Chậm hơn, có tương tác người dùng
```

### Bước 3: Phân tích Request URLs
**Sắp xếp theo:**
- **URL** → Tìm các endpoint bị attack
- **Response Code** → Tìm 200, 302, 404, 500
- **Time** → Timeline tấn công

**Tìm kiếm pattern:**
```
Search for: UNION
Search for: SELECT
Search for: SLEEP
Search for: CONCAT
Search for: 0x (hex encoding)
```

### Bước 4: Export Report
- Statistics → Request per IP
- Top URLs accessed
- Error codes distribution
- Timeline graph

---

## PHẦN 2: PHÂN TÍCH LOG VỚI SAWMILL (Windows)

### Bước 1: Create New Profile
1. Mở SawMill
2. Profile → New Profile
3. Log Format: **Apache Combined Log**
4. Import `AppServer.csv`

### Bước 2: Generate Reports

**Report 1: Overview Dashboard**
- Total requests: ~300+
- Unique IPs: 4
- Time range: Apr 2018 - Dec 2018
- Peak activity: 30/Dec/2018 07:13-07:28

**Report 2: IP Analysis**
- Sort by: Number of requests
- 192.168.174.157: 200+ requests (automated)
- 192.168.174.150: 100+ requests (manual)
- 192.168.116.146: 6 requests (reconnaissance)
- 192.168.10.105: 6 requests (reconnaissance)

**Report 3: URL Analysis**
- Most accessed: `/dvwa/vulnerabilities/sqli/`
- Attack vectors:
  - `/phpMyAdmin/` - 8 accesses
  - `/dvwa/` - 150+ accesses
  - `/phpinfo.php` - 4 accesses
  - `/?-d+allow_url_include=1...` - 3 PHP injection attempts

**Report 4: Response Codes**
- 200 OK: 250+ (successful attacks)
- 302 Redirect: 20+ (DVWA redirects)
- 404 Not Found: 10+ (reconnaissance)
- 500 Error: 1 (PHP CGI injection failed)

**Report 5: Timeline Analysis**
```
Timeline Graph:
  Apr 2018: ▂ (2 requests - scanning)
  May 2018: ▂ (6 requests - scanning)
  Dec 2018: ████████████ (280+ requests - main attack)
```

### Bước 3: Filter & Drill Down

**Filter SQL Injection:**
```sql
URL contains: 
  - "UNION"
  - "SELECT"
  - "AND"
  - "OR"
  - "%27" (')
  - "0x" (hex)
```

**Result:** 200+ SQL injection attempts

**Filter PHP Injection:**
```
URL contains:
  - "allow_url_include"
  - "safe_mode"
  - "disable_functions"
  - "php://input"
```

**Result:** 3 PHP CGI injection attempts (all failed)

### Bước 4: Export Evidence
- PDF Report với graphs
- CSV filtered data
- Screenshots của attack patterns

---

## PHẦN 3: PHÂN TÍCH PCAP VỚI WIRESHARK

### Bước 1: Mở File PCAP
```bash
# Trên Kali Linux
wireshark "Network traffic.pcap" &

# Hoặc trên Windows
"C:\Program Files\Wireshark\Wireshark.exe" "Network traffic.pcap"
```

### Bước 2: Display Filters Quan Trọng

#### Filter 1: HTTP Traffic Only
```
http
```

#### Filter 2: Attacker Traffic
```
ip.src == 192.168.174.157 or ip.src == 192.168.174.150
```

#### Filter 3: SQL Injection Attempts
```
http.request.uri contains "UNION" or 
http.request.uri contains "SELECT" or
http.request.uri contains "%27"
```

#### Filter 4: POST Requests (Potential Attacks)
```
http.request.method == "POST"
```

#### Filter 5: PHP Injection Attempts
```
http.request.uri contains "allow_url_include" or
http.request.uri contains "safe_mode"
```

#### Filter 6: Successful Responses
```
http.response.code == 200
```

#### Filter 7: Authentication
```
http.cookie or http.authorization
```

### Bước 3: Follow TCP Streams

**Stream 1: DVWA Login**
```
1. Find POST to /dvwa/login.php
2. Right-click → Follow → TCP Stream
3. Xem username/password: admin/password
4. Xem session cookie: PHPSESSID
```

**Stream 2: SQL Injection**
```
1. Find GET to /dvwa/vulnerabilities/sqli/?id=...
2. Follow TCP Stream
3. Xem toàn bộ conversation
4. Identify attack patterns
```

**Stream 3: PHP CGI Injection**
```
1. Find POST to /?-d+allow_url_include...
2. Follow TCP Stream
3. Xem payload details
4. Response: 500 Internal Server Error
```

### Bước 4: Export Data

**Export HTTP Objects:**
```
File → Export Objects → HTTP
→ Save all HTML, CSS, JS responses
```

**Export Specific Packets:**
```
File → Export Specified Packets
→ Filter: http.request.uri contains "UNION"
→ Save as: sqli_attacks.pcap
```

**Export as CSV:**
```
File → Export Packet Dissections → As CSV
→ Visible columns only
→ Save as: http_traffic.csv
```

### Bước 5: Statistics Analysis

**Conversations:**
```
Statistics → Conversations → TCP
→ Sort by Packets
→ Identify attacker connections
```

**HTTP Analysis:**
```
Statistics → HTTP → Requests
→ Top requested URIs
→ Status codes distribution
```

**IO Graph:**
```
Statistics → IO Graphs
→ Y Axis: Packets per second
→ Filter: ip.src == 192.168.174.157
→ Visualize attack intensity
```

### Bước 6: Timeline Reconstruction

1. **File → Export → Time Sequence (Stevens)**
2. Graph hiển thị:
   - Connection establishment
   - Data transfer
   - Attack bursts
   - Connection termination

---

## PHẦN 4: PHÂN TÍCH VỚI KALI LINUX TOOLS

### Tool 1: grep (Command Line Analysis)

**Count SQL Injection Attempts:**
```bash
grep -c "UNION" AppServer.csv
grep -c "SELECT" AppServer.csv
grep -c "%27" AppServer.csv
```

**Extract Attacker IPs:**
```bash
grep -oP '\d+\.\d+\.\d+\.\d+' AppServer.csv | sort -u
```

**Timeline Analysis:**
```bash
grep "192.168.174.157" AppServer.csv | \
  awk -F',' '{print $4}' | \
  cut -d':' -f1-2 | \
  sort | uniq -c
```

**Find Specific Attack Patterns:**
```bash
# PHP Injection
grep "allow_url_include" AppServer.csv

# SQL Injection with UNION
grep "UNION.*SELECT" AppServer.csv

# Time-based SQLi
grep -E "(SLEEP|WAITFOR|PG_SLEEP)" AppServer.csv
```

### Tool 2: tshark (Command Line Wireshark)

**Extract HTTP Requests:**
```bash
tshark -r "Network traffic.pcap" -Y "http.request" \
  -T fields -e frame.time -e ip.src -e http.request.uri
```

**Extract POST Data:**
```bash
tshark -r "Network traffic.pcap" -Y "http.request.method == POST" \
  -T fields -e ip.src -e http.request.uri -e http.file_data
```

**Extract Cookies:**
```bash
tshark -r "Network traffic.pcap" -Y "http.cookie" \
  -T fields -e ip.src -e http.cookie
```

**Count Packets by IP:**
```bash
tshark -r "Network traffic.pcap" -q -z conv,ip
```

### Tool 3: tcpdump (Quick Analysis)

**Filter HTTP Traffic:**
```bash
tcpdump -r "Network traffic.pcap" -A 'tcp port 80'
```

**Extract to Text:**
```bash
tcpdump -r "Network traffic.pcap" -nnvvXS > traffic_dump.txt
```

### Tool 4: Python Script (Custom Analysis)

**sqli_detector.py:**
```python
#!/usr/bin/env python3
import csv
import re

sqli_patterns = [
    r"UNION.*SELECT",
    r"'.*OR.*'.*=.*'",
    r"SLEEP\(\d+\)",
    r"WAITFOR.*DELAY",
    r"CONCAT\(",
    r"0x[0-9a-f]+",
    r"--[^\n]*$",
    r";\s*DROP",
]

with open('AppServer.csv', 'r') as f:
    reader = csv.DictReader(f)
    sqli_count = 0
    
    for row in reader:
        url = row.get('Request', '')
        for pattern in sqli_patterns:
            if re.search(pattern, url, re.IGNORECASE):
                sqli_count += 1
                print(f"[SQLI] {row['IP Address']} → {url[:100]}...")
                break
    
    print(f"\nTotal SQL Injection attempts: {sqli_count}")
```

**Run:**
```bash
chmod +x sqli_detector.py
./sqli_detector.py
```

---

## PHẦN 5: TẠO BÁO CÁO CUỐI CÙNG

### Cấu trúc Report:

**1. Executive Summary**
- Tóm tắt tình huống
- Mức độ nguy hiểm
- Tác động

**2. Timeline**
- Chronological events
- Graph/chart minh họa

**3. Technical Analysis**
- Attack vectors chi tiết
- Tools sử dụng
- Exploitation attempts

**4. Evidence**
- Screenshots từ Wireshark
- Log entries quan trọng
- PCAP analysis results

**5. Indicators of Compromise (IOCs)**
- Attacker IPs
- Attack signatures
- Malicious patterns

**6. Recommendations**
- Immediate actions
- Long-term security improvements
- Monitoring requirements

### Tools để tạo Report:

**1. Apache Log Viewer**
- Export Statistics as PDF
- Include graphs và charts

**2. SawMill**
- Generate comprehensive HTML report
- Export as PDF với logo

**3. Wireshark**
- Screenshot important packets
- Export conversation statistics

**4. LibreOffice/Word**
- Combine tất cả evidence
- Format professional document

**5. Markdown → PDF**
```bash
# Sử dụng pandoc
pandoc BAO_CAO_DIEU_TRA.md -o BAO_CAO_FINAL.pdf \
  --toc --highlight-style=tango
```

---

## CHECKLIST ĐIỀU TRA

### Phase 1: Log Analysis
- [ ] Import log vào Apache Log Viewer
- [ ] Import log vào SawMill
- [ ] Identify attacker IPs
- [ ] Count requests per IP
- [ ] Identify attack patterns
- [ ] Create timeline
- [ ] Export statistics

### Phase 2: PCAP Analysis
- [ ] Open trong Wireshark
- [ ] Apply display filters
- [ ] Follow TCP streams
- [ ] Extract HTTP objects
- [ ] Identify credentials
- [ ] Identify session cookies
- [ ] Export evidence

### Phase 3: Pattern Recognition
- [ ] Identify reconnaissance phase
- [ ] Identify enumeration phase
- [ ] Identify exploitation attempts
- [ ] Identify successful attacks
- [ ] Document attack flow

### Phase 4: Correlation
- [ ] Match log entries với PCAP
- [ ] Verify timestamps
- [ ] Confirm attacker IPs
- [ ] Validate attack success
- [ ] Build complete narrative

### Phase 5: Reporting
- [ ] Write executive summary
- [ ] Document technical details
- [ ] Include screenshots
- [ ] List IOCs
- [ ] Provide recommendations
- [ ] Review và finalize

---

## TÀI LIỆU THAM KHẢO

### Online Resources:
- Wireshark User Guide: https://www.wireshark.org/docs/
- Apache Log Format: https://httpd.apache.org/docs/current/logs.html
- OWASP SQL Injection: https://owasp.org/www-community/attacks/SQL_Injection
- CVE-2012-1823: https://nvd.nist.gov/vuln/detail/CVE-2012-1823

### Books:
- "Network Forensics" - Sherri Davidoff
- "The Art of Memory Forensics" - Michael Hale Ligh
- "Practical Packet Analysis" - Chris Sanders

### Tools Documentation:
- Wireshark Display Filters: https://wiki.wireshark.org/DisplayFilters
- tcpdump man page: `man tcpdump`
- tshark manual: `man tshark`

---

**Lưu ý:** Đây là tài liệu hướng dẫn cho mục đích học tập và nghiên cứu. Không sử dụng các kỹ thuật này cho mục đích bất hợp pháp.
