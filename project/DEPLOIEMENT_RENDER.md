# Déploiement sur Render — LCAPP

Ce guide déploie le **front-end** LCAPP (Vite + React) en tant que **Static Site**
sur [Render](https://render.com).

- **Front-end** → Render (statique, gratuit).
- **Back-end** → **Supabase** (base + Edge Functions), inchangé. À configurer
  **avant** le front (le front a besoin de l'URL et de la clé Supabase).

Un fichier [`render.yaml`](render.yaml) (Blueprint) est déjà présent à la racine :
Render lit tout automatiquement (build, dossier de sortie, réécritures SPA) et
te demande seulement les 2 secrets Supabase.

---

## Prérequis

- Le repo poussé sur **GitHub / GitLab** (Render déploie depuis un repo Git).
- Un compte [Render](https://render.com) (connexion via GitHub recommandée).
- Le back-end **Supabase** opérationnel (cf. `DEPLOIEMENT_VERCEL.md` §1 —
  migrations, Edge Functions, secret Resend). Les étapes Supabase sont
  identiques quel que soit l'hébergeur du front.

---

## 1. Pousser le repo sur GitHub (si pas déjà fait)

```bash
# créer un repo vide sur GitHub, puis dans le dossier du projet :
git add render.yaml DEPLOIEMENT_RENDER.md .env.example
git commit -m "Ajout config de déploiement Render"
git remote add origin https://github.com/<user>/<repo>.git
git push -u origin main
```

---

## 2. Déployer via le Blueprint (recommandé)

1. Aller sur [dashboard.render.com/blueprints](https://dashboard.render.com/blueprints)
   → **New Blueprint Instance**.
2. Sélectionner le repo GitHub du projet.
3. Render détecte `render.yaml` et propose de créer le service **lcapp** (Static Site).
4. Il demande les 2 variables `sync: false` — les renseigner :

   | Variable                 | Valeur                          | Source                    |
   | ------------------------ | ------------------------------- | ------------------------- |
   | `VITE_SUPABASE_URL`      | `https://xxxxxxxx.supabase.co`  | Supabase → Settings → API |
   | `VITE_SUPABASE_ANON_KEY` | `eyJ...` (clé anon publique)    | Supabase → Settings → API |

5. **Apply** → Render build (`npm ci && npm run build`) puis publie `dist/`.

> Ne jamais mettre la clé `service_role` ni la clé Resend ici : elles restent
> en secrets Supabase.

---

## 3. (Alternative) Sans Blueprint, création manuelle

Si tu préfères configurer à la main plutôt que via `render.yaml` :

1. Dashboard Render → **New +** → **Static Site**.
2. Connecter le repo.
3. Réglages :
   - **Build Command** : `npm ci && npm run build`
   - **Publish Directory** : `dist`
4. **Environment** → ajouter `VITE_SUPABASE_URL` et `VITE_SUPABASE_ANON_KEY`.
5. (Optionnel, SPA) **Redirects/Rewrites** → `Source: /*`, `Destination: /index.html`,
   `Action: Rewrite`.
6. **Create Static Site**.

---

## 4. Déploiements continus

Une fois le service créé, **chaque `git push`** sur la branche suivie
redéclenche automatiquement le build + déploiement. Les PR peuvent générer des
*Preview Environments* (selon le plan).

---

## 5. Vérifications post-déploiement

- [ ] La page se charge sans erreur console `Missing Supabase environment variables`.
- [ ] Inscription / connexion fonctionnent.
- [ ] Suppression de compte → Edge Function `delete-account` + e-mail Resend.
- [ ] Notifications e-mail (`send-notification-email`) reçues.
- [ ] Images produits affichées.

---

## Rappel sécurité

- Seules les variables `VITE_*` (URL + clé **anon**) vont sur Render.
- Clés `service_role` et **Resend** → uniquement en secrets Supabase.
- `.env` local ignoré par git ; utiliser `.env.example` comme modèle.
