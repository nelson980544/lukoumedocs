# 📇 Module CRM interne Lukoumé

Outil commercial **interne** de l'équipe Lukoumé pour gérer la **prospection** et le **suivi**
des compagnies de transport clientes (pipeline commercial, contacts, opportunités, activités,
devis/factures, objectifs).

> ⚠️ À ne pas confondre avec le produit vendu aux compagnies. Le CRM est l'outil de vente de
> Lukoumé **elle-même**, et il est **exclusivement accessible aux `super_admin`**.

---

## Accès & Authentification

- **URL de base** : `/admin/super/crm`
- **Rôle requis** : `super_admin` dans `user_profiles`
- **Garde front** : chaque page est enveloppée par `<CompanyGuard requireSuperAdmin>`
  (redirection vers `/admin/super/login` sinon)
- **Garde base de données** : RLS `FOR ALL USING (public.is_super_admin())` sur **toutes** les tables `crm_*`
- **Entrée navigation** : sidebar super admin → catégorie **« Commercial »** → **« CRM Lukoumé »**

Double verrou : même si la garde front était contournée, la RLS renvoie 0 ligne à un non-`super_admin`.

---

## Architecture

Le module suit l'architecture en 3 couches du projet (cf. `CLAUDE.md`) :

| Couche | Emplacement |
|--------|-------------|
| **Données** (hooks Supabase) | `app/(backoffice)/admin/super/crm/hooks/` |
| **Présentation** (composants) | `app/(backoffice)/admin/super/crm/components/` |
| **Orchestration** (pages) | `app/(backoffice)/admin/super/crm/**/page.tsx` |
| Libellés centralisés (FR) | `app/(backoffice)/admin/super/crm/labels.ts` |
| Types TypeScript | `app/types/crm.ts` |
| Migrations SQL | `database/migrations/061_crm_module.sql`, `062_crm_sync_companies.sql` |

> 🗒️ L'espace `/admin` est **locale-exempt** (cf. `middleware.ts`). Le CRM n'utilise donc pas
> `next-intl` mais un dictionnaire FR centralisé dans `labels.ts`, extensible à d'autres langues.

---

## Schéma de base de données

Migration **`061_crm_module.sql`**. Toutes les tables sont préfixées **`crm_`** pour éviter la
collision avec la table plateforme `companies`. Conventions réutilisées :
`public.is_super_admin()` (mig. 001) et `public.update_updated_at_column()` (mig. 031).

> 📐 **Référence exhaustive colonne par colonne** : voir [crm-schema.md](./crm-schema.md).

| Table | Rôle |
|-------|------|
| `crm_pipeline` | Pipelines commerciaux (`name`, `code`, `is_default`, `position`) |
| `crm_pipeline_stage` | Étapes d'un pipeline (`name`, `color`, `position`, `is_rdv_stage`) |
| `crm_source_and_type` | Sources de lead & types d'activité configurables |
| `crm_company` | **Fiche centrale** : compagnie prospectée/cliente |
| `crm_contact` | Interlocuteurs rattachés à une compagnie |
| `crm_deal` | Opportunités d'abonnement (montant, MRR, probabilité, étape) |
| `crm_activity` | Tâches / actions commerciales (échéance, résultat) |
| `crm_calendar_event` | RDV / événements planifiés |
| `crm_goal_setting` | Objectifs (prospection/jour, activités/sem, signatures/sem, fenêtre conversion) |
| `crm_deal_document` | Devis / factures (avec `payload` jsonb + champs de synchro) |

### Champs transverses

- `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`
- `created_at` / `updated_at` (trigger `update_updated_at_column`)
- `deleted_at TIMESTAMPTZ` (**soft delete**) sur les entités principales
- `owner_user_id UUID REFERENCES auth.users(id)`

### Énumérations

| Champ | Valeurs |
|-------|---------|
| `crm_company.transport_mode` | `bus` · `bateau` · `mixte` |
| `crm_company.status` | `lead` · `client` · `partenaire` |
| `crm_contact.lead_source` | `reseau` · `terrain` · `salon` · `client` · `partenaire` · `apporteur` · `prospect` |

