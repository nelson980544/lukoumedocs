# 💳 API — Paiements

TripGabon supporte 4 méthodes de paiement adaptées au marché gabonais.

---

## Flux général

```
Client → POST /api/<provider>/init  →  URL de redirection
                                          ↓
                                  Client paie sur le portail
                                          ↓
Provider → POST /api/<provider>/callback  (webhook sécurisé)
                                          ↓
                              booking → "paid"
                              decrement_seats
                              Commission → calculateCommission()
                              Payout SHAP → Mobile Money compagnie
                              Notification Telegram + Email client
```

---

## `POST /api/stripe/init`

Initialise une session Stripe Checkout (paiement par carte bancaire).

### Authentification
Aucune authentification requise.

### Corps de la requête

```json
{
  "amount": 15000,
  "email": "client@example.com",
  "bookingId": 42,
  "tripId": 10,
  "source": "web"
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `amount` | `number` | ✅ | Montant en **XAF** (devise zéro-décimale Stripe — `15000` = 15 000 FCFA) |
| `email` | `string` | ❌ | Email pré-rempli dans l'interface Stripe |
| `bookingId` | `number` | ✅ | ID de la réservation |
| `tripId` | `number` | ✅ | ID du voyage |
| `source` | `string` | ❌ | Origine (`"web"`, `"admin"`, etc.) — repris dans l'URL de succès |

### Réponse `200`

```json
{
  "url": "https://checkout.stripe.com/pay/cs_test_..."
}
```

Rediriger le client vers `url`. Après paiement :
- ✅ Succès → `/booking/success?id=<encodedId>&method=stripe`
- ❌ Annulation → `/book/<tripId>?error=cancelled`

---

## `POST /api/stripe/webhook`

Webhook Stripe — confirmation automatique de paiement.

### Authentification
Signature HMAC via header `stripe-signature` + variable `STRIPE_WEBHOOK_SECRET`

> ⚠️ Ce endpoint est appelé **directement par Stripe**, pas par votre frontend.

### Événement traité : `checkout.session.completed`

Pipeline déclenché dans l'ordre :
1. `bookings` → `status: "paid"`, `payment_method: "stripe"`
2. Calcul commission via `calculateCommission()`
3. Mise à jour `companies.account_balance`
4. Notification Telegram
5. Email de confirmation (`/utils/email-service.ts`)

### Réponse `200`

```json
{ "received": true }
```

---

## `POST /api/singpay/init`

Initialise un paiement Mobile Money via **SingPay** (Airtel / Moov Gabon).

### Authentification
Aucune.

### Corps de la requête

```json
{
  "amount": 15000,
  "phone": "074000000",
  "bookingId": 42,
  "source": "web"
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `amount` | `number` | Montant en XAF |
| `phone` | `string` | Numéro de téléphone Airtel ou Moov |
| `bookingId` | `number` | ID de la réservation |
| `source` | `string` | Optionnel — source pour la redirection |

### Réponse `200`

```json
{
  "success": true,
  "redirectUrl": "https://gateway.singpay.ga/pay/...",
  "transactionId": "txn_abc123",
  "message": "Redirection vers le paiement"
}
```

---

## `POST /api/singpay/callback`

Webhook SingPay — confirmation de paiement Mobile Money.

### Authentification
- Header `x-webhook-secret` correspondant à `SINGPAY_WEBHOOK_SECRET`
- IP allowlist (`SINGPAY_ALLOWED_IPS`)

### Idempotence
Si le booking est déjà `paid`, le webhook retourne `200` sans rien modifier.

### Pipeline déclenché
1. Vérification montant (anti sous-paiement)
2. `bookings` → `status: "paid"`, `payment_method: "singpay"`
3. Commission via `calculateCommission()`
4. Payout SHAP vers Mobile Money compagnie
5. Notification Telegram
6. Email client

---

## `POST /api/ebilling/init`

Initialise une facture **E-Billing** et envoie un **USSD Push** (demande de code PIN directement sur le téléphone du client — sans redirection).

### Authentification
Aucune. Le montant est **toujours récupéré depuis la base de données**, jamais depuis le client.

### Corps de la requête

```json
{
  "phone": "074000000",
  "bookingId": 42,
  "source": "admin"
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `phone` | `string` | Numéro (07x = Airtel, 06x = Moov — détection automatique) |
| `bookingId` | `number` | ID de la réservation |
| `source` | `string` | Optionnel |

### Fonctionnement interne

```
1. Récupère le montant réel depuis bookings (sécurité côté serveur)
2. Détecte l'opérateur automatiquement (airtelmoney / moovmoney4)
3. Crée la facture E-Billing (scope OAuth: invoice:create)
4. Envoie le USSD Push (scope OAuth: payment:create)
   → Le client reçoit une invite de paiement sur son téléphone
5. Retourne billId + redirectUrl (portail de repli si push échoue)
```

### Réponse `200`

```json
{
  "success": true,
  "billId": "BILL-123456",
  "redirectUrl": "https://billing-easy.net?invoice=BILL-123456&...",
  "pushSent": true
}
```

---

## `POST /api/ebilling/callback`

Webhook E-Billing — confirmation de paiement Mobile Money.

### Authentification
- Header `x-webhook-secret` + IP allowlist (`EBILLING_ALLOWED_IPS`)
- En production sans protection configurée → `503` (fail-closed)

### Corps reçu (depuis E-Billing)

```json
{
  "reference": "42",
  "transactionid": "TXN-XYZ",
  "paymentsystem": "airtelmoney",
  "amount": 15000,
  "billingid": "BILL-123456"
}
```

### Vérifications de sécurité

| Contrôle | Description |
|----------|-------------|
| Statut `pending` | Si booking déjà `paid` → doublon ignoré (idempotence) |
| Montant confirmé ≥ montant attendu | Anti sous-paiement |
| Double-lock SQL | `WHERE status = 'pending'` dans l'UPDATE |

### Pipeline déclenché

1. `bookings` → `status: "paid"`, `payment_method: "ebilling"`
2. `decrement_seats` (sauf si déjà fait côté guichet — flag `seats_decremented`)
3. Commission via `calculateCommission()`
4. Notification Telegram
5. Notification push PWA client
6. Récompenses fidélité (`issueLoyaltyRewards`)
7. Payout SHAP vers Mobile Money compagnie

### Réponse `200`

```json
{ "success": true }
```

> ⚠️ HTTP 200 **obligatoire** pour acquitter la transaction côté E-Billing (sinon E-Billing retentera le webhook).

---

## `POST /api/lygos/init`

Initialise un paiement via **Lygos** (opérateur Mobile Money alternatif).

### Authentification
Aucune.

### Corps de la requête

```json
{
  "amount": 15000,
  "phone": "074000000",
  "bookingId": 42
}
```

### Réponse `200`

```json
{
  "success": true,
  "redirectUrl": "https://pay.lygos.app/..."
}
```

> Le numéro de téléphone est normalisé avant envoi (suppression de l'indicatif +241, des espaces, etc.).
