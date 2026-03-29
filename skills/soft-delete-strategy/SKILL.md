---
name: soft-delete-strategy
description: 'Reversible deletion patterns for SaaS records — soft delete, undelete, restore, and hard purge. Use when: soft delete, soft deletion, reversible deletion, undelete, restore deleted, deleted_at column, is_deleted flag, soft delete pattern, soft delete Go, soft delete PostgreSQL, trash, recycle bin, deleted record recovery, paranoid delete, hard delete, purge deleted, soft delete index, soft delete query filter, soft delete cascade, GDPR soft delete, soft delete ORM, soft delete exclusion.'
argument-hint: 'Describe which entities need soft delete (e.g., projects, documents, users), whether you need a restore/undo window (and how long), and any compliance constraints that require eventual hard purge (GDPR, data minimization).'
---

# Soft Delete Strategy

## When to Use

Invoke this skill when you need to:
- Delete domain records reversibly — support undo, archive, or trash UI patterns
- Exclude soft-deleted records from all queries without repeating `WHERE deleted_at IS NULL`
- Restore (undelete) individual records within a configurable recovery window
- Hard-purge soft-deleted records after a retention period for compliance and storage hygiene
- Handle cascading soft deletes across related entities
- Ensure unique constraints still work correctly alongside soft-deleted rows

---

## Soft Delete vs Hard Delete — Decision Table

| Scenario | Approach | Reason |
|---|---|---|
| User deletes a document | Soft delete | Undo, audit trail, recovery |
| Admin removes a team member | Hard delete of membership (user account soft-deleted) | Membership row holds no unique user data |
| GDPR erasure request | Hard delete after purge window | Legal obligation |
| Tenant cancellation | `pending_deletion` → hard purge after retention | Recovery window + compliance |
| Billing record pruning | Hard delete after statutory retention window | Legal hold may apply |
| Access token revoke | Hard delete or status column | No undo needed; security-sensitive |

---

## Step 1 — Schema Pattern

```sql
-- Pattern: add deleted_at to any table that needs soft delete
ALTER TABLE documents ADD COLUMN deleted_at TIMESTAMPTZ;
ALTER TABLE projects  ADD COLUMN deleted_at TIMESTAMPTZ;

-- PARTIAL INDEX: active records are the hot path — only index non-deleted rows
CREATE UNIQUE INDEX idx_documents_slug_active
    ON documents(tenant_id, slug)
    WHERE deleted_at IS NULL;

-- Index for purge job: find rows ready for hard purge
CREATE INDEX idx_documents_deleted_at
    ON documents(deleted_at)
    WHERE deleted_at IS NOT NULL;

-- Optional: separate archive table for long-term storage without polluting the main table
CREATE TABLE documents_archive (LIKE documents INCLUDING ALL);
```

**Why `deleted_at` over `is_deleted` boolean:**
- `deleted_at` records WHEN deletion happened — useful for retention enforcement and audit
- Boolean columns cannot carry temporal information for purge scheduling
- `deleted_at IS NULL` indexes are as fast as a boolean index

---

## Step 2 — Repository Pattern

Enforce the `deleted_at IS NULL` filter at the repository boundary — not in every service call.

```go
type DocumentRepo interface {
    GetByID(ctx context.Context, tenantID, id uuid.UUID) (*Document, error)         // excludes deleted
    GetByIDIncludeDeleted(ctx context.Context, tenantID, id uuid.UUID) (*Document, error) // for restore/admin
    List(ctx context.Context, tenantID uuid.UUID, filter ListFilter) ([]*Document, error)  // excludes deleted
    SoftDelete(ctx context.Context, tenantID, id uuid.UUID, deletedBy uuid.UUID) error
    Restore(ctx context.Context, tenantID, id uuid.UUID, restoredBy uuid.UUID) error
    HardDelete(ctx context.Context, tenantID, id uuid.UUID) error
    ListDueForPurge(ctx context.Context, olderThan time.Time) ([]*Document, error)
}

// Default Get ALWAYS excludes deleted rows
func (r *documentRepo) GetByID(ctx context.Context, tenantID, id uuid.UUID) (*Document, error) {
    var doc Document
    err := r.db.QueryRow(ctx,
        `SELECT * FROM documents WHERE id = $1 AND tenant_id = $2 AND deleted_at IS NULL`,
        id, tenantID,
    ).Scan(&doc)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, ErrNotFound
    }
    return &doc, err
}

func (r *documentRepo) SoftDelete(ctx context.Context, tenantID, id uuid.UUID, deletedBy uuid.UUID) error {
    res, err := r.db.Exec(ctx,
        `UPDATE documents SET deleted_at = now(), deleted_by = $3
         WHERE id = $1 AND tenant_id = $2 AND deleted_at IS NULL`,
        id, tenantID, deletedBy,
    )
    if rowsAffected(res) == 0 {
        return ErrNotFound // already deleted or not found
    }
    return err
}

func (r *documentRepo) Restore(ctx context.Context, tenantID, id uuid.UUID, restoredBy uuid.UUID) error {
    // Check slug uniqueness before restore — a new doc with the same slug may have been created
    conflicts, err := r.checkSlugConflict(ctx, tenantID, id)
    if err != nil {
        return err
    }
    if conflicts {
        return ErrRestoreConflict
    }

    res, err := r.db.Exec(ctx,
        `UPDATE documents SET deleted_at = NULL, deleted_by = NULL, restored_at = now(), restored_by = $3
         WHERE id = $1 AND tenant_id = $2 AND deleted_at IS NOT NULL`,
        id, tenantID, restoredBy,
    )
    if rowsAffected(res) == 0 {
        return ErrNotRestoreable
    }
    return err
}
```

