# Final Validation Results

The project was tested end to end after the build was completed.

| Area | Test | Result |
|---|---|---|
| AD / DNS | `dcdiag /test:dns` | PASS |
| DNS | `nslookup dc01.corp.local` -> `192.168.207.10` | PASS |
| DNS | `nslookup corp.local` -> `192.168.207.10` | PASS |
| Domain login | Finance domain account login | PASS |
| Mapped drives | Finance `F:` drive visible and usable | PASS |
| Department isolation | Finance denied from other department folders | PASS |
| Security GPO | CMD blocked for normal user | PASS |
| Security GPO | Control Panel blocked | PASS |
| Security GPO | Settings blocked | PASS |
| Account lockout | 5 incorrect attempts lock `test-user` | PASS |
| Account unlock | ADUC unlock + correct login | PASS |
| Workstation firewall | Domain Profile active, firewall on | PASS |
| Workstation firewall | Unmatched inbound blocked / outbound allowed | PASS |
| Printer | Shared printer present in `shell:PrintersFolder` | PASS |
| Linux static IP | `eth0 = 192.168.207.20/24` | PASS |
| Linux DNS | `ping -c 4 dc01.corp.local` | PASS |
| SSH | `ssh.service` active | PASS |
| Apache | `apache2.service` active | PASS |
| UFW | Active, inbound deny, 22/80 allowed | PASS |
| Linux AD trust | `sudo net ads testjoin` -> `Join is OK` | PASS |
| Linux AD identity | `getent passwd HR-User` returns domain user | PASS |
| Linux permissions | `webuser` can access `/srv/intranet-files` | PASS |

## Final Windows DNS Validation

```cmd
dcdiag /test:dns
nslookup dc01.corp.local
nslookup corp.local
```

Expected / observed result:

```text
DC01 passed test DNS
dc01.corp.local -> 192.168.207.10
corp.local -> 192.168.207.10
```

## Final Linux Validation

```bash
ip addr
ping -c 4 dc01.corp.local
sudo systemctl status ssh --no-pager
sudo systemctl status apache2 --no-pager
sudo ufw status verbose
sudo net ads testjoin
getent passwd HR-User
sudo -u webuser ls -l /srv/intranet-files
```

Key observed results:

```text
192.168.207.20/24
0% packet loss to DC01
SSH active
Apache active
UFW active
Join is OK
AD user resolves
webuser can see test.txt
```

## Known Test Limitation

The removable-storage GPO was configured but not physically tested because a USB device was not available.

The shared printer was present and usable after GPP cleanup. For an even stricter future test, a disposable clean user profile could be used to verify automatic printer recreation without any earlier manual connection history.
