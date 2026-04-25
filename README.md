# [Backstage](https://backstage.io)

This is your newly scaffolded Backstage App, Good Luck!

To start the app, run the following on the terminal
 
```sh
yarn install
yarn test:all
yarn tsc
yarn start
yarn build:backend
yarn build:all
```

## First set of git commands = from git init to push the first time 

```bash
echo "# backstage-from-scatch-demo" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:username/backstage-from-scatch-demo.git
git push -u origin main


OR 

# …or push an existing repository from the command line
git remote add origin git@github.com:Olumuyiwa/backstage-from-scatch-demo.git
git branch -M main
git push -u origin main
```

## For github CI/CD do the following 
```
mkdir .github 
cd .github 
mkdir workflows
```

# Complete Flow: Deploy Backstage to a GCP VM

## Phase 1 — Create Google Cloud Account

1️⃣ **Create a Gmail account** (recommended format)

Example:

```
name-test-currentmonth-year
```

Example:

```
olutestmarch2026@gmail.com
```

2️⃣ Go to the Google Cloud Console

```
https://console.cloud.google.com
```

3️⃣ **Sign up for Google Cloud**

You will receive the **$300 free credit** for new users.

4️⃣ **Enable 2-Step Verification**

Go to:

```
Google Account → Security → 2-Step Verification
```

This is required for security and recommended before creating resources.

---

# Phase 2 — Create a GCP Virtual Machine

1️⃣ Open **Compute Engine**
1. Enable the "Compute Engine API"
2.

```

## Creating a CI CD process
1. Create a file `ci_cd.yml` in the .github/workflows
2. Add the following 

Console → Compute Engine → VM Instances
  
Create Instance
```

3️⃣ Configure the VM

Example settings:

| Setting      | Value                     |
| ------------ | ------------------------- |
| Name         | backstage-vm              |
| Region       | europe-west1 (or closest) |
| Machine type | e2-medium                 |
| Boot disk    | Ubuntu 22.04              |
| Disk size    | 10GB                      |

4️⃣ **Allow HTTP access**

Check the box:
```

Allow HTTP traffic

```

5️⃣ Click Create
```

Your VM will now start.

---

# Phase 3 — Allow Port 3000 for Backstage

Backstage runs on **port 3000** by default.

Create a firewall rule.

Go to:

```
VPC Network → Firewall
```

Click **Create Firewall Rule**

Example configuration:

| Field     | Value           |
| --------- | --------------- |
| Name      | allow-backstage |
| Targets   | All instances   |
| Source IP | 0.0.0.0/0       |
| Protocol  | TCP             |
| Port      | 3000            |

Save the rule.

Now your VM can accept traffic on:

```
http://VM_EXTERNAL_IP:3000
```

---

# Phase 4 — Create SSH Keys Locally

On your computer run:

```bash
# This didnt work 
ssh-keygen -t rsa -b 4096 -f private_key.pem

# This worked
ssh-keygen -t rsa -b 4096 -C "olutrains@gmail.com" -f private_key.pem

```

After adding the public key to the VM and the private key to Github, connectivity was sorted 
To test connectivity, this was run locally 

```
ssh -i private_key olutrains@34.77.212.107
```

This creates:

```
private_key.pem
private_key.pem.pub
```

Example files:

```
private_key.pem       (private key)
private_key.pem.pub   (public key)
```

---

# Phase 5 — Add SSH Key to the VM

Go to:

```
Compute Engine → VM Instance → Edit
```

Find the **SSH Keys section**.

Paste the contents of:

```
private_key.pem.pub
```

Save.

Your VM now allows SSH using that key.

---

# Phase 6 — Test SSH Connection

From your local machine run:

```bash
ssh -i private_key.pem USERNAME@VM_EXTERNAL_IP
```

Example:

```bash
ssh -i private_key.pem mayo@34.120.55.10
```

If successful you will connect to your VM.

---

# Phase 7 — Prepare GitHub Secrets

In your **GitHub** repository:

Go to:

```
Settings → Secrets → Actions
```

Create these secrets.

| Secret           | Example      |
| ---------------- | ------------ |
| GCP_SSH_USERNAME | mayo         |
| GCP_INSTANCE_IP  | 34.120.55.10 |

---

# Phase 8 — Add the SSH Private Key to GitHub Actions

Upload your private key as a repository secret.

Create secret:

```
GCP_SSH_KEY
```

Paste contents of:

```
private_key.pem
```

---

# Phase 9 — CI/CD Deployment Flow

When code is pushed:

1️⃣ GitHub Actions builds the project
2️⃣ It connects to the VM via SSH
3️⃣ It copies the application
4️⃣ It installs dependencies
5️⃣ It starts the server

Deployment pipeline example:

```
Developer pushes code
        ↓