### Seed initial (idempotent)

- 1 pipeline par défaut + 5 étapes : **Lead → Qualifié → RDV → Proposition → Signé**
- Sources/types de base (Réseau, Terrain, Salon, Apporteur, Appel, Email, Visite)
- 1 ligne `crm_goal_setting` : 5 prospection/jour, 20 activités/sem, 2 signatures/sem, fenêtre 30 j

### RLS (sur chaque table)

```sql
ALTER TABLE public.crm_xxx ENABLE ROW LEVEL SECURITY;
CREATE POLICY "crm_xxx_super_admin" ON public.crm_xxx
    FOR ALL USING (public.is_super_admin()) WITH CHECK (public.is_super_admin());
```

---

## Synchronisation automatique des inscriptions → CRM

Migration **`062_crm_sync_companies.sql`**. Quand une compagnie s'inscrit sur la plateforme
(insertion dans `public.companies`), une fiche CRM est créée **automatiquement** et son statut
suit le cycle de vie plateforme.

### Colonne de liaison

```sql
ALTER TABLE public.crm_company
  ADD COLUMN platform_company_id UUID REFERENCES public.companies(id) ON DELETE SET NULL;
-- index unique partiel → anti-doublon
```

### Mapping

| `companies.type` | → `crm_company.transport_mode` |
|------------------|-------------------------------|
| `bus` | `bus` |
| `boat` | `bateau` |

| `companies.status` | → `crm_company.status` |
|--------------------|------------------------|
| `pending` (et autres / `rejected`) | `lead` |
| `active` | `client` |
| `suspended` | `client` |

### Triggers

| Trigger | Événement | Effet |
|---------|-----------|-------|
| `trg_company_to_crm_insert` | `AFTER INSERT ON companies` | Crée la fiche CRM (statut mappé) si aucune n'est déjà liée |
| `trg_company_to_crm_status` | `AFTER UPDATE OF status ON companies` | Met à jour le statut CRM **sauf** si la fiche est passée en `partenaire` |

- La fonction `public.sync_company_to_crm()` est **`SECURITY DEFINER`** : l'utilisateur qui
  s'inscrit n'est pas `super_admin`, il faut donc contourner la RLS super_admin-only pour écrire
  dans `crm_company`.
- Une fiche mise **manuellement** en `partenaire` n'est jamais rétrogradée par la synchro.
- **Backfill idempotent** en fin de migration : importe les compagnies déjà inscrites absentes du CRM.

### Limites connues

- **Doublons** : si un prospect a été saisi à la main puis que la compagnie s'inscrit, une 2ᵉ
  fiche (liée) est créée → fusion manuelle (pas de déduplication automatique par nom).
- **Contacts** : la synchro crée la *compagnie*, pas (encore) de `crm_contact` depuis
  `contact_email`/`owner_id` de la plateforme.

---

## Routes (App Router)

Schéma identique pour chaque entité (`companies`, `contacts`, `deals`) :

```
/admin/super/crm                       → redirige vers /companies
/admin/super/crm/companies             → tableau de listing
/admin/super/crm/companies/new         → création
/admin/super/crm/companies/[id]        → fiche (onglets)
/admin/super/crm/companies/[id]/edit   → édition
```

| Route | Contenu |
|-------|---------|
| `/companies` | Listing : recherche plein texte, filtres mode+statut, tri, sélection multiple → suppression groupée (soft delete), pastille **« Inscrite »** pour les fiches issues de la plateforme |
| `/companies/[id]` | Bandeau KPIs + onglets **Général**, **Contacts**, et onglets opérationnels (Ventes, Lignes, Plannings, Embarquements, Activités, Devis/Factures, Messages — états vides structurés à brancher) |
| `/contacts` | Listing : recherche, filtre source, toggles **newsletter/sms** inline, action **mailto** |
| `/contacts/[id]` | Fiche détaillée (RGPD horodaté, source, compagnie liée) |
| `/deals` | **Kanban** (dnd-kit) : colonnes = étapes ; filtres pipeline + propriétaire |
| `/deals/[id]` | Fiche opportunité (montant, MRR, probabilité, clôture) |
| `/activities` | Listing des tâches + action « Terminer » |
| `/calendar` | Liste des événements/RDV planifiés |
| `/pipelines` | Visualisation des pipelines et de leurs étapes |
| `/sources` | Sources & types configurables (toggle actif) |
| `/synthese` | KPIs vs objectifs (`crm_goal_setting`) |

