# 🚢 Documentation — Lukoumé / TripGabon

Documentation complète des **45 routes API** et des interfaces admin de la plateforme TripGabon.

## 🗂️ Table des matières

| Module | Description | Routes |
|--------|-------------|--------|
| [🛡️ Super Admin & Company Admin](./docs/superadmin.md) | Dashboards, onboarding, modules, signature contrat | — |
| [📇 CRM interne Lukoumé](./docs/crm.md) | Prospection & suivi des compagnies clientes (réservé super_admin) | — |
| [📐 CRM — Référence schéma](./docs/crm-schema.md) | Détail colonne par colonne des tables `crm_*` | — |
| [👤 Agents & Administrateurs](./docs/admin.md) | Création et gestion des comptes agents/conducteurs | 3 |
| [🏢 Onboarding & E-Signature](./docs/onboarding.md) | Auto-inscription, KYB, OTP, contrat électronique | 8 |
| [💳 Paiements](./docs/paiements.md) | Stripe, SingPay, E-Billing (Airtel/Moov), Lygos | 7 |
| [📬 Notifications](./docs/notifications.md) | Emails, Telegram, WhatsApp, Support | 10 |
| [🎟️ Billetterie](./docs/billetterie.md) | Signature et vérification QR codes | 3 |
| [📦 Réservations](./docs/reservations.md) | Statut, voyages, Google Wallet | 5 |
| [🤖 Intelligence Artificielle](./docs/ia.md) | Insights IA et chat contextuel | 2 |
| [📊 Statistiques & Crons](./docs/stats-crons.md) | Récapitulatifs et rappels automatiques | 2 |
| [🎁 Fidélité & Push](./docs/fidelite-push.md) | Coupons de fidélité et abonnements push | 2 |
| [🔍 Metadata & Découverte](./docs/metadata.md) | OIDC, OAuth, Markdown | 3 |

---

## 🔐 Authentification

La grande majorité des routes nécessite un **token Bearer JWT Supabase** :

```http
Authorization: Bearer <supabase_jwt_token>
```

### Types de protection

| Type | Header | Utilisé par |
|------|--------|-------------|
| Bearer JWT Supabase | `Authorization: Bearer <token>` | Routes admin, IA, billetterie |
| CRON_SECRET | `Authorization: Bearer <CRON_SECRET>` | Crons Vercel |
| Webhook secret | `x-webhook-secret` ou `x-api-secret` | Callbacks paiement |
| Signature HMAC Stripe | `stripe-signature` | Webhook Stripe |
| Bot secret interne | `x-bot-secret` | Bot WhatsApp vente |
| Telegram secret | `X-Telegram-Bot-Api-Secret-Token` | Webhook Telegram |

---

## 🌍 Contexte marché

La plateforme est conçue pour le **marché gabonais** :

- 💰 Devise : **XAF (FCFA)** — tous les montants sont en francs CFA
- 📱 Paiements prioritaires : **Airtel Money** (07x) & **Moov Money** (06x)
- 🌐 Connectivité : optimisé pour les réseaux **3G/4G instables**
- 📲 Canal principal : **WhatsApp** et **Telegram** pour les notifications agents

---

## ⚙️ Variables d'environnement

<details>
<summary>Voir toutes les variables requises</summary>

| Variable | Description |
|----------|-------------|
| `NEXT_PUBLIC_SUPABASE_URL` | URL du projet Supabase |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Clé publique Supabase |
| `SUPABASE_SERVICE_ROLE_KEY` | Clé service Supabase (admin) |
| `STRIPE_SECRET_KEY` | Clé secrète Stripe |
| `STRIPE_WEBHOOK_SECRET` | Secret de vérification webhook Stripe |
| `SINGPAY_CLIENT_ID` | ID client SingPay |
| `SINGPAY_CLIENT_SECRET` | Secret SingPay |
| `SINGPAY_WALLET_ID` | ID portefeuille SingPay |
| `SINGPAY_WEBHOOK_SECRET` | Secret webhook SingPay |
| `EBILLING_ENV` | `prod` ou `lab` |
| `EBILLING_CLIENT_ID` | OAuth2 client ID E-Billing |
| `EBILLING_CLIENT_SECRET` | OAuth2 client secret E-Billing |
| `EBILLING_WEBHOOK_SECRET` | Secret webhook E-Billing |
| `EBILLING_ALLOWED_IPS` | IP whitelist E-Billing (séparées par virgule) |
| `SHAP_API_ID` | ID API SHAP (payout Mobile Money) |
| `SHAP_API_SECRET` | Secret API SHAP |
| `LYGOS_API_KEY` | Clé API Lygos |
| `TELEGRAM_BOT_TOKEN` | Token du bot Telegram |
| `TELEGRAM_CHAT_ID` | Chat ID Telegram de notification |
| `TELEGRAM_WEBHOOK_SECRET` | Secret de validation webhook Telegram |
| `OPENWA_WEBHOOK_SECRET` | Secret HMAC webhook WhatsApp |
| `WHATSAPP_BOT_SECRET` | Secret d'accès interne au bot |
| `NVIDIA_API_KEY` | Clé API NVIDIA (Llama 3.1 70B) |
| `GOOGLE_WALLET_ISSUER_ID` | Issuer ID Google Wallet |
| `GOOGLE_WALLET_PRIVATE_KEY` | Clé privée RSA Google Wallet |
| `GOOGLE_WALLET_SERVICE_ACCOUNT_EMAIL` | Email compte de service Google |
| `TICKET_HMAC_SECRET` | Secret HMAC pour signatures de billets (min 32 chars) |
| `CRON_SECRET` | Secret d'accès aux endpoints cron Vercel |
| `NEXT_PUBLIC_APP_URL` | URL de base de l'application |
| `RESEND_API_KEY` | Clé API Resend (emails transactionnels) |

</details>

---

## 📟 Codes HTTP courants

| Code | Signification |
|------|---------------|
| `200` | Succès |
| `400` | Requête invalide (champs manquants ou format incorrect) |
| `401` | Non authentifié (token absent ou invalide) |
| `403` | Accès refusé (rôle insuffisant ou mauvaise compagnie) |
| `404` | Ressource introuvable |
| `500` | Erreur serveur interne |
| `502` | Erreur gateway (API tierce indisponible) |
| `503` | Service indisponible (configuration manquante) |

---

## 🔄 Flux de paiement

```
Client → POST /api/<provider>/init
              ↓ retourne une URL de redirection
         Client redirigé vers le portail de paiement
              ↓ (après confirmation par l'opérateur)
Provider → POST /api/<provider>/callback  [webhook sécurisé]
              ↓
         booking.status = "paid"
         decrement_seats (si pas déjà fait)
         Commission calculée via calculateCommission()
         Solde compagnie mis à jour
         Payout SHAP vers Mobile Money compagnie
         Notification Telegram admin
         Email confirmation client
         Notification push PWA client
```

---

## 🏗️ Stack technique

- **Framework** : Next.js 14+ (App Router)
- **Base de données** : Supabase (PostgreSQL + Auth + Storage)
- **Paiements** : Stripe · SingPay · E-Billing · Lygos
- **Notifications** : Resend (email) · Telegram Bot API · OpenWA (WhatsApp)
- **IA** : NVIDIA API (meta/llama-3.1-70b-instruct)
- **Billets** : HMAC-SHA256 (Web Crypto API) + Google Wallet JWT
- **Déploiement** : Vercel (Hobby → pas de crons sub-daily)
- **Langue** : TypeScript strict

---

*Maintenu par l'équipe TripGabon · Dernière mise à jour : 27 Juin 2026*
