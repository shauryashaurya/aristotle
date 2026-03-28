# Assessment Engine: Technical Specification

**Platform:** Palantir Foundry (Workshop, Pipeline Builder, Functions, AIP, Ontology)
**Scope:** Phase 1 build (3-4 days, 2 engineers). Phase 2 extensions documented in Appendix A.

---

## 0. Overview

### 0.1 Purpose

This system is an adaptive assessment platform built on Palantir Foundry. It ingests a structured question bank (JSON), converts open-ended questions into multiple-choice format using AIP, delivers assessments through a Workshop application, captures fine-grained telemetry, estimates user ability using Item Response Theory, and provides analytics dashboards for individual, cohort, and question-level analysis.

The system supports two distinct assessment modes:

- **Summative mode**: Timed, scored, standardized conditions. No feedback during the assessment. Results are official and contribute to the user's record and cohort comparisons.
- **Formative mode**: Untimed or loosely timed. Immediate feedback after each question (correct answer, explanation, justification from the source data). Focuses on learning. Results are diagnostic only and do not affect official scores.

A third mode, **practice mode**, is a variant of formative assessment that targets weak areas identified in prior summative attempts.

### 0.2 Phase 1 Scope

Phase 1 delivers a working end-to-end system. The boundary is defined as follows.

**In scope (Phase 1):**

- JSON ingestion with AIP-driven MCQ generation at ingest time
- Ontology with all core object types, link types, and actions
- Summative assessment flow (timed, adaptive, scored)
- Formative assessment flow (feedback-enabled, untimed)
- Practice mode (targeted at weak topics)
- 2PL IRT with bootstrap calibration from source difficulty priors
- Three-regime adaptive selection (metadata-driven, blended, fully statistical) that operates sensibly from the very first user onward, with full auditability of regime state, transition triggers, and per-question selection signals
- Per-user AIP-generated question variants and options, stored at the user-attempt level for reproducibility
- Telemetry capture and aggregation
- User feature computation
- Individual, cohort, and question effectiveness dashboards (using Workshop-native chart types)
- AIP chat panel for natural-language queries against assessment data

**Out of scope (Phase 2, see Appendix A):**

- Behavioral prediction model (needs training data that does not yet exist)
- Guessing detection model (same reason)
- Automated question promotion pipeline
- MLOps infrastructure (model registry, drift monitoring, automated retraining)
- Advanced chart types not native to Workshop (heatmaps, funnel charts, box plots)
- Role-based access control beyond Foundry's default permissions

### 0.3 Key Architectural Decisions

**D-001: Workshop state management.** Assessment state is managed through an Ontology object updated via Function-backed Actions. Each answer submission triggers a single Function that performs all state transitions synchronously: record response, update theta, select next question. AIP variant generation is pre-computed at assessment start for all candidate questions to avoid per-question latency during the assessment.

**D-002: MCQ generation strategy.** The source JSON contains open-ended questions with model answers and justifications. At ingestion time, AIP generates an initial set of 4 MCQ options per question (1 correct, 3 distractors) derived from the model answer. These serve as the canonical options for the question. At serve time, AIP generates fresh user-specific options for each QuestionInstance, taking into account the user's theta, the question's Bloom level, and the difficulty being tested. The user-specific options are stored on the QuestionInstance and are never shared across users or attempts.

**D-003: IRT bootstrapping.** On initial deployment with no response data, `b_param` (difficulty) is set from the source JSON's `difficulty_prior` field, normalized to the IRT scale. `a_param` (discrimination) is initialized from `effectiveness_prior`, giving the system differentiated discrimination estimates from day one. These are recalibrated as response data accumulates through an explicit three-regime transition process (see Section 4).

**D-004: AIP variant pre-generation and throughput.** When an assessment starts, the system pre-generates AIP variants for a pool of candidate questions (roughly 2x the assessment length) and caches them as QuestionInstance records. For a 45-question assessment, this means approximately 90 AIP calls at assessment start. These calls should be parallelized where Foundry Functions support it. If AIP rate limits restrict throughput, the system reduces the candidate pool size (minimum: 1.2x the assessment length) or falls back to canonical options for a portion of candidates. The expected total pre-generation time is under 60 seconds with parallelization. Each QuestionInstance stores the rendered question text, the user-specific options, the correct option index, and the AIP prompt version used to generate them.

**D-005: Three-regime adaptive selection.** The question selection algorithm operates in three regimes depending on available data. Each regime is fully specified in Section 4 with its scoring formula, parameter estimation method, transition trigger, transition math, and data requirements. Regime state is tracked at the question level. Selection signals are tracked at the question-instance level. Regime transitions are auditable.

**D-006: Error handling and idempotency.** All write operations (creating Responses, updating Attempt counters, updating Question stats) are designed to be idempotent. Re-running a pipeline or re-processing a completion event produces the same result. If SubmitResponse fails mid-execution, the Attempt state does not advance past the number of actually-persisted Responses. AIP failures fall back to canonical options rather than blocking the assessment. The nightly Question Stats Pipeline recounts all statistics from the responses dataset (source of truth), correcting any inconsistencies from in-flight updates. See Section 8.8 for details.

### 0.4 Intentionally Unused Source Fields

The source JSON contains two fields that are ingested and stored but not consumed by any Phase 1 selection formula, pipeline, or dashboard logic:

- **relevance** (1-5): Stored on the Question object. Could be incorporated into the metadata_score formula in a future iteration (e.g., as a weight multiplier on the composite score) or used by AIP chat for natural-language queries about question quality. Not used in Phase 1 to keep the selection formula tractable and tunable with fewer dimensions.

- **estimatedMinutes**: Stored on the Question object. Could be used in future for time-budgeting within an assessment (e.g., if a user has 5 minutes remaining, select only questions with estimatedMinutes <= 5). Not used in Phase 1 because time enforcement operates at the assessment level, not per-question.

These fields are available for AIP chat queries (e.g., "Which questions have low relevance?") and for future selection formula enhancements.

---

## 1. Ontology Design

### 1.1 Object Types

#### 1.1.1 Question

Backed by: `question_bank_active`

| Property | Type | Required | Notes |
|---|---|---|---|
| questionId | String | Yes | Primary key |
| topic | String | Yes | |
| topicNumber | Integer | Yes | |
| parts | String | No | Sub-topic or concept area |
| bloomLevel | String | Yes | Bloom's taxonomy level |
| bloomRank | Integer | Yes | Numeric rank: Knowledge=1, Comprehension=2, Application=3, Analysis=4, Synthesis=5, Evaluation=6 |
| questionText | String | Yes | Original question text |
| answer | String | Yes | Model answer (used by AIP for option generation) |
| justification | String | No | Why this question matters (shown in formative mode) |
| canonicalOptions | String array | Yes | Baseline MCQ options generated at ingest |
| canonicalCorrectIndex | Integer | Yes | Correct index in canonical options |
| difficultyPrior | Double | Yes | From source JSON (1-5 scale) |
| genericity | Integer | Yes | From source JSON (1-5). Higher = tests broader concepts. Used in Regime 1 selection |
| relevance | Integer | Yes | From source JSON (1-5). Stored but not used in Phase 1 selection (see Section 0.4) |
| effectivenessPrior | Double | Yes | From source JSON. Initial discrimination proxy. Used to initialize aParam |
| estimatedMinutes | Double | No | From source JSON. Stored but not used in Phase 1 (see Section 0.4) |
| aParam | Double | Yes | IRT discrimination. Current value (evolves through regimes) |
| aParamInitial | Double | Yes | IRT discrimination at initialization. Retained for audit |
| bParam | Double | Yes | IRT difficulty. Current value (evolves through regimes) |
| bParamInitial | Double | Yes | IRT difficulty at initialization. Retained for audit |
| origin | String | Yes | One of: base, generated |
| responseCount | Long | Yes | Total responses received (drives regime selection) |
| exposureCount | Long | Yes | Times served |
| correctCount | Long | Yes | Times answered correctly |
| calibrationRegime | String | Yes | Current regime: metadata, blended, calibrated |
| regimeEnteredAt | Timestamp | Yes | When the question entered its current regime |
| regimePriorBParam | Double | No | b_param value before the most recent regime transition. Null if no transition has occurred |
| regimePriorAParam | Double | No | a_param value before the most recent regime transition |
| bParamEmpirical | Double | No | Latest empirically computed b. Null if responseCount < 5 |
| aParamEmpirical | Double | No | Latest empirically computed a. Null if responseCount < 20 |
| isActive | Boolean | Yes | Whether available for selection |
| createdAt | Timestamp | Yes | |

