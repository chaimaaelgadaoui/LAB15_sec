# Lab 15 — Analyse Dynamique Android : Inspection TLS/HTTPS et Gestion du SSL Pinning

**Étudiante :** Chaimaa Elgadaoui  
**Cours :** Sécurité des applications mobiles  

---

> ⚠️ **Avertissement éthique** : Ces techniques sont utilisées uniquement dans un cadre légal (formation, audit autorisé, tests sur appareils personnels). Le contournement du SSL pinning vise l'inspection de trafic à des fins de sécurité. Aucune donnée réelle n'a été exposée.

---

## Objectifs

- Installer et vérifier Frida côté PC et `frida-server` sur Android.
- Mettre en place un proxy (Burp Suite) et un certificat CA sur l'appareil.
- Neutraliser le SSL pinning via hooks Java (TrustManager / Conscrypt / OkHttp / WebView).
- Valider en capturant le trafic HTTPS déchiffré.

---

## Partie 1 — Environnement & Outils

J'ai commencé par vérifier que tous les outils nécessaires étaient installés et fonctionnels sur mon PC Windows.

| Outil        | Version détectée  | Statut |
|--------------|-------------------|--------|
| Python       | 3.11.9            | ✅     |
| pip          | 24.0              | ✅     |
| ADB          | 1.0.41 (v37.0.0)  | ✅     |
| frida (CLI)  | 17.9.11           | ✅     |
| frida (lib)  | 17.9.11           | ✅     |

- **OS PC :** Windows 10.0.26200
- **Appareil :** Émulateur Android (`emulator-5554`) — statut : `device` ✅
- **Architecture CPU :** `x86_64`

Commandes exécutées :
```
python --version
pip --version
adb version
python -m pip install --upgrade frida frida-tools
frida --version
python -c "import frida; print(frida.__version__)"
adb devices
adb shell getprop ro.product.cpu.abi
```

> Screenshot : images/SS-01.png

---

## Partie 2 — Déploiement de frida-server sur l'émulateur

Ayant identifié l'architecture `x86_64`, j'ai téléchargé `frida-server-17.9.11-android-x86_64.xz` depuis les releases officielles GitHub, puis je l'ai décompressé et transféré sur l'émulateur.

```
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "su 0 /data/local/tmp/frida-server &"
frida-ps -Uai
```

| Action                  | Résultat                          | Statut |
|-------------------------|-----------------------------------|--------|
| Push frida-server       | 110 MB transféré à 61.0 MB/s      | ✅     |
| chmod 755               | Permissions appliquées            | ✅     |
| Lancement frida-server  | Déjà actif sur le port 27042      | ✅     |
| frida-ps -Uai           | Liste des apps visible            | ✅     |

- **frida-server version :** `17.9.11` (correspond exactement au client)
- **Apps cibles détectées sur l'émulateur :**
  - `jakhar.aseem.diva` — DIVA (Damn Insecure and Vulnerable App)
  - `owasp.mstg.uncrackable3` — OWASP UnCrackable Level 3

> Screenshot : images/SS-02.png

---

## Partie 3 — Configuration Proxy & Certificat CA

J'ai configuré Burp Suite comme proxy TLS pour intercepter le trafic HTTPS de l'émulateur.

**Configuration Burp Suite :**
- Proxy listener : `127.0.0.1:8080` ✅
- Redirection ADB : `adb reverse tcp:8080 tcp:8080`
- Proxy Wi-Fi émulateur : `127.0.0.1:8080` configuré manuellement

**Certificat CA :**

J'ai téléchargé le certificat CA de Burp depuis `http://127.0.0.1:8080` sur l'émulateur (`cacert.der`, 986 bytes), puis je l'ai récupéré sur le PC via ADB pour le convertir :

```
adb pull /sdcard/Download/cacert.der C:\Users\USER\Downloads\cacert.der
python -c "
from cryptography import x509
from cryptography.hazmat.primitives import serialization
with open('cacert.der', 'rb') as f:
    cert = x509.load_der_x509_certificate(f.read())
with open('cacert.pem', 'wb') as f:
    f.write(cert.public_bytes(serialization.Encoding.PEM))
"
```

Le hash généré : `9a5ba575.0`

