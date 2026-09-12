# Troubleshooting Log

This file documents the main problems encountered during the lab and the reasoning used to solve them.

## 1. NTFS Modify but user could not create files

**Symptom:** HR user had NTFS `Modify` on the HR folder but could not create a `.txt` file over the network.

**Investigation:** NTFS permissions did not show an obvious Deny. Effective Access was difficult to evaluate remotely, so the permissions were checked locally on DC01.

**Root cause:** The parent share allowed `Everyone: Read` but did not allow `Change`.

**Fix:** Share permission was changed to allow `Read + Change`, while detailed department restrictions remained in NTFS.

**Lesson:** Share and NTFS permissions are separate layers. Over the network, the effective result is limited by the more restrictive layer.

---

## 2. HR could access Finance and Management

**Symptom:** After the Share permission was opened enough to permit changes, HR could also create/delete files in other department folders.

**Root cause:** Broad NTFS permissions were inherited from `C:\CompanyData`, including generic `Users`-style access.

**Fix:** Inheritance was disabled where appropriate on department folders and unnecessary broad entries were removed. Department security groups retained `Modify` only on their own folders.

**Lesson:** Inheritance can silently spread access to child folders.

---

## 3. Duplicate / similar AD object names

**Symptom:** Objects such as `HRUser`, `HR-User`, `FinanceUser`, and `Finance-user` created confusion during testing.

**Fix:** Used `whoami` to confirm the actual signed-in identity and later standardized access through `GG_*` security groups.

**Lesson:** Naming standards matter. Avoid creating similarly named test and production-style objects.

---

## 4. Windows Network view was empty

**Symptom:** The visual Network section did not always show the server/share.

**Fix:** Used direct UNC paths such as:

```text
\\DC01\CompanyData
```

**Lesson:** Network Discovery UI and SMB connectivity are not the same thing.

---

## 5. Remote Effective Access could not be evaluated

**Symptom:** Windows reported that the current user did not have permission to evaluate effective access rights for the remote source.

**Fix:** Performed the check locally on DC01 using an administrative account.

---

## 6. Microsoft Print to PDF could not be shared

**Symptom:** Windows displayed `Sharing is not supported for this type of printer.`

**Root cause:** Microsoft Print to PDF is a local virtual print-to-file device.

**Fix:** Created a new lab printer using `Generic / Text Only` and shared it as `CORP-Office-Printer-Shared`.

---

## 7. Printer name conflict

**Symptom:** New printer creation failed because the original test printer already used the name `CORP-Office-Printer`.

**Fix:** Created `CORP-Office-Printer-Shared` and deleted the incorrect old test printer.

---

## 8. Deployed Printer Connections Group Policy warning

**Symptom:** `gpupdate /force` showed:

```text
Windows failed to apply the Deployed Printer Connections settings.
```

**Investigation:** The printer share itself was reachable through `\\DC01`, and the print queue opened manually.

**Fix:** Removed the older `Deploy with Group Policy` connection and used:

```text
User Configuration
-> Preferences
-> Control Panel Settings
-> Printers
-> Shared Printer
Action: Update
```

Final `gpupdate /force` completed without the previous warning.

---

## 9. Event Viewer GroupPolicy log showed Access Denied (5)

**Fix:** Reopened Event Viewer with `Run as administrator` and inspected:

```text
Applications and Services Logs
-> Microsoft
-> Windows
-> GroupPolicy
-> Operational
```

---

## 10. Linux could resolve internal AD names but not internet package repositories

**Symptom:** `apt update` failed with:

```text
Temporary failure resolving 'http.kali.org'
```

**Context:** LINUX01 correctly used DC01 (`192.168.207.10`) as DNS and could resolve `dc01.corp.local`.

**Root cause:** Public DNS forwarding from DC01 was not functioning for this lab.

**Fix:** Added DNS Forwarders on DC01, including `8.8.8.8` and `1.1.1.1`.

**Lesson:** Do not bypass AD DNS on domain-integrated systems just to fix internet resolution; fix forwarding instead.

---

## 11. SSSD packages were unavailable on Kali

**Symptom:** APT could not locate `sssd`, `sssd-tools`, `libnss-sss`, and related packages.

**Investigation:** Kali repository configuration under `/etc/apt/sources.list.d/kali.sources` was checked and package indexes were refreshed.

**Decision:** Did not mix Debian repositories or download random packages.

**Fix:** Used Samba + Winbind + Kerberos as the AD integration method.

**Lesson:** Avoid destabilizing a distribution to force one preferred implementation.

---

## 12. `wbinfo -t` failed while the domain join was valid

**Symptom:** Plain `wbinfo -t` returned `WBC_ERR_WINBIND_NOT_AVAILABLE`.

**Checks:**

```bash
sudo systemctl status winbind --no-pager
ls -l /run/samba
ls -l /run/samba/winbindd
sudo journalctl -u winbind -n 50 --no-pager
sudo net ads testjoin
wbinfo -p
```

Results showed:

- Winbind active
- Winbind pipe existed
- `Join is OK`
- Local Winbind ping succeeded

The elevated trust test succeeded:

```bash
sudo wbinfo -t
```

Additional validation:

```bash
sudo wbinfo --ping-dc
wbinfo --online-status
```

confirmed an active connection to `DC01.corp.local`.

---

## 13. `kali` user could not list `/srv/intranet-files`

**Symptom:** `ls -l /srv/intranet-files` returned `Permission denied`.

**Root cause:** This was expected. The directory was deliberately configured as:

```text
root:webadmins
770
```

and `kali` was not in `webadmins`.

**Verification:**

```bash
sudo -u webuser ls -l /srv/intranet-files
```

worked and showed `test.txt`.

**Lesson:** A permission-denied message can be proof that the access control is working correctly.