---

## Step 3 — Unique Constraint Handling

Soft delete breaks naive unique constraints. Use partial indexes:

```sql
-- WRONG: this unique index blocks re-creating a same-name document after soft delete
CREATE UNIQUE INDEX idx_documents_name ON documents(tenant_id, name);

-- CORRECT: partial index only covers active (non-deleted) rows
CREATE UNIQUE INDEX idx_documents_name_active ON documents(tenant_id, name)
    WHERE deleted_at IS NULL;
```

For slug re-use after deletion (e.g., user deletes a project named "alpha" and creates a new one with the same name):

```sql
-- Strategy 1: rename slug on soft delete (append _deleted_{id suffix})
UPDATE documents SET slug = slug || '_deleted_' || LEFT(id::text, 8), deleted_at = now()
WHERE id = $1;

-- Strategy 2: use partial unique index (preferred — no slug mutation needed)
-- (see above)
```

---

## Step 4 — Cascading Soft Delete

When deleting a parent record, soft-delete children in the same transaction:

```go
func (s *projectService) SoftDelete(ctx context.Context, tenantID, projectID uuid.UUID, deletedBy uuid.UUID) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        // Soft-delete the parent
        if err := s.projectRepo.SoftDelete(ctx, tx, tenantID, projectID, deletedBy); err != nil {
            return fmt.Errorf("delete project: %w", err)
        }

        // Cascade: soft-delete all documents in this project
        if err := s.documentRepo.SoftDeleteByProject(ctx, tx, tenantID, projectID, deletedBy); err != nil {
            return fmt.Errorf("cascade delete documents: %w", err)
        }

        // Cascade: soft-delete all comments on those documents
        if err := s.commentRepo.SoftDeleteByProject(ctx, tx, tenantID, projectID, deletedBy); err != nil {
            return fmt.Errorf("cascade delete comments: %w", err)
        }

        return nil
    })
}

// Cascade restore: restore children when parent is restored
func (s *projectService) Restore(ctx context.Context, tenantID, projectID uuid.UUID, restoredBy uuid.UUID) error {
    return s.db.Transaction(ctx, func(tx DB) error {
        if err := s.projectRepo.Restore(ctx, tx, tenantID, projectID, restoredBy); err != nil {
            return err
        }
        // Only restore children deleted at the same time as the parent (not independently deleted ones)
        if err := s.documentRepo.RestoreByProject(ctx, tx, tenantID, projectID); err != nil {
            return fmt.Errorf("cascade restore documents: %w", err)
        }
        return nil
    })
}
```

---

## Step 5 — Hard Purge Job

```go
const SoftDeleteRetentionDays = 90 // configurable; GDPR: minimize unnecessary retention

// Run daily — hard-delete soft-deleted rows older than retention period
func (s *purgeSvc) PurgeSoftDeleted(ctx context.Context) error {
    cutoff := time.Now().UTC().AddDate(0, 0, -SoftDeleteRetentionDays)

    // Each entity type has its own purge — cascading in FK order
    entities := []struct {
        name  string
        purge func(ctx context.Context, olderThan time.Time) (int64, error)
    }{
        {"comments", s.commentRepo.HardDeleteOlderThan},
        {"documents", s.documentRepo.HardDeleteOlderThan},
        {"projects",  s.projectRepo.HardDeleteOlderThan},
    }

    for _, e := range entities {
        count, err := e.purge(ctx, cutoff)
        if err != nil {
            slog.Error("purge failed", "entity", e.name, "err", err)
            continue
        }
        slog.Info("purged records", "entity", e.name, "count", count)
    }
    return nil
}
```

---

## Schema Additions Checklist

When adding `deleted_at` to a new table:

```sql
-- 1. Add columns
ALTER TABLE <table> ADD COLUMN deleted_at  TIMESTAMPTZ;
ALTER TABLE <table> ADD COLUMN deleted_by  UUID REFERENCES users(id);
ALTER TABLE <table> ADD COLUMN restored_at TIMESTAMPTZ;
ALTER TABLE <table> ADD COLUMN restored_by UUID REFERENCES users(id);

-- 2. Create partial unique indexes to replace existing unique constraints
-- (drop old unique index, add WHERE deleted_at IS NULL version)

-- 3. Create index for purge job
CREATE INDEX idx_<table>_deleted_at ON <table>(deleted_at) WHERE deleted_at IS NOT NULL;
```

---

## Quality Checks

- [ ] All default `GetByID` and `List` queries include `AND deleted_at IS NULL` — never rely on the caller
- [ ] Unique constraints use partial indexes `WHERE deleted_at IS NULL` — no false conflicts
- [ ] `SoftDelete` checks `deleted_at IS NULL` before updating — returns `ErrNotFound` if already deleted
- [ ] `Restore` checks for slug/name conflict before clearing `deleted_at`
- [ ] Cascading soft delete and restore happen in one DB transaction
- [ ] Hard purge job runs on a schedule; skips records under legal hold
- [ ] `deleted_at` timestamp is used for purge scheduling — not `is_deleted` boolean
- [ ] GDPR erasure requests bypass the soft delete retention window — immediate hard purge path

## After Completion

- Use **`customer-offboarding`** for tenant-level deletion which combines soft delete + purge
- Use **`data-retention-policies`** for configuring how long soft-deleted records are retained
- Use **`audit-trail-design`** to log who deleted and restored records
- Use **`migration-safety`** for adding `deleted_at` and partial indexes to existing tables safely
