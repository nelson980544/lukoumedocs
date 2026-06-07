# 🎁 API — Fidélité & Notifications Push

---

## `POST /api/loyalty/check`

Déclenche l'attribution des **récompenses de fidélité** pour un utilisateur (filet de sécurité côté client — le principal déclenchement se fait dans les callbacks de paiement).

### Authentification
```
Authorization: Bearer <jwt_supabase>
```

### Corps de la requête

```json
{
  "userId": "uuid-utilisateur"
}
```

### Comportement

Appelle `issueLoyaltyRewards(userId)` qui :
1. Compte le nombre total de voyages de l'utilisateur
2. Vérifie les paliers de fidélité (ex: 5 voyages = coupon -10%)
3. Émet les coupons non encore attribués
4. Opération **idempotente** — ne crée pas de doublon

### Réponse `200`

```json
{
  "success": true,
  "rewarded": true,
  "couponsIssued": ["LOYAL10-ABC123"]
}
```

| Champ | Description |
|-------|-------------|
| `rewarded` | `true` si un ou plusieurs coupons ont été émis |
| `couponsIssued` | Liste des codes coupons créés |

### Paliers de fidélité (exemple)

| Voyages effectués | Récompense |
|------------------|------------|
| 5 | Coupon -10% sur la prochaine réservation |
| 10 | Coupon -15% |
| 20 | Coupon -20% + accès classe Business offerte |

### Erreurs

| Code | Raison |
|------|--------|
| `401` | Token absent ou invalide |
| `500` | Erreur Supabase lors de l'émission des coupons |

---

## `POST /api/push/subscribe`

Enregistre ou met à jour un abonnement **Web Push** (PWA) pour un utilisateur authentifié.

### Authentification
```
Authorization: Bearer <jwt_supabase>
```

### Corps de la requête

```json
{
  "subscription": {
    "endpoint": "https://fcm.googleapis.com/fcm/send/...",
    "keys": {
      "p256dh": "BNcRdreALRFXTkOOUHK...",
      "auth": "tBHItJI5svbpez7KI4CCXg"
    }
  }
}
```

Ce payload est l'objet `PushSubscription` retourné par `navigator.serviceWorker.ready.then(sw => sw.pushManager.subscribe(...))`.

### Comportement

- **Upsert** par `endpoint` dans la table `push_subscriptions`
- Un même appareil ne génère qu'une seule entrée (pas de doublons)
- Associé à l'`user_id` de l'utilisateur authentifié

### Réponse `200`

```json
{ "success": true }
```

### Utilisation côté client

```javascript
// 1. Demander la permission
const permission = await Notification.requestPermission();
if (permission !== 'granted') return;

// 2. Obtenir le service worker
const sw = await navigator.serviceWorker.ready;

// 3. S'abonner au push
const subscription = await sw.pushManager.subscribe({
  userVisibleOnly: true,
  applicationServerKey: '<VAPID_PUBLIC_KEY>'
});

// 4. Enregistrer côté serveur
await fetch('/api/push/subscribe', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({ subscription })
});
```

### Notifications push envoyées par la plateforme

| Événement | Message |
|-----------|---------|
| Paiement confirmé | "Votre billet est prêt ! Bon voyage 🚢" |
| Rappel de départ (J) | "Votre voyage part dans 2h30" |
| Coupon fidélité émis | "Vous avez gagné un bon de réduction !" |
| Modification voyage | "Votre voyage a été modifié — vérifiez les détails" |
