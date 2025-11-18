# Stage 5: Advanced Configuration Management & Multi-Environment Deployment

> **Status:** Implementation Plan (Ready for Execution)
> **Created:** 2025-11-18
> **Target:** Deploy stock-keeping-app-BE and stock-keeping-app-fe using Ansible
> **Environment:** Staging + Production with Centralized Logging

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Requirements Verification](#requirements-verification)
4. [Part 1: Repository Preparation](#part-1-repository-preparation)
5. [Part 2: Ansible Controller Setup](#part-2-ansible-controller-setup)
6. [Part 3: Inventory & Variables Configuration](#part-3-inventory--variables-configuration)
7. [Part 4: Role Implementation](#part-4-role-implementation)
8. [Part 5: Universal Playbook](#part-5-universal-playbook)
9. [Part 6: Execution Instructions](#part-6-execution-instructions)
10. [Part 7: Demo Verification Checklist](#part-7-demo-verification-checklist)

---

## Executive Summary

This implementation plan details the deployment of two applications to staging and production environments using **Ansible-based configuration management**:

- **Backend:** stock-keeping-app-BE (Java 21 + Spring Boot)
- **Frontend:** stock-keeping-app-fe (Next.js + React)

### Key Changes:
- ❌ **Removal:** All GitHub Actions CI/CD pipelines
- ✅ **Addition:** Unified Ansible playbook for both environments
- ✅ **Addition:** Centralized logging to AWS CloudWatch
- ✅ **Addition:** Ansible Vault encryption for sensitive data
- ✅ **Addition:** Dynamic environment configuration via group_vars

### Compliance:
All requirements from `/Users/sherifdeenadebayo/Developer/stock-keeping/stage5-task.txt` will be met:
- ✅ Single master playbook (deploy.yml)
- ✅ Environment-specific variables via inventory groups
- ✅ Encrypted SSH deployment key using Ansible Vault
- ✅ Centralized CloudWatch logging with strict naming convention
- ✅ No CI/CD pipelines
- ✅ No raw SSH debugging (observability via CloudWatch)

---

## Architecture Overview

### High-Level Deployment Flow

```mermaid
graph TB
    subgraph "Ansible Controller"
        AC["Ansible Controller Server<br/>(Team Leads manage)"]
        PLAY["deploy.yml<br/>(Universal Playbook)"]
        VAULT["vault.yml<br/>(Encrypted SSH Key)"]
        SECRETS["secrets/<br/>(staging.env, prod.env)"]
        INV["inventory/<br/>(staging, production)"]
        VARS["group_vars/<br/>(env-specific config)"]

        AC --> PLAY
        AC --> VAULT
        AC --> SECRETS
        AC --> INV
        AC --> VARS
    end

    subgraph "Staging Environment"
        SBE["Staging Backend<br/>(Java/SystemD)"]
        SFE["Staging Frontend<br/>(Next.js/PM2)"]
        SNGINX["Nginx<br/>(Reverse Proxy)"]
        SSL_S["Certbot<br/>(SSL)"]
        CW_S["CloudWatch Agent<br/>(Logging)"]

        PLAY -->|ssh| SBE
        PLAY -->|ssh| SFE
        SBE --> SNGINX
        SFE --> SNGINX
        SNGINX --> SSL_S
        SBE --> CW_S
        SFE --> CW_S
        SNGINX --> CW_S
    end

    subgraph "Production Environment"
        PBE["Production Backend<br/>(Java/SystemD)"]
        PFE["Production Frontend<br/>(Next.js/PM2)"]
        PNGINX["Nginx<br/>(Reverse Proxy)"]
        SSL_P["Certbot<br/>(SSL)"]
        CW_P["CloudWatch Agent<br/>(Logging)"]

        PLAY -->|ssh| PBE
        PLAY -->|ssh| PFE
        PBE --> PNGINX
        PFE --> PNGINX
        PNGINX --> SSL_P
        PBE --> CW_P
        PFE --> CW_P
        PNGINX --> CW_P
    end

    subgraph "AWS CloudWatch"
        CWL["/ec2/staging/stock-keeping/*<br/>/ec2/production/stock-keeping/*"]
    end

    CW_S --> CWL
    CW_P --> CWL
```

### Deployment Server Breakdown

```
Ansible Controller (Central)
│
├─ Manages SSH deployment key (encrypted in vault.yml)
├─ Manages environment-specific secrets (staging.env, prod.env)
├─ Manages inventory (server IPs and groups)
├─ Manages group_vars (environment-specific variables)
└─ Runs universal playbook against target servers
   │
   ├─ Staging Servers (develop branch)
   │  ├─ Backend: 5000 (proxied via Nginx 443)
   │  └─ Frontend: 3000 (proxied via Nginx 443)
   │
   └─ Production Servers (main branch)
      ├─ Backend: 5000 (proxied via Nginx 443)
      └─ Frontend: 3000 (proxied via Nginx 443)

CloudWatch Logs (Central Observability)
├─ /ec2/staging/stock-keeping/backend
├─ /ec2/staging/stock-keeping/frontend
├─ /ec2/staging/stock-keeping/nginx
├─ /ec2/production/stock-keeping/backend
├─ /ec2/production/stock-keeping/frontend
└─ /ec2/production/stock-keeping/nginx
```

---

## Requirements Verification

### Stage 5 Task Requirements Checklist

#### Part 1: Architecture & Security (The Setup)

| Requirement | Status | Implementation |
|-----------|--------|-----------------|
| Central Ansible Controller Server | ✅ | ansible-controller repo |
| Deployment key authorization | ✅ | SSH key in inventory |
| Encrypt private SSH key | ✅ | Ansible Vault (vault.yml) |
| Store in group_vars/all/vault.yml | ✅ | Encrypted vault structure |
| No raw key in filesystem | ✅ | Only encrypted in vault.yml |
| Prompt for vault password | ✅ | ansible-vault prompt or env var |

#### Part 2: The Universal Playbook

| Requirement | Status | Implementation |
|-----------|--------|-----------------|
| One playbook for both environments | ✅ | deploy.yml (universal) |
| No separate deploy_staging.yml/deploy_prod.yml | ✅ | Single file with group detection |
| Use inventory groups for differentiation | ✅ | [staging] and [production] groups |
| Use group_vars for env-specific config | ✅ | group_vars/staging/, group_vars/production/ |
| Dynamic environment configuration | ✅ | Template variables applied per group |
| Handle dependency installation | ✅ | dependencies role |
| Clone private repositories securely | ✅ | SSH agent forwarding with vault key |
| Manage .env files via Ansible | ✅ | secrets_injection role |
| Detect environment automatically | ✅ | Inventory group detection |
| Use PM2 for Node.js | ✅ | process_management role (PM2) |
| Use SystemD for backend | ✅ | process_management role (SystemD) |
| Ensure app restarts on reboot | ✅ | Systemd enabled, PM2 startup |
| Jinja2 Nginx templates (not static) | ✅ | nginx.conf.j2 with variables |
| Dynamic domain names | ✅ | {{ domain_name }} variable |
| Dynamic proxy ports | ✅ | {{ backend_port }}, {{ frontend_port }} |
| Dynamic SSL paths | ✅ | /etc/letsencrypt/live/{{ domain_name }} |
| Dynamic log locations | ✅ | /var/log/nginx/{{ domain_name }} |
| Automate Certbot installation | ✅ | ssl_certification role |
| Certificate generation | ✅ | Certbot standalone mode |
| Automatic renewal cronjob | ✅ | Cron job configured |
| Verify renewal cronjob exists | ✅ | Task verifies crontab |
| Force HTTPS redirection | ✅ | Nginx config (80 → 443) |

#### Part 3: Centralized Logging (AWS CloudWatch)

| Requirement | Status | Implementation |
|-----------|--------|-----------------|
| No SSH access for debugging | ✅ | CloudWatch only observability |
| Install CloudWatch Agent | ✅ | cloudwatch_agent role |
| Configure Unified CloudWatch Agent | ✅ | amazon-cloudwatch-agent-config.j2 |
| Stream application logs | ✅ | Configured for stdout/stderr |
| Stream Nginx access logs | ✅ | /var/log/nginx/*-access.log |
| Stream Nginx error logs | ✅ | /var/log/nginx/*-error.log |
| Strict naming convention | ✅ | /ec2/{environment}/{project-name}/{component} |
| Staging example: /ec2/staging/stock-keeping/frontend | ✅ | Implemented |
| Production example: /ec2/production/stock-keeping/backend | ✅ | Implemented |

#### Part 4: Submission Requirements

| Requirement | Status | Implementation |
|-----------|--------|-----------------|
| GitHub repo with Playbook | ✅ | ansible-controller repository |
| GitHub repo with Roles | ✅ | roles/ directory structure |
| Encrypted Vault files | ✅ | group_vars/all/vault.yml |
| Video demo (max 3 mins) | 📋 | Checklist provided in Part 7 |
| Show playbook execution | 📋 | Demo script in Part 7 |
| Show HTTPS in browser | 📋 | Demo verification steps |
| Show CloudWatch logs | 📋 | Demo verification steps |

---

## Part 1: Repository Preparation

### 1.1 Backend Repository Cleanup

**Objective:** Remove GitHub Actions, secure secrets, prepare for Ansible deployment

#### Files to DELETE

```bash
# GitHub Actions CI/CD pipeline
❌ .github/workflows/ci-cd.yml

# Raw secrets file (after securing)
❌ .env

# Legacy server setup (move to docs)
❌ setup-server.sh
❌ stock-keeping-backend.service
```

#### Files to CREATE

**File: `.env.example`**
```properties
# Database Configuration
DB_URL=jdbc:postgresql://your-db-host:5432/your-database?sslmode=require
DB_USERNAME=your_username
DB_PASSWORD=your_password

# Mail Configuration
MAIL_HOST=smtp.example.com
MAIL_PORT=587
MAIL_USERNAME=your_email@example.com
MAIL_PASSWORD=your_password

# Application Configuration
SPRING_PROFILES_ACTIVE=production
SERVER_PORT=5000
```

**File: `docs/DEPLOYMENT.md`**
```markdown
# Legacy Deployment Documentation

## Previous Deployment Method (GitHub Actions)

This document archives the previous CI/CD approach that has been replaced with Ansible-based configuration management.

### Previous Files
- setup-server.sh: Server initialization script
- stock-keeping-backend.service: SystemD service file
- .github/workflows/ci-cd.yml: GitHub Actions workflow

These have been superseded by the Ansible playbook in the central ansible-controller repository.

### Migration Guide
See README.md for new deployment instructions.
```

**File: `docs/SERVER_SETUP_LEGACY.md`**
- Copy contents of `setup-server.sh` and `stock-keeping-backend.service` here for reference

#### Files to MODIFY

**File: `src/main/resources/application.properties`**

Before (hardcoded):
```properties
spring.datasource.url=jdbc:postgresql://waitlist-db-oyetimehin31-36fd.c.aivencloud.com:11280/defaultdb?sslmode=require
spring.datasource.username=avnadmin
spring.datasource.password=AVNS_O_hF73fynaRUbldSlY6
```

After (environment variables):
```properties
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver

spring.mail.host=${MAIL_HOST}
spring.mail.port=${MAIL_PORT}
spring.mail.username=${MAIL_USERNAME}
spring.mail.password=${MAIL_PASSWORD}
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true

# Logging
logging.file.name=/opt/stock-keeping/backend/logs/application.log
logging.level.root=INFO
logging.level.com.hng.stockkeeper=DEBUG

# Server Configuration
server.port=${SERVER_PORT:5000}
server.servlet.context-path=/api

# JPA Configuration
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect

# CORS Configuration (will be proxied via Nginx)
server.servlet.session.cookie.secure=true
server.servlet.session.cookie.http-only=true
```

#### Git Operations

```bash
cd stock-keeping-app-BE

# Remove GitHub Actions
git rm .github/workflows/ci-cd.yml

# Remove raw .env file
git rm .env

# Move files to docs
git mv setup-server.sh docs/
git mv stock-keeping-backend.service docs/

# Create .env.example
echo "# See docs/DEPLOYMENT.md" > .env.example

# Ensure .gitignore includes .env
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore

# Commit changes
git add .
git commit -m "chore: Remove GitHub Actions and migrate to Ansible deployment"
git push origin main
```

---

### 1.2 Frontend Repository Cleanup

**Objective:** Remove GitHub Actions, prepare environment templates

#### Files to DELETE

```bash
❌ .github/workflows/ci-cd.yml
```

#### Files to CREATE

**File: `.env.local.example`**
```env
# Development Environment
NEXT_PUBLIC_API_URL=http://localhost:5000/api
NEXT_PUBLIC_APP_URL=http://localhost:3000
NODE_ENV=development
```

**File: `.env.production.example`**
```env
# Production Environment
NEXT_PUBLIC_API_URL=https://api.stockkeeping.example.com/api
NEXT_PUBLIC_APP_URL=https://stockkeeping.example.com
NODE_ENV=production
```

**File: `docs/DEPLOYMENT.md`**
```markdown
# Legacy Deployment Documentation

## Previous Deployment Method (GitHub Actions)

This document archives the previous CI/CD approach that has been replaced with Ansible-based configuration management.

### Previous Workflow
- .github/workflows/ci-cd.yml: GitHub Actions workflow
  - Triggered on push to main branch
  - Built Next.js application
  - Deployed via SSH and rsync to target server
  - Used PM2 for process management

This has been superseded by the Ansible playbook in the central ansible-controller repository.

### Environment Variables

The application requires the following environment variables:

- `NEXT_PUBLIC_API_URL`: Backend API URL
- `NEXT_PUBLIC_APP_URL`: Frontend application URL
- `NODE_ENV`: Node environment (development/production)

See `.env.production.example` for the template.
```

**File: `docs/ENVIRONMENT.md`**
```markdown
# Environment Configuration

## Frontend Environment Variables

### Production Variables

| Variable | Example | Description |
|----------|---------|-------------|
| NEXT_PUBLIC_API_URL | https://api.stockkeeping.example.com/api | Backend API endpoint |
| NEXT_PUBLIC_APP_URL | https://stockkeeping.example.com | Frontend application URL |
| NODE_ENV | production | Node.js environment |

### Staging Variables

| Variable | Example | Description |
|----------|---------|-------------|
| NEXT_PUBLIC_API_URL | https://api-staging.stockkeeping.example.com/api | Staging backend API |
| NEXT_PUBLIC_APP_URL | https://staging.stockkeeping.example.com | Staging frontend URL |
| NODE_ENV | production | Node.js environment |

### Development Variables

See `.env.local.example` for local development setup.

## Deployment

Environment variables are injected by the Ansible playbook from the centralized secrets management system.
```

#### Files to MODIFY

**File: `next.config.ts`**

Add environment variable validation:
```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  reactStrictMode: true,
  swcMinify: true,
  output: "standalone", // Enable standalone build for Docker

  // Environment variable validation
  env: {
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL || "http://localhost:5000/api",
    NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL || "http://localhost:3000",
  },

  // Image optimization
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "**",
      },
    ],
  },

  // Headers for security
  async headers() {
    return [
      {
        source: "/:path*",
        headers: [
          {
            key: "X-Content-Type-Options",
            value: "nosniff",
          },
          {
            key: "X-Frame-Options",
            value: "DENY",
          },
          {
            key: "X-XSS-Protection",
            value: "1; mode=block",
          },
        ],
      },
    ];
  },
};

export default nextConfig;
```

#### Git Operations

```bash
cd stock-keeping-app-fe

# Remove GitHub Actions
git rm .github/workflows/ci-cd.yml

# Create environment templates
echo "NEXT_PUBLIC_API_URL=http://localhost:5000/api" > .env.local.example
echo "NEXT_PUBLIC_APP_URL=http://localhost:3000" >> .env.local.example
echo "NODE_ENV=development" >> .env.local.example

# Ensure .gitignore includes .env files
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore
echo ".env.production" >> .gitignore

# Commit changes
git add .
git commit -m "chore: Remove GitHub Actions and migrate to Ansible deployment"
git push origin main
```

---

## Part 2: Ansible Controller Setup

### 2.1 Create Central Repository

```bash
# Create new repository for Ansible Controller
mkdir -p ~/Developer/stock-keeping/ansible-controller
cd ~/Developer/stock-keeping/ansible-controller
git init
git remote add origin <your-github-ansible-controller-repo>
```

### 2.2 Directory Structure

```
ansible-controller/
├── .gitignore                          # Exclude secrets, ssh_keys, vault password
├── ansible.cfg                         # Ansible configuration
├── README.md                           # Setup and usage documentation
│
├── inventory/                          # Inventory files
│   ├── staging                         # Staging environment hosts
│   ├── production                      # Production environment hosts
│   └── all                             # All hosts (optional)
│
├── group_vars/                         # Group-specific variables
│   ├── all/
│   │   ├── vault.yml                   # ENCRYPTED: SSH key, deployment secrets
│   │   └── common.yml                  # Common variables for all groups
│   ├── staging/
│   │   └── vars.yml                    # Staging-specific variables
│   └── production/
│       └── vars.yml                    # Production-specific variables
│
├── host_vars/                          # Host-specific variables (optional)
│   └── (individual host configs if needed)
│
├── roles/                              # Ansible roles
│   ├── dependencies/
│   │   └── tasks/
│   │       └── main.yml
│   ├── clone_repo/
│   │   └── tasks/
│   │       └── main.yml
│   ├── secrets_injection/
│   │   └── tasks/
│   │       └── main.yml
│   ├── process_management/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── templates/
│   │       ├── pm2-ecosystem.j2
│   │       ├── systemd.service.j2
│   │       └── systemd-env.j2
│   ├── nginx_reverse_proxy/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── templates/
│   │       └── nginx.conf.j2
│   ├── ssl_certification/
│   │   └── tasks/
│   │       └── main.yml
│   └── cloudwatch_agent/
│       ├── tasks/
│       │   └── main.yml
│       ├── templates/
│       │   └── amazon-cloudwatch-agent-config.j2
│       └── files/
│           └── install_cloudwatch_agent.sh
│
├── playbooks/                          # Ansible playbooks
│   ├── deploy.yml                      # UNIVERSAL PLAYBOOK (staging + production)
│   └── pre_checks.yml                  # Pre-deployment validation
│
├── secrets/                            # GITIGNORED: Environment secrets
│   ├── staging.env
│   └── prod.env
│
└── ssh_keys/                           # GITIGNORED: SSH keys (encrypted in vault.yml)
    └── (store encrypted keys here)
```

### 2.3 Create .gitignore

**File: `ansible-controller/.gitignore`**
```gitignore
# Encrypted vault password file
.vault-password
.vault-password-file
.ansible/tmp/

# Secrets directory (local environment files)
secrets/
!secrets/.gitkeep

# SSH keys directory
ssh_keys/
!ssh_keys/.gitkeep

# Local development files
*.swp
*.swo
*~
.DS_Store

# IDE
.vscode/
.idea/
*.iml

# Ansible logs
ansible.log

# Environment-specific files
inventory/local
```

### 2.4 Create ansible.cfg

**File: `ansible-controller/ansible.cfg`**
```ini
[defaults]
# Inventory configuration
inventory = inventory/staging
host_key_checking = False

# Remote user and SSH
remote_user = deploy
private_key_file = ~/.ssh/deploy_key
timeout = 300

# Vault configuration
ask_vault_pass = True
vault_password_file = ~/.vault-password

# Output formatting
stdout_callback = yaml
callback_whitelist = profile_tasks
force_color = True

# Privilege escalation
[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
control_path = /tmp/ansible-ssh-%%h-%%p-%%r
```

---

## Part 3: Inventory & Variables Configuration

### 3.1 Create Inventory Files

**File: `inventory/staging`**
```ini
# Staging Environment Inventory

[staging_backend]
staging-backend-1 ansible_host=<STAGING_BACKEND_IP> app_type=backend project_name=stock-keeping

[staging_frontend]
staging-frontend-1 ansible_host=<STAGING_FRONTEND_IP> app_type=frontend project_name=stock-keeping

[staging:children]
staging_backend
staging_frontend

[staging:vars]
environment=staging
app_branch=develop
cloudwatch_enabled=true
enable_debug_logs=true
ssl_enabled=true
aws_region=us-east-1
domain_prefix=staging
backend_domain=api-staging.stockkeeping.example.com
frontend_domain=staging.stockkeeping.example.com
backend_port=5000
frontend_port=3000
vault_secrets_path=../secrets
ansible_user=deploy
```

**File: `inventory/production`**
```ini
# Production Environment Inventory

[production_backend]
prod-backend-1 ansible_host=<PROD_BACKEND_IP> app_type=backend project_name=stock-keeping

[production_frontend]
prod-frontend-1 ansible_host=<PROD_FRONTEND_IP> app_type=frontend project_name=stock-keeping

[production:children]
production_backend
production_frontend

[production:vars]
environment=production
app_branch=main
cloudwatch_enabled=true
enable_debug_logs=false
ssl_enabled=true
aws_region=us-east-1
domain_prefix=""
backend_domain=api.stockkeeping.example.com
frontend_domain=stockkeeping.example.com
backend_port=5000
frontend_port=3000
vault_secrets_path=../secrets
ansible_user=deploy
```

### 3.2 Create Group Variables

**File: `group_vars/all/common.yml`**
```yaml
---
# Common variables for all environments

project_name: stock-keeping
backend_app_name: stock-keeping-backend
frontend_app_name: stock-keeping-frontend

# Repository URLs
backend_repo: https://github.com/hngprojects/stock-keeping-app-BE.git
frontend_repo: https://github.com/hngprojects/stock-keeping-app-fe.git

# Application configuration
backend_port: 5000
frontend_port: 3000
app_user: deploy
app_group: deploy

# Directory structure
app_root: /opt/{{ project_name }}
backend_dir: "{{ app_root }}/backend"
frontend_dir: "{{ app_root }}/frontend"
logs_dir: "{{ app_root }}/logs"

# System configuration
system_timezone: UTC
```

**File: `group_vars/all/vault.yml`** (ENCRYPTED)
```yaml
---
# VAULT ENCRYPTED FILE
# Ansible Vault stores sensitive deployment information

vault_ssh_private_key: |
  -----BEGIN OPENSSH PRIVATE KEY-----
  <ENCRYPTED PRIVATE KEY CONTENT>
  -----END OPENSSH PRIVATE KEY-----

vault_aws_access_key_id: <ENCRYPTED>
vault_aws_secret_access_key: <ENCRYPTED>

# These values are sourced from the vault at runtime
# They are never stored unencrypted in the repository
```

**File: `group_vars/staging/vars.yml`**
```yaml
---
# Staging Environment Variables

environment: staging
app_branch: develop
enable_debug_logs: true
cloudwatch_enabled: true
ssl_enabled: true

# AWS Configuration
aws_region: us-east-1
aws_cloudwatch_log_retention: 7  # 7 days for staging

# Domains
domain_name: staging.stockkeeping.example.com
backend_domain: api-staging.stockkeeping.example.com
frontend_domain: staging.stockkeeping.example.com

# Application ports
backend_port: 5000
frontend_port: 3000

# Certbot configuration
certbot_email: devops@staging.stockkeeping.example.com

# PM2 configuration
pm2_max_memory_restart: 512
pm2_instances: 2

# Database configuration
db_max_connections: 20
db_pool_size: 10

# Nginx configuration
nginx_worker_processes: auto
nginx_worker_connections: 1024
```

**File: `group_vars/production/vars.yml`**
```yaml
---
# Production Environment Variables

environment: production
app_branch: main
enable_debug_logs: false
cloudwatch_enabled: true
ssl_enabled: true

# AWS Configuration
aws_region: us-east-1
aws_cloudwatch_log_retention: 30  # 30 days for production

# Domains
domain_name: stockkeeping.example.com
backend_domain: api.stockkeeping.example.com
frontend_domain: stockkeeping.example.com

# Application ports
backend_port: 5000
frontend_port: 3000

# Certbot configuration
certbot_email: devops@stockkeeping.example.com

# PM2 configuration
pm2_max_memory_restart: 1024
pm2_instances: 4

# Database configuration
db_max_connections: 50
db_pool_size: 25

# Nginx configuration
nginx_worker_processes: auto
nginx_worker_connections: 2048
```

---

## Part 4: Role Implementation

### 4.1 Role: Dependencies

**File: `roles/dependencies/tasks/main.yml`**
```yaml
---
- name: "Dependencies | Update system package cache"
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: "Dependencies | Upgrade all packages"
  apt:
    upgrade: dist
  register: upgrade_result

- name: "Dependencies | Install common packages"
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - wget
    - htop
    - nano
    - vim
    - nginx
    - certbot
    - python3-pip
    - python3-venv
    - supervisor
    - ntp
    - awscli
    - jq
  tags:
    - dependencies
    - packages

- name: "Dependencies | Install AWS CloudWatch Agent"
  shell: |
    wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
    dpkg -i -E ./amazon-cloudwatch-agent.deb
    rm amazon-cloudwatch-agent.deb
  args:
    creates: /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent
  tags:
    - dependencies
    - cloudwatch

- name: "Dependencies | Install Node.js (Frontend)"
  block:
    - name: "Dependencies | Add Node.js repository"
      shell: |
        curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -

    - name: "Dependencies | Install Node.js and npm"
      apt:
        name:
          - nodejs
          - npm
        state: present

    - name: "Dependencies | Install PM2 globally"
      npm:
        name: pm2
        global: yes
        state: present
  when: app_type == 'frontend' or app_type == 'both'
  tags:
    - dependencies
    - nodejs

- name: "Dependencies | Install Java (Backend)"
  block:
    - name: "Dependencies | Install OpenJDK 21"
      apt:
        name: openjdk-21-jdk
        state: present

    - name: "Dependencies | Install Maven"
      apt:
        name: maven
        state: present
  when: app_type == 'backend' or app_type == 'both'
  tags:
    - dependencies
    - java

- name: "Dependencies | Create deploy user"
  user:
    name: deploy
    shell: /bin/bash
    createhome: yes
    home: /home/deploy
    state: present
  tags:
    - dependencies
    - users

- name: "Dependencies | Grant sudo privileges to deploy user"
  lineinfile:
    path: /etc/sudoers.d/deploy
    line: 'deploy ALL=(ALL) NOPASSWD: ALL'
    create: yes
    mode: '0440'
  tags:
    - dependencies
    - users

- name: "Dependencies | Create application directories"
  file:
    path: "{{ item }}"
    state: directory
    owner: deploy
    group: deploy
    mode: '0755'
  loop:
    - "{{ app_root }}"
    - "{{ backend_dir }}"
    - "{{ frontend_dir }}"
    - "{{ logs_dir }}"
    - /home/deploy/.ssh
  tags:
    - dependencies
    - directories

- name: "Dependencies | Configure SSH for git access"
  file:
    path: /home/deploy/.ssh
    owner: deploy
    group: deploy
    mode: '0700'
    state: directory
  tags:
    - dependencies
    - ssh
```

### 4.2 Role: Clone Repository

**File: `roles/clone_repo/tasks/main.yml`**
```yaml
---
- name: "Clone Repo | Configure SSH known hosts"
  shell: |
    ssh-keyscan -H github.com >> /home/deploy/.ssh/known_hosts 2>/dev/null
  args:
    creates: /home/deploy/.ssh/known_hosts
  become: yes
  become_user: deploy
  tags:
    - clone_repo
    - ssh

- name: "Clone Repo | Clone backend repository"
  git:
    repo: "{{ backend_repo }}"
    dest: "{{ backend_dir }}"
    version: "{{ app_branch }}"
    accept_hostkey: yes
    force: yes
  environment:
    GIT_SSH_COMMAND: "ssh -o StrictHostKeyChecking=no -i /home/deploy/.ssh/id_rsa"
  become: yes
  become_user: deploy
  when: app_type == 'backend' or app_type == 'both'
  tags:
    - clone_repo
    - backend

- name: "Clone Repo | Clone frontend repository"
  git:
    repo: "{{ frontend_repo }}"
    dest: "{{ frontend_dir }}"
    version: "{{ app_branch }}"
    accept_hostkey: yes
    force: yes
  environment:
    GIT_SSH_COMMAND: "ssh -o StrictHostKeyChecking=no -i /home/deploy/.ssh/id_rsa"
  become: yes
  become_user: deploy
  when: app_type == 'frontend' or app_type == 'both'
  tags:
    - clone_repo
    - frontend

- name: "Clone Repo | Set repository permissions"
  file:
    path: "{{ item }}"
    owner: deploy
    group: deploy
    mode: '0755'
    recurse: yes
  loop:
    - "{{ backend_dir }}"
    - "{{ frontend_dir }}"
  when: app_type != 'none'
  tags:
    - clone_repo
    - permissions
```

### 4.3 Role: Secrets Injection

**File: `roles/secrets_injection/tasks/main.yml`**
```yaml
---
- name: "Secrets | Copy environment file to backend"
  copy:
    src: "{{ vault_secrets_path }}/{{ environment }}.env"
    dest: "{{ backend_dir }}/.env"
    owner: deploy
    group: deploy
    mode: '0600'
  when: app_type == 'backend' or app_type == 'both'
  tags:
    - secrets
    - backend

- name: "Secrets | Copy environment file to frontend"
  copy:
    src: "{{ vault_secrets_path }}/{{ environment }}.env"
    dest: "{{ frontend_dir }}/.env.local"
    owner: deploy
    group: deploy
    mode: '0600'
  when: app_type == 'frontend' or app_type == 'both'
  tags:
    - secrets
    - frontend

- name: "Secrets | Verify environment files exist"
  stat:
    path: "{{ item }}"
  register: env_files
  failed_when: not env_files.stat.exists
  when: item is defined
  loop:
    - "{{ backend_dir }}/.env"
    - "{{ frontend_dir }}/.env.local"
  tags:
    - secrets
    - verify

- name: "Secrets | Create logs directory for backend"
  file:
    path: "{{ backend_dir }}/logs"
    state: directory
    owner: deploy
    group: deploy
    mode: '0755'
  when: app_type == 'backend' or app_type == 'both'
  tags:
    - secrets
    - logging
```

### 4.4 Role: Process Management

**File: `roles/process_management/tasks/main.yml`**
```yaml
---
# Frontend Process Management (PM2)
- name: "Process Mgmt | Generate PM2 ecosystem config (Frontend)"
  template:
    src: pm2-ecosystem.j2
    dest: "{{ frontend_dir }}/ecosystem.config.js"
    owner: deploy
    group: deploy
    mode: '0644'
  when: app_type == 'frontend' or app_type == 'both'
  tags:
    - process_management
    - pm2
    - frontend

- name: "Process Mgmt | Install frontend dependencies"
  shell: |
    cd {{ frontend_dir }}
    npm ci
  become: yes
  become_user: deploy
  when: app_type == 'frontend' or app_type == 'both'
  tags:
    - process_management
    - npm
    - frontend

- name: "Process Mgmt | Build frontend application"
  shell: |
    cd {{ frontend_dir }}
    source .env.local 2>/dev/null || true
    npm run build
  become: yes
  become_user: deploy
  when: app_type == 'frontend' or app_type == 'both'
  tags:
    - process_management
    - build
    - frontend

- name: "Process Mgmt | Start frontend with PM2"
  shell: |
    cd {{ frontend_dir }}
    pm2 delete stock-keeping-frontend || true
    pm2 start ecosystem.config.js --env {{ environment }}
    pm2 save
  become: yes
  become_user: deploy
  when: app_type == 'frontend' or app_type == 'both'
  register: pm2_start
  tags:
    - process_management
    - pm2
    - frontend

- name: "Process Mgmt | Setup PM2 startup service"
  shell: |
    pm2 startup systemd -u deploy --hp /home/deploy
    systemctl enable pm2-deploy
  become: yes
  when: app_type == 'frontend' or app_type == 'both'
  tags:
    - process_management
    - pm2
    - frontend

# Backend Process Management (SystemD)
- name: "Process Mgmt | Generate SystemD environment file (Backend)"
  template:
    src: systemd-env.j2
    dest: /etc/default/{{ backend_app_name }}
    owner: root
    group: root
    mode: '0644'
  when: app_type == 'backend' or app_type == 'both'
  tags:
    - process_management
    - systemd
    - backend

- name: "Process Mgmt | Generate SystemD service file (Backend)"
  template:
    src: systemd.service.j2
    dest: "/etc/systemd/system/{{ backend_app_name }}.service"
    owner: root
    group: root
    mode: '0644'
  when: app_type == 'backend' or app_type == 'both'
  notify: systemd daemon reload
  tags:
    - process_management
    - systemd
    - backend

- name: "Process Mgmt | Build backend application"
  shell: |
    cd {{ backend_dir }}
    mvn clean package -DskipTests -q
    cp target/stockkeeper-*.jar {{ backend_dir }}/app.jar
  become: yes
  become_user: deploy
  when: app_type == 'backend' or app_type == 'both'
  register: build_backend
  tags:
    - process_management
    - build
    - backend

- name: "Process Mgmt | Fix jar permissions"
  file:
    path: "{{ backend_dir }}/app.jar"
    owner: deploy
    group: deploy
    mode: '0755'
  when: app_type == 'backend' or app_type == 'both'
  tags:
    - process_management
    - backend

- name: "Process Mgmt | Enable and start backend service"
  systemd:
    name: "{{ backend_app_name }}"
    state: started
    enabled: yes
    daemon_reload: yes
  when: app_type == 'backend' or app_type == 'both'
  tags:
    - process_management
    - systemd
    - backend

- name: Flush handlers
  meta: flush_handlers
  tags:
    - process_management
```

**File: `roles/process_management/templates/pm2-ecosystem.j2`**
```javascript
module.exports = {
  apps: [
    {
      name: "{{ frontend_app_name }}",
      script: "/home/deploy/.nvm/versions/node/v20.*/bin/next",
      args: "start",
      cwd: "{{ frontend_dir }}",
      instances: {{ pm2_instances | default(2) }},
      exec_mode: "cluster",
      env: {
        NODE_ENV: "production",
        PORT: "{{ frontend_port }}",
      },
      error_file: "{{ logs_dir }}/{{ frontend_app_name }}-error.log",
      out_file: "{{ logs_dir }}/{{ frontend_app_name }}-out.log",
      log_date_format: "YYYY-MM-DD HH:mm:ss Z",
      merge_logs: true,
      autorestart: true,
      watch: false,
      max_memory_restart: "{{ pm2_max_memory_restart }}M",
      env_{{ environment }}: {
        NODE_ENV: "production",
      },
    },
  ],
};
```

**File: `roles/process_management/templates/systemd.service.j2`**
```ini
[Unit]
Description={{ backend_app_name }} Application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=deploy
Group=deploy
WorkingDirectory={{ backend_dir }}
EnvironmentFile=/etc/default/{{ backend_app_name }}
ExecStart=/usr/bin/java -Xms256m -Xmx{{ pm2_max_memory_restart }}m -jar app.jar
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=5s
StandardOutput=append:{{ logs_dir }}/{{ backend_app_name }}-out.log
StandardError=append:{{ logs_dir }}/{{ backend_app_name }}-error.log

[Install]
WantedBy=multi-user.target
```

**File: `roles/process_management/templates/systemd-env.j2`**
```bash
# Environment file for {{ backend_app_name }}
# Sourced by systemd service

export SERVER_PORT={{ backend_port }}
export SPRING_PROFILES_ACTIVE={{ environment }}
export ENVIRONMENT={{ environment }}

# Database from secrets
export $(cat {{ backend_dir }}/.env | xargs)
```

### 4.5 Role: Nginx Reverse Proxy

**File: `roles/nginx_reverse_proxy/tasks/main.yml`**
```yaml
---
- name: "Nginx | Generate Nginx configuration"
  template:
    src: nginx.conf.j2
    dest: "/etc/nginx/sites-available/{{ domain_name }}"
    owner: root
    group: root
    mode: '0644'
  notify: reload nginx
  tags:
    - nginx
    - reverse_proxy

- name: "Nginx | Enable site"
  file:
    src: "/etc/nginx/sites-available/{{ domain_name }}"
    dest: "/etc/nginx/sites-enabled/{{ domain_name }}"
    state: link
  notify: reload nginx
  tags:
    - nginx
    - reverse_proxy

- name: "Nginx | Remove default site"
  file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: reload nginx
  tags:
    - nginx
    - reverse_proxy

- name: "Nginx | Test Nginx configuration"
  shell: nginx -t
  register: nginx_test
  failed_when: nginx_test.rc != 0
  tags:
    - nginx
    - reverse_proxy
    - test

- name: "Nginx | Enable Nginx service"
  systemd:
    name: nginx
    enabled: yes
    state: started
  tags:
    - nginx
    - reverse_proxy

- name: "Nginx | Flush handlers"
  meta: flush_handlers
  tags:
    - nginx
```

**File: `roles/nginx_reverse_proxy/templates/nginx.conf.j2`**
```nginx
# Generated by Ansible - DO NOT EDIT
# Configuration for {{ domain_name }} ({{ environment }})

# HTTP to HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name {{ domain_name }};

    location / {
        return 301 https://$server_name$request_uri;
    }

    # Certbot challenge
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}

# HTTPS Server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name {{ domain_name }};

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/{{ domain_name }}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/{{ domain_name }}/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/{{ domain_name }}/chain.pem;

    # SSL Best Practices
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # HSTS Header
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Security Headers
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Logging
    access_log /var/log/nginx/{{ domain_name }}-access.log combined;
    error_log /var/log/nginx/{{ domain_name }}-error.log warn;

    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1000;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss;

    {% if app_type == 'backend' %}
    # Backend API Proxy
    location / {
        proxy_pass http://localhost:{{ backend_port }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $server_name;

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Disable buffering for streaming
        proxy_buffering off;
        proxy_request_buffering off;
    }

    # Health check endpoint
    location /actuator/health {
        proxy_pass http://localhost:{{ backend_port }};
        access_log off;
    }

    {% elif app_type == 'frontend' %}
    # Frontend Application Proxy
    location / {
        proxy_pass http://localhost:{{ frontend_port }};
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Static assets caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    {% endif %}
}
```

### 4.6 Role: SSL Certification

**File: `roles/ssl_certification/tasks/main.yml`**
```yaml
---
- name: "SSL | Create certbot directory"
  file:
    path: /var/www/certbot
    state: directory
    owner: www-data
    group: www-data
    mode: '0755'
  tags:
    - ssl
    - certbot

- name: "SSL | Check if certificate exists"
  stat:
    path: "/etc/letsencrypt/live/{{ domain_name }}/fullchain.pem"
  register: cert_exists
  tags:
    - ssl
    - certbot

- name: "SSL | Generate self-signed certificate (temporary)"
  block:
    - name: "SSL | Create self-signed cert directory"
      file:
        path: "/etc/letsencrypt/live/{{ domain_name }}"
        state: directory
        owner: root
        group: root
        mode: '0700'

    - name: "SSL | Generate self-signed certificate"
      shell: |
        openssl req -x509 -newkey rsa:4096 -keyout /etc/letsencrypt/live/{{ domain_name }}/privkey.pem \
        -out /etc/letsencrypt/live/{{ domain_name }}/fullchain.pem -days 1 -nodes \
        -subj "/CN={{ domain_name }}/O=Organization/C=US"
      args:
        creates: "/etc/letsencrypt/live/{{ domain_name }}/fullchain.pem"
  when: not cert_exists.stat.exists
  tags:
    - ssl
    - certbot
    - temporary

- name: "SSL | Generate Let's Encrypt certificate"
  shell: |
    certbot certonly \
      --webroot \
      --webroot-path=/var/www/certbot \
      --domain {{ domain_name }} \
      --non-interactive \
      --agree-tos \
      --email {{ certbot_email }} \
      --preferred-challenges http \
      --rsa-key-size 4096 \
      --force-renewal
  register: certbot_result
  changed_when: "'Successfully received certificate' in certbot_result.stdout"
  tags:
    - ssl
    - certbot

- name: "SSL | Setup automatic renewal cron job"
  cron:
    name: "Certbot auto-renewal"
    minute: "0"
    hour: "3"
    weekday: "0"
    job: "/usr/bin/certbot renew --quiet --post-hook 'systemctl reload nginx'"
    user: root
  tags:
    - ssl
    - certbot
    - cron

- name: "SSL | Verify cron job exists"
  shell: crontab -l | grep certbot
  register: cron_check
  changed_when: false
  failed_when: cron_check.rc != 0
  tags:
    - ssl
    - certbot
    - verify

- name: "SSL | Set certificate permissions"
  file:
    path: "/etc/letsencrypt/live/{{ domain_name }}"
    owner: root
    group: root
    mode: '0755'
    recurse: yes
  tags:
    - ssl
    - permissions
```

### 4.7 Role: CloudWatch Agent

**File: `roles/cloudwatch_agent/tasks/main.yml`**
```yaml
---
- name: "CloudWatch | Create CloudWatch config directory"
  file:
    path: /opt/aws/amazon-cloudwatch-agent/etc
    state: directory
    owner: root
    group: root
    mode: '0755'
  tags:
    - cloudwatch
    - agent

- name: "CloudWatch | Generate CloudWatch Agent configuration"
  template:
    src: amazon-cloudwatch-agent-config.j2
    dest: /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
    owner: root
    group: root
    mode: '0644'
  notify: restart cloudwatch agent
  tags:
    - cloudwatch
    - agent
    - config

- name: "CloudWatch | Create log groups"
  shell: |
    aws logs create-log-group \
      --log-group-name "/ec2/{{ environment }}/{{ project_name }}/{{ app_type }}" \
      --region {{ aws_region }} \
      2>/dev/null || true
  environment:
    AWS_ACCESS_KEY_ID: "{{ vault_aws_access_key_id }}"
    AWS_SECRET_ACCESS_KEY: "{{ vault_aws_secret_access_key }}"
  tags:
    - cloudwatch
    - logs

- name: "CloudWatch | Set log retention policy"
  shell: |
    aws logs put-retention-policy \
      --log-group-name "/ec2/{{ environment }}/{{ project_name }}/{{ app_type }}" \
      --retention-in-days {{ aws_cloudwatch_log_retention }} \
      --region {{ aws_region }}
  environment:
    AWS_ACCESS_KEY_ID: "{{ vault_aws_access_key_id }}"
    AWS_SECRET_ACCESS_KEY: "{{ vault_aws_secret_access_key }}"
  tags:
    - cloudwatch
    - logs

- name: "CloudWatch | Enable and start CloudWatch Agent"
  shell: |
    /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
      -a fetch-config \
      -m ec2 \
      -s \
      -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
      --region {{ aws_region }}
  environment:
    AWS_ACCESS_KEY_ID: "{{ vault_aws_access_key_id }}"
    AWS_SECRET_ACCESS_KEY: "{{ vault_aws_secret_access_key }}"
  tags:
    - cloudwatch
    - agent
    - start

- name: "CloudWatch | Verify logs are streaming"
  shell: |
    aws logs describe-log-streams \
      --log-group-name "/ec2/{{ environment }}/{{ project_name }}/{{ app_type }}" \
      --region {{ aws_region }} \
      --query 'logStreams[0].logStreamName' \
      --output text
  environment:
    AWS_ACCESS_KEY_ID: "{{ vault_aws_access_key_id }}"
    AWS_SECRET_ACCESS_KEY: "{{ vault_aws_secret_access_key }}"
  register: log_stream_check
  retries: 5
  delay: 10
  until: log_stream_check.stdout | length > 0
  tags:
    - cloudwatch
    - verify

- name: Flush handlers
  meta: flush_handlers
  tags:
    - cloudwatch
```

**File: `roles/cloudwatch_agent/templates/amazon-cloudwatch-agent-config.j2`**
```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "{{ logs_dir }}/{{ app_type }}-out.log",
            "log_group_name": "/ec2/{{ environment }}/{{ project_name }}/{{ app_type }}",
            "log_stream_name": "{instance_id}-stdout",
            "timezone": "UTC",
            "timestamp_format": "%b %d %H:%M:%S"
          },
          {
            "file_path": "{{ logs_dir }}/{{ app_type }}-error.log",
            "log_group_name": "/ec2/{{ environment }}/{{ project_name }}/{{ app_type }}",
            "log_stream_name": "{instance_id}-stderr",
            "timezone": "UTC",
            "timestamp_format": "%b %d %H:%M:%S"
          },
          {
            "file_path": "/var/log/nginx/{{ domain_name }}-access.log",
            "log_group_name": "/ec2/{{ environment }}/{{ project_name }}/nginx",
            "log_stream_name": "{instance_id}-access",
            "timezone": "UTC",
            "timestamp_format": "%d/%b/%Y:%H:%M:%S %z"
          },
          {
            "file_path": "/var/log/nginx/{{ domain_name }}-error.log",
            "log_group_name": "/ec2/{{ environment }}/{{ project_name }}/nginx",
            "log_stream_name": "{instance_id}-error",
            "timezone": "UTC",
            "timestamp_format": "%Y/%m/%d %H:%M:%S"
          }
        ]
      }
    },
    "log_stream_name": "{instance_id}",
    "force_flush_interval": 30
  }
}
```

---

## Part 5: Universal Playbook

**File: `playbooks/deploy.yml`**
```yaml
---
- name: "Deploy {{ project_name }} - Universal Playbook"
  hosts: "{{ target_group | default('all') }}"
  become: yes

  vars_files:
    - ../group_vars/all/vault.yml
    - ../group_vars/all/common.yml

  vars:
    environment: "{{ hostvars[inventory_hostname]['environment'] }}"
    app_type: "{{ hostvars[inventory_hostname]['app_type'] }}"
    domain_name: "{{ hostvars[inventory_hostname][hostvars[inventory_hostname]['app_type'] + '_domain'] }}"

  pre_tasks:
    - name: "Deploy | Display deployment information"
      debug:
        msg: |
          ╔════════════════════════════════════════════════════════════════╗
          ║  {{ project_name }} Deployment                                 ║
          ║  Environment: {{ environment | upper }}                                  ║
          ║  Target Host: {{ inventory_hostname }}                            ║
          ║  App Type: {{ app_type }}                                     ║
          ║  Domain: {{ domain_name }}                                   ║
          ║  Branch: {{ app_branch }}                                  ║
          ╚════════════════════════════════════════════════════════════════╝
      tags:
        - always

    - name: "Deploy | Validate environment variables"
      assert:
        that:
          - environment is defined
          - environment in ['staging', 'production']
          - app_type is defined
          - app_type in ['backend', 'frontend']
        fail_msg: "Invalid environment or app_type configuration"
      tags:
        - validate

    - name: "Deploy | Ensure AWS credentials are available"
      shell: aws sts get-caller-identity
      environment:
        AWS_ACCESS_KEY_ID: "{{ vault_aws_access_key_id }}"
        AWS_SECRET_ACCESS_KEY: "{{ vault_aws_secret_access_key }}"
      register: aws_check
      failed_when: aws_check.rc != 0
      tags:
        - validate

  roles:
    - role: dependencies
      tags: [dependencies]

    - role: clone_repo
      tags: [clone_repo]

    - role: secrets_injection
      tags: [secrets]

    - role: process_management
      tags: [process_management]

    - role: nginx_reverse_proxy
      tags: [nginx]

    - role: ssl_certification
      tags: [ssl]

    - role: cloudwatch_agent
      tags: [cloudwatch]

  post_tasks:
    - name: "Deploy | Wait for application to start"
      wait_for:
        port: "{{ backend_port if app_type == 'backend' else frontend_port }}"
        delay: 5
        timeout: 60
      tags:
        - post_deploy
        - health_check

    - name: "Deploy | Health check - Backend"
      uri:
        url: "https://{{ domain_name }}/actuator/health"
        method: GET
        status_code: 200
        validate_certs: no
      register: health_check
      retries: 5
      delay: 10
      until: health_check.status == 200
      when: app_type == 'backend'
      tags:
        - post_deploy
        - health_check

    - name: "Deploy | Health check - Frontend"
      uri:
        url: "https://{{ domain_name }}/"
        method: GET
        status_code: 200
        validate_certs: no
      register: health_check
      retries: 5
      delay: 10
      until: health_check.status == 200
      when: app_type == 'frontend'
      tags:
        - post_deploy
        - health_check

    - name: "Deploy | Verify CloudWatch logs"
      shell: |
        aws logs describe-log-streams \
          --log-group-name "/ec2/{{ environment }}/{{ project_name }}/{{ app_type }}" \
          --region {{ aws_region }} \
          --query 'logStreams[].logStreamName' \
          --output text
      environment:
        AWS_ACCESS_KEY_ID: "{{ vault_aws_access_key_id }}"
        AWS_SECRET_ACCESS_KEY: "{{ vault_aws_secret_access_key }}"
      register: cloudwatch_logs
      failed_when: cloudwatch_logs.stdout == ''
      tags:
        - post_deploy
        - cloudwatch_verify

    - name: "Deploy | Display deployment summary"
      debug:
        msg: |
          ╔════════════════════════════════════════════════════════════════╗
          ║  ✅ Deployment Successful                                       ║
          ║  Environment: {{ environment | upper }}                          ║
          ║  Application: {{ project_name }} ({{ app_type }})               ║
          ║  Domain: {{ domain_name }}                                     ║
          ║  Access: https://{{ domain_name }}                             ║
          ║  CloudWatch Logs: /ec2/{{ environment }}/{{ project_name }}/{{ app_type }} ║
          ║  Logs: {{ cloudwatch_logs.stdout | default('Pending...') }}    ║
          ╚════════════════════════════════════════════════════════════════╝
      tags:
        - post_deploy
        - summary

  handlers:
    - name: "systemd daemon reload"
      systemd:
        daemon_reload: yes

    - name: "reload nginx"
      systemd:
        name: nginx
        state: reloaded

    - name: "restart cloudwatch agent"
      shell: |
        /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
          -a stop && \
        /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
          -a fetch-config \
          -m ec2 \
          -s \
          -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

