Guide rapide — Falco sur Kind (quelques heures)
À exécuter idéalement via Claude Code (debug interactif possible si le driver eBPF pose problème selon ton kernel).
Prérequis
Docker, kubectl, kind, helm installés
Vérifier le kernel (impacte le choix du driver Falco) :
```bash
uname -r
```
Falco "modern eBPF" nécessite un kernel récent (≥5.8). Si problème, fallback possible vers le driver "ebpf" classique ou le module noyau — à ajuster en fonction de ce que renvoie `helm show values falcosecurity/falco` (les valeurs du chart évoluent, vérifier la doc officielle à jour).
1. Créer le cluster
```bash
kind create cluster --name falco-demo
kubectl cluster-info --context kind-falco-demo
```
2. Installer Falco
```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set driver.kind=modern_ebpf \
  --set tty=true
```
Si le pod Falco crash ou reste en CrashLoopBackOff, vérifier les logs (`kubectl logs -n falco -l app.kubernetes.io/name=falco`) — c'est généralement le driver. Basculer sur `--set driver.kind=ebpf` ou `--set driver.kind=module` si nécessaire (Claude Code peut chercher la doc à jour et ajuster).
3. Suivre les alertes en temps réel
```bash
kubectl logs -f -n falco -l app.kubernetes.io/name=falco
```
4. Générer des événements à détecter
Déployer un pod de test :
```bash
kubectl run test-pod --image=busybox -- sleep 3600
```
Déclencher des alertes classiques (règles par défaut Falco) :
```bash
# Déclenche "Terminal shell in container"
kubectl exec -it test-pod -- sh

# Dans le shell, déclenche "Read sensitive file untrusted"
cat /etc/shadow

# Déclenche potentiellement "Launch Suspicious Network Tool"
which nc || apk add netcat-openbsd
```
Capturer les logs Falco correspondants (screenshots pour la slide "mise en œuvre").
5. (Bonus si temps restant) Falcosidekick + UI
Pour un dashboard web plus visuel :
```bash
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --set driver.kind=modern_ebpf \
  --set tty=true \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true

kubectl port-forward -n falco svc/falco-falcosidekick-ui 2802:2802
```
Accès : http://localhost:2802 — capture d'écran du dashboard listant les alertes déclenchées à l'étape 4.
Captures à préparer pour le PPTX
`kubectl get pods -n falco -o wide` (DaemonSet Falco actif sur le(s) node(s))
Logs Falco montrant au moins 2 alertes différentes (étape 4)
(Si fait) Dashboard Falcosidekick UI
`helm get values falco -n falco` (preuve de config) — optionnel
Difficultés probables (à réutiliser dans "difficultés rencontrées")
Compatibilité driver eBPF / version kernel hôte
DaemonSet Falco nécessite des privilèges élevés (`privileged: true`, `hostPID`) — point à discuter en "retour d'expérience" (surface d'attaque de l'outil de sécurité lui-même)
Volume d'alertes / bruit avec les règles par défaut → besoin de tuning