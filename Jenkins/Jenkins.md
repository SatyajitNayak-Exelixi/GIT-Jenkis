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

Install Jenkins. On a new VM, create the keyring directory and download the current Jenkins 2026 signing key:

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key \
  | sudo tee /etc/apt/keyrings/jenkins-keyring.asc >/dev/null
```

Add the Jenkins stable repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null
```

Verify the key and repository:

```bash
sudo gpg --show-keys --with-fingerprint /etc/apt/keyrings/jenkins-keyring.asc
sudo apt update
```

Install and start Jenkins:

```bash
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
sudo vi /etc/sudoers.d/jenkins-exlixi
```

In `vi`, press `i` to enter insert mode, add the rule, then press `Esc` and type `:wq` followed by `Enter` to save and exit.

Add:

```text
jenkins ALL=(root) NOPASSWD: /usr/bin/mkdir, /usr/bin/rsync, /usr/bin/ln, /usr/bin/rm, /usr/sbin/nginx, /usr/bin/systemctl reload nginx, /usr/bin/systemctl start nginx, /usr/bin/systemctl enable nginx, /usr/bin/systemctl status nginx --no-pager
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
7. Replace the placeholders in the Groovy script below.
8. Paste the completed script into the Jenkins script editor.
9. Select **Save**.

The Groovy script is stored in Jenkins, but the application code is **not** written inside the script. The script explicitly checks out `index.html`, `styles.css`, `script.js`, `nginx.conf`, and the other files from Git. It then validates them, copies them to the web root, enables the Nginx site, and starts/reloads Nginx.

```groovy
pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_APPLICATION', choices: ['Yes', 'No'], description: 'Are you okay to deploy the Exelixi application?')
        string(name: 'CHANGE_REQUEST_NUMBER', defaultValue: '', description: 'Enter the approved Change Request number')
    }

    environment {
        GIT_URL = 'https://github.com/SatyajitNayak-Exelixi/GIT-Jenkis.git'
        GIT_BRANCH = 'main'
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
                git branch: env.GIT_BRANCH, url: env.GIT_URL
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

        stage('Deployment Approval') {
            steps {
                script {
                    if (!params.CHANGE_REQUEST_NUMBER?.trim()) {
                        error 'A Change Request number is required before deployment.'
                    }
                    if (params.DEPLOY_APPLICATION != 'Yes') {
                        error 'Deployment cancelled. The approval parameter was set to No.'
                    }
                    echo "Deployment approved for Change Request ${params.CHANGE_REQUEST_NUMBER}"
                }
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

When running the pasted script, Jenkins does not require **Pipeline script from SCM** configuration. Select **Build with Parameters** and provide exactly two values:

- `DEPLOY_APPLICATION`: select `Yes` to deploy or `No` to cancel
- `CHANGE_REQUEST_NUMBER`: enter the approved Change Request number

The public Git repository and `main` branch are fixed inside the pipeline. The Nginx configuration and frontend files are retrieved from Git during the **Checkout Code** stage.

### 6.1. Visualize the Pipeline

Install plugins that display pipeline stages and the stage graph:

1. Open **Manage Jenkins > Plugins**.
2. Open **Available plugins**.
3. Search for **Pipeline: Stage View** and install it.
4. Search for **Pipeline Graph View** and install it if available.
5. Restart Jenkins.

After restart and saving the pipeline job configuration:

1. Open the Jenkins dashboard.
2. Select the pipeline job.
3. Start a new build.
4. Open the build and select **Pipeline Overview**, **Stages**, or **Pipeline Steps**.

## 7. Run the Jenkins Pipeline

1. Open `Exelixi-Nginx-Deployment`.
2. Select **Build with Parameters**.
3. Set `DEPLOY_APPLICATION` to `Yes` or `No`.
4. Enter `CHANGE_REQUEST_NUMBER`.
5. Select **Build**.
6. Open **Build History** and select the build number.
7. Open **Console Output**.

Before deployment, the pipeline validates both values:

- If `DEPLOY_APPLICATION` is `No`, deployment is cancelled.
- If `CHANGE_REQUEST_NUMBER` is blank, the build fails.
- The deployment starts only when both approval and Change Request number are valid.

A successful build shows:

- Code checkout completed
- Source files validated
- Deployment approved
- Change Request number validated
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

### Build is waiting for the next available executor

This status means Jenkins is running, but the build has no executor node available:

```text
Still waiting to schedule task
Waiting for next available executor
```

Configure an executor on the Jenkins server:

1. Open **Manage Jenkins**.
2. Select **Nodes**.
3. Select **Built-In Node**.
4. Select **Configure**.
5. Set **Number of executors** to at least `1`.
6. Set usage to **Use this node as much as possible**.
7. Select **Save**.

Then verify the node status:

- The built-in node must be **online**.
- Its executor count must be at least `1`.
- Jenkins must not be in **quiet down** mode.

Confirm that the pipeline begins with `agent any`. If a temporary agent is used instead, ensure at least one agent is online and that it has Java and Git installed.

**Verified fix:** set the **Built-In Node** executor count to `1` or more and use **Use this node as much as possible**. The build should then leave the scheduling wait and enter the pipeline stages.

### Disk space is below the Jenkins threshold

A warning such as the following means Jenkins has automatically marked the Built-In Node offline:

```text
Disk space is below threshold of 1.00 GiB
Only 444.84 MiB out of 454.31 MiB left on /tmp
```

Check the current disk usage:

```bash
df -hT / /tmp
mount | grep ' on /tmp '
```

If `/tmp` is shown as `tmpfs` with approximately 455 MiB, it is too small for Jenkins's 1 GiB disk-space threshold. Remount `/tmp` with a larger temporary size:

```bash
sudo mount -o remount,size=2G /tmp
df -hT /tmp
```

To retain this setting after reboot, add or update an `/tmp` entry in `/etc/fstab`:

```bash
sudo vi /etc/fstab
```

Add:

```text
tmpfs /tmp tmpfs defaults,nosuid,nodev,size=2G 0 0
```

Save the file and verify it after reboot with `df -hT /tmp`.

Clean package and temporary files:

```bash
sudo apt clean
sudo apt autoremove -y
sudo journalctl --vacuum-size=100M
sudo find /tmp -mindepth 1 -maxdepth 1 -exec rm -rf {} +
```

Remove Jenkins workspaces only while no build is running:

```bash
sudo find /var/lib/jenkins/workspace -mindepth 1 -maxdepth 1 -type d -exec rm -rf {} +
```

Verify that `/tmp` has more free space:

```bash
df -h /tmp
sudo systemctl restart jenkins
sudo systemctl status jenkins --no-pager
```

In Jenkins, open **Manage Jenkins > Nodes**, select **Built-In Node**, and use **Bring this node back online**. If the warning remains, increase the EC2 root volume and extend the filesystem:

```bash
lsblk
sudo growpart /dev/nvme0n1 1
sudo resize2fs /dev/nvme0n1p1
df -h /
```

Use `lsblk` to confirm the actual partition before running `growpart` or `resize2fs`.

### Jenkins repository key or source needs repair

Use these commands only when Jenkins is being reinstalled or an existing repository/key is invalid:

```bash
sudo rm -f /etc/apt/keyrings/jenkins-keyring.asc
sudo rm -f /usr/share/keyrings/jenkins-keyring.asc
sudo rm -f /etc/apt/sources.list.d/jenkins.list
This 403 error means Jenkins CSRF protection rejected a configuration or form submission without a valid security crumb:

```text
No valid crumb was included in the request
```

It is not an application or Nginx deployment error.

Try these steps in order:

1. Refresh the Jenkins browser page.
2. Sign out of Jenkins and sign back in.
3. Open Jenkins using the exact Jenkins URL configured under **Manage Jenkins > System**.
4. Close the browser and reopen Jenkins; clear cookies for the VM IP or hostname if the page is cached.
5. On the Jenkins VM, verify the service and restart it:

```bash
sudo systemctl status jenkins --no-pager
sudo systemctl restart jenkins
```

Then wait until Jenkins is ready and open:

```text
http://<VM_PUBLIC_IP>:8080/
```

Do not install or configure Jenkins behind an untrusted reverse proxy during the lab unless the proxy forwards the Jenkins crumb and session cookies correctly.

If Jenkins is exposed through HTTP, keep the Jenkins URL, browser URL, and security-group access consistent. Avoid older Jenkins tabs containing expired CSRF tokens, and retry the Save action after signing in again.

## Production Recommendations

- Restrict Jenkins port `8080` to an administrator IP or VPN.
- Use HTTPS and a domain name.
- Install a TLS certificate through Let's Encrypt or AWS Certificate Manager.
- Use Jenkins credentials for private Git repositories.
- Back up `/var/www/exlixi` and `/etc/nginx`.
- Pin Jenkins and plugin versions for operational stability.
