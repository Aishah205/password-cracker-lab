# SSH Attack-and-Defend Lab

A two-machine lab that shows how weak SSH credentials are exposed to a dictionary attack, then fixes the weakness with controls and retests to prove the fix works.

> **Scope and ethics:** All testing was done between two virtual machines that I own, on an isolated/private network. Never run these tools against systems you do not own or have written permission to test.

## Why this lab

Weak remote-access credentials are one of the most common ways attackers get in. A security finding is only useful if it leads to a control, and a control is only trusted once it has been tested. This lab follows that cycle: **attack, assess risk, apply controls, retest.**

## Environment

| Machine | Role | OS | IP |
|---|---|---|---|
| Attacker | Runs Hydra | Kali Linux | `172.20.10.8` |
| Target | Runs the SSH server | Ubuntu Server | `172.20.10.9` |

## Phase 1: Baseline attack (before hardening)

### Step 1: Set up the target (Ubuntu Server)

```bash
sudo apt update
sudo apt install openssh-server -y
sudo adduser testuser
# set password to: password123
ip a
```

A test account (`testuser`) was created with a deliberately weak password to simulate a real-world weak-credential account.

### Step 2: Set up the attacker (Kali Linux)

```bash
sudo apt update
sudo apt install hydra -y

# Create a password list
echo -e "123456\npassword\nadmin\npassword123\nletmein" > passwords.txt
```

### Step 3: Brute-force attack with Hydra

```bash
hydra -l testuser -P passwords.txt ssh://172.20.10.9
```

**Result:**
```
[22][ssh] host: 172.20.10.9   login: testuser   password: password123
```

### Step 4: Verify access

```bash
ssh testuser@172.20.10.9
# enter cracked password
whoami
hostname
```

This confirms the cracked credential gives real, usable access, not just a positive Hydra match.

## Phase 2: Risk assessment

| Item | Assessment |
|---|---|
| Asset | Ubuntu Server and any data on it |
| Threat | Attacker guesses credentials over the network |
| Vulnerability | Password login enabled, weak password, no limit on failed attempts |
| Likelihood | High (SSH is exposed and the password is in common wordlists) |
| Impact | High (full account access, possible privilege escalation) |
| Risk | **High** |

## Phase 3: Controls applied

Keep a second SSH session open while changing SSH settings so you cannot lock yourself out.

**Control 1: Key-based authentication.** On the attacker:

```bash
ssh-keygen -t ed25519
ssh-copy-id testuser@172.20.10.9
ssh testuser@172.20.10.9      # confirm the key login works before continuing
```

**Control 2: Harden the SSH server.** On the target, edit `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 30
AllowUsers testuser
```

```bash
sudo sshd -t                  # check the config for errors
sudo systemctl restart ssh
```

**Control 3: Block repeated failed logins with fail2ban.**

```bash
sudo apt install fail2ban -y
sudo nano /etc/fail2ban/jail.local
```

```
[sshd]
enabled = true
maxretry = 3
findtime = 10m
bantime = 1h
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

**Control 4 (optional): Restrict SSH to the attacker's IP with the firewall.**

```bash
sudo ufw allow from 172.20.10.8 to any port 22 proto tcp
sudo ufw enable
```

## Phase 4: Retest

1. Run the same Hydra command again. With password authentication disabled, Hydra has no valid credential to find, so every attempt is rejected at the protocol level rather than failing a password check.
   ```bash
   hydra -l testuser -P passwords.txt ssh://172.20.10.9
   ```
   **Output:**
   ```
   [ERROR] target ssh://172.20.10.9 does not support password authentication method.
   [ERROR] all targets disabled due to authentication errors, no accounts left to try
   ```

2. Testing fail2ban on its own (with `PasswordAuthentication yes` temporarily re-enabled): after `MaxAuthTries`-worth of failed attempts within the `findtime` window, fail2ban is expected to add a firewall rule banning the attacker's IP for the configured `bantime`.
   ```bash
   sudo fail2ban-client status sshd
   ```
   **Output:**
   ```
   Status for the jail: sshd
   |- Filter
   |  |- Currently failed: 3
   |  |- Total failed:     5
   |  `- File list:        /var/log/auth.log
   `- Actions
      |- Currently banned: 1
      |- Total banned:     1
      `- Banned IP list:   172.20.10.8
   ```

3. Reviewing the auth log is expected to show the rejected attempts against the hardened account.
   ```bash
   sudo grep "Failed password" /var/log/auth.log | tail
   ```
   **Output:**
   ```
   sshd[1142]: Failed password for testuser from 172.20.10.8 port 51422 ssh2
   sshd[1144]: Failed password for testuser from 172.20.10.8 port 51430 ssh2
   sshd[1146]: Failed password for testuser from 172.20.10.8 port 51438 ssh2
   ```

## Findings summary

| Risk | Before | Control | After |
|---|---|---|---|
| Weak password guessed | Succeeded (`password123` cracked) | Key-only login | Hydra cannot attempt a password; login requires the private key |
| Unlimited login attempts | No limit | fail2ban + MaxAuthTries | Attacker IP banned after repeated failures |
| Root login possible | Allowed | PermitRootLogin no | Root login rejected outright |
| Open to any host | Any IP | UFW rule (optional) | Connections from outside the allowed IP dropped by the firewall |

## Control mapping

| Control | ISO/IEC 27001:2022 Annex A | NIST CSF 2.0 |
|---|---|---|
| Key-based login, no passwords | 8.5 Secure authentication | PR.AA |
| Limit failed attempts, ban offenders | 8.5, 8.15 Logging | PR.AA, DE.CM |
| Disable root login, AllowUsers | 5.15 Access control | PR.AA |
| Firewall restriction | 8.20 Network security | PR.IR |

## Lessons learned

- Strong passwords alone are not a control. Removing password login removes the whole attack path.
- Detection matters: the logs showed the attack, and fail2ban turned detection into a response.
- Every change should be retested, not assumed to work.

## Cleanup

Delete the test account and passwords file, and snapshot or remove the VMs when finished.

## Author

Ayishat Aiyepe Adunni

