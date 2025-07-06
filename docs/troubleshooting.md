# Troubleshooting Guide

## Common Issues and Solutions

### 🔧 Port Mapping Issues

**Problem**: Cannot access WAF on port 8000
**Solution**: Ensure you're using `8000:8080` not `8000:80`

```bash
# Check if port 8000 is being used
sudo netstat -tulpn | grep :8000

# Verify Docker port mapping
docker-compose ps
```

**Problem**: Port 8000 already in use
**Solution**: Stop the conflicting service or change the port

```bash
# Find what's using port 8000
sudo lsof -i :8000

# Kill the process (replace PID with actual process ID)
sudo kill -9 PID

# Or change the port in docker-compose.yml
ports:
  - "8001:8080"  # Use different external port
```

### 🌐 Container Communication Issues

**Problem**: WAF cannot reach DVWA
**Solution**: Check network connectivity

```bash
# Test container communication
docker-compose exec modsecurity-waf curl -I http://dvwa-app:80

# Check network configuration
docker network inspect dvwa-modsecurity-waf_waf-network

# Verify containers are on the same network
docker-compose ps
```

**Problem**: Backend connection errors
**Solution**: Verify backend URL configuration

```bash
# Check environment variables
docker-compose exec modsecurity-waf env | grep BACKEND

# Should show: BACKEND=http://dvwa-app:80
```

### 📁 Configuration File Issues

**Problem**: Custom rules not loading
**Solution**: Check volume mount and file permissions

```bash
# Verify file exists
ls -la modsec-custom.conf

# Check file permissions
chmod 644 modsec-custom.conf

# Verify volume mount
docker-compose exec modsecurity-waf ls -la /etc/modsecurity.d/

# Check if custom config is loaded
docker-compose exec modsecurity-waf cat /etc/modsecurity.d/modsec-custom.conf
```

**Problem**: Custom rules directory not found
**Solution**: Create the directory and restart

```bash
# Create custom rules directory
mkdir -p custom-rules

# Add placeholder file
touch custom-rules/.gitkeep

# Restart containers
docker-compose restart modsecurity-waf
```

### 📊 Service Health Issues

**Problem**: Containers failing health checks
**Solution**: Check service status and logs

```bash
# Check container health
docker-compose ps

# View detailed health status
docker inspect modsecurity-waf | grep -A 10 Health

# Check logs for errors
docker-compose logs modsecurity-waf
docker-compose logs dvwa
```

**Problem**: DVWA not responding
**Solution**: Check MySQL and web service

```bash
# Check DVWA logs
docker-compose logs dvwa

# Access DVWA container
docker-compose exec dvwa bash

# Check MySQL connection inside container
mysql -u dvwa -p dvwa
```

### 🔍 WAF Not Blocking Attacks

**Problem**: Attacks passing through WAF
**Solution**: Verify rule engine and configuration

```bash
# Check ModSecurity status
docker-compose exec modsecurity-waf grep -r "SecRuleEngine" /etc/modsecurity.d/

# Verify paranoia level
docker-compose exec modsecurity-waf env | grep PARANOIA

# Check if rules are loaded
docker-compose logs modsecurity-waf | grep -i "rule"
```

**Problem**: False positives blocking legitimate requests
**Solution**: Adjust paranoia level or disable specific rules

```bash
# Lower paranoia level in docker-compose.yml
environment:
  - PARANOIA=1  # Lower from 2 or higher

# Or disable specific rules by adding to modsec-custom.conf
SecRuleRemoveById 1001
```

## 🐛 Debugging Commands

### Basic Diagnostics

```bash
# Check all container status
docker-compose ps

# View real-time logs
docker-compose logs -f

# Check specific service logs
docker-compose logs modsecurity-waf
docker-compose logs dvwa

# Check container resource usage
docker stats
```

### Network Debugging

```bash
# Test WAF accessibility
curl -I http://localhost:8000

# Test DVWA direct access
curl -I http://localhost:80

# Test from inside WAF container
docker-compose exec modsecurity-waf curl -I http://dvwa-app:80

# Check DNS resolution
docker-compose exec modsecurity-waf nslookup dvwa-app
```

### ModSecurity Specific

```bash
# Check ModSecurity version
docker-compose exec modsecurity-waf nginx -V 2>&1 | grep -i modsecurity

# View ModSecurity configuration
docker-compose exec modsecurity-waf cat /etc/modsecurity.d/modsecurity.conf

# Check audit log
docker-compose exec modsecurity-waf tail -f /var/log/modsec_audit.log

# Search for blocked requests
docker-compose logs modsecurity-waf | grep -i "denied\|blocked"
```

### File System Debugging

```bash
# Check volume mounts
docker-compose exec modsecurity-waf ls -la /etc/modsecurity.d/

# Verify custom config exists
docker-compose exec modsecurity-waf cat /etc/modsecurity.d/modsec-custom.conf

# Check nginx configuration
docker-compose exec modsecurity-waf nginx -t
```

## 🔄 Common Solutions

### Complete Reset

```bash
# Stop all containers
docker-compose down

# Remove volumes (careful - this removes data!)
docker-compose down -v

# Remove images (forces fresh download)
docker-compose down --rmi all

# Start fresh
docker-compose up -d
```

### Restart Individual Services

```bash
# Restart only WAF
docker-compose restart modsecurity-waf

# Restart only DVWA
docker-compose restart dvwa

# Restart with fresh build
docker-compose up -d --force-recreate modsecurity-waf
```

### Configuration Reload

```bash
# Reload nginx configuration
docker-compose exec modsecurity-waf nginx -s reload

# Restart with new config
docker-compose restart modsecurity-waf
```

## 📋 Diagnostic Checklist

When troubleshooting, check these items in order:

1. **Container Status**: Are all containers running?
   ```bash
   docker-compose ps
   ```

2. **Port Accessibility**: Can you reach the services?
   ```bash
   curl -I http://localhost:8000
   curl -I http://localhost:80
   ```

3. **Network Connectivity**: Can containers communicate?
   ```bash
   docker-compose exec modsecurity-waf ping dvwa-app
   ```

4. **Configuration Files**: Are configs loaded correctly?
   ```bash
   docker-compose exec modsecurity-waf cat /etc/modsecurity.d/modsec-custom.conf
   ```

5. **Logs**: Any errors in the logs?
   ```bash
   docker-compose logs --tail=50
   ```

6. **Resource Usage**: Sufficient resources available?
   ```bash
   docker stats
   ```

## 🆘 Emergency Recovery

If everything fails:

```bash
# Nuclear option - remove everything
docker-compose down -v --rmi all
docker system prune -a --force

# Start fresh
docker-compose up -d

# Wait for services to be ready
sleep 30
docker-compose ps
```

## 📞 Getting Help

If you're still having issues:

1. **Check the logs** first: `docker-compose logs`
2. **Search for similar issues** in the GitHub repository
3. **Create a detailed issue** with:
   - Your `docker-compose.yml` configuration
   - Output of `docker-compose ps`
   - Relevant log excerpts
   - Steps to reproduce the problem

## 🔧 Performance Tuning

If containers are slow or resource-constrained:

```bash
# Increase memory limits in docker-compose.yml
services:
  modsecurity-waf:
    mem_limit: 512m
  dvwa:
    mem_limit: 256m

# Monitor resource usage
docker stats
```

Remember: Most issues are related to port mapping, network connectivity, or configuration file problems. Check these first before diving into complex debugging!