**[AC-OBJ-001]** Every question in question_bank_active is accessible as a Question object. calibrationRegime is "metadata" when responseCount < 5, "blended" when 5 <= responseCount < 20, "calibrated" when responseCount >= 20. Both current and initial parameter values are stored. Regime transition history is auditable via regimeEnteredAt, regimePriorBParam, and regimePriorAParam.

#### 1.1.2 QuestionInstance

Backed by: `question_instances`

This is the user-level record of exactly what a specific user saw. Each instance stores AIP-generated options unique to this user and attempt.

| Property | Type | Required | Notes |
|---|---|---|---|
| instanceId | String | Yes | Primary key (UUID) |
| renderedQuestionText | String | Yes | AIP-rephrased question text for this user |
| renderedOptions | String array | Yes | AIP-generated options for this user. Unique per user per attempt |
| correctOptionIndex | Integer | Yes | Index of correct option in renderedOptions |
| thetaAtServe | Double | Yes | User's theta when this question was served |
| thetaConfidenceAtServe | String | Yes | User's thetaConfidence when served |
| sequenceNumber | Integer | Yes | Position in the assessment (1-indexed) |
| selectionRegime | String | Yes | Which regime scored this question: metadata, blended, calibrated |
| selectionScoreComposite | Double | Yes | Final composite score used for ranking |
| scoreInfoProxy | Double | Yes | The information_proxy or irt_information sub-score |
| scoreGenericity | Double | Yes | The genericity_score sub-score (0.0-1.0, or 1.0 if neutral) |
| scoreBloom | Double | Yes | The bloom_appropriateness sub-score (0.0-1.0) |
| scoreFreshness | Double | Yes | The freshness sub-score |
| scoreBlendAlpha | Double | No | The alpha blend factor. Null for Regime 1 and 3. Value in [0,1] for Regime 2 |
| aipPromptVersion | String | Yes | Identifier of the AIP prompt template used (see Section 5.5) |
| servedAt | Timestamp | Yes | |

**[AC-OBJ-002]** Every question displayed to a user has exactly one QuestionInstance record written before the question is rendered. The renderedOptions are generated by AIP specifically for this user and stored here. Replaying an attempt retrieves identical question text and options. All four selection sub-scores are recorded.

#### 1.1.3 Attempt

Backed by: `attempts`

| Property | Type | Required | Notes |
|---|---|---|---|
| attemptId | String | Yes | Primary key |
| mode | String | Yes | summative, formative, practice |
| configNumQuestions | Integer | Yes | |
| configDurationMinutes | Integer | No | Null for formative/practice if untimed |
| startTime | Timestamp | Yes | |
| endTime | Timestamp | No | Null until completed |
| numQuestionsAnswered | Integer | Yes | |
| numCorrect | Integer | Yes | |
| finalTheta | Double | No | Null until completed |
| thetaConfidence | String | No | Confidence at completion |
| percentile | Double | No | Null until completed or if cohort < 5 |
| isComplete | Boolean | Yes | |
| regimeMixSummary | String | No | JSON: {"metadata": N, "blended": N, "calibrated": N} |

**[AC-OBJ-003]** Exactly one Attempt record per assessment session. regimeMixSummary is populated at completion.

#### 1.1.4 Response

Backed by: `responses`

| Property | Type | Required | Notes |
|---|---|---|---|
| responseId | String | Yes | Primary key |
| selectedOptionIndex | Integer | Yes | |
| isCorrect | Boolean | Yes | |
| responseTimeMs | Long | Yes | |
| timeToFirstActionMs | Long | No | |
| numOptionChanges | Integer | Yes | |
| submittedAt | Timestamp | Yes | |

**[AC-OBJ-004]** Exactly one Response per QuestionInstance per Attempt.

#### 1.1.5 UserProfile

Backed by: `user_features`

| Property | Type | Required | Notes |
|---|---|---|---|
| userId | String | Yes | Primary key |
| globalTheta | Double | Yes | |
| thetaConfidence | String | Yes | prior, low, moderate, high |
| avgResponseTimeMs | Double | No | |
| totalAttempts | Integer | Yes | Summative only |
| totalPracticeAttempts | Integer | Yes | |
| totalQuestionsAnswered | Integer | Yes | |
| lastAttemptDate | Timestamp | No | |
| updatedAt | Timestamp | Yes | |

**[AC-OBJ-005]** One UserProfile per user. thetaConfidence: "prior" (totalQuestionsAnswered=0), "low" (1-9), "moderate" (10-29), "high" (30+).

#### 1.1.6 TopicMastery

Backed by: `topic_mastery`

| Property | Type | Required | Notes |
|---|---|---|---|
| id | String | Yes | Composite: userId_topicNumber |
| userId | String | Yes | |
| topic | String | Yes | |
| topicNumber | Integer | Yes | |
| topicTheta | Double | Yes | |
| accuracy | Double | Yes | |
| questionCount | Integer | Yes | |
| isWeak | Boolean | Yes | |
| updatedAt | Timestamp | Yes | |

**[AC-OBJ-006]** One TopicMastery per user per topic. isWeak is true when topicTheta < (globalTheta - 1.0) or topicTheta < -0.5.

#### 1.1.7 AssessmentConfig

Backed by: `assessment_configs`

| Property | Type | Required | Notes |
|---|---|---|---|
| configId | String | Yes | Primary key |
| label | String | Yes | |
| mode | String | Yes | |
| numQuestions | Integer | Yes | Default 45 |
| durationMinutes | Integer | No | Default 30 for summative |
| topicWeights | String (JSON) | No | {topicNumber: weight}. See balanced distribution definition below |
| difficultyRange | String (JSON) | No | {"min": x, "max": y} |
| isDefault | Boolean | Yes | |

**Balanced topic distribution:** When topicWeights is null (the default), the system uses a balanced distribution defined as: each topic receives equal weight, regardless of how many questions exist for that topic in the bank. For example, if there are 5 topics and 45 questions in the assessment, each topic is targeted for 9 questions (45/5). If a topic has fewer than 9 available questions, the system uses all available questions for that topic and redistributes the remainder proportionally across topics with surplus questions. The 20% tolerance in the selection algorithm (Section 4.2, Step 2) accommodates minor imbalances.

**[AC-OBJ-007]** At least one default config per mode. When topicWeights is null, the system applies the balanced distribution defined above.

### 1.2 Link Types

| Link | From | To | Cardinality |
|---|---|---|---|
| attempt_takenBy | Attempt | UserProfile | Many-to-One |
| attempt_hasInstances | Attempt | QuestionInstance | One-to-Many |
| instance_derivedFrom | QuestionInstance | Question | Many-to-One |
| instance_hasResponse | QuestionInstance | Response | One-to-One |
| response_partOfAttempt | Response | Attempt | Many-to-One |
| user_hasMastery | UserProfile | TopicMastery | One-to-Many |
| attempt_usedConfig | Attempt | AssessmentConfig | Many-to-One |

