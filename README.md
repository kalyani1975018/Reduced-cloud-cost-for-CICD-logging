# Jenkins Build Logs → S3 Automation

This project automatically monitors a Jenkins server, and for every job that exists (or gets newly created), creates a matching folder in an S3 bucket. Every time a job is built, that build's log file is uploaded into its job's folder in S3, named by build number (e.g. `1.log`, `2.log`, `3.log`). The whole process is scheduled to run automatically every day using `cron`.

## Prerequisites

- An AWS account with permission to launch EC2 instances and create S3 buckets
- Basic familiarity with SSH

---

## Step 1: Launch an EC2 Instance

1. Go to the AWS Console → EC2 → **Launch Instance**
2. Name it something identifiable, e.g. `Shell-scripting Project`
3. Choose **Ubuntu Server** (latest LTS) as the AMI
4. Choose an instance type (this project used `c7i-flex.large` — anything with at least 2 vCPU / 2GB+ RAM works fine for Jenkins)
5. Create or select a key pair (`.pem` file) — you'll need this to SSH in
6. Under **Network settings**, allow inbound traffic on:
   - Port `22` (SSH)
   - Port `8080` (Jenkins default UI port)
7. Launch the instance in your preferred region (this project used `us-east-1`, N. Virginia)

Connect to it:
```bash
ssh -i your-key.pem ubuntu@<your-ec2-public-ip>
```

## Step 2: Install Java (required by Jenkins)

Jenkins runs on Java, so install it first:
```bash
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
java -version
```
You should see a Java version printed out, confirming it installed correctly.

## Step 3: Install Jenkins

```bash

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
```

Start and enable Jenkins so it runs automatically on reboot:
```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```
---

## Step 4: Unlock Jenkins & Install Plugins

1. In your browser, go to:
   ```
   http://<your-ec2-public-ip>:8080
   ```
2. Jenkins will ask for an **initial admin password**. Get it by running:
   ```bash
   cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Copy the password shown and paste it into the browser prompt
4. Click **Install suggested plugins** and wait for it to finish
5. Create your first admin user when prompted (username/password of your choice)

## Step 5: Create Jenkins Jobs

Create at least two jobs so you can see the per-job folder logic working. These are set up as **Pipeline** jobs:

1. Jenkins Dashboard → **New Item**
2. Enter a name (e.g. `First-job`) → select **Pipeline** → OK
3. Scroll down to the **Pipeline** section, set to `Pipeline script`, and paste in a simple pipeline, for example:
   ```groovy
   pipeline {
       agent any
       stages {
           stage('Continuous Download') {
               steps {
                   git branch: 'main', url: 'https://github.com/kalyani1975018/maven.git'
               }
           }
       }
   }
   ```
4. Save, then click **Build Now** a few times to generate multiple build logs
5. Repeat the same process to create a **second job** (e.g. `Second-job`) — you can point it at the same repo or a different one

> **Note:** the `git` step in the pipeline clones the given repository as part of the build. Make sure the repo URL is publicly accessible, or add credentials in Jenkins if it's private.

## Step 6: Create the S3 Bucket

Using the AWS Console:
1. Go to **S3** → **Create bucket**
2. Bucket name: `shell-script-project` (must be globally unique — if taken, choose your own name and update the script accordingly)
3. Choose your region (keep note of it — you'll need it for AWS CLI config)
4. Leave other settings as default (Block Public Access ON is recommended) → **Create bucket**

Or via AWS CLI (after CLI is installed/configured — see Step 7):
```bash
aws s3 mb s3://shell-script-project --region us-east-1
```

## Step 7: Install & Configure AWS CLI on the EC2 (Ubuntu) Server

```bash
cd ~
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install -y unzip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

Configure it with credentials that have S3 permissions:
```bash
aws configure
```
You'll be prompted for:
```
AWS Access Key ID:     <your access key>
AWS Secret Access Key: <your secret key>
Default region name:   <e.g. us-east-1>
Default output format: json
```

## Step 8: The Script

Create a file named upload-jenkins-buildjobs-logs.sh in ubuntu home directory (`/home/ubuntu/`):

```bash
#!/bin/bash
# Variable

# Author: Kalyani
# Description: Shell script to upload jenkins log files to s3 bucket

JENKINS_HOME="/var/lib/jenkins"                # Your Jenkins home directory
S3_BUCKET="s3://shell-script-project"          # Your S3 bucket name (no trailing slash)
DATE=$(date +%Y-%m-%d)                         # Today's date

# Check if AWS CLI is installed
if ! command -v aws &> /dev/null; then
    echo "AWS CLI is not installed. Please install it to proceed."
    exit 1
fi

# Iterate through all job directories
for job_dir in "$JENKINS_HOME/jobs/"*/; do
    job_name=$(basename "$job_dir")

    # Create a folder for this job in S3, only if it doesn't already exist
    if ! aws s3 ls "$S3_BUCKET/$job_name/" &> /dev/null; then
        aws s3api put-object \
            --bucket "$(echo "$S3_BUCKET" | sed 's|s3://||')" \
            --key "$job_name/" &> /dev/null
        echo "Created new folder in S3 for job: $job_name"
    fi

    # Iterate through build directories for the job
    for build_dir in "$job_dir/builds/"*/; do
        # Get build number and log file path
        build_number=$(basename "$build_dir")
        log_file="$build_dir/log"

        # Check if log file exists and was created today
        if [ -f "$log_file" ] && [ "$(date -r "$log_file" +%Y-%m-%d)" == "$DATE" ]; then
            # Upload log file into that job's folder, named by build number
            s3_target="$S3_BUCKET/$job_name/$build_number.log"
            aws s3 cp "$log_file" "$s3_target" --only-show-errors

            if [ $? -eq 0 ]; then
                echo "Uploaded: $job_name/$build_number -> $s3_target"
            else
                echo "Failed to upload: $job_name/$build_number"
            fi
        fi
    done
done
```