---

## Couche données (hooks)

| Hook | Responsabilité |
|------|----------------|
| `useCrmCompanies` / `useCrmCompany` | Listing (+ nb contacts), fiche, CRUD, soft delete |
| `useCrmContacts` / `useCrmContact` | Listing/fiche, CRUD, soft delete, toggle newsletter/sms |
| `useCrmDeals` / `useCrmDeal` | Listing Kanban, CRUD, **`moveDeal`** (drag → update étape, optimiste) |
| `useCrmPipelines` | Pipelines + étapes (`stagesOf`), réordonnancement |
| `useCrmSources` | Sources/types, toggle `is_active` |
| `useCrmActivities` | Activités, soft delete, `markCompleted` |
| `useCrmCalendar` | Événements de calendrier |
| `useCrmSynthese` | Agrégats KPIs (comptages `head:true`) vs objectifs |

Toutes les requêtes passent par le **client Supabase ANON** (`app/utils/supabaseClient.ts`),
filtrent `deleted_at IS NULL`, et gèrent les états `loading`/`error`.

---

## Composants clés

| Composant | Rôle |
|-----------|------|
| `CrmShell` | Garde `requireSuperAdmin` + barre d'onglets commune |
| `CrmDataTable` | Tableau générique : tri par colonne, sélection multiple, suppression groupée |
| `CrmUI` | Helpers (loading, error, badge, en-tête de page, champs de formulaire) |
| `CompanyForm` / `ContactForm` / `DealForm` | Formulaires création/édition (custom Tailwind, validation + toasts) |
| `CompanyHeader` | Bandeau d'en-tête de la fiche compagnie (statut, mode, KPIs) |
| `DealKanban` | Tableau Kanban draggable (dnd-kit `DndContext` / `useDraggable` / `useDroppable`) |

---

## Page Synthèse — KPIs

Calculés par `useCrmSynthese` et comparés à `crm_goal_setting` :

| Indicateur | Calcul |
|------------|--------|
| **Prospection / jour** | Activités créées aujourd'hui vs `prospecting_goal_per_day` |
| **Activités / semaine** | Activités créées sur 7 j vs `activity_goal_per_week` |
| **Signatures / semaine** | Compagnies passées `client` sur 7 j vs `signature_goal_per_week` |
| **Taux de conversion** | `clients / leads` sur la fenêtre `conversion_window_days` |

---

## Mise en service

1. Appliquer les migrations dans l'ordre sur Supabase (SQL editor ou `supabase db push`) :
   - `061_crm_module.sql` (tables, RLS, triggers, seed)
   - `062_crm_sync_companies.sql` (liaison + synchro auto + backfill)
2. **Vérifier la RLS** : un compte non `super_admin` → `select * from crm_company` renvoie 0 ligne.
3. **Vérifier le backfill** : `SELECT count(*) FROM companies` ≈ nb de `crm_company` avec
   `platform_company_id IS NOT NULL`.
4. **Parcours UI** (login `/admin/super/login`) :
   - `/admin/super/crm/companies` : listing, recherche, filtres, suppression groupée
   - Créer une compagnie → fiche → onglets → contact rattaché (toggles newsletter/sms)
   - `/admin/super/crm/deals` : glisser une carte entre étapes → `pipeline_stage_id` mis à jour
   - `/admin/super/crm/synthese` : KPIs vs objectifs
5. **Synchro** : créer une compagnie `status='pending'` → fiche CRM en `lead` ; passer à
   `active` → fiche en `client`.

---

*Maintenu par l'équipe TripGabon · Module CRM — Dernière mise à jour : 27 Juin 2026*
