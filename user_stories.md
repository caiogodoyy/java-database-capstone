# User Storys

## Admin User Stories

**Title:** Admin login
*As an **admin**, I want to log into the portal with my username and password, so that I can manage the platform securely.*
**Acceptance Criteria:**

1. Given valid credentials, when I submit the login form, then I’m authenticated and redirected to the admin dashboard.
2. Given invalid credentials, when I submit the login form, then I see an error and remain on the login page.
3. Session must expire after a period of inactivity (e.g., 30 minutes) and require re-login.
   
**Priority:** High
<br>**Story Points:** 3
<br>**Notes:**

* Enforce password hashing and rate limiting; support “forgot password” via separate story.

**Title:** Admin logout
*As an **admin**, I want to log out of the portal, so that I protect system access when I’m finished.*
**Acceptance Criteria:**

1. When I click “Log out,” the session is invalidated server-side.
2. I’m redirected to the login page and back navigation does not restore the session.
3. Any privileged API calls after logout return 401/403.
   
**Priority:** High
<br>**Story Points:** 1
<br>**Notes:**

* Ensure CSRF protection and SameSite cookies.

**Title:** Add doctor profile
*As an **admin**, I want to add doctors to the portal, so that providers can appear in search and accept appointments.*
**Acceptance Criteria:**

1. I can create a doctor with required fields: name, email (unique), specialization, license number, and status.
2. On save, the doctor receives an optional welcome email (toggleable).
3. Validation prevents duplicates (email/license) and returns actionable errors.
   
**Priority:** High
<br>**Story Points:** 5
<br>**Notes:**

* Consider bulk import (CSV) as a follow-up.

**Title:** Delete doctor profile
*As an **admin**, I want to delete a doctor’s profile, so that I can remove providers who no longer practice in the clinic.*
**Acceptance Criteria:**

1. Deleting prompts a confirmation that explains impact (e.g., future appointments).
2. If the doctor has future appointments, system prevents hard delete and offers “disable” (soft delete) instead.
3. Audit log records who performed the delete/disable and when.
   
**Priority:** High
<br>**Story Points:** 5
<br>**Notes:**

* Prefer soft delete to preserve historical records.

**Title:** Run monthly appointments report (MySQL stored procedure)
*As an **admin**, I want to run a stored procedure in the MySQL CLI to get the number of appointments per month, so that I can track usage statistics.*
**Acceptance Criteria:**

1. A stored procedure (e.g., `sp_monthly_appointments`) exists and returns month, year, total count.
2. Procedure can be triggered via CLI and (optionally) via an admin UI action that displays the results.
3. Results are exportable (CSV) and restricted to admin role.
   
**Priority:** Medium
<br>**Story Points:** 5
<br>**Notes:**

* Ensure DB privileges and safe execution; index on appointment date.

---

## Patient User Stories

**Title:** Browse doctors without login
*As a **patient**, I want to view a list of doctors without logging in, so that I can explore options before registering.*
**Acceptance Criteria:**

1. Public endpoint/page lists doctors with specialization and basic profile info.
2. Filters by specialization and search by name are available.
3. Private data (emails/phone) is hidden until login.
   
**Priority:** High
<br>**Story Points:** 3
<br>**Notes:**

* Cache responses for performance.

**Title:** Patient sign-up
*As a **patient**, I want to sign up using my email and password, so that I can book appointments.*
**Acceptance Criteria:**

1. Registration requires unique email, strong password, and email verification link.
2. On verification, the account becomes active and I can log in.
3. Errors show clear guidance (email already used, weak password, etc.).
   
**Priority:** High
<br>**Story Points:** 5
<br>**Notes:**

* Consider social login later.

**Title:** Patient login
*As a **patient**, I want to log into the portal, so that I can manage my bookings.*
**Acceptance Criteria:**

1. Valid credentials sign me in and route me to my dashboard.
2. Invalid credentials show a non-revealing error.
3. Session persistence (remember me) is optional and secure.
   
**Priority:** High
<br>**Story Points:** 3
<br>**Notes:**

* MFA as future enhancement.