**[AC-LINK-001]** All links navigable in Workshop.

### 1.3 Action Types

#### 1.3.1 StartAssessment

**Input:** userId (String), configId (String)

**Behavior:**
1. **Concurrent assessment guard:** Check whether this user has an existing Attempt with isComplete=false. If one exists and its start_time is less than 24 hours ago, reject the request and return the existing attempt_id so the user can resume. If one exists but is older than 24 hours, mark it as complete with the data recorded so far (partial completion) and proceed.
2. Create an Attempt object with mode from config, isComplete=false.
3. Load the user's current theta and thetaConfidence (or theta=0.0, thetaConfidence="prior" if new).
4. Select a candidate pool of questions (2x configNumQuestions) using the regime-aware selection (Section 4).
5. For each candidate, call AIP to generate a user-specific variant. The prompt includes the user's current theta and thetaConfidence so AIP can calibrate distractor difficulty. Write each as a QuestionInstance with all sub-scores recorded. If AIP rate limits are hit mid-generation, switch remaining candidates to canonical options with aipPromptVersion="fallback".
6. Select the first question from the pool.
7. Return the first QuestionInstance.

**[AC-ACT-001]** StartAssessment rejects if the user has an in-progress assessment less than 24 hours old (returns the existing attempt_id). It produces a new Attempt and at least one QuestionInstance. The QuestionInstance's renderedOptions are unique to this user. All selection sub-scores are populated.

#### 1.3.2 SubmitResponse

**Input:** attemptId (String), instanceId (String), selectedOptionIndex (Integer), responseTimeMs (Long), timeToFirstActionMs (Long, nullable), numOptionChanges (Integer)

**Behavior:**
1. Validate: attempt not complete, instance belongs to attempt, no prior response for this instance.
2. Validate time: if mode is summative and (now - Attempt.startTime) > configDurationMinutes * 60 + 5 seconds, reject as expired.
3. Determine correctness from the instance's correctOptionIndex (not the base question's canonical index, since options may differ).
4. Write a Response object.
5. Update running theta using regime-weighted EAP (Section 4.5).
6. Update Attempt (increment counters).
7. If formative: include answer and justification in return payload.
8. Check termination: numQuestionsAnswered >= configNumQuestions, or time expired (summative).
9. If not terminated: select next question from pre-generated pool. Return next QuestionInstance.
10. If terminated: call CompleteAssessment internally.

**[AC-ACT-002]** Correctness is determined against the instance's correctOptionIndex (user-specific options), not the base question's canonical options. Expired submissions are rejected. Response within 1 second.

#### 1.3.3 CompleteAssessment

**Input:** attemptId (String)

**Behavior:**
1. Compute final theta.
2. Compute percentile (null if cohort < 5).
3. Compute regimeMixSummary: count of QuestionInstances by selectionRegime for this attempt.
4. Mark Attempt complete.
5. Trigger async update to UserProfile and TopicMastery.
6. Increment responseCount, exposureCount, correctCount on each served Question. Check regime transition thresholds (Section 4.6). If a question crosses a threshold, update its calibrationRegime, recompute its parameters, and record the transition metadata. These updates are best-effort; the nightly Question Stats Pipeline (Section 6.3) is the authoritative source and will correct any inconsistencies.

**[AC-ACT-003]** After completion: Attempt.isComplete=true, regimeMixSummary populated, finalTheta non-null. Any question that crossed a regime threshold during this completion has its calibrationRegime, regimeEnteredAt, regimePriorAParam, and regimePriorBParam updated.

#### 1.3.4 StartPractice

**Input:** userId (String)

**Behavior:**
1. Read TopicMastery where isWeak=true.
2. If none, return "no practice needed."
3. Create dynamic config: mode=practice, numQuestions=15, no time limit, topicWeights 80% weak topics.
4. Delegate to StartAssessment.

**[AC-ACT-004]** Practice contains at least 80% weak-topic questions.

---

## 2. Data Architecture

### 2.1 Datasets

All datasets are created with explicit schemas. No inference.

The schemas follow the Ontology object definitions in Section 1.1. The following sections specify only the dataset-specific details not already covered in the Ontology.

#### 2.1.1 question_bank_base

Immutable copy of the ingested JSON. All source fields preserved, including relevance and estimatedMinutes.

**[AC-DS-001]** Row count equals source JSON array length. All non-nullable columns populated. Numeric fields correctly typed.

#### 2.1.2 question_bank_active

Operational question bank. All columns from Question object type (Section 1.1.1), including: aParamInitial, bParamInitial, regimeEnteredAt, regimePriorBParam, regimePriorAParam, bParamEmpirical, aParamEmpirical.

**[AC-DS-002]** After ingestion: every row has canonicalOptions with 4 elements. a_param varies (initialized from effectiveness_prior). b_param varies (initialized from difficulty_prior). calibration_regime="metadata" for all rows. a_param_initial = a_param. b_param_initial = b_param. regime_entered_at = ingestion timestamp.

#### 2.1.3 question_instances

All columns from QuestionInstance object type (Section 1.1.2), including all four selection sub-scores, scoreBlendAlpha, and aipPromptVersion.

**[AC-DS-003]** Written before display. renderedOptions are user-specific (generated by AIP for this user-attempt combination). All sub-scores populated. aipPromptVersion indicates which prompt was used or "fallback" if canonical options were used.

#### 2.1.4 responses

All columns from Response object type (Section 1.1.4).

**[AC-DS-004]** One response per instance per attempt. response_time_ms > 0.

#### 2.1.5 attempts

All columns from Attempt object type (Section 1.1.3), including regimeMixSummary.

**[AC-DS-005]** One row per session. regimeMixSummary populated at completion.

#### 2.1.6 user_features

All columns from UserProfile object type (Section 1.1.5).

**[AC-DS-006]** One row per user. theta_confidence consistent with total_questions_answered.

#### 2.1.7 topic_mastery

All columns from TopicMastery object type (Section 1.1.6).

**[AC-DS-007]** One row per user-topic pair.

#### 2.1.8 assessment_configs

All columns from AssessmentConfig object type (Section 1.1.7).

**[AC-DS-008]** At least one default config per mode.

### 2.2 Ingestion Pipeline

**Steps:**

1. **Parse and validate**: Read JSON. Validate required fields including genericity, relevance, effectiveness. If genericity or relevance is null for a record, default to 3 (neutral) and log a warning. If effectiveness is null, default to 3.0. Write to question_bank_base.

2. **Bloom rank computation**: Map bloom_level string to bloom_rank integer: Knowledge=1, Comprehension=2, Application=3, Analysis=4, Synthesis=5, Evaluation=6. If bloom_level does not match any of these, default to 3 (Application) and log a warning.

3. **MCQ generation via AIP**: For each question, call AIP (prompt in Section 5.1). Output: canonicalOptions (4 elements), canonicalCorrectIndex.

4. **IRT initialization from metadata**:
   - b_param = (difficulty_prior - 3) * 1.0. Maps the 1-5 scale to [-2.0, +2.0].
   - a_param = 0.3 + (effectiveness_prior / 5.0) * 1.4. Maps to [0.58, 1.7].
   - Store these as both the current values (a_param, b_param) and the initial values (a_param_initial, b_param_initial).

5. **Write to question_bank_active**: Set response_count=0, exposure_count=0, correct_count=0, calibration_regime="metadata", regime_entered_at=now, b_param_empirical=null, a_param_empirical=null, regime_prior_b_param=null, regime_prior_a_param=null, is_active=true.

**[AC-PIPE-001]** After ingestion: row count matches, options valid, a_param varies, b_param varies, bloom_rank populated, all regime audit fields initialized.

---

## 3. Workshop Application

### 3.1 Application Structure

