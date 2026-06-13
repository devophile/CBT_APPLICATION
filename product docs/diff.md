# 🔍 Validation Report: CBT Platform PRD & Phase Plans

**Date:** 2026-06-13  
**Reference Documents:** `product_feature.md`, `technical_prd.md`, `phase1_plan.md`, `phase2_plan.md`, `phase3_plan.md`

After cross-referencing the product features, technical PRD, and phase-wise implementation plans, several critical gaps, architectural flaws, and logical errors have been identified. If unaddressed, these will cause functional failures, security vulnerabilities, and data inconsistencies during development or in production.

---

## 1. Architectural & Logic Errors

### 1.1 SSE Connection Management Bug (Phase 3)
- **Location:** `phase3_plan.md` (Task 4.3 - `app/api/notification.py`)
- **Error:** The real-time notification system uses a dictionary `active_connections: dict[str, asyncio.Queue] = {}` to map a `user_id` to a **single** queue. If a teacher opens multiple browser tabs, the second tab overwrites the dictionary entry. When the first tab is closed, it triggers the `finally` block (`active_connections.pop(user_id, None)`), which severs the connection for the active second tab.
- **Impact:** Broken real-time notifications for users with multiple tabs open.
- **Fix:** Map `user_id` to a list or set of queues, managing individual connections rather than overriding them.

### 1.2 Storage Leak with Question Images (Phase 2)
- **Location:** `phase2_plan.md` (Task 1)
- **Error:** The plan introduces Cloudinary for image uploads and provides an `IStorageService.delete_image` method. However, the API endpoints for deleting a question (`DELETE /teacher/exams/{exam_id}/questions/{id}`) or replacing an image are not instructed to invoke this deletion method.
- **Impact:** Orphaned images will accumulate on Cloudinary over time, causing storage leaks and unnecessary costs.
- **Fix:** Hook the storage deletion logic into the question update and delete workflows.

### 1.3 Race Condition in Exam Start (Phase 1)
- **Location:** `technical_prd.md` / `phase1_plan.md` (Task 5.5 - `start_exam`)
- **Error:** The endpoint checks if an attempt already exists `existing = await db.execute(...)` **before** acquiring a row lock `with_for_update()`. If a student double-clicks the "Start Exam" button, two concurrent requests can bypass this check, deducting coins twice. The second insertion into `exam_attempts` will fail with an unhandled `IntegrityError` due to the unique constraint.
- **Impact:** Unhandled 500 Server Errors on the frontend and potential double billing in edge cases.
- **Fix:** Move the existing attempt check *inside* the locked transaction block, or catch the `IntegrityError` and gracefully return a `409 Conflict`.

---

## 2. Product vs. Technical Discrepancies

### 2.1 Critical MVP Feature Missing (Phase 1)
- **Location:** `phase1_plan.md` vs `product_feature.md`
- **Error:** `product_feature.md` explicitly states that if a student is blocked from a private exam due to insufficient teacher funds, the *"System automatically sends a notification to the teacher."* However, `phase1_plan.md` states: `❌ NOT Included: Email notifications / In-app notification system`.
- **Impact:** Teachers will have no idea why their students are unable to take the exam during the Phase 1 launch, completely breaking the user loop for resolving wallet balances.
- **Fix:** Move basic email notifications or simple dashboard alerts for "Low Balance Blocks" into Phase 1.

### 2.2 Missing Audit Trail for Admin Credits
- **Location:** `technical_prd.md` / `phase1_plan.md` (Task 2.1)
- **Error:** The `credit_wallet` function passes an `admin_id` to record who issued the credit, but the `Transaction` database model lacks an `admin_id` or `created_by` column.
- **Impact:** Super Admins can manually mint coins, but the platform cannot audit *which* admin issued the credit.
- **Fix:** Add an `admin_id` (or `created_by`) column to the `transactions` table.

### 2.3 Negative Marking Calculation Risk
- **Location:** `phase1_plan.md` (Task 4.1 - `score_calculator.py`)
- **Error:** The score formula is: `score = (Decimal(correct) * positive_marks) - (Decimal(wrong) * negative_marks)`. If a user mistakenly enters a negative number in the UI (e.g., `-1`), the formula becomes `- (wrong * -1)`, which will **add** marks for incorrect answers.
- **Impact:** Compromised exam results.
- **Fix:** Enforce `negative_marks >= 0` at the API validation layer (Pydantic), or apply `abs()` inside the calculation.

### 2.4 Bilingual Exam Data Integrity Vulnerability
- **Location:** `product_feature.md` / `technical_prd.md`
- **Error:** A teacher can create an exam set to `single` language, add a bunch of English-only questions, and then go back and edit the exam settings to change the language to `bilingual`. The system does not validate if existing questions possess the required Bengali translations.
- **Impact:** The frontend will crash or display empty text blocks when students attempt the exam.
- **Fix:** Prevent editing the `language` of an exam if questions have already been added, or force the teacher to provide translations before saving the change.

---

## 3. Implementation Flaws in Code & Planning

### 3.1 Unsafe Analytics Aggregation (Phase 2)
- **Location:** `phase2_plan.md` (Task 3.2 - `get_exam_analytics`)
- **Error:** The code calculates `max_score: max(scores)`. If an exam has attempts but no scores are valid/calculated (an empty `scores` list), `max()` will throw a `ValueError: max() arg is an empty sequence`.
- **Fix:** Provide a default fallback: `max(scores, default=0)`.

### 3.2 Redis Session Split-Brain (Phase 3)
- **Location:** `phase3_plan.md` (Task 1.1)
- **Error:** Phase 3 introduces Redis caching for sessions (`session:{user_id}`). However, Phase 1 already implemented "single device login" using the `active_session_id` column in the PostgreSQL `users` table. The Phase 3 plan does not explain how these two systems synchronize.
- **Impact:** Potential split-brain state where a user's session is valid in PostgreSQL but expired in Redis (or vice versa), breaking authentication.
- **Fix:** Explicitly deprecate the PostgreSQL `active_session_id` in favor of Redis, or clearly document the sync mechanism between them.

### 3.3 Soft Deletion Query Leaks
- **Location:** `phase1_plan.md` (Edge Cases)
- **Error:** The plan states: *"Teacher soft-deleted → exams hidden | All exam queries filter `teacher.is_deleted == False`"*. Relying on developers to manually join the `User` table and filter `is_deleted == False` on every student-facing exam query is highly error-prone.
- **Impact:** If forgotten on a single endpoint, exams of deleted teachers will leak into the public platform.
- **Fix:** Implement a global SQLAlchemy `with_loader_criteria` or create specific database views to automatically filter out entities owned by soft-deleted users.
