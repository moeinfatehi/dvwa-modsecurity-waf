# Testing Guide

## 🎯 Overview

This guide provides comprehensive testing procedures for verifying WAF protection functionality. The WAF is accessible on port 8000, while DVWA runs directly on port 80.

## 🚀 Quick Test Setup

### 1. Initial Setup Verification

```bash
# Verify containers are running
docker-compose ps

# Check WAF accessibility
curl -I http://localhost:8000

# Check DVWA direct access
curl -I http://localhost:80

# Wait for services to fully initialize
sleep 10
```

### 2. DVWA Configuration

1. **Access DVWA via WAF**: http://localhost:8000
2. **Login**: Default credentials are `admin/password`
3. **Setup Database**: Click "Create/Reset Database"
4. **Set Security Level**: Go to "DVWA Security" → Set to "Low"

## 🧪 Automated Testing Script

Create a comprehensive test script:

```bash
#!/bin/bash

echo "🛡️  DVWA ModSecurity WAF Testing Script"
echo "======================================="
echo

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Test function
test_attack() {
    local test_name="$1"
    local url="$2"
    local expected_code="$3"
    
    echo -n "Testing $test_name... "
    
    # Test via WAF (should be blocked)
    response_code=$(curl -s -o /dev/null -w "%{http_code}" "$url")
    
    if [ "$response_code" = "$expected_code" ]; then
        echo -e "${GREEN}✓ BLOCKED${NC} (HTTP $response_code)"
        return 0
    else
        echo -e "${RED}✗ NOT BLOCKED${NC} (HTTP $response_code)"
        return 1
    fi
}

# Test URLs
WAF_BASE="http://localhost:8000"
DIRECT_BASE="http://localhost:80"

echo "🔍 Testing WAF Protection (via port 8000):"
echo "----------------------------------------"

# SQL Injection Tests
test_attack "SQL Injection (Basic)" \
    "$WAF_BASE/vulnerabilities/sqli/?id=1' OR '1'='1&Submit=Submit" \
    "403"

test_attack "SQL Injection (Union)" \
    "$WAF_BASE/vulnerabilities/sqli/?id=1' UNION SELECT 1,2&Submit=Submit" \
    "403"

test_attack "SQL Injection (Time-based)" \
    "$WAF_BASE/vulnerabilities/sqli/?id=1' AND SLEEP(5)--&Submit=Submit" \
    "403"

# XSS Tests
test_attack "XSS (Basic Script)" \
    "$WAF_BASE/vulnerabilities/xss_r/?name=<script>alert(1)</script>" \
    "403"

test_attack "XSS (Event Handler)" \
    "$WAF_BASE/vulnerabilities/xss_r/?name=<img src=x onerror=alert(1)>" \
    "403"

test_attack "XSS (JavaScript URL)" \
    "$WAF_BASE/vulnerabilities/xss_r/?name=<a href=javascript:alert(1)>test</a>" \
    "403"

# Command Injection Tests
test_attack "Command Injection (Basic)" \
    "$WAF_BASE/vulnerabilities/exec/?ip=127.0.0.1;cat /etc/passwd&Submit=Submit" \
    "403"

test_attack "Command Injection (With Pipe)" \
    "$WAF_BASE/vulnerabilities/exec/?ip=127.0.0.1|whoami&Submit=Submit" \
    "403"

test_attack "Command Injection (Backticks)" \
    "$WAF_BASE/vulnerabilities/exec/?ip=127.0.0.1\`id\`&Submit=Submit" \
    "403"

# Path Traversal Tests
test_attack "Path Traversal (Basic)" \
    "$WAF_BASE/vulnerabilities/fi/?page=../../../etc/passwd" \
    "403"

test_attack "Path Traversal (Encoded)" \
    "$WAF_BASE/vulnerabilities/fi/?page=..%2F..%2F..%2Fetc%2Fpasswd" \
    "403"

echo
echo "🔓 Testing Direct Access (via port 80 - should NOT be blocked):"
echo "---------------------------------------------------------------"

# Test that attacks work when bypassing WAF
direct_response=$(curl -s -o /dev/null -w "%{http_code}" "$DIRECT_BASE/vulnerabilities/sqli/?id=1' OR '1'='1&Submit=Submit")
if [ "$direct_response" = "200" ]; then
    echo -e "Direct SQL Injection: ${YELLOW}✓ ALLOWED${NC} (HTTP $direct_response) - WAF bypassed"
else
    echo -e "Direct SQL Injection: ${RED}✗ BLOCKED${NC} (HTTP $direct_response) - Unexpected"
fi

echo
echo "📊 Testing Complete!"
echo "==================="
echo
echo "💡 Tips:"
echo "- If attacks are NOT blocked, check WAF logs: docker-compose logs modsecurity-waf"
echo "- If direct access fails, ensure DVWA is properly configured"
echo "- Check rule configuration in modsec-custom.conf"
```

