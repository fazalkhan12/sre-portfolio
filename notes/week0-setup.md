# Week 0 Setup Log — 25 Sep 2026

Reference for everything set up on day one of the UAE/Gulf SRE study plan (AWS track + Jenkins + Ansible).

## Decisions made

| Decision | Choice | Why |
|---|---|---|
| Cloud track | AWS (SAA first) | Stronger plan structure, first cert by Week 7, reuses existing AWS account |
| Study tracker | `UAE_SRE_Study_Tracker_AWS.xlsx` | 25 weeks, 28 Sep 2026 → 21 Mar 2027, 9 hrs/week |
| Lab terminal | AWS CloudShell (Mumbai) | Corporate laptop has install limits and SSL inspection; Mac not available |
| Portfolio host | GitHub (`fazalkhan12/sre-portfolio`) | Recruiters look at GitHub; GitLab added as a second push remote in Week 14 |
| Region | Asia Pacific (Mumbai) `ap-south-1` | Closest to Pune |

## 1. Account security

**Root user MFA** — already enabled (virtual device `Authapp`). Root has **no access keys**; keep it that way.

**IAM admin user**

- User: `fazal-admin`, console access, custom password, no forced reset
- Group: `admins` with the AWS managed policy `AdministratorAccess`
- Sign-in URL: `https://243478202579.signin.aws.amazon.com/console`
- MFA on `fazal-admin`: **skipped for now** (can be added under IAM → Users → fazal-admin → Security credentials)

**Billing access for IAM users** — activated once as root:
Account menu → Account → *IAM user and role access to Billing information* → Edit → Activate IAM Access.

> Rule from now on: use `fazal-admin` for everything. Root only for billing/account-level changes.

## 2. Budget and cost control

**Budget `monthly-lab-budget`** (Customize → Cost budget)

- Monthly, recurring, fixed, **$30**
- Scope: filter **Charge type → Exclude → Credit**
  (otherwise credits make cost show $0 and alerts never fire)
- Metric: **Unblended costs** (not *Net* unblended — that re-applies credits)
- Alerts: **$10 actual**, **$30 actual**, **$30 forecasted** → email

**Cost investigation**

- Bills page shows $0.00 per service because credits net everything out → use **Cost Explorer** instead.
- Cost Explorer settings that show real usage: Group by *Service* (or *Usage type*), Charge type **Exclude Credit**, date range covering the current month.
- Finding: **EC2-Other ≈ $4.35 (Aug), ≈ $7.30 Sep-to-date**, forecast ≈ $9/month.
- Cause: one **100 GiB gp3 EBS volume** from the old EC2 box. EBS is billed even when the instance is stopped.
- Fix: terminated the instance; confirmed **0 volumes, 0 owned snapshots, 0 Elastic IPs**.

> Lesson: size lab disks at 20 GiB (≈ $2/month). Stopping an instance does not stop disk charges.

Credits remaining at time of writing: **≈ US$107**.

## 3. CloudShell toolchain

Open: console top bar → `>_` icon (region: Mumbai). Only the home folder persists (1 GB); tools live in `~/bin`.

```bash
# identity check
aws sts get-caller-identity

# personal bin on PATH
mkdir -p ~/bin && echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc && source ~/.bashrc

# Terraform
cd /tmp && curl -sLO https://releases.hashicorp.com/terraform/1.16.4/terraform_1.16.4_linux_amd64.zip \
  && unzip -o terraform_1.16.4_linux_amd64.zip terraform -d ~/bin && cd ~

# kubectl (latest stable, includes kustomize)
cd /tmp && curl -sLO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
  && chmod +x kubectl && mv kubectl ~/bin/ && cd ~

# Helm 3
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | HELM_INSTALL_DIR=$HOME/bin USE_SUDO=false bash

# eksctl
cd /tmp && curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" \
  && tar -xzf eksctl_Linux_amd64.tar.gz -C ~/bin && cd ~
```

**Installed versions**

| Tool | Version | Source |
|---|---|---|
| AWS CLI | 2.36.50 | Pre-installed |
| Python | 3.13.15 | Pre-installed |
| git | 2.50.1 | Pre-installed |
| jq | 1.8.1 | Pre-installed |
| Terraform | 1.16.4 | `~/bin` |
| kubectl | 1.37.1 (Kustomize 5.8.1) | `~/bin` |
| Helm | 3.22.0 | `~/bin` |
| eksctl | 0.230.0 | `~/bin` |

Home folder usage after install: **379 MB of 1 GB**.

**Verify everything in one line**

```bash
python3 --version; git --version; jq --version; aws --version; terraform version | head -1; kubectl version --client | head -1; helm version --short; eksctl version; du -sh ~
```

**Known limits and plans**

- Terraform's AWS provider is several hundred MB → in Week 8, point Terraform's data dir to `/tmp` so it doesn't fill the home folder.
- No Docker, sessions time out after ~20–30 min idle → Ansible and Jenkins (Weeks 12–15) move to a small EC2 workstation (20 GiB disk, stopped when idle).
- CloudShell deletes home-folder data after a long period of no use (AWS states 120 days) → rerun the commands above if tools disappear.

## 4. GitHub portfolio

**Repo:** https://github.com/fazalkhan12/sre-portfolio (public, with README)

**SSH key from CloudShell**

```bash
ssh-keygen -t ed25519 -C "cloudshell-sre-portfolio" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub   # added to GitHub → Settings → SSH keys, title "AWS CloudShell"
ssh -T -o StrictHostKeyChecking=accept-new git@github.com   # "Hi fazalkhan12!"
```

**Clone and structure**

```bash
git config --global user.name "Fazal Khan"
git clone git@github.com:fazalkhan12/sre-portfolio.git && cd sre-portfolio
mkdir -p notes/postmortems terraform ansible python observability
touch notes/postmortems/.gitkeep terraform/.gitkeep ansible/.gitkeep python/.gitkeep observability/.gitkeep
git add . && git commit -m "Set up portfolio structure" && git push origin main
```

```
sre-portfolio/
├── README.md
├── notes/
│   └── postmortems/
├── terraform/
├── ansible/
├── python/
└── observability/
```

**Week 14 (GitLab CI):** add GitLab as a second push destination so one `git push` updates both:

```bash
git remote set-url --add --push origin git@gitlab.com:<you>/sre-portfolio.git
```

## 5. Pending items

- [ ] Linux Foundation ticket: CKS eligibility with expired CKA (needed by ~Week 21)
- [ ] CV: show CKA as "expired [year]" rather than current
- [ ] Optional: MFA on `fazal-admin`
- [ ] Tracker: mark Week 0 Theory / Lab / Done-when = Y, log today's hours

## 6. Next session

**Week 1 — IAM (starts Mon 28 Sep 2026)**
Day 1 topic: IAM policy evaluation (explicit deny → allow → implicit deny), with a short CloudShell exercise.