GitHub Actions runs workflow
        ↓
SSH into VM
        ↓
Copy application
        ↓
Install Node.js 24
        ↓
Install dependencies
        ↓
Build Backstage
        ↓
Start backend server
        ↓
Application accessible on port 3000
```

---

# Phase 10 — Access the Application

Open a browser:

```
http://VM_EXTERNAL_IP:3000
```

Example:

```
http://34.120.55.10:3000
```

You should now see the **Backstage portal**.

---

# Final Deployment Architecture

```
Developer
    ↓
GitHub Repository
    ↓
GitHub Actions CI/CD
    ↓
SSH Deployment
    ↓
GCP Virtual Machine
    ↓
Node.js 24
    ↓
Backstage Backend
    ↓
Port 3000
    ↓
Public Access
```

---

## Update the github actions file in .github/workflows folder with this updated content 

```
 
name: ci-and-cd-to-gcp-vm-backstage

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

jobs:
# =========================
# CI — BUILD & TEST
# =========================
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js 24
        uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: 'yarn'

      - name: Install Dependencies
        run: yarn install --frozen-lockfile

      - name: Compile TypeScript
        run: yarn tsc

      - name: Run Unit Tests
        run: yarn test:all

      - name: Build Backstage
        run: yarn build:all


# =========================
# CD — DEPLOY TO GCP VM
# =========================
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      # Create SSH key from GitHub Secret
      - name: Create SSH key
        run: |
          echo "${{ secrets.GCP_SSH_KEY }}" > private_key.pem
          chmod 600 private_key.pem

      # Copy project to VM
      - name: Copy Backstage project to VM
        run: |
          scp -r -o StrictHostKeyChecking=no \
          -i private_key.pem \
          . \
          ${{ secrets.GCP_SSH_USERNAME }}@${{ secrets.GCP_INSTANCE_IP }}:/home/${{ secrets.GCP_SSH_USERNAME }}/backstage

      # Prepare VM (Node 24 + Yarn)
      - name: Prepare VM environment
        run: |
          ssh -o StrictHostKeyChecking=no -i private_key.pem \
          ${{ secrets.GCP_SSH_USERNAME }}@${{ secrets.GCP_INSTANCE_IP }} << 'EOF'

            echo "Stopping existing Backstage..."
            pkill -f backstage || true

            echo "Updating packages..."
            sudo apt-get update

            echo "Installing Node.js 24..."
            curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
            sudo apt-get install -y nodejs

            echo "Installing Yarn..."
            sudo npm install -g yarn

            node -v
            npm -v
            yarn -v

          EOF

      # Install + Build on VM
      - name: Install dependencies and build on VM
        run: |
          ssh -o StrictHostKeyChecking=no -i private_key.pem \
          ${{ secrets.GCP_SSH_USERNAME }}@${{ secrets.GCP_INSTANCE_IP }} << 'EOF'

            cd ~/backstage

            echo "Installing dependencies..."
            yarn install --frozen-lockfile

            echo "Building Backstage..."
            yarn build

          EOF

      # Start Backstage
      - name: Start Backstage application
        run: |
          ssh -o StrictHostKeyChecking=no -i private_key.pem \
          ${{ secrets.GCP_SSH_USERNAME }}@${{ secrets.GCP_INSTANCE_IP }} << 'EOF'

            cd ~/backstage

            echo "Starting Backstage backend..."
            nohup yarn start > backstage.log 2>&1 &

            echo "Deployment completed."

          EOF

```


## deploy **Backstage** to a **Google Cloud** VM using **GitHub Actions**.

I organized it into **logical phases** so the process is easy to follow.

---

done