**Views:**
1. Assessment Launcher (home page)
2. Assessment View (question-by-question interaction)
3. Results View (post-assessment summary)
4. Practice Launcher
5. Individual Dashboard
6. Cohort Dashboard
7. Question Effectiveness Dashboard
8. Admin: Question Bank Browser
9. Admin: Assessment Configuration

### 3.2 Assessment Launcher

- Config selector dropdown (filtered by mode).
- Config details display.
- "Start Assessment" button triggers StartAssessment.
- If the user has an in-progress assessment (isComplete=false, less than 24 hours old), display a "Resume Assessment" button instead of "Start Assessment", linked to the existing attempt.
- Past attempts table showing: date, mode, score, theta, thetaConfidence, regimeMixSummary (as a compact string like "M:12 B:18 C:15" for metadata/blended/calibrated).

**[AC-UI-001]** Loads within 3 seconds. Past attempts show regime mix per attempt. In-progress assessments show a resume option.

### 3.3 Assessment View

- **Top bar**: Timer (countdown for summative, elapsed for formative, hidden for untimed), progress indicator, mode badge.
- **Question area**: Rendered question text from the current QuestionInstance (user-specific AIP-generated text).
- **Options area**: Radio buttons for the renderedOptions (user-specific AIP-generated options). These are unique to this user and this attempt.
- **Action bar**: "Submit" (disabled until selection), "Next" (formative only, after submit).
- **Feedback panel** (formative/practice only): Correctness, model answer, justification.

**Correctness determination**: When the user selects option index N and submits, correctness is evaluated against the QuestionInstance's correctOptionIndex, not the base question's canonicalCorrectIndex. The user-specific options may have a different ordering or different distractor content than the canonical set.

**Assessment resumption**: If the user navigates away and returns, the Assessment View loads by querying the in-progress Attempt and its linked QuestionInstances. The next question to display is determined by finding the QuestionInstance with sequenceNumber = numQuestionsAnswered + 1. All previously answered questions are skipped.

**[AC-UI-002]** Questions render one at a time. User cannot go back. Options shown are the user-specific renderedOptions from the QuestionInstance. Correctness uses the instance-level correct index. Resumption loads the correct next question.

### 3.4 Results View

- Score, theta with confidence label, percentile (or "Insufficient cohort data").
- Regime mix summary for this attempt (e.g., "34 questions used statistical selection, 11 used metadata-based selection").
- Topic breakdown bar chart.
- Answer review panel.
- "Start Practice" button if weak topics exist.

**[AC-UI-003]** Results display within 5 seconds. Low-confidence theta distinguished. Regime mix visible.

### 3.5 Practice Launcher

- Weak topics list, recommended practice length, "Start Practice" button.

**[AC-UI-004]** If no weak topics, message displayed, no start button.

---

## 4. Three-Regime Adaptive Selection System

This section is the complete technical specification of the regime system. It covers the scoring formula, parameter estimation method, transition conditions, transition math, and data requirements for each regime.

### 4.0 Design Rationale

The fundamental problem: IRT-based adaptive testing requires calibrated item parameters (a, b) derived from real response data. A new system has no response data. If we wait for calibration before behaving adaptively, the first N users receive a suboptimal experience (random or fixed-order question selection).

The solution: use the metadata already present in the source question bank (genericity, Bloom level, effectiveness rating, difficulty rating) as proxies for IRT parameters. These metadata signals are not as precise as calibrated IRT parameters, but they are far more informative than random selection. As response data accumulates, the system smoothly transitions from metadata-driven to statistically-driven selection on a per-question basis.

### 4.1 Regime Definitions

#### Regime 1: Metadata-Driven

**Applicable when:** Question.responseCount < 5

**Parameter state:** a_param and b_param are at their initial values derived from source metadata during ingestion. No empirical adjustment has occurred.

**Available signals for scoring:**
- effectiveness_prior (mapped to a_param_initial): the question author's estimate of how well this question discriminates between knowledgeable and unknowledgeable users.
- difficulty_prior (mapped to b_param_initial): the author's estimate of question difficulty.
- genericity (1-5): how broadly the question tests a concept. A genericity of 5 means the question tests a fundamental, widely-applicable concept. A genericity of 1 means it tests a narrow, specific detail.
- bloom_rank (1-6): the cognitive level being tested.
- exposure_count: how many times the question has been served (for freshness).

**Scoring formula:**

```
metadata_score(Q, theta, thetaConfidence) =
    0.40 * information_proxy(Q, theta)
  + 0.25 * genericity_score(Q, thetaConfidence)
  + 0.20 * bloom_appropriateness(Q, theta)
  + 0.15 * freshness(Q)
```

**Sub-function definitions:**

**information_proxy(Q, theta):**
Uses the metadata-initialized a and b to compute Fisher information.
```
P = 1 / (1 + exp(-Q.a_param * (theta - Q.b_param)))
I = Q.a_param^2 * P * (1 - P)
```
Normalize to [0, 1]: divide by the maximum possible I (which occurs when P=0.5): I_max = Q.a_param^2 * 0.25. So:
```
information_proxy = I / I_max = 4 * P * (1 - P)
```
This equals 1.0 when Q.b_param = theta (question difficulty matches user ability) and decreases as the mismatch grows. It is independent of a_param after normalization, which is appropriate because a_param is poorly calibrated in Regime 1.

**genericity_score(Q, thetaConfidence):**
```
if thetaConfidence in ("prior", "low"):
    genericity_score = Q.genericity / 5.0
else:
    genericity_score = 1.0  # neutral, does not influence ranking
```
Rationale: when we know little about the user, generic questions that test broad concepts are better initial differentiators. A user who fails a generic question about a topic's core concept is unlikely to succeed on a specific detail question in the same topic. Once we have a moderate or better theta estimate, genericity becomes irrelevant and should not bias selection.

**bloom_appropriateness(Q, theta):**
```
expected_bloom = map_theta_to_bloom(theta)
bloom_appropriateness = 1.0 - abs(Q.bloom_rank - expected_bloom) / 6.0
```
Where map_theta_to_bloom is:
```
if theta < -1.5:  expected_bloom = 1.0  (Knowledge)
elif theta < -0.5: expected_bloom = 2.0  (Comprehension)
elif theta < 0.5:  expected_bloom = 3.0  (Application)
elif theta < 1.5:  expected_bloom = 4.0  (Analysis)
else:              expected_bloom = 5.0  (Synthesis/Evaluation)
```
Rationale: Bloom's taxonomy provides a second dimension of question difficulty orthogonal to the IRT difficulty parameter. Lower-Bloom questions (recall, comprehension) are appropriate for establishing baseline competence. Higher-Bloom questions (analysis, evaluation) are appropriate for differentiating advanced users.

**freshness(Q):**
```
freshness = 1.0 / (1.0 + Q.exposure_count)
```
Ranges from 1.0 (never served) to near 0 (heavily served). Prevents any single question from dominating early assessments when the pool is small.

#### Regime 2: Blended

**Applicable when:** 5 <= Question.responseCount < 20

**Parameter state:** b_param has been partially updated using empirical data (observed correct rate). a_param retains its initial value (insufficient data for discrimination estimation). Both empirical and prior values are available.

**Available signals:** All Regime 1 signals plus:
- b_param_empirical: difficulty computed from observed responses.
- observed correct_rate: correct_count / response_count.

**Scoring formula:**

```
blended_score(Q, theta, thetaConfidence) =
    alpha * irt_score(Q, theta)
  + (1 - alpha) * metadata_score(Q, theta, thetaConfidence)
```

Where:
```
alpha = (Q.response_count - 5) / 15.0
```
alpha ranges linearly from 0.0 at responseCount=5 to 1.0 at responseCount=20.

