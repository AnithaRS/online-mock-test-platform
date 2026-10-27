
## 1. Document Control & Executive Summary
### 1.1 Business Context & Purpose
Educational institutes need to assess student readiness through timed, standardized mock exams. Manual paper-based tests or unstructured Google Forms fail to deliver:
* Real-time automated grading.
* Granular attempt tracking across multiple sittings.
* Instant visibility for teachers into **which students did not appear**.
* Deep question-level analytics (which questions confused students the most).
This document details the functional specifications, business rules, user workflows, data contracts, and non-functional requirements (NFRs) necessary for developers to build an enterprise-grade **Online Mock Test Application**.
---
## 2. User Roles & Access Control Matrix (RBAC)
The system enforces strict Role-Based Access Control (RBAC). A user cannot switch roles without administrative privilege re-assignment.
| Feature / Capability | Teacher | Student | Guest / Anonymous |
| :--- | :---: | :---: | :---: |
| Authenticate via Credentials | Allowed | Allowed | Denied |
| Create, Edit, or Delete Questions in Question Bank | Allowed | Denied | Denied |
| Bulk Upload Questions via CSV / Excel | Allowed | Denied | Denied |
| Create, Configure, and Publish Mock Tests | Allowed | Denied | Denied |
| Take / Attempt a Published Mock Test | Denied | Allowed | Denied |
| View Immediate Score & Explanations Post-Test | Denied | Allowed (Own Score Only) | Denied |
| View Personal Past Attempts History | Denied | Allowed (Own History Only) | Denied |
| View Aggregate Class Analytics Dashboard | Allowed | Denied | Denied |
| View List of "Not Attended" Students | Allowed | Denied | Denied |
| View Individual Student Attempt Breakdown | Allowed | Denied | Denied |
---
## 3. High-Level System Workflow

```mermaid
flowchart TD
    subgraph Teacher_Workflow [1. Teacher Setup]
        A[Add Questions to Question Bank] --> B[Create Mock Test]
        B --> C[Configure Timer & Allowed Attempts]
        C --> D[Publish Test]
    end

    subgraph Student_Workflow [2. Student Test Experience]
        D --> E[View Available Tests in Lobby]
        E --> F[Start Timed Attempt]
        F --> G[Select Answers - Auto-Saved]
        G --> H[Submit Manually OR Auto-Submit on Timeout]
        H --> I[View Instant Score & Explanations]
    end

    subgraph Analytics_Workflow [3. Analytics & Reporting]
        H --> J[Analytics Engine Processes Submission]
        J --> K[Attendance Roster: Attended vs Not Attended]
        J --> L[Performance: Correct / Incorrect / Unanswered]
    end
```

### End-to-End Flow Summary
1. **Preparation:** Teacher adds MCQs to the Question Bank, selects questions, sets duration and attempt limits, and publishes the test.
2. **Execution:** Student enters the test, takes the exam against a live server countdown timer with continuous auto-saving.
3. **Completion:** Test submits on student confirmation or automatically when the timer reaches `00:00:00`.
4. **Immediate Feedback:** Student immediately


---
## 4. Detailed Functional Requirements (Module by Module)
### Module 1: Question Bank Management (Teacher Only)
#### 1.1 Objective
Provide a centralized repository where teachers curate multiple-choice questions (MCQs) before bundling them into tests.
#### 1.2 Specifications & Business Rules
* **Question Schema:**
  * **Question Text:** Rich text (supports text, mathematical symbols, or code snippets).
  * **Options:** Exactly **4 choices** (Option A, Option B, Option C, Option D).
  * **Correct Answer:** Exactly one option must be marked as `Correct`.
  * **Explanation / Solution:** Optional text explaining why the answer is correct (revealed to students post-submission).
  * **Metadata:** Subject / Topic tag (e.g., `Physics > Thermodynamics`), Difficulty Level (`Easy`, `Medium`, `Hard`).
