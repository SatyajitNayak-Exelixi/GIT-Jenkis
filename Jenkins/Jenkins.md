# Deploy the Exelixi Nginx Frontend with Jenkins on One AWS VM

This guide uses **one Ubuntu EC2 VM** as the Jenkins server, application server, and Nginx host. Jenkins checks out the repository, installs the frontend files on the same VM, validates Nginx, and starts or reloads Nginx.

## Architecture

```text
Developer
   |
   | Push or manual build
   v
Git Repository
   |
   | Checkout
   v
Ubuntu AWS VM
+-------------------------------+
| Jenkins controller            |
| Jenkins pipeline workspace    |
| Nginx web server              |
| /var/www/exlixi               |
+-------------------------------+
   |
   | HTTP port 80
   v
Browser
```

## 1. Create and Configure the AWS VM

Create one Ubuntu 22.04 or 24.04 EC2 instance:

- Instance type: `t3.micro` or similar
- OS: Ubuntu 22.04/24.04 LTS
- Storage: at least 8 GB
- Create or select a key pair

Add an inbound security-group rule for each required port:

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | Your public IP | Secure SSH administration |
| 80 | TCP | `0.0.0.0/0` | Public Nginx website access |
| 8080 | TCP | Your public IP | Jenkins UI |

Do not expose Jenkins port `8080` to the entire internet in a real production environment.

Connect to the VM:

```bash
ssh -i /path/to/key.pem ubuntu@<VM_PUBLIC_IP>
```

Verify that Ubuntu is reachable and update the packages:

```bash
sudo apt update
sudo apt upgrade -y
```

## 2. Install Jenkins and Nginx

Install Java, Git, Nginx, and supporting tools:

```bash
sudo apt install -y fontconfig openjdk-21-jre git nginx rsync
java -version
nginx -v
```

If OpenJDK 21 is unavailable, use `openjdk-17-jre` instead.

Install Jenkins:

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

printf '%s\n' \
  "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null

sudo apt update
sudo apt install -y jenkins

sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Nginx is installed now, but it should be enabled and started **after** Jenkins has deployed the Exelixi files and installed the Nginx site configuration. This prevents Nginx from starting with a missing or default Exelixi web root.

Verify Jenkins and validate the initial Nginx installation:

```bash
sudo systemctl status jenkins --no-pager
sudo nginx -t
```

## 3. Open Jenkins and Complete Initial Setup

Open Jenkins in a browser:

```text
http://<VM_PUBLIC_IP>:8080
```

Retrieve the initial administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Complete these setup steps:

1. Enter the initial password.
2. Select **Install suggested plugins**.
3. Create an administrator account.
4. Set the Jenkins URL to `http://<VM_PUBLIC_IP>:8080/`.

## 4. Application Files

The frontend files are located in:

```text
Jenkins/exlixi-nginx-app/
```

Required files:

- `index.html`
- `styles.css`
- `script.js`
- `nginx.conf`
- `Jenkinsfile`

Commit and push the repository to GitHub or another Git server.

## 5. Configure Jenkins on the Same VM

Jenkins must have permission to deploy files to `/var/www/exlixi` and manage Nginx. Open sudoers for Jenkins:

```bash
sudo visudo -f /etc/sudoers.d/jenkins-exlixi
```

Add:

```text
jenkins ALL=(root) NOPASSWD: /usr/bin/mkdir, /usr/bin/rsync, /usr/bin/ln, /usr/bin/rm, /usr/sbin/nginx, /bin/systemctl reload nginx, /bin/systemctl start nginx, /bin/systemctl enable nginx, /bin/systemctl status nginx
```

Save the file, then verify Jenkins can run it without a password:

```bash
sudo -u jenkins sudo -n nginx -t
sudo -u jenkins sudo -n systemctl status nginx --no-pager
```

Create the web root:

```bash
sudo mkdir -p /var/www/exlixi
sudo chown -R jenkins:www-data /var/www/exlixi
sudo chmod 755 /var/www/exlixi
```

## 6. Create the Jenkins Pipeline Job

In Jenkins:

1. Select **New Item**.
2. Enter `Exelixi-Nginx-Deployment`.
3. Select **Pipeline**.
4. Select **OK**.
5. Set **Build Triggers** as needed:
   - **GitHub hook trigger** for automatic builds
   - **Poll SCM** for scheduled builds
   - Leave both unchecked for manual builds
6. Under **Pipeline**, select **Pipeline script**.
7. Paste the Groovy script below.
8. Select **Save**.

This script first checks out the repository, validates the frontend, copies the files to the web root, enables the Nginx site, and starts/reloads Nginx. This is after the Git checkout and file deployment, so Nginx is started with the Exelixi site configuration and files already present.

