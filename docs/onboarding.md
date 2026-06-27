# 🏢 API — Onboarding & Signature Électronique

Ces routes gèrent l'inscription en libre-service des compagnies, la gestion des documents KYB, les emails automatiques et la signature électronique du contrat de partenariat via OTP.

---

## Flux général

```
/onboarding (6 étapes) → RPC submit_company_onboarding
                        → [Email: invitation avec lien signup]
                        → [Notification super admin]
       → /admin/signup?token=xxx
       → Signature du contrat (OTP)
       → /admin/setup (4 étapes : lieux → équipe → KYB → attente)
       → [Email B: documents reçus]
       → Super admin valide (1 clic)
       → [Email C: compte activé]
       → Dashboard actif

Super admin rejette → [Email D: rejet avec raison]
```

> **Note :** La page d'auto-inscription publique est `/onboarding` (`app/(backoffice)/onboarding/`), un wizard 6 étapes qui appelle la RPC Supabase `submit_company_onboarding` et les routes `/api/onboarding/notify` et `/api/onboarding/invite`. La route `/api/companies/apply` est un endpoint alternatif REST disponible mais non utilisé par le flux principal.

---

## `POST /api/companies/apply`

Endpoint alternatif d'auto-inscription. Crée une compagnie, génère le token d'invitation et déclenche les emails. Le flux principal passe par `/onboarding` + RPC `submit_company_onboarding`.

### Authentification
Aucune — route publique.

### Corps de la requête

