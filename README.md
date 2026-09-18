*This project has been created as part of the 42 curriculum by andjajas*

# DESCRIPTION

In this project I created my first virtual machine (VM) in VirtualBox using specific instructions and how to set up the latest Debian operating system inside, while implementing strict rules. All of the bonus section is also implemented.<br>

<br>

## OPERATING SYSTEM CHOICE:

My choice has fallen on Debian over Rocky Linux, because I'm new at system administration and this is
the highly recommended beginner-friendly option. Debian has a large, mature community, extensive documentation, and a package ecosystem (apt) that is widely used and well-documented, which made troubleshooting easier while learning the material.<br>
Rocky Linux is a Red Hat Enterprise Linux (RHEL) clone which has more predictable long-term support cycles.
The disadvantage of Rocky Linus is that there is a steeper learning curve, especially with SELinux.<br>

I found Debian's documentation and community support easier to navigate as a first-time sysadmin, and AppArmor's simpler, path-based policy model was more approachable than configuring SELinux's more granular but complex labeling system, which I witnessed from peers that implemented Rocky Linux in this project.<br>

### Debian vs Rocky Linux

| | Debian | Rocky Linux |
|---|---|---|
| Package manager | `apt` (used directly), built on `.deb` packages | `dnf`/`yum`, built on `.rpm` packages |
| Release philosophy | Community-driven, very stable, slower release cycle | Enterprise-focused (RHEL-compatible clone), predictable long-term support cycles |
| Security module | AppArmor (path-based, simpler policy model) | SELinux (label-based, more granular but more complex to configure) |
| Firewall tool | UFW (Uncomplicated Firewall, simple syntax) | firewalld (zone-based, more complex but more flexible) |
| Typical use case | General-purpose servers, desktops, widely used for beginners | Enterprise/production servers requiring RHEL compatibility |
| Beginner-friendliness | Generally considered easier to start with | Steeper learning curve, especially with SELinux |

### AppArmor vs SELinux

Both AppArmor and SELinux are Linux Security Modules (LSMs) that enforce Mandatory Access Control (MAC) — restricting what a process can do beyond standard Unix permissions, even if it's running as root.

AppArmor (used on Debian, as required by the subject) works by attaching a security profile to a specific program's file path. Each profile is a relatively human-readable list of what that program is allowed to access (files, network, capabilities). Because it's tied to paths rather than the files themselves, it's generally considered easier to read and write policies for, which made it more approachable while I was still learning the underlying concepts.

SELinux (used on Rocky) takes a different approach: it labels every file, process, and resource with a security context, and enforces access rules based on those labels rather than file paths. This is more granular and flexible — a file can be moved and keeps its label/protection — but the label-based model is also more complex to reason about and configure correctly, which the subject itself notes ("Setting up Rocky is quite complex").

I confirmed AppArmor was running with:

sudo aa-status | less

which reports the AppArmor module is loaded and lists profiles in enforce/complain mode.

Summary: AppArmor trades some flexibility for simplicity — a reasonable tradeoff for a project focused on learning fundamentals rather than fine-grained enterprise policy management.

### UFW vs firewalld

Both are front-end tools that manage the same underlying kernel firewall functionality (netfilter/iptables or nftables), just with different models and syntax.

UFW (Uncomplicated Firewall, used on Debian) is designed around simplicity: rules are added with short, direct commands like sudo ufw allow 4242, and the tool maintains a simple ordered rule list. This made it fast to set up exactly what the subject requires — allowing only port 4242 and denying everything else by default.

firewalld (used on Rocky) is built around the concept of zones — predefined trust levels (e.g. public, internal, trusted) that group sets of rules together. Instead of adding a raw port rule directly, you typically assign an interface or source to a zone and then open ports/services within that zone. This is more flexible for complex network setups (multiple interfaces with different trust levels) but requires understanding the zone model first, which adds a layer of complexity not needed for a single-interface VM like this project's.

I confirmed UFW was active with:

