# 🚀 Create APK — Convertir un site web en APK Android

> Génère un APK Android natif (TWA) à partir de n'importe quelle URL, directement depuis ton navigateur, via GitHub Actions — sans installer Android Studio.

---

## ✨ Fonctionnement

1. Tu entres ton **token GitHub** et l'**URL du site** à convertir
2. L'outil déclenche un workflow GitHub Actions via `repository_dispatch`
3. Le workflow utilise **Bubblewrap** pour compiler une **Trusted Web Activity (TWA)**
4. L'APK est signé, packagé comme artefact et disponible au téléchargement

> 💡 Une **TWA** n'est pas une simple WebView : c'est une app Android native qui délègue l'affichage à Chrome. Le site tourne dans Chrome, sans barre d'URL, comme une vraie app.

---

## 📋 Prérequis — Côté GitHub

- Un compte GitHub avec accès au repo `mikefri/Create_APK`
- Un **Personal Access Token (PAT)** avec les permissions :
  - `repo`
  - `workflow`
- Les **secrets GitHub** suivants configurés dans le repo :
  - `KEYSTORE_PASSWORD` — mot de passe du keystore Android
  - `KEY_PASSWORD` — mot de passe de la clé de signature

> 🔑 [Créer un token GitHub](https://github.com/settings/tokens/new)  
> ⚙️ [Configurer les secrets du repo](https://github.com/mikefri/Create_APK/settings/secrets/actions)

---

## 🌐 Prérequis — Côté site web

C'est la partie la plus importante. Le build utilise **Bubblewrap / TWA**, qui impose des contraintes strictes sur le site source.

### ✅ 1. HTTPS obligatoire

Ton site doit être accessible en `https://`. Les URLs en `http://` sont rejetées par Android et par Bubblewrap.

---

### ✅ 2. Un fichier `manifest.json` (Web App Manifest) — **obligatoire**

Bubblewrap en a besoin pour initialiser le projet. Sans lui, **le build plante**.

Il doit être accessible à cette URL :
```
https://mon-site.com/manifest.json
```

**Exemple minimal fonctionnel :**
```json
{
  "name": "Mon Application",
  "short_name": "MonApp",
  "description": "Description de mon app",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#2ea44f",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

> ⚠️ L'icône en **512×512 px** est obligatoire — le build échoue sans elle.

---

### ✅ 3. Le manifest doit être référencé dans le HTML

Dans le `<head>` de ton `index.html` :
```html
<link rel="manifest" href="/manifest.json">
```

---

### ⚠️ 4. Fichier `assetlinks.json` — recommandé fortement

Sans ce fichier, l'APK est généré et fonctionne, **mais Android affiche une barre d'URL Chrome** — ce qui casse l'effet "app native".

À placer à :
```
https://mon-site.com/.well-known/assetlinks.json
```

Tu obtiendras le contenu exact de ce fichier **après** avoir généré l'APK et récupéré l'empreinte SHA-256 du keystore. Exemple de structure :
```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.monapp",
    "sha256_cert_fingerprints": ["AA:BB:CC:..."]
  }
}]
```

---

## 📊 Tableau récapitulatif

| Condition | Impact si absent |
|---|---|
| Site en HTTPS | ❌ Build impossible |
| `manifest.json` accessible | ❌ Build plante immédiatement |
| Icône 512×512 dans le manifest | ❌ Build échoue |
| `display: "standalone"` dans le manifest | ⚠️ Rendu comme onglet, pas une app |
| `assetlinks.json` configuré | ⚠️ Barre d'URL Chrome visible dans l'app |
| Site accessible publiquement | ❌ Introuvable depuis GitHub Actions |

---

## 🖥️ Utilisation

1. Ouvre `index.html` dans ton navigateur (ou héberge-le)
2. Colle ton token GitHub dans le champ **🔑 Token GitHub**
3. Entre l'URL complète du site (ex: `https://mon-site.com`)
4. Clique sur **Générer l'APK**
5. Suis la progression en temps réel :
   - 📤 Envoi du build à GitHub
   - ⚙️ Compilation via Bubblewrap (~3–5 min)
   - 📦 APK prêt
6. Télécharge le `.zip` contenant ton APK signé

---

## 📁 Structure du projet

```
Create_APK/
├── index.html                  # Interface web principale
├── .github/
│   └── workflows/
│       └── build.yml           # Workflow GitHub Actions (Bubblewrap + signature)
└── README.md
```

---

## ⚙️ Workflow GitHub Actions

Déclenché via `repository_dispatch` de type `build-app`.

**Payload attendu :**
```json
{
  "event_type": "build-app",
  "client_payload": {
    "url": "https://mon-site.com",
    "name": "MonApp"
  }
}
```

**Étapes du build :**
1. Setup Java 17 + Android SDK + Build Tools 34
2. Installation de Bubblewrap CLI
3. Génération d'un keystore de signature
4. `bubblewrap init` — lecture du manifest du site
5. `bubblewrap build` — compilation de l'APK
6. Signature et vérification de l'APK
7. Upload de l'artefact (rétention 7 jours)

---

## 🔒 Sécurité

- Le token GitHub **ne transite pas par un serveur tiers** — les requêtes partent directement de ton navigateur vers l'API GitHub
- Les mots de passe du keystore sont stockés en **secrets chiffrés GitHub**, jamais en clair
- Utilise un token avec les **permissions minimales** nécessaires
- Le keystore généré est unique par build — conserve-le si tu veux mettre à jour l'app sur le Play Store

---

## 🐛 Dépannage

| Erreur | Cause probable | Solution |
|---|---|---|
| Erreur 401 / 403 | Token invalide ou expiré | Génères-en un nouveau |
| Erreur 404 | Repo introuvable ou privé | Vérifie l'accès au repo |
| `bubblewrap init` échoue | `manifest.json` absent ou mal formé | Vérifie l'URL et le contenu du manifest |
| Build échoué — icône manquante | Pas d'icône 512×512 | Ajoute l'icône et mets à jour le manifest |
| APK généré mais barre Chrome visible | `assetlinks.json` absent | Configure le fichier sur ton serveur |
| Aucun artefact après build | APK non produit | Consulte les logs dans GitHub Actions |

---

## 📄 Licence

MIT — libre d'utilisation, modification et distribution.
