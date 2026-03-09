# Final Database Schema — Multi-Sport Tournament & Auction Platform

**Version:** 1.0  
**Status:** For Developer Review & Verification  
**Database:** PostgreSQL  
**References:** SOW_Auction_Module.md, SOW_Multi_Sport_Tournament_Platform.md

---

## Document Control

| Item | Description |
|------|-------------|
| **Purpose** | Single source of truth for all database tables, columns, types, and relationships. Use for implementation, code review, and onboarding. |
| **Scope** | Full platform: Identity, Organizers, Tournaments, Players, Auction, Matches/Stats, Media, E-Commerce, Community, System. |
| **Conventions** | Snake_case table/column names; singular table names; `id` as primary key; `*_id` for foreign keys; `created_at`/`updated_at` for audit. |

---

## Table of Contents

1. [Conventions & Naming](#1-conventions--naming)
2. [Domain: Identity & Access](#2-domain-identity--access)
3. [Domain: Organizers & Subscriptions](#3-domain-organizers--subscriptions)
4. [Domain: Sports, Tournaments & Teams](#4-domain-sports-tournaments--teams)
5. [Domain: Players & Registration](#5-domain-players--registration)
6. [Domain: Auction Module](#6-domain-auction-module)
7. [Domain: Matches, Scoring & Statistics](#7-domain-matches-scoring--statistics)
8. [Domain: Media & Streaming](#8-domain-media--streaming)
9. [Domain: E-Commerce & Sponsorship](#9-domain-e-commerce--sponsorship)
10. [Domain: Community & Communication](#10-domain-community--communication)
11. [Domain: System & Integration](#11-domain-system--integration)
12. [Entity Relationship Summary](#12-entity-relationship-summary)
13. [Glossary](#13-glossary)

---

## 1. Conventions & Naming

- **Primary keys:** `id` — type `UUID` with `DEFAULT gen_random_uuid()` (or `BIGSERIAL` if preferred).
- **Foreign keys:** `<entity>_id` (e.g. `user_id`, `tournament_id`).
- **Timestamps:** `TIMESTAMPTZ`; use `created_at`, `updated_at`; optional `deleted_at` for soft deletes.
- **Nullable:** Documented per column; required fields have `NOT NULL`.
- **Multi-tenancy:** Tables scoped to organizer or tournament carry `organizer_id` and/or `tournament_id` for filtering and RLS.

---

## 2. Domain: Identity & Access

**Role:** Manages platform users, roles, and permissions. One `user` can have multiple roles (organizer, player, team_owner, etc.) and is linked to profile tables (organizer, player) as needed.

**Relationships:** `user` is the central entity; `user_role` links to `role`; `user` is referenced by `organizer`, `player`, `team.owner_user_id`, `bid.bidder_user_id`, `order`, `payment`, `notification`, etc.

### 2.1 Table: `user`

Stores all platform accounts (organizers, team owners, players, scorers, spectators). Used for login and authorization.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| email | VARCHAR(255) | NO | — | Unique login email |
| email_verified_at | TIMESTAMPTZ | YES | NULL | When email was verified |
| password_hash | VARCHAR(255) | YES | NULL | Bcrypt/Argon2 hash; NULL for OAuth-only |
| phone | VARCHAR(20) | YES | NULL | Contact number |
| phone_verified_at | TIMESTAMPTZ | YES | NULL | When phone was verified |
| full_name | VARCHAR(255) | YES | NULL | Display name |
| avatar_url | VARCHAR(500) | YES | NULL | Profile image URL |
| status | VARCHAR(20) | NO | 'active' | active, inactive, suspended, pending_verification |
| last_login_at | TIMESTAMPTZ | YES | NULL | Last successful login |
| locale | VARCHAR(10) | YES | 'en' | Preferred language |
| timezone | VARCHAR(50) | YES | 'UTC' | User timezone |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |
| deleted_at | TIMESTAMPTZ | YES | NULL | Soft delete timestamp |

**Unique constraints:** `email` (where deleted_at IS NULL).

---

### 2.2 Table: `role`

Defines platform roles used for RBAC. Referenced by `user_role`.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| code | VARCHAR(50) | NO | — | Unique: super_admin, organizer, team_owner, player, scorer, spectator |
| name | VARCHAR(100) | NO | — | Display name |
| description | TEXT | YES | NULL | Role description |
| is_system | BOOLEAN | NO | true | System roles are not deletable |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** `code`.

---

### 2.3 Table: `user_role`

Join table: which roles a user has. Enables one user to be both player and team_owner in different contexts.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| user_id | UUID | NO | — | FK → user.id |
| role_id | UUID | NO | — | FK → role.id |
| created_at | TIMESTAMPTZ | NO | now() | When role was assigned |

**Unique constraints:** (user_id, role_id).  
**Relationships:** Many-to-many between `user` and `role`.

---

### 2.4 Table: `permission`

Optional fine-grained permissions (e.g. tournament.create, auction.manage). Used with `role_permission` for RBAC.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| code | VARCHAR(100) | NO | — | Unique permission code |
| name | VARCHAR(255) | NO | — | Display name |
| module | VARCHAR(50) | YES | NULL | Grouping: tournament, auction, scoring, etc. |
| description | TEXT | YES | NULL | Description |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** `code`.

---

### 2.5 Table: `role_permission`

Links roles to permissions. A role can have many permissions.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| role_id | UUID | NO | — | FK → role.id |
| permission_id | UUID | NO | — | FK → permission.id |
| created_at | TIMESTAMPTZ | NO | now() | When permission was granted |

**Unique constraints:** (role_id, permission_id).  
**Relationships:** Many-to-many between `role` and `permission`.

---

### 2.6 Table: `user_session` (optional)

Stores active sessions for token revocation and multi-device management.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| user_id | UUID | NO | — | FK → user.id |
| token_hash | VARCHAR(255) | NO | — | Hashed JWT or session token |
| device_info | VARCHAR(255) | YES | NULL | Device/browser description |
| ip_address | INET | YES | NULL | Client IP |
| expires_at | TIMESTAMPTZ | NO | — | Session expiry |
| revoked_at | TIMESTAMPTZ | YES | NULL | When session was revoked |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |

**Relationships:** user 1–N user_session.

---

## 3. Domain: Organizers & Subscriptions

**Role:** Organizers are the primary customers who create tournaments and use the auction portal. Subscriptions and payments are tracked here (organizer billing; player registration payments are organizer-managed offline and only status is stored in `tournament_player_registration`).

**Relationships:** `user` 1–0/1 `organizer`; `organizer` 1–N `tournament`, 1–N `subscription`; `subscription_plan` 1–N `subscription`; `payment` can reference `subscription` or `order`.

### 3.1 Table: `organizer`

Profile for a tournament organizing entity. Usually one per user; links to `user` for login.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| user_id | UUID | NO | — | FK → user.id (account owner) |
| organization_name | VARCHAR(255) | YES | NULL | Business/org name |
| contact_name | VARCHAR(255) | YES | NULL | Primary contact |
| contact_email | VARCHAR(255) | YES | NULL | Contact email (can differ from user email) |
| contact_phone | VARCHAR(20) | YES | NULL | Contact phone |
| address_line1 | VARCHAR(255) | YES | NULL | Street address |
| address_line2 | VARCHAR(255) | YES | NULL | Address line 2 |
| city | VARCHAR(100) | YES | NULL | City |
| state | VARCHAR(100) | YES | NULL | State/region |
| country | VARCHAR(100) | YES | NULL | Country |
| postal_code | VARCHAR(20) | YES | NULL | Postal code |
| logo_url | VARCHAR(500) | YES | NULL | Organizer logo |
| website_url | VARCHAR(500) | YES | NULL | Website |
| status | VARCHAR(20) | NO | 'active' | active, suspended, inactive |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |
| deleted_at | TIMESTAMPTZ | YES | NULL | Soft delete |

**Relationships:** user 1–0/1 organizer; organizer 1–N tournament, 1–N subscription.

---

### 3.2 Table: `subscription_plan`

Defines plan types: per-tournament, monthly, annual, enterprise. Referenced by `subscription`.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| code | VARCHAR(50) | NO | — | Unique: per_tournament, monthly, annual, enterprise |
| name | VARCHAR(100) | NO | — | Display name |
| description | TEXT | YES | NULL | Plan description |
| plan_type | VARCHAR(30) | NO | — | auction_only, full_platform, enterprise |
| billing_interval | VARCHAR(20) | YES | NULL | one_time, monthly, annual |
| price_amount | DECIMAL(12,2) | NO | — | Price in base currency |
| price_currency | VARCHAR(3) | NO | 'INR' | Currency code |
| max_tournaments_per_month | INTEGER | YES | NULL | NULL = unlimited |
| features_json | JSONB | YES | NULL | Feature flags or limits |
| is_active | BOOLEAN | NO | true | Whether plan is available for purchase |
| sort_order | INTEGER | YES | 0 | Display order |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** `code`.

---

### 3.3 Table: `subscription`

An active or past subscription instance for an organizer. Links to payments and invoices.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| organizer_id | UUID | NO | — | FK → organizer.id |
| subscription_plan_id | UUID | NO | — | FK → subscription_plan.id |
| status | VARCHAR(20) | NO | — | active, cancelled, expired, past_due, trialing |
| started_at | TIMESTAMPTZ | NO | — | Subscription start |
| ends_at | TIMESTAMPTZ | YES | NULL | End date (NULL for open-ended) |
| cancelled_at | TIMESTAMPTZ | YES | NULL | When cancelled |
| external_subscription_id | VARCHAR(255) | YES | NULL | Gateway subscription ID (e.g. Stripe) |
| metadata_json | JSONB | YES | NULL | Extra gateway data |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** organizer N–1; subscription_plan N–1; subscription 1–N invoice, 1–N payment (for renewals).

---

### 3.4 Table: `payment`

All platform payments: organizer subscriptions, marketplace orders, etc. Player registration fees are not processed here (organizer-managed offline); only organizer subscription and e-commerce payments.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| user_id | UUID | YES | NULL | FK → user.id (payer) |
| organizer_id | UUID | YES | NULL | FK → organizer.id (when paying for subscription) |
| subscription_id | UUID | YES | NULL | FK → subscription.id |
| order_id | UUID | YES | NULL | FK → order.id (e-commerce) |
| invoice_id | UUID | YES | NULL | FK → invoice.id |
| amount | DECIMAL(12,2) | NO | — | Payment amount |
| currency | VARCHAR(3) | NO | 'INR' | Currency code |
| status | VARCHAR(30) | NO | — | pending, completed, failed, refunded, cancelled |
| payment_gateway | VARCHAR(50) | YES | NULL | razorpay, payu, stripe, etc. |
| gateway_transaction_id | VARCHAR(255) | YES | NULL | Gateway reference |
| gateway_response_json | JSONB | YES | NULL | Raw gateway response |
| paid_at | TIMESTAMPTZ | YES | NULL | When payment succeeded |
| failure_reason | TEXT | YES | NULL | Error message if failed |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** user N–1; organizer N–1; subscription N–1; order N–1; invoice N–1.

---

### 3.5 Table: `invoice`

Invoice records for subscriptions or one-time charges. Optional but recommended for compliance.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| organizer_id | UUID | NO | — | FK → organizer.id |
| subscription_id | UUID | YES | NULL | FK → subscription.id |
| invoice_number | VARCHAR(50) | NO | — | Unique human-readable number |
| status | VARCHAR(20) | NO | — | draft, issued, paid, overdue, cancelled |
| amount | DECIMAL(12,2) | NO | — | Invoice amount |
| currency | VARCHAR(3) | NO | 'INR' | Currency code |
| tax_amount | DECIMAL(12,2) | YES | 0 | Tax component |
| due_date | DATE | YES | NULL | Payment due date |
| paid_at | TIMESTAMPTZ | YES | NULL | When paid |
| pdf_url | VARCHAR(500) | YES | NULL | Generated invoice PDF |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** invoice_number.  
**Relationships:** organizer N–1; subscription N–1; payment 1–N references invoice.

---

## 4. Domain: Sports, Tournaments & Teams

**Role:** Core sports catalog, tournament types, venues, and tournament instances with their teams. Teams are per-tournament; `team_membership` links players to teams after auction or manual assignment.

**Relationships:** sport 1–N tournament; organizer 1–N tournament; tournament 1–N team, 1–N team_membership; team 1–N team_membership N–1 player; venue 1–N match (later).

### 4.1 Table: `sport`

Master list of sports (Cricket, Volleyball, Football, Basketball, Badminton, Kabaddi, etc.).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| code | VARCHAR(30) | NO | — | Unique: cricket, volleyball, football, etc. |
| name | VARCHAR(100) | NO | — | Display name |
| description | TEXT | YES | NULL | Short description |
| icon_url | VARCHAR(500) | YES | NULL | Sport icon |
| is_active | BOOLEAN | NO | true | Whether sport is selectable |
| sort_order | INTEGER | YES | 0 | Display order |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** `code`.

---

### 4.2 Table: `competition_level`

Tournament level/type: Open, Community, District, State, National, School, College, Corporate, International, etc.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| code | VARCHAR(50) | NO | — | Unique code |
| name | VARCHAR(100) | NO | — | Display name |
| description | TEXT | YES | NULL | Description |
| sort_order | INTEGER | YES | 0 | Display order |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** `code`.

---

### 4.3 Table: `age_category`

Age groups: U13, U16, U19, Open, etc.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| code | VARCHAR(30) | NO | — | Unique code |
| name | VARCHAR(100) | NO | — | Display name |
| min_age | INTEGER | YES | NULL | Minimum age (inclusive) |
| max_age | INTEGER | YES | NULL | Maximum age (inclusive); NULL = no cap |
| description | TEXT | YES | NULL | Description |
| sort_order | INTEGER | YES | 0 | Display order |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** `code`.

---

### 4.4 Table: `venue`

Stadiums/grounds where matches can be played. Referenced by `match`.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| name | VARCHAR(255) | NO | — | Venue name |
| address_line1 | VARCHAR(255) | YES | NULL | Street address |
| address_line2 | VARCHAR(255) | YES | NULL | Address line 2 |
| city | VARCHAR(100) | YES | NULL | City |
| state | VARCHAR(100) | YES | NULL | State |
| country | VARCHAR(100) | YES | NULL | Country |
| postal_code | VARCHAR(20) | YES | NULL | Postal code |
| latitude | DECIMAL(10,7) | YES | NULL | Geo latitude |
| longitude | DECIMAL(10,7) | YES | NULL | Geo longitude |
| contact_phone | VARCHAR(20) | YES | NULL | Venue contact |
| capacity | INTEGER | YES | NULL | Spectator capacity |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |
| deleted_at | TIMESTAMPTZ | YES | NULL | Soft delete |

**Relationships:** venue 1–N match.

---

### 4.5 Table: `tournament`

A tournament instance. Belongs to an organizer and a sport; has status and config for auction, registration, format.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| organizer_id | UUID | NO | — | FK → organizer.id |
| sport_id | UUID | NO | — | FK → sport.id |
| competition_level_id | UUID | YES | NULL | FK → competition_level.id |
| age_category_id | UUID | YES | NULL | FK → age_category.id |
| name | VARCHAR(255) | NO | — | Tournament name |
| slug | VARCHAR(255) | YES | NULL | URL-friendly unique slug |
| description | TEXT | YES | NULL | Description |
| start_date | DATE | YES | NULL | Tournament start |
| end_date | DATE | YES | NULL | Tournament end |
| format | VARCHAR(30) | YES | NULL | knockout, round_robin, league, pools, ladder, hybrid |
| status | VARCHAR(30) | NO | 'draft' | draft, setup, ready, in_progress, completed, cancelled |
| registration_type | VARCHAR(20) | NO | 'free' | free, paid (organizer-managed payment) |
| registration_deadline | TIMESTAMPTZ | YES | NULL | Deadline for player registration |
| auction_enabled | BOOLEAN | NO | false | Whether auction is used for this tournament |
| rules_text | TEXT | YES | NULL | Custom rules/regulations |
| banner_url | VARCHAR(500) | YES | NULL | Tournament banner image |
| logo_url | VARCHAR(500) | YES | NULL | Tournament logo |
| public_registration_link_token | VARCHAR(64) | YES | NULL | Token for public registration URL |
| metadata_json | JSONB | YES | NULL | Extra config (prize pool, etc.) |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |
| created_by | UUID | YES | NULL | FK → user.id |
| updated_by | UUID | YES | NULL | FK → user.id |
| deleted_at | TIMESTAMPTZ | YES | NULL | Soft delete |

**Unique constraints:** (organizer_id, slug) or slug globally.  
**Relationships:** organizer N–1; sport N–1; competition_level N–1; age_category N–1; tournament 1–N team, 1–0/1 auction, 1–N tournament_player_registration, 1–N match.

---

### 4.6 Table: `tournament_stage`

Optional groups/stages within a tournament (e.g. Group A, Group B, League Stage, Playoffs). Used for scheduling and standings.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| tournament_id | UUID | NO | — | FK → tournament.id |
| name | VARCHAR(100) | NO | — | Stage name (e.g. Group A, Playoffs) |
| code | VARCHAR(30) | YES | NULL | Short code |
| stage_type | VARCHAR(30) | YES | NULL | group, league, knockout, playoff |
| sort_order | INTEGER | YES | 0 | Order within tournament |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** tournament N–1; tournament_stage 1–N match (match.stage_id).

---

### 4.7 Table: `team`

Team within a specific tournament. Has an owner (user); members are in `team_membership` (filled after auction or manual add).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| tournament_id | UUID | NO | — | FK → tournament.id |
| name | VARCHAR(255) | NO | — | Team name |
| short_name | VARCHAR(50) | YES | NULL | Abbreviation |
| owner_user_id | UUID | YES | NULL | FK → user.id (team owner/captain) |
| owner_name | VARCHAR(255) | YES | NULL | Owner display name (if no user) |
| owner_contact_email | VARCHAR(255) | YES | NULL | Owner email |
| owner_contact_phone | VARCHAR(20) | YES | NULL | Owner phone |
| logo_url | VARCHAR(500) | YES | NULL | Team logo |
| status | VARCHAR(20) | NO | 'active' | active, inactive, disqualified |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |
| created_by | UUID | YES | NULL | FK → user.id |
| deleted_at | TIMESTAMPTZ | YES | NULL | Soft delete |

**Relationships:** tournament N–1; user N–1 (owner_user_id); team 1–N team_membership, 1–N auction_team, 1–N match_participant.

---

### 4.8 Table: `team_membership`

Links a player to a team for a tournament. Populated after auction (sold players + icon players) or by manual assignment. Used for rosters and scoring.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| team_id | UUID | NO | — | FK → team.id |
| player_id | UUID | NO | — | FK → player.id |
| tournament_id | UUID | NO | — | FK → tournament.id (denormalized for queries) |
| role | VARCHAR(30) | YES | NULL | captain, vice_captain, player |
| jersey_number | INTEGER | YES | NULL | Squad number |
| is_icon_player | BOOLEAN | NO | false | True if pre-assigned icon (not from auction bid) |
| status | VARCHAR(20) | NO | 'active' | active, inactive, removed |
| joined_at | TIMESTAMPTZ | NO | now() | When added to team |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** (team_id, player_id) or (tournament_id, player_id) depending on business rule (one player per team per tournament).  
**Relationships:** team N–1; player N–1; tournament N–1. Post-auction sync creates rows here from auction_player (sold) and auction_icon_player.

---

## 5. Domain: Players & Registration

**Role:** Player profiles (can link to `user`) and per-tournament registration with approval and payment status. Auction eligibility is derived from approved registrations (and payment_status when registration is paid).

**Relationships:** user 1–0/1 player; player 1–N tournament_player_registration; tournament 1–N tournament_player_registration; tournament_player_registration feeds auction_player.

### 5.1 Table: `player`

Cross-tournament player identity. May or may not have an associated user account (e.g. registered via link without account).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| user_id | UUID | YES | NULL | FK → user.id (nullable for guest registrations) |
| primary_sport_id | UUID | YES | NULL | FK → sport.id |
| full_name | VARCHAR(255) | NO | — | Player name |
| date_of_birth | DATE | YES | NULL | DOB for age category |
| gender | VARCHAR(20) | YES | NULL | male, female, other, prefer_not_to_say |
| profile_photo_url | VARCHAR(500) | YES | NULL | Profile image |
| district | VARCHAR(100) | YES | NULL | District (for geographic stats) |
| state | VARCHAR(100) | YES | NULL | State |
| country | VARCHAR(100) | YES | NULL | Country |
| bio | TEXT | YES | NULL | Short bio |
| contact_email | VARCHAR(255) | YES | NULL | Contact email |
| contact_phone | VARCHAR(20) | YES | NULL | Contact phone |
| status | VARCHAR(20) | NO | 'active' | active, inactive, suspended |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |
| deleted_at | TIMESTAMPTZ | YES | NULL | Soft delete |

**Relationships:** user N–1; sport N–1; player 1–N tournament_player_registration, 1–N auction_player, 1–N team_membership, 1–N player_match_stat.

---

### 5.2 Table: `player_document`

Documents uploaded for a player (ID proof, certificates). Can be scoped to registration if needed.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| player_id | UUID | NO | — | FK → player.id |
| tournament_player_registration_id | UUID | YES | NULL | FK → tournament_player_registration.id (if doc is per registration) |
| document_type | VARCHAR(50) | NO | — | id_proof, certificate, photo, other |
| file_url | VARCHAR(500) | NO | — | Stored file URL |
| file_name | VARCHAR(255) | YES | NULL | Original filename |
| mime_type | VARCHAR(100) | YES | NULL | MIME type |
| file_size_bytes | BIGINT | YES | NULL | File size |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** player N–1; tournament_player_registration N–1 (optional).

---

### 5.3 Table: `tournament_player_registration`

A player’s registration for one tournament. Includes approval and organizer-managed payment status. Base price and category can be set here and copied to auction_player.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| tournament_id | UUID | NO | — | FK → tournament.id |
| player_id | UUID | NO | — | FK → player.id |
| registration_status | VARCHAR(30) | NO | 'pending' | pending, approved, rejected |
| payment_status | VARCHAR(30) | NO | 'unpaid' | unpaid, pending, paid, rejected (when registration_type = paid) |
| registration_type | VARCHAR(20) | NO | 'free' | free, paid (copied from tournament or overridden) |
| category | VARCHAR(50) | YES | NULL | batsman, bowler, all_rounder, setter, etc. (sport-specific) |
| position | VARCHAR(50) | YES | NULL | Playing position label |
| base_price_points | INTEGER | YES | NULL | Base price in points (for auction); set by organizer |
| previous_stats_json | JSONB | YES | NULL | Snapshot of stats at registration |
| payment_receipt_url | VARCHAR(500) | YES | NULL | Uploaded receipt (organizer-managed payment) |
| organizer_notes | TEXT | YES | NULL | Internal notes from organizer |
| rejected_reason | TEXT | YES | NULL | Reason if registration_status = rejected |
| registered_at | TIMESTAMPTZ | NO | now() | When registration was submitted |
| approved_at | TIMESTAMPTZ | YES | NULL | When approved (if applicable) |
| approved_by | UUID | YES | NULL | FK → user.id |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** (tournament_id, player_id).  
**Relationships:** tournament N–1; player N–1. Approved (and paid if required) registrations are used to build auction_player entries.

---

## 6. Domain: Auction Module

**Role:** Points-based auction per tournament: config (points, team size, icon player), auction_team (points balance), auction_player (pool with base price), bids, icon assignments, and generated reports. Feeds team_membership on completion.

**Relationships:** tournament 1–0/1 auction; auction 1–N auction_team N–1 team; auction 1–N auction_player N–1 player; auction_team 1–0/1 auction_icon_player; auction 1–N bid; auction 1–N auction_report.

### 6.1 Table: `auction`

One auction per tournament. Holds configuration and lifecycle (draft → scheduled → in_progress → completed).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| tournament_id | UUID | NO | — | FK → tournament.id (UNIQUE) |
| organizer_id | UUID | NO | — | FK → organizer.id |
| status | VARCHAR(30) | NO | 'draft' | draft, scheduled, in_progress, completed, cancelled |
| points_per_team | INTEGER | NO | — | Equal points given to each team (e.g. 1000) |
| team_size | INTEGER | NO | — | Required squad size (11, 13, 15, or custom) |
| icon_player_enabled | BOOLEAN | NO | false | Whether icon player feature is on |
| bid_increment_type | VARCHAR(20) | YES | 'fixed' | fixed, step, custom |
| bid_increment_value | INTEGER | YES | NULL | Increment in points (e.g. 5, 10) |
| bid_timer_seconds | INTEGER | YES | NULL | Countdown per player (seconds) |
| auction_type | VARCHAR(30) | YES | 'standard' | standard, category_based, round_based |
| scheduled_at | TIMESTAMPTZ | YES | NULL | Planned start time |
| started_at | TIMESTAMPTZ | YES | NULL | Actual start time |
| completed_at | TIMESTAMPTZ | YES | NULL | When auction ended |
| current_auction_player_id | UUID | YES | NULL | FK → auction_player.id (player currently on block) |
| metadata_json | JSONB | YES | NULL | Extra config (rounds, category order, etc.) |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |
| created_by | UUID | YES | NULL | FK → user.id |

**Unique constraints:** tournament_id.  
**Relationships:** tournament 1–0/1; organizer N–1; auction 1–N auction_team, auction_player, bid, auction_icon_player, auction_report, auction_session_event.

---

### 6.2 Table: `auction_team`

Team’s participation in an auction with points balance. One row per team per auction.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| auction_id | UUID | NO | — | FK → auction.id |
| team_id | UUID | NO | — | FK → team.id |
| starting_points | INTEGER | NO | — | Initial points (usually = auction.points_per_team) |
| points_remaining | INTEGER | NO | — | Current balance (updated on each winning bid) |
| points_spent | INTEGER | NO | 0 | Total spent so far |
| max_players | INTEGER | NO | — | Same as auction.team_size (squad size) |
| players_acquired | INTEGER | NO | 0 | Count of players won in auction |
| auto_bid_enabled | BOOLEAN | NO | false | Future: auto-bid up to limit |
| auto_bid_max_points | INTEGER | YES | NULL | Future: cap for auto-bid |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** (auction_id, team_id).  
**Relationships:** auction N–1; team N–1; auction_team 1–N bid; auction_team 1–0/1 auction_icon_player.

---

### 6.3 Table: `auction_icon_player`

Pre-assigned icon player per team for this auction. Icon player does not go through bidding and does not consume points.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| auction_id | UUID | NO | — | FK → auction.id |
| team_id | UUID | NO | — | FK → team.id |
| player_id | UUID | NO | — | FK → player.id |
| assigned_at | TIMESTAMPTZ | NO | now() | When assigned |
| assigned_by | UUID | YES | NULL | FK → user.id |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |

**Unique constraints:** (auction_id, team_id) — one icon per team; (auction_id, player_id) — one player can be icon for only one team.  
**Relationships:** auction N–1; team N–1; player N–1. Icon players are excluded from auction_player pool and must be written to team_membership with is_icon_player = true on completion.

---

### 6.4 Table: `auction_player`

A player in the auction pool for this auction. Built from approved (and paid if required) tournament_player_registration; status tracks sold/unsold/withdrawn.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| auction_id | UUID | NO | — | FK → auction.id |
| player_id | UUID | NO | — | FK → player.id |
| tournament_player_registration_id | UUID | YES | NULL | FK → tournament_player_registration.id |
| base_price_points | INTEGER | NO | — | Starting/minimum bid in points |
| category | VARCHAR(50) | YES | NULL | batsman, bowler, all_rounder, etc. |
| display_order | INTEGER | YES | NULL | Order in auction (for category/round ordering) |
| status | VARCHAR(30) | NO | 'scheduled' | scheduled, on_block, sold, unsold, withdrawn |
| sold_to_team_id | UUID | YES | NULL | FK → team.id (when status = sold) |
| final_bid_points | INTEGER | YES | NULL | Winning bid amount |
| sold_at | TIMESTAMPTZ | YES | NULL | When sold |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** auction N–1; player N–1; team N–1 (sold_to_team_id); tournament_player_registration N–1. When status = sold, create team_membership(team_id = sold_to_team_id, player_id, tournament_id, is_icon_player = false).

---

### 6.5 Table: `bid`

Every bid placed during auction. Used for history, current high bid, and audit.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| auction_id | UUID | NO | — | FK → auction.id |
| auction_player_id | UUID | NO | — | FK → auction_player.id |
| auction_team_id | UUID | NO | — | FK → auction_team.id (bidding team) |
| bidder_user_id | UUID | YES | NULL | FK → user.id (who placed bid; may be team owner) |
| bid_points | INTEGER | NO | — | Bid amount in points |
| sequence | INTEGER | NO | — | Order of bid for this auction_player (1, 2, 3…) |
| is_winning_bid | BOOLEAN | NO | false | True for the bid that won the player |
| created_at | TIMESTAMPTZ | NO | now() | When bid was placed |

**Relationships:** auction N–1; auction_player N–1; auction_team N–1; user N–1. Index on (auction_player_id, sequence) and (auction_id, created_at) for live feed and history.

---

### 6.6 Table: `auction_session_event`

Optional timeline of auction control events (started, paused, resumed, player_opened, player_closed, ended). Useful for replay and analytics.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| auction_id | UUID | NO | — | FK → auction.id |
| event_type | VARCHAR(50) | NO | — | started, paused, resumed, player_opened, player_closed, ended |
| auction_player_id | UUID | YES | NULL | FK → auction_player.id (when player_opened/closed) |
| payload_json | JSONB | YES | NULL | Extra data (e.g. timer used, winning_team_id) |
| triggered_by | UUID | YES | NULL | FK → user.id |
| created_at | TIMESTAMPTZ | NO | now() | Event time |

**Relationships:** auction N–1; auction_player N–1; user N–1.

---

### 6.7 Table: `auction_report`

Metadata for generated auction reports (PDF/Excel/CSV). File stored in object storage; URL and summary stored here.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| auction_id | UUID | NO | — | FK → auction.id |
| format | VARCHAR(20) | NO | — | pdf, excel, csv |
| file_url | VARCHAR(500) | YES | NULL | Download URL |
| file_size_bytes | BIGINT | YES | NULL | File size |
| summary_json | JSONB | YES | NULL | Summary stats (total_sold, total_unsold, points_per_team, etc.) |
| generated_at | TIMESTAMPTZ | NO | now() | When report was generated |
| generated_by | UUID | YES | NULL | FK → user.id |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |

**Relationships:** auction N–1; user N–1 (generated_by).

---

## 7. Domain: Matches, Scoring & Statistics

**Role:** Matches belong to tournaments; match_participant links teams; match_event stores fine-grained events (ball-by-ball, goals, etc.); player_match_stat and team_match_stat are per-match aggregates; player_tournament_stat and player_geographic_stat are rollups for analytics.

**Relationships:** tournament 1–N match; match 1–N match_participant N–1 team; match 1–N match_event; match 1–N player_match_stat N–1 player; match 1–N team_match_stat N–1 team; player 1–N player_tournament_stat N–1 tournament.

### 7.1 Table: `match`

A single match in a tournament. Sport and venue determine scoring rules and display.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| tournament_id | UUID | NO | — | FK → tournament.id |
| sport_id | UUID | NO | — | FK → sport.id |
| venue_id | UUID | YES | NULL | FK → venue.id |
| stage_id | UUID | YES | NULL | FK → tournament_stage.id |
| match_number | INTEGER | YES | NULL | Sequence number in tournament |
| scheduled_at | TIMESTAMPTZ | YES | NULL | Scheduled start |
| started_at | TIMESTAMPTZ | YES | NULL | Actual start |
| completed_at | TIMESTAMPTZ | YES | NULL | When match ended |
| status | VARCHAR(30) | NO | 'scheduled' | scheduled, in_progress, completed, abandoned, cancelled |
| format_config_json | JSONB | YES | NULL | Overs (cricket), sets (volleyball), halves (football), etc. |
| toss_winner_team_id | UUID | YES | NULL | FK → team.id (cricket toss) |
| toss_decision | VARCHAR(50) | YES | NULL | bat, bowl, etc. |
| result_summary | TEXT | YES | NULL | Short result text (e.g. "Team A won by 5 wickets") |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** tournament N–1; sport N–1; venue N–1; tournament_stage N–1; team N–1 (toss_winner_team_id); match 1–N match_participant, match_event, player_match_stat, team_match_stat.

---

### 7.2 Table: `match_participant`

Teams in a match (home/away or team1/team2). Holds final score per side.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| match_id | UUID | NO | — | FK → match.id |
| team_id | UUID | NO | — | FK → team.id |
| is_home | BOOLEAN | NO | false | Home team |
| is_away | BOOLEAN | NO | false | Away team |
| score_value | INTEGER | YES | NULL | Main score (runs, goals, points) |
| score_extra_json | JSONB | YES | NULL | Wickets, overs, sets, etc. |
| result | VARCHAR(20) | YES | NULL | win, loss, tie, nr (no result) |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** (match_id, team_id).  
**Relationships:** match N–1; team N–1. Typically two rows per match (home + away).

---

### 7.3 Table: `match_event`

Fine-grained events for live scoring (ball-by-ball, goal, point, wicket, etc.). Event_type + data (JSONB) support multiple sports.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| match_id | UUID | NO | — | FK → match.id |
| event_type | VARCHAR(50) | NO | — | cricket_ball, football_goal, volleyball_point, etc. |
| sequence | INTEGER | NO | — | Order within match |
| team_id | UUID | YES | NULL | FK → team.id (team concerned) |
| player_id | UUID | YES | NULL | FK → player.id (actor: batsman, scorer, etc.) |
| secondary_player_id | UUID | YES | NULL | FK → player.id (e.g. bowler, assister) |
| data | JSONB | YES | NULL | Sport-specific payload (runs, extras, wicket type, etc.) |
| occurred_at | TIMESTAMPTZ | NO | now() | When event occurred |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |

**Relationships:** match N–1; team N–1; player N–1 (x2). Index on (match_id, sequence) for replay and stats derivation.

---

### 7.4 Table: `player_match_stat`

Per-player, per-match aggregated stats. Derived from match_event; structure can be sport-specific columns or JSONB.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| match_id | UUID | NO | — | FK → match.id |
| player_id | UUID | NO | — | FK → player.id |
| team_id | UUID | NO | — | FK → team.id (team they played for) |
| sport_id | UUID | NO | — | FK → sport.id |
| batting_runs | INTEGER | YES | 0 | Cricket: runs scored |
| batting_balls | INTEGER | YES | 0 | Cricket: balls faced |
| batting_fours | INTEGER | YES | 0 | Cricket: fours |
| batting_sixes | INTEGER | YES | 0 | Cricket: sixes |
| bowling_wickets | INTEGER | YES | 0 | Cricket: wickets taken |
| bowling_runs_conceded | INTEGER | YES | 0 | Cricket: runs conceded |
| bowling_overs | DECIMAL(6,2) | YES | 0 | Cricket: overs bowled |
| bowling_maidens | INTEGER | YES | 0 | Cricket: maiden overs |
| fielding_catches | INTEGER | YES | 0 | Cricket: catches |
| fielding_run_outs | INTEGER | YES | 0 | Cricket: run outs |
| goals | INTEGER | YES | 0 | Football/other: goals |
| assists | INTEGER | YES | 0 | Football/other: assists |
| points_scored | INTEGER | YES | 0 | Volleyball/Basketball: points |
| stats_json | JSONB | YES | NULL | Other sport-specific stats |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** (match_id, player_id).  
**Relationships:** match N–1; player N–1; team N–1; sport N–1.

---

### 7.5 Table: `team_match_stat`

Per-team, per-match totals (score, wickets, etc.). Used for standings and scorecards.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| match_id | UUID | NO | — | FK → match.id |
| team_id | UUID | NO | — | FK → team.id |
| score_value | INTEGER | YES | NULL | Main score |
| score_extra_json | JSONB | YES | NULL | Wickets, overs, sets, etc. |
| result | VARCHAR(20) | YES | NULL | win, loss, tie, nr |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** (match_id, team_id).  
**Relationships:** match N–1; team N–1.

---

### 7.6 Table: `player_tournament_stat`

Aggregated per-player, per-tournament stats. Built from player_match_stat for leaderboards and reports.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| player_id | UUID | NO | — | FK → player.id |
| tournament_id | UUID | NO | — | FK → tournament.id |
| sport_id | UUID | NO | — | FK → sport.id |
| matches_played | INTEGER | NO | 0 | Matches participated |
| batting_runs | INTEGER | YES | 0 | Cricket: total runs |
| batting_average | DECIMAL(8,2) | YES | NULL | Cricket: average |
| batting_strike_rate | DECIMAL(8,2) | YES | NULL | Cricket: strike rate |
| bowling_wickets | INTEGER | YES | 0 | Cricket: wickets |
| bowling_economy | DECIMAL(6,2) | YES | NULL | Cricket: economy |
| goals | INTEGER | YES | 0 | Football: goals |
| assists | INTEGER | YES | 0 | Football: assists |
| points_scored | INTEGER | YES | 0 | Volleyball/Basketball: points |
| stats_json | JSONB | YES | NULL | Other aggregates |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** (player_id, tournament_id).  
**Relationships:** player N–1; tournament N–1; sport N–1.

---

### 7.7 Table: `player_geographic_stat`

Rollup of player performance by geography and competition type (district, state, national, school, corporate, etc.) for multi-dimensional analytics.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| player_id | UUID | NO | — | FK → player.id |
| sport_id | UUID | NO | — | FK → sport.id |
| dimension | VARCHAR(30) | NO | — | district, state, national |
| dimension_value | VARCHAR(100) | YES | NULL | District name, state name, or 'national' |
| competition_type | VARCHAR(50) | YES | NULL | open, school, college, corporate, etc. |
| matches_played | INTEGER | NO | 0 | Matches in this dimension |
| stats_json | JSONB | YES | NULL | Aggregated metrics (runs, wickets, goals, etc.) |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** (player_id, sport_id, dimension, dimension_value, competition_type) or similar.  
**Relationships:** player N–1; sport N–1.

---

## 8. Domain: Media & Streaming

**Role:** Store references to uploaded media (photos, videos) and link them to entities (match, tournament, team, player). Live streams for matches or tournaments.

**Relationships:** media_asset 1–N entity_media; entity_media references (entity_type, entity_id); live_stream references match or tournament.

### 8.1 Table: `media_asset`

Uploaded file metadata. Actual files in object storage (S3, etc.); URL stored here.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| file_url | VARCHAR(500) | NO | — | Full URL to file |
| file_type | VARCHAR(20) | NO | — | image, video, document |
| mime_type | VARCHAR(100) | YES | NULL | MIME type |
| file_size_bytes | BIGINT | YES | NULL | File size |
| width_px | INTEGER | YES | NULL | Image/video width |
| height_px | INTEGER | YES | NULL | Image/video height |
| duration_seconds | INTEGER | YES | NULL | Video duration |
| thumbnail_url | VARCHAR(500) | YES | NULL | Thumbnail for video |
| uploaded_by | UUID | YES | NULL | FK → user.id |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |

**Relationships:** user N–1 (uploaded_by); media_asset 1–N entity_media.

---

### 8.2 Table: `entity_media`

Polymorphic link between media_asset and any entity (match, tournament, team, player, post).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| media_asset_id | UUID | NO | — | FK → media_asset.id |
| entity_type | VARCHAR(50) | NO | — | match, tournament, team, player, post |
| entity_id | UUID | NO | — | ID of the entity |
| role | VARCHAR(50) | YES | NULL | profile_photo, banner, highlight, gallery, etc. |
| sort_order | INTEGER | YES | 0 | Display order |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |

**Relationships:** media_asset N–1. Index on (entity_type, entity_id) for listing by entity.

---

### 8.3 Table: `live_stream`

Live stream configuration for a match or tournament (YouTube, Facebook, custom RTMP).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| match_id | UUID | YES | NULL | FK → match.id |
| tournament_id | UUID | YES | NULL | FK → tournament.id |
| provider | VARCHAR(50) | NO | — | youtube, facebook, custom, etc. |
| stream_url | VARCHAR(500) | YES | NULL | Public viewing URL |
| stream_key | VARCHAR(255) | YES | NULL | Ingestion key (keep secure) |
| status | VARCHAR(20) | NO | 'scheduled' | scheduled, live, ended, failed |
| started_at | TIMESTAMPTZ | YES | NULL | When stream went live |
| ended_at | TIMESTAMPTZ | YES | NULL | When stream ended |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** match N–1; tournament N–1. At least one of match_id or tournament_id should be set.

---

## 9. Domain: E-Commerce & Sponsorship

**Role:** Product catalog, orders, order items, shipping; payments already in `payment`. Sponsors and sponsorship assignments to tournaments/teams.

**Relationships:** product_category 1–N product; user 1–N order; order 1–N order_item N–1 product; payment N–1 order; sponsor 1–N sponsorship_assignment; tournament/team N–1 sponsorship_assignment.

### 9.1 Table: `product_category`

Categories for marketplace (Bats, Balls, Jerseys, etc.).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| parent_id | UUID | YES | NULL | FK → product_category.id (for hierarchy) |
| name | VARCHAR(255) | NO | — | Category name |
| slug | VARCHAR(255) | YES | NULL | URL slug |
| description | TEXT | YES | NULL | Description |
| image_url | VARCHAR(500) | YES | NULL | Category image |
| sort_order | INTEGER | YES | 0 | Display order |
| is_active | BOOLEAN | NO | true | Whether category is visible |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** product_category self-reference (parent_id); product_category 1–N product.

---

### 9.2 Table: `product`

Marketplace product (sports accessories, merchandise).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| product_category_id | UUID | NO | — | FK → product_category.id |
| name | VARCHAR(255) | NO | — | Product name |
| slug | VARCHAR(255) | YES | NULL | URL slug |
| description | TEXT | YES | NULL | Full description |
| short_description | VARCHAR(500) | YES | NULL | Short blurb |
| sku | VARCHAR(100) | YES | NULL | Stock keeping unit |
| price | DECIMAL(12,2) | NO | — | Selling price |
| currency | VARCHAR(3) | NO | 'INR' | Currency |
| compare_at_price | DECIMAL(12,2) | YES | NULL | Original price (for display) |
| cost_price | DECIMAL(12,2) | YES | NULL | Cost (internal) |
| quantity_in_stock | INTEGER | NO | 0 | Available quantity |
| image_url | VARCHAR(500) | YES | NULL | Primary image |
| images_json | JSONB | YES | NULL | Array of additional image URLs |
| is_active | BOOLEAN | NO | true | Whether product is listed |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |
| deleted_at | TIMESTAMPTZ | YES | NULL | Soft delete |

**Relationships:** product_category N–1; product 1–N order_item.

---

### 9.3 Table: `order`

E-commerce order placed by a user.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| user_id | UUID | NO | — | FK → user.id (customer) |
| order_number | VARCHAR(50) | NO | — | Unique human-readable order number |
| status | VARCHAR(30) | NO | 'pending' | pending, confirmed, processing, shipped, delivered, cancelled |
| subtotal | DECIMAL(12,2) | NO | — | Sum of line items before tax/shipping |
| tax_amount | DECIMAL(12,2) | YES | 0 | Tax |
| shipping_amount | DECIMAL(12,2) | YES | 0 | Shipping charge |
| total_amount | DECIMAL(12,2) | NO | — | Total charged |
| currency | VARCHAR(3) | NO | 'INR' | Currency |
| shipping_address_id | UUID | YES | NULL | FK → shipping_address.id |
| notes | TEXT | YES | NULL | Customer notes |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** order_number.  
**Relationships:** user N–1; shipping_address N–1; order 1–N order_item; payment references order_id.

---

### 9.4 Table: `order_item`

Line item in an order (product + quantity + price snapshot).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| order_id | UUID | NO | — | FK → order.id |
| product_id | UUID | NO | — | FK → product.id |
| quantity | INTEGER | NO | — | Quantity ordered |
| unit_price | DECIMAL(12,2) | NO | — | Price per unit at time of order |
| total_price | DECIMAL(12,2) | NO | — | quantity * unit_price |
| product_snapshot_json | JSONB | YES | NULL | Name, sku at order time |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |

**Relationships:** order N–1; product N–1.

---

### 9.5 Table: `shipping_address`

Saved shipping address; referenced by order.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| user_id | UUID | NO | — | FK → user.id |
| label | VARCHAR(100) | YES | NULL | Home, Office, etc. |
| address_line1 | VARCHAR(255) | NO | — | Street address |
| address_line2 | VARCHAR(255) | YES | NULL | Address line 2 |
| city | VARCHAR(100) | NO | — | City |
| state | VARCHAR(100) | YES | NULL | State |
| country | VARCHAR(100) | NO | — | Country |
| postal_code | VARCHAR(20) | YES | NULL | Postal code |
| phone | VARCHAR(20) | YES | NULL | Contact phone |
| is_default | BOOLEAN | NO | false | Default address for user |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** user N–1; shipping_address 1–N order.

---

### 9.6 Table: `sponsor`

Sponsor organization (brand). Linked to tournaments/teams via sponsorship_assignment.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| name | VARCHAR(255) | NO | — | Sponsor name |
| logo_url | VARCHAR(500) | YES | NULL | Logo image |
| website_url | VARCHAR(500) | YES | NULL | Website |
| contact_email | VARCHAR(255) | YES | NULL | Contact |
| contact_phone | VARCHAR(20) | YES | NULL | Contact |
| is_active | BOOLEAN | NO | true | Whether sponsor is active |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** sponsor 1–N sponsorship_assignment.

---

### 9.7 Table: `sponsorship_package`

Predefined packages (Gold, Silver, Bronze) with benefits. Optional; can be referenced by sponsorship_assignment.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| code | VARCHAR(50) | NO | — | Unique code |
| name | VARCHAR(100) | NO | — | Display name |
| description | TEXT | YES | NULL | Benefits description |
| sort_order | INTEGER | YES | 0 | Display order |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** code.

---

### 9.8 Table: `sponsorship_assignment`

Links a sponsor to a tournament and/or team with optional package and term.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| sponsor_id | UUID | NO | — | FK → sponsor.id |
| tournament_id | UUID | YES | NULL | FK → tournament.id |
| team_id | UUID | YES | NULL | FK → team.id |
| sponsorship_package_id | UUID | YES | NULL | FK → sponsorship_package.id |
| start_date | DATE | YES | NULL | Start of sponsorship |
| end_date | DATE | YES | NULL | End of sponsorship |
| amount | DECIMAL(12,2) | YES | NULL | Sponsorship amount |
| currency | VARCHAR(3) | YES | 'INR' | Currency |
| notes | TEXT | YES | NULL | Internal notes |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** sponsor N–1; tournament N–1; team N–1; sponsorship_package N–1. At least one of tournament_id or team_id typically set.

---

## 10. Domain: Community & Communication

**Role:** Posts, comments, reactions, follows, announcements, messages, and notifications. Supports social and engagement features.

**Relationships:** user 1–N post, comment, reaction, follow, notification; post 1–N comment, reaction; entity_media can link to post; follow references (entity_type, entity_id).

### 10.1 Table: `post`

User-generated content; can be linked to tournament, match, team, or player via entity_type/entity_id or dedicated FKs.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| user_id | UUID | NO | — | FK → user.id (author) |
| entity_type | VARCHAR(50) | YES | NULL | tournament, match, team, player |
| entity_id | UUID | YES | NULL | ID of linked entity |
| title | VARCHAR(255) | YES | NULL | Post title |
| body | TEXT | NO | — | Post content |
| visibility | VARCHAR(20) | NO | 'public' | public, followers_only, private |
| status | VARCHAR(20) | NO | 'published' | draft, published, archived |
| like_count | INTEGER | NO | 0 | Denormalized count |
| comment_count | INTEGER | NO | 0 | Denormalized count |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |
| deleted_at | TIMESTAMPTZ | YES | NULL | Soft delete |

**Relationships:** user N–1; post 1–N comment, reaction.

---

### 10.2 Table: `comment`

Comments on posts (or optionally on matches, etc. via entity_type/entity_id).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| post_id | UUID | NO | — | FK → post.id |
| user_id | UUID | NO | — | FK → user.id (author) |
| parent_id | UUID | YES | NULL | FK → comment.id (for replies) |
| body | TEXT | NO | — | Comment text |
| like_count | INTEGER | NO | 0 | Denormalized count |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |
| deleted_at | TIMESTAMPTZ | YES | NULL | Soft delete |

**Relationships:** post N–1; user N–1; comment self-reference (parent_id); comment 1–N reaction.

---

### 10.3 Table: `reaction`

Likes/reactions on posts or comments. Can be extended for reaction type (like, love, etc.).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| user_id | UUID | NO | — | FK → user.id |
| target_type | VARCHAR(30) | NO | — | post, comment |
| target_id | UUID | NO | — | post.id or comment.id |
| reaction_type | VARCHAR(20) | NO | 'like' | like, love, etc. |
| created_at | TIMESTAMPTZ | NO | now() | When reacted |

**Unique constraints:** (user_id, target_type, target_id).  
**Relationships:** user N–1.

---

### 10.4 Table: `follow`

User follows an entity (player, team, tournament). Used for feeds and notifications.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| follower_user_id | UUID | NO | — | FK → user.id (follower) |
| entity_type | VARCHAR(50) | NO | — | player, team, tournament |
| entity_id | UUID | NO | — | ID of followed entity |
| created_at | TIMESTAMPTZ | NO | now() | When follow started |

**Unique constraints:** (follower_user_id, entity_type, entity_id).  
**Relationships:** user N–1 (follower).

---

### 10.5 Table: `announcement`

System or organizer announcements; can be scoped to a tournament.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| tournament_id | UUID | YES | NULL | FK → tournament.id (NULL = platform-wide) |
| organizer_id | UUID | YES | NULL | FK → organizer.id |
| title | VARCHAR(255) | NO | — | Announcement title |
| body | TEXT | NO | — | Content |
| priority | VARCHAR(20) | NO | 'normal' | low, normal, high, urgent |
| published_at | TIMESTAMPTZ | YES | NULL | When published (NULL = draft) |
| expires_at | TIMESTAMPTZ | YES | NULL | When to stop showing |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |
| created_by | UUID | YES | NULL | FK → user.id |

**Relationships:** tournament N–1; organizer N–1; user N–1.

---

### 10.6 Table: `notification`

In-app (and optionally push) notifications for users.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| user_id | UUID | NO | — | FK → user.id (recipient) |
| type | VARCHAR(50) | NO | — | auction_started, team_assigned, match_reminder, etc. |
| title | VARCHAR(255) | YES | NULL | Notification title |
| body | TEXT | YES | NULL | Notification body |
| payload_json | JSONB | YES | NULL | Deep link / entity refs |
| read_at | TIMESTAMPTZ | YES | NULL | When read (NULL = unread) |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |

**Relationships:** user N–1. Index on (user_id, read_at) for inbox.

---

### 10.7 Table: `notification_preference`

Per-user preferences for notification channels and types (email, push, in_app).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| user_id | UUID | NO | — | FK → user.id (UNIQUE) |
| email_enabled | BOOLEAN | NO | true | Email notifications |
| push_enabled | BOOLEAN | NO | true | Push notifications |
| in_app_enabled | BOOLEAN | NO | true | In-app notifications |
| preferences_json | JSONB | YES | NULL | Per-type toggles (e.g. auction_reminder: true) |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Unique constraints:** user_id.  
**Relationships:** user 1–1.

---

### 10.8 Table: `message` (optional)

In-app direct or group messages. Simplified schema; can be extended for threads and channels.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| sender_id | UUID | NO | — | FK → user.id |
| recipient_id | UUID | YES | NULL | FK → user.id (for DM) |
| conversation_id | UUID | YES | NULL | For group chats |
| body | TEXT | NO | — | Message content |
| read_at | TIMESTAMPTZ | YES | NULL | When recipient read |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |

**Relationships:** user N–1 (sender, recipient).

---

## 11. Domain: System & Integration

**Role:** API clients, webhooks, and audit logs for security and integrations.

**Relationships:** api_client 1–N api_key; webhook 1–N webhook_event; user 1–N audit_log (as actor).

### 11.1 Table: `api_client`

Third-party or internal API consumers. Holds name and status; keys in api_key.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| name | VARCHAR(255) | NO | — | Client name |
| description | TEXT | YES | NULL | Description |
| status | VARCHAR(20) | NO | 'active' | active, inactive, revoked |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** api_client 1–N api_key.

---

### 11.2 Table: `api_key`

API keys for api_client. Store hashed key; show prefix only in UI.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| api_client_id | UUID | NO | — | FK → api_client.id |
| key_prefix | VARCHAR(20) | NO | — | First N chars of key (for display) |
| key_hash | VARCHAR(255) | NO | — | Hashed full key |
| scopes_json | JSONB | YES | NULL | Allowed scopes (e.g. read:auction) |
| expires_at | TIMESTAMPTZ | YES | NULL | Key expiry |
| last_used_at | TIMESTAMPTZ | YES | NULL | Last use time |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |

**Relationships:** api_client N–1.

---

### 11.3 Table: `webhook`

Outbound webhook configurations (e.g. on auction completed). Called by backend when events occur.

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| api_client_id | UUID | YES | NULL | FK → api_client.id |
| organizer_id | UUID | YES | NULL | FK → organizer.id |
| url | VARCHAR(500) | NO | — | Webhook URL |
| secret | VARCHAR(255) | YES | NULL | Signing secret |
| events_json | JSONB | YES | NULL | Event types to send (e.g. auction.completed) |
| status | VARCHAR(20) | NO | 'active' | active, inactive |
| created_at | TIMESTAMPTZ | NO | now() | Record creation time |
| updated_at | TIMESTAMPTZ | NO | now() | Last update time |

**Relationships:** api_client N–1; organizer N–1; webhook 1–N webhook_event.

---

### 11.4 Table: `webhook_event`

Delivery log for each webhook invocation (success/failure, response code).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| webhook_id | UUID | NO | — | FK → webhook.id |
| event_type | VARCHAR(50) | NO | — | Event that triggered |
| payload_json | JSONB | YES | NULL | Payload sent |
| response_status | INTEGER | YES | NULL | HTTP status from recipient |
| response_body | TEXT | YES | NULL | Response body (truncated if large) |
| failure_reason | TEXT | YES | NULL | Error message if failed |
| created_at | TIMESTAMPTZ | NO | now() | When delivery was attempted |

**Relationships:** webhook N–1.

---

### 11.5 Table: `audit_log`

System-wide audit trail for sensitive actions (create/update/delete on important entities).

| Field Name | Data Type | Nullable | Default | Description |
|------------|-----------|----------|---------|-------------|
| id | UUID | NO | gen_random_uuid() | Primary key |
| entity_type | VARCHAR(50) | NO | — | user, tournament, auction, bid, etc. |
| entity_id | UUID | NO | — | ID of affected entity |
| action | VARCHAR(20) | NO | — | create, update, delete |
| actor_user_id | UUID | YES | NULL | FK → user.id (who did it) |
| before_json | JSONB | YES | NULL | State before (for update/delete) |
| after_json | JSONB | YES | NULL | State after (for create/update) |
| ip_address | INET | YES | NULL | Client IP |
| user_agent | VARCHAR(500) | YES | NULL | Client user agent |
| created_at | TIMESTAMPTZ | NO | now() | When action occurred |

**Relationships:** user N–1 (actor_user_id). Index on (entity_type, entity_id) and (actor_user_id, created_at).

---

## 12. Entity Relationship Summary

- **user** → has many **user_role** → **role**; optional **permission** via **role_permission**.
- **user** → 0/1 **organizer**; **organizer** → many **tournament**, **subscription**; **subscription** → **subscription_plan**; **payment** can link to **subscription** or **order**.
- **tournament** → **organizer**, **sport**, **competition_level**, **age_category**; **tournament** → many **team**, **tournament_player_registration**, **match**; 0/1 **auction**.
- **team** → **tournament**, **user** (owner); **team** → many **team_membership** ↔ **player**; **team** → **auction_team**, **match_participant**.
- **player** → **user** (optional); **player** → many **tournament_player_registration**, **auction_player**, **team_membership**, **player_match_stat**.
- **auction** → **tournament**, **organizer**; **auction** → **auction_team** (→ team), **auction_player** (→ player), **bid**, **auction_icon_player**, **auction_report**.
- **match** → **tournament**, **sport**, **venue**, **tournament_stage**; **match** → **match_participant** (→ team), **match_event**, **player_match_stat**, **team_match_stat**.
- **media_asset** → **entity_media** (polymorphic entity_type/entity_id); **live_stream** → match or tournament.
- **order** → **user**; **order** → **order_item** → **product** → **product_category**; **payment** → **order**; **sponsorship_assignment** → **sponsor**, **tournament**/ **team**.
- **post** → **user**; **comment** → **post**, **user**; **reaction** → user + target; **follow** → user + entity; **notification** → **user**; **notification_preference** → **user**.
- **api_client** → **api_key**; **webhook** → **webhook_event**; **audit_log** → **user** (actor).

---

## 13. Glossary

| Term | Definition |
|------|-------------|
| **Auction (points-based)** | Bidding in points only; each team gets equal points; no real money in bids. |
| **Icon player** | Pre-assigned to a team before auction; does not consume points or go through bidding. |
| **Organizer-managed payment** | Player registration fees collected offline by organizer; platform only stores payment_status (paid/unpaid). |
| **Base price** | Minimum bid (in points) for a player in auction. |
| **team_membership** | Row linking a player to a team for a tournament; created after auction (sold + icon) or manually. |
| **tournament_player_registration** | Player’s registration for one tournament; approval and payment status determine auction eligibility. |
| **RLS** | Row-Level Security (PostgreSQL feature for multi-tenant data isolation). |
| **Soft delete** | Row kept with `deleted_at` set instead of physical delete. |

---

**End of document.** For implementation, generate migrations from this schema; add indexes on foreign keys and frequently filtered columns (e.g. status, organizer_id, tournament_id, created_at) as needed.