Save this as `test-waf.sh` and run:

```bash
chmod +x test-waf.sh
./test-waf.sh
```

## 🔍 Manual Testing Procedures

### SQL Injection Testing

**Test 1: Basic SQL Injection**
```bash
# Via WAF (should be blocked)
curl "http://localhost:8000/vulnerabilities/sqli/?id=1' OR '1'='1&Submit=Submit"

# Direct access (should work)
curl "http://localhost:80/vulnerabilities/sqli/?id=1' OR '1'='1&Submit=Submit"
```

**Test 2: Advanced SQL Injection**
```bash
# UNION-based injection
curl "http://localhost:8000/vulnerabilities/sqli/?id=1' UNION SELECT 1,version()&Submit=Submit"

# Boolean-based injection
curl "http://localhost:8000/vulnerabilities/sqli/?id=1' AND 1=1&Submit=Submit"

# Time-based injection
curl "http://localhost:8000/vulnerabilities/sqli/?id=1' AND SLEEP(5)--&Submit=Submit"
```

### Cross-Site Scripting (XSS) Testing

**Test 1: Reflected XSS**
```bash
# Basic script tag
curl "http://localhost:8000/vulnerabilities/xss_r/?name=<script>alert('XSS')</script>"

# Event handler
curl "http://localhost:8000/vulnerabilities/xss_r/?name=<img src=x onerror=alert('XSS')>"

# JavaScript URL
curl "http://localhost:8000/vulnerabilities/xss_r/?name=<a href=javascript:alert('XSS')>Click</a>"
```

**Test 2: Stored XSS**
```bash
# Via web interface (visit http://localhost:8000/vulnerabilities/xss_s/)
# Try entering: <script>alert('Stored XSS')</script>
```

### Command Injection Testing

**Test 1: Basic Command Injection**
```bash
# With semicolon
curl "http://localhost:8000/vulnerabilities/exec/?ip=127.0.0.1;id&Submit=Submit"

# With pipe
curl "http://localhost:8000/vulnerabilities/exec/?ip=127.0.0.1|whoami&Submit=Submit"

# With backticks
curl "http://localhost:8000/vulnerabilities/exec/?ip=127.0.0.1\`cat /etc/passwd\`&Submit=Submit"
```

### File Inclusion Testing

**Test 1: Path Traversal**
```bash
# Basic traversal
curl "http://localhost:8000/vulnerabilities/fi/?page=../../../etc/passwd"

# URL encoded
curl "http://localhost:8000/vulnerabilities/fi/?page=..%2F..%2F..%2Fetc%2Fpasswd"

# Double encoding
curl "http://localhost:8000/vulnerabilities/fi/?page=..%252F..%252F..%252Fetc%252Fpasswd"
```

## 📊 Browser Testing

### Using Browser Developer Tools

1. **Open Developer Tools** (F12)
2. **Go to Network tab**
3. **Access**: http://localhost:8000
4. **Try attacks through the web interface**
5. **Check response codes**: 403 = Blocked, 200 = Allowed

### Testing with Burp Suite