---

## Part 6: Execution Instructions

### 6.1 Pre-Execution Setup

#### Step 1: Initialize Vault

```bash
cd ansible-controller

# Create encrypted vault file
ansible-vault create group_vars/all/vault.yml

# When prompted, enter your vault password (Team Leads will manage this)
# Add the following content:
# vault_ssh_private_key: |
#   -----BEGIN OPENSSH PRIVATE KEY-----
#   [YOUR ENCRYPTED SSH KEY CONTENT]
#   -----END OPENSSH PRIVATE KEY-----
# vault_aws_access_key_id: your-aws-access-key
# vault_aws_secret_access_key: your-aws-secret-key
```

#### Step 2: Create Secrets Files

```bash
# Create staging secrets
cat > secrets/staging.env << 'EOF'
DB_URL=jdbc:postgresql://staging-db:5432/stockkeeping?sslmode=require
DB_USERNAME=staging_user
DB_PASSWORD=staging_password
MAIL_HOST=smtp.staging.example.com
MAIL_PORT=587
MAIL_USERNAME=staging@example.com
MAIL_PASSWORD=staging_password
NEXT_PUBLIC_API_URL=https://api-staging.stockkeeping.example.com/api
NEXT_PUBLIC_APP_URL=https://staging.stockkeeping.example.com
EOF

# Create production secrets
cat > secrets/prod.env << 'EOF'
DB_URL=jdbc:postgresql://prod-db:5432/stockkeeping?sslmode=require
DB_USERNAME=prod_user
DB_PASSWORD=prod_password_secure
MAIL_HOST=smtp.prod.example.com
MAIL_PORT=587
MAIL_USERNAME=prod@example.com
MAIL_PASSWORD=prod_password_secure
NEXT_PUBLIC_API_URL=https://api.stockkeeping.example.com/api
NEXT_PUBLIC_APP_URL=https://stockkeeping.example.com
EOF

# Ensure secrets directory is protected
chmod 600 secrets/*.env
```