sudo ufw status verbose

which shows the firewall active, default-deny incoming, and only port 4242 (SSH) and port 80 (lighttpd, bonus) explicitly allowed.

Summary: for a single-purpose VM with one network interface, UFW's flat rule list was more direct to set up and verify than configuring firewalld's zone-based model would have been.

## VirtualBox vs UTM

I used VirtualBox, since it's available and works reliably on the machine I used at Codam. VirtualBox is cross-platform (Windows/Linux/macOS Intel), open-source, and has a long track record for this kind of project. UTM is primarily aimed at macOS (especially Apple Silicon, where VirtualBox has historically had limited support), using Apple's built-in virtualization framework — it's the subject's recommended alternative specifically for Mac M1/M2 users where VirtualBox doesn't run natively.

<br>

## DESIGN CHOICES

### Partitioning, Security & Storage Policy

We split the hard drive into separate sections (partitions) so that if one part breaks or fills up, it doesn't crash the whole computer.
The disk was partitioned using LVM on top of a LUKS-encrypted physical partition, following the bonus structure (splitting beyond the mandatory minimum of 2 encrypted partitions):

### Why each part is separated:

* **/boot (476M):** Left unlocked so the computer can physically turn on and ask for your password before decrypting the rest of the drive.
* **/ (9.31G):** The brain. Holds the main operating system files. 
* **swap (2.14G):** Backup memory for when the computer runs out of RAM.
* **/home (4.66G):** Your personal files. Separated so you can wipe the OS later without losing your homework/data.
* **/var (2.79G):** Holds files that constantly change, like background app files and databases.
* **/srv (2.79G):** Where web server files live if this computer hosts a website.
* **/tmp (2.79G):** Temporary files. Locked down so hackers can't sneak hidden apps in here and run them.
* **/var/log (3.72G):** System history logs. Separated so if an app goes crazy and writes a billion logs, it doesn't freeze the main system.

When setting up the partitions size values were assigned with the numbers shown in the Born2beroot project's subject (page13).
Small amounts get consumed by LUKS/LVM overhead before the actual logical volume was created. This is why the numbers shown in the VM deviate slightly from the subject example, this is normal, expected overhead cost. As stated in the subject: "The example shows arbitrary disk sizes. You need to determine the appropriate size for each partition to ensure proper operation while avoiding unnecessary disk usage.".

### User management

We enforce strict rules on who can access the system and what they are allowed to do. This keeps user data isolated and prevents accidental system-wide damage.

#### Access Rules

* **The Root Account:** Disabled for direct login. All administrative tasks must use `sudo`. This creates an audit trail of who ran what command.
* **Standard Users:** Limited to their own `/home/` directories. Users cannot view, modify, or delete files belonging to other users.
* **System Accounts:** Non-human accounts (like `www-data` for web servers) are stripped of login shells (`/bin/false`) so they cannot be hijacked to log into the machine.

#### Password & Authentication Rules

* **No Passwords for SSH:** Remote login via password is completely disabled. Users must use secure **SSH Keys** to connect.
* **Sudo Timeout:** `sudo` privileges expire after 5 minutes of inactivity, requiring the user to re-enter their password to prevent unauthorized access to unattended screens.
* **Password Complexity:** Local passwords must be at least 12 characters long and include a mix of letters, numbers, and symbols to block brute-force attacks.

### Services Installed

The following services were deliberately installed to meet the subject's requirements, keeping the install minimal (no graphical interface at any point):

* openssh-server — provides remote access to the VM via SSH, configured to listen on port 4242 (not the default 22) with root login disabled.<br>
* ufw — firewall, configured to deny all incoming traffic by default and explicitly allow only the ports required by running services (4242 for SSH, 80 for lighttpd).<br>
* sudo — installed and configured with a custom policy (limited password attempts, full input/output logging, restricted secure_path) rather than relying on direct root access.<br>
* libpam-pwquality — enforces the password complexity policy (minimum length, character variety, repetition limits) required alongside /etc/login.defs's expiry settings.<br>