* **Creation Channels:**
  1. **Single Entry:** Form-based UI to type the question, 4 choices, and select the radio button for the correct answer.
  2. **Bulk Upload:** CSV / XLSX import. The system must validate headers (`question_text, option_a, option_b, option_c, option_d, correct_option, explanation`) and reject corrupt rows with specific line-number error messages before committing to the database.
---
### Module 2: Test Creation & Configuration Engine (Teacher Only)
#### 2.1 Objective
Allow teachers to assemble questions into a formal test and define assessment constraints.
#### 2.2 Functional Steps
1. **Basic Info:** Teacher inputs `Test Title`, `Subject`, and `Description`.
2. **Question Selection:**
   * Teacher is presented with the Question Bank catalog.
   * Can search/filter by Topic, Subject, or Difficulty.
   * Teacher selects questions using checkboxes. A running tally displays: `Total Selected: N Questions`.
3. **Configurable Test Settings:**
   * **Time Limit:** Duration in minutes (e.g., `30 mins`, `60 mins`).
   * **Allowed Attempts:** Positive integer (e.g., `1`, `2`, `3`, or `Unlimited`).
   * **Assigned Batch / Class:** Group of students eligible to take this test.
   * **Status:** `Draft` (hidden from students) or `Published` (immediately visible to students).
---
### Module 3: Student Exam Execution Engine (Student Only)
#### 3.1 Objective
Provide a distraction-free, reliable, real-time testing environment.
#### 3.2 Functional Specifications
* **Pre-Exam Screen (Lobby):**
  * Displays: Test Title, Instructions, Total Questions, Time Limit, and Remaining Allowed Attempts (e.g., *"Attempt 1 of 3 used"*).
  * If a student has reached the maximum allowed attempts, the **"Start Test"** button is disabled with the warning: *"You have exhausted all allowed attempts for this test."*
* **Active Exam Interface:**
  * **Countdown Timer:** Prominently visible sticky timer at the top.
    * When `Time Remaining <= 5 minutes`, the timer pulses yellow/amber.
    * When `Time Remaining <= 1 minute`, the timer pulses red.
  * **Question Palette / Navigation Drawer:**
    * Grid showing numbers 1 to N.
    * Color-coded status for each question:
      * **Grey:** Not Visited.
      * **Red / Amber:** Visited, but Unanswered.
      * **Green:** Answered.
      * **Purple:** Marked for Review (allows student to bookmark a question).
  * **Draft Auto-Saving:** Every time a student selects or changes a radio button, the selection is instantly persisted to the backend session. If the user accidentally refreshes or loses connection, their selected options are preserved.
* **Submission Logic:**
  1. **Manual Submission:** Student clicks "Submit Test". A modal displays summary metrics: *"You answered 18 of 20 questions. 2 are unanswered. Are you sure you want to finish?"*
  2. **Auto-Submission (Timer Expiry):** When the countdown reaches `00:00:00`, the system automatically freezes all inputs, captures the latest state, and submits without requiring student confirmation.
---
### Module 4: Instant Evaluation & Feedback (Student Only)
#### 4.1 Objective
Immediately calculate and display test performance upon submission.
#### 4.2 Scoring Engine
* **Evaluation Rules:**
  * Score = Total count of correct answers × 1.0.
  * Percentage = (Score / Total Questions) × 100.
* **Immediate Post-Submission Result Screen:**
  * **Scorecard:** Total Score (e.g., `16 / 20`), Accuracy %, Time Spent.
  * **Question-by-Question Review:**
    * The student's chosen option.
    * The correct option (highlighted in green).
    * If the student's answer was incorrect, their selection is highlighted in red.
    * If left unanswered, labeled as `Unanswered`.
    * Official explanation note displayed below each question.
