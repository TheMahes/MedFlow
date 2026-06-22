# MedFlow — Comprehensive Audit & Implementation Plan

## Codebase Audit Summary

I audited all **50+ source files** across the MedFlow codebase. Below is the complete status report.

---

## ✅ Working Features

| Feature | Status | Notes |
|---|---|---|
| Admin registration & login | ✅ Working | [registerClinic](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/lib/auth/actions.ts#L10-L94) creates auth user + tenant + tenant_user |
| Login flow | ✅ Working | [LoginPage](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/app/(auth)/login/page.tsx) resolves tenant slug post-login |
| Department CRUD | ✅ Working | Full create/update/delete in [actions.ts](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/lib/tenant/actions.ts#L12-L98) |
| Counter CRUD | ⚠️ Partial | Works but has user-creation logic baked in (see below) |
| User invitation | ⚠️ Partial | Creates auth user but **never shows credentials** |
| Queue Board | ✅ Working | Realtime via [useRealtimeQueue](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/hooks/use-realtime-queue.ts) |
| Reception (Add Patient) | ✅ Working | [AddPatientForm](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/components/queue/add-patient-form.tsx) with token generation |
| Display Screens | ✅ Working | Token-based public access via [/display/[token]](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/app/display/[token]/page.tsx) |
| Analytics | ✅ Working | [AnalyticsDashboard](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/components/admin/analytics-dashboard.tsx) with daily/weekly stats |
| Settings | ✅ Working | [SettingsForm](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/components/admin/settings-form.tsx) |
| Sidebar RBAC | ✅ Working | [sidebar.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/components/layout/sidebar.tsx#L47-L49) filters nav by role |
| Sign-out | ✅ Working | [header.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/components/layout/header.tsx#L12-L16) |

---

## ❌ Broken / Incomplete Features

### 1. User Onboarding — No Credentials Shown

**File:** [users-manager.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/components/admin/users-manager.tsx#L43-L57)

The `handleInvite` function calls `inviteUser()` which creates a Supabase Auth account with a temporary password, but:
- The **generated password is never returned** to the frontend — [inviteUser](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/lib/tenant/actions.ts#L249-L313) returns `{ success: true }` without the password
- The UI shows a generic "User invited" success message — no credentials modal exists
- Invited users (doctors, receptionists) **cannot log in** because they never receive their credentials

### 2. Counter Creation — Mixed Responsibilities

**File:** [counters-manager.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/components/admin/counters-manager.tsx#L64-L88)

The counter creation form:
- Requires staff email + full name fields
- Calls [createCounter](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/lib/tenant/actions.ts#L102-L198) which creates a Supabase Auth user, links them to the tenant, AND creates the counter
- Has a credentials modal (lines 348-403) that shows the generated password
- **Problem:** User creation should NOT be tied to counter creation. The counters page is doing the Users page's job.

### 3. My Counter — "No Counter Assigned"

**File:** [counter/page.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/app/(dashboard)/[tenantSlug]/counter/page.tsx#L16-L24)

The lookup queries:
```sql
.eq('serving_doctor_id', session.tenantUser.id)
```
This is **correct** — it uses `session.tenantUser.id` (the `tenant_users.id` UUID, which is what `counters.serving_doctor_id` references). The problem is that **no doctors have been properly assigned through the normal flow**:
- Users invited via the Users page have no counter assignment
- Users created via Counters page may work, but it's the wrong flow

### 4. Doctor Dashboard — Missing

No dedicated doctor dashboard exists. All roles see the same [DashboardPage](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/app/(dashboard)/[tenantSlug]/page.tsx) with generic stats. Doctors need:
- Assigned counter info
- Personal patient stats
- Quick actions (open counter, start queue)

### 5. RBAC — Server-Side Route Protection Gaps

| Route | Protection | Issue |
|---|---|---|
| `/{slug}` (Dashboard) | `getSessionUser` | ✅ Auth only, OK for all roles |
| `/{slug}/queue` | `getSessionUser` | ✅ All roles can view queue |
| `/{slug}/reception` | `getSessionUser` | ⚠️ **No role check** — should require `queue:add` |
| `/{slug}/counter` | `getSessionUser` | ⚠️ **No role check** — should require doctor/admin |
| `/{slug}/departments` | `requirePermission` | ✅ `departments:manage` |
| `/{slug}/counters` | `requirePermission` | ✅ `counters:manage` |
| `/{slug}/users` | `requirePermission` | ✅ `users:manage` |
| `/{slug}/display` | `requirePermission` | ✅ `display:manage` |
| `/{slug}/analytics` | `requirePermission` | ✅ `analytics:view` |
| `/{slug}/settings` | `requirePermission` | ✅ `tenant:manage` |

### 6. No Password Change Flow

- No "change password" page exists
- No "forgot password" flow exists
- Invited users have temporary passwords with no way to change them
- No first-login password change enforcement

### 7. RLS — Recursive Policy Issue (Previously Fixed)

**File:** [003_fix_tenant_users_rls.sql](file:///Users/maheskaliraj/Documents/MedFlow/medflow/supabase/migrations/003_fix_tenant_users_rls.sql)

Migration 003 creates `SECURITY DEFINER` helper functions, but it's unclear if it was applied. The app-level fix (using `createAdminClient()` for user mutations) was applied in the previous conversation turn. However, the `get_user_tenant_ids()` function in [001_create_schema.sql](file:///Users/maheskaliraj/Documents/MedFlow/medflow/supabase/migrations/001_create_schema.sql#L139-L143) uses `sql` language which can be inlined by the PostgreSQL planner, potentially causing recursion in the SELECT policy.

### 8. Realtime — Missing Counter Updates

[useRealtimeQueue](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/hooks/use-realtime-queue.ts) subscribes to `queue_entries` changes only. There's no realtime subscription for:
- `counters` table changes (already in Supabase publication but no client subscription)
- Counter assignment updates don't propagate to the doctor's My Counter view

---

## User Review Required

> [!IMPORTANT]
> **Counter Creation Refactor:** The current `createCounter` action in [actions.ts](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/lib/tenant/actions.ts#L102-L198) creates auth users as part of counter creation. The plan refactors this so counters only reference **existing** tenant_users. This means counter creation will require doctors to be created first via the Users page. **Is this the desired workflow?**

> [!WARNING]
> **Database Migration Required:** The RLS fix migration (003) needs to be verified as applied on your Supabase instance. Additionally, the `get_user_tenant_ids()` function should be recreated as `plpgsql` to prevent query-planner inlining. **This requires running SQL against your live Supabase database.** I'll provide the migration file, but you'll need to run it in the Supabase SQL Editor.

---

## Open Questions

> [!IMPORTANT]
> 1. **Password delivery method:** Should the admin see the temporary password in a modal (copy-to-clipboard), or should we also send an email via Supabase's built-in email service? The plan currently implements the modal approach since Supabase email delivery requires SMTP configuration.
>
> 2. **Mandatory password change on first login:** Should doctors be forced to change their temporary password on first login? This would require adding a `password_changed` flag to `user_metadata` and a redirect in the proxy. The plan includes this.
>
> 3. **Should receptionists also see "My Counter"?** Currently the sidebar shows "My Counter" for doctors and admins only. Receptionists might also need a counter workspace.

---

## Proposed Changes

### Phase 1 — User Onboarding & Credentials

Refactor the Users page to be the single source of truth for user account creation with visible credential display.

---

#### [MODIFY] [actions.ts](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/lib/tenant/actions.ts)

**`inviteUser` → rename to `createUser`:**
- Return the generated temporary password to the caller
- Return `{ success, email, generatedPassword?, isNewUser }` instead of `{ success }`

#### [MODIFY] [users-manager.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/components/admin/users-manager.tsx)

- Rename "Invite Team Member" → "Add Team Member"  
- Add a **Credentials Modal** (reuse the pattern from `counters-manager.tsx` lines 348-403) that shows:
  - Email
  - Temporary Password
  - Copy Email button
  - Copy Password button
- Show modal after successful user creation
- For existing users, show "User already exists" message without credentials

---

### Phase 2 — Counter Management Refactor

Remove all user creation logic from the Counters module. Counters should only manage counters.

---

#### [MODIFY] [actions.ts](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/lib/tenant/actions.ts)

**`createCounter` refactor:**
- Remove email/fullName parameters
- Remove auth user creation logic (lines 119-147)
- Add `servingDoctorId` parameter (optional `tenant_users.id`)
- Counter creation becomes: name + department + optional doctor assignment
- Signature: `createCounter(tenantId, name, departmentId, servingDoctorId?, tenantSlug)`

#### [MODIFY] [counters-manager.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/components/admin/counters-manager.tsx)

- Remove email/fullName input fields from the Add Counter form
- Remove the `Credentials` type and `credentialsModal` state
- Remove the credentials modal (lines 348-403)
- Add a doctor dropdown in the Add Counter form (select from existing `doctors` prop)
- Counter form becomes: Counter Name + Department + Doctor (optional)

---

### Phase 3 — Doctor Authentication & Password Management

---

#### [NEW] [change-password/page.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/app/(auth)/change-password/page.tsx)

New page for changing password. Accessible at `/change-password`. Shows:
- Current password field
- New password field  
- Confirm new password field
- Submit button

#### [MODIFY] [actions.ts](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/lib/auth/actions.ts)

Add `changePassword` server action:
- Verify current password
- Update password via Supabase `auth.updateUser`
- Set `user_metadata.password_changed = true`

#### [MODIFY] [proxy.ts](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/proxy.ts)

Add first-login redirect logic:
- After session refresh, check `user.user_metadata.password_changed`
- If `false` and user is on a dashboard route → redirect to `/change-password`
- Skip check for public routes and the change-password page itself

#### [MODIFY] [actions.ts](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/lib/tenant/actions.ts)

In the new `createUser` function:
- Set `user_metadata.password_changed = false` when creating new users
- This flags them for mandatory password change on first login

---

### Phase 4 — Doctor Dashboard & My Counter

---

#### [MODIFY] [page.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/app/(dashboard)/[tenantSlug]/page.tsx)

Make the dashboard role-aware:
- **Admin/Super Admin:** Show current overview stats (existing behavior)
- **Doctor:** Show doctor-specific dashboard:
  - Assigned counter & department
  - Today's personal stats (patients seen, waiting at my counter, avg consultation time)
  - Quick action: "Open My Counter" button
  - Current patient (if serving)
- **Receptionist:** Show reception-focused view:
  - Quick "Add Patient" button
  - Today's queue stats

#### [MODIFY] [counter/page.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/app/(dashboard)/[tenantSlug]/counter/page.tsx)

Add server-side permission check:
```typescript
await requirePermission(tenantSlug, 'queue:call');
```

#### [MODIFY] [counter-view.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/components/queue/counter-view.tsx)

Enhance the counter workspace:
- Add "Recall Patient" action (for skipped patients)
- Add "Transfer Patient" action (move to different department)
- Show serving time timer for current patient
- Add counter open/close toggle

#### [MODIFY] [reception/page.tsx](file:///Users/maheskaliraj/Documents/MedFlow/medflow/src/app/(dashboard)/[tenantSlug]/reception/page.tsx)

Add server-side permission check:
```typescript
await requirePermission(tenantSlug, 'queue:add');
```

---

### Phase 5 — RLS & Database Hardening

---

#### [NEW] [004_fix_rls_and_helpers.sql](file:///Users/maheskaliraj/Documents/MedFlow/medflow/supabase/migrations/004_fix_rls_and_helpers.sql)

New migration that:
1. Recreates `get_user_tenant_ids()` as `plpgsql` (not `sql`) with `SECURITY DEFINER` to prevent query-planner inlining
2. Verifies all `tenant_users` policies use the `SECURITY DEFINER` helper functions
3. Ensures no recursive RLS patterns remain

```sql
-- Fix get_user_tenant_ids to use plpgsql to prevent inlining
CREATE OR REPLACE FUNCTION get_user_tenant_ids()
RETURNS SETOF UUID AS $$
BEGIN
    RETURN QUERY
    SELECT tenant_id FROM tenant_users
    WHERE user_id = auth.uid() AND is_active = TRUE;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;
```

---

## Verification Plan

### Automated Tests
No existing test suite. Will verify via:
```bash
npm run build
```
to ensure no TypeScript compilation errors.

### Manual Verification

#### Phase 1 — User Onboarding
1. Admin creates a new doctor user via Users page
2. Verify credentials modal appears with email + temp password
3. Log out, log in as the new doctor with temp credentials
4. Verify redirect to change-password page
5. Change password → verify redirect to dashboard

#### Phase 2 — Counter Management  
1. Create a counter with doctor dropdown (no email/password fields)
2. Verify counter appears with correct doctor assignment
3. Log in as assigned doctor → verify "My Counter" shows the counter

#### Phase 3 — Doctor Workflow
1. Doctor logs in → sees doctor-specific dashboard
2. Navigate to My Counter → see assigned counter
3. Call Next Patient → verify queue board updates in realtime
4. Complete Patient → verify status updates across all views
5. Skip Patient → verify recall works

#### Phase 4 — RBAC
1. Login as receptionist → verify only Reception, Queue Board, Dashboard visible
2. Try accessing `/counters` URL directly → verify redirect
3. Login as doctor → verify only Dashboard, Queue, My Counter visible
4. Login as display role → verify only Queue Board visible

#### Phase 5 — RLS
1. Run migration 004 in Supabase SQL Editor
2. Delete a team member from Users page → verify no recursion error
3. Update a user's role → verify no recursion error
