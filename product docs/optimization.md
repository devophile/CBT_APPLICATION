# 🚀 Architectural & Database Optimization Report — CBT Platform

This document outlines the deep-dive architectural analysis, database join optimizations, and concurrency management strategies for the Devophile CBT Platform.

---

## 📊 1. High-Concurrency Write Paths (The "Exam Start" & Wallet Flow)

**Scenario:** Hundreds of students in a batch start an exam at the exact same minute (`POST /student/exams/{id}/start`).
**Operations:** Check exam access -> Calculate cost -> Deduct from wallet -> Create attempt.

*   **Current Design:** Uses SQLAlchemy `with_for_update()` to apply a `SELECT ... FOR UPDATE` row-level lock on the specific `Wallet` row.
*   **Join Analysis:** 
    *   Access verification uses `select(ExamBatch).join(BatchStudent).where(...)`. This is a clean `INNER JOIN` across associative tables with Primary/Foreign Keys. Highly optimized; Postgres handles this in `< 1ms`.
*   **Architectural Verdict:** Excellent. `with_for_update()` prevents race conditions (e.g., a student double-clicking "Start" and bypassing a 0-balance check). 
*   **Optimization Chance (Ledger Integrity):** The wallet relies on `Wallet.balance` being perfectly in sync with `SUM(Transaction.amount)`. 
    *   *Recommendation:* Add a nightly DB cron job/script that runs a reconciliation query to catch floating-point drift or lock failures:
        ```sql
        SELECT w.id, w.balance, SUM(t.amount) 
        FROM wallets w 
        JOIN transactions t ON w.id = t.wallet_id 
        GROUP BY w.id 
        HAVING w.balance != SUM(t.amount);
        ```

## 🧠 2. Heavy Computation & Memory Usage (Analytics & Scoring)

**Scenario A: Exam Scoring (`POST /student/exams/{id}/submit`)**
*   **Current Design:** Fetches all questions, pulls the student's `JSONB` answers, and runs a Python `for` loop to calculate the score based on `positive_marks` and `negative_marks`.
*   **Verdict:** Highly Optimized. Doing this in Python is CPU-bound and lightning-fast. Storing answers as `JSONB` instead of an `attempt_answers` table with millions of rows is a brilliant decision that saves massive I/O overhead.

**Scenario B: Teacher Analytics (`GET /teacher/exams/{id}/analytics`)**
*   **Current Design (Original PRD):** The PRD specifies pulling **all** submitted attempts into Python memory to calculate averages, max/min, and distributions:
    ```python
    attempts = attempts.all() # Loads ALL rows into memory
    scores = [float(a[0].score) for a in attempts]
    avg_score = sum(scores) / len(scores)
    ```
*   **Architectural Flaw:** If an exam goes "Global" and gets 50,000 attempts, pulling 50,000 rows (with `User` JOINs) into Python memory just to calculate an average will cause the FastAPI worker to spike in RAM and potentially crash (OOM). 
*   **🔥 Critical Optimization Chance (DB-Level Aggregation):** 
    Mathematical aggregations MUST be pushed down to PostgreSQL. 
    *   *Optimized SQLAlchemy Query:*
        ```python
        # Let Postgres do the heavy lifting
        stats = await db.execute(
            select(
                func.count(ExamAttempt.id).label("total_attempts"),
                func.avg(ExamAttempt.score).label("avg_score"),
                func.max(ExamAttempt.score).label("max_score"),
                func.min(ExamAttempt.score).label("min_score"),
                func.avg(ExamAttempt.time_taken_seconds).label("avg_time")
            )
            .where(ExamAttempt.exam_id == exam_id, ExamAttempt.is_submitted == True)
        )
        ```
    *   For the **Score Distribution Bucket**, use Postgres `width_bucket` or a `CASE` statement grouping directly in the DB query, pulling only 5-10 rows of bucket data instead of 50,000 rows of raw attempts. You only pull raw rows for `LIMIT 5` (Top Performers).