```json
{
  "companyName": "Transport Libreville SARL",
  "companyType": "bus",
  "contactEmail": "contact@transport-lbv.ga",
  "contactPhone": "074123456",
  "city": "Libreville",
  "legalName": "Transport Libreville SARL"
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `companyName` | `string` | ✅ | Nom commercial de la compagnie |
| `companyType` | `"bus"` \| `"boat"` | ✅ | Type de transport |
| `contactEmail` | `string` | ✅ | Email du responsable (destinataire de l'invitation) |
| `contactPhone` | `string` | ❌ | Numéro de téléphone |
| `city` | `string` | ❌ | Ville principale d'opération |
| `legalName` | `string` | ❌ | Raison sociale légale |

### Réponse `200`

```json
{
  "success": true,
  "companyId": "uuid-compagnie"
}
```

### Actions déclenchées

1. Insertion dans `companies` avec `status: 'pending'`, `kyb_status: 'pending'`, tous modules désactivés
2. Génération d'un token UUID dans `company_invitations` (expiration : 7 jours)
3. Appel interne → `POST /api/onboarding/invite` (email avec lien de création de compte)
4. Appel interne → `POST /api/onboarding/notify` (notification du super admin)
5. Email de confirmation "Demande reçue" envoyé au contact (Template A)

### Erreurs

| Code | Raison |
|------|--------|
| `400` | `companyName`, `companyType` ou `contactEmail` manquant |
| `500` | Erreur Supabase (insertion) ou Resend (email) |

---

## `POST /api/onboarding/invite`

Envoie l'email d'invitation avec le lien de création de compte. Appelé automatiquement par `/api/companies/apply` mais peut être rappelé manuellement.

### Authentification
Aucune (route interne — appelée server-side).

### Corps de la requête

```json
{
  "companyName": "Transport Libreville SARL",
  "contactEmail": "contact@transport-lbv.ga",
  "invitationToken": "uuid-token"
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `companyName` | `string` | ❌ | Nom de la compagnie |
| `contactEmail` | `string` | ✅ | Destinataire de l'email |
| `invitationToken` | `string` | ✅ | Token UUID issu de `company_invitations` |

### Réponse `200`

```json
{ "success": true }
```

### 🔒 Sécurité

- L'URL de base est dérivée depuis `NEXT_PUBLIC_APP_URL` côté serveur — **jamais depuis le body** (anti-phishing).
- Le lien généré : `{APP_URL}/admin/signup?token={invitationToken}`

---

## `POST /api/onboarding/notify`

Notifie le super admin par email lorsqu'une nouvelle candidature est soumise.

### Authentification
Aucune (route interne).

### Corps de la requête

```json
{
  "companyName": "Transport Libreville SARL",
  "companyType": "bus",
  "contactEmail": "contact@transport-lbv.ga",
  "contactPhone": "074123456",
  "legalName": "Transport Libreville SARL"
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `companyName` | `string` | ✅ | Nom de la compagnie (max 200 chars) |
| `companyType` | `"bus"` \| `"boat"` | ✅ | Type de transport |
| `contactEmail` | `string` | ✅ | Email de contact (validé par Zod) |
| `contactPhone` | `string` | ❌ | Téléphone (max 30 chars) |
| `legalName` | `string` | ❌ | Raison sociale (max 200 chars) |

### 🔒 Sécurité

- Validation Zod stricte + échappement HTML de tous les champs (protection XSS).
- Destinataire fixe en dur côté serveur : `nossimaemane@gmail.com`.

---

## `POST /api/onboarding/documents`

Upload les documents KYB d'une compagnie vers Supabase Storage. Déclenche l'email "Documents reçus" après soumission complète.

### Authentification
```
Authorization: Bearer <jwt_supabase>
```
Rôle requis : `company_admin` de la compagnie concernée.

### Corps de la requête

`multipart/form-data`

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `company_id` | `string` (UUID) | ✅ | UUID de la compagnie |
| `type` | `"nif"` \| `"patente"` \| `"insurance"` | ✅ | Type de document |
| `file` | `File` | ✅ | PDF, JPG, PNG ou WEBP |

### Réponse `200`

```json
{
  "success": true,
  "fileUrl": "https://supabase-url/storage/v1/object/company-documents/uuid/nif_1234567890.pdf",
  "documentsComplete": false
}
```

| Champ | Description |
|-------|-------------|
| `fileUrl` | URL publique du fichier uploadé |
| `documentsComplete` | `true` si les 3 documents (NIF, Patente, Assurance) sont uploadés |

### Pipeline

1. Upload vers bucket `company-documents` — chemin : `{company_id}/{type}_{timestamp}.{ext}`
2. Upsert dans `company_documents` (contrainte `UNIQUE(company_id, type)`)
3. Si les 3 documents sont présents : mise à jour `companies.status → 'pending_approval'`
4. Si les 3 documents sont présents : envoi email Template B "Documents reçus"

### Erreurs

| Code | Raison |
|------|--------|
| `400` | `company_id`, `type` ou fichier manquant · type non reconnu |
| `401` | Non authentifié |
| `403` | La compagnie ne correspond pas au compte connecté |
| `500` | Erreur upload Supabase Storage |

---

## `POST /api/onboarding/otp/send`

Génère un OTP à 6 chiffres et l'envoie par email. Utilisé lors de la signature électronique du contrat de partenariat.

### Authentification
Aucune — appelé depuis la page `/admin/signup` après création du compte mais avant connexion complète.

### Corps de la requête

```json
{
  "email": "contact@transport-lbv.ga",
  "companyId": "uuid-compagnie"
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `email` | `string` | ✅ | Email du signataire |
| `companyId` | `string` (UUID) | ✅ | UUID de la compagnie |

### Réponse `200`

```json
{ "success": true }
```

### Pipeline

1. Invalide tous les OTPs existants non utilisés pour cet email (`used_at = now()`)
2. Génère un code à 6 chiffres : `Math.floor(100000 + Math.random() * 900000)`
3. Insère dans `onboarding_otps` avec `expires_at = now() + 10 minutes`
4. Envoie l'email avec le code en grand format monospace via Resend

### Erreurs

| Code | Raison |
|------|--------|
| `400` | `email` ou `companyId` manquant |
| `500` | Erreur Supabase (insertion) ou Resend (email) |

> ⏱ Le code expire après **10 minutes** et ne peut être utilisé qu'**une seule fois**.

---

## `POST /api/onboarding/otp/verify`

Vérifie le code OTP, enregistre la signature électronique et envoie le récapitulatif de contrat.

### Authentification
Aucune — appelé depuis la page `/admin/signup`.

### Corps de la requête

```json
{
  "email": "contact@transport-lbv.ga",
  "otp": "847291",
  "companyId": "uuid-compagnie",
  "userId": "uuid-auth-user",
  "signatoryName": "Jean-Baptiste Nzé",
  "contractSnapshot": {
    "company_type": "bus",
    "ticket_commission_percent": 3,
    "options": [
      { "name": "Bagages supplémentaires", "base_price": 2000, "platform_commission": 200 }
    ]
  }
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `email` | `string` | ✅ | Email du signataire |
| `otp` | `string` | ✅ | Code OTP saisi |
| `companyId` | `string` (UUID) | ✅ | UUID de la compagnie |
| `userId` | `string` (UUID) | ❌ | UUID Supabase Auth du signataire |
| `signatoryName` | `string` | ✅ | Nom complet du signataire |
| `contractSnapshot` | `object` | ❌ | Snapshot JSONB du contrat signé |

### Réponse `200`

```json
{
  "success": true,
  "signatureHash": "a3f8c91b2d4e...",
  "signedAt": "2026-06-27T14:32:00.000Z"
}
```

### Pipeline

1. Vérification OTP : cherche dans `onboarding_otps` (email + companyId + code + non utilisé + non expiré)
2. Marque l'OTP comme utilisé (`used_at = now()`)
3. Génère l'empreinte SHA-256 : `hash(companyId:email:signedAt:TripGabon-v1.0-2026)`
4. Récupère l'IP du signataire (`x-forwarded-for` ou `x-real-ip`)
5. Insère dans `company_contracts` (snapshot complet, hash, IP, date)
6. Merge dans `companies.onboarding_data` : `{ contract_signed: true, signed_at, signature_hash }`
7. Envoie l'email de confirmation avec le récapitulatif du contrat et le hash SHA-256

### Erreurs

| Code | Raison |
|------|--------|
| `400` | Champs manquants · Code invalide · Code expiré · Code déjà utilisé |
| `500` | Erreur d'insertion dans `company_contracts` |

### 🔐 Valeur légale

| Élément | Valeur |
|---------|--------|
| Algorithme | SHA-256 |
| Sel | `TripGabon-v1.0-2026` |
| Données signées | `companyId + email + horodatage` |
| Stockage | Table `company_contracts` (immuable) |
| Preuve | Email de confirmation avec hash envoyé au signataire |

---

## `POST /api/onboarding/activation`

Envoie l'email d'activation (Template C) quand le super admin valide une compagnie.

### Authentification
Route interne — appelée par le dashboard super admin.

### Corps de la requête

```json
{
  "companyName": "Transport Libreville SARL",
  "contactEmail": "contact@transport-lbv.ga"
}
```

### Réponse `200`

```json
{ "success": true }
```

---

## `POST /api/onboarding/rejection`

Envoie l'email de rejet (Template D) avec la raison et un lien pour re-soumettre.

### Authentification
Route interne — appelée par le dashboard super admin.

### Corps de la requête

```json
{
  "companyName": "Transport Libreville SARL",
  "contactEmail": "contact@transport-lbv.ga",
  "rejectionReason": "Documents KYB illisibles. Merci de re-soumettre en haute résolution."
}
```

| Champ | Type | Requis | Description |
|-------|------|--------|-------------|
| `companyName` | `string` | ✅ | Nom de la compagnie |
| `contactEmail` | `string` | ✅ | Destinataire |
| `rejectionReason` | `string` | ❌ | Raison du rejet affichée dans l'email |

### Réponse `200`

```json
{ "success": true }
```

---

## Tables Supabase

### `onboarding_otps`

| Colonne | Type | Description |
|---------|------|-------------|
| `id` | `UUID` | Clé primaire |
| `email` | `TEXT` | Email du destinataire |
| `company_id` | `UUID` | FK → `companies.id` |
| `otp_code` | `TEXT` | Code à 6 chiffres |
| `expires_at` | `TIMESTAMPTZ` | Expiration (10 min après création) |
| `used_at` | `TIMESTAMPTZ` | Horodatage d'utilisation (`NULL` = non utilisé) |
| `created_at` | `TIMESTAMPTZ` | Date de création |

### `company_contracts`

| Colonne | Type | Description |
|---------|------|-------------|
| `id` | `UUID` | Clé primaire |
| `company_id` | `UUID` | FK → `companies.id` |
| `user_id` | `UUID` | FK → `auth.users.id` (signataire) |
| `contract_version` | `TEXT` | Ex : `v1.0-2026` |
| `signed_at` | `TIMESTAMPTZ` | Horodatage de signature |
| `signatory_name` | `TEXT` | Nom complet du signataire |
| `signatory_email` | `TEXT` | Email du signataire |
| `ip_address` | `TEXT` | Adresse IP au moment de la signature |
| `signature_hash` | `TEXT UNIQUE` | Empreinte SHA-256 |
| `contract_snapshot` | `JSONB` | Conditions acceptées (commissions, options) |
| `company_type` | `TEXT` | `bus` ou `boat` |
| `created_at` | `TIMESTAMPTZ` | Date de création |

### `company_documents`

| Colonne | Type | Description |
|---------|------|-------------|
| `company_id` | `UUID` | FK → `companies.id` |
| `type` | `TEXT` | `nif` \| `patente` \| `insurance` |
| `file_url` | `TEXT` | URL Supabase Storage |
| `status` | `TEXT` | `pending` \| `approved` \| `rejected` |
| `rejection_reason` | `TEXT` | Raison du rejet (si applicable) |

> Contrainte : `UNIQUE(company_id, type)` — un seul document actif par type par compagnie.

---

## Emails automatiques

| Template | Déclencheur | Destinataire | Expéditeur |
|----------|-------------|--------------|------------|
| **A** — Demande reçue | `/api/companies/apply` | Contact compagnie | `onboarding@saintraphael.ga` |
| **A** — Invitation | `/api/onboarding/invite` | Contact compagnie | `onboarding@saintraphael.ga` |
| **Notif admin** | `/api/onboarding/notify` | `nossimaemane@gmail.com` | `admin@saintraphael.ga` |
| **B** — Docs reçus | `/api/onboarding/documents` (3 docs) | Contact compagnie | `onboarding@saintraphael.ga` |
| **OTP** — Code signature | `/api/onboarding/otp/send` | Signataire | `onboarding@saintraphael.ga` |
| **Confirmation signature** | `/api/onboarding/otp/verify` | Signataire | `onboarding@saintraphael.ga` |
| **C** — Compte activé | `/api/onboarding/activation` | Contact compagnie | `onboarding@saintraphael.ga` |
| **D** — Compte rejeté | `/api/onboarding/rejection` | Contact compagnie | `onboarding@saintraphael.ga` |