Bonus services, for the WordPress site and additional hardening:<br>

* lighttpd, mariadb-server, php — the minimal web stack required to serve a functional WordPress installation, chosen because it's explicitly specified by the subject over more common alternatives like NGINX/Apache.<br>
* fail2ban — added as the "service of your choice," monitoring SSH login attempts and automatically banning IPs after repeated failed attempts, to protect against brute-force attacks that the password policy alone doesn't prevent.<br>

Each service was chosen because it directly serves a specific subject requirement, rather than installing anything beyond what the mandatory and bonus parts call for — consistent with the subject's instruction to keep the server minimal.<br>

<br>
<br>

# INSTRUCTIONS
Compilation
* This project doesn't involve compiling any code — the deliverable is a fully configured virtual machine, not a compiled binary.

Installation
* Install VirtualBox (or UTM if VirtualBox is unavailable, e.g. on Apple Silicon Macs).
Create a new VM and install the latest stable release of Debian, following the encrypted LVM partitioning scheme described above.
Once the base install is complete, boot the VM and configure it following the steps below.

Configuration / Setup

* Run through the following, in order, on the freshly installed Debian VM:

* Install sudo, configure it via /etc/sudoers.d/, and add a non-root user to the sudo and user42 groups.
Configure SSH (/etc/ssh/sshd_config): change the port to 4242, disable root login, restart the service.
Install and configure the UFW firewall, allowing only port 4242 (and port 80 if the WordPress bonus is set up).
Configure the password policy in /etc/login.defs and /etc/pam.d/common-password.
Change all account passwords (including root's) to comply with the new policy.
Place monitoring.sh in /usr/local/bin/ and schedule it in root's crontab (*/10 * * * * and @reboot).
(Bonus) Install lighttpd, mariadb-server, and php, and set up WordPress in /var/www/html.
(Bonus) Install and configure fail2ban to protect SSH against brute-force attempts.

Execution

* Once configured, the VM runs as a standalone server — there's no script to launch manually:

* Power on the VM in VirtualBox/UTM. monitoring.sh runs automatically at boot and every 10 minutes via cron, broadcasting system stats to all logged-in terminals via wall.
To access the machine remotely, set up port forwarding in VirtualBox (Settings → Network → Advanced → Port Forwarding), then connect with:
  ssh <username>@localhost -p <host_port>
(Bonus) With host port 8081 forwarded, the WordPress site is reachable in a browser at http://localhost:8081/.
Guest port was set to 80.
<br>

System Administration & Inspection Commands

* Use these exact commands to verify system status, manage users, monitor networking, and audit security configurations.

#### 1. System Identity & OS Verification
* **Check OS Version:** `head -n 2 /etc/os-release`
* **Check Current Hostname:** `hostname` or `hostnamectl`
* **Change Hostname:** `sudo hostnamectl set-hostname <newhostname>`
* **View Network Host Mappings:** `cat /etc/hosts`

#### 2. Disk, Partition & LVM Auditing
* **View Block Devices & Layout:** `lsblk`
* **Check Logical Volume (LVM) Status:** `sudo lvs`
* **Check Partition Disk Usage:** `df -h`

#### 3. User, Group & Password Policy Management
* **Switch to Root / Change Password:** Use `su` to switch user, and `passwd` to update passwords.
* **Identify Current Active Users:** `whoami` (current shell) and `who` (all logged-in sessions).
* **List All Human Users (UID >= 1000):** `awk -F: '$3 >= 1000 {print $1}' /etc/passwd`
* **Check Group Memberships:** `getent group sudo`, `getent group user42`, or type `groups` for your own.
* **Create New Users & Groups:** 
  ```bash
  sudo adduser <newusername>
  sudo addgroup <newgroupname>
  sudo adduser <username> <newgroupname>
  ```
* **Audit Password Aging & Complexity Rules:** Read policy files with `sudo less /etc/login.defs` and `sudo cat /etc/pam.d/common-password`.

#### 4. Sudo Security & Auditing
* **Verify Sudo Installation & Version:** `dpkg -l | grep sudo`, `which sudo`, and `sudo -V`.
* **Add a User to Sudo Group:** `sudo adduser <username> sudo`
* **Audit Sudo Privilege Logs:** Inspect logs with `sudo ls -la /var/log/sudo` and `sudo cat /var/log/sudo/sudo_log`.

#### 5. Firewall Management (UFW)
* **Check Firewall Status:** `dpkg -l | grep ufw`, `sudo ufw status verbose`, and `sudo systemctl status ufw`.
* **Open & Close Specific Ports (e.g., 8080):**
  ```bash
  sudo ufw allow 8080
  sudo ufw delete allow 8080
  ```

#### 6. SSH Service Auditing
* **Verify SSH Server & Service Status:** `dpkg -l | grep ssh` and `sudo systemctl status ssh`.
* **Check Active Listening SSH Ports:** `sudo ss -tunlp | grep ssh`

#### 7. Automation & Background Tasks
* **Edit Root Cron Jobs:** `sudo crontab -u root -e` (used to schedule recurring security scripts or updates).

#### 8. For monitoring.sh
* to read the file: `less /usr/local/bin/monitoring.sh`
* `lscpu -p | grep -v '#' | sort -u -t, -k3,3 | wc -l` <br>
To get the physical number of cores (=Sockets), you cannot actually test it inside the VM, but outside it (and only on the computers which have a different number of virtual cores than the physical core only the computers on floor 1 inner circle (12 physical cores and 20 virtual cores outside the VM) all other computers have 4 physical cores and 4 virtual cores, so it won't distinguish).<br>

#### 9. signature
* to get the signature goto the folder where the .vdi file is located and use:
`sha1sum <vm_name>.vdi`

<br>
<br>

# RESOURCES

My peers at Codam: vcoevert, olistokes, alkhan, yuhma<br>

[Born2beroot guide: bit outdated, contains small errors](https://noreply.gitbook.io/born2beroot)<br>

[why number of physical processors is the number of physical sockets](https://docs.edisglobal.com/faq/upgrades-downgrades/cpu-cores-vs-cpu-sockets-in-vps-environments/understanding-cpu-cores-vs-sockets-in-vps)<br>

[lscpu](https://www.geeksforgeeks.org/linux-unix/gathering-system-information-using-commands-like-lscpu-lspci-and-lsblk/)<br>

#### AI Usage
I used Claude (Anthropic) throughout this project as a learning aid, following 42's guidelines on AI use — asking for explanations and guided reasoning rather than direct answers or copy-pasteable solutions.<br>
Specific ways I used it:<br>

Debugging and verification:<br> when a guide's suggested command produced unexpected results (e.g. counting CPU sockets vs. cores using lscpu -p), I used Claude to reason through why a command was wrong rather than being handed the fix directly — I identified the bug myself with a peer, then worked through the corrected sort/awk logic with Claude's guidance.<br>
Line-by-line script explanation:<br> for monitoring.sh, I had Claude walk me through what each command and flag does (awk, sort -k, vmstat, ss, journalctl, etc.) so I could explain the script's logic during my peer evaluation, rather than running a script I didn't understand.<br>
Conceptual explanations:<br> for topics I didn't fully understand (LVM, sockets vs. cores vs. threads, su vs sudo, how SSH port forwarding works through VirtualBox NAT), I asked Claude to explain the underlying concepts, often using analogies, until I could restate them in my own words.<br>
Troubleshooting:<br> for issues like a Fail2ban config conflict (duplicate [sshd] section) and a port-forwarding connection failure, I used Claude to help me systematically test and isolate the actual cause, rather than being told the fix outright.<br>

All actual configuration decisions, file edits, and commands were typed and executed by me inside the VM. Claude was not given access to the VM and did not write or apply any configuration directly.
