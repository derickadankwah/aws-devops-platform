# Phase 1 — Secure Linux Server Baseline on AWS EC2

> Part of the **Kestrel Platform**, a training project in which I build an enterprise-style cloud platform phase by phase while studying DevOps and Cloud Engineering. Kestrel Logistics is a fictional company used as the training scenario; all infrastructure and evidence below is real work I performed in my own AWS account.

## Business scenario

Kestrel Logistics' delivery-tracking application runs on a developer's laptop and goes offline whenever that laptop sleeps. Before moving anything to the cloud, the CTO asked for a secure, documented, cost-controlled Linux server baseline on AWS. This phase delivers that baseline: a locked-down AWS account, a hardened access path, a verified Amazon Linux 2023 server, and a documented shutdown procedure.

## Architecture

```
                        ADMIN LAPTOP (macOS)
                  ┌───────────────────────────┐
                  │ OpenSSH client            │
                  │ ~/.ssh/kestrel-dev-key.pem│  (permissions 400)
                  │ ~/.ssh/config → kestrel-dev│
                  └────────────┬──────────────┘
                               │ SSH, TCP 22, from <admin-ip>/32 only
                               ▼
   ┌───────────────── AWS ACCOUNT (root locked with MFA) ─────────────────┐
   │  IAM user derick-admin (MFA, no access keys)                         │
   │  AWS Budgets: zero-spend + monthly cost budget → email alerts        │
   │                                                                      │
   │  Region: us-east-1 (N. Virginia) · AZ us-east-1d                     │
   │  ┌──────────────────── Default VPC (172.31.0.0/16) ───────────────┐  │
   │  │  ┌──────────── Security group: kestrel-ssh-sg ──────────────┐  │  │
   │  │  │  Inbound: TCP 22 from <admin-ip>/32 only                 │  │  │
   │  │  │  ┌────────────────────────────────────────────────────┐  │  │  │
   │  │  │  │ EC2: kestrel-dev-01                                │  │  │  │
   │  │  │  │ Amazon Linux 2023 · t3.micro · 8 GiB gp3 (NVMe)    │  │  │  │
   │  │  │  │ Tags: Project=kestrel, Env=dev, Owner=Derick       │  │  │  │
   │  │  │  └────────────────────────────────────────────────────┘  │  │  │
   │  │  └──────────────────────────────────────────────────────────┘  │  │
   │  └────────────────────────────────────────────────────────────────┘  │
   └──────────────────────────────────────────────────────────────────────┘
```

## What I built

| Component | Configuration | Why |
|---|---|---|
| Root user | MFA enabled (virtual authenticator) | Root has unrestricted power and cannot be limited by IAM policies |
| IAM user | `derick-admin`, AdministratorAccess, console-only, MFA, **no access keys**, forced password reset on first login | Daily work through an identity whose permissions can be limited, monitored and revoked |
| Billing access | IAM access to billing confirmed (Budgets opened as `derick-admin`) | IAM users are blocked from billing pages by default, even with admin rights |
| Budgets | Zero-spend budget ($1) and monthly cost budget ($20) with email alerts | Detect unexpected charges early |
| Key pair | `kestrel-dev-key`, ED25519, `.pem`, stored in `~/.ssh` with `400` permissions | Modern algorithm; private key readable only by me |
| Security group | `kestrel-ssh-sg`: inbound SSH (TCP 22) from `<admin-ip>/32` only; default outbound | Least privilege; SSH is not exposed to the internet |
| EC2 instance | `kestrel-dev-01`, t3.micro, AMI `al2023-ami-2023.12.20261012.0-kernel-6.18-x86_64`, us-east-1d, default VPC, 8 GiB gp3 root volume | Small, low-cost, AWS-optimized Fedora-family server |
| Tags | `Name=kestrel-dev-01`, `Project=kestrel`, `Env=dev`, `Owner=Derick` | Separate this project's resources from others in the same account for cost tracking and safe cleanup |
| SSH config | `Host kestrel-dev` alias with `IdentitiesOnly yes`; existing Colima `Include` line preserved | Repeatable, mistake-resistant connections |

