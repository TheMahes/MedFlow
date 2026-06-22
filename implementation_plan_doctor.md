# Implementation Plan: Counter Creation with User Setup

## Overview
Modify the counter creation flow so that when a new counter is created, an associated staff user (doctor) can also be created or assigned in the same step. This ensures counters are immediately usable and admins are provided with the temporary credentials for new staff.

## Proposed Changes

### `src/lib/tenant/actions.ts`
Modify the `createCounter` server action to accept `email` and `fullName` parameters.
1. **User Creation/Lookup**:
   - Check if an auth user with the given email exists.
   - If not, generate a temporary password (UUID-based + `Aa1!`) and create the user via `adminClient.auth.admin.createUser`, setting `user_metadata.full_name`.
2. **Tenant Linking**:
   - Check if the user is already linked to the tenant in `tenant_users`.
   - If not, link them with the role `doctor` (required for counter assignment).
3. **Counter Creation**:
   - Create the counter record, assigning `serving_doctor_id` to the `tenant_user.id`.
4. **Return Data**:
   - Return `{ success: true, counter, email, generatedPassword, isNewUser }`.

### `src/components/admin/counters-manager.tsx`
Update the `CountersManager` UI to collect the new fields and display the result.
1. **State Updates**:
   - Add state for `newEmail` and `newFullName`.
   - Add state for `credentialsModal` to hold the result of the creation.
2. **Form UI**:
   - Add inputs for "Staff Email" and "Staff Full Name" to the "Add Counter" section.
3. **Submission Handling**:
   - Update `handleAdd` to pass the new fields to `createCounter`.
   - On success, set the `credentialsModal` state with the returned data.
4. **Success Modal**:
   - Render a modal when `credentialsModal` is active.
   - If `isNewUser` is true: Display the email, temporary password, and a "Copy Credentials" button.
   - If `isNewUser` is false: Display a message indicating the existing user was assigned to the counter.

## Security & RLS
- **Tenant Isolation**: `createCounter` will continue to use `guardAction` to ensure the caller has `counters:manage` permission for the specified `tenantId`.
- **Admin Client**: User creation and tenant linking will use `createAdminClient()` (bypassing RLS) since these actions inherently require cross-tenant or elevated privileges, matching the existing `inviteUser` pattern.
- **Data Integrity**: The temporary password will only be returned to the client once and is not stored in plain text.

## User Review Required
> [!IMPORTANT]  
> Please review this plan. Once approved, I will implement the changes.
