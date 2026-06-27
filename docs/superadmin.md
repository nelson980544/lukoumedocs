# 🛡️ Dashboard Super Admin

Le Super Admin est le seul rôle avec une visibilité totale sur toutes les compagnies. Il accède à son dashboard via `/admin/super` (protégé par `role = 'super_admin'` dans `user_profiles`).

---

## Accès & Authentification

- **URL** : `/admin/super`
- **Rôle requis** : `super_admin` dans `user_profiles`
- **Auth** : Supabase Auth (email/mot de passe)
- Toute tentative d'accès avec un rôle insuffisant redirige vers `/admin`

---

## Vue d'ensemble — Onglets du dashboard

| Onglet | Description |
|--------|-------------|
| **Compagnies** | Liste toutes les compagnies, filtre par statut, actions de validation |
| **Utilisateurs** | Liste tous les `user_profiles` (agents, admins, conducteurs) |
| **Audit** | Journal de toutes les actions admin (`admin_audit_logs`) |

---

## Gestion des compagnies

### Statuts possibles

| Statut | Description | Couleur UI |
|--------|-------------|------------|
| `pending` | Candidature soumise, en attente de traitement | Jaune |
| `pending_approval` | Documents KYB uploadés, en cours de vérification | Orange |
| `active` | Compte actif, peut vendre des billets | Vert |
| `rejected` | Candidature rejetée | Rouge |
| `suspended` | Compte suspendu (accès bloqué) | Gris |

### Statuts KYB

| Statut | Description |
|--------|-------------|
| `pending` | Documents non encore soumis |
| `submitted` | Documents soumis, en attente de vérification |
| `approved` | Documents validés |
| `rejected` | Documents rejetés (raison stockée dans `onboarding_data`) |

---

## Actions disponibles

### Valider une compagnie

Déclenché depuis la **CompanyValidationModal** → bouton "Valider".

**Effets :**
1. `companies.status → 'active'`
2. `companies.kyb_status → 'approved'`
3. Attribution automatique des modules selon le type de compagnie (voir ci-dessous)
4. Appel `POST /api/onboarding/activation` → Email Template C envoyé au contact
5. Log dans `admin_audit_logs` : `action: 'company_validated'`

### Rejeter une compagnie

Déclenché depuis la **CompanyValidationModal** → formulaire de rejet avec raison.

**Effets :**
1. `companies.status → 'rejected'`
2. `companies.kyb_status → 'rejected'`
3. Stockage de la raison dans `companies.onboarding_data.rejection_reason`
4. Appel `POST /api/onboarding/rejection` → Email Template D avec raison
5. Log dans `admin_audit_logs` : `action: 'company_rejected'`

### Changer le statut (actions rapides)

Accessible depuis le menu de chaque compagnie dans la liste :

| Action | Statut résultant |
|--------|-----------------|
| Activer | `active` |
| Suspendre | `suspended` |
| Rejeter | `rejected` |

### Gérer les modules

Via la **CompanyModulesModal** — le super admin peut activer/désactiver chaque module individuellement après l'activation.

---

## Modules automatiques à l'activation

Lors de l'activation (`status → 'active'`), les modules sont automatiquement configurés :

| Module | Bus 🚌 | Bateau 🚢 | Description |
|--------|--------|-----------|-------------|
| `trips` | ✅ | ✅ | Gestion des trajets |
| `guichet` | ✅ | ✅ | Vente au guichet |
| `scanner` | ✅ | ✅ | Scanner QR embarquement |
| `fleet` | ✅ | ✅ | Gestion de la flotte |
| `planning` | ✅ | ✅ | Planification |
| `manifests` | ✅ | ✅ | Manifestes de voyage |
| `passengers` | ✅ | ✅ | Gestion passagers |
| `team` | ✅ | ✅ | Gestion de l'équipe |
| `stats` | ✅ | ✅ | Statistiques |
| `reports` | ✅ | ✅ | Rapports |
| `alerts` | ✅ | ✅ | Alertes |
| `support` | ✅ | ✅ | Support |
| `infovoyageurs` | ✅ | ✅ | Info voyageurs |
| `coupons` | ❌ | ❌ | Coupons de réduction |
| `yield` | ❌ | ❌ | Yield management |
| `widget` | ❌ | ❌ | Widget intégrable |
| `extras` | ❌ | ❌ | Options supplémentaires |

> Le super admin peut ajuster ces modules à tout moment via la CompanyModulesModal.

---

## Invitations manuelles

Le super admin peut envoyer manuellement une invitation à une compagnie existante via le bouton "Inviter" dans le dashboard.

**Données requises :**
- Email de contact
- Nom de la compagnie
- Token d'invitation (généré automatiquement)

**Action :** Appel `POST /api/onboarding/invite`

---

## Journal d'audit (`admin_audit_logs`)

Toutes les actions du super admin sont tracées :

| Colonne | Description |
|---------|-------------|
| `action` | Ex : `company_validated`, `company_rejected`, `status_changed` |
| `target_type` | Ex : `company` |
| `target_id` | UUID de la cible |
| `details` | JSONB avec contexte (raison, ancien statut, etc.) |
| `created_at` | Horodatage |

---

## Schéma de données — `companies`