**irt_score(Q, theta):** Uses blended parameters:
```
b_effective = (1 - alpha) * Q.b_param_initial + alpha * Q.b_param_empirical
a_effective = Q.a_param  # still metadata-initialized; no empirical a yet

P = 1 / (1 + exp(-a_effective * (theta - b_effective)))
I = a_effective^2 * P * (1 - P)
irt_score = I / (a_effective^2 * 0.25)  # normalized to [0, 1]
```

**metadata_score(Q, theta, thetaConfidence):** Same formula as Regime 1.

The alpha blend factor is recorded on the QuestionInstance as scoreBlendAlpha for audit.

**Continuity at boundaries:**
- At responseCount=5, alpha=0: blended_score = metadata_score. No discontinuity with Regime 1.
- At responseCount=20, alpha=1: blended_score = irt_score using b_effective = b_param_empirical. This is continuous with Regime 3 because Regime 3 also uses the empirically-derived b_param.

#### Regime 3: Calibrated

**Applicable when:** Question.responseCount >= 20

**Parameter state:** Both a_param and b_param have been empirically estimated. The parameters reflect real user performance data.

**Available signals:** Calibrated IRT parameters, exposure count.

**Scoring formula:**

```
calibrated_score(Q, theta) =
    0.85 * irt_information(Q, theta)
  + 0.15 * freshness(Q)
```

**irt_information(Q, theta):**
```
P = 1 / (1 + exp(-Q.a_param * (theta - Q.b_param)))
I = Q.a_param^2 * P * (1 - P)
irt_information = I / I_max_global
```
Where I_max_global is the maximum Fisher information across all Regime 3 questions at any theta. In practice, I_max_global = max(Q.a_param^2 * 0.25) for all calibrated questions. This value is computed once at the start of each assessment and cached.

**freshness(Q):** Same as Regime 1.

Genericity and Bloom appropriateness are not used in Regime 3 because calibrated IRT parameters subsume these signals.

### 4.2 Question Selection Pipeline

This runs inside SubmitResponse (and StartAssessment for the first question).

**Input:** theta, thetaConfidence, served instance IDs, config, candidate pool.

**Step 1: Filter.** Remove already-served candidates.

**Step 2: Topic balance.** Compute served topic distribution against the target weights (from config topicWeights, or the balanced distribution defined in Section 1.1.7 if null). If any topic is underrepresented by more than 20% of its target weight, restrict candidates to that topic.

**Step 3: Score.** For each candidate, compute the score using the appropriate regime formula. Record all four sub-scores and the blend alpha (if applicable).

**Step 4: Exposure control.** If the top candidate's base question has exposure_count above the 90th percentile of all active questions, skip it.

**Step 5: Select.** Return the top candidate. Write selection metadata to the QuestionInstance.

**[AC-ENG-001]** No duplicates within an attempt. Topic distribution within 20% tolerance of target (balanced or configured). All sub-scores recorded. Exposure control prevents overuse.

### 4.3 Per-User Option Generation

When a QuestionInstance is created during StartAssessment's pre-generation step, AIP generates options specific to this user's context.

**AIP input for option generation:**
```
{
  "question_text": Q.questionText,
  "model_answer": Q.answer,
  "topic": Q.topic,
  "bloom_level": Q.bloomLevel,
  "difficulty": Q.difficultyPrior,
  "user_theta": user.globalTheta,
  "user_theta_confidence": user.thetaConfidence
}
```

**AIP prompt (see Section 5.2 for full text):**
The prompt instructs AIP to:
- Generate the correct option from the model answer.
- Generate 3 distractors calibrated to the question's Bloom level and difficulty.
- If user theta is high (> 1.0) and theta confidence is moderate or high, make distractors more subtle (common advanced misconceptions). If user theta is low (< -1.0), make distractors more distinct (common beginner mistakes).
- Randomize the position of the correct option.

**Storage:** The generated options are stored on the QuestionInstance as renderedOptions with the corresponding correctOptionIndex. These are immutable once written. If the same user retakes the assessment, new instances are generated with fresh AIP calls, producing different options.

**Fallback:** If AIP fails for a specific instance, fall back to the base question's canonicalOptions and canonicalCorrectIndex. Record aipPromptVersion as "fallback" on the instance.

**[AC-ENG-002]** Each QuestionInstance has user-specific options. Two users taking the same assessment see different option sets for the same base question. The same user retaking the assessment sees different options. Correctness is always evaluated against the instance-level correctOptionIndex.

### 4.4 Cold-Start Sequencing Behavior

For a first-time user (thetaConfidence = "prior"), the Regime 1 scoring formula produces the following observable behavior, not by hardcoding but as a natural consequence of the weights:

**First 5-10 questions:**
- genericity_score is active (weight 0.25) and favors high-genericity questions (score approaches 1.0 for genericity=5).
- bloom_appropriateness is active (weight 0.20) and favors bloom_rank near 3 (Application) since theta starts at 0.
- information_proxy favors questions with b_param near 0 (medium difficulty).
- Combined effect: the system serves broad, medium-difficulty, application-level questions first.

**After 10 questions:**
- thetaConfidence transitions to "low" (if total_questions_answered was 0 before this attempt) or "moderate" (if this is a second attempt). At "moderate", genericity_score becomes neutral (1.0 for all questions). Selection is now driven by information_proxy, bloom_appropriateness, and freshness.
- theta has moved from 0 to reflect early performance. If the user answered most questions correctly, theta is positive and bloom_appropriateness shifts toward higher Bloom levels. If the user struggled, theta is negative and bloom_appropriateness shifts toward lower levels.

**[AC-ENG-003]** For a new user, first 5 questions have avg genericity >= 3.5 and avg bloom_rank <= 3.5, assuming the bank has sufficient variety.

### 4.5 Theta Estimation with Regime-Weighted Likelihood

**Method:** Expected A Posteriori (EAP) on a discrete grid.

**Grid:** theta values from -4.0 to +4.0 in steps of 0.1 (81 points).

**For each grid point theta_k:**

Compute the log-likelihood of the observed response pattern:

```
log_L(theta_k) = sum over all answered questions i of:
    w_i * [y_i * log(P_i(theta_k)) + (1 - y_i) * log(1 - P_i(theta_k))]
```

Where:
- y_i = 1 if correct, 0 if incorrect.
- P_i(theta_k) = 1 / (1 + exp(-a_i * (theta_k - b_i)))
- a_i, b_i are the question's current a_param and b_param.
- w_i is the regime weight:
  - If question i's calibrationRegime is "metadata": w_i = 0.5
  - If "blended": w_i = 0.5 + 0.5 * alpha_i (where alpha_i = (responseCount_i - 5) / 15)
  - If "calibrated": w_i = 1.0

**Prior:** log_prior(theta_k) = -0.5 * theta_k^2 (standard normal).

**Posterior:** log_posterior(theta_k) = log_L(theta_k) + log_prior(theta_k).

Convert to probabilities: posterior_k = exp(log_posterior(theta_k)) / sum(exp(log_posterior(theta_j))).

**Theta estimate:** theta_hat = sum(theta_k * posterior_k).

**Clamping:** Clamp theta_hat to [-3.0, +3.0].

**[AC-ENG-004]** Theta is finite for any response pattern. Regime 1 questions contribute at half weight. Regime 3 at full weight. Regime 2 at intermediate weight proportional to its blend factor.

### 4.6 Regime Transition Mechanics

Regime transitions are triggered by changes to a question's responseCount. The transitions occur during the Question Stats Pipeline (Section 6.3) as the authoritative mechanism, and also during CompleteAssessment (Section 1.3.3) as a best-effort immediate update.

#### Transition: Metadata to Blended

**Trigger:** Question.responseCount crosses from below 5 to 5 or above.

**Parameter updates:**

