# Production Deployment Guide

This guide covers how to deploy and run your Jenkins CI/CD Java application in a production environment.

## Table of Contents
1. [Pre-Production Checklist](#pre-production-checklist)
2. [Environment Setup](#environment-setup)
3. [Jenkins Server Setup](#jenkins-server-setup)
4. [Application Deployment](#application-deployment)
5. [Monitoring & Logging](#monitoring--logging)
6. [Security Hardening](#security-hardening)
7. [Troubleshooting](#troubleshooting)

---

## Pre-Production Checklist

- [ ] Code review completed
- [ ] All tests passing (100% coverage recommended)
- [ ] Security scan completed
- [ ] Load testing performed
- [ ] Backup strategy in place
- [ ] Rollback plan documented
- [ ] SSL/TLS certificates ready
- [ ] Environment variables configured
- [ ] Database migrations tested
- [ ] Documentation updated

---

## Environment Setup

### 1. Production Server Requirements

**Hardware:**
```
CPU: 4+ cores
RAM: 8GB+ (16GB+ recommended)
Storage: 50GB+ SSD
Network: 1Gbps connection
```

**Operating System:**
```bash
# Ubuntu 20.04 LTS (Recommended)
# or CentOS 7+
# or Amazon Linux 2
```

### 2. Install Required Tools

```bash
#!/bin/bash
# Update system
sudo apt-get update
sudo apt-get upgrade -y

# Install Java 11
sudo apt-get install -y openjdk-11-jdk-headless
java -version

# Install Maven
sudo apt-get install -y maven
mvn --version

# Install Git
sudo apt-get install -y git

# Install Docker (Optional - for containerized deployment)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Jenkins
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install -y jenkins

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

---

## Jenkins Server Setup

### 1. Access Jenkins

```
URL: http://your-server-ip:8080
Initial Admin Password:
  sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### 2. Jenkins Configuration

#### Step 1: Install Recommended Plugins
- Pipeline
- Git
- Maven Integration
- Blue Ocean (for better visualization)
- Email Extension
- Performance Plugin
- SonarQube Scanner (for code quality)

#### Step 2: Configure System Settings

Go to **Manage Jenkins** → **Configure System**

```properties
# Java Path
Java Home: /usr/lib/jvm/java-11-openjdk-amd64

# Maven
Maven Home: /usr/share/maven

# Git
Git executable: /usr/bin/git

# Email Notifications
SMTP Server: smtp.gmail.com
SMTP Port: 587
Use SSL: true
```

#### Step 3: Create GitHub Credentials

1. Go to **Manage Jenkins** → **Manage Credentials**
2. Click **Global** → **Add Credentials**
3. Select **SSH Key** or **Username with password**
4. Paste your GitHub SSH key or PAT token

```bash
# Generate SSH key
ssh-keygen -t rsa -b 4096 -f ~/.ssh/jenkins_github
# Copy public key to GitHub Settings → SSH Keys
```

### 4. Create Production Pipeline Job

```groovy
// Jenkins Declarative Pipeline Configuration
pipeline {
    agent {
        label 'production'  // Use specific agents for production
    }

    options {
        timestamps()
        timeout(time: 2, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()  // Prevent concurrent builds
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['staging', 'production'],
            description: 'Deployment environment'
        )
        string(
            name: 'VERSION',
            defaultValue: '1.0.0',
            description: 'Application version'
        )
    }

    environment {
        APP_NAME = 'jenkins-app'
        REGISTRY = 'docker.io'  // Docker Hub or your registry
        REGISTRY_CREDENTIALS = credentials('docker-registry-creds')
        SONAR_TOKEN = credentials('sonarqube-token')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
                junit 'target/surefire-reports/*.xml'
            }
        }

        stage('Code Quality') {
            steps {
                sh '''
                    mvn sonar:sonar \
                        -Dsonar.projectKey=jenkins-app \
                        -Dsonar.host.url=http://sonarqube-server:9000 \
                        -Dsonar.login=${SONAR_TOKEN}
                '''
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh '''
                        docker build -t ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER} .
                        docker tag ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER} ${REGISTRY}/${APP_NAME}:latest
                    '''
                }
            }
        }

        stage('Push to Registry') {
            when {
                branch 'main'
            }
            steps {
                script {
                    sh '''
                        echo ${REGISTRY_CREDENTIALS_PSW} | docker login -u ${REGISTRY_CREDENTIALS_USR} --password-stdin
                        docker push ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
                        docker push ${REGISTRY}/${APP_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                script {
                    if (params.ENVIRONMENT == 'staging') {
                        sh '''
                            ssh -i ~/.ssh/prod-key ubuntu@staging-server << EOF
                            cd /opt/jenkins-app
                            docker pull ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
                            docker-compose down
                            docker-compose up -d
                            EOF
                        '''
                    }
                }
            }
        }

        stage('Smoke Tests') {
            steps {
                sh '''
                    sleep 10
                    curl -f http://localhost:8080/health || exit 1
                '''
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input 'Deploy to Production?'
                }
                script {
                    sh '''
                        ssh -i ~/.ssh/prod-key ubuntu@prod-server << EOF
                        cd /opt/jenkins-app
                        docker pull ${REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
                        docker-compose -f docker-compose.prod.yml down
                        docker-compose -f docker-compose.prod.yml up -d
                        EOF
                    '''
                }
            }
        }

        stage('Post-Deployment Verification') {
            steps {
                sh '''
                    sleep 15
                    curl -f http://production-server:8080/health || exit 1
                    echo "Production deployment verified!"
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            sh '''
                echo "Build successful! Version: ${BUILD_NUMBER}"
                # Send Slack notification
                curl -X POST ${SLACK_WEBHOOK} \
                    -H 'Content-Type: application/json' \
                    -d '{"text":"✓ Deployment successful! Build #${BUILD_NUMBER}"}'
            '''
        }
        failure {
            sh '''
                # Send Slack notification
                curl -X POST ${SLACK_WEBHOOK} \
                    -H 'Content-Type: application/json' \
                    -d '{"text":"✗ Deployment failed! Build #${BUILD_NUMBER}"}'
            '''
        }
    }
}
```

---

## Application Deployment

### Option 1: Direct JAR Deployment

```bash
# Create application directory
sudo mkdir -p /opt/jenkins-app
sudo chown -R jenkins:jenkins /opt/jenkins-app

# Copy JAR file
sudo cp target/jenkins-app-1.0.0.jar /opt/jenkins-app/

# Create systemd service
sudo tee /etc/systemd/system/jenkins-app.service > /dev/null <<EOF
[Unit]
Description=Jenkins Java Application
After=network.target

[Service]
Type=simple
User=jenkins
WorkingDirectory=/opt/jenkins-app
ExecStart=/usr/bin/java -Xmx512m -Xms256m -jar jenkins-app-1.0.0.jar
Restart=always
RestartSec=10
StandardOutput=append:/var/log/jenkins-app/app.log
StandardError=append:/var/log/jenkins-app/error.log

[Install]
WantedBy=multi-user.target
EOF

# Enable and start service
sudo mkdir -p /var/log/jenkins-app
sudo chown jenkins:jenkins /var/log/jenkins-app
sudo systemctl daemon-reload
sudo systemctl enable jenkins-app
sudo systemctl start jenkins-app

# Check status
sudo systemctl status jenkins-app
```

### Option 2: Docker Deployment

#### Dockerfile

```dockerfile
FROM openjdk:11-jre-slim

WORKDIR /app

# Copy built JAR
COPY target/jenkins-app-*.jar app.jar

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Run application
ENTRYPOINT ["java", "-Xmx512m", "-Xms256m", "-jar", "app.jar"]
EXPOSE 8080
```

#### Docker Compose (Staging)

```yaml
# docker-compose.yml
version: '3.8'

services:
  jenkins-app:
    image: docker.io/your-username/jenkins-app:latest
    container_name: jenkins-app
    ports:
      - "8080:8080"
    environment:
      - JAVA_OPTS=-Xmx512m -Xms256m
      - ENVIRONMENT=staging
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped
    networks:
      - jenkins-network

  # Optional: Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    depends_on:
      - jenkins-app
    networks:
      - jenkins-network

networks:
  jenkins-network:
    driver: bridge
```

#### Docker Compose (Production)

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  jenkins-app:
    image: docker.io/your-username/jenkins-app:${BUILD_NUMBER}
    container_name: jenkins-app
    ports:
      - "8080:8080"
    environment:
      - JAVA_OPTS=-Xmx2g -Xms1g
      - ENVIRONMENT=production
    volumes:
      - jenkins-app-logs:/app/logs
      - jenkins-app-data:/app/data
    restart: always
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
    networks:
      - jenkins-network
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - jenkins-app
    restart: always
    networks:
      - jenkins-network

volumes:
  jenkins-app-logs:
  jenkins-app-data:

networks:
  jenkins-network:
    driver: bridge
```

#### Nginx Configuration

```nginx
# nginx.conf
upstream jenkins_app {
    server jenkins-app:8080;
}

server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    # SSL Certificates
    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json;

    # Reverse proxy
    location / {
        proxy_pass http://jenkins_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 90;
    }

    # Health check endpoint
    location /health {
        proxy_pass http://jenkins_app;
        access_log off;
    }
}
```

### Deployment Steps

```bash
# 1. Build locally
mvn clean package

# 2. Create Docker image
docker build -t jenkins-app:1.0.0 .

# 3. Tag for registry
docker tag jenkins-app:1.0.0 docker.io/your-username/jenkins-app:1.0.0

# 4. Push to registry
docker push docker.io/your-username/jenkins-app:1.0.0

# 5. SSH into production server
ssh -i ~/.ssh/prod-key ubuntu@prod-server

# 6. Pull latest image
docker pull docker.io/your-username/jenkins-app:1.0.0

# 7. Start containers
docker-compose -f docker-compose.prod.yml up -d

# 8. Verify deployment
docker-compose ps
docker logs jenkins-app
curl http://localhost:8080/health
```

---

## Monitoring & Logging

### 1. Application Logging

```bash
# View logs
sudo journalctl -u jenkins-app -f

# Or for Docker
docker logs -f jenkins-app

# Centralized logging with ELK Stack
# Install Filebeat on server:
curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.14.0-linux-x86_64.tar.gz
tar xzvf filebeat-7.14.0-linux-x86_64.tar.gz
```

### 2. Monitoring Tools

**Install Prometheus + Grafana:**

```bash
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /etc/prometheus:/etc/prometheus \
  prom/prometheus

docker run -d \
  --name grafana \
  -p 3000:3000 \
  grafana/grafana
```

### 3. Health Checks

```java
// Add to your Main.java
@RestController
public class HealthController {
    
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        return ResponseEntity.ok(Map.of(
            "status", "UP",
            "timestamp", LocalDateTime.now().toString()
        ));
    }
}
```

---

## Security Hardening

### 1. Network Security

```bash
# Configure firewall
sudo ufw enable
sudo ufw allow 22/tcp       # SSH
sudo ufw allow 80/tcp       # HTTP
sudo ufw allow 443/tcp      # HTTPS
sudo ufw deny 8080/tcp      # Jenkins internal only

# Use VPN or SSH tunneling for Jenkins access
ssh -L 8080:localhost:8080 ubuntu@prod-server
```

### 2. Jenkins Security

```groovy
// In Jenkins Configuration
- Enable CSRF protection
- Use API tokens instead of passwords
- Configure webhook secrets
- Enable Jenkins CLI authentication
- Use SSH keys for Git access
```

### 3. Environment Variables

```bash
# Create .env file
ENVIRONMENT=production
APP_PORT=8080
LOG_LEVEL=INFO
DATABASE_URL=jdbc:mysql://db-server:3306/jenkins_app
DB_USER=${DB_USER}
DB_PASSWORD=${DB_PASSWORD}
JWT_SECRET=${JWT_SECRET}

# Make sure to use Jenkins credentials for sensitive data
```

### 4. SSL/TLS Certificate

```bash
# Using Let's Encrypt
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot certonly --standalone -d your-domain.com
sudo certbot renew --dry-run  # Test renewal
```

---

## Troubleshooting

### Application Won't Start

```bash
# Check logs
sudo journalctl -u jenkins-app -n 50

# Check port usage
sudo netstat -tlnp | grep 8080

# Check Java process
ps aux | grep java

# Kill and restart
sudo systemctl restart jenkins-app
```

### Docker Issues

```bash
# Check container status
docker ps -a
docker logs jenkins-app

# Restart container
docker restart jenkins-app

# Rebuild and redeploy
docker-compose down
docker-compose up -d --build
```

### Performance Issues

```bash
# Monitor resource usage
docker stats jenkins-app

# Adjust JVM heap size in docker-compose.prod.yml
JAVA_OPTS=-Xmx2g -Xms1g  # Allocate more memory if needed

# Check disk space
df -h
du -sh /var/lib/docker
```

### Network Connectivity

```bash
# Test application endpoint
curl -v http://localhost:8080/health

# Test DNS
nslookup your-domain.com

# Test SSL
curl -v https://your-domain.com

# Check firewall
sudo iptables -L -n
```

---

## Rollback Procedure

```bash
# Keep previous versions tagged
docker tag docker.io/your-username/jenkins-app:1.0.0 docker.io/your-username/jenkins-app:1.0.0-backup

# To rollback
docker-compose down
docker pull docker.io/your-username/jenkins-app:0.9.9
docker-compose -f docker-compose.prod.yml up -d

# Verify
curl http://localhost:8080/health
```

---

## Production Checklist - Final

- [ ] Application tested in staging environment
- [ ] Database migrations completed and backed up
- [ ] SSL/TLS certificates installed
- [ ] Backup and recovery plan tested
- [ ] Monitoring and alerting configured
- [ ] Team trained on deployment process
- [ ] Incident response plan documented
- [ ] All credentials stored securely in Jenkins
- [ ] Health checks implemented and passing
- [ ] Load testing completed successfully

**Support & Documentation:**
- Jenkins: https://www.jenkins.io/
- Docker: https://docs.docker.com/
- SonarQube: https://docs.sonarqube.org/