```groovy
pipeline {
    agent any

    environment {
        APP_SOURCE = 'Jenkins/exlixi-nginx-app'
        WEB_ROOT = '/var/www/exlixi'
        NGINX_SITE = '/etc/nginx/sites-available/exlixi'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Validate Source') {
            steps {
                sh '''
                    set -e
                    test -f "${APP_SOURCE}/index.html"
                    test -f "${APP_SOURCE}/styles.css"
                    test -f "${APP_SOURCE}/script.js"
                    test -f "${APP_SOURCE}/nginx.conf"
                    echo "Exelixi source files found"
                '''
            }
        }

        stage('Deploy to Nginx') {
            steps {
                sh '''
                    set -e

                    sudo mkdir -p "${WEB_ROOT}"
                    sudo rsync -av --delete \
                        --exclude '.git/' \
                        --exclude 'Jenkinsfile' \
                        "${APP_SOURCE}/" "${WEB_ROOT}/"

                    sudo rsync -av "${APP_SOURCE}/nginx.conf" "${NGINX_SITE}"
                    sudo ln -sfn "${NGINX_SITE}" /etc/nginx/sites-enabled/exlixi
                    sudo rm -f /etc/nginx/sites-enabled/default

                    sudo nginx -t
                    sudo systemctl enable nginx
                    sudo systemctl start nginx
                    sudo systemctl reload nginx
                '''
            }
        }

        stage('Verify Application') {
            steps {
                sh '''
                    set -e
                    curl --fail --silent --show-error http://127.0.0.1/ | grep -i "Exelixi"
                    echo "Exelixi application is responding on port 80"
                '''
            }
        }
    }

    post {
        success {
            echo 'Exelixi frontend deployed successfully.'
        }
        failure {
            echo 'Deployment failed. Review the Jenkins console output.'
        }
    }
}
```

If Jenkins is configured with **Pipeline script from SCM** instead, set:

- SCM: `Git`
- Repository URL: your repository URL
- Branch: `*/main` or the default branch
- Script Path: `Jenkins/exlixi-nginx-app/Jenkinsfile`

The repository can be configured to use the same script directly.

## 7. Run the Jenkins Pipeline

1. Open `Exelixi-Nginx-Deployment`.
2. Select **Build Now**.
3. Open **Build History**.
4. Select the build number.
5. Open **Console Output**.

A successful build shows:

- Code checkout completed
- Source files validated
- Files copied to `/var/www/exlixi`
- Nginx configuration accepted
- Nginx running on port 80
- The local HTTP check returned the Exelixi page

## 8. Browse the Application

Open this URL in a browser:

```text
http://<VM_PUBLIC_IP>/
```

A port is required in the Jenkins URL, but Nginx on HTTP port `80` normally uses no explicit port in the browser URL.

For local checking on the VM:

```bash
curl -I http://127.0.0.1/
curl --fail http://127.0.0.1/ | grep -i Exelixi
sudo nginx -t
sudo systemctl status nginx --no-pager
```

## 9. Manual Deployment on the VM

Use this process if Jenkins is unavailable or to perform a manual lab deployment.

### Step 1: Clone the Repository

```bash
cd ~
git clone <REPOSITORY_URL> exlixi-project
cd exlixi-project
```

Checkout a branch if required:

```bash
git checkout main
git pull origin main
```

### Step 2: Create the Web Root

```bash
sudo mkdir -p /var/www/exlixi
sudo chown -R ubuntu:www-data /var/www/exlixi
```

### Step 3: Copy the Frontend Files

```bash
sudo rsync -av --delete \
  Jenkins/exlixi-nginx-app/ \
  /var/www/exlixi/

sudo rm -f /var/www/exlixi/Jenkinsfile
```

### Step 4: Install the Nginx Site

```bash
sudo cp Jenkins/exlixi-nginx-app/nginx.conf \
  /etc/nginx/sites-available/exlixi

sudo ln -sfn /etc/nginx/sites-available/exlixi \
  /etc/nginx/sites-enabled/exlixi

sudo rm -f /etc/nginx/sites-enabled/default
```

### Step 5: Test and Start Nginx

```bash
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl reload nginx
```

### Step 6: Verify the Manual Deployment

```bash
curl -I http://127.0.0.1/
curl --fail http://127.0.0.1/ | grep -i Exelixi
sudo systemctl status nginx --no-pager
```

Open:

```text
http://<VM_PUBLIC_IP>/
```

## 10. Optional Jenkins Webhook

In GitHub:

1. Open **Repository > Settings > Webhooks**.
2. Select **Add webhook**.
3. Set Payload URL to `http://<VM_PUBLIC_IP>:8080/github-webhook/`.
4. Set Content type to `application/json`.
5. Select **Just the push event**.
6. Save the webhook.

In Jenkins, enable **GitHub hook trigger for GITScm polling** in the pipeline job.

For production, use HTTPS and place Jenkins behind a reverse proxy or load balancer.

## 11. Troubleshooting

### Jenkins cannot check out Git

- Verify the repository URL and branch.
- Add Jenkins Git credentials if the repository is private.
- Check outbound network access from the VM.
- Confirm the repository contains the Jenkinsfile path.

### Jenkins cannot write to `/var/www` or reload Nginx

- Verify `/etc/sudoers.d/jenkins-exlixi`.
- Run `sudo -u jenkins sudo -n nginx -t`.
- Confirm the Jenkins user belongs to the correct system account.
- Review Jenkins console output for permission errors.

### Nginx returns the default page

```bash
sudo ls -l /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl reload nginx
```

Remove `/etc/nginx/sites-enabled/default` and confirm `/etc/nginx/sites-enabled/exlixi` exists.

### Jenkins is not accessible

- Confirm TCP `8080` is allowed in the AWS security group.
- Confirm the VM public IP is reachable from your network.
- Run `sudo systemctl status jenkins --no-pager`.

### Nginx is not accessible

- Confirm TCP `80` is allowed in the AWS security group.
- Confirm Nginx is running.
- Run `sudo nginx -t`.
- Check `sudo cat /var/log/nginx/error.log`.

## Production Recommendations

- Restrict Jenkins port `8080` to an administrator IP or VPN.
- Use HTTPS and a domain name.
- Install a TLS certificate through Let's Encrypt or AWS Certificate Manager.
- Use Jenkins credentials for private Git repositories.
- Back up `/var/www/exlixi` and `/etc/nginx`.
- Pin Jenkins and plugin versions for operational stability.