## 🔍 3. Indexing & Table Joins (Listing & Searches)

**Scenario:** "Get my active exams" or "Search students by email"

*   **Soft Deletes (`is_deleted = True`):**
    *   *Issue:* The schema relies heavily on `is_deleted = False` filters. Standard B-Tree indexes on `teacher_id` will still scan deleted rows before filtering them out.
    *   *🔥 Optimization Chance (Partial Indexes):* Create PostgreSQL Partial Indexes. They only index rows where `is_deleted = False`, making the index incredibly small and fast.
        ```python
        Index("ix_exams_active_teacher", Exam.teacher_id, 
              postgresql_where=(Exam.is_deleted == False))
        ```

*   **Search Functionality (`?search=john@example.com`):**
    *   *Issue:* Doing `User.email.ilike(f"%{search}%")` results in a Full Table Scan (Sequential Scan). As the user base grows past 100k, admin dashboards will slow to a crawl.
    *   *🔥 Optimization Chance (GIN/Trigram Indexes):* Enable the `pg_trgm` extension in PostgreSQL and apply a GIN index to searchable text fields.
        ```python
        # Requires: CREATE EXTENSION IF NOT EXISTS pg_trgm;
        Index("ix_users_email_trgm", User.email, 
              postgresql_using="gin", postgresql_ops={"email": "gin_trgm_ops"})
        ```

## 📉 4. N+1 Query Traps & Data Over-fetching

**Scenario:** Fetching an Exam to start it (`GET /exams/{id}`)
*   **Current Design:** `select(Exam).options(selectinload(Exam.questions))`
*   **Verdict:** Good. `selectinload` prevents the N+1 problem (it uses 2 queries: `SELECT * FROM exams` then `SELECT * FROM questions WHERE exam_id IN (...)`).
*   **Optimization Chance (Denormalization):**
    To calculate the total cost of an exam, the system currently fetches *all* questions to do `len(exam.questions) * config.cost_per_question`. 
    *   If a student is just browsing the global exam list (`GET /student/exams/global`), fetching thousands of questions across 20 exams just to display the "Coin Cost" on the UI card is highly inefficient.
    *   *Fix:* Denormalize the question count. Add `question_count: Mapped[int] = mapped_column(Integer, default=0)` to the `Exam` model. Update this integer using a database trigger or at the service layer whenever a question is added/deleted. This allows you to display exam costs on listing pages via a single lightweight query without joining the `questions` table at all.

## 🗄️ 5. Long-Term Data Growth (The Analytics Goldmine)

**Scenario:** Future feature requests, e.g., "Which question was answered incorrectly the most?"
*   **Current Design:** Answers are locked inside a `JSONB` column (`{"question_id": "selected_option"}`).
*   **Optimization Chance (JSONB Indexing):** 
    By default, PostgreSQL reads the entire JSONB blob to search inside it. If Phase 4 requires detailed question analytics, you will need a GIN index on the JSON keys.
    ```python
    Index("ix_exam_attempts_answers_gin", ExamAttempt.answers, postgresql_using="gin")
    ```
    This ensures that querying `WHERE answers ? 'specific_question_id'` utilizes an index rather than scanning every attempt in history.

---

### 📋 Action Items for Development

1.  **Refactor Analytics Services:** Do not use `attempts.all()` to calculate sums/averages in Python. Rewrite `get_exam_analytics` to use SQLAlchemy `func.avg()`, `func.max()`, and `func.count()`.
2.  **Add Partial Indexes:** Apply `postgresql_where=(is_deleted == False)` to indexes on the `exams` and `users` tables.
3.  **Denormalize `question_count`:** Add a `question_count` integer to the `Exam` model to calculate exam costs on list views without needing to `JOIN` or load the `questions` table.
4.  **Enforce Pagination:** Ensure the raw attempt fetching for CSV exports and analytics grids strictly uses offset/limit or chunking, avoiding large memory dumps.