# LAB 12 : Bypass de la Détection de Root Android avec Medusa

## Avertissement et objectifs

**Objectif** : Réaliser, pas à pas (niveau débutant), un bypass de la détection de root Android en utilisant l’outil **Medusa** (un framework d'instrumentation modulaire basé sur Frida), puis valider que l’application ne « voit » plus le root.

**Usage éthique** : N'utilisez ces techniques que sur des applications/appareils que vous êtes autorisé à tester. Ce guide fournit aussi un **plan B avec Frida pur** si Medusa n’est pas disponible ou échoue.

---

## Prérequis

Avant de commencer, assurez-vous d'avoir :

- **Python 3.8+** et `pip` :
  ```bash
  python --version
  pip --version
  ```
- **ADB (Android Platform Tools)** : [Télécharger ici](https://developer.android.com/tools/releases/platform-tools)
- **Un appareil Android/immulateur** 
- **Le package de l’application cible**, par exemple `com.example.rootcheck`.

> **Astuce Windows** : Utilisez PowerShell. Sur Linux/macOS : bash/zsh.

---

## Étape 1 — Préparer l’environnement Android et Frida

Même si Medusa automatise beaucoup de choses, il repose généralement sur Frida côté PC et un service (`frida-server`) côté appareil.

### 1.1 Installer Frida côté PC

```bash
pip install --upgrade frida frida-tools
```

Vérifiez l’installation :

```bash
frida --version
python -c "import frida; print(frida.__version__)"
```

### 1.2 Vérifier la connexion ADB

```bash
adb version
adb devices
```

Vous devez voir votre appareil avec l’état `device` (et pas `unauthorized`). Si `unauthorized`, rebranchez le câble et acceptez la demande sur le téléphone.

### 1.3 Démarrer `frida-server` sur l’appareil

1. **Identifiez l’ABI du CPU** :
   ```bash
   adb shell getprop ro.product.cpu.abi
   ```

2. **Téléchargez** la version correspondante de `frida-server` depuis [les releases de Frida](https://github.com/frida/frida/releases).  
   Exemple : `frida-server-16.x.x-android-arm64.xz`

3. **Décompressez et poussez** le binaire :
   ```bash
   adb push frida-server /data/local/tmp/
   adb shell chmod 755 /data/local/tmp/frida-server
   ```

4. **Lancez le serveur** (laissez ce terminal ouvert) :
   ```bash
   adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
   ```

5. **(Optionnel) Redirigez les ports** :
   ```bash
   adb forward tcp:27042 tcp:27042
   adb forward tcp:27043 tcp:27043
   ```

6. **Vérifiez** que tout fonctionne :
   ```bash
   frida-ps -Uai
   ```
   Vous devriez voir la liste des applications installées.

> **Remarque** : Certains modules Medusa peuvent fonctionner sans lancer manuellement `frida-server`, mais pour un débutant, cette méthode est la plus simple et la plus fiable .

---

## Étape 2 — Installer Medusa

Medusa est un framework d’instrumentation modulaire qui propose des recettes prêtes à l’emploi.

### 2.1 Cloner le repo

```bash
git clone https://github.com/Ch0pin/medusa.git
cd medusa
```

### 2.2 Installer les dépendances

```bash
pip install -r requirements.txt
```

### 2.3 Vérifier l’installation

```bash
python medusa.py --help
```

Vous devriez voir apparaître les sous-commandes disponibles (`--package`, `--attach`, les modules `root-bypass`, etc.) .

> **Si Medusa n’est pas disponible** : passez directement au **Plan B — Frida pur** plus bas.

---

## Étape 3 — Comprendre la détection de root (vue débutant)

Les applications Android détectent le root de plusieurs manières :

- **Côté Java** :
  - Lecture de `android.os.Build.TAGS` (recherche de `test-keys`)
  - Vérification de l’existence de fichiers (`/system/xbin/su`, `busybox`)
  - Exécution de commandes (`Runtime.getRuntime().exec("su")`)

- **Côté natif (C/C++)** :
  - Appels système (`open`, `access`, `stat`) sur des chemins suspects
  - Lecture de `/proc/mounts`

**L’idée du bypass** : « hooker » ces appels pour renvoyer des réponses mensongères (par exemple, faire croire que `su` n’existe pas).

Medusa propose généralement un module prêt à l’emploi pour cela : `root-bypass` .

---

## Étape 4 — Lancer l’app avec Medusa et activer le bypass root

### 4.1 Injection dès le lancement (spawn)

```bash
python medusa.py --usb --spawn com.example.rootcheck --module root-bypass
```

Ou selon la syntaxe de votre version :

```bash
medusa run --package com.example.rootcheck --feature root-bypass --usb --no-pause
```

### 4.2 Attacher à une application déjà ouverte

Si l’application crash lorsqu’on l’injecte trop tôt :

```bash
python medusa.py --usb --attach "NomDuProcessus" --module root-bypass
```

### 4.3 Observation attendue

- Dans la console, Medusa affiche des logs indiquant les hooks installés (`Build.TAGS → release-keys`, `File.exists` bypass, etc.).
- Sur l’écran de l’application, le message « Root detected » disparaît ou l’application ne bloque plus .

---

## Étape 5 — Validation du bypass

1. **Sans Medusa** : Ouvrez l’application et constatez qu’elle détecte le root.
2. **Avec Medusa** : Relancez l’application via Medusa. Vérifiez que l’application passe en mode « non rootée ».
---

## Conclusion

Ce laboratoire vous a montré comment :

- **Medusa** automatise le processus de bypass grâce à ses modules prêts à l’emploi.
- **Frida pur** offre un contrôle plus fin mais nécessite de l’écriture de scripts.
- Les techniques de hooking Java et natif sont complémentaires.
