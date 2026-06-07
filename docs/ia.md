# 🤖 API — Intelligence Artificielle

Deux endpoints IA basés sur **NVIDIA API** (modèle `meta/llama-3.1-70b-instruct`), contextualisés pour le marché du transport gabonais.

---

## `POST /api/ia/insights`

Génère **4 insights stratégiques** pour une compagnie de transport, en JSON structuré.

### Authentification
```
Authorization: Bearer <jwt_supabase>
```

### Corps de la requête

```json
{
  "companyContext": "Compagnie Lukoumé — 3 voyages par semaine Libreville↔Port-Gentil. Taux de remplissage moyen : 78%. Revenu mensuel : 4 500 000 FCFA. 12 réservations annulées ce mois. Période haute : juillet-août."
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `companyContext` | `string` | Contexte métier de la compagnie (stats, tendances, données opérationnelles) |

### Réponse `200`

```json
{
  "insights": [
    {
      "id": "1",
      "type": "alerte",
      "title": "Taux d'annulation en hausse",
      "description": "12 réservations annulées ce mois, soit +40% vs mois précédent.",
      "action": "Mettre en place une politique de non-remboursement à J-48h."
    },
    {
      "id": "2",
      "type": "opportunite",
      "title": "Période haute juillet-août",
      "description": "Historiquement +35% de demande en saison estivale.",
      "action": "Ajouter un voyage supplémentaire le vendredi."
    },
    {
      "id": "3",
      "type": "performance",
      "title": "Taux de remplissage optimal",
      "description": "78% de remplissage moyen, proche du seuil de rentabilité maximale.",
      "action": "Tester une légère hausse tarifaire (+5%) pour les réservations J-3."
    },
    {
      "id": "4",
      "type": "operation",
      "title": "Optimisation des départs",
      "description": "Les départs du lundi ont 23% de remplissage inférieur.",
      "action": "Décaler le départ du lundi au mardi pour regrouper la demande."
    }
  ]
}
```

### Types d'insights

| Type | Description |
|------|-------------|
| `alerte` | Problème nécessitant une action rapide |
| `opportunite` | Occasion de croissance identifiée |
| `performance` | Analyse de la performance actuelle |
| `operation` | Recommandation opérationnelle |

### Paramètres NVIDIA

| Paramètre | Valeur |
|-----------|--------|
| Modèle | `meta/llama-3.1-70b-instruct` |
| `stream` | `false` |
| `max_tokens` | `1024` |
| `temperature` | `0.5` |

### Erreurs

| Code | Raison |
|------|--------|
| `401` | Token Supabase absent ou invalide |
| `500` | `NVIDIA_API_KEY` manquante |
| `502` | API NVIDIA indisponible ou réponse JSON invalide |

---

## `POST /api/ia/chat`

Chat conversationnel IA **en streaming** (SSE — Server-Sent Events), contextualisé pour le marché gabonais.

### Authentification
```
Authorization: Bearer <jwt_supabase>
```

### Corps de la requête

```json
{
  "messages": [
    { "role": "user", "content": "Comment améliorer mon taux de remplissage ?" },
    { "role": "assistant", "content": "Voici quelques pistes..." },
    { "role": "user", "content": "Et pour la période des fêtes ?" }
  ],
  "companyContext": "Compagnie Lukoumé, route Libreville-Port-Gentil."
}
```

| Champ | Type | Requis | Limites |
|-------|------|--------|---------|
| `messages` | `array` | ✅ | 1 à 20 messages |
| `messages[].role` | `"user"` \| `"assistant"` | ✅ | — |
| `messages[].content` | `string` | ✅ | Max 4000 caractères par message |
| `companyContext` | `string` | ❌ | Max 2000 caractères |

### Réponse `200` — Stream SSE

```
Content-Type: text/event-stream

data: {"choices":[{"delta":{"content":"Voici"}}]}
data: {"choices":[{"delta":{"content":" mes"}}]}
data: {"choices":[{"delta":{"content":" recommandations..."}}]}
data: [DONE]
```

Consommer côté client avec `EventSource` ou `fetch` + lecture du stream.

### Exemple côté client (JavaScript)

```javascript
const response = await fetch('/api/ia/chat', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({ messages, companyContext })
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  const chunk = decoder.decode(value);
  // Traiter les lignes SSE...
}
```

### Paramètres NVIDIA

| Paramètre | Valeur |
|-----------|--------|
| Modèle | `meta/llama-3.1-70b-instruct` |
| `stream` | `true` |
| `max_tokens` | `1024` |
| `temperature` | `0.7` |

### Prompt système injecté

Si `companyContext` est fourni :
```
<companyContext>
Réponds en français, sois concis et donne des conseils actionnables adaptés au marché gabonais.
```

Sinon :
```
Tu es un conseiller IA pour une compagnie de transport au Gabon. Réponds en français.
```

### Erreurs

| Code | Raison |
|------|--------|
| `400` | Payload invalide (Zod) — messages manquants, trop longs, trop nombreux |
| `401` | Token Supabase absent ou invalide |
| `500` | `NVIDIA_API_KEY` manquante |
| `502` | API NVIDIA indisponible |