---
### Module 5: Student Attempt History Portal (Student Only)
#### 5.1 Objective
Empower students to monitor their progress and review past performance over time.
#### 5.2 Functional Specifications
* In the student's personal dashboard, every test shows an **Attempt Ledger**:
  * `Attempt #` (1, 2, 3...)
  * `Date & Timestamp`
  * `Score Achieved`
  * `Time Taken`
  * `Action Link`: "Review Answers" (re-opens the read-only post-submission review for that specific historical attempt).
---
### Module 6: Teacher Analytics & Test Performance Dashboard (Teacher Only)
#### 6.1 Objective
Provide deep educational insights into class attendance and question efficacy.
#### 6.2 Key Dashboard Views
##### View A: Test Attendance Summary
* **Total Assigned Students:** Total enrolled students in the class.
* **Attended Count:** Count of unique students who submitted at least 1 attempt.
* **Not Attended Count:** Count of eligible students with 0 attempts.
* **Actionable "Not Attended" Roster:**
  * A table listing: Student Name, Roll Number, Email Address.
  * Quick-action button: *"Export Absentee List (CSV)"* or *"Send Reminder Notification"*.
##### View B: Aggregated Test Performance Metrics
For a selected test, the dashboard aggregates across all submissions:
1. **Total Questions Answered Correctly:** Sum of all correct answers across all student attempts.
2. **Total Questions Answered Incorrectly:** Sum of all wrong answers.
3. **Total Questions Left Unanswered:** Sum of questions skipped or unselected.
4. **Class Average Score:** Mean score across students.
5. **Score Distribution Chart:** Histogram of score brackets (e.g., `< 40%`, `40–70%`, `> 70%`).
##### View C: Question-Level Item Analysis (Diagnostic Matrix)
A table listing every question in the test with performance metrics:

| Q# | Question Snippet | % Correct | % Incorrect | % Skipped | Difficulty Assessment |
| :---: | :--- | :---: | :---: | :---: | :---: |
| Q1 | What is the default scope of a Spring Bean in Spring Boot? | 86% | 9% | 5% | Low (Core Concept Mastered) |
| Q2 | In SQL, which clause filters rows *before* aggregation: WHERE or HAVING? | 62% | 31% | 7% | Medium (Standard Query Logic) |
| Q3 | In Java, which exception causes `@Transactional` to rollback by default? | 26% | 64% | 10% | High (Needs Revision in Class) |
| Q4 | What is the internal data structure used by `ConcurrentHashMap` in Java 8+? | 31% | 55% | 14% | High (Deep-Dive Concept) |

##### View D: Student-Wise Attempts Breakdown
* Expandable row per student showing:
  * Number of attempts made (e.g., `2 / 2`).
  * Best Score vs. Latest Score.
  * Link to inspect the student's exact submission sheet.


---

### Table Details & Field Specifications

#### 1. `users` — Authentication & Batch Identification
*Consolidates Teachers and Students into one table with role flags and batch identifiers.*

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `VARCHAR(36)` | `PRIMARY KEY` | Unique user UUID |
| `name` | `VARCHAR(100)` | `NOT NULL` | Full legal name |
| `email` | `VARCHAR(150)` | `NOT NULL, UNIQUE` | Unique login email |
| `password_hash` | `VARCHAR(255)` | `NOT NULL` | Encrypted credential |
| `role` | `VARCHAR(20)` | `NOT NULL` | `'TEACHER'` or `'STUDENT'` |
| `batch_id` | `VARCHAR(50)` | `NULL` | Student class batch (e.g., `'BATCH-2026-CS'`) |
| `roll_number` | `VARCHAR(50)` | `NULL` | Student Roll Number for absentee reports |
| `created_at` | `TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP` | Account creation date |

---