Make it executable:
```bash
chmod +x upload-jenkins-buildjobs-logs.sh
```

Test it manually first:
```bash
/bin/bash /home/ubuntu/upload-jenkins-buildjobs-logs.sh
```

## Step 9: Automate with Cron

Open the cron editor:
```bash
crontab -e
```
(First time only — it may ask you to choose an editor; type `1` for `nano`.)

Add this line at the bottom (runs daily at **11:00 PM UTC**):
```
0 23 * * * /bin/bash /home/ubuntu/upload-jenkins-buildjobs-logs.sh >> /home/ubuntu/jenkins_s3_upload.log 2>&1
```

> **Note on timezones:** the EC2 instance clock is UTC by default. Check with `timedatectl`. If you want the script to run at 11 PM **India time (IST, UTC+5:30)**, use `30 17 * * *` instead (5:30 PM UTC = 11:00 PM IST).

Save and exit (in nano: `Ctrl+O`, `Enter`, then `Ctrl+X`).

Confirm it saved:
```bash
crontab -l
```

### Testing without waiting for the scheduled time

To confirm cron is working immediately rather than waiting until 11 PM, temporarily set the schedule to a couple of minutes from now:
```bash
date                          # check current time, e.g. 12:50
crontab -e                    # set to 2 minutes ahead, e.g. "52 12 * * *"
```
Wait a few minutes, then check:
```bash
cat /home/ubuntu/jenkins_s3_upload.log
```
If you see `Uploaded: ...` lines, it's working. Then reset the schedule back to your real target time (`0 23 * * *` or `30 17 * * *`).

Referanced screenshots:

<img width="1920" height="1080" alt="Screenshot (225)" src="https://github.com/user-attachments/assets/d880413b-a18d-4bfd-9110-2adbd855c00c" />

<img width="1920" height="1080" alt="Screenshot (226)" src="https://github.com/user-attachments/assets/6d53aeac-9707-4fde-910d-f3c5449dbe7f" />

<img width="1920" height="1080" alt="Screenshot (228)" src="https://github.com/user-attachments/assets/02e1f68b-59d8-4cd3-95d0-f10e72a293bf" />

<img width="1920" height="1080" alt="Screenshot (229)" src="https://github.com/user-attachments/assets/72014d7f-3cd8-46cb-adfd-7cfe4530c62c" />

<img width="1920" height="1080" alt="Screenshot (230)" src="https://github.com/user-attachments/assets/02922957-970e-4cc6-b9f4-6ed9f6079e32" />

<img width="1920" height="1080" alt="Screenshot (231)" src="https://github.com/user-attachments/assets/402ce090-3346-42ec-b352-c6b5ba41cf4b" />

<img width="1920" height="1080" alt="Screenshot (236)" src="https://github.com/user-attachments/assets/3c448e3d-c06d-4165-9ce5-37bec625c418" />

<img width="1920" height="1080" alt="Screenshot (232)" src="https://github.com/user-attachments/assets/11b553ae-bdb7-4668-be7d-38fd6e60e337" />

<img width="1920" height="1080" alt="Screenshot (257)" src="https://github.com/user-attachments/assets/49a4bc1a-6174-425d-b4a8-8f202270e27d" />

<img width="1920" height="1080" alt="Screenshot (256)" src="https://github.com/user-attachments/assets/b9c4c2a1-181c-4b88-85d6-0eab6c82d25f" />

<img width="1920" height="1080" alt="Screenshot (258)" src="https://github.com/user-attachments/assets/3689ed3f-12fe-4840-ae9b-09ecfa65907e" />

<img width="1920" height="1080" alt="Screenshot (259)" src="https://github.com/user-attachments/assets/9e9a4304-48c2-4df7-bf02-b40bafa20699" />

<img width="1920" height="1080" alt="Screenshot (260)" src="https://github.com/user-attachments/assets/c9680c7c-808e-4a6b-be4c-6539aaa2b7e0" />

<img width="1920" height="1080" alt="Screenshot (264)" src="https://github.com/user-attachments/assets/dcc8c9da-ef7d-4770-b792-f44c5ce94d75" />

<img width="1920" height="1080" alt="Screenshot (266)" src="https://github.com/user-attachments/assets/96c057aa-2d78-40b2-a10c-46a1f039b9ed" />

<img width="1920" height="1080" alt="Screenshot (267)" src="https://github.com/user-attachments/assets/11039e96-6d30-4c40-84c1-ef741cdc1775" />

<img width="1920" height="1080" alt="Screenshot (268)" src="https://github.com/user-attachments/assets/54666610-ea55-43d1-b70e-47540a0c9584" />

