# 🎟️ API — Billetterie & QR Codes

La billetterie repose sur une signature **HMAC-SHA256** (Web Crypto API, compatible Edge Runtime et Node.js 18+). Chaque billet a un code au format `bookingId-passengerId-signature` (signature tronquée à 8 hex).

---

## Format d'un code QR

```
42-7-a3f8c91b
│  │  └── Signature HMAC-SHA256 (8 hex), clé = TICKET_HMAC_SECRET
│  └───── passengerId (entier BIGINT)
└──────── bookingId (entier BIGINT)
```

> ⚠️ Le format hérité à 2 parties (`bookingId-passengerId`) est **refusé** — seuls les codes signés sont acceptés.

---

## `POST /api/tickets/sign`

Génère une signature HMAC pour un billet passager. Utilisé lors de l'émission du QR code.

### Authentification
```
Authorization: Bearer <jwt_supabase>
```

L'appelant doit être soit **le propriétaire de la réservation**, soit un **agent/admin de la compagnie** concernée.

### Corps de la requête

```json
{
  "bookingId": 42,
  "passengerId": 7
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `bookingId` | `number` (entier positif) | ID de la réservation |
| `passengerId` | `number` (entier positif) | ID du passager |

### Réponse `200`

```json
{
  "signature": "a3f8c91b"
}
```

Le code QR final à encoder est : `42-7-a3f8c91b`

### Erreurs

| Code | Raison |
|------|--------|
| `400` | `bookingId` ou `passengerId` manquant ou non entier positif |
| `401` | Token absent ou session invalide |
| `403` | L'utilisateur n'est ni propriétaire du booking ni agent de la compagnie |
| `404` | Passager introuvable pour ce booking |
| `500` | Secret HMAC absent ou trop court (< 32 caractères) |

---

## `POST /api/tickets/verify`

Vérifie un code QR scanné. Utilisé par le scanner d'embarquement.

### Authentification
Aucune (appelé par le scanner, souvent hors-ligne → l'auth se fait côté client).

### Corps de la requête

```json
{
  "code": "42-7-a3f8c91b"
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `code` | `string` | Code QR scanné au format `bookingId-passengerId-signature` |

### Réponse `200`

```json
{
  "bookingId": "42",
  "passengerId": "7",
  "isValid": true
}
```

| Champ | Description |
|-------|-------------|
| `bookingId` | ID de la réservation extrait du code |
| `passengerId` | ID du passager extrait du code |
| `isValid` | `true` si la signature est correcte, `false` sinon |

### Cas d'erreur

| Situation | `isValid` | Code HTTP |
|-----------|-----------|-----------|
| Code manquant ou non-string | — | `400` |
| Format invalide (pas 3 parties) | — | `400` |
| Signature incorrecte | `false` | `200` |
| Secret HMAC manquant serveur | `false` | `500` |

> 💡 **Anti-replay** : après un scan valide, le client doit immédiatement marquer le billet comme `checked_in: true` en **LocalStorage** avant toute synchro réseau.

---

## `POST /api/whatsapp/bot/sale`

Crée une réservation et génère les QR codes depuis le bot WhatsApp agent.

### Authentification
```
x-bot-secret: <WHATSAPP_BOT_SECRET>
```
Secret interne — jamais exposé côté client.

### Corps de la requête

```json
{
  "tripId": 10,
  "companyId": "uuid-compagnie",
  "passengers": [
    { "firstName": "Marie", "lastName": "Nzé", "phone": "074000000" }
  ],
  "payerPhone": "074000000",
  "travelClass": "standard"
}
```

### Réponse `200`

```json
{
  "success": true,
  "bookingId": 42,
  "tickets": [
    {
      "passengerId": 7,
      "firstName": "Marie",
      "lastName": "Nzé",
      "code": "42-7-a3f8c91b",
      "qrBase64": "data:image/png;base64,..."
    }
  ]
}
```

### Pipeline

1. Vérifie disponibilité des places
2. Crée le booking (`status: "paid"`, `source: "whatsapp"`)
3. Insère les passagers
4. Décrémente les places (`decrement_seats`)
5. Génère les signatures HMAC pour chaque passager
6. Encode les QR codes en base64 (PNG)
7. Retourne les billets pour envoi via WhatsApp