#### 2. `questions` — Question Bank
*Stores MCQs with flat choices (`option_a` through `option_d`) to eliminate an unnecessary 4-row options child table.*

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `VARCHAR(36)` | `PRIMARY KEY` | Unique question UUID |
| `question_text` | `TEXT` | `NOT NULL` | Question body (Markdown / LaTeX supported) |
| `option_a` | `TEXT` | `NOT NULL` | Option A content |
| `option_b` | `TEXT` | `NOT NULL` | Option B content |
| `option_c` | `TEXT` | `NOT NULL` | Option C content |
| `option_d` | `TEXT` | `NOT NULL` | Option D content |
| `correct_option` | `CHAR(1)` | `NOT NULL` | `'A'`, `'B'`, `'C'`, or `'D'` (never sent during exam) |
| `explanation` | `TEXT` | `NULL` | Solution reasoning (revealed post-test) |
| `subject` | `VARCHAR(80)` | `NOT NULL` | Subject classification (e.g., `'Computer Science'`) |
| `topic` | `VARCHAR(100)` | `NOT NULL` | Sub-topic tag (e.g., `'SQL & Relational DB'`) |
| `difficulty` | `VARCHAR(20)` | `DEFAULT 'MEDIUM'` | `'EASY'`, `'MEDIUM'`, or `'HARD'` |
| `created_by` | `VARCHAR(36)` | `FK -> users.id` | Teacher who created the question |
| `created_at` | `TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP` | Creation timestamp |

---

#### 3. `mock_tests` — Exam Configuration
*Defines duration, attempt limits, target batch, and visibility.*

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `VARCHAR(36)` | `PRIMARY KEY` | Unique test UUID |
| `title` | `VARCHAR(150)` | `NOT NULL` | Title of the test |
| `description` | `TEXT` | `NULL` | Instructions and syllabus summary |
| `duration_minutes` | `INTEGER` | `NOT NULL` | Time limit in minutes (e.g., `30`, `60`) |
| `max_attempts` | `INTEGER` | `DEFAULT 1` | Allowed attempts (`1`, `2`, `3`, or `0` for unlimited) |
| `batch_id` | `VARCHAR(50)` | `NOT NULL` | Eligible batch assigned to this test |
| `status` | `VARCHAR(20)` | `DEFAULT 'DRAFT'` | `'DRAFT'` (hidden) or `'PUBLISHED'` (active) |
| `created_by` | `VARCHAR(36)` | `FK -> users.id` | Teacher who created the test |
| `created_at` | `TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP` | Creation timestamp |
| `published_at` | `TIMESTAMP` | `NULL` | Timestamp when test was published |

---

#### 4. `test_questions` — Junction Table & Sequence
*Maps questions to tests and preserves question order.*

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `VARCHAR(36)` | `PRIMARY KEY` | Unique mapping UUID |
| `test_id` | `VARCHAR(36)` | `FK -> mock_tests.id ON DELETE CASCADE` | Associated test |
| `question_id` | `VARCHAR(36)` | `FK -> questions.id ON DELETE RESTRICT` | Associated question |
| `question_order` | `INTEGER` | `NOT NULL` | Display sequence (1, 2, 3...) |

> [!NOTE]
> Composite unique constraints ensure clean data integrity:
> - `UNIQUE(test_id, question_id)`: Prevents duplicate questions in a test.
> - `UNIQUE(test_id, question_order)`: Guarantees unique question numbering.

---

#### 5. `test_attempts` — Student Sittings & Scores
*Tracks individual attempt sessions against a server-authoritative clock.*

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `VARCHAR(36)` | `PRIMARY KEY` | Unique attempt session UUID |
| `test_id` | `VARCHAR(36)` | `FK -> mock_tests.id` | Target test |
| `student_id` | `VARCHAR(36)` | `FK -> users.id` | Attempting student |
| `attempt_number` | `INTEGER` | `NOT NULL` | Attempt counter (1, 2, 3...) |
| `started_at` | `TIMESTAMP` | `NOT NULL` | **Server-authoritative** exam start time |
| `submitted_at` | `TIMESTAMP` | `NULL` | Submission time (NULL while in progress) |
| `time_spent_seconds`| `INTEGER` | `DEFAULT 0` | Actual time taken by the student |
| `total_score` | `DECIMAL(5,2)` | `DEFAULT 0.00` | Count of correct choices × 1.0 |
| `percentage` | `DECIMAL(5,2)` | `DEFAULT 0.00` | Final score percentage |
| `status` | `VARCHAR(20)` | `DEFAULT 'IN_PROGRESS'` | `'IN_PROGRESS'`, `'SUBMITTED'`, `'AUTO_SUBMITTED'` |