Compute b_param_empirical:
```
p_correct = Q.correct_count / Q.response_count
# Clamp p_correct to avoid log(0):
p_correct = max(0.05, min(0.95, p_correct))
b_param_empirical = -log(p_correct / (1 - p_correct))
```
Note: this is the logit of the incorrect rate, which maps to the IRT difficulty parameter. A question answered correctly by 80% of users (p_correct=0.8) gets b_param_empirical = -log(0.8/0.2) = -1.39 (easy). A question answered correctly by 20% gets b_param_empirical = +1.39 (hard).

Blend b_param:
```
alpha_transition = (Q.response_count - 5) / 15.0  # will be 0.0 at exactly 5 responses
b_param_new = (1 - alpha_transition) * Q.b_param_initial + alpha_transition * Q.b_param_empirical
# Clamp: b_param_new = max(-4.0, min(4.0, b_param_new))
```

a_param: unchanged (retain metadata-initialized value). Discrimination estimation requires knowing the respondents' theta values and at least 20 data points for a meaningful correlation.

**Record transition:**
```
Q.regime_prior_b_param = Q.b_param  # value before transition
Q.regime_prior_a_param = Q.a_param  # value before transition (unchanged, but recorded)
Q.b_param = b_param_new
Q.b_param_empirical = b_param_empirical
Q.calibration_regime = "blended"
Q.regime_entered_at = now
```

**[AC-TRANS-001]** At exactly 5 responses, b_param is updated as a weighted blend favoring the prior (alpha=0 means 100% prior at the threshold, smoothly shifting to empirical). The transition is recorded with prior values preserved.

#### Transition: Blended to Calibrated

**Trigger:** Question.responseCount crosses from below 20 to 20 or above.

**Parameter updates:**

Recompute b_param_empirical using all responses (same formula as above, but with more data points so it is more stable).

Compute a_param_empirical:

This requires the respondents' theta values. The pipeline must join the question's responses with the respondents' user_features to get each respondent's global_theta at the time of response.

```
For all responses to this question:
    theta_values = [respondent.global_theta for each response]
    correctness_values = [1 if correct, 0 if not for each response]

r_pb = point_biserial_correlation(correctness_values, theta_values)
```

The point-biserial correlation measures how well the question separates high-theta from low-theta users. It ranges from -1 to +1. We convert to the IRT a_param scale:

```
a_param_empirical = r_pb * 2.5
# Clamp: a_param_empirical = max(0.3, min(3.0, a_param_empirical))
```

The scaling factor of 2.5 maps a moderate correlation (r=0.4, which is typical for a decent question) to a_param approximately 1.0, which is a standard discrimination value in 2PL models.

If the point-biserial cannot be computed (e.g., all responses are correct or all are incorrect, or all respondents have the same theta), retain a_param_initial and log a warning.

Set b_param:
```
b_param_new = b_param_empirical  # prior is now fully overridden
# Clamp: max(-4.0, min(4.0, b_param_new))
```

**Record transition:**
```
Q.regime_prior_b_param = Q.b_param
Q.regime_prior_a_param = Q.a_param
Q.b_param = b_param_new
Q.a_param = a_param_empirical (or retain if computation failed)
Q.b_param_empirical = b_param_empirical
Q.a_param_empirical = a_param_empirical (or null if computation failed)
Q.calibration_regime = "calibrated"
Q.regime_entered_at = now
```

**[AC-TRANS-002]** At 20 responses, b_param is fully empirical. a_param is computed from point-biserial correlation with respondent thetas. Both are clamped. Prior values are preserved for audit.

#### Post-Calibration Updates

**Trigger:** Question is already in "calibrated" regime and receives additional responses.

**Behavior:** On each nightly pipeline run, recompute b_param_empirical and a_param_empirical using all accumulated responses. Update Q.b_param and Q.a_param. Do not change calibration_regime or regime_entered_at (the question remains "calibrated"). Do update b_param_empirical and a_param_empirical.

**[AC-TRANS-003]** Calibrated questions continue to refine their parameters as more data arrives. No regime change occurs.

### 4.7 Regime Tracking Summary

| What is tracked | Where it is stored | Granularity | Purpose |
|---|---|---|---|
| Current calibration regime | Question.calibrationRegime | Per question | Determines which scoring formula is used when this question is a candidate |
| When current regime was entered | Question.regimeEnteredAt | Per question | Audit trail for regime transitions |
| Parameter values before last transition | Question.regimePriorAParam, regimePriorBParam | Per question | Audit: what changed during the transition |
| Current vs initial parameters | Question.aParam vs aParamInitial, bParam vs bParamInitial | Per question | Audit: total drift from initialization |
| Empirical parameter values | Question.bParamEmpirical, aParamEmpirical | Per question | The latest data-derived values (may differ from blended aParam/bParam in Regime 2) |
| Which regime scored this question for a specific user | QuestionInstance.selectionRegime | Per user-question-attempt | Audit: what drove the selection for this specific user |
| All sub-scores used in selection | QuestionInstance.scoreInfoProxy, scoreGenericity, scoreBloom, scoreFreshness | Per user-question-attempt | Audit: the exact contribution of each signal |
| Blend factor | QuestionInstance.scoreBlendAlpha | Per user-question-attempt (Regime 2 only) | Audit: how much weight went to IRT vs metadata |
| Composite score | QuestionInstance.selectionScoreComposite | Per user-question-attempt | Audit: the final ranking score |
| Regime mix for an attempt | Attempt.regimeMixSummary | Per attempt | Summary view: how data-mature was the assessment for this user |

**[AC-TRACK-001]** The combination of Question-level regime fields and QuestionInstance-level selection fields provides a complete audit trail. An analyst can determine: (a) what regime each question is in now, (b) when and how it transitioned, (c) for any specific assessment, which regime was used for each question and what sub-scores drove the selection, (d) for the overall assessment, what proportion of questions used each regime.

---

## 5. AIP Integration

### 5.1 MCQ Generation Prompt (Ingestion Time)

```
System: You are an assessment designer. Given a question, its model answer, topic, and difficulty level, generate exactly 4 multiple-choice options. One option must be the correct answer (derived from the model answer). The other 3 must be plausible but clearly incorrect distractors that test common misconceptions about the topic.

Rules:
- The correct option must be a concise, accurate restatement of the key point in the model answer.
- Distractors must be plausible to someone who has partial understanding.
- Distractors must not be trivially eliminable.
- All options must be similar in length and style.
- Randomize the position of the correct option.
- Return JSON only: {"options": ["...", "...", "...", "..."], "correct_option_index": N}

Input:
Topic: {topic}
Bloom Level: {bloom_level}
Difficulty: {difficulty_prior}/5
Question: {question_text}
Model Answer: {answer}
```

**[AC-AIP-001]** Valid JSON, 4 options, correct_option_index in [0,3]. Retry once on failure. Mark is_active=false on second failure.

### 5.2 User-Specific Option Generation Prompt (Serve Time)

```
System: You are generating a unique version of an assessment question for a specific test-taker. Rephrase the question to test the same concept. Generate 4 new multiple-choice options.

Rules:
- Rephrase the question while preserving its meaning, topic, and difficulty.
- Generate 1 correct option derived from the model answer.
- Generate 3 distractors appropriate to the difficulty and Bloom level.
- Distractor calibration based on user context:
  - If user_theta > 1.0 and user_theta_confidence is "moderate" or "high": make distractors subtle. Target advanced misconceptions. Distractors should require deep understanding to eliminate.
  - If user_theta < -1.0 and user_theta_confidence is "moderate" or "high": make distractors more distinct. Target common beginner errors. Distractors should be clearly wrong to someone with foundational knowledge.
  - Otherwise (new user or moderate ability): use standard distractors at the stated difficulty level.
- Randomize the position of the correct option.
- Return JSON only: {"question_text": "...", "options": ["...", "...", "...", "..."], "correct_option_index": N}

Input:
Original Question: {question_text}
Model Answer: {answer}
Topic: {topic}
Bloom Level: {bloom_level}
Difficulty: {difficulty_prior}/5
User Theta: {user_theta}
User Theta Confidence: {user_theta_confidence}
```

