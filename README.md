# Enroll-App Deployment Guide

Complete guide for deploying a React/Vite application to a Linux server using GitHub Actions CI/CD pipeline.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Server Setup](#server-setup)
- [SSH Key Configuration](#ssh-key-configuration)
- [Nginx Configuration](#nginx-configuration)
- [GitHub Actions Setup](#github-actions-setup)
- [Environment Variables](#environment-variables)
- [Deployment Process](#deployment-process)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- A Linux server (Ubuntu/Debian recommended)
- Domain name pointing to your server
- Node.js and npm installed on server
- Git installed on server and local machine
- GitHub repository with your React/Vite app

## Login to Server

Standard SSH login method:

```bash
ssh -l root -p 22 xx.x.xx.xx
# OR
ssh root@xx.x.xx.xx -p 22
```

## Server Setup

### 1. Update Server and Install Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git nginx
```

### 2. Install Node.js and npm using NVM

It is recommended to use [nvm](https://github.com/nvm-sh/nvm) to manage Node.js versions.

```bash
# Install Node.js (system default, optional)
sudo apt-get install -y nodejs

# Install nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# Load nvm (you may need to restart your terminal or run the next line)
source ~/.bashrc

# List available Node.js versions
nvm list-remote

# Example output:
#        v16.20.2
#        v17.9.1
#        v18.20.2
#        v20.14.0
#        v21.7.3
#        v22.2.0

# Install Node.js version 18 (recommended)
nvm install 18

# Use Node.js 18
nvm use 18

# Verify installations
node -v
npm -v
nvm -v
```

### 3. Verify Installations

```bash
node --version
npm --version
nginx -version
git --version
```

### 3. Create Deployment Directory

```bash
sudo mkdir -p /var/www/html/Admittance/_work/Enroll-App/Enroll-App
sudo chown -R $USER:$USER /var/www/html/Admittance/
sudo chmod -R 755 /var/www/html/Admittance/
```

### 4. Clone Your Repository

```bash
cd /var/www/html/Admittance/_work/Enroll-App/
git clone https://github.com/yourusername/your-repo-name.git Enroll-App
```

## SSH Key Configuration

### 1. Generate SSH Key Pair (Run on Local Machine)

```bash
ssh-keygen -t rsa -b 4096 -C "github-actions@yourdomain.com" -f ~/.ssh/github_actions_key
```

When prompted:

- Press Enter for file location (or specify custom path)
- Press Enter twice to leave passphrase empty (required for CI/CD)

### 2. Add Public Key to Server

**Option A - Using ssh-copy-id:**

```bash
ssh-copy-id -i ~/.ssh/github_actions_key.pub username@your-server-ip
```

**Option B - Manual method (one-liner):**

```bash
cat ~/.ssh/github_actions_key.pub | ssh username@your-server-ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

**Option C - Manual copy-paste:**

1. On your local machine, display your public key:
   ```bash
   cat ~/.ssh/github_actions_key.pub
   ```
2. Copy the entire output.

3. On your server, run:
   ```bash
   mkdir -p ~/.ssh
   echo "paste-your-public-key-here" >> ~/.ssh/authorized_keys
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/authorized_keys
   ```

### 3. Test SSH Connection

**Standard SSH login to server:**

```bash
ssh -l root -p 22 xx.x.xx.xx
# OR
ssh root@xx.x.xx.xx -p 22
```

**Test with your new SSH key:**

```bash
ssh -i ~/.ssh/github_actions_key username@your-server-ip
# OR with specific port
ssh -i ~/.ssh/github_actions_key -p 22 username@your-server-ip
```

### 4. Set Proper Permissions on Server

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

## Nginx Configuration

### 1. Create Nginx Server Block

```bash
sudo nano /etc/nginx/sites-available/yourdomain.com.conf
```

### 2. Add Configuration

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name yourdomain.com www.yourdomain.com;

    # Document root for your React app build files
    root /var/www/html/Admittance/_work/Enroll-App/Enroll-App/build;
    index index.html index.htm;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied expired no-cache no-store private auth;
    gzip_types
        application/atom+xml
        application/javascript
        application/json
        application/ld+json
        application/manifest+json
        application/rss+xml
        application/vnd.geo+json
        application/vnd.ms-fontobject
        application/x-font-ttf
        application/x-web-app-manifest+json
        font/opentype
        image/bmp
        image/svg+xml
        image/x-icon
        text/cache-manifest
        text/css
        text/plain
        text/vcard
        text/vnd.rim.location.xloc
        text/vtt
        text/x-component
        text/x-cross-domain-policy;

    # Handle React Router (SPA routing)
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Static assets caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Error pages
    error_page 404 /index.html;

    # Security
    server_tokens off;

    # Logging
    access_log /var/log/nginx/yourdomain.com.access.log;
    error_log /var/log/nginx/yourdomain.com.error.log;
}
```

### 3. Enable Site and Test Configuration

```bash
sudo ln -s /etc/nginx/sites-available/yourdomain.com.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## GitHub Actions Setup

### 1. Create Workflow Directory

```bash
mkdir -p .github/workflows
```

### 2. Create Deployment Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: React App Project

on: [push, workflow_dispatch]

jobs:
  deploy:
    runs-on: self-hosted
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v3
        with:
          clean: true
          fetch-depth: 1

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: "18"
          cache: "npm"

      - name: Cache node modules
        uses: actions/cache@v3
        with:
          path: |
            node_modules
            ~/.npm
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-

      - name: Cache React build
        uses: actions/cache@v3
        with:
          path: |
            build
            node_modules/.cache
          key: ${{ runner.os }}-react-build-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('src/**/*.[jt]s', 'src/**/*.[jt]sx') }}

      - name: Create .env file for build
        run: |
          cat > .env << EOL
          REACT_APP_API_KEY=${{ secrets.REACT_APP_API_KEY }}
          REACT_APP_AUTH_DOMAIN=${{ secrets.REACT_APP_AUTH_DOMAIN }}
          REACT_APP_DATABASE_URL=${{ secrets.REACT_APP_DATABASE_URL }}
          REACT_APP_PROJECT_ID=${{ secrets.REACT_APP_PROJECT_ID }}
          REACT_APP_STORAGE_BUCKET=${{ secrets.REACT_APP_STORAGE_BUCKET }}
          REACT_APP_MESSAGING_SENDER_ID=${{ secrets.REACT_APP_MESSAGING_SENDER_ID }}
          REACT_APP_CONFIRMATION_EMAIL_REDIRECT=${{ secrets.REACT_APP_CONFIRMATION_EMAIL_REDIRECT }}
          REACT_APP_FLUTTERWAVE_PUBLIC_KEY=${{ secrets.REACT_APP_FLUTTERWAVE_PUBLIC_KEY }}
          REACT_APP_SUPER_ADMIN_EMAIL=${{ secrets.REACT_APP_SUPER_ADMIN_EMAIL }}
          SKIP_PREFLIGHT_CHECK=true
          EOL

      - name: Install Dependencies
        run: |
          echo "Starting npm install at $(date)"
          npm ci --prefer-offline --no-audit --no-fund
          echo "Finished npm install at $(date)"
        timeout-minutes: 10

      - name: Build React App
        env:
          NODE_OPTIONS: "--max_old_space_size=4096"
          CI: "false"
          GENERATE_SOURCEMAP: "false"
          REACT_APP_API_KEY: ${{ secrets.REACT_APP_API_KEY }}
          REACT_APP_AUTH_DOMAIN: ${{ secrets.REACT_APP_AUTH_DOMAIN }}
          REACT_APP_DATABASE_URL: ${{ secrets.REACT_APP_DATABASE_URL }}
          REACT_APP_PROJECT_ID: ${{ secrets.REACT_APP_PROJECT_ID }}
          REACT_APP_STORAGE_BUCKET: ${{ secrets.REACT_APP_STORAGE_BUCKET }}
          REACT_APP_MESSAGING_SENDER_ID: ${{ secrets.REACT_APP_MESSAGING_SENDER_ID }}
          REACT_APP_CONFIRMATION_EMAIL_REDIRECT: ${{ secrets.REACT_APP_CONFIRMATION_EMAIL_REDIRECT }}
          REACT_APP_FLUTTERWAVE_PUBLIC_KEY: ${{ secrets.REACT_APP_FLUTTERWAVE_PUBLIC_KEY }}
          REACT_APP_SUPER_ADMIN_EMAIL: ${{ secrets.REACT_APP_SUPER_ADMIN_EMAIL }}
        run: |
          echo "Starting build process at $(date)..."
          npm run build

          if [ -d "build" ]; then
            echo "Build completed successfully at $(date)"
            ls -la build/
          else
            echo "Build failed - directory not found"
            exit 1
          fi
        timeout-minutes: 15

      - name: Deploy to Server
        if: success()
        uses: appleboy/ssh-action@master
        env:
          REACT_APP_API_KEY: ${{ secrets.REACT_APP_API_KEY }}
          REACT_APP_AUTH_DOMAIN: ${{ secrets.REACT_APP_AUTH_DOMAIN }}
          REACT_APP_DATABASE_URL: ${{ secrets.REACT_APP_DATABASE_URL }}
          REACT_APP_PROJECT_ID: ${{ secrets.REACT_APP_PROJECT_ID }}
          REACT_APP_STORAGE_BUCKET: ${{ secrets.REACT_APP_STORAGE_BUCKET }}
          REACT_APP_MESSAGING_SENDER_ID: ${{ secrets.REACT_APP_MESSAGING_SENDER_ID }}
          REACT_APP_CONFIRMATION_EMAIL_REDIRECT: ${{ secrets.REACT_APP_CONFIRMATION_EMAIL_REDIRECT }}
          REACT_APP_FLUTTERWAVE_PUBLIC_KEY: ${{ secrets.REACT_APP_FLUTTERWAVE_PUBLIC_KEY }}
          REACT_APP_SUPER_ADMIN_EMAIL: ${{ secrets.REACT_APP_SUPER_ADMIN_EMAIL }}
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          port: ${{ secrets.PORT }}
          key: ${{ secrets.SSHKEY }}
          envs: |
            REACT_APP_API_KEY,
            REACT_APP_AUTH_DOMAIN,
            REACT_APP_DATABASE_URL,
            REACT_APP_PROJECT_ID,
            REACT_APP_STORAGE_BUCKET,
            REACT_APP_MESSAGING_SENDER_ID,
            REACT_APP_CONFIRMATION_EMAIL_REDIRECT,
            REACT_APP_FLUTTERWAVE_PUBLIC_KEY,
            REACT_APP_SUPER_ADMIN_EMAIL
          script: |
            echo "Starting server deployment at $(date)"

            if [ ! -d "/var/www/html/Admittance/_work/Enroll-App/Enroll-App" ]; then
              echo "Creating deployment directory..."
              mkdir -p /var/www/html/Admittance/_work/Enroll-App/Enroll-App
            fi

            cd /var/www/html/Admittance/_work/Enroll-App/Enroll-App || exit 1

            cat > .env << EOL
            REACT_APP_API_KEY=${REACT_APP_API_KEY}
            REACT_APP_AUTH_DOMAIN=${REACT_APP_AUTH_DOMAIN}
            REACT_APP_DATABASE_URL=${REACT_APP_DATABASE_URL}
            REACT_APP_PROJECT_ID=${REACT_APP_PROJECT_ID}
            REACT_APP_STORAGE_BUCKET=${REACT_APP_STORAGE_BUCKET}
            REACT_APP_MESSAGING_SENDER_ID=${REACT_APP_MESSAGING_SENDER_ID}
            REACT_APP_CONFIRMATION_EMAIL_REDIRECT=${REACT_APP_CONFIRMATION_EMAIL_REDIRECT}
            REACT_APP_FLUTTERWAVE_PUBLIC_KEY=${REACT_APP_FLUTTERWAVE_PUBLIC_KEY}
            REACT_APP_SUPER_ADMIN_EMAIL=${REACT_APP_SUPER_ADMIN_EMAIL}
            SKIP_PREFLIGHT_CHECK=true
            EOL

            echo "Pulling latest code..."
            git pull origin main

            echo "Installing dependencies on server..."
            npm install

            echo "Building project on server..."
            npm run build

            if [ -d "build" ]; then
              echo "Build folder created successfully"
              ls -la build/
            else
              echo "Build folder not found - build may have failed"
              exit 1
            fi

            if [ -f "./.scripts/deploy.sh" ]; then
              chmod +x ./.scripts/deploy.sh
              ./.scripts/deploy.sh
            fi

            echo "Server deployment completed at $(date)"
          timeout: 5m
          command_timeout: 8m

  cleanup:
    needs: deploy
    if: always()
    runs-on: self-hosted
    steps:
      - name: Setup Node.js for cleanup
        uses: actions/setup-node@v3
        with:
          node-version: "18"

      - name: Cleanup
        run: |
          echo "Starting cleanup at $(date)"
          rm -rf node_modules/.cache 2>/dev/null || echo "No node_modules cache to clean"
          rm -rf /tmp/runner_temp/* 2>/dev/null || echo "No temp files to clean"
          echo "Final memory status:"
          free -h
          echo "Cleanup completed at $(date)"
```

## Environment Variables

### 1. GitHub Repository Secrets

Go to your repository → Settings → Secrets and variables → Actions

<!--
If the image below does not render, make sure you have added the file `assets/github1.png` to your repository.
-->

![GitHub Actions Secrets Settings](assets/github1.png)

Add these secrets:

| Secret Name                             | Description                  | Example                                  |
| --------------------------------------- | ---------------------------- | ---------------------------------------- |
| `HOST`                                  | Server IP or domain          | `192.168.1.100`                          |
| `USERNAME`                              | SSH username                 | `ubuntu`                                 |
| `PORT`                                  | SSH port                     | `22`                                     |
| `SSHKEY`                                | Private key content          | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| `REACT_APP_API_KEY`                     | Firebase API key             | `AIzaSyC...`                             |
| `REACT_APP_AUTH_DOMAIN`                 | Firebase auth domain         | `yourapp.firebaseapp.com`                |
| `REACT_APP_DATABASE_URL`                | Firebase database URL        | `https://yourapp.firebaseio.com`         |
| `REACT_APP_PROJECT_ID`                  | Firebase project ID          | `your-project-id`                        |
| `REACT_APP_STORAGE_BUCKET`              | Firebase storage bucket      | `your-project.appspot.com`               |
| `REACT_APP_MESSAGING_SENDER_ID`         | Firebase messaging sender ID | `123456789`                              |
| `REACT_APP_CONFIRMATION_EMAIL_REDIRECT` | Email redirect URL           | `https://yourdomain.com/confirm`         |
| `REACT_APP_FLUTTERWAVE_PUBLIC_KEY`      | Flutterwave public key       | `FLWPUBK_TEST-...`                       |
| `REACT_APP_SUPER_ADMIN_EMAIL`           | Super admin email            | `admin@yourdomain.com`                   |

### 2. Get Private Key Content

```bash
cat ~/.ssh/github_actions_key
```

Copy the entire output including the BEGIN and END lines.

## Deployment Process

### 1. Commit and Push Changes

```bash
git add .
git commit -m "Add deployment configuration"
git push origin main
```

### 2. Monitor Deployment

- Go to your repository on GitHub
- Click on "Actions" tab
- Watch your workflow run

### 3. Verify Deployment

```bash
# Check if build folder exists
ls -la /var/www/html/Admittance/_work/Enroll-App/Enroll-App/build/

# Check nginx status
sudo systemctl status nginx

# Check nginx logs
sudo tail -f /var/log/nginx/yourdomain.com.error.log
```

## Troubleshooting

### Common Issues

**1. SSH Authentication Failed**

```bash
# Standard login to server
ssh -l root -p 22 xx.x.xx.xx

# Test SSH connection manually with key
ssh -i ~/.ssh/github_actions_key username@your-server-ip

# Test with specific port
ssh -i ~/.ssh/github_actions_key -p 22 username@your-server-ip

# Check authorized_keys on server
cat ~/.ssh/authorized_keys | grep "github-actions"
```

**2. Build Folder Not Found**

```bash
# Check if npm run build works locally
npm run build

# Check server nodejs version
node --version
npm --version
```

**3. Permission Denied**

```bash
# Fix directory permissions
sudo chown -R $USER:$USER /var/www/html/Admittance/
sudo chmod -R 755 /var/www/html/Admittance/
```

**4. Nginx 404 Error**

```bash
# Test nginx configuration
sudo nginx -t

# Check if build files exist
ls -la /var/www/html/Admittance/_work/Enroll-App/Enroll-App/build/
```

**5. Environment Variables Not Working**

- Verify all secrets are added in GitHub repository settings
- Check that secret names match exactly in workflow file
- Ensure .env file is created during deployment

### Useful Commands

```bash
# Restart services
sudo systemctl restart nginx
sudo systemctl restart ssh

# Check logs
sudo journalctl -u nginx -f
sudo tail -f /var/log/nginx/error.log

# Manual deployment test
cd /var/www/html/Admittance/_work/Enroll-App/Enroll-App
git pull origin main
npm install
npm run build
```

## Security Considerations

1. **Use SSH keys instead of passwords**
2. **Keep secrets secure in GitHub repository settings**
3. **Regularly update dependencies**
4. **Monitor server logs for suspicious activity**
5. **Consider using HTTPS with SSL certificates**

## Next Steps

1. **Set up SSL certificate** using Let's Encrypt
2. **Configure domain DNS** to point to your server
3. **Set up monitoring** and alerting
4. **Implement automated backups**
5. **Consider using a CDN** for better performance

---

## Support

If you encounter issues:

1. Check the GitHub Actions logs
2. Review server logs
3. Verify all configuration files
4. Test SSH connection manually
5. Ensure all secrets are properly configured

Happy deploying! 🚀

# github-CI-CD-deployment
