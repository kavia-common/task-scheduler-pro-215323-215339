# Task Scheduler Pro — PostgreSQL Schema

This document describes the database schema created for the Task Scheduler Pro app.

Connection command (source of truth): `db_connection.txt`

## Extensions

- `pgcrypto` (for `gen_random_uuid()`)

## Tables

### `users`
Holds user identity and basic account state.

Columns:
- `id` (uuid, PK, default `gen_random_uuid()`)
- `email` (text, required)
- `password_hash` (text, required)
- `full_name` (text, optional)
- `is_active` (boolean, default true)
- `created_at` (timestamptz, default now)
- `updated_at` (timestamptz, default now; maintained by trigger)

Constraints / Indexes:
- Unique **case-insensitive** email via unique index:
  - `ux_users_email_lower` on `lower(email)`
- `idx_users_created_at` on `created_at DESC`

### `user_sessions`
Stores refresh sessions / login sessions for revocation and expiry tracking.

Columns:
- `id` (uuid, PK)
- `user_id` (uuid, FK → `users.id`, ON DELETE CASCADE)
- `refresh_token_hash` (text, required)
- `created_at` (timestamptz, default now)
- `expires_at` (timestamptz, required)
- `revoked_at` (timestamptz, nullable)

Constraints / Indexes:
- `ux_user_sessions_refresh_token_hash` unique on `refresh_token_hash`
- `idx_user_sessions_user_id` on `user_id`

### `tasks`
Core task entity with deadlines, priority, and status.

Columns:
- `id` (uuid, PK)
- `user_id` (uuid, FK → `users.id`, ON DELETE CASCADE)
- `title` (text, required)
- `description` (text, optional)
- `status` (text, default `todo`)
- `priority` (smallint, default 3, 1..5)
- `due_at` (timestamptz, nullable)
- `is_all_day` (boolean, default false)
- `completed_at` (timestamptz, nullable)
- `created_at` (timestamptz, default now)
- `updated_at` (timestamptz, default now; maintained by trigger)

Checks:
- `status` ∈ `('todo','in_progress','done','cancelled')`
- `priority` between 1 and 5
- completion consistency:
  - if `status='done'` then `completed_at IS NOT NULL`
  - else `completed_at IS NULL`

Indexes:
- `idx_tasks_user_status_due` on `(user_id, status, due_at)`
- `idx_tasks_due_at` on `due_at` where `due_at IS NOT NULL`
- `idx_tasks_priority` on `(user_id, priority)`

### `notification_rules`
Defines notification/scheduling rules (relative-to-due or cron-based).

Columns:
- `id` (uuid, PK)
- `user_id` (uuid, FK → `users.id`, ON DELETE CASCADE)
- `task_id` (uuid, FK → `tasks.id`, ON DELETE CASCADE, nullable for user-wide rules)
- `channel` (text, default `in_app`)
- `minutes_before_due` (int, default 60)
- `schedule_type` (text, default `relative_to_due`)
- `cron` (text, nullable; required when `schedule_type='cron'`)
- `is_enabled` (boolean, default true)
- `last_scheduled_at` (timestamptz, nullable)
- `created_at` (timestamptz, default now)
- `updated_at` (timestamptz, default now; maintained by trigger)

Checks:
- `channel` ∈ `('in_app','email','sms','webhook')`
- `schedule_type` ∈ `('relative_to_due','cron')`
- for `relative_to_due`: `minutes_before_due >= 0` and `cron IS NULL`
- for `cron`: `cron IS NOT NULL`

Indexes:
- `idx_notification_rules_user_enabled` on `(user_id, is_enabled)`
- `idx_notification_rules_task_id` on `task_id` where `task_id IS NOT NULL`
- `idx_notification_rules_schedule_type` on `schedule_type`

### `notification_deliveries`
Concrete scheduled delivery instances for auditing and scheduler processing.

Columns:
- `id` (uuid, PK)
- `rule_id` (uuid, FK → `notification_rules.id`, ON DELETE CASCADE)
- `task_id` (uuid, FK → `tasks.id`, ON DELETE SET NULL)
- `due_at` (timestamptz, nullable snapshot)
- `scheduled_for` (timestamptz, required)
- `sent_at` (timestamptz, nullable)
- `status` (text, default `scheduled`)
- `provider_message_id` (text, nullable)
- `error` (text, nullable)
- `created_at` (timestamptz, default now)

Checks:
- `status` ∈ `('scheduled','sent','failed','cancelled')`

Indexes:
- `idx_notification_deliveries_rule_status` on `(rule_id, status)`
- `idx_notification_deliveries_scheduled_for` on `scheduled_for` where `status='scheduled'`
- `idx_notification_deliveries_task_id` on `task_id` where `task_id IS NOT NULL`

## Triggers

A shared trigger function keeps `updated_at` current on updates:

- `set_updated_at()` function
- Triggers:
  - `trg_users_updated_at` on `users`
  - `trg_tasks_updated_at` on `tasks`
  - `trg_notification_rules_updated_at` on `notification_rules`

## Minimal seed data (for development)

- A demo user:
  - `demo@taskschedulerpro.local` with `password_hash='CHANGE_ME'`
- One sample task for the demo user
- One `notification_rules` entry (in_app, 60 minutes before due)

Note: The backend should replace `CHANGE_ME` with a real password hash when implementing auth flows.
