# 📦 API — Réservations & Voyages

---

## `GET /api/bookings/status`

Récupère le statut d'une réservation à partir d'un ID encodé (utilisé sur la page de confirmation de paiement).

### Authentification
Aucune. L'ID est encodé (obfuscation) et la réponse ne divulgue pas de PII.

### Paramètre de requête

| Paramètre | Type | Description |
|-----------|------|-------------|
| `id` | `string` | ID de réservation **encodé** (via `booking-encoder.ts`) |

### Exemple

```
GET /api/bookings/status?id=Xk9mN3p
```

### Réponse `200`

```json
{
  "status": "paid",
  "id": "42"
}
```

### Statuts possibles

| Statut | Description |
|--------|-------------|
| `pending` | En attente de paiement |
| `paid` | Paiement confirmé |
| `cancelled` | Annulé |
| `expired` | Délai de paiement dépassé |

### Erreurs

| Code | Raison |
|------|--------|
| `400` | Paramètre `id` manquant ou ID décodé invalide |
| `404` | Réservation introuvable (réponse générique — anti-énumération) |

---

## `PATCH /api/trips/[id]/stops`

Met à jour les **arrêts intermédiaires** et les **tarifs par segment** d'un voyage.

### Authentification
```
Authorization: Bearer <jwt_supabase>
```
Rôle requis : `company_admin` ou `super_admin`

### Paramètre de route

| Paramètre | Type | Description |
|-----------|------|-------------|
| `id` | `number` | ID du voyage |

### Corps de la requête

```json
{
  "stops": [
    { "name": "Port-Gentil", "order": 1, "duration_minutes": 120 },
    { "name": "Omboué", "order": 2, "duration_minutes": 60 }
  ],
  "segmentPrices": [
    { "from": "Libreville", "to": "Port-Gentil", "price": 15000 },
    { "from": "Libreville", "to": "Omboué", "price": 20000 }
  ]
}
```

### Réponse `200`

```json
{ "success": true }
```

### Comportement

- Supprime **tous** les anciens arrêts et tarifs segment
- Insère les nouveaux
- Opération non transactionnelle — utilise des requêtes séquentielles delete/insert

---

## `POST /api/driver-rating`

Enregistre une note pour un conducteur (1 à 5 étoiles + critères secondaires).

### Authentification
```
Authorization: Bearer <jwt_supabase>
```

### Corps de la requête

```json
{
  "bookingId": 42,
  "driverId": "uuid-driver",
  "rating": 5,
  "criteria": {
    "punctuality": 5,
    "safety": 4,
    "comfort": 5
  },
  "comment": "Excellent conducteur, très ponctuel."
}
```

### Réponse `200`

```json
{ "success": true }
```

### Contrainte anti-doublon
Un utilisateur ne peut noter qu'**une seule fois** par couple `(bookingId, driverId)`.

---

## `GET /api/driver-rating`

Récupère les notes d'un conducteur (données anonymisées, accès public via RLS Supabase).

### Authentification
Aucune (RLS Supabase — lecture publique des notes anonymisées).

### Paramètre de requête

| Paramètre | Description |
|-----------|-------------|
| `driverId` | UUID du conducteur |

### Réponse `200`

```json
{
  "ratings": [
    { "rating": 5, "criteria": { "punctuality": 5 }, "created_at": "2026-01-10T08:00:00Z" }
  ],
  "average": 4.8,
  "count": 24
}
```

---

## `POST /api/wallet`

Génère un **JWT signé Google Wallet** pour ajouter les billets dans l'application Google Pay.

### Authentification
```
Authorization: Bearer <jwt_supabase>
```
L'appelant doit être le propriétaire du booking **ou** un agent/admin de la compagnie.

### Corps de la requête

```json
{
  "bookingId": 42,
  "passengers": [
    { "id": 7, "first_name": "Marie", "last_name": "Nzé" }
  ],
  "trip": {
    "departure_location": "Libreville",
    "arrival_location": "Port-Gentil",
    "departure_time": "2026-06-15T08:00:00Z",
    "company_name": "Lukoumé",
    "status_wallet": "ACTIVE"
  },
  "booking": {
    "details": { "class": "business" }
  }
}
```

### Réponse `200`

```json
{
  "url": "https://pay.google.com/gp/v/save/eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

Rediriger le client vers `url` pour ajouter ses billets dans Google Wallet.

### Fonctionnalités du billet Google Wallet

| Champ | Valeur |
|-------|--------|
| Type | `TransitObject` (bateau) |
| Code QR | `bookingId-passengerId` |
| Classe | `ECONOMY` ou `BUSINESS` |
| Statut retard | Message dynamique si `status_wallet: "DELAYED"` |
| Géofencing | Activé si configuré dans `wallet_settings` |
| Messages | Politique bagages + heure d'arrivée conseillée |

### Variables d'environnement requises

| Variable | Description |
|----------|-------------|
| `GOOGLE_WALLET_ISSUER_ID` | Issuer ID Google Wallet |
| `GOOGLE_WALLET_PRIVATE_KEY` | Clé privée RSA (PEM) |
| `GOOGLE_WALLET_SERVICE_ACCOUNT_EMAIL` | Email du compte de service Google |