**Title:** Patient logout
*As a **patient**, I want to log out of the portal, so that I secure my account.*
**Acceptance Criteria:**

1. Logout invalidates session and clears tokens.
2. Redirect to public landing page; back button doesn’t restore access.
3. API calls post-logout return 401/403.
   
**Priority:** High
<br>**Story Points:** 1
<br>**Notes:**

* Apply same session policies as admin.

**Title:** Book one-hour appointment
*As a **patient**, I want to log in and book an hour-long appointment, so that I can consult with a doctor.*
**Acceptance Criteria:**

1. I can pick a doctor, date, and one-hour slot from available times only.
2. System validates conflicts and the doctor’s unavailability before confirming.
3. On success, I see a confirmation and receive an email notification with details.
   
**Priority:** High
<br>**Story Points:** 8
<br>**Notes:**

* Block overlapping bookings; support timezone display; consider cancellation/reschedule in separate stories.

**Title:** View upcoming appointments
*As a **patient**, I want to view my upcoming appointments, so that I can prepare accordingly.*
**Acceptance Criteria:**

1. Dashboard lists future appointments with doctor, date/time, location/mode.
2. Past appointments are hidden by default with a toggle to view history.
3. Each item links to details and offers reschedule/cancel actions (if within policy).
   
**Priority:** High
<br>**Story Points:** 3
<br>**Notes:**

* Show status badges (Confirmed/Cancelled/Completed).

---

## Doctor User Stories

**Title:** Doctor login
*As a **doctor**, I want to log into the portal, so that I can manage my appointments.*
**Acceptance Criteria:**

1. Valid credentials take me to my doctor dashboard.
2. Invalid credentials show a generic error and throttle repeated attempts.
3. Session timeout policy is enforced.
   
**Priority:** High
<br>**Story Points:** 3
<br>**Notes:**

* Support role-based landing pages.

**Title:** Doctor logout
*As a **doctor**, I want to log out of the portal, so that I protect my data.*
**Acceptance Criteria:**

1. Logout ends my session and clears tokens/cookies.
2. No authenticated endpoints are accessible afterward.
3. Redirect to doctor login page.
   
**Priority:** High
<br>**Story Points:** 1
<br>**Notes:**

* Mirror patient/admin behavior.

**Title:** View appointment calendar
*As a **doctor**, I want to view my appointment calendar, so that I can stay organized.*
**Acceptance Criteria:**

1. Calendar shows daily/weekly views with confirmed, pending, and cancelled statuses.
2. I can click an event to see patient name, reason, and notes (privacy-safe).
3. Calendar updates in real time when bookings change.
   
**Priority:** High
<br>**Story Points:** 8
<br>**Notes:**

* Consider ICS export integration later.

**Title:** Mark unavailability
*As a **doctor**, I want to mark my unavailability, so that patients can only book available slots.*
**Acceptance Criteria:**

1. I can add recurring or one-off blocks (date/time ranges) of unavailability.
2. Blocked times immediately disappear from patient booking options.
3. Conflicts with existing appointments require confirmation and patient notification.
   
**Priority:** High
<br>**Story Points:** 5
<br>**Notes:**

* Support holidays and clinic-wide blackout windows.

**Title:** Update profile
*As a **doctor**, I want to update my profile (specialization and contact information), so that patients have up-to-date information.*
**Acceptance Criteria:**

1. I can edit specialization, bio, clinic location, and contact preferences.
2. Changes are validated and reflected in patient search results.
3. Sensitive fields are role-restricted and audited.
   
**Priority:** Medium
<br>**Story Points:** 3
<br>**Notes:**

* Optional approval workflow by admin.

**Title:** View patient details for upcoming appointments
*As a **doctor**, I want to view the patient details for my upcoming appointments, so that I can be prepared.*
**Acceptance Criteria:**

1. From a calendar event, I can access patient essentials: name, age, reason for visit, and prior notes/prescriptions (if permitted).
2. Access is limited to appointments where I am the assigned provider.
3. All access is audited for compliance.
   
**Priority:** High
<br>**Story Points:** 5
<br>**Notes:**

* Respect data minimization; show only necessary information.
