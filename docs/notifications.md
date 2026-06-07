# 📬 API — Notifications & Communication

Gestion des emails, bots Telegram et WhatsApp, support client et onboarding.

---

## Emails

### `POST /api/send-email`

Envoie l'email de confirmation de réservation avec le billet PDF en pièce jointe (via **Resend**).

#### Authentification
```
Authorization: Bearer <jwt_supabase>
```
L'utilisateur authentifié doit être le payeur de la réservation, et celle-ci doit être en statut `paid`.

#### Corps de la requête

```json
{ "bookingId": 42 }
```

#### Réponse `200`

```json
{ "success": true }
```

---

### `GET /api/test-email`

Envoie un email de test pour vérifier la configuration Resend.

#### Authentification
```
Authorization: Bearer <CRON_SECRET>
```

#### Réponse `200`

```json
{ "success": true, "message": "Email de test envoyé" }
```

---

### `POST /api/onboarding/notify`

Notifie les super-admins par email lors d'une nouvelle **demande d'onboarding** d'une compagnie.

#### Authentification
Aucune (route publique). Toutes les données sont validées via **Zod** et échappées (anti-XSS).

#### Corps de la requête

```json
{
  "companyName": "Trans-Gabon Express",
  "contactName": "Marie Nzé",
  "contactEmail": "marie@transgabon.com",
  "contactPhone": "074000000",
  "message": "Nous souhaitons rejoindre la plateforme."
}
```

| Champ | Requis | Description |
|-------|--------|-------------|
| `companyName` | ✅ | Nom de la compagnie |
| `contactName` | ✅ | Nom du contact |
| `contactEmail` | ✅ | Email valide |
| `contactPhone` | ❌ | Téléphone |
| `message` | ❌ | Message libre (max 1000 caractères) |

#### Réponse `200`

```json
{ "success": true }
```

---

### `POST /api/onboarding/invite`

Envoie un **email d'invitation** pour l'activation d'un compte admin compagnie.

#### Authentification
```
Authorization: Bearer <jwt_supabase>
```
Rôle requis : `super_admin`

#### Corps de la requête

```json
{
  "email": "admin@compagnie.com",
  "companyName": "Trans-Gabon Express",
  "inviteToken": "token-unique-uuid"
}
```

#### Réponse `200`

```json
{ "success": true }
```

L'email contient un lien d'inscription avec le token pré-rempli.

---

### `POST /api/support/notify`

Envoie des notifications liées au support :
- **Nouveau ticket** → email aux admins TripGabon
- **Réponse admin** → email à la compagnie concernée

#### Authentification
```
Authorization: Bearer <jwt_supabase>
```
Pour les réponses admin, le rôle `super_admin` est requis.

#### Corps de la requête

```json
{
  "type": "new_ticket",
  "ticketId": "uuid",
  "subject": "Problème de paiement",
  "message": "Le paiement n'a pas été confirmé.",
  "companyId": "uuid-compagnie"
}
```

| `type` | Description |
|--------|-------------|
| `"new_ticket"` | Nouveau ticket créé par une compagnie |
| `"admin_reply"` | Réponse de l'admin TripGabon |

#### Réponse `200`

```json
{ "success": true }
```

---

## WhatsApp

### `POST /api/whatsapp/notify`

Envoie une notification de réservation confirmée via le **bot WhatsApp Railway**.

#### Authentification
```
Authorization: Bearer <jwt_supabase>
```

#### Corps de la requête

```json
{ "bookingId": 42 }
```

Les détails du booking (passagers, trajet, montant) sont récupérés automatiquement depuis Supabase.

---

### `POST /api/whatsapp/webhook`

Reçoit les messages entrants via **OpenWA**.

#### Authentification
Signature HMAC-SHA256 via header `x-openwa-signature` + `OPENWA_WEBHOOK_SECRET`

> Les messages de groupe sont ignorés. Seules les conversations privées sont traitées.

#### Réponse `200`

```json
{ "ok": true }
```

---

### `POST /api/whatsapp/bot/auth`

Génère un token de liaison à 6 chiffres (expire dans 10 minutes) pour connecter un compte WhatsApp à un compte TripGabon.

#### Authentification
```
Authorization: Bearer <jwt_supabase>
```

#### Réponse `200`

```json
{
  "token": "483920",
  "expiresAt": "2026-01-15T10:10:00Z"
}
```

---

## Telegram

### `POST /api/telegram/webhook`

Reçoit les messages et commandes du **bot Telegram**.

#### Authentification
Header `X-Telegram-Bot-Api-Secret-Token` = `TELEGRAM_WEBHOOK_SECRET`

#### Commandes bot disponibles

| Commande | Rôle requis | Description |
|----------|-------------|-------------|
| `/start <token>` | Tous | Lie le compte Telegram au compte TripGabon |
| `1` | Agent / Admin | Voir les voyages du jour |
| `2` | Agent / Admin | Voir les réservations récentes |
| `3` | Super Admin | Récapitulatif statistiques complet (toutes compagnies) |
| Vente guidée | Agent | Processus de vente étape par étape |

#### Réponse `200`

```json
{ "ok": true }
```

---

### `GET /api/telegram/setup-webhook`

Récupère les informations du webhook Telegram actuellement configuré.

#### Authentification
```
Authorization: Bearer <jwt_supabase>
```
Rôle requis : `super_admin`

#### Réponse `200`

```json
{
  "url": "https://tripgabonadmin.vercel.app/api/telegram/webhook",
  "has_custom_certificate": false,
  "pending_update_count": 0
}
```

---

### `POST /api/telegram/setup-webhook`

Configure le webhook Telegram pour pointer vers cette instance. L'URL est **détectée automatiquement** depuis les headers de la requête.

#### Authentification
```
Authorization: Bearer <jwt_supabase>
```
Rôle requis : `super_admin`

#### Réponse `200`

```json
{
  "success": true,
  "url": "https://tripgabonadmin.vercel.app/api/telegram/webhook"
}
```

---

### `POST /api/telegram/bot/auth`

Génère un token de liaison à 6 chiffres pour connecter Telegram à TripGabon.

#### Authentification
```
Authorization: Bearer <jwt_supabase>
```

#### Réponse `200`

```json
{
  "token": "274819",
  "expiresAt": "2026-01-15T10:10:00Z"
}
```

---

### `GET /api/test-telegram`

Envoie un message de test dans le chat Telegram configuré.

#### Authentification
```
Authorization: Bearer <CRON_SECRET>
```

#### Réponse `200`

```json
{ "success": true }
```
