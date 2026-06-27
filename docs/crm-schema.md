# 📐 Référence de schéma — Tables CRM (`crm_*`)

Référence exhaustive, colonne par colonne, des tables du [module CRM](./crm.md).
Source : `database/migrations/061_crm_module.sql` et `062_crm_sync_companies.sql`.

**Conventions communes** à toutes les tables :

- `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
- `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()` (maintenu par le trigger `update_updated_at_column`)
- `deleted_at TIMESTAMPTZ` → **soft delete** (présent uniquement sur les entités principales)
- **RLS** : `FOR ALL USING (public.is_super_admin()) WITH CHECK (public.is_super_admin())`

> Légende : 🔑 clé primaire · 🔗 clé étrangère · ⬇️ soft delete · ⏱️ horodatage auto

---

## `crm_pipeline`

Pipelines commerciaux.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| `id` 🔑 | UUID | PK | Identifiant |
| `name` | TEXT | NOT NULL | Nom du pipeline |
| `code` | TEXT | | Code technique optionnel |
| `is_default` | BOOLEAN | NOT NULL DEFAULT false | Pipeline sélectionné par défaut |
| `position` | INTEGER | NOT NULL DEFAULT 0 | Ordre d'affichage |
| `created_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `updated_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |

---

## `crm_pipeline_stage`

Étapes (colonnes Kanban) d'un pipeline.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| `id` 🔑 | UUID | PK | Identifiant |
| `pipeline_id` 🔗 | UUID | NOT NULL → `crm_pipeline(id)` **ON DELETE CASCADE** | Pipeline parent |
| `name` | TEXT | NOT NULL | Nom de l'étape |
| `color` | TEXT | | Couleur hex (badge/colonne) |
| `position` | INTEGER | NOT NULL DEFAULT 0 | Ordre dans le pipeline |
| `is_rdv_stage` | BOOLEAN | NOT NULL DEFAULT false | Marque l'étape « RDV » |
| `created_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `updated_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |

**Index** : `idx_crm_pipeline_stage_pipeline(pipeline_id, position)`

---

## `crm_source_and_type`

Sources de lead **et** types d'activité (table unifiée, configurable).

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| `id` 🔑 | UUID | PK | Identifiant |
| `name` | TEXT | NOT NULL | Libellé |
| `code` | TEXT | | Code technique |
| `color` | TEXT | | Couleur hex |
| `position` | INTEGER | NOT NULL DEFAULT 0 | Ordre d'affichage |
| `is_active` | BOOLEAN | NOT NULL DEFAULT true | Visible/utilisable |
| `is_lead_source` | BOOLEAN | NOT NULL DEFAULT false | Utilisable comme source de lead |
| `is_task_type` | BOOLEAN | NOT NULL DEFAULT false | Utilisable comme type d'activité |
| `description` | TEXT | | Description libre |
| `created_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `updated_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |

---

## `crm_company` ⬇️

Fiche centrale : compagnie de transport prospectée/cliente.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| `id` 🔑 | UUID | PK | Identifiant |
| `name` | TEXT | NOT NULL | Nom de la compagnie |
| `transport_mode` | TEXT | CHECK ∈ {`bus`,`bateau`,`mixte`} | Mode de transport |
| `country` | TEXT | | Pays |
| `city` | TEXT | | Ville |
| `registration_number` | TEXT | | N° d'immatriculation |
| `fleet_size` | INTEGER | | Taille de la flotte |
| `status` | TEXT | NOT NULL DEFAULT `lead`, CHECK ∈ {`lead`,`client`,`partenaire`} | Statut commercial |
| `owner_user_id` 🔗 | UUID | → `auth.users(id)` ON DELETE SET NULL | Commercial responsable |
| `notes` | TEXT | | Notes libres |
| `platform_company_id` 🔗 | UUID | → `companies(id)` ON DELETE SET NULL | Lien vers la compagnie plateforme (mig. 062) |
| `created_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `updated_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `deleted_at` ⬇️ | TIMESTAMPTZ | | Soft delete |

**Index** : `idx_crm_company_active(status) WHERE deleted_at IS NULL` ·
`idx_crm_company_platform(platform_company_id)` **UNIQUE** `WHERE platform_company_id IS NOT NULL`

---

## `crm_contact` ⬇️

Interlocuteur rattaché à une compagnie.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| `id` 🔑 | UUID | PK | Identifiant |
| `company_id` 🔗 | UUID | → `crm_company(id)` **ON DELETE CASCADE** | Compagnie rattachée |
| `user_id` 🔗 | UUID | → `auth.users(id)` ON DELETE SET NULL | Compte utilisateur lié (optionnel) |
| `owner_user_id` 🔗 | UUID | → `auth.users(id)` ON DELETE SET NULL | Commercial responsable |
| `first_name` | TEXT | | Prénom |
| `last_name` | TEXT | | Nom |
| `email` | TEXT | | Email |
| `phone` | TEXT | | Téléphone |
| `job_title` | TEXT | | Fonction |
| `lead_source` | TEXT | CHECK ∈ {`reseau`,`terrain`,`salon`,`client`,`partenaire`,`apporteur`,`prospect`} | Source du lead |
| `rgpd` | BOOLEAN | NOT NULL DEFAULT false | Consentement RGPD |
| `rgpd_date` | TIMESTAMPTZ | | Date du consentement (horodatée à la coche) |
| `newsletter` | BOOLEAN | NOT NULL DEFAULT false | Opt-in newsletter |
| `sms` | BOOLEAN | NOT NULL DEFAULT false | Opt-in SMS |
| `last_interaction_at` | TIMESTAMPTZ | | Dernière interaction |
| `created_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `updated_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `deleted_at` ⬇️ | TIMESTAMPTZ | | Soft delete |

**Index** : `idx_crm_contact_company(company_id) WHERE deleted_at IS NULL`

---

## `crm_deal` ⬇️

Opportunité d'abonnement Lukoumé.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| `id` 🔑 | UUID | PK | Identifiant |
| `company_id` 🔗 | UUID | → `crm_company(id)` **ON DELETE CASCADE** | Compagnie |
| `pipeline_id` 🔗 | UUID | → `crm_pipeline(id)` ON DELETE SET NULL | Pipeline |
| `pipeline_stage_id` 🔗 | UUID | → `crm_pipeline_stage(id)` ON DELETE SET NULL | Étape courante |
| `owner_user_id` 🔗 | UUID | → `auth.users(id)` ON DELETE SET NULL | Commercial responsable |
| `name` | TEXT | NOT NULL | Nom de l'opportunité |
| `amount` | NUMERIC | | Montant (FCFA) |
| `mrr` | NUMERIC | | Revenu mensuel récurrent |
| `status` | TEXT | | Statut libre |
| `probability` | INTEGER | | Probabilité (0–100) |
| `expected_close_at` | TIMESTAMPTZ | | Date de clôture prévue |
| `position` | INTEGER | NOT NULL DEFAULT 0 | Ordre dans la colonne Kanban |
| `notes` | TEXT | | Notes |
| `created_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `updated_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `deleted_at` ⬇️ | TIMESTAMPTZ | | Soft delete |

**Index** : `idx_crm_deal_company(company_id) WHERE deleted_at IS NULL` ·
`idx_crm_deal_stage(pipeline_stage_id, position) WHERE deleted_at IS NULL`

---

## `crm_activity` ⬇️

Tâche / action commerciale.

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| `id` 🔑 | UUID | PK | Identifiant |
| `company_id` 🔗 | UUID | → `crm_company(id)` **ON DELETE CASCADE** | Compagnie |
| `contact_id` 🔗 | UUID | → `crm_contact(id)` ON DELETE SET NULL | Contact concerné |
| `deal_id` 🔗 | UUID | → `crm_deal(id)` ON DELETE SET NULL | Opportunité liée |
| `source_and_type_id` 🔗 | UUID | → `crm_source_and_type(id)` ON DELETE SET NULL | Type d'activité |
| `owner_user_id` 🔗 | UUID | → `auth.users(id)` ON DELETE SET NULL | Responsable |
| `title` | TEXT | NOT NULL | Intitulé |
| `type` | TEXT | | Type (libre) |
| `status` | TEXT | | Statut |
| `due_at` | TIMESTAMPTZ | | Échéance |
| `completed_at` | TIMESTAMPTZ | | Date de réalisation |
| `description` | TEXT | | Description |
| `result_note` | TEXT | | Note de résultat |
| `is_definitive_failure` | BOOLEAN | NOT NULL DEFAULT false | Échec définitif |
| `created_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `updated_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `deleted_at` ⬇️ | TIMESTAMPTZ | | Soft delete |

**Index** : `idx_crm_activity_company(company_id, due_at) WHERE deleted_at IS NULL` ·
`idx_crm_activity_deal(deal_id) WHERE deleted_at IS NULL`

---

## `crm_calendar_event`

RDV / événement planifié (pas de soft delete — suppression dure).

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| `id` 🔑 | UUID | PK | Identifiant |
| `company_id` 🔗 | UUID | → `crm_company(id)` **ON DELETE CASCADE** | Compagnie |
| `contact_id` 🔗 | UUID | → `crm_contact(id)` ON DELETE SET NULL | Contact |
| `deal_id` 🔗 | UUID | → `crm_deal(id)` ON DELETE SET NULL | Opportunité |
| `source_and_type_id` 🔗 | UUID | → `crm_source_and_type(id)` ON DELETE SET NULL | Type |
| `owner_user_id` 🔗 | UUID | → `auth.users(id)` ON DELETE SET NULL | Responsable |
| `title` | TEXT | NOT NULL | Titre |
| `description` | TEXT | | Description |
| `start_at` | TIMESTAMPTZ | | Début |
| `end_at` | TIMESTAMPTZ | | Fin |
| `location` | TEXT | | Lieu |
| `created_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `updated_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |

**Index** : `idx_crm_calendar_start(start_at)`

---

## `crm_goal_setting`

Objectifs pour la page Synthèse (généralement une seule ligne).

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| `id` 🔑 | UUID | PK | Identifiant |
| `prospecting_goal_per_day` | INTEGER | NOT NULL DEFAULT 0 | Objectif prospection/jour |
| `activity_goal_per_week` | INTEGER | NOT NULL DEFAULT 0 | Objectif activités/semaine |
| `signature_goal_per_week` | INTEGER | NOT NULL DEFAULT 0 | Objectif signatures/semaine |
| `conversion_window_days` | INTEGER | NOT NULL DEFAULT 30 | Fenêtre du taux de conversion (jours) |
| `created_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `updated_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |

---

## `crm_deal_document`

Devis / factures (éventuellement synchronisés avec un système externe).

| Colonne | Type | Contraintes | Description |
|---------|------|-------------|-------------|
| `id` 🔑 | UUID | PK | Identifiant |
| `deal_id` 🔗 | UUID | → `crm_deal(id)` **ON DELETE CASCADE** | Opportunité liée |
| `company_id` 🔗 | UUID | → `crm_company(id)` ON DELETE SET NULL | Compagnie |
| `contact_id` 🔗 | UUID | → `crm_contact(id)` ON DELETE SET NULL | Contact |
| `type` | TEXT | | Type de document (devis/facture…) |
| `number` | TEXT | | Numéro de document |
| `title` | TEXT | | Titre |
| `status` | TEXT | | Statut |
| `remote_id` | TEXT | | ID dans le système externe |
| `remote_url` | TEXT | | URL du document distant |
| `sync_status` | TEXT | | État de synchronisation |
| `synced_at` | TIMESTAMPTZ | | Dernière synchronisation |
| `amount` | NUMERIC | | Montant |
| `payload` | JSONB | | Charge utile brute |
| `created_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |
| `updated_at` ⏱️ | TIMESTAMPTZ | DEFAULT now() | |

