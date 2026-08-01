# Déploiement sur Vercel — LCAPP

Ce guide déploie l'application **LCAPP** :

- **Front-end** : Vite + React + TypeScript → hébergé sur **Vercel** (site statique).
- **Back-end** : **Supabase** (base de données + Edge Functions). Vercel n'héberge
  *pas* le back-end ; Supabase doit être opérationnel **avant** de déployer le front.

> Ordre impératif : **1) Supabase → 2) Vercel**. Le front a besoin de l'URL et de la
> clé Supabase pour fonctionner.

---

## Prérequis

- Un compte [Vercel](https://vercel.com) (connexion via GitHub recommandée).
- Un compte [Supabase](https://supabase.com).
- Un compte [Resend](https://resend.com) (envoi d'e-mails par les Edge Functions).
- Le repo poussé sur un remote git (GitHub / GitLab / Bitbucket) — voir §4.
- En local : Node 20+ et npm.

---

## 1. Back-end Supabase

### 1.1 Créer le projet

1. [app.supabase.com](https://app.supabase.com) → **New project**.
2. Noter le mot de passe de la base (généré à la création).
3. Une fois créé, aller dans **Project Settings → API** et récupérer :
   - **Project URL** → servira de `VITE_SUPABASE_URL`
   - **anon public** → servira de `VITE_SUPABASE_ANON_KEY`
   - **service_role** (secret) → utilisé côté serveur uniquement, jamais dans le front.

### 1.2 Installer la CLI Supabase et se lier au projet

```bash
npm install -g supabase
supabase login
supabase link --project-ref <PROJECT_REF>   # le ref est dans l'URL du dashboard
```

### 1.3 Appliquer les migrations (schéma + données initiales)

Le dossier `supabase/migrations/` contient l'ensemble du schéma (tables, RLS,
rangs, produits, groupes, etc.).

```bash
supabase db push
```

### 1.4 Déployer les Edge Functions

```bash
supabase functions deploy delete-account
supabase functions deploy send-notification-email
```

### 1.5 Configurer le secret Resend

Les deux fonctions envoient des e-mails via Resend.
`SUPABASE_URL` et `SUPABASE_SERVICE_ROLE_KEY` sont fournis automatiquement par
le runtime Supabase ; seule la clé Resend est à ajouter :

```bash
supabase secrets set RESEND_API_KEY=<TA_CLE_RESEND>
```

### 1.6 (Selon besoin) Images produits

La migration `..._create_product_images_storage.sql` crée un bucket Storage pour
les images produits. Si les visuels doivent être servis depuis Supabase plutôt
que depuis `public/`, uploader les images dans ce bucket.

---

## 2. Front-end sur Vercel

### 2.1 Variables d'environnement à définir

Dans Vercel → projet → **Settings → Environment Variables**, ajouter (pour les
environnements *Production*, *Preview* et *Development*) :

| Nom                      | Valeur                              | Source                         |
| ------------------------ | ----------------------------------- | ------------------------------ |
| `VITE_SUPABASE_URL`      | `https://xxxxxxxx.supabase.co`      | Supabase → Settings → API      |
| `VITE_SUPABASE_ANON_KEY` | `eyJ...` (clé anon publique)        | Supabase → Settings → API      |

> Vite n'expose au navigateur **que** les variables préfixées par `VITE_`.
> Ne jamais mettre la clé `service_role` ici.

### 2.2 Réglages de build (détectés automatiquement)

Vercel reconnaît Vite. Vérifier dans **Settings → Build & Development** :

- **Framework Preset** : `Vite`
- **Build Command** : `npm run build`
- **Output Directory** : `dist`
- **Install Command** : `npm ci`

---

## 3. Déployer

### Option A — via l'interface Vercel (recommandé)

1. [vercel.com/new](https://vercel.com/new) → **Import Git Repository** → choisir le repo.
2. Renseigner les variables d'environnement du §2.1.
3. **Deploy**.
4. Chaque `git push` sur la branche de production redéploie automatiquement ;
   les autres branches / PR génèrent des *Preview Deployments*.

### Option B — via la CLI Vercel

```bash
npm install -g vercel
vercel login
vercel            # premier déploiement (preview) + configuration du projet
vercel --prod     # déploiement en production
```

Ajouter les variables avec `vercel env add VITE_SUPABASE_URL` puis
`vercel env add VITE_SUPABASE_ANON_KEY` (ou via le dashboard).

---

## 4. Pousser le repo sur GitHub (si pas déjà fait)

```bash
# créer un repo vide sur GitHub, puis :
git remote add origin https://github.com/<user>/<repo>.git
git push -u origin main
```

---

## 5. Vérifications post-déploiement

- [ ] La page se charge sans erreur console `Missing Supabase environment variables`.
- [ ] Inscription / connexion fonctionnent (auth Supabase joignable).
- [ ] La suppression de compte déclenche bien l'Edge Function `delete-account`
      (et l'e-mail Resend part).
- [ ] Les notifications e-mail (`send-notification-email`) arrivent.
- [ ] Les images produits s'affichent.

---

## Rappel sécurité

- Seules les variables `VITE_*` (URL + clé **anon**) vont côté front / Vercel.
- La clé **service_role** et la clé **Resend** restent uniquement en secrets
  Supabase — jamais dans le repo ni dans Vercel.
- Le fichier `.env` local est ignoré par git ; utiliser `.env.example` comme modèle.
