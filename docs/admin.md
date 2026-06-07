# 👤 API — Gestion des Agents & Administrateurs

Ces routes permettent à un `company_admin` ou `super_admin` de créer, consulter et modifier les comptes agents et conducteurs au sein de leur compagnie.

---

## `POST /api/admin/create-agent`

Crée un nouvel agent ou conducteur associé à une compagnie.

### Authentification
```
Authorization: Bearer <jwt_supabase>
```
Rôle requis : `company_admin` ou `super_admin`

### Corps de la requête

```json
{
  "email": "jean.dupont",
  "password": "motdepasse123",
  "fullName": "Jean Dupont",
  "companyId": "uuid-de-la-compagnie",
  "role": "agent"
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `email` | `string` | ✅ | Email ou identifiant simple. Si sans `@`, converti automatiquement en `prenom.nom@tripgabon.local` |
| `password` | `string` | ✅ | Mot de passe |
| `fullName` | `string` | ❌ | Nom complet affiché |
| `companyId` | `string` (UUID) | ✅ | UUID de la compagnie |
| `role` | `"agent"` \| `"driver"` | ❌ | Rôle attribué — défaut : `"agent"` |

### Réponse `200`

```json
{
  "success": true,
  "user": {
    "id": "uuid",
    "email": "jean.dupont@tripgabon.local",
    "created_at": "2026-01-15T10:00:00Z"
  }
}
```

### Erreurs

| Code | Raison |
|------|--------|
| `400` | Champs manquants ou email déjà utilisé dans Supabase Auth |
| `401` | Token JWT absent ou expiré |
| `403` | Rôle insuffisant · tentative d'action sur une autre compagnie |
| `500` | Erreur Supabase (rollback automatique si le profil échoue après création Auth) |

### 🔒 Sécurité

- Le `currentUserId` éventuellement envoyé dans le body est **ignoré** — l'identité de l'appelant est toujours vérifiée depuis le token JWT.
- Un `company_admin` ne peut créer des agents que dans **sa propre compagnie**.
- **Rollback** : si l'insertion du profil `user_profiles` échoue, l'utilisateur Auth est supprimé.

---

## `GET /api/admin/agents/[id]`

Récupère les détails d'un agent (données Auth + profil Supabase).

### Authentification
```
Authorization: Bearer <jwt_supabase>
```
Rôle requis : `company_admin` ou `super_admin`

### Paramètre de route

| Paramètre | Type | Description |
|-----------|------|-------------|
| `id` | `string` (UUID) | ID Supabase Auth de l'agent |

### Réponse `200`

```json
{
  "user": {
    "id": "uuid",
    "email": "jean.dupont@tripgabon.local",
    "last_sign_in_at": "2026-01-15T09:00:00Z",
    "created_at": "2026-01-10T08:00:00Z"
  },
  "profile": {
    "id": "uuid",
    "full_name": "Jean Dupont",
    "role": "agent",
    "company_id": "uuid-compagnie",
    "is_active": true
  }
}
```

### Erreurs

| Code | Raison |
|------|--------|
| `401` | Non authentifié |
| `403` | `company_admin` tentant d'accéder à un agent d'une autre compagnie |
| `404` | Agent introuvable dans Auth ou dans `user_profiles` |

---

## `PATCH /api/admin/agents/[id]`

Met à jour un agent existant (nom, email, rôle, statut actif, mot de passe).

### Authentification
```
Authorization: Bearer <jwt_supabase>
```
Rôle requis : `company_admin` ou `super_admin`

### Paramètre de route

| Paramètre | Type | Description |
|-----------|------|-------------|
| `id` | `string` (UUID) | ID de l'agent à modifier |

### Corps de la requête (tous les champs sont optionnels)

```json
{
  "fullName": "Jean-Pierre Dupont",
  "email": "jp.dupont",
  "role": "driver",
  "isActive": false,
  "password": "nouveauMotDePasse"
}
```

| Champ | Type | Description |
|-------|------|-------------|
| `fullName` | `string` | Nouveau nom complet |
| `email` | `string` | Nouvel email ou identifiant (converti en `@tripgabon.local` si pas de `@`) |
| `role` | `string` | Nouveau rôle (`"agent"`, `"driver"`, `"company_admin"`) |
| `isActive` | `boolean` | Activer / désactiver le compte |
| `password` | `string` | Nouveau mot de passe (min 6 caractères) |

### Réponse `200`

```json
{
  "success": true,
  "message": "Agent mis à jour"
}
```

### Erreurs

| Code | Raison |
|------|--------|
| `400` | Erreur de mise à jour Supabase Auth |
| `401` | Non authentifié |
| `403` | Agent d'une autre compagnie · tentative d'escalade vers `super_admin` |
| `404` | Agent introuvable |

### 🔒 Sécurité

- Seul un `super_admin` peut attribuer le rôle `super_admin` (**anti-escalade de privilège**).
- Les changements d'email et de mot de passe sont appliqués simultanément dans Auth ET dans `user_profiles`.
