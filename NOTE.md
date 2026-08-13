# 📦 Guide de publication — simplo

Comment travailler sur la librairie et publier une nouvelle version sur npm.

---

## 🔁 Workflow quotidien

### 1. Coder

Modifier la librairie normalement (fonctions, composants, corrections…).

### 2. Commit

```bash
git add .
git commit -m "Description des modifications"
```

⚠️ Le commit est **obligatoire avant la release** (sinon erreur `Git working directory not clean`).

### 3. Release

```bash
npm run release:patch    # correction de bug        → 1.0.2 → 1.0.3
npm run release:minor    # nouvelle fonctionnalité  → 1.0.2 → 1.1.0
npm run release:major    # changement cassant        → 1.0.2 → 2.0.0
```

C'est tout. Cette commande fait automatiquement :

1. Bump de la version dans `package.json`
2. Création du commit de version + du tag `vX.Y.Z`
3. Push du commit et du tag sur GitHub
4. GitHub Actions détecte le tag → build (`tsc`) → publie sur npm (Trusted Publishing / OIDC)

### 4. Vérifier (optionnel)

- Onglet **Actions** du repo GitHub (le run doit être vert)
- Ou en terminal :

```bash
npm view simplo version
```

---

## 🎯 Quelle version choisir ? (SemVer)

| Commande | Quand l'utiliser | Exemple |
|---|---|---|
| `release:patch` | Correction de bug, ajustement interne, typo | `1.0.2 → 1.0.3` |
| `release:minor` | Ajout d'une fonctionnalité (rétrocompatible) | `1.0.2 → 1.1.0` |
| `release:major` | Changement cassant (API modifiée ou supprimée) | `1.0.2 → 2.0.0` |

Pour **ajouter quelque chose** à la librairie → `release:minor` dans la plupart des cas.

---

## ⚙️ Les scripts (dans `package.json`)

À placer dans la section `"scripts"` :

```json
"scripts": {
  "build": "tsc",
  "prepublishOnly": "npm run build",
  "release:patch": "npm version patch && git push --follow-tags",
  "release:minor": "npm version minor && git push --follow-tags",
  "release:major": "npm version major && git push --follow-tags"
}
```

| Script | Rôle |
|---|---|
| `build` | Compile TypeScript → `dist/` (JS + types `.d.ts`) |
| `prepublishOnly` | Lancé automatiquement par npm avant toute publication : garantit un build frais |
| `release:*` | Bump version + tag + push → déclenche la publication automatique |

---

## 🤖 Le workflow GitHub Actions

Fichier : `.github/workflows/publish.yml`

```yaml
name: Publish to npm

on:
  push:
    tags:
      - "v*"

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          registry-url: https://registry.npmjs.org
      - run: npm install -g npm@latest
      - run: npm ci
      - run: npm publish
```

Points importants :

- `on: push: tags: "v*"` → se déclenche au push de tout tag `v1.0.3`, `v2.0.0`, etc.
- `id-token: write` → **indispensable** pour le Trusted Publishing (OIDC)
- `npm install -g npm@latest` → npm ≥ 11.5.1 requis pour le flux OIDC
- Aucun token, aucun secret : l'authentification passe par la liaison de confiance configurée sur npmjs.com

---

## 🔐 Configuration Trusted Publisher (déjà en place)

Sur npmjs.com → package `simplo` → **Settings** → **Trusted Publisher** :

| Champ | Valeur |
|---|---|
| Publisher | GitHub Actions |
| Organization or user | `AnthonyBoschat` |
| Repository | `Simplo` |
| Workflow filename | `publish.yml` |
| Environment | *(vide)* |

⚠️ Le champ `repository` du `package.json` doit correspondre **exactement** au repo GitHub (vérification de provenance) :

```json
"repository": {
  "type": "git",
  "url": "git+https://github.com/AnthonyBoschat/Simplo.git"
}
```

---

## 🚨 Règles d'or

- ❌ **Jamais** de `npm publish` en local — tout passe par GitHub Actions
- ❌ **Jamais** de modification manuelle du champ `version` — c'est `npm version` qui gère
- ❌ **Jamais** re-run un job échoué après avoir corrigé le code — le run rebuild l'ancien tag. Toujours créer une **nouvelle** version (`release:patch`)
- ✅ Toujours **commit avant** `release:*`
- ✅ Les tags sont créés et poussés automatiquement — ne pas les manipuler à la main

---

## 🩺 Dépannage rapide

| Symptôme | Cause | Solution |
|---|---|---|
| `Git working directory not clean` | Modifications non commitées | `git add . && git commit -m "..."` puis relancer |
| Le workflow ne se déclenche pas | Tag pas poussé | `git ls-remote --tags origin` pour vérifier, `git push origin vX.Y.Z` si absent |
| Erreur 404 sur `npm publish` (CI) | Trusted Publisher mal configuré | Vérifier user/repo/workflow **exacts** (casse comprise) sur npmjs.com |
| Erreur 422 provenance `repository.url` | Champ `repository` absent ou différent du repo | Corriger `package.json`, commit, **nouvelle** release |
| Run échoué après correction | Re-run = rebuild de l'ancien tag | `npm run release:patch` pour créer un nouveau tag |