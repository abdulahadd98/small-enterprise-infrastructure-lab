# Command Reference

## Windows Networking / DNS

```cmd
ipconfig
```
Shows basic IP configuration.

```cmd
ipconfig /all
```
Shows detailed IP, DNS, DHCP, gateway, MAC, and suffix information.

```cmd
ping <target>
```
Tests network reachability.

```cmd
nslookup corp.local
nslookup dc01.corp.local
```
Tests DNS resolution.

```cmd
dcdiag /test:dns
```
Runs Domain Controller DNS diagnostics.

## Windows Identity / Group Policy

```cmd
whoami
```
Shows the current Windows security identity.

```cmd
gpupdate /force
```
Forces user and computer Group Policy processing.

```cmd
gpresult /r
```
Shows applied GPO summary.

```cmd
gpresult /h C:\GPReport.html
```
Creates a detailed HTML Group Policy report.

```cmd
gpresult /scope user /v
```
Shows verbose user-side policy results.

```text
rsop.msc
```
Opens graphical Resultant Set of Policy.

```text
wf.msc
```
Opens Windows Defender Firewall with Advanced Security.

```text
shell:PrintersFolder
```
Opens the Windows Printers folder directly.

## Windows-to-Linux SSH

```powershell
ssh kali@192.168.207.20
```
Starts an SSH session from Windows to LINUX01.

## Linux Networking

```bash
ip addr
ip route
```
Shows IP addresses and routing information.

```bash
nmcli connection show
```
Shows NetworkManager profiles.

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.addresses 192.168.207.20/24
sudo nmcli connection modify "Wired connection 1" ipv4.gateway 192.168.207.2
sudo nmcli connection modify "Wired connection 1" ipv4.dns 192.168.207.10
sudo nmcli connection modify "Wired connection 1" ipv4.method manual
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```
Configures and applies the static IPv4 profile.

```bash
cat /etc/resolv.conf
```
Shows active DNS resolver configuration.

```bash
ping -c 4 dc01.corp.local
```
Tests DNS plus network connectivity to DC01.

## Linux Service Management

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh --no-pager
```
Enables, starts, and checks SSH.

```bash
sudo systemctl enable apache2
sudo systemctl start apache2
sudo systemctl status apache2 --no-pager
```
Enables, starts, and checks Apache.

## Kerberos / Samba / Winbind

```bash
kinit Administrator@CORP.LOCAL
klist
```
Requests and displays Kerberos tickets.

```bash
testparm
```
Validates Samba configuration.

```bash
sudo net ads join -U Administrator
```
Joins LINUX01 to Active Directory.

```bash
sudo net ads testjoin
```
Validates the machine trust.

```bash
sudo systemctl enable --now winbind
sudo systemctl status winbind --no-pager
```
Enables/starts and checks Winbind.

```bash
wbinfo -p
sudo wbinfo -t
sudo wbinfo --ping-dc
wbinfo --online-status
wbinfo -u
wbinfo -g
```
Tests Winbind daemon/domain trust and enumerates AD identities.

```bash
getent passwd HR-User
```
Checks whether Linux NSS can resolve a domain user.

```bash
grep -E '^(passwd|group):' /etc/nsswitch.conf
```
Checks whether `winbind` participates in user/group lookups.

## UFW

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw enable
sudo ufw status verbose
```
Allows SSH/HTTP, enables UFW, and displays policy.

## Linux Users / Groups / Permissions

```bash
sudo groupadd webadmins
sudo useradd -m webuser
sudo usermod -aG webadmins webuser
sudo mkdir -p /srv/intranet-files
sudo chown root:webadmins /srv/intranet-files
sudo chmod 770 /srv/intranet-files
ls -ld /srv/intranet-files
sudo -u webuser touch /srv/intranet-files/test.txt
sudo -u webuser ls -l /srv/intranet-files
```
Creates a group/user, protects a directory with group permissions, and validates access.
