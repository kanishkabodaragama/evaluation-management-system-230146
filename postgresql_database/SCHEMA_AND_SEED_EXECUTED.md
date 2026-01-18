# PostgreSQL schema + minimal seed data (executed)

This document captures the **exact logical schema** and **seed data** applied to the `postgresql_database` container for local testing.

Connection (from `db_connection.txt`):
- `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

> Note: This repo’s DB container guidance says to execute SQL **one statement at a time** via `psql -c "..."`. The statements below are presented that way (sometimes grouped with `;` as they were executed for triggers).  
> Tables include `created_at`/`updated_at` timestamps and `status` fields where relevant, plus constraints and analytics-friendly composite indexes.

---

## 1) Extensions / shared function

### 1.1 pgcrypto (UUID generation)
```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

### 1.2 updated_at trigger function
```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger
LANGUAGE plpgsql
AS $$ 
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$;
```

---

## 2) Tables

### 2.1 roles
```sql
CREATE TABLE IF NOT EXISTS roles (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL UNIQUE,
  description text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
```

Trigger:
```sql
DROP TRIGGER IF EXISTS trg_roles_updated_at ON roles;
CREATE TRIGGER trg_roles_updated_at
BEFORE UPDATE ON roles
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

### 2.2 users
Stores app user profile and role. References Supabase auth via `auth_user_id` (UUID).  
(For this local Postgres container, we do not enforce FK to `auth.users`.)

```sql
CREATE TABLE IF NOT EXISTS users (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  auth_user_id uuid UNIQUE,
  email text UNIQUE,
  display_name text,
  role_id uuid REFERENCES roles(id) ON UPDATE CASCADE ON DELETE SET NULL,
  status text NOT NULL DEFAULT 'active' CHECK (status IN ('active','disabled')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
```

Indexes:
```sql
CREATE INDEX IF NOT EXISTS idx_users_role_id ON users(role_id);
```

Trigger:
```sql
DROP TRIGGER IF EXISTS trg_users_updated_at ON users;
CREATE TRIGGER trg_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

### 2.3 employees
```sql
CREATE TABLE IF NOT EXISTS employees (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  employee_code text UNIQUE,
  full_name text NOT NULL,
  email text UNIQUE,
  team text,
  title text,
  status text NOT NULL DEFAULT 'active' CHECK (status IN ('active','inactive')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
```

Trigger:
```sql
DROP TRIGGER IF EXISTS trg_employees_updated_at ON employees;
CREATE TRIGGER trg_employees_updated_at
BEFORE UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

### 2.4 criteria
```sql
CREATE TABLE IF NOT EXISTS criteria (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL UNIQUE,
  description text,
  weight numeric(6,3) NOT NULL DEFAULT 1.0 CHECK (weight > 0),
  sort_order int NOT NULL DEFAULT 0,
  is_active boolean NOT NULL DEFAULT true,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
```

Indexes:
```sql
CREATE INDEX IF NOT EXISTS idx_criteria_active_sort ON criteria(is_active, sort_order);
```

Trigger:
```sql
DROP TRIGGER IF EXISTS trg_criteria_updated_at ON criteria;
CREATE TRIGGER trg_criteria_updated_at
BEFORE UPDATE ON criteria
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

### 2.5 review_sessions
```sql
CREATE TABLE IF NOT EXISTS review_sessions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  description text,
  start_date date,
  end_date date,
  status text NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','active','closed','archived')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT review_sessions_dates_chk CHECK (
    end_date IS NULL OR start_date IS NULL OR end_date >= start_date
  )
);
```

Indexes:
```sql
CREATE INDEX IF NOT EXISTS idx_review_sessions_status ON review_sessions(status);
```

Trigger:
```sql
DROP TRIGGER IF EXISTS trg_review_sessions_updated_at ON review_sessions;
CREATE TRIGGER trg_review_sessions_updated_at
BEFORE UPDATE ON review_sessions
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

### 2.6 session_assignments
Join table mapping `reviewer_user_id ↔ employee_id` within a `session_id`.
```sql
CREATE TABLE IF NOT EXISTS session_assignments (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id uuid NOT NULL REFERENCES review_sessions(id) ON UPDATE CASCADE ON DELETE CASCADE,
  employee_id uuid NOT NULL REFERENCES employees(id) ON UPDATE CASCADE ON DELETE CASCADE,
  reviewer_user_id uuid NOT NULL REFERENCES users(id) ON UPDATE CASCADE ON DELETE CASCADE,
  status text NOT NULL DEFAULT 'assigned'
    CHECK (status IN ('assigned','in_progress','completed','cancelled')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT session_assignments_unique UNIQUE (session_id, employee_id, reviewer_user_id)
);
```

Indexes (analytics helpers):
```sql
CREATE INDEX IF NOT EXISTS idx_session_assignments_session_employee
  ON session_assignments(session_id, employee_id);

CREATE INDEX IF NOT EXISTS idx_session_assignments_session_reviewer
  ON session_assignments(session_id, reviewer_user_id);
```

Trigger:
```sql
DROP TRIGGER IF EXISTS trg_session_assignments_updated_at ON session_assignments;
CREATE TRIGGER trg_session_assignments_updated_at
BEFORE UPDATE ON session_assignments
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

### 2.7 submissions
One submission per `(session_id, employee_id, reviewer_user_id)` as requested.
```sql
CREATE TABLE IF NOT EXISTS submissions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id uuid NOT NULL REFERENCES review_sessions(id) ON UPDATE CASCADE ON DELETE CASCADE,
  employee_id uuid NOT NULL REFERENCES employees(id) ON UPDATE CASCADE ON DELETE CASCADE,
  reviewer_user_id uuid NOT NULL REFERENCES users(id) ON UPDATE CASCADE ON DELETE CASCADE,
  assignment_id uuid REFERENCES session_assignments(id) ON UPDATE CASCADE ON DELETE SET NULL,
  status text NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','submitted','reopened')),
  overall_comment text,
  submitted_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT submissions_one_per_pair UNIQUE (session_id, employee_id, reviewer_user_id)
);
```

Indexes (analytics helpers):
```sql
CREATE INDEX IF NOT EXISTS idx_submissions_session_employee
  ON submissions(session_id, employee_id);

CREATE INDEX IF NOT EXISTS idx_submissions_session_reviewer
  ON submissions(session_id, reviewer_user_id);
```

Trigger:
```sql
DROP TRIGGER IF EXISTS trg_submissions_updated_at ON submissions;
CREATE TRIGGER trg_submissions_updated_at
BEFORE UPDATE ON submissions
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

### 2.8 scores
```sql
CREATE TABLE IF NOT EXISTS scores (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  submission_id uuid NOT NULL REFERENCES submissions(id) ON UPDATE CASCADE ON DELETE CASCADE,
  criterion_id uuid NOT NULL REFERENCES criteria(id) ON UPDATE CASCADE ON DELETE CASCADE,
  score_value numeric(5,2) NOT NULL CHECK (score_value >= 0 AND score_value <= 10),
  comment text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT scores_one_per_criterion UNIQUE (submission_id, criterion_id)
);
```

Index:
```sql
CREATE INDEX IF NOT EXISTS idx_scores_submission_id ON scores(submission_id);
```

Trigger:
```sql
DROP TRIGGER IF EXISTS trg_scores_updated_at ON scores;
CREATE TRIGGER trg_scores_updated_at
BEFORE UPDATE ON scores
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

---

## 3) Seed data (minimal, deterministic IDs)

### 3.1 Roles
```sql
INSERT INTO roles (id, name, description)
VALUES ('00000000-0000-0000-0000-000000000001','admin','Administrator')
ON CONFLICT (name) DO UPDATE SET description = EXCLUDED.description;

INSERT INTO roles (id, name, description)
VALUES ('00000000-0000-0000-0000-000000000002','reviewer','Reviewer')
ON CONFLICT (name) DO UPDATE SET description = EXCLUDED.description;
```

### 3.2 Users (local testing profiles)
These are **not** tied to Supabase auth, but the schema supports `auth_user_id` for that linkage.
```sql
INSERT INTO users (id, email, display_name, role_id)
VALUES ('11111111-1111-1111-1111-111111111111','admin@example.com','Admin User','00000000-0000-0000-0000-000000000001')
ON CONFLICT (email) DO UPDATE SET display_name = EXCLUDED.display_name, role_id = EXCLUDED.role_id;

INSERT INTO users (id, email, display_name, role_id)
VALUES ('22222222-2222-2222-2222-222222222222','reviewer1@example.com','Reviewer One','00000000-0000-0000-0000-000000000002')
ON CONFLICT (email) DO UPDATE SET display_name = EXCLUDED.display_name, role_id = EXCLUDED.role_id;

INSERT INTO users (id, email, display_name, role_id)
VALUES ('33333333-3333-3333-3333-333333333333','reviewer2@example.com','Reviewer Two','00000000-0000-0000-0000-000000000002')
ON CONFLICT (email) DO UPDATE SET display_name = EXCLUDED.display_name, role_id = EXCLUDED.role_id;
```

### 3.3 Employees (3)
```sql
INSERT INTO employees (id, employee_code, full_name, email, team, title)
VALUES ('aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaa1','E001','Alice Johnson','alice@example.com','Engineering','Intern')
ON CONFLICT (employee_code) DO UPDATE SET
  full_name=EXCLUDED.full_name, email=EXCLUDED.email, team=EXCLUDED.team, title=EXCLUDED.title;

INSERT INTO employees (id, employee_code, full_name, email, team, title)
VALUES ('aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaa2','E002','Bob Smith','bob@example.com','Engineering','Junior Developer')
ON CONFLICT (employee_code) DO UPDATE SET
  full_name=EXCLUDED.full_name, email=EXCLUDED.email, team=EXCLUDED.team, title=EXCLUDED.title;

INSERT INTO employees (id, employee_code, full_name, email, team, title)
VALUES ('aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaa3','E003','Carol Lee','carol@example.com','Product','Product Intern')
ON CONFLICT (employee_code) DO UPDATE SET
  full_name=EXCLUDED.full_name, email=EXCLUDED.email, team=EXCLUDED.team, title=EXCLUDED.title;
```

### 3.4 Criteria (4)
```sql
INSERT INTO criteria (id, name, description, weight, sort_order)
VALUES ('bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbb1','Technical Skills','Quality of technical work and learning progression',1.0,10)
ON CONFLICT (name) DO UPDATE SET
  description=EXCLUDED.description, weight=EXCLUDED.weight, sort_order=EXCLUDED.sort_order, is_active=true;

INSERT INTO criteria (id, name, description, weight, sort_order)
VALUES ('bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbb2','Communication','Clarity, responsiveness, and collaboration',1.0,20)
ON CONFLICT (name) DO UPDATE SET
  description=EXCLUDED.description, weight=EXCLUDED.weight, sort_order=EXCLUDED.sort_order, is_active=true;

INSERT INTO criteria (id, name, description, weight, sort_order)
VALUES ('bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbb3','Ownership','Takes initiative and delivers reliably',1.0,30)
ON CONFLICT (name) DO UPDATE SET
  description=EXCLUDED.description, weight=EXCLUDED.weight, sort_order=EXCLUDED.sort_order, is_active=true;

INSERT INTO criteria (id, name, description, weight, sort_order)
VALUES ('bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbb4','Growth Mindset','Accepts feedback and improves over time',1.0,40)
ON CONFLICT (name) DO UPDATE SET
  description=EXCLUDED.description, weight=EXCLUDED.weight, sort_order=EXCLUDED.sort_order, is_active=true;
```

### 3.5 Review session (1 active)
```sql
INSERT INTO review_sessions (id, name, description, start_date, end_date, status)
VALUES ('cccccccc-cccc-cccc-cccc-ccccccccccc1','Q1 Internship Review','Sample active session for local testing',CURRENT_DATE - 7, CURRENT_DATE + 7,'active')
ON CONFLICT (id) DO UPDATE SET
  name=EXCLUDED.name, description=EXCLUDED.description, start_date=EXCLUDED.start_date, end_date=EXCLUDED.end_date, status=EXCLUDED.status;
```

### 3.6 Session assignments (3)
```sql
INSERT INTO session_assignments (id, session_id, employee_id, reviewer_user_id, status)
VALUES ('dddddddd-dddd-dddd-dddd-ddddddddddd1','cccccccc-cccc-cccc-cccc-ccccccccccc1','aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaa1','22222222-2222-2222-2222-222222222222','assigned')
ON CONFLICT (session_id, employee_id, reviewer_user_id) DO UPDATE SET status=EXCLUDED.status;

INSERT INTO session_assignments (id, session_id, employee_id, reviewer_user_id, status)
VALUES ('dddddddd-dddd-dddd-dddd-ddddddddddd2','cccccccc-cccc-cccc-cccc-ccccccccccc1','aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaa2','22222222-2222-2222-2222-222222222222','assigned')
ON CONFLICT (session_id, employee_id, reviewer_user_id) DO UPDATE SET status=EXCLUDED.status;

INSERT INTO session_assignments (id, session_id, employee_id, reviewer_user_id, status)
VALUES ('dddddddd-dddd-dddd-dddd-ddddddddddd3','cccccccc-cccc-cccc-cccc-ccccccccccc1','aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaa3','33333333-3333-3333-3333-333333333333','assigned')
ON CONFLICT (session_id, employee_id, reviewer_user_id) DO UPDATE SET status=EXCLUDED.status;
```

### 3.7 Optional sample submission + scores
```sql
INSERT INTO submissions (id, session_id, employee_id, reviewer_user_id, assignment_id, status, overall_comment, submitted_at)
VALUES ('eeeeeeee-eeee-eeee-eeee-eeeeeeeeeee1','cccccccc-cccc-cccc-cccc-ccccccccccc1','aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaa1','22222222-2222-2222-2222-222222222222','dddddddd-dddd-dddd-dddd-ddddddddddd1','submitted','Strong progress overall.', now())
ON CONFLICT (session_id, employee_id, reviewer_user_id) DO UPDATE SET
  status=EXCLUDED.status, overall_comment=EXCLUDED.overall_comment, submitted_at=EXCLUDED.submitted_at;

INSERT INTO scores (id, submission_id, criterion_id, score_value, comment)
VALUES ('ffffffff-ffff-ffff-ffff-fffffffffff1','eeeeeeee-eeee-eeee-eeee-eeeeeeeeeee1','bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbb1',8.5,'Good fundamentals')
ON CONFLICT (submission_id, criterion_id) DO UPDATE SET score_value=EXCLUDED.score_value, comment=EXCLUDED.comment;

INSERT INTO scores (id, submission_id, criterion_id, score_value, comment)
VALUES ('ffffffff-ffff-ffff-ffff-fffffffffff2','eeeeeeee-eeee-eeee-eeee-eeeeeeeeeee1','bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbb2',9.0,'Clear communication')
ON CONFLICT (submission_id, criterion_id) DO UPDATE SET score_value=EXCLUDED.score_value, comment=EXCLUDED.comment;
```

---

## 4) Notes for integration

- The `users.auth_user_id` column is designed to store the Supabase Auth user UUID.
- In production with Supabase Postgres, you can optionally add a FK to `auth.users(id)` if desired; for this standalone container it’s kept as a plain UUID to avoid dependency on Supabase’s internal schemas.
- Analytics-friendly composite indexes exist on:
  - `session_assignments(session_id, employee_id)`
  - `session_assignments(session_id, reviewer_user_id)`
  - `submissions(session_id, employee_id)`
  - `submissions(session_id, reviewer_user_id)`

---