---

#### 6. `student_answers` — Answer Records & Review Bookmarks
*Stores per-question selections with draft auto-saving and marked-for-review bookmarks.*

| Column | Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | `VARCHAR(36)` | `PRIMARY KEY` | Unique answer record UUID |
| `attempt_id` | `VARCHAR(36)` | `FK -> test_attempts.id ON DELETE CASCADE` | Active attempt session |
| `question_id` | `VARCHAR(36)` | `FK -> questions.id` | Question being answered |
| `selected_option`| `CHAR(1)` | `NULL` | `'A'`, `'B'`, `'C'`, `'D'`, or `NULL` (skipped) |
| `is_marked_for_review` | `BOOLEAN` | `DEFAULT FALSE` | Bookmark flag for **Purple** palette state |
| `is_correct` | `BOOLEAN` | `NULL` | Evaluated on submission (`true` / `false`) |

---

## 📊 Teacher Analytics & Reporting Tables

These tables display real-time insights on class attendance, question efficacy, and individual progress.

### View A: Test Attendance & Absentee Roster

**Class Attendance Summary:**
| Total Enrolled | Attended Count | Not Attended (Absent) | Attendance Rate |
| :---: | :---: | :---: | :---: |
| **30 Students** | **18 Students** | **12 Students** | **60.0%** |

**Actionable "Not Attended" Absentee Roster:**
| Roll Number | Student Name | Email Address | Batch | Attendance Status |
| :--- | :--- | :--- | :---: | :---: |
| `CS-2026-004` | Aarav Sharma | `aarav.s@college.edu` | `Batch CS-A` | 🔴 **Absent (0 Attempts)** |
| `CS-2026-009` | Diya Patel | `diya.p@college.edu` | `Batch CS-A` | 🔴 **Absent (0 Attempts)** |
| `CS-2026-015` | Rohan Verma | `rohan.v@college.edu` | `Batch CS-A` | 🔴 **Absent (0 Attempts)** |
| `CS-2026-021` | Sneha Nair | `sneha.n@college.edu` | `Batch CS-A` | 🔴 **Absent (0 Attempts)** |
| `CS-2026-027` | Vikram Singh | `vikram.s@college.edu` | `Batch CS-A` | 🔴 **Absent (0 Attempts)** |

---

### View B: Aggregated Test Performance Metrics

| Metric Category | Count / Value | Derivation Formula |
| :--- | :---: | :--- |
| **Total Test Submissions** | **24 Attempts** | Sum of attempts across 18 attended students |
| **Total Correct Answers** | **336 Answers** | `COUNT(is_correct = TRUE)` |
| **Total Incorrect Answers** | **108 Answers** | `COUNT(is_correct = FALSE AND selected_option IS NOT NULL)` |
| **Total Questions Skipped** | **36 Answers** | `COUNT(selected_option IS NULL)` |
| **Class Average Score** | **70.0% (14.0 / 20)** | `AVG(test_attempts.total_score)` |
| **Highest Score** | **95.0% (19.0 / 20)** | `MAX(test_attempts.total_score)` |
| **Lowest Score** | **35.0% (7.0 / 20)** | `MIN(test_attempts.total_score)` |

**Performance Distribution Brackets:**
| Performance Bracket | Score Range | Students | Proportion | Visual Distribution |
| :--- | :---: | :---: | :---: | :--- |
| **Critical Revision Required** | `< 40%` | 2 | 11.1% | `■■` |
| **Moderate Understanding** | `40% – 70%` | 7 | 38.9% | `■■■■■■■` |
| **Mastery Achieved** | `> 70%` | 9 | 50.0% | `■■■■■■■■■` |