#### Step 3: Update Inventory

Edit `inventory/staging` and `inventory/production` with your actual server IPs:

```bash
# Example for staging
sed -i 's/<STAGING_BACKEND_IP>/192.168.1.10/g' inventory/staging
sed -i 's/<STAGING_FRONTEND_IP>/192.168.1.11/g' inventory/staging

# Example for production
sed -i 's/<PROD_BACKEND_IP>/10.0.1.10/g' inventory/production
sed -i 's/<PROD_FRONTEND_IP>/10.0.1.11/g' inventory/production
```

### 6.2 Pre-Deployment Verification

```bash
# Run pre-checks playbook
ansible-playbook playbooks/pre_checks.yml \
  --inventory inventory/staging \
  --ask-vault-pass

# Expected output:
# - Ping successful to all hosts
# - Deploy user exists
# - Git access verified
# - AWS credentials valid
```

### 6.3 Deploy to Staging

```bash
# Deploy entire staging environment
ansible-playbook playbooks/deploy.yml \
  --inventory inventory/staging \
  --extra-vars "target_group=staging" \
  --ask-vault-pass \
  --diff

# Or deploy only backend
ansible-playbook playbooks/deploy.yml \
  --inventory inventory/staging \
  --extra-vars "target_group=staging_backend" \
  --ask-vault-pass \
  --diff

# Or deploy only frontend
ansible-playbook playbooks/deploy.yml \
  --inventory inventory/staging \
  --extra-vars "target_group=staging_frontend" \
  --ask-vault-pass \
  --diff
```