| Action                      | Résultat                            | Statut |
|-----------------------------|-------------------------------------|--------|
| Burp Suite proxy listener   | 127.0.0.1:8080                      | ✅     |
| adb reverse tcp:8080        | Trafic redirigé vers Burp           | ✅     |
| Proxy Wi-Fi émulateur       | 127.0.0.1:8080 configuré            | ✅     |
| Téléchargement cacert.der   | 986 bytes récupéré depuis Burp      | ✅     |
| Conversion PEM + hash       | 9a5ba575.0 généré                   | ✅     |
| Installation système        | ⚠️ Read-only (image Google Play)   | ⚠️     |

> **Note :** L'émulateur utilise une image Google Play dont le système de fichiers est en lecture seule. L'installation du certificat CA en tant que certificat système via ADB n'a pas été possible. Le bypass SSL a donc été entièrement délégué à Frida via le script `sslpin_bypass_universal.js`.

> Screenshot : images/SS-03.png

---

## Partie 4 — Script SSL Bypass Universel

J'ai créé le fichier `sslpin_bypass_universal.js` dans le dossier `Downloads` (3519 bytes). Ce script Frida couvre tous les vecteurs de SSL pinning courants sur Android :

| Composant ciblé                  | Mécanisme de bypass                          |
|----------------------------------|----------------------------------------------|
| `SSLContext.init`                | Injection d'un TrustManager permissif        |
| `X509TrustManager`              | Patch de checkServerTrusted/checkClientTrusted|
| `Conscrypt TrustManagerImpl`    | Neutralisation checkTrusted/verifyChain       |
| `OkHttp CertificatePinner`      | Bypass de CertificatePinner.check             |
| `WebView onReceivedSslError`    | Acceptation automatique des erreurs SSL       |

```
New-Item -Name "sslpin_bypass_universal.js" -ItemType File
notepad sslpin_bypass_universal.js
```

> Screenshot : images/SS-04.png

---

## Partie 5 — Injection Frida sur l'app DIVA

J'ai ciblé l'application DIVA (`jakhar.aseem.diva`) déjà installée sur l'émulateur. Après avoir ouvert l'app manuellement, j'ai attaché Frida en mode attach via le nom du processus :

```
frida -U -n "Diva" -l sslpin_bypass_universal.js
```

**Logs obtenus :**
```
[+] SSL bypass: SSLContext.init patched
[+] SSL bypass: X509TrustManager patches attempted
[+] SSL bypass: com.android.org.conscrypt.TrustManagerImpl patched
[+] Universal SSL pinning bypass installed
[Android Emulator 5554::Diva ]->
```

| Hook                        | Résultat         | Statut |
|-----------------------------|------------------|--------|
| SSLContext.init             | Patché           | ✅     |
| X509TrustManager            | Patché           | ✅     |
| Conscrypt TrustManagerImpl  | Patché           | ✅     |
| OkHttp CertificatePinner    | Non applicable   | N/A    |
| WebView onReceivedSslError  | Non applicable   | N/A    |

> **Note :** L'émulateur Google Play étant en mode "jailed", le mode spawn (`-f`) n'était pas disponible. J'ai utilisé le mode attach (`-n`) après ouverture manuelle de l'app.

> Screenshot : images/SS-05.png

---

## Partie 6 — Validation du trafic HTTPS intercepté

Après injection du script, j'ai navigué vers `https://httpbin.org/get` depuis le navigateur Chrome de l'émulateur. Burp Suite a immédiatement intercepté et déchiffré le trafic HTTPS dans l'onglet **Proxy → HTTP History**.

**Proxy :** Burp Suite Community v2026.4.3 — `127.0.0.1:8080`

| Requête interceptée       | Méthode | Status | TLS | IP              |
|---------------------------|---------|--------|-----|-----------------|
| https://httpbin.org/get   | GET     | 503    | ✅  | 98.91.36.5      |
| https://httpbin.org/get   | GET     | 503    | ✅  | 98.91.36.5      |
| http://update.googleapis  | POST    | 200    | —   | 216.58.205.46   |
| http://clientservices...  | GET     | 200    | —   | 192.178.25.110  |

> Le code 503 retourné par `httpbin.org` indique une indisponibilité temporaire du serveur, mais la connexion TLS a bien été établie, interceptée et déchiffrée par Burp Suite. La colonne TLS cochée (✅) confirme que le trafic HTTPS est visible en clair.

> Screenshot : images/SS-06.png





---

