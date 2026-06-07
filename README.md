# CloudPulse — Ansible Configuration

> **Repo 2 of 3** — Run this **SECOND**, from the Ansible server. Configures the Jenkins server with everything it needs to run CI/CD pipelines.

This repository contains Ansible roles that turn a bare Amazon Linux 2023 EC2 instance into a fully working **Jenkins CI/CD server** — installing Java, Docker, Jenkins, and the AWS toolchain.

---

## Where This Fits

```
Phase 1 → cloudpulse-bootstrap   → Creates Jenkins + Ansible EC2
Phase 2 → cloudpulse-ansible     → Configures Jenkins (THIS REPO)
Phase 3 → cloudpulse-infra       → Creates EKS/VPC/ECR via Jenkins
```

You run this playbook **from the Ansible server** (created in Phase 1), targeting the **Jenkins server** over SSH.

---

## What It Installs (4 Roles)

The playbook runs 4 roles **in order** — each role is self-contained with its own `tasks/` and `defaults/`, and ends with **verification** steps that assert the tool installed correctly (the playbook fails fast if something is missing):

| # | Role | What It Does | Key Variables |
|---|------|--------------|---------------|
| 1 | `java` | Installs Java 21 (Amazon Corretto) — Jenkins runtime | `java_package` |
| 2 | `docker` | Installs Docker, starts the service (restart **handler**) | `docker_package` |
| 3 | `aws-tools` | Installs AWS CLI v2, kubectl, Git, Terraform, the **Flux CLI** (for GitOps), and **Trivy** (image scanner) | `kubectl_version`, `trivy_version` |
| 4 | `jenkins` | Installs Jenkins, adds it to the `docker` group, configures a custom temp dir, installs Python 3 (reload/restart **handlers**) | `jenkins_repo_url`, `jenkins_gpg_key`, `jenkins_user`, `jenkins_tmp_dir` |

> **Why a custom Jenkins temp dir?** The `jenkins` role moves Jenkins' temp directory from `/tmp` (a small RAM-backed `tmpfs`) to `/var/tmp/jenkins` on disk. This prevents out-of-memory errors in the Jenkins UI during large builds.

---

## Repository Structure

```
cloudpulse-ansible/
├── setup-jenkins.yml          # Main playbook — runs all 4 roles in order
├── inventory.ini              # Target hosts (Jenkins server IP goes here)
├── group_vars/
│   └── jenkins.yml            # Host-level variables (e.g. python interpreter)
└── roles/                     # Roles run in this order: java → docker → aws-tools → jenkins
    ├── java/
    │   ├── tasks/main.yml
    │   └── defaults/main.yml
    ├── docker/
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml   # Restart Docker on config change
    │   └── defaults/main.yml
    ├── aws-tools/
    │   ├── tasks/main.yml
    │   └── defaults/main.yml
    └── jenkins/
        ├── tasks/main.yml
        ├── handlers/main.yml   # Reload/restart Jenkins on config change
        └── defaults/main.yml
```

---

## Usage

### Step 1 — SSH into the Ansible server

(IP comes from the `cloudpulse-bootstrap` output)

```bash
ssh -i "C:\Users\Admin\.ssh\cloudpulse-key.pem" ec2-user@<ANSIBLE_PUBLIC_IP>
```

### Step 2 — Clone this repo on the Ansible server

```bash
cd /home/ec2-user/
git clone https://github.com/rajeshdangi409/cloudpulse-ansible.git
cd cloudpulse-ansible/
```

### Step 3 — Point the inventory at your Jenkins server

```bash
sed -i 's/<JENKINS_EC2_IP>/<JENKINS_PUBLIC_IP>/' inventory.ini
```

### Step 4 — Copy the SSH key to the Ansible server

Ansible needs the `.pem` key to SSH into the Jenkins server. From your **local machine** (new terminal):

```bash
scp -i "C:\Users\Admin\.ssh\cloudpulse-key.pem" \
    "C:\Users\Admin\.ssh\cloudpulse-key.pem" \
    ec2-user@<ANSIBLE_PUBLIC_IP>:~/.ssh/cloudpulse-key.pem
```

Then back on the Ansible server:

```bash
chmod 400 /home/ec2-user/.ssh/cloudpulse-key.pem
```

### Step 5 — Run the playbook

```bash
# Test connectivity first
ansible jenkins -i inventory.ini -m ping

# Run the full setup
ansible-playbook -i inventory.ini setup-jenkins.yml
```

---

## Inventory File

```ini
[jenkins]
<JENKINS_PUBLIC_IP> ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/cloudpulse-key.pem

[all:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

> `StrictHostKeyChecking=no` skips the interactive "are you sure you want to connect?" prompt — useful for automation against fresh servers.

---

## Verifying the Setup

After the playbook finishes, SSH into the **Jenkins server** and confirm:

```bash
ssh -i ~/.ssh/cloudpulse-key.pem ec2-user@<JENKINS_PUBLIC_IP>

java -version          # Java 21
docker --version       # Docker
sudo systemctl status jenkins   # active (running)
aws --version          # AWS CLI v2
kubectl version --client
terraform -version
```

Then open the Jenkins UI:

```
http://<JENKINS_PUBLIC_IP>:8080
```

Get the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

## Next Step

Once Jenkins is configured (plugins installed, pipelines created), proceed to **Phase 3**:

➡️ **[cloudpulse-infra](https://github.com/rajeshdangi409/cloudpulse-infra)** — Jenkins runs Terraform to build the EKS cluster, VPC, and ECR.

---

## Idempotency

Ansible is **idempotent** — running the playbook multiple times is safe. Already-installed packages and already-applied changes are skipped, so re-running only fixes what's missing.
