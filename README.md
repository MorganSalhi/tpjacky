# TP Falco — Détection runtime sur Kubernetes (Kind)

Annexe technique du devoir. Ce dépôt contient le guide de TP (`tp_falco_kind_guide.md`) et la configuration réellement appliquée (`falco-values.yaml`).

---

## Sujet

[Falco](https://falco.org/) est un outil de détection de menaces à l'exécution pour conteneurs et Kubernetes, diplômé CNCF (graduated). Il exploite **eBPF** pour instrumenter les appels système du kernel sans modifier les conteneurs, et évalue en temps réel des règles de sécurité prédéfinies (ou personnalisées) pour émettre des alertes.

L'objectif du TP est de déployer Falco sur un cluster Kubernetes local, de déclencher des comportements suspects et d'observer les alertes correspondantes.

---

## Environnement

| Composant | Version |
|---|---|
| Plateforme | GitHub Codespace (Linux) |
| Kernel | 6.8.0-1052-azure |
| Docker | 29.3.0 |
| Kind | v0.23.0 |
| kubectl | v1.36.2 |
| Helm | v3.21.1 |
| Falco (chart) | falcosecurity/falco |

---

## Déploiement

**Cluster Kind :** `falco-demo` (1 nœud control-plane)

**Installation Falco via Helm :**

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set driver.kind=modern_ebpf \
  --set tty=true
```

**Configuration appliquée** (`falco-values.yaml`) :

```yaml
driver:
  kind: modern_ebpf
falcosidekick:
  enabled: true
  webui:
    enabled: true
tty: true
```

**Statut DaemonSet :**

```
NAME          READY   STATUS    NODE
falco-c6947   2/2     Running   falco-demo-control-plane
```

Le driver `modern_ebpf` requiert kernel ≥ 5.8 (kernel azure 6.8.0 : compatible). Ordre de repli si incompatibilité : `modern_ebpf` → `ebpf` → `module`.

---

## Alertes capturées

### 1. Lecture de fichier sensible — `/etc/shadow`

**Règle Falco :** `Sensitive file opened for reading by non-trusted program`  
**Déclenchement :** `kubectl exec test-pod -- sh -c "cat /etc/shadow"`

```
14:40:38.711053193  Warning
file=/etc/shadow
process=cat /etc/shadow  |  parent=sh
container=test-pod  |  image=docker.io/library/busybox:latest
pod=test-pod  |  namespace=default
```

`/etc/shadow` contient les mots de passe hachés des utilisateurs Linux. Son accès par un processus non privilégié dans un conteneur est systématiquement signalé par Falco.

---

### 2. Exécution d'un binaire absent de l'image de base

**Règle Falco :** `Executing binary not part of base image`  
**Déclenchement :** exécution de `mount`, `kubectl`, `claude.exe` depuis l'intérieur d'un conteneur (binaires ajoutés après la construction de l'image)

```
14:40:27.822051295  Critical
process=mount  |  parent=runc:[1:CHILD]
exe_flags=EXE_WRITABLE|EXE_UPPER_LAYER
container=test-pod  |  image=docker.io/library/busybox:latest
pod=test-pod  |  namespace=default
```

Le flag `EXE_UPPER_LAYER` indique que le binaire se trouve dans les couches supérieures (writables) du système de fichiers overlay, et non dans l'image de base immuable — signal fort d'une modification post-déploiement.

---

### 3. Shell interactif dans un conteneur

**Règle Falco :** `A shell was spawned in a container with an attached terminal`  
**Déclenchement :** `kubectl exec -it test-pod -- sh` avec pseudo-TTY alloué

```
14:51:52.731880253  Notice
process=sh -c echo shell-interactif-declenche; id; hostname
parent=containerd-shim  |  terminal=34816
container=test-pod  |  image=docker.io/library/busybox:latest
pod=test-pod  |  namespace=default
```

**Contournement PTY nécessaire :** la règle exige `proc.tty != 0`. Depuis un shell non-interactif (GitHub Codespace, Claude Code), `kubectl exec -t` ne suffit pas car le processus parent n'a pas de terminal. Solution appliquée :

```bash
script -q -c "kubectl exec -it test-pod -- sh -c 'id; hostname'" /dev/null
```

`script` crée un pseudo-terminal (PTY) pour son processus enfant, ce qui permet à `kubectl` d'allouer un TTY dans le conteneur (`terminal=34816`).

---

## Falcosidekick UI

Déployé via `helm upgrade --set falcosidekick.enabled=true --set falcosidekick.webui.enabled=true`, exposé sur `http://localhost:2802`.

**Accès :**

| Champ | Valeur |
|---|---|
| URL | `http://localhost:2802` |
| Login | `admin` |
| Mot de passe | `admin` |

> Identifiants par défaut du chart Helm (secret Kubernetes `falco-falcosidekick-ui`, champ `FALCOSIDEKICK_UI_USER`).

**Statistiques observées (24h) :**

- **8 alertes** au total
- **100 % Critical** (règle `Executing binary not part of base image`)
- Tous les processus incriminés sont de l'**outillage légitime** du Codespace : `kubectl`, `claude.exe`, `ugrep`, `rg`

**Analyse — faux positifs :**

Ces alertes illustrent une limite connue du déploiement Falco en environnement de développement : le DaemonSet surveille **tous les conteneurs du nœud**, y compris celui du Codespace lui-même. Les binaires détectés (`kubectl`, outils Claude Code) sont présents dans les couches overlay de l'image du Codespace, non dans l'image de base, d'où le flag `EXE_UPPER_LAYER` et les alertes `Critical`.

En production, on réduit ce bruit par :
- des règles d'exclusion ciblées (`exceptions` dans les règles Falco) ;
- le champ `container_image` pour restreindre la règle aux images métier ;
- l'intégration Falcosidekick → SIEM pour du triage automatisé.

---

## Difficultés et solutions

| Difficulté | Solution appliquée |
|---|---|
| Choix du driver eBPF | `modern_ebpf` (kernel 6.8.0 ≥ 5.8) ; vérifier `uname -r` avant de choisir |
| Privilèges élevés du DaemonSet | Falco requiert `privileged: true` ou capabilities `SYS_ADMIN`/`SYS_PTRACE` — inhérent à l'instrumentation kernel |
| Règle "Terminal shell" non déclenchée | Shell non-interactif → `proc.tty=0`. Contournement : `script -q -c "kubectl exec -it ..."` pour forcer l'allocation PTY |
| Bruit / faux positifs | Alertes `EXE_UPPER_LAYER` sur l'outillage légitime du Codespace — à filtrer par règles d'exception en prod |

---

## Conclusion

Ce TP illustre concrètement le fonctionnement de Falco comme IDS (Intrusion Detection System) kernel-level pour Kubernetes. Les trois alertes déclenchées couvrent des vecteurs d'attaque réels : exfiltration de credentials (`/etc/shadow`), persistance via binaire injecté (`EXE_UPPER_LAYER`), et accès interactif post-compromission (shell avec TTY).

Le retour d'expérience principal est la nécessité d'un **tuning des règles** dès le déploiement : sans exclusions adaptées à l'environnement, le ratio signal/bruit dégrade l'opérabilité. Falcosidekick et son UI facilitent cette phase en centralisant et visualisant les alertes avant intégration dans un SIEM.