---

### View C: Question-Level Item Analysis (Diagnostic Matrix)

Identifies concept mastery versus problem areas that require classroom revision.

| Q# | Topic Tag | Question Snippet | % Correct | % Wrong | % Skipped | Difficulty | Pedagogical Diagnosis |
| :---: | :--- | :--- | :---: | :---: | :---: | :---: | :--- |
| **Q1** | Spring Core | What is the default scope of a Spring Bean? | **86%** | 9% | 5% | `EASY` | 🟢 Concept Mastered |
| **Q2** | SQL Queries | Filters rows before aggregation: WHERE or HAVING? | **62%** | 31% | 7% | `MEDIUM` | 🟡 Satisfactory Understanding |
| **Q3** | Transactions | Which exception causes `@Transactional` rollback? | **26%** | 64% | 10% | `HARD` | 🔴 **Needs Class Revision** |
| **Q4** | Concurrency | Structure of `ConcurrentHashMap` in Java 8+? | **31%** | 55% | 14% | `HARD` | 🔴 **Needs Deep-Dive Lecture** |
| **Q5** | REST APIs | HTTP status code returned when resource created? | **91%** | 5% | 4% | `EASY` | 🟢 Concept Mastered |

---

### View D: Student-Wise Attempts Breakdown

| Roll Number | Student Name | Attempts | Best Score | Latest Score | Time Spent | Final Status |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| `CS-2026-001` | Ananya Roy | 2 / 2 | **18 / 20 (90%)** | 18 / 20 (90%) | 24m 12s | 🟢 Submitted |
| `CS-2026-002` | Devansh Mehta | 2 / 2 | **15 / 20 (75%)** | 12 / 20 (60%) | 29m 45s | 🟡 Auto-Submitted |
| `CS-2026-003` | Ishaan Gupta | 1 / 2 | **14 / 20 (70%)** | 14 / 20 (70%) | 31m 10s | 🟢 Submitted |
| `CS-2026-005` | Priya Rao | 1 / 2 | **19 / 20 (95%)** | 19 / 20 (95%) | 18m 05s | 🟢 Submitted |

---

## 👨‍🎓 Student Experience & Attempt Ledger

### Question Palette Color Code System

| Palette State | Color Indicator | Meaning & Behavior |
| :--- | :---: | :--- |
| **Not Visited** | ⚪ `Grey` | Question has not been viewed yet |
| **Visited Unanswered** | 🟠 `Amber` | Question viewed but no option selected |
| **Answered** | 🟢 `Green` | Option selected and auto-saved to server |
| **Marked for Review** | 🟣 `Purple` | Question bookmarked by student for re-checking |

---

### Personal Attempt History Ledger (Student Portal)

| Attempt # | Date & Timestamp | Score Achieved | Percentage | Time Taken | Submission Mode |
| :---: | :--- | :---: | :---: | :---: | :---: |
| **Attempt 2** | Oct 27, 2026, 09:30 AM | **18 / 20** | **90%** | 24m 12s | Manual Submission |
| **Attempt 1** | Oct 26, 2026, 03:15 PM | **14 / 20** | **70%** | 28m 50s | Auto-Submit (Timer Expiry) |

---

## 🛡️ Edge Cases & System Safeguards

> [!TIP]
> **Key Architectural Safeguards:**
> 1. **Network Drop / Tab Reload:** Elapsed time is calculated using `CURRENT_TIMESTAMP - started_at` on the server. Refreshing the browser preserves all answers and resumes the accurate remaining time.
> 2. **Attempt Lockout:** When `attempt_number >= max_attempts`, the API returns `HTTP 403 ATTEMPTS_EXHAUSTED` and disables the "Start Test" button in the Lobby.
> 3. **Tamper Protection:** The `correct_option` and `explanation` fields are strictly filtered out of API payloads during active exam sessions.
> 4. **Grace Period on Expiry:** Submissions post-timeout are capped to a strict 10-second network buffer, evaluating only selections saved prior to expiry.