| Colonne | Type | Description |
|---------|------|-------------|
| `id` | `UUID` | Clé primaire |
| `name` | `TEXT` | Nom commercial |
| `type` | `TEXT` | `bus` \| `boat` |
| `status` | `TEXT` | `pending` \| `pending_approval` \| `active` \| `rejected` \| `suspended` |
| `kyb_status` | `TEXT` | `pending` \| `submitted` \| `approved` \| `rejected` |
| `owner_id` | `UUID` | FK → `auth.users.id` (company_admin principal) |
| `contact_email` | `TEXT` | Email de contact principal |
| `enabled_modules` | `JSONB` | Modules activés/désactivés |
| `onboarding_data` | `JSONB` | Données de progression (contract_signed, signed_at, rejection_reason…) |
| `created_at` | `TIMESTAMPTZ` | Date de création |

---

# 🏢 Dashboard Company Admin

Le Company Admin gère sa propre compagnie via `/admin`. Accès protégé par `role = 'company_admin'` dans `user_profiles`.

---

## Accès & Authentification

- **URL** : `/admin`
- **Rôle requis** : `company_admin` (ou `super_admin`)
- **Auth** : Supabase Auth
- Un `company_admin` ne voit que les données de **sa propre compagnie** (isolation multi-tenant)

---

## Wizard de configuration initial (`/admin/setup`)

Affiché après la première connexion, avant d'accéder au dashboard principal.

### Étapes

| Étape | Contenu | Condition de passage |
|-------|---------|---------------------|
| **0** — Bienvenue | Présentation des 3 étapes à venir | Clic sur "Commencer" |
| **1** — Lieux | Création de la première gare/port (ville, adresse, type) | Au moins 1 lieu créé |
| **2** — Équipe | Création du premier agent (nom, identifiant) | Au moins 1 agent créé |
| **3** — Documents KYB | Upload NIF, Patente de Transport, Assurance RC | 3 documents uploadés |
| **4** — Attente | Écran d'attente de validation (status `pending_approval`) | Validation par le super admin |

### Logique de redirection

```
company.status = 'active' ET lieux + agents créés → /admin (dashboard)
company.status ≠ 'active' ET lieux + agents créés → Étape 4 (attente)
Pas de lieux → Étape 1
Lieux créés, pas d'agents → Étape 2
```

---

## Signature électronique du contrat (`/admin/signup`)

La signature a lieu **lors de la création du compte** (`/admin/signup?token=xxx`), après le formulaire d'inscription.

### Flux en 4 phases

```
Phase 1 : form      → Formulaire d'inscription (email, mot de passe, nom)
Phase 2 : contract  → Lecture du contrat de partenariat + saisie du nom du signataire
Phase 3 : otp       → Réception du code OTP par email + saisie
Phase 4 : signed    → Confirmation avec hash SHA-256 + redirection vers /admin/setup
```

### Contenu du contrat (v1.0-2026)

Le contrat est **personnalisé par type** (bus 🚌 / bateau 🚢) et inclut :

| Article | Contenu |
|---------|---------|
| 1 — Objet | Référencement sur la plateforme TripGabon |
| 2 — Accès plateforme | Outils mis à disposition (dashboard, scanner, guichet…) |
| 3 — Commission billets | **3 % HT** du prix passager (web & mobile) |
| 4 — Commission options | Montant fixe FCFA par option vendue (variable selon l'option) |
| 5 — Reversements | Règlement sous 7 jours ouvrés après encaissement |
| 6 — Obligations compagnie | Service, horaires, SAV |
| 7 — Obligations TripGabon | Support, disponibilité plateforme |
| 8 — Données | RGPD / droit gabonais |
| 9 — Durée & Résiliation | Durée indéterminée, préavis 30 jours |
| 10 — Droit applicable | Droit de la République Gabonaise |

### Preuve légale

Après signature réussie, le signataire reçoit par email :
- Son **nom complet**
- Son **email**
- La **date et heure exacte** de signature
- Son **adresse IP**
- L'**empreinte SHA-256** unique (`company_contracts.signature_hash`)
- Le **récapitulatif des conditions** (commissions, options)

---

## Modules disponibles

Les modules sont gérés par le super admin. Chaque module correspond à une section du dashboard :

| Module | Section UI |
|--------|------------|
| `trips` | Gestion des trajets |
| `guichet` | Vente au guichet |
| `scanner` | Scanner d'embarquement |
| `fleet` | Flotte (bus/bateaux) |
| `planning` | Planning des rotations |
| `manifests` | Manifestes voyageurs |
| `passengers` | Base passagers |
| `team` | Agents & conducteurs |
| `stats` | Statistiques de ventes |
| `reports` | Rapports exportables |
| `alerts` | Alertes opérationnelles |
| `support` | Tickets support |
| `infovoyageurs` | Info voyageurs (retards, annulations) |
| `coupons` | Coupons de réduction |
| `yield` | Tarification dynamique |
| `widget` | Widget intégrable sur site externe |
| `extras` | Options supplémentaires (bagages, repas…) |

---

## Gestion de l'équipe

Via `POST /api/admin/create-agent` — voir [docs/admin.md](./admin.md) pour le détail des routes.

| Rôle | Accès |
|------|-------|
| `company_admin` | Dashboard complet, gestion des agents |
| `agent` | Guichet, scanner, manifestes |
| `driver` | Vue conducteur, planning |

> Un `company_admin` ne peut créer des comptes que dans **sa propre compagnie**.