## Security decisions

- **Root MFA:** a leaked root password alone can no longer take over the account.
- **IAM user for daily work:** limits the blast radius if credentials are compromised; access can be revoked.
- **No access keys yet:** long-lived programmatic keys are a common leak source; I'll create them only when a tool needs them.
- **SSH restricted to `/32`:** only my IP can reach port 22. `0.0.0.0/0` would invite automated login attempts from the entire internet.
- **Key permissions `400`:** SSH refuses private keys that other users can read.
- **Host fingerprint verified before first connection:** I compared the ED25519 fingerprint from the EC2 system log with the one shown by SSH before typing `yes`, to guard against man-in-the-middle impersonation.
- **Reviewed security group reused:** I selected `kestrel-ssh-sg` instead of the launch wizard's default, which would have opened SSH to `0.0.0.0/0`.

## Server baseline (verified from inside the server)

| Item | Value | Command used |
|---|---|---|
| Distro | Amazon Linux 2023 (`2023.12.20261012`), `ID_LIKE="fedora"`, support ends 2029-06-30 | `cat /etc/os-release` |
| Kernel | `6.18.54-121.166.amzn2023.x86_64` | `uname -r` |
| Architecture | `x86_64` | `uname -m` |
| Login shell | `/bin/bash` | `echo $SHELL` |
| CPUs | 2 vCPUs, Intel Xeon Platinum 8259CL @ 2.50GHz | `nproc`, `head -5 /proc/cpuinfo` |
| Memory | 913 MiB total (of 1 GiB advertised), no swap | `free -h` |
| Root disk | `/dev/nvme0n1p1`, 8.0 G, 20% used | `df -h /` |
| Block devices | `nvme0n1` with partitions `p1` (/), `p127` (1M), `p128` (/boot/efi) | `lsblk` |
| Package manager | dnf 4.14.0 (rpm 4.16.1.3 underneath) | `dnf --version` |
| Hostname | `ip-172-31-45-214.ec2.internal` (private, not internet-reachable) | `hostname` |

**Observations**
- The AWS console names the root volume `/dev/xvda`, but Linux sees `nvme0n1`: on Nitro-based instances, EBS volumes are presented as NVMe devices. Always confirm device names with `lsblk`.
- Total memory is below the advertised 1 GiB because the kernel and reserved memory are taken before user programs run.
- No swap: if memory runs out, the kernel's OOM killer terminates processes. Important when deploying applications on this instance size.
- Server time is UTC; my local time is UTC−4.

## Filesystem findings

- **usrmerge:** `/bin -> usr/bin`, `/sbin -> usr/sbin`, `/lib -> usr/lib`, `/lib64 -> usr/lib64` are symlinks kept for compatibility with older scripts.
- **Root's home is `/root`**, not under `/home`; permissions `dr-xr-x---`, so it needs `sudo` to read.
- **`/proc` and `/sys` are virtual:** `/proc/cpuinfo` shows size 0, yet prints CPU details, because the kernel generates the content when the file is read.
- **Logs:** no `/var/log/messages` on AL2023; system logs live in the systemd journal (`/var/log/journal`, read with `journalctl`). Also present: `cloud-init.log`, `dnf.log`, `audit/`.
- **Audit trail:** every `sudo` command I ran was recorded in the journal with user, terminal, working directory and full command.
- **`/tmp` sticky bit:** `drwxrwxrwt`; everyone can write, but users can only delete their own files.
- **Commands:** `cd` is a shell builtin; `ls` is an alias for `ls --color=auto` that runs the external program `/usr/bin/ls`.
- The trailing `.` in permission strings indicates an SELinux security label.

