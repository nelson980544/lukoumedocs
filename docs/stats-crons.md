# 📊 API — Statistiques & Crons

---

## `GET /api/stats/recap`

Calcule et envoie un **récapitulatif statistique** (quotidien / mensuel / annuel) via Telegram. Déclenché par le cron Vercel ou manuellement par un super admin.

### Authentification

| Méthode | Header | Usage |
|---------|--------|-------|
| Cron Vercel | `Authorization: Bearer <CRON_SECRET>` | Appel automatique |
| Manuel | `Authorization: Bearer <jwt_supabase>` (rôle `super_admin`) | `POST /api/stats/recap` |

### `GET` — Appel cron Vercel

```
GET /api/stats/recap
Authorization: Bearer <CRON_SECRET>
```

### `POST` — Appel manuel super_admin

```
POST /api/stats/recap
Authorization: Bearer <jwt_supabase>
```

### Métriques calculées

| Métrique | Description |
|----------|-------------|
| Revenu total | Somme des `total_price` des bookings `paid` |
| Nombre de billets | Compte des `number_of_seats` |
| Taux de remplissage | Places vendues / capacité totale |
| Vues voyages | Compteur de vues des pages voyage |
| Taux no-show | Passagers non présentés |
| Comparaison J-1 | Évolution vs la veille (%) |

### Exemple de message Telegram envoyé

```
📊 Récap quotidien — 07 juin 2026

🎟️ Billets vendus : 47 (+12% vs hier)
💰 Revenu : 705 000 FCFA (+8%)
🚢 Taux de remplissage : 82%
👻 No-show : 3 passagers

📅 Ce mois : 1 240 billets — 18 600 000 FCFA
📆 Cette année : 8 430 billets — 126 450 000 FCFA
```

### Réponse `200`

```json
{
  "success": true,
  "stats": {
    "daily": { "revenue": 705000, "tickets": 47, "occupancy": 0.82 },
    "monthly": { "revenue": 18600000, "tickets": 1240 },
    "annual": { "revenue": 126450000, "tickets": 8430 }
  }
}
```

---

## `GET /api/cron/departure-reminders`

Envoie des **rappels de départ** aux passagers dont le voyage part dans les 2 à 3 prochaines heures.

### Authentification
```
Authorization: Bearer <CRON_SECRET>
```

### Planification Vercel

```json
{
  "path": "/api/cron/departure-reminders",
  "schedule": "0 6 * * *"
}
```

> ⚠️ Plan Vercel Hobby — crons limités à **une fois par jour**. Le cron tourne à 6h UTC (7h heure de Libreville).

### Comportement

1. Sélectionne les voyages avec `departure_time` dans `[now+2h, now+3h]`
2. Récupère les passagers de chaque voyage
3. Envoie pour chaque passager :
   - 🔔 Notification **push PWA** (`/utils/push-service.ts`)
   - 📩 Notification **in-app** (table `notifications`)

### Message de rappel

```
🚢 Votre voyage Libreville → Port-Gentil part dans 2h30 !
Présentez-vous à l'embarcadère 30 minutes avant le départ.
```

### Réponse `200`

```json
{
  "success": true,
  "notified": 23
}
```

| Champ | Description |
|-------|-------------|
| `notified` | Nombre de passagers notifiés |