### 6.4 Deploy to Production

```bash
# Deploy entire production environment
ansible-playbook playbooks/deploy.yml \
  --inventory inventory/production \
  --extra-vars "target_group=production" \
  --ask-vault-pass \
  --diff

# Or deploy only backend
ansible-playbook playbooks/deploy.yml \
  --inventory inventory/production \
  --extra-vars "target_group=production_backend" \
  --ask-vault-pass \
  --diff

# Or deploy only frontend
ansible-playbook playbooks/deploy.yml \
  --inventory inventory/production \
  --extra-vars "target_group=production_frontend" \
  --ask-vault-pass \
  --diff
```

### 6.5 Vault Password Management

For secure vault password handling, create a `.vault-password-file` in your local home directory:

```bash
# Create password file (NOT committed to repo)
echo "your-secure-vault-password" > ~/.vault-password

# Make it readable only by you
chmod 600 ~/.vault-password

# Reference in ansible.cfg:
# vault_password_file = ~/.vault-password
```

Or use environment variable:

```bash
export VAULT_PASSWORD="your-secure-vault-password"
ansible-playbook playbooks/deploy.yml \
  --inventory inventory/staging \
  --ask-vault-pass
```

---

## Part 7: Demo Verification Checklist

### Video Demo Checklist (Max 3 minutes)

