# DVWA with ModSecurity WAF Protection

A containerized setup for running DVWA (Damn Vulnerable Web Application) protected by ModSecurity WAF using Docker Compose.

## 🎯 Project Overview

This project demonstrates how to protect a vulnerable web application (DVWA) using ModSecurity WAF (Web Application Firewall) with custom rules. It's perfect for:

- Security testing and learning
- WAF rule development and testing
- Penetration testing practice
- Security awareness training

## 🏗️ Architecture

```
Internet → ModSecurity WAF (Port 8000) → DVWA (Port 80)
```

- **ModSecurity WAF**: Acts as a reverse proxy with WAF capabilities
- **DVWA**: Vulnerable web application for testing
- **Custom Network**: Isolated Docker network for secure communication

## 📋 Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+
- At least 2GB RAM available

## 🚀 Quick Start

1. **Clone the repository:**
   ```bash
   git clone https://github.com/moeinfatehi/dvwa-modsecurity-waf.git
   cd dvwa-modsecurity-waf
   ```

2. **Start the services:**
   ```bash
   docker compose up -d
   ```

3. **Wait for containers to be ready:**
   ```bash
   docker compose ps
   ```

4. **Access the applications:**
   - **DVWA via WAF**: http://localhost:8000
   - **DVWA Direct**: http://localhost:80 (for comparison)

## 🔧 Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `BACKEND` | `http://dvwa-app:80` | Backend server URL |
| `MODSEC_RULE_ENGINE` | `On` | ModSecurity rule engine (On/Off/DetectionOnly) |
| `MODSEC_AUDIT_ENGINE` | `On` | Enable audit logging |
| `MODSEC_AUDIT_LOG_PARTS` | `ABDEFHIJZ` | Audit log parts to include |
| `PARANOIA` | `1` | Paranoia level (1-4, higher = more strict) |

### Custom Rules

The WAF includes custom rules for common vulnerabilities:

- **SQL Injection** (Rule ID: 1001)
- **Cross-Site Scripting (XSS)** (Rule ID: 1002)
- **Command Injection** (Rule ID: 1003)
- **Path Traversal** (Rule ID: 1004)

Rules are defined in `modsec-custom.conf` and can be customized as needed.

## 🧪 Testing WAF Protection

### DVWA Login
1. Access DVWA via WAF: http://localhost:8000
2. Default credentials: `admin/password`
3. Go to DVWA Security and set to "Low"

### SQL Injection Test
```bash
# This should be blocked by the WAF
curl "http://localhost:8000/vulnerabilities/sqli/?id=1' OR '1'='1&Submit=Submit"

# Compare with direct access (should work)
curl "http://localhost:80/vulnerabilities/sqli/?id=1' OR '1'='1&Submit=Submit"
```

### XSS Test
```bash
# This should be blocked by the WAF
curl "http://localhost:8000/vulnerabilities/xss_r/?name=<script>alert(1)</script>"
```

### Command Injection Test
```bash
# This should be blocked by the WAF
curl "http://localhost:8000/vulnerabilities/exec/?ip=127.0.0.1;cat /etc/passwd&Submit=Submit"
```

### Path Traversal Test
```bash
# This should be blocked by the WAF
curl "http://localhost:8000/vulnerabilities/fi/?page=../../../etc/passwd"
```

## 📊 Monitoring and Logs

### View WAF Logs
```bash
# Real-time logs
docker compose logs -f modsecurity-waf

# Blocked requests only
docker compose logs modsecurity-waf | grep -i denied

# ModSecurity specific logs
docker compose logs modsecurity-waf | grep -i modsecurity
```

### Access Audit Logs
```bash
# Access the container
docker compose exec modsecurity-waf bash

# View audit logs
tail -f /var/log/modsec_audit.log
```

## 🛠️ Development and Customization

### Adding Custom Rules

1. Edit `modsec-custom.conf` to add new rules
2. Restart the WAF container:
   ```bash
   docker compose restart modsecurity-waf
   ```

### Adjusting Paranoia Level

Edit `docker-compose.yml` and change the `PARANOIA` environment variable:
```yaml
environment:
  - PARANOIA=2  # Increase for stricter protection
```

### Creating Custom Rule Files

1. Create rule files in the `custom-rules/` directory
2. They will be automatically loaded by ModSecurity

## 📁 Project Structure

```
dvwa-modsecurity-waf/
├── docker-compose.yml          # Main orchestration file
├── modsec-custom.conf          # Custom ModSecurity rules
├── custom-rules/               # Directory for additional rule files
├── README.md                   # This file
└── .gitignore                  # Git ignore file
```

## 🔍 Troubleshooting

### Container Communication Issues
```bash
# Check if containers can communicate
docker compose exec modsecurity-waf curl -I http://dvwa-app:80
```

### Port Conflicts
```bash
# Check what's using the ports
sudo netstat -tulpn | grep :8000
sudo netstat -tulpn | grep :80
```

### Reset Everything
```bash
# Stop and remove everything
docker compose down -v
docker compose up -d
```

### Check Container Status
```bash
# View all containers
docker compose ps

# View specific container logs
docker compose logs dvwa
docker compose logs modsecurity-waf
```

## 🚨 Security Considerations

⚠️ **Important**: This setup is for testing and educational purposes only.

- DVWA is intentionally vulnerable - never expose it to the internet
- Use strong passwords in production environments
- Regularly update container images for security patches
- Consider network segmentation for production deployments

## 📚 Learning Resources

- [ModSecurity Documentation](https://github.com/SpiderLabs/ModSecurity/wiki)
- [OWASP ModSecurity Core Rule Set](https://owasp.org/www-project-modsecurity-core-rule-set/)
- [DVWA Documentation](https://github.com/digininja/DVWA)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable
5. Submit a pull request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- [OWASP ModSecurity Core Rule Set](https://owasp.org/www-project-modsecurity-core-rule-set/)
- [DVWA Project](https://github.com/digininja/DVWA)
- [ModSecurity Community](https://github.com/SpiderLabs/ModSecurity)

## 📞 Support

If you encounter issues or have questions:

1. Check the [Troubleshooting](#-troubleshooting) section
2. Review container logs: `docker compose logs`
3. Open an issue on GitHub
4. Check the original troubleshooting guide in the repository

---

**Happy Security Testing! 🔒**