**[AC-AIP-002]** Output is valid JSON. Options are user-specific. If AIP fails, fall back to canonicalOptions. aipPromptVersion recorded per Section 5.5.

### 5.3 Practice Question Generation Prompt

Used when the question bank has insufficient questions for a weak topic.

```
System: Generate a new assessment question for the given topic and difficulty level. The question must be original and test the specified concept area. Provide the question, a model answer, and 4 multiple-choice options.

Rules:
- The question must be at Bloom level: {bloom_level} or lower.
- Difficulty target: {target_difficulty}/5.
- Topic: {topic}, Concept area: {parts}.
- Return JSON only: {"question_text": "...", "answer": "...", "options": ["...", "...", "...", "..."], "correct_option_index": N}
```

**[AC-AIP-003]** Generated questions written to question_bank_active with origin="generated", calibration_regime="metadata", response_count=0.

### 5.4 AIP Chat (Dashboard Queries)

```
System: You are an assessment analytics assistant. Answer using ONLY the data below. Never fabricate numbers. When a metric has low confidence (theta with thetaConfidence="low", or a question in "metadata" calibration regime), note this. When referencing a question's parameters, state its calibration regime.

Data:
{context_payload_as_json}

User query: {query}
```

**[AC-AIP-004]** Responses reference specific values. Regime and confidence noted where relevant. No hallucinated statistics.

### 5.5 AIP Prompt Version Management

Each AIP prompt template is assigned a version string. These are hardcoded constants in Phase 1.

| Prompt | Version String | Stored On |
|---|---|---|
| Ingestion MCQ generation (Section 5.1) | "v1-ingest" | Not stored per-instance (applied at ingestion to all questions) |
| Serve-time user-specific generation (Section 5.2) | "v1-serve" | QuestionInstance.aipPromptVersion |
| AIP failure fallback (canonical options used) | "fallback" | QuestionInstance.aipPromptVersion |
| Practice question generation (Section 5.3) | "v1-practice" | Not stored per-instance (applied at generation) |
| AIP chat (Section 5.4) | "v1-chat" | Not stored (ephemeral queries) |

When a prompt template is modified in a future iteration, the version string should be incremented (e.g., "v2-serve"). This allows analysis of whether prompt changes affected question quality or assessment outcomes by comparing QuestionInstances generated under different versions.

**[AC-AIP-005]** Every QuestionInstance has a non-null aipPromptVersion. The value is either "v1-serve" or "fallback".

---

## 6. Pipelines

### 6.1 Ingestion Pipeline

As described in Section 2.2.

### 6.2 User Features Pipeline

**Trigger:** After each completed attempt (async).

**Steps:**
1. Recompute global_theta via regime-weighted EAP over all user's responses (Section 4.5).
2. Compute theta_confidence from total_questions_answered.
3. Compute avg_response_time_ms.
4. For each topic: compute topic_theta, accuracy, is_weak.
5. Write user_features and topic_mastery.

**[AC-PIPE-002]** Updated within 30 seconds.

### 6.3 Question Stats Pipeline

**Trigger:** Nightly, and opportunistically during CompleteAssessment.

This pipeline is the authoritative source for question statistics and regime state. Any counts or regime updates made by CompleteAssessment are overridden by this pipeline's recount from the responses dataset.

**Steps:**
1. For each question: recount response_count, exposure_count, correct_count from responses dataset.
2. For each question, check regime transition thresholds:
   - If response_count >= 5 and calibration_regime = "metadata": execute Metadata-to-Blended transition (Section 4.6).
   - If response_count >= 20 and calibration_regime = "blended": execute Blended-to-Calibrated transition (Section 4.6).
   - If calibration_regime = "calibrated" and new responses exist: re-estimate parameters (Section 4.6 post-calibration).
3. Compute effectiveness: a_param * (1 - abs(correct_count/response_count - 0.5) * 2).
4. Write back to question_bank_active.

**[AC-PIPE-003]** Regime transitions occur at correct thresholds. All transition audit fields updated. Parameters clamped. effectiveness computed. Counts match the responses dataset exactly.

### 6.4 Percentile Pipeline

**Trigger:** After each completed summative attempt.

**Steps:** Compute percentile from same-config attempts. Null if cohort < 5.

**[AC-PIPE-004]** Percentile between 0-100 or null.

---

## 7. Dashboards

### 7.1 Individual Performance Dashboard

**Filter bar:** User, attempt, topic.

**Charts:**

**7.1.1 Theta Progression (Line Chart)**
- Within-attempt: X=sequence number, Y=theta. Color-code points by selectionRegime of the question.
- Across-attempts: X=attempt number, Y=final_theta. Marker size or opacity by thetaConfidence.

**7.1.2 Topic Mastery (Bar Chart)**
- Sorted ascending by accuracy. Green/red for weak.

**7.1.3 Response Time Distribution (Bar Chart)**
- Bucketed response_time_ms.

**7.1.4 Accuracy vs. Difficulty (Scatter)**
- X=b_param, Y=is_correct. Color by calibration_regime of the question.

**7.1.5 Behavioral Summary (Table)**
- avg_response_time_ms, avg num_option_changes, total questions, total correct, final_theta (with confidence), percentile.

**7.1.6 Percentile (KPI)**
- Large number or "Insufficient data."

**7.1.7 Selection Regime Breakdown for Attempt (Stacked Bar or Pie)**
- Shows how many questions in this attempt were selected via metadata, blended, calibrated. Derived from regimeMixSummary.

**7.1.8 AIP Chat Panel**

**[AC-DASH-001]** All charts from live data. Regime information visible on theta progression and accuracy charts.

### 7.2 Cohort Dashboard

**Filter bar:** Config, time range, topic.

**Charts:**

**7.2.1 Score Distribution (Histogram)**
**7.2.2 Topic Performance (Grouped Bar)**
**7.2.3 Theta Progression Across Attempts (Line)**
**7.2.4 Engagement Metrics (Table)**
**7.2.5 User Segmentation (Scatter)**
**7.2.6 Calibration Coverage (KPI Widgets)**
- Count of questions per regime. Percentage calibrated.
**7.2.7 AIP Chat Panel**

**[AC-DASH-002]** Calibration coverage visible.

### 7.3 Question Effectiveness Dashboard

**Filter bar:** Topic, origin, calibration_regime, difficulty range.

**Charts:**

**7.3.1 Difficulty Calibration (Scatter)**
- X=b_param, Y=correct_rate. Marker style by regime (hollow=metadata, half=blended, solid=calibrated). Only questions with response_count >= 5 shown.

**7.3.2 Discrimination Distribution (Histogram)**
- Separate series for metadata-initialized vs calibrated a_param.

**7.3.3 Parameter Drift (Scatter)**
- X=b_param_initial, Y=b_param (current). Perfect calibration = diagonal line. Points far from diagonal indicate the prior was inaccurate. Only for blended/calibrated questions.

**7.3.4 Option Selection Frequency (Stacked Bar)**
- Filtered to response_count >= 10. Correct option highlighted.

**7.3.5 Exposure Distribution (Bar)**

**7.3.6 Calibration Regime by Topic (Stacked Bar)**
- X=topics, Y=count. Stacked: metadata, blended, calibrated.

**7.3.7 Base vs Generated (Grouped Bar)**

**7.3.8 AIP Chat Panel**

**[AC-DASH-003]** Regime visible on all relevant charts. Parameter drift chart shows how well initial calibration matched reality.