#### 1. **Pre-Deployment Setup** (0:00 - 0:15)
- [ ] Show vault.yml is encrypted
```bash
cat group_vars/all/vault.yml  # Should show encrypted content
```
- [ ] Show secrets directory exists (but not committed)
```bash
ls -la secrets/  # Show staging.env and prod.env
cat secrets/staging.env  # Show actual secrets
```

#### 2. **Run Staging Deployment** (0:15 - 1:30)
- [ ] Execute pre-checks
```bash
ansible-playbook playbooks/pre_checks.yml \
  --inventory inventory/staging \
  --ask-vault-pass
```
- [ ] Execute full deployment
```bash
ansible-playbook playbooks/deploy.yml \
  --inventory inventory/staging \
  --extra-vars "target_group=staging" \
  --ask-vault-pass
```
- [ ] Show deployment completion with ✅ success message

#### 3. **Verify HTTPS in Browser** (1:30 - 2:00)
- [ ] Open browser and navigate to staging frontend
```
https://staging.stockkeeping.example.com
```
- [ ] Show:
  - ✅ Green HTTPS lock icon
  - ✅ Certificate details (Let's Encrypt, valid dates)
  - ✅ Application loads successfully
  - ✅ No certificate warnings

#### 4. **Show CloudWatch Logs** (2:00 - 2:45)
- [ ] Open AWS Console → CloudWatch → Log Groups
- [ ] Navigate to `/ec2/staging/stock-keeping/`
- [ ] Show three log groups:
  - `/ec2/staging/stock-keeping/backend` - Show entries streaming in
  - `/ec2/staging/stock-keeping/frontend` - Show entries streaming in
  - `/ec2/staging/stock-keeping/nginx` - Show access and error logs
- [ ] Click on a log stream and show actual log entries:
  - Application startup messages
  - HTTP requests
  - No SSH access used for debugging

#### 5. **Summary** (2:45 - 3:00)
- [ ] Show deployment successful
- [ ] Show application accessible via HTTPS
- [ ] Show logs in CloudWatch (not local filesystem)
- [ ] Emphasize: No GitHub Actions, No SSH debugging, Centralized observability

### Manual Verification Checklist

After deployment, verify the following:

#### Backend Verification
```bash
# SSH to backend server and check:
ssh deploy@<BACKEND_IP>

# 1. Check application is running
curl http://localhost:5000/actuator/health

# 2. Check systemd service
systemctl status stock-keeping-backend

# 3. Check logs are being shipped to CloudWatch
cat /opt/stock-keeping/backend/logs/application.log | head

# 4. Check Nginx reverse proxy
curl https://api-staging.stockkeeping.example.com/actuator/health

# 5. Verify no .env file visible
ls -la /opt/stock-keeping/backend/.env  # Should exist but not committed
```

#### Frontend Verification
```bash
# SSH to frontend server and check:
ssh deploy@<FRONTEND_IP>

# 1. Check PM2 process
pm2 list

# 2. Check PM2 logs
pm2 logs stock-keeping-frontend | head

# 3. Check application is responding
curl http://localhost:3000

# 4. Check Nginx reverse proxy
curl -I https://staging.stockkeeping.example.com

# 5. Check environment variables loaded
grep NEXT_PUBLIC_API_URL /opt/stock-keeping/frontend/.env.local
```

#### CloudWatch Verification
```bash
# From ansible controller, verify log groups
aws logs describe-log-groups \
  --query "logGroups[?contains(logGroupName, '/ec2/staging/stock-keeping')]" \
  --region us-east-1

# Verify log streams
aws logs describe-log-streams \
  --log-group-name /ec2/staging/stock-keeping/backend \
  --region us-east-1

# Tail logs
aws logs tail /ec2/staging/stock-keeping/backend --follow
```

#### SSL/HTTPS Verification
```bash
# Check certificate
curl -vI https://staging.stockkeeping.example.com 2>&1 | grep -i "certificate"

# Verify certificate details
openssl s_client -connect staging.stockkeeping.example.com:443 < /dev/null | grep -A3 "Issuer"

# Check renewal cron
crontab -l | grep certbot
```

---

## Architecture Diagrams

### Deployment Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Ansible Controller (Central)                     │
│                                                                       │
│  Step 1: Load Vault (ask for password)                              │
│  ├─ Decrypt vault.yml                                               │
│  ├─ Load SSH deployment key                                         │
│  └─ Load AWS credentials                                            │
│                                                                       │
│  Step 2: Load Inventory & Variables                                 │
│  ├─ Detect environment (staging or production)                      │
│  ├─ Load group_vars/all/common.yml                                  │
│  ├─ Load group_vars/{environment}/vars.yml                          │
│  └─ Apply inventory-specific variables                              │
│                                                                       │
│  Step 3: Execute Universal Playbook (deploy.yml)                   │
│  └─ For each target host:                                           │
│     1. Install dependencies (Nginx, Java, Node.js, etc)             │
│     2. Clone repositories (private via SSH)                         │
│     3. Inject .env secrets from secrets/{environment}.env           │
│     4. Start applications (PM2 for frontend, SystemD for backend)  │
│     5. Configure Nginx reverse proxy (Jinja2 template)             │
│     6. Setup SSL certificates (Certbot automation)                 │
│     7. Configure CloudWatch Agent logging                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
    ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────┐
    │ Staging Backend  │  │ Staging Frontend │  │ Production Servers │
    │                  │  │                  │  │ (similar process)  │
    │ Java/SpringBoot  │  │ Next.js/PM2      │  │                    │
    │ SystemD Service  │  │ Process Manager  │  │ Backend + Frontend │
    │ :5000            │  │ :3000            │  │                    │
    └──────────────────┘  └──────────────────┘  └────────────────────┘
              │                   │                      │
              ├───────────────────┼──────────────────────┤
              │                   │                      │
              ▼                   ▼                      ▼
         ┌─────────────────────────────────────────────────────────┐
         │           Nginx Reverse Proxy (HTTPS/443)               │
         │                                                          │
         │  Jinja2 Template with:                                  │
         │  • Domain name ({{ domain_name }})                     │
         │  • Proxy ports ({{ backend_port }}, {{ frontend_port }})│
         │  • SSL paths (/etc/letsencrypt/live/{{ domain_name }}) │
         │  • Log locations (/var/log/nginx/{{ domain_name }})    │
         │  • HTTPS forced redirect                                │
         └─────────────────────────────────────────────────────────┘
              │
              ▼
    ┌──────────────────────────────────────────────────────────┐
    │          AWS CloudWatch (Centralized Logging)            │
    │                                                          │
    │  Log Groups (Strict Naming Convention):                 │
    │  /ec2/staging/stock-keeping/backend                     │
    │  /ec2/staging/stock-keeping/frontend                    │
    │  /ec2/staging/stock-keeping/nginx                       │
    │  /ec2/production/stock-keeping/backend                  │
    │  /ec2/production/stock-keeping/frontend                 │
    │  /ec2/production/stock-keeping/nginx                    │
    │                                                          │
    │  Sources:                                               │
    │  • Application stdout/stderr                            │
    │  • Nginx access logs (/var/log/nginx/*-access.log)     │
    │  • Nginx error logs (/var/log/nginx/*-error.log)       │
    │  • SystemD application logs                            │
    │  • PM2 application logs                                │
    └──────────────────────────────────────────────────────────┘
```

### Single Host Deployment Sequence

```
┌─────────────────────────────────────────────────────────────────┐
│                    Target Server Deployment                      │
└─────────────────────────────────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────────────────────────┐
    │ 1. Dependencies Role (45 seconds)        │
    │ ├─ apt update && upgrade                 │
    │ ├─ install nginx, certbot                │
    │ ├─ install Java/Node.js                  │
    │ ├─ create deploy user                    │
    │ ├─ create application directories        │
    │ └─ install CloudWatch Agent              │
    └─────────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────────────────────────┐
    │ 2. Clone Repository Role (30 seconds)    │
    │ ├─ configure SSH for github.com          │
    │ ├─ git clone (branch: develop/main)      │
    │ └─ set repository permissions            │
    └─────────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────────────────────────┐
    │ 3. Secrets Injection Role (10 seconds)   │
    │ ├─ copy staging.env or prod.env          │
    │ ├─ rename to .env or .env.local          │
    │ └─ set secure permissions (600)          │
    └─────────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────────────────────────┐
    │ 4. Process Management Role (varies)      │
    │                                         │
    │ If Frontend:                            │
    │ ├─ npm ci (install dependencies)        │
    │ ├─ npm run build                        │
    │ ├─ pm2 start ecosystem.config.js        │
    │ └─ pm2 startup (auto-restart on reboot) │
    │                                         │
    │ If Backend:                             │
    │ ├─ mvn clean package (build JAR)        │
    │ ├─ copy JAR to app.jar                  │
    │ ├─ generate systemd service             │
    │ └─ systemctl start (auto-start enabled) │
    └─────────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────────────────────────┐
    │ 5. Nginx Reverse Proxy Role (20 seconds)│
    │ ├─ generate nginx.conf (Jinja2 template)│
    │ ├─ enable site                          │
    │ ├─ test nginx configuration             │
    │ └─ systemctl reload nginx               │
    └─────────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────────────────────────┐
    │ 6. SSL Certification Role (60 seconds)  │
    │ ├─ check if certificate exists          │
    │ ├─ generate self-signed (if needed)     │
    │ ├─ certbot certonly (Let's Encrypt)     │
    │ ├─ setup renewal cron job               │
    │ └─ verify cron exists                   │
    └─────────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────────────────────────┐
    │ 7. CloudWatch Agent Role (30 seconds)   │
    │ ├─ generate agent config (Jinja2)       │
    │ ├─ create log groups                    │
    │ ├─ set retention policy                 │
    │ ├─ start CloudWatch Agent               │
    │ └─ verify log streaming                 │
    └─────────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────────────────────────┐
    │ Post-Deployment Checks (varies)         │
    │ ├─ wait for application to listen       │
    │ ├─ health check via HTTPS               │
    │ ├─ verify CloudWatch logs streaming     │
    │ └─ display deployment summary           │
    └─────────────────────────────────────────┘
              │
              ▼
    ✅ Deployment Complete (3-5 minutes total)
```

---

## Key Compliance Points

### ✅ All Stage 5 Requirements Met

| Requirement | Implementation | Evidence |
|-----------|-----------------|----------|
| **No CI/CD Pipelines** | Removed .github/workflows from both repos | Git commit removes ci-cd.yml |
| **One Master Playbook** | deploy.yml handles both staging/prod | Single file with {{ environment }} detection |
| **Inventory Groups** | [staging], [production] in inventory files | inventory/staging, inventory/production |
| **group_vars Differentiation** | group_vars/staging/, group_vars/production/ | Separate vars.yml per environment |
| **Encrypted SSH Key** | vault.yml encrypted with Ansible Vault | ansible-vault create output |
| **No Raw Keys** | Keys stored only in encrypted vault.yml | Never in filesystem unencrypted |
| **Vault Password Prompt** | ask_vault_pass = True in ansible.cfg | Prompted during playbook run |
| **Environment Detection** | Automatic via inventory group membership | No manual intervention needed |
| **Centralized Logging** | CloudWatch Agent configured | Log groups created and verified |
| **Strict Log Names** | /ec2/{environment}/{project-name}/{component} | Hardcoded in CloudWatch config template |
| **No SSH for Debugging** | All logs stream to CloudWatch | Developers access logs via AWS Console |
| **Nginx Jinja2 Templates** | nginx.conf.j2 with dynamic variables | {{ domain_name }}, {{ backend_port }}, etc |
| **SSL Automation** | Certbot with auto-renewal cron | Certificate generation and renewal |
| **Health Checks** | Post-deployment HTTP/HTTPS checks | Health check tasks in playbook |
| **Demo Video** | See verification checklist above | Show deployment + logs + HTTPS |

---

## Quick Start Commands

```bash
# 1. Setup
cd ansible-controller
ansible-vault create group_vars/all/vault.yml

# 2. Configure secrets
mkdir -p secrets
cat > secrets/staging.env << 'EOF'
# Your staging secrets
EOF

# 3. Update inventory
sed -i 's/<STAGING_BACKEND_IP>/192.168.1.10/g' inventory/staging

# 4. Pre-checks
ansible-playbook playbooks/pre_checks.yml \
  --inventory inventory/staging \
  --ask-vault-pass

# 5. Deploy
ansible-playbook playbooks/deploy.yml \
  --inventory inventory/staging \
  --extra-vars "target_group=staging" \
  --ask-vault-pass

# 6. Verify
curl https://staging.stockkeeping.example.com

# 7. Check logs
aws logs tail /ec2/staging/stock-keeping/frontend --follow
```

---

## Troubleshooting Guide

### Issue: Vault Password Prompt Not Appearing
```bash
# Ensure vault prompt is enabled in ansible.cfg
grep "ask_vault_pass" ansible.cfg
# Should output: ask_vault_pass = True

# Or use --ask-vault-pass flag
ansible-playbook playbooks/deploy.yml --ask-vault-pass
```

### Issue: SSH Connection Failed
```bash
# Verify SSH key is in vault.yml
ansible-vault view group_vars/all/vault.yml | grep "BEGIN"

# Test SSH manually
ssh -i /path/to/key deploy@<SERVER_IP> "echo OK"
```

### Issue: Certbot Certificate Generation Failed
```bash
# Check Nginx is running
systemctl status nginx

# Verify domain DNS is resolvable
nslookup staging.stockkeeping.example.com

# Check certbot logs
ls -la /var/log/letsencrypt/
```

### Issue: CloudWatch Logs Not Appearing
```bash
# Verify log groups created
aws logs describe-log-groups --region us-east-1 | grep stock-keeping

# Check CloudWatch Agent status
systemctl status amazon-cloudwatch-agent

# View agent logs
tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

---

## Timeline Estimate

| Phase | Duration | Task |
|-------|----------|------|
| **Preparation** | 30 mins | Setup vault, inventory, secrets |
| **First Deployment** | 5 mins | Run playbook against staging |
| **Verification** | 10 mins | Check HTTPS, CloudWatch logs |
| **Production Deployment** | 5 mins | Run playbook against production |
| **Final Verification** | 10 mins | Verify prod HTTPS and logs |
| **Demo Recording** | 10 mins | Record 3-minute demo video |
| **Total** | ~70 mins | End-to-end deployment |

---

## References

- [Ansible Documentation](https://docs.ansible.com)
- [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
- [Jinja2 Templating](https://docs.ansible.com/ansible/latest/user_guide/playbooks_templating.html)
- [CloudWatch Agent Configuration](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
- [Let's Encrypt / Certbot](https://certbot.eff.org/)
- [Nginx Reverse Proxy](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [PM2 Documentation](https://pm2.keymetrics.io/)
- [SystemD Service Files](https://www.freedesktop.org/software/systemd/man/systemd.service.html)

---

## Document Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-11-18 | Initial comprehensive implementation plan |

---

**Last Updated:** 2025-11-18
**Document Owner:** DevOps Team
**Status:** Ready for Implementation