1. **Configure proxy** to intercept requests
2. **Set proxy to** 127.0.0.1:8080
3. **Visit** http://localhost:8000
4. **Intercept and modify requests** to include payloads
5. **Forward requests** and observe responses

## 🔬 Advanced Testing

### Custom Payload Testing

Create a file `payloads.txt`:

```text
' OR '1'='1
<script>alert(1)</script>
; cat /etc/passwd
../../../etc/passwd
${7*7}
{{7*7}}
<img src=x onerror=alert(1)>
javascript:alert(1)
' UNION SELECT 1,2,3--
1' AND SLEEP(5)--
```

Test with curl:

```bash
#!/bin/bash
while IFS= read -r payload; do
    echo "Testing payload: $payload"
    encoded_payload=$(echo "$payload" | sed 's/ /%20/g')
    response=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:8000/vulnerabilities/sqli/?id=$encoded_payload&Submit=Submit")
    echo "Response: $response"
    echo "---"
done < payloads.txt
```

### Performance Testing

```bash
# Load testing with curl
for i in {1..100}; do
    curl -s "http://localhost:8000/" > /dev/null &
done
wait

# Check if WAF is still responsive
curl -I http://localhost:8000
```

## 📋 Test Results Analysis

### Expected Results

| Test Type | Via WAF (Port 8000) | Direct (Port 80) |
|-----------|---------------------|------------------|
| SQL Injection | 403 Forbidden | 200 OK |
| XSS | 403 Forbidden | 200 OK |
| Command Injection | 403 Forbidden | 200 OK |
| Path Traversal | 403 Forbidden | 200 OK |
| Normal Requests | 200 OK | 200 OK |

### Analyzing Logs

```bash
# View blocked requests
docker-compose logs modsecurity-waf | grep -i "denied\|blocked"

# View detailed audit log
docker-compose exec modsecurity-waf tail -f /var/log/modsec_audit.log

# Count blocked requests
docker-compose logs modsecurity-waf | grep -c "denied"

# View specific rule matches
docker-compose logs modsecurity-waf | grep "id \"1001\""
```

## 🛠️ Custom Test Development

### Creating New Tests

1. **Identify attack vector**
2. **Create test payload**
3. **Test against WAF**
4. **Verify blocking**
5. **Document results**

Example custom test:

```bash
# Test for new attack pattern
test_custom_attack() {
    local payload="<svg onload=alert(1)>"
    local url="http://localhost:8000/vulnerabilities/xss_r/?name=$payload"
    
    response=$(curl -s -o /dev/null -w "%{http_code}" "$url")
    
    if [ "$response" = "403" ]; then
        echo "✓ Custom SVG XSS blocked"
    else
        echo "✗ Custom SVG XSS not blocked"
    fi
}
```

## 🎯 Penetration Testing Integration

### With OWASP ZAP

1. **Configure ZAP** to proxy through http://localhost:8000
2. **Run spider** to crawl DVWA
3. **Execute active scan** with payloads
4. **Compare results** with direct access

### With Nikto

```bash
# Scan via WAF
nikto -h http://localhost:8000

# Scan direct
nikto -h http://localhost:80

# Compare results
```

## 📈 Reporting

### Generate Test Report

```bash
#!/bin/bash
echo "# WAF Testing Report" > test_report.md
echo "Generated: $(date)" >> test_report.md
echo "" >> test_report.md

# Run tests and append results
./test-waf.sh >> test_report.md

echo "" >> test_report.md
echo "## Log Analysis" >> test_report.md
echo "\`\`\`" >> test_report.md
docker-compose logs modsecurity-waf | grep -i "denied" | tail -10 >> test_report.md
echo "\`\`\`" >> test_report.md
```

## 🔄 Continuous Testing

Set up automated testing with cron:

```bash
# Add to crontab
0 */6 * * * /path/to/test-waf.sh >> /var/log/waf-tests.log 2>&1
```

This comprehensive testing guide ensures your WAF is properly protecting DVWA and can help identify any configuration issues or bypass attempts.