---

## 8. Non-Functional Requirements

### 8.1 Latency

**[R-NF-001]** Question transitions (from submit to next question render) within 2 seconds under normal conditions.

**[AC-NF-001]** Measured by recording the elapsed time between SubmitResponse invocation and the next QuestionInstance render, as observed in Foundry Function execution logs or Workshop event timestamps. Tested with at least 3 concurrent users.

### 8.2 Data Integrity

**[R-NF-002]** No partial writes. If SubmitResponse fails mid-execution, the Attempt state must not advance past the number of actually-persisted Responses.

**[AC-NF-002]** Verified by: (a) inspecting the Attempt record and counting linked Responses after a simulated interruption (close browser mid-submission), (b) confirming numQuestionsAnswered equals the count of Response records linked to the Attempt, (c) confirming the assessment can be resumed from the correct question after the interruption.

### 8.3 Deterministic Replay

**[R-NF-003]** Given attempt_id, reconstruct full assessment including user-specific options from QuestionInstance records.

**[AC-NF-003]** A query joining attempts, question_instances, and responses on attempt_id returns the complete ordered sequence with rendered options.

### 8.4 Concurrent Users

**[R-NF-004]** 10 concurrent sessions without corruption.

**[AC-NF-004]** Tested with 10 simultaneous sessions. No cross-contamination of assessment state between users.

### 8.5 Graceful Degradation

**[R-NF-005]** System functions on a fresh bank with zero users using metadata-driven selection.

**[AC-NF-005]** Verified on fresh bank. Questions are non-random (biased toward high-genericity, medium-difficulty, lower-Bloom). Theta computed. Results displayed.

### 8.6 Regime Auditability

**[R-NF-006]** For any question, an administrator can determine: its current regime, when it entered that regime, what its parameters were before the last transition, what its initial parameters were, and how many responses drove the current calibration.

**[AC-NF-006]** All fields in Section 4.7 are queryable via the Admin Question Bank Browser or direct dataset queries.

### 8.7 Option Isolation

**[R-NF-007]** User A's options for a given base question are never visible to or reused for User B. Each user's assessment is self-contained at the QuestionInstance level.

**[AC-NF-007]** Querying question_instances for two different users' attempts on the same base_question_id returns different renderedOptions (unless AIP happened to generate identical text, which is statistically unlikely but not a violation).

### 8.8 Error Handling, Recovery, and Resumption

**[R-NF-008]** The system handles failures gracefully across three scenarios: AIP unavailability, function execution failures, and user session interruptions.

**AIP unavailability:**
- During ingestion: if AIP fails to generate canonical options for a question after one retry, the question is marked is_active=false and excluded from assessments. It is logged for manual review.
- During StartAssessment pre-generation: if AIP fails for a specific candidate, that candidate uses canonical options with aipPromptVersion="fallback". If AIP is entirely unavailable (all calls fail), all candidates use canonical options. The assessment proceeds normally.
- During AIP chat: if AIP fails to respond, the chat panel displays an error message. No data is affected.

**Function execution failures:**
- All write operations are designed to be idempotent. The key invariant is: numQuestionsAnswered on the Attempt equals the number of Response records linked to it. If SubmitResponse partially fails (Response written but Attempt counter not incremented, or vice versa), the system detects the inconsistency on the next call by checking the actual Response count against the Attempt counter and reconciling.
- The nightly Question Stats Pipeline recounts all statistics from the responses dataset, correcting any stale or inconsistent counts on question_bank_active.

**User session interruptions (browser close, network loss):**
- The Attempt remains in isComplete=false state.
- On return, the Assessment Launcher shows a "Resume Assessment" option (Section 3.2).
- Resumption loads the Attempt and its linked QuestionInstances. The next question is the one with sequenceNumber = numQuestionsAnswered + 1 (already pre-generated during StartAssessment).
- The timer for summative assessments continues from the original start_time. If the user returns after the time limit has expired, the assessment is immediately completed with whatever questions were answered.
- If the user does not return within 24 hours, the next StartAssessment call marks the old Attempt as partially complete and starts a new one.

**[AC-NF-008]** (a) An assessment started with AIP unavailable completes successfully using canonical options. (b) An assessment interrupted by closing the browser can be resumed from the correct question. (c) A summative assessment resumed after its time limit has expired is immediately completed.

### 8.9 Data Retention

**[R-NF-009]** In Phase 1, all data is retained indefinitely. No automated cleanup or archiving is performed.

The datasets that grow linearly with usage are: question_instances (one record per question per attempt per user), responses (one record per answered question), and attempts (one record per session). For a system with 100 users each taking 10 assessments of 45 questions, this produces approximately 45,000 question_instances, 45,000 responses, and 1,000 attempts. This is well within Foundry's capacity.

Phase 2 should implement retention policies if usage exceeds 10,000 assessments or if dataset size becomes a performance concern. See Appendix A.

**[AC-NF-009]** No data is deleted automatically in Phase 1.

---

## Appendix A: Phase 2 Capabilities (Deferred)

**A.1 Behavioral Prediction Model.** Requires 500+ completed attempts.

**A.2 Guessing Detection Model.** Requires calibration data.

**A.3 Automated Question Promotion Pipeline.** origin="generated" questions promoted after meeting quality thresholds.

**A.4 MLOps Infrastructure.** Model registry, drift detection.

**A.5 Advanced Dashboard Visualizations.** Heatmaps, funnel charts, box plots.

**A.6 Role-Based Access Control.** Test-taker, administrator, question author roles.

**A.7 Multi-Select Question Support.** Partial credit scoring for multi-answer questions.

**A.8 Regime Transition History Table.** A dedicated dataset recording every regime transition as a separate row (question_id, from_regime, to_regime, transition_timestamp, response_count_at_transition, params_before, params_after). Phase 1 stores only the most recent transition on the Question object. Phase 2 would provide full history.

**A.9 Data Retention Policies.** Automated archiving of question_instances and responses older than a configurable threshold. Aggregated statistics would be retained; raw records would be moved to cold storage.

---

## Appendix B: Glossary

| Term | Definition |
|---|---|
| theta | User ability estimate on IRT scale, typically -4 to +4 |
| a_param | IRT discrimination. Higher = better at differentiating users |
| b_param | IRT difficulty. The theta at which P(correct)=0.5 |
| 2PL | 2-Parameter Logistic: P = 1 / (1 + exp(-a(theta-b))) |
| EAP | Expected A Posteriori. Bayesian theta estimation with prior |
| Fisher information | I = a^2 * P * (1-P). Measures how much a question reduces uncertainty about theta |
| calibration_regime | Data maturity of a question: metadata (0-4 responses), blended (5-19), calibrated (20+) |
| thetaConfidence | Data maturity of a user's theta: prior (0 questions), low (1-9), moderate (10-29), high (30+) |
| genericity | Source metadata (1-5). How broadly a question tests a concept. Used in Regime 1 selection |
| relevance | Source metadata (1-5). Question relevance to the topic. Stored but not used in Phase 1 |
| bloom_rank | Numeric Bloom's taxonomy (1-6). Knowledge through Evaluation |
| point-biserial correlation | Correlation between a binary variable (correct/incorrect) and a continuous variable (theta). Used to estimate a_param |
| alpha (blend factor) | In Regime 2, the weight given to empirical data vs metadata. Linearly interpolates from 0 (all metadata) to 1 (all empirical) as response_count goes from 5 to 20 |
| canonicalOptions | The baseline MCQ options generated at ingestion time. Serve as fallback if AIP fails at serve time |
| renderedOptions | The user-specific MCQ options generated by AIP at serve time and stored on the QuestionInstance |
| balanced distribution | Equal weight per topic, capped by available questions, with surplus redistributed proportionally |
