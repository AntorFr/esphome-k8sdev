# ESPHome K8s Dev Environment

Image Docker de développement ESPHome avec VS Code Tunnel pour développer dans Kubernetes tout en utilisant VS Code Desktop sur Mac.

## 🚀 Fonctionnalités

- Environnement ESPHome complet (basé sur le devcontainer officiel, branche `dev`)
- VS Code CLI intégré pour connexion via Remote Tunnels
- Multi-architecture : `linux/amd64` et `linux/arm64`
- Build automatique via GitHub Actions

## 📦 Image Docker

```
ghcr.io/<OWNER>/esphome-k8sdev:dev
ghcr.io/<OWNER>/esphome-k8sdev:<sha>
```

### Pull l'image

```bash
docker pull ghcr.io/<OWNER>/esphome-k8sdev:dev
```

## 🔧 Déclencher le build

### Automatiquement
Le build se déclenche automatiquement sur chaque push sur la branche `main`.

### Manuellement
1. Aller sur l'onglet **Actions** du repository GitHub
2. Sélectionner le workflow **Build and Push ESPHome Dev Image**
3. Cliquer sur **Run workflow**
4. Sélectionner la branche `main` et cliquer sur **Run workflow**

## ☸️ Déploiement Kubernetes

### Manifeste complet

Créer un fichier `esphome-dev.yaml` :

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: esphome-dev

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: esphome-data
  namespace: esphome-dev
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  # Décommenter et adapter si vous avez une StorageClass spécifique
  # storageClassName: longhorn

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: esphome-dev
  namespace: esphome-dev
spec:
  serviceName: esphome-dev
  replicas: 1
  selector:
    matchLabels:
      app: esphome-dev
  template:
    metadata:
      labels:
        app: esphome-dev
    spec:
      securityContext:
        fsGroup: 1000
      containers:
        - name: esphome
          image: ghcr.io/<OWNER>/esphome-k8sdev:dev
          imagePullPolicy: Always
          ports:
            - containerPort: 6052
              name: dashboard
          volumeMounts:
            - name: data
              mountPath: /home/esphome
              subPath: home
            - name: data
              mountPath: /workspaces
              subPath: workspaces
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "4Gi"
              cpu: "2000m"
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: esphome-data

---
# Service optionnel pour accéder au dashboard ESPHome
apiVersion: v1
kind: Service
metadata:
  name: esphome-dashboard
  namespace: esphome-dev
spec:
  selector:
    app: esphome-dev
  ports:
    - port: 6052
      targetPort: 6052
      name: dashboard
  type: ClusterIP
```

### Déployer

```bash
# Remplacer <OWNER> par votre username GitHub dans le fichier
sed -i 's/<OWNER>/votre-username/g' esphome-dev.yaml

# Appliquer le manifeste
kubectl apply -f esphome-dev.yaml
```

## 🔗 Connexion depuis VS Code Desktop

### 1. Récupérer le device code

Au premier démarrage, le conteneur va demander une authentification GitHub. Récupérez le code dans les logs :

```bash
kubectl logs -f -n esphome-dev statefulset/esphome-dev
```

Vous verrez quelque chose comme :

```
*
* Visual Studio Code Server
*
* By using the software, you agree to
* the Visual Studio Code Server License Terms (https://aka.ms/vscode-server-license) and
* the Microsoft Privacy Statement (https://privacy.microsoft.com/en-US/privacystatement).
*
To grant access to the server, please log into https://github.com/login/device and use code XXXX-XXXX
```

### 2. Authentifier

1. Ouvrir https://github.com/login/device
2. Entrer le code `XXXX-XXXX` affiché dans les logs
3. Autoriser l'accès

### 3. Se connecter depuis VS Code Desktop

1. Installer l'extension **Remote - Tunnels** (`ms-vscode.remote-server`)
2. Ouvrir la palette de commandes (`Cmd+Shift+P`)
3. Taper **Remote-Tunnels: Connect to Tunnel...**
4. Sélectionner **GitHub** comme provider
5. Choisir le tunnel **esphome-k8s**

Vous êtes maintenant connecté à votre environnement ESPHome dans Kubernetes ! 🎉

## 🖥️ Dashboard ESPHome

Le dashboard ESPHome écoute sur le port `6052`. Plusieurs options pour y accéder :

### Option 1 : Port forward depuis VS Code Remote

Une fois connecté via le tunnel, VS Code peut automatiquement forwarder les ports. Lancez ESPHome :

```bash
esphome dashboard /workspaces/config
```

VS Code détectera le port 6052 et proposera de le forwarder.

### Option 2 : Port forward kubectl

```bash
kubectl port-forward -n esphome-dev svc/esphome-dashboard 6052:6052
```

Puis ouvrir http://localhost:6052

### Option 3 : Ingress (avancé)

Créer un Ingress pour exposer le service `esphome-dashboard` sur votre domaine.

## 📁 Structure des volumes

| Chemin conteneur | subPath | Description |
|------------------|---------|-------------|
| `/home/esphome` | `home` | Configuration utilisateur, cache VS Code, historique |
| `/workspaces` | `workspaces` | Projets ESPHome, code source |

## 🔄 Mise à jour

L'image est rebuilde automatiquement à chaque push sur `main`. Pour mettre à jour le déploiement :

```bash
kubectl rollout restart -n esphome-dev statefulset/esphome-dev
```

## 🐛 Troubleshooting

### Le tunnel ne démarre pas

Vérifier les logs :
```bash
kubectl logs -n esphome-dev statefulset/esphome-dev
```

### Réinitialiser l'authentification VS Code

```bash
kubectl exec -it -n esphome-dev esphome-dev-0 -- rm -rf /home/esphome/.vscode-cli
kubectl rollout restart -n esphome-dev statefulset/esphome-dev
```

### Accéder au shell du conteneur

```bash
kubectl exec -it -n esphome-dev esphome-dev-0 -- /bin/bash
```

## 📝 Notes

- L'image est basée sur la branche `dev` d'ESPHome pour avoir les dernières fonctionnalités
- Le cache PlatformIO est pré-installé pour accélérer les compilations
- Utilisez GitHub Copilot dans VS Code pour une expérience de développement optimale !
