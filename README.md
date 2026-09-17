*This project has been created as part of the 42 curriculum by andjajas*

# DESCRIPTION

In this project I created my first virtual machine (VM) in VirtualBox using specific instructions and how to set up the latest Debian operating system inside, while implementing strict rules. All of the bonus section is also implemented.<br>

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

UFW vs. firewalld

Both are front-end tools that manage the same underlying kernel firewall functionality (netfilter/iptables or nftables), just with different models and syntax.

UFW (Uncomplicated Firewall, used on Debian) is designed around simplicity: rules are added with short, direct commands like sudo ufw allow 4242, and the tool maintains a simple ordered rule list. This made it fast to set up exactly what the subject requires — allowing only port 4242 and denying everything else by default.

firewalld (used on Rocky) is built around the concept of zones — predefined trust levels (e.g. public, internal, trusted) that group sets of rules together. Instead of adding a raw port rule directly, you typically assign an interface or source to a zone and then open ports/services within that zone. This is more flexible for complex network setups (multiple interfaces with different trust levels) but requires understanding the zone model first, which adds a layer of complexity not needed for a single-interface VM like this project's.

I confirmed UFW was active with:

sudo ufw status verbose

which shows the firewall active, default-deny incoming, and only port 4242 (SSH) and port 80 (lighttpd, bonus) explicitly allowed.

Summary: for a single-purpose VM with one network interface, UFW's flat rule list was more direct to set up and verify than configuring firewalld's zone-based model would have been.

## VirtualBox vs UTM

I used VirtualBox, since it's available and works reliably on the machine I used at Codam. VirtualBox is cross-platform (Windows/Linux/macOS Intel), open-source, and has a long track record for this kind of project. UTM is primarily aimed at macOS (especially Apple Silicon, where VirtualBox has historically had limited support), using Apple's built-in virtualization framework — it's the subject's recommended alternative specifically for Mac M1/M2 users where VirtualBox doesn't run natively.

## DESIGN CHOICES

### Partitioning

The disk was partitioned using LVM on top of a LUKS-encrypted physical partition, following the bonus structure (splitting beyond the mandatory minimum of 2 encrypted partitions):

/boot        476M       — unencrypted, required for bootloader access before decryption<br>
/            9.31G          (root)<br>
swap         2.14G<br>
/home        4.66G<br>
/var         2.79G<br>
/srv         2.79G<br>
/tmp         2.79G<br>
/var/log     3.72G<br>

When setting up the partitions size values were assigned with the numbers shown in the example.
Small amounts get consumed by LUKS/LVM overhead before the actual logical volume was created. This is why the numbers deviate slightly from the subject example, this is normal, expected overhead cost. As stated in the subject: "The example shows arbitrary disk sizes. You need to determine the appropriate size for each partition to ensure proper operation while avoiding unnecessary disk usage.".

### Security policies

### User management

### Services installed


# INSTRUCTIONS

Useful commands:

su
passwd

sudo ufw status verbose
sudo systemctl status ufw

sudo ss -tunlp | grep ssh
sudo systemctl status ssh

head -n 2 /etc/os-release # or: less /etc/os-release

whoami
who
awk -F: '$3 >= 1000 {print $1}' /etc/passwd

getent group sudo
getent group user42
groups

sudo less /etc/login.defs
sudo cat /etc/pam.d/common-password

sudo adduser <newusername>
sudo addgroup <newgroupname>
sudo adduser <username> <newgroupname>

hostnamectl
sudo hostnamectl set-hostname <newhostname>
hostname
cat /etc/hosts

lsblk
sudo lvs
df -h

dpkg -l | grep sudo
which sudo
sudo -V
sudo adduser <username> sudo
sudo ls -la /var/log/sudo
sudo cat /var/log/sudo/sudo_log

To get the physical number of cores (=Sockets), you cannot actually test it inside the VM,
but outside it (and only on the computers which have a different number of virtual cores than the physical core only the computers on floor 1 inner circle (12 physical cores and 20 virtual cores outside the VM)
all other computers have 4 physical cores and 4 virtual cores, so it won't distinguish) with:

lscpu -p | grep -v '#' | sort -u -t, -k3,3 | wc -l


# RESOURCES

My peers at Codam: vcoevert, olistokes, alkhan, yuhma

[why number of physical processors is the number of physical sockets](https://docs.edisglobal.com/faq/upgrades-downgrades/cpu-cores-vs-cpu-sockets-in-vps-environments/understanding-cpu-cores-vs-sockets-in-vps)<br>
[lscpu](https://www.geeksforgeeks.org/linux-unix/gathering-system-information-using-commands-like-lscpu-lspci-and-lsblk/)

## AI Usage

I used Claude (Anthropic) throughout this project as a learning aid, following 42's guidelines on AI use — asking for explanations and guided reasoning rather than direct answers or copy-pasteable solutions.<br>
Specific ways I used it:<br>

Debugging and verification:<br> when a guide's suggested command produced unexpected results (e.g. counting CPU sockets vs. cores using lscpu -p), I used Claude to reason through why a command was wrong rather than being handed the fix directly — I identified the bug myself with a peer, then worked through the corrected sort/awk logic with Claude's guidance.<br>
Line-by-line script explanation:<br> for monitoring.sh, I had Claude walk me through what each command and flag does (awk, sort -k, vmstat, ss, journalctl, etc.) so I could explain the script's logic during my peer evaluation, rather than running a script I didn't understand.<br>
Conceptual explanations:<br> for topics I didn't fully understand (LVM, sockets vs. cores vs. threads, su vs sudo, how SSH port forwarding works through VirtualBox NAT), I asked Claude to explain the underlying concepts, often using analogies, until I could restate them in my own words.<br>
Troubleshooting:<br> for issues like a Fail2ban config conflict (duplicate [sshd] section) and a port-forwarding connection failure, I used Claude to help me systematically test and isolate the actual cause, rather than being told the fix outright.<br>

All actual configuration decisions, file edits, and commands were typed and executed by me inside the VM. Claude was not given access to the VM and did not write or apply any configuration directly.
