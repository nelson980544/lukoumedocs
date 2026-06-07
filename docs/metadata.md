# 🔍 API — Metadata & Découverte

Routes de découverte utilisées par les agents IA, les outils OAuth et les crawlers de contenu.

---

## `GET /api/markdown/home`

Expose une description **Markdown** du service TripGabon (présentation, destinations, tarifs, FAQ).

Utilisé par les agents IA (ex: ChatGPT, Claude) pour découvrir et comprendre l'offre de la plateforme.

### Authentification
Aucune (contenu public).

### En-têtes de réponse

```
Content-Type: text/markdown
Cache-Control: public, max-age=3600
```

### Réponse `200` (exemple)

```markdown
# TripGabon — Réservation de billets de bateau au Gabon

TripGabon est la plateforme de référence pour la réservation de billets
de transport maritime au Gabon.

## Destinations desservies
- Libreville ↔ Port-Gentil (3h30)
- Libreville ↔ Omboué (5h)
- Port-Gentil ↔ Mayumba (4h)

## Tarifs
- Classe Standard : à partir de 12 000 FCFA
- Classe Business : à partir de 25 000 FCFA

## Modes de paiement
- 📱 Airtel Money (07x)
- 📱 Moov Money (06x)
- 💳 Carte bancaire (Visa/Mastercard)

## FAQ
...
```

### Cache

Le contenu est mis en cache **1 heure** (`max-age=3600`) pour réduire la charge sur les réseaux instables.

---

## `GET /api/oidc-discovery`

Expose les **métadonnées OIDC** (RFC 8414 — OAuth 2.0 Authorization Server Metadata) pointant vers Supabase Auth.

Permet aux agents IA et aux clients OAuth de découvrir automatiquement les endpoints d'authentification.

### Authentification
Aucune (endpoint de découverte public).

### Réponse `200`

```json
{
  "issuer": "https://xyz.supabase.co/auth/v1",
  "authorization_endpoint": "https://xyz.supabase.co/auth/v1/authorize",
  "token_endpoint": "https://xyz.supabase.co/auth/v1/token",
  "userinfo_endpoint": "https://xyz.supabase.co/auth/v1/user",
  "jwks_uri": "https://xyz.supabase.co/auth/v1/.well-known/jwks.json",
  "response_types_supported": ["code", "token"],
  "grant_types_supported": ["authorization_code", "refresh_token", "password"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"]
}
```

### Standard implémenté

[RFC 8414 — OAuth 2.0 Authorization Server Metadata](https://datatracker.ietf.org/doc/html/rfc8414)

---

## `GET /api/oauth-protected-resource`

Expose les **métadonnées de ressource protégée OAuth 2.0** (RFC 9728).

Indique aux clients OAuth quel serveur d'autorisation protège cette API et quels scopes sont supportés.

### Authentification
Aucune (endpoint de découverte public).

### Réponse `200`

```json
{
  "resource": "https://tripgabonadmin.vercel.app",
  "authorization_servers": [
    "https://xyz.supabase.co/auth/v1"
  ],
  "scopes_supported": [
    "openid",
    "profile",
    "email"
  ],
  "bearer_methods_supported": ["header"],
  "resource_documentation": "https://tripgabonadmin.vercel.app/api/markdown/home"
}
```

### Standard implémenté

[RFC 9728 — OAuth 2.0 Protected Resource Metadata](https://datatracker.ietf.org/doc/html/rfc9728)

### Cas d'usage

Ces deux endpoints de découverte permettent à des **agents IA** (ex: GPT Actions, Claude tools) de :
1. Découvrir automatiquement l'URL d'authentification
2. Savoir comment obtenir un token d'accès
3. Comprendre les scopes disponibles
4. Accéder aux ressources protégées de l'API TripGabon