---



## 6. Critical Business Rules & Edge Cases
| Scenario / Edge Case | Business Rule & System Reaction |
| :--- | :--- |
| **Network Loss / Tab Close During Test** | The backend stores the `started_at` timestamp. If a student closes their tab and reopens it after 10 minutes, the timer continues running from the server clock. Their previously selected answers are restored from the database. |
| **Submission After Timer Has Expired** | If a student's client device attempts to submit answers past the expiration timestamp + 10 seconds grace period, the server rejects late inputs and evaluates only whatever was saved prior to expiration. |
| **Attempt Limit Exhaustion** | If `max_attempts = 2`, and the student has 2 completed attempts in `TEST_ATTEMPTS`, any request to initiate a 3rd attempt returns an HTTP `403 Forbidden` error with the code `ATTEMPTS_EXHAUSTED`. |
| **Teacher Edits a Published Test** | Once at least one student has initiated an attempt on a test, the teacher cannot remove questions or alter the duration. Only title/description edits are permitted to maintain data consistency. |
| **Question Left Unanswered** | In `STUDENT_ANSWERS`, `selected_option` is recorded as `NULL`, `is_correct = false`. On the dashboard, it increments the `Unanswered` metric. |
---
## 7. Non-Functional Requirements (NFRs)
### 7.1 Performance & Scalability
* **Concurrent Exam Start:** The system must handle up to **500 students simultaneously clicking "Start Test"** within a 30-second window without database connection pool exhaustion or timeouts.
* **Auto-Save Latency:** Background saving of answer clicks must complete with an API response time of **< 200 ms**.
* **Instant Evaluation:** The calculation of scores and generation of the review screen must occur in **< 1.0 second** upon clicking Submit.
### 7.2 Integrity & Security
* **Tamper-Proof Answers:** The correct answers (`correct_option`) and explanations must **never** be sent to the student's browser during the active test session. Inspecting Network DevTools or HTML DOM must not reveal the answer key.
* **Server-Authoritative Clock:** The countdown timer must synchronize against the server time upon initialization to prevent students from modifying their system clock to gain extra time.
### 7.3 Usability & Device Support
* Fully responsive layout optimized for Desktop/Laptop screens (1366x768 and above) and Tablets (e.g., iPads / Chromebooks commonly used in schools).
---
## 8. QA Acceptance Criteria
### Scenario 1: Teacher Successfully Creates a Timed Test

Given a Teacher is logged into the platform
And there are at least 20 questions available in the Question Bank
When the Teacher selects 10 questions
And sets Duration to "45" minutes
And sets Allowed Attempts to "2"
And clicks "Publish Test"
Then the test status changes to "PUBLISHED"
And the test immediately appears on the Student dashboard.

### Scenario 2: Student Exceeds Allowed Attempts
Given a Student has already completed 2 attempts on a test where Max Attempts is "2"
When the Student visits the Test Lobby screen
Then the "Start Test" button is disabled
And an informative alert states: "You have used all 2 allowed attempts."

### Scenario 3: Auto-Submit on Timer Expiration

Given a Student is actively taking a mock test
And the countdown timer reaches "00:00:00"
When the time expires
Then the test interface automatically locks to prevent further selections
And the attempt is automatically submitted to the server
And the Student is redirected to the Score Summary screen.

### Scenario 4: Teacher Dashboard Correctly Reflects "Not Attended"
Given a class has 30 enrolled students
And a published test has been submitted by 18 students
When the Teacher opens the Test Analytics Dashboard
Then "Attended Count" displays "18"
And "Not Attended Count" displays "12"
And the "Not Attended" table lists exactly the 12 students who have 0 registered attempts.