**Index** : `idx_crm_deal_document_deal(deal_id)`

---

## Graphe des relations

```
auth.users ──owner_user_id──┐
companies ──platform_company_id──► crm_company ◄──┐
                                      │           │
                  ┌───────────────────┼───────────┼────────────────┐
                  ▼                   ▼           ▼                ▼
             crm_contact         crm_deal    crm_activity   crm_calendar_event
                  │                   │  ▲          ▲ │              ▲
                  └───────contact_id──┘  │          │ └──contact_id──┘
                                         │          │
crm_pipeline ──► crm_pipeline_stage ─────┘          │ source_and_type_id
                                         │          ▼
                              crm_deal_document   crm_source_and_type
```

- `crm_company` est le pivot : `crm_contact`, `crm_deal`, `crm_activity`,
  `crm_calendar_event` la référencent (cascade à la suppression de la compagnie).
- `crm_pipeline` → `crm_pipeline_stage` → référencé par `crm_deal` (Kanban).
- `crm_source_and_type` alimente la source des contacts et le type des activités/événements.

---

## Fonctions & triggers (mig. 062)

| Objet | Type | Rôle |
|-------|------|------|
| `crm_map_mode(text)` | function (IMMUTABLE) | `bus→bus`, `boat→bateau` |
| `crm_map_status(text)` | function (IMMUTABLE) | `active`/`suspended → client`, sinon `lead` |
| `sync_company_to_crm()` | function (SECURITY DEFINER) | Crée/maj la fiche CRM depuis `companies` |
| `trg_company_to_crm_insert` | trigger AFTER INSERT | Crée la fiche CRM à l'inscription |
| `trg_company_to_crm_status` | trigger AFTER UPDATE OF status | Suit le statut (hors `partenaire`) |

---

*Maintenu par l'équipe TripGabon · Référence schéma CRM — Dernière mise à jour : 27 Juin 2026*