## Problems I hit and how I solved them

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | Assigning MFA to `derick-admin` failed: "Authentication code for device is not valid", plus a "You need permissions" banner | Rejected codes, most likely from the wrong authenticator entry or non-consecutive codes; the banner was a side effect of the failed request | Deleted the failed entry, rescanned a fresh QR code under a new device name, entered two consecutive codes; verified by an MFA prompt at `derick-admin` sign-in |
| 2 | Budget showed $68.78 spent against a $10 budget (687%) | Expected spend from another project in the same AWS account | Investigated before acting (no deletions), confirmed the source, raised the budget to $20 so alerts reflect unusual spend; tagging used to separate projects |
| 3 | `ssh` failed: "Identity file ... not accessible" and "Could not resolve hostname" | Relative key path (key had moved to `~/.ssh`) and an incomplete DNS name | Used the full key path and the public IP; handled each error separately |
| 4 | `ssh kestrel-dev` appeared to do nothing | I ran it from inside the EC2 server instead of my Mac | Read the shell prompt to identify the machine, exited, reran from the Mac |
| 5 | Editor save prompt showed stray characters after the filename | Possible extra characters typed after Ctrl+O | Cleared them before saving; confirmed with `ls ~/.ssh` that no stray file existed |

**Noted for later:** OpenSSH warns that the connection does not use a post-quantum key exchange. The session is still encrypted; hardening the server's SSH configuration is planned for the security phase.

## Cost considerations

- Account is on the AWS credit-based free plan; usage draws down credits shared with another project.
- Approximate us-east-1 on-demand prices: t3.micro about $0.0104/hour, public IPv4 about $0.005/hour, gp3 storage about $0.08/GB-month (about $0.64/month for 8 GiB). Roughly $0.37/day while running.
- **Stop** halts compute charges and keeps the disk (still billed) and data; the public IP changes on restart. **Terminate** permanently deletes the instance and, by default, its root volume.
- Practice: stop the instance at the end of every session.

## What I learned

## What I learned

- **Builtin vs external commands:** `type cd` showed that `cd` is a shell builtin, because changing directory changes the shell's own location, so Bash has to do it itself. `ls` is a separate program (`/usr/bin/ls`) that the shell starts for me, and on this server it is wrapped in an alias that adds colours. Aliases only exist in my interactive shell, so scripts may behave differently.
- **Why `/proc` files show size 0:** `/proc` is not stored on the disk. When I read a file like `/proc/cpuinfo`, the kernel creates the content at that moment, so there is nothing saved and the size shows 0.
- **The sticky bit on `/tmp`:** the `t` at the end of `drwxrwxrwt` means everyone can create files in `/tmp`, but each user can only delete or rename their own files.
- **Stop vs terminate:** stopping keeps the instance and its disk, so I can start it again, but the disk still costs money and the public IP changes. Terminating deletes the instance permanently, and by default its disk too.
- **Choosing my own security group:** the launch wizard's default would have opened SSH to the whole internet (`0.0.0.0/0`). My group only allows my own IP (`/32`), which follows least privilege.
- **Reading the prompt:** I once ran a command on the server when I meant to run it on my Mac. Now I check the prompt before every command to know which machine I am on.
- **Reading errors one at a time:** one SSH command gave two different errors (a wrong file path and an incomplete hostname). Fixing each separately was faster than guessing.
- **Evidence over claims:** saying "it's done" is not proof. Showing the output, the setting, or a test result is.
- **Investigate before deleting:** when my budget showed unexpected spend, I identified the source first instead of deleting resources, because deletion cannot be undone.
- **Billing is not real time:** AWS cost and credit data can take hours to update, so "no charge yet" does not mean "no charge".

## What I would improve

- Replace SSH keys with **AWS Systems Manager Session Manager** (no open port 22, access governed by IAM and logged).
- Configure post-quantum key exchange on the server's SSH daemon.
- Use **separate AWS accounts** per project or environment instead of sharing one account.
- Use **IAM Identity Center** instead of a long-lived IAM user.
- Add swap or choose a larger instance before running memory-hungry applications.
- Automate provisioning with **Terraform** (planned for Phase 9).
