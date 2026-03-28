# Assessment Engine: Work Breakdown Structure and Implementation Plan

**Team:** 2 Engineers (Engineer A, Engineer B)
**Duration:** 4 days
**Platform:** Palantir Foundry

---

## 1. Work Breakdown Structure

Every work item is a discrete deliverable tagged with an ID, assigned engineer, and dependencies.

### Layer 0: Foundation (Day 1)

| ID | Work Item | Owner | Dependencies | Deliverable |
|---|---|---|---|---|
| F-01 | Create all 8 datasets with explicit schemas. question_bank_active includes: canonical_options, canonical_correct_index, a_param_initial, b_param_initial, regime_entered_at, regime_prior_b_param, regime_prior_a_param, b_param_empirical, a_param_empirical, response_count, calibration_regime. question_instances includes: score_info_proxy, score_genericity, score_bloom, score_freshness, score_blend_alpha, theta_confidence_at_serve, aip_prompt_version, selection_regime, selection_score_composite. attempts includes regime_mix_summary. user_features includes theta_confidence | A | None | 8 datasets with all columns correctly typed |
| F-02 | Define Ontology object types (Question, QuestionInstance, Attempt, Response, UserProfile, TopicMastery, AssessmentConfig) with all properties mapped to dataset columns. Verify: canonical options fields, regime audit fields on Question, sub-score fields on QuestionInstance, regimeMixSummary on Attempt, thetaConfidence on UserProfile and Attempt | A | F-01 | 7 object types visible in Ontology Manager, all properties mapped |
| F-03 | Define Ontology link types (attempt_takenBy, attempt_hasInstances, instance_derivedFrom, instance_hasResponse, response_partOfAttempt, user_hasMastery, attempt_usedConfig) | A | F-02 | All 7 links navigable in Ontology |
| F-04 | Design and validate two AIP prompts. First: the ingestion-time MCQ generation prompt (question + model answer in, 4 options + correct index out). Test with 10 sample questions, verify JSON format and distractor quality. Second: the serve-time user-specific option generation prompt, which takes user theta and thetaConfidence as additional inputs and calibrates distractor subtlety. Test by generating options for the same question at theta=-1 (low ability), theta=0 (mid), and theta=+1.5 (high ability). Verify the distractors observably vary in difficulty across these levels | B | None | Two validated prompt templates. Serve-time prompt produces meaningfully different distractors at different theta levels |
| F-05 | Build ingestion pipeline: parse JSON, validate required fields (including genericity, relevance, effectiveness), compute bloom_rank (Knowledge=1 through Evaluation=6), write to question_bank_base | A | F-01 | question_bank_base populated with all source rows, bloom_rank computed |
| F-06 | Build AIP MCQ generation and IRT initialization step in ingestion pipeline. For each base question: call AIP for canonical options, map effectiveness_prior to a_param using a = 0.3 + (effectiveness_prior / 5.0) * 1.4, map difficulty_prior to b_param using b = (difficulty_prior - 3) * 1.0, store as both current and initial values, set response_count=0, calibration_regime="metadata", regime_entered_at=now, all empirical fields=null. Write to question_bank_active | B | F-04, F-05 | question_bank_active populated. Every row has 4-element canonical options array, differentiated a_param and b_param, regime audit fields initialized |
| F-07 | Seed assessment_configs with default records: summative (45 questions, 30 minutes), formative (45 questions, untimed), practice (15 questions, untimed) | A | F-01 | 3 config records exist |
| F-08 | Joint validation. Verify: (1) a_param varies across questions (not a flat value), (2) b_param varies, (3) bloom_rank is populated and correct, (4) all canonical options arrays have exactly 4 elements, (5) a_param_initial = a_param and b_param_initial = b_param for all rows, (6) calibration_regime = "metadata" for all rows, (7) regime_entered_at is non-null, (8) Ontology links are navigable end-to-end | A+B | F-02, F-03, F-06, F-07 | Validation log showing all 8 checks pass |

**End of Day 1 integration point:** Both engineers jointly run F-08. If a_param or b_param are flat (indicating the mapping formulas were not applied correctly), fix before proceeding. The differentiated priors are essential for Regime 1 selection to work on Day 2.

---

### Layer 1: Assessment Engine (Day 2)

| ID | Work Item | Owner | Dependencies | Deliverable |
|---|---|---|---|---|
| E-01 | Implement theta estimation function (EAP on discrete grid). Grid: theta from -4.0 to +4.0 in steps of 0.1. Prior: standard normal. Regime-weighted likelihood: metadata questions weighted at 0.5, blended at 0.5 + 0.5*alpha, calibrated at 1.0. Clamp output to [-3.0, +3.0]. Unit test with 4 cases: (a) all-correct pattern produces positive theta, (b) all-incorrect produces negative, (c) mixed pattern with regime weights produces different theta than unweighted version, (d) clamping works at extremes | A | F-06 | Function returns correct theta for all 4 test vectors |
| E-02 | Implement three-regime question selection scoring. Build each sub-function independently: information_proxy (Fisher information from a,b,theta, normalized to [0,1]), genericity_score (Q.genericity/5.0 for low-confidence users, 1.0 otherwise), bloom_appropriateness (distance from expected Bloom level mapped by theta), freshness (1/(1+exposure_count)). Compose into Regime 1 formula: 0.40*info + 0.25*genericity + 0.20*bloom + 0.15*freshness. Implement Regime 3 formula: 0.85*irt_information + 0.15*freshness. Regime 2 blending (alpha-based interpolation) can be deferred to Day 3 morning (M-00) since all questions start in Regime 1. Unit tests: (a) new user (theta=0, thetaConfidence="prior"): top 5 have avg genericity >= 3.5 and avg bloom_rank <= 3.5, (b) high-theta user: top 5 favor higher Bloom and matching difficulty, (c) exposure control: overexposed question is skipped, (d) topic balancing: underrepresented topic (relative to balanced distribution or configured weights) restricts candidates, (e) all 4 sub-scores are returned alongside composite | A | E-01 | Scoring function tested across Regime 1 and 3. Sub-scores returned for audit |
| E-03 | Implement AIP serve-time variant generation wrapper. Takes: question object + user theta + user thetaConfidence. Calls AIP with the serve-time prompt (designed in F-04). Validates JSON output (4 options, correct_option_index in [0,3]). On AIP failure: falls back to canonical options from the base question. Returns: renderedQuestionText, renderedOptions, correctOptionIndex, aipPromptVersion ("v1-serve" or "fallback") | B | F-04 | Wrapper function tested. Valid output returned. Fallback to canonical options works |
| E-04 | Implement StartAssessment Function. Steps: (0) concurrent assessment guard: check for existing in-progress Attempt (isComplete=false) for this user; if one exists and is less than 24 hours old, reject and return the existing attempt_id; if older than 24 hours, mark it as partially complete and proceed. (1) Create Attempt object (isComplete=false). (2) Load user's theta and thetaConfidence from UserProfile (or theta=0.0 and "prior" for new users). (3) Select candidate pool (~2x config question count) using regime-aware scoring (E-02). (4) Call AIP wrapper (E-03) for each candidate to generate user-specific variants; if AIP rate limits are hit mid-generation, switch remaining candidates to canonical options with aipPromptVersion="fallback". (5) Write each as a QuestionInstance with all sub-scores populated (scoreInfoProxy, scoreGenericity, scoreBloom, scoreFreshness, scoreBlendAlpha, selectionRegime, selectionScoreComposite, thetaConfidenceAtServe, aipPromptVersion). (6) Select first question from pool, return it | A | E-01, E-02, E-03, F-02 | Function callable. Concurrent guard rejects if in-progress attempt exists. QuestionInstances have user-specific renderedOptions. All sub-score fields populated. AIP rate limit fallback works |
| E-05 | Implement SubmitResponse Function. Steps: validate (attempt not complete, instance belongs to attempt, no prior response for this instance), validate time (if summative and elapsed > config duration + 5 seconds, reject as expired), determine correctness using instance's correctOptionIndex (not the base question's canonical index), write Response object, update running theta via regime-weighted EAP (E-01), update Attempt counters (numQuestionsAnswered, numCorrect), if formative mode: include answer and justification from base question in return payload, check termination (question count reached, or time expired for summative), if not terminated: select next question from pre-generated pool using scoring (E-02) and record sub-scores on that instance, return next QuestionInstance (and feedback if formative). If terminated: call CompleteAssessment | A | E-01, E-02, E-04 | Full question-answer cycle works. Correctness uses instance-level index. Expired submissions rejected. Sub-scores recorded on each served instance |
| E-06 | Implement CompleteAssessment Function. Steps: compute final theta from full response pattern, compute percentile (null if fewer than 5 same-config completed attempts exist), compute regimeMixSummary (JSON counting instances by selectionRegime for this attempt), set thetaConfidence on Attempt based on total questions answered, mark Attempt complete (isComplete=true, endTime=now). Trigger async: update responseCount, exposureCount, correctCount on each served Question. Regime transition checks can be deferred to the nightly pipeline (M-06) to keep this function lean | A | E-05 | Attempt complete with regimeMixSummary, finalTheta, thetaConfidence. Question counts updated |
| E-07 | Build Workshop Assessment Launcher page. Layout: config selector dropdown (filtered by mode), config details display, "Start Assessment" button wired to StartAssessment action. If the user has an in-progress assessment (isComplete=false, less than 24 hours old), show "Resume Assessment" button instead of "Start Assessment", linked to the existing attempt. Past attempts table showing columns: date, mode, score, theta, thetaConfidence, regimeMixSummary (displayed as compact string, e.g., "M:12 B:18 C:15") | B | F-02, F-03, F-07 | Page loads, shows configs, shows past attempts with regime mix. In-progress assessment shows resume option. Start button works |
| E-08 | Build Workshop Assessment View page. Layout: top bar with timer (countdown for summative, elapsed for formative), progress indicator ("Question 7 of 45"), mode badge. Question area displays renderedQuestionText from QuestionInstance. Options area with radio buttons for renderedOptions (the user-specific AIP-generated options, not canonical). Submit button wired to SubmitResponse. Ensure correctness in formative feedback uses the instance-level correct index. Progress bar updates after each submission. Resumption: if loading an in-progress attempt, query the Attempt and its linked QuestionInstances; display the question with sequenceNumber = numQuestionsAnswered + 1. For summative, resume the timer from the original start_time; if time has already expired, trigger CompleteAssessment immediately | B | E-04, E-05 | Full question-answer flow works in UI. User sees AIP-generated options unique to them. Resumption loads the correct next question |

**End of Day 2 integration point:** Both engineers run a complete summative assessment as a new user through the Workshop UI. Critical checks: (a) the first 5 questions are observably high-genericity and lower-Bloom (inspect QuestionInstance records to verify), (b) renderedOptions differ from canonicalOptions (AIP generated user-specific options), (c) all QuestionInstance sub-scores are populated and non-null, (d) selectionRegime = "metadata" for all instances (no calibrated data exists yet), (e) theta updates question-by-question, (f) assessment completes and regimeMixSummary is recorded. If selection looks random, debug E-02 together before Day 3.

---

### Layer 2: Modes, Results, Pipelines (Day 3)

| ID | Work Item | Owner | Dependencies | Deliverable |
|---|---|---|---|---|
| M-00 | Implement Regime 2 blending in E-02 (if deferred from Day 2). Add the alpha-based blend between metadata_score and irt_score: blended_score = alpha * irt_score + (1-alpha) * metadata_score, where alpha = (responseCount - 5) / 15. Test continuity: at responseCount=5 (alpha=0), blended_score equals metadata_score. At responseCount=20 (alpha=1), blended_score equals irt_score | A | E-02 | Regime 2 blending works. No discontinuity at boundaries |
| M-01 | Add formative mode to Assessment View. After the user submits, display a feedback panel showing: correctness (correct/incorrect), the model answer (from the base Question's answer field), and the justification (from the base Question's justification field). Add a "Next" button that advances to the next question. The feedback panel appears between Submit and Next. In summative mode, this panel is hidden and the next question appears immediately after Submit | B | E-08 | Formative assessment works end-to-end. Feedback shows after each question |
| M-02 | Add timer to Assessment View. Summative mode: display a countdown timer starting from config_duration_minutes. When time reaches 0, disable further interaction and trigger CompleteAssessment. Backend validation in SubmitResponse: reject submissions where (submitted_at - start_time) exceeds config_duration_minutes * 60 + 5 seconds (5-second grace for network latency). Formative mode: display elapsed time (informational only, no enforcement) | B | E-08 | Timer enforces limit in summative. Assessment auto-completes on expiry. Late submissions rejected |
| M-03 | Implement StartPractice Function. Steps: read TopicMastery records for this user, identify topics where isWeak=true, if none exist return "no practice needed" message, otherwise create a dynamic AssessmentConfig (mode=practice, numQuestions=15, no time limit, topicWeights set to 80% weak topics and 20% other), delegate to StartAssessment with this config | A | E-04 | Practice sessions contain at least 80% weak-topic questions |
| M-04 | Build Workshop Results View. Layout: score (numCorrect / numQuestionsAnswered), final theta labeled "Ability Estimate" with thetaConfidence indicator (if "prior" or "low", display: "Estimate confidence is low; more assessments will improve accuracy"), percentile (or "Insufficient cohort data for percentile" if null), regime mix summary for this attempt (e.g., "34 questions used statistical selection, 11 used metadata-based selection"), topic breakdown bar chart (sorted ascending by accuracy, green for non-weak, red for weak topics), answer review expansion panel (list all questions with: sequence number, rendered question text, user's selected option, correct option, correctness; in formative mode also show model answer and justification), "Start Practice" button (visible only if weak topics exist), "Return to Home" button | B | E-06, M-01 | Results render with confidence indicators, regime summary, null-percentile handling, and topic breakdown |
| M-05 | Build User Features Pipeline. Trigger: after each completed attempt (async). Steps: collect all responses for this user across all attempts, recompute global_theta using regime-weighted EAP over the full response set (same method as E-01 but over all historical responses), compute theta_confidence from total_questions_answered, compute avg_response_time_ms, for each topic: compute topic_theta from topic-specific responses, compute accuracy (correct/total), set is_weak (true if topic_theta < global_theta - 1.0 or topic_theta < -0.5), write/overwrite user_features and topic_mastery records | A | F-01, E-06 | user_features and topic_mastery updated. theta_confidence consistent with total_questions_answered |
| M-06 | Build Question Stats Pipeline. Trigger: nightly scheduled job. Steps: (1) For each question, recount response_count, exposure_count, correct_count from the responses dataset (this is the authoritative count, overriding any in-flight updates from E-06). (2) For each question, check regime transition thresholds. (3) If response_count >= 5 and calibration_regime = "metadata": compute b_param_empirical = -log(p_correct / (1 - p_correct)) where p_correct = max(0.05, min(0.95, correct_count/response_count)). Compute alpha = (response_count - 5) / 15. Blend: b_param_new = (1-alpha) * b_param_initial + alpha * b_param_empirical. Clamp to [-4, +4]. Record: regime_prior_b_param = old b_param, regime_prior_a_param = old a_param, b_param = b_param_new, calibration_regime = "blended", regime_entered_at = now. (4) If response_count >= 20 and calibration_regime = "blended": recompute b_param_empirical with full data. Join responses for this question with user_features to get each respondent's global_theta. Compute point-biserial correlation between correctness (0/1) and respondent theta values. Convert to a_param_empirical = r_pb * 2.5. Clamp a_param to [0.3, 3.0]. If point-biserial cannot be computed (no variance in correctness or theta), retain a_param and log warning. Set b_param = b_param_empirical (clamped). Record transition metadata. Set calibration_regime = "calibrated". (5) If calibration_regime = "calibrated" and new responses exist: re-estimate b_param and a_param using full data. Do not change regime or regime_entered_at. (6) Compute effectiveness = a_param * (1 - abs(correct_count/response_count - 0.5) * 2). Write back to question_bank_active | A | F-06, E-06 | Regime transitions occur at correct thresholds. Point-biserial computed for calibrated transitions. All transition audit fields updated. Parameters clamped |
| M-07 | Build Percentile Computation Pipeline. Trigger: after each completed summative attempt. Collect final_theta for all completed summative attempts with the same config_id. If fewer than 5 exist, set percentile = null. Otherwise, compute percentile rank. Write back to the attempt record | A | E-06 | Percentile populated (0-100) or null for small cohorts |
| M-08 | Build Practice Launcher page in Workshop. Layout: list of weak topics (from TopicMastery where isWeak=true), recommended practice length (15 questions), "Start Practice" button wired to StartPractice action. If no weak topics exist, display message "No weak areas identified. Complete a summative assessment first." and hide the start button | B | M-03, M-05 | Practice launcher works. No-weak-topics case handled |

**End of Day 3 integration point:** Both engineers verify: (a) formative assessment works with feedback after each question, (b) practice mode works and targets weak topics, (c) timer enforces the summative time limit, (d) User Features Pipeline produces correct theta_confidence and topic_mastery, (e) Question Stats Pipeline transitions questions at the correct thresholds (simulate by manually adjusting response_count if insufficient real test data). If regime transitions are not firing, debug M-06 threshold logic together.

---

### Layer 3: Dashboards, AIP Chat, Polish (Day 4)

| ID | Work Item | Owner | Dependencies | Deliverable |
|---|---|---|---|---|
| D-01 | Build Individual Performance Dashboard. Filter bar: user selector (defaults to logged-in user), attempt selector, topic filter. Charts: (1) Theta Progression line chart, within-attempt view (X=sequence number, Y=theta, points color-coded by selectionRegime of the question) and across-attempts view (X=attempt number, Y=final_theta, marker opacity by thetaConfidence). (2) Topic Mastery bar chart (sorted ascending by accuracy, green for non-weak, red for weak). (3) Response Time Distribution bar chart (bucketed: 0-10s, 10-20s, 20-30s, 30-60s, 60s+). (4) Accuracy vs Difficulty scatter (X=b_param, Y=is_correct, color by question's calibrationRegime). (5) Behavioral Summary table (avg response_time, avg option_changes, total questions, total correct, final_theta with confidence label, percentile or "N/A"). (6) Percentile KPI widget (large number or "Insufficient data"). (7) Regime Mix for Attempt (stacked bar or pie chart from regimeMixSummary showing metadata/blended/calibrated counts) | B | M-05, M-07 | Dashboard renders from live data. Selecting a different attempt updates all charts. Regime info visible on theta progression and accuracy charts. Regime mix chart per attempt |
| D-02 | Build Cohort Dashboard. Filter bar: config selector, time range, topic filter. Charts: (1) Score Distribution histogram (X=final_theta bucketed, Y=user count). (2) Topic Performance Comparison grouped bar (X=topics, Y=avg accuracy). (3) Theta Progression Across Attempts line (X=attempt number, Y=avg final_theta). (4) Engagement Metrics table (avg response time, avg completion rate, avg option changes). (5) User Segmentation scatter (X=global_theta, Y=avg_response_time_ms from user_features). (6) Calibration Coverage KPI widgets: count of questions in each regime (metadata, blended, calibrated) and percentage calibrated | B | M-05, M-07 | Dashboard renders. Config filter works. Calibration coverage visible |
| D-03 | Build Question Effectiveness Dashboard. Filter bar: topic filter, origin filter (base/generated), calibration_regime filter, difficulty range slider. Charts: (1) Difficulty Calibration scatter (X=b_param, Y=observed correct_rate; marker style by regime: hollow=metadata, half-filled=blended, solid=calibrated; only questions with response_count >= 5). (2) Discrimination Distribution histogram (separate series for metadata-initialized a_param vs calibrated a_param). (3) Parameter Drift scatter (X=b_param_initial, Y=b_param current; only blended/calibrated questions; perfect calibration = diagonal line; points far from diagonal indicate the prior was inaccurate). (4) Option Selection Frequency stacked bar (X=question_id subset, Y=proportion selecting each option; correct option highlighted; filtered to response_count >= 10). (5) Exposure Distribution bar (X=question_id or topic, Y=exposure_count). (6) Calibration Regime by Topic stacked bar (X=topics, Y=count of questions, stacked: metadata/blended/calibrated). (7) Base vs Generated Comparison grouped bar (X=metric such as avg correct_rate and avg a_param, grouped by origin) | A | M-06 | Dashboard renders. Regime visible on all relevant charts. Parameter drift chart shows calibration accuracy. Low-data questions filtered or annotated |
| D-04 | Build AIP Chat system. Context builder: assemble context payload as JSON, including calibration_regime for questions, thetaConfidence for users, and response_count to indicate data maturity. Prompt includes instruction: "When a metric has low confidence (theta with thetaConfidence='low', or a question in 'metadata' calibration regime), note this uncertainty in your answer." Wire chat panel to all 3 dashboards. Individual dashboard: context includes user's profile, topic mastery, last 3 attempts. Cohort dashboard: context includes aggregate stats, distributions, calibration coverage. Question dashboard: context includes question properties, response stats, regime info | A | D-01, D-02, D-03 | Chat answers grounded queries on all dashboards. States confidence level when data is immature. No hallucinated statistics |
| D-05 | Admin page: Question Bank Browser. Object table with columns: questionId, topic, bloomLevel, origin, calibrationRegime, responseCount, a_param, b_param, isActive. Filterable by topic, origin, calibrationRegime. Sortable by any column. This page is also useful for verifying regime transitions are occurring | B | F-02 | Admin can browse and filter questions by regime |
| D-06 | Admin page: Assessment Configuration editor. Form to create or edit AssessmentConfig records: label, mode, numQuestions, durationMinutes, topicWeights (JSON editor or structured input), difficultyRange, isDefault. No code change required to add a new config | A | F-07 | Admin can create and modify configs through the UI |
| D-07 | End-to-end integration testing. Both engineers run through the following scenarios together. (1) Cold-start: ingest fresh question bank, start summative as new user. Verify first 5 questions are high-genericity/low-Bloom (inspect QuestionInstance records). Verify all sub-scores are populated. Verify renderedOptions differ from canonicalOptions. Verify selectionRegime = "metadata" for all instances. (2) Warm-up: simulate 10 users completing assessments (batch script or manual). Run Question Stats Pipeline. Verify some questions transition to "blended". Verify parameter drift chart has data points. Verify calibration coverage KPIs update. (3) Formative: start formative assessment, answer 5 questions. Verify feedback shown after each using instance-level correct answer. Verify answer review shows renderedOptions. (4) Practice: complete a summative with deliberately poor performance on specific topics. Verify weak topics detected. Start practice. Verify practice questions are 80%+ from weak topics. (5) Option isolation: two users take the same assessment config. Query question_instances for both users on the same base_question_id. Verify renderedOptions differ between users. (6) Timer: start a summative with a short duration (2 minutes). Let timer expire. Verify assessment auto-completes. Attempt a late submission. Verify rejection. (7) Small cohort: with fewer than 5 completed assessments for a config, verify percentile is null, not 0. Complete 2 more. Verify percentile appears. (8) AIP fallback: if possible, simulate AIP failure during StartAssessment (e.g., disconnect network briefly). Verify canonical options are used. Verify aipPromptVersion = "fallback" on affected instances. (9) Resumption: start a summative, answer 5 questions, close browser. Reopen. Verify launcher shows "Resume Assessment". Click resume. Verify the 6th question is displayed (not question 1). For summative, verify timer reflects elapsed time from original start | A+B | All | Test log with all 9 scenarios documented and passing |

**End of Day 4:** System is operational. All flows tested. Known issues documented.

---

## 2. Dependency Graph

```
Layer 0 (Day 1):
  F-01 ──> F-02 ──> F-03
   |         |
   |         └──> F-07 ──> F-08
   |
   └──> F-05 ──> F-06 ──> F-08
                    ^
  F-04 ────────────┘
        └──────────────> (used in Layer 1 E-03)

Layer 1 (Day 2):
  F-06 ──> E-01 ──> E-02 ──> E-04 ──> E-05 ──> E-06
  F-04 ──> E-03 ──────────> E-04
  F-02 ──────────────> E-07
  F-03 ──────────────> E-07
  F-07 ──────────────> E-07
  E-04 + E-05 ──────> E-08

Layer 2 (Day 3):
  E-02 ──> M-00 (if deferred)
  E-08 ──> M-01, M-02
  E-04 ──> M-03
  E-06 ──> M-04, M-05, M-06, M-07
  M-03 + M-05 ──> M-08

Layer 3 (Day 4):
  M-05 + M-07 ──> D-01, D-02
  M-06 ──> D-03
  D-01 + D-02 + D-03 ──> D-04
  F-02 ──> D-05
  F-07 ──> D-06
  ALL ──> D-07
```

---

## 3. Execution Approach

### 3.1 Parallel Tracks

Engineer A (Backend, Data, Logic): Datasets, Ontology, Functions (including the three-regime scoring algorithm and theta estimation), Pipelines (including regime transition logic and point-biserial computation), AIP chat integration.

Engineer B (AIP, UI, Dashboards): AIP prompt engineering (both ingestion and serve-time prompts), Workshop UI pages, dashboard construction.

The tracks are independent within each day and converge at integration points.

### 3.2 Integration Points

There are 3 integration points where both engineers must sync and jointly verify.

**IP-1 (End of Day 1):** Joint validation (F-08). The critical check is that a_param and b_param vary across questions (the metadata-derived initialization worked). If these are flat, the cold-start selection will not function correctly.

**IP-2 (End of Day 2):** Run a complete summative assessment as a new user through Workshop. The critical check is that the first 5 questions are observably biased toward high-genericity, lower-Bloom questions, and that user-specific AIP options are being generated and stored. If selection looks random, debug the scoring algorithm together.

**IP-3 (End of Day 3):** Verify formative mode, practice mode, timer enforcement, and pipeline outputs. The critical check is that the Question Stats Pipeline correctly transitions questions between regimes when thresholds are crossed.

### 3.3 Day 2 Simplification Options

Day 2 is the most logic-intensive day for Engineer A. Two simplifications are available if needed:

**Option 1:** Defer Regime 2 blending to Day 3 morning (M-00). Since all questions start in Regime 1 and no question will reach Regime 3 on Day 2 (no response data exists yet), this has zero functional impact on Day 2 testing. Regime 2 is only needed once questions accumulate 5+ responses, which happens after multiple assessments.

**Option 2:** Simplify E-06 (CompleteAssessment) by deferring inline regime transition checks to the nightly pipeline (M-06). E-06 still updates question counts, but the regime field updates happen in the pipeline rather than immediately. The tradeoff is that regime transitions happen nightly instead of after each attempt. Acceptable for Phase 1.

### 3.4 Scope Compression Options

If the team falls behind schedule, these items can be deferred in priority order (lowest priority first):

1. Admin pages (D-05, D-06): useful but the system functions without them.
2. Cohort Dashboard (D-02): valuable but not needed for individual assessment flow.
3. Question Effectiveness Dashboard (D-03): important for ongoing quality but not for initial launch.
4. AIP Chat (D-04): a convenience layer over the dashboards.
5. Parameter Drift chart (D-03 item 3): a single chart within a dashboard.
6. Regime mix display on results page (part of M-04): informational, not functional.

Items that should not be deferred: summative flow, formative flow, practice flow, Regime 1 selection, theta estimation, Results View, Individual Dashboard (at minimum: theta progression, topic mastery, percentile), User Features Pipeline, Question Stats Pipeline (regime transitions are needed for correctness over time).

---

## 4. Definition of Done

The build is complete when all of the following are true:

1. A new user with no prior history can start a summative assessment, answer 45 questions with a countdown timer, and receive a results page with score, theta (marked as low confidence), topic breakdown, and regime mix summary. The first 5 questions served are biased toward high-genericity, lower-Bloom questions.

2. A new user can start a formative assessment, answer questions, see feedback (correct answer + model answer + justification) after each question, and receive results.

3. After a summative attempt with weak topics identified, the user can start a practice session that targets those weak topics (80%+ of questions from weak topics).

4. Each QuestionInstance stores user-specific renderedOptions generated by AIP. Two users taking the same assessment see different options for the same base question. Correctness is always evaluated against the instance-level correctOptionIndex.

5. Every QuestionInstance has all selection sub-scores populated: scoreInfoProxy, scoreGenericity, scoreBloom, scoreFreshness, selectionScoreComposite, selectionRegime, and scoreBlendAlpha (for Regime 2).

6. The Question Stats Pipeline transitions questions from "metadata" to "blended" at 5 responses (with blended b_param) and from "blended" to "calibrated" at 20 responses (with empirical a_param from point-biserial and empirical b_param). All transition metadata is recorded.

7. Theta estimation downweights Regime 1 questions at 0.5 and Regime 3 at 1.0.

8. The Individual Dashboard shows theta progression (color-coded by selection regime), topic mastery, response time distribution, percentile (or "insufficient data"), and regime mix for the selected attempt.

9. The Cohort Dashboard shows score distribution, topic performance, and calibration coverage (count of questions per regime).

10. The Question Effectiveness Dashboard shows difficulty calibration with regime-coded markers, parameter drift, and option selection frequency.

11. The AIP chat panel answers at least one grounded query per dashboard. When data confidence is low, the response says so.

12. No question appears twice in a single assessment.

13. Timer enforcement works: summative assessments auto-complete when time expires, and the backend rejects late submissions.

14. All data is persisted: attempts, responses, question instances (with user-specific options and sub-scores), user features, topic mastery.

15. The system functions correctly on a freshly ingested question bank with zero prior users. Questions served are non-random. Theta is computed. Results are displayed.

---

## 5. Test Cases

### Selection Algorithm Tests (for E-02)

| Test ID | Setup | Expected | Validates |
|---|---|---|---|
| SEL-01 | theta=0.0, thetaConfidence="prior", all Regime 1 | Top 5 avg genericity >= 3.5, avg bloom_rank <= 3.5 | Cold-start bias |
| SEL-02 | theta=1.5, thetaConfidence="moderate", all Regime 1 | Top 5 favor higher Bloom and matching difficulty | Theta-responsive selection |
| SEL-03 | theta=0.5, thetaConfidence="moderate", mix of Regime 1 and 3 | Both regimes present. Regime 3 scored by calibrated formula, Regime 1 by metadata formula | Mixed-regime scoring |
| SEL-04 | Two identical questions, one Regime 2 (alpha=0.93), one Regime 3 | Scores similar. No sharp discontinuity | Boundary continuity |
| SEL-05 | Top candidate has exposure_count above 90th percentile | Skipped. Second candidate returned | Exposure control |
| SEL-06 | Topic A underrepresented by > 20% | Only Topic A candidates considered | Topic balancing |
| SEL-07 | Any selection | All 4 sub-scores returned. For Regime 1: weighted sum of sub-scores = composite | Audit trail completeness |

### Regime Transition Tests (for M-06)

| Test ID | Setup | Expected | Validates |
|---|---|---|---|
| TRANS-01 | Question with 4 responses | calibration_regime = "metadata". No parameter change | Below threshold |
| TRANS-02 | Question with 6 responses, 2 correct | calibration_regime = "blended". b_param blended (alpha=1/15). regime_prior_b_param populated | Metadata-to-blended |
| TRANS-03 | Question with 22 responses, varied respondent thetas | calibration_regime = "calibrated". b_param = empirical. a_param from point-biserial. Transition metadata recorded | Blended-to-calibrated |
| TRANS-04 | Question with 20 responses, all correct | b_param_empirical clamped to -4.0. a_param: retain initial (no variance in correctness) | Edge case |
| TRANS-05 | Question with 20 responses, all respondents same theta | a_param: retain initial (no variance in theta). Warning logged | Edge case |
| TRANS-06 | Already calibrated question gets 10 more responses | Parameters re-estimated. Regime and regime_entered_at unchanged | Post-calibration refinement |

### Integration Tests (for D-07)

| Test ID | Scenario | Verifies |
|---|---|---|
| INT-01 | Cold-start: fresh bank, new user | First 5 high-gen/low-Bloom. Sub-scores populated. User-specific options generated. selectionRegime = "metadata" |
| INT-02 | Warm-up: simulate 10 users, run pipeline | Regime transitions occur. Dashboards show calibration coverage. Parameter drift chart has data |
| INT-03 | Formative flow | Feedback shown. Instance-level correct answer used. Answer review shows renderedOptions |
| INT-04 | Practice flow | Weak topics detected. Practice targets them (80%+) |
| INT-05 | Option isolation | Two users' instances for same base question have different renderedOptions |
| INT-06 | Timer enforcement | Auto-complete on expiry. Late submission rejected |
| INT-07 | Small cohort percentile | Percentile null when cohort < 5. Populated when cohort >= 5 |
| INT-08 | AIP fallback | Canonical options used on failure. aipPromptVersion = "fallback" |
| INT-09 | Assessment resumption | Launcher shows "Resume". Resumed assessment loads correct next question. Summative timer reflects original start |

---

## 6. Day-by-Day Schedule

### Day 1: Foundation

| Block | Engineer A | Engineer B |
|---|---|---|
| Morning | F-01: Create 8 datasets with full schemas. F-05: Build ingestion pipeline with bloom_rank computation and field validation | F-04: Design and validate both AIP prompts (ingestion + serve-time). Test with 10 questions. Validate serve-time prompt produces different distractors at different theta levels |
| Afternoon | F-02: Define all 7 Ontology object types with all properties. F-03: Define link types. F-07: Seed assessment configs | F-06: Build AIP MCQ generation + IRT initialization pipeline step. Run against full question bank |
| End of day | **IP-1: Joint validation (F-08)** |

### Day 2: Assessment Engine

| Block | Engineer A | Engineer B |
|---|---|---|
| Morning | E-01: Theta estimation with regime weighting. E-02 (start): Build and unit-test each sub-function independently | E-03: AIP serve-time wrapper with fallback. E-07: Assessment Launcher page |
| Afternoon | E-02 (complete): Compose scoring formulas, run unit tests. E-04: StartAssessment. E-05: SubmitResponse. E-06: CompleteAssessment | E-08: Assessment View page with user-specific option rendering |
| End of day | **IP-2: Run full summative assessment as new user. Verify cold-start selection and user-specific options** |

### Day 3: Modes and Pipelines

| Block | Engineer A | Engineer B |
|---|---|---|
| Morning | M-00: Regime 2 blending (if deferred). M-03: StartPractice. M-05: User Features Pipeline | M-01: Formative mode (feedback panel). M-02: Timer |
| Afternoon | M-06: Question Stats Pipeline with regime transitions and point-biserial. M-07: Percentile Pipeline | M-04: Results View. M-08: Practice Launcher |
| End of day | **IP-3: Verify formative, practice, timer, pipelines, regime transitions** |

### Day 4: Dashboards and Polish

| Block | Engineer A | Engineer B |
|---|---|---|
| Morning | D-03: Question Effectiveness Dashboard (7 charts including parameter drift). D-06: Config editor | D-01: Individual Dashboard (7 charts with regime annotations). D-05: Question Bank Browser with regime filter |
| Afternoon | D-04: AIP Chat (context builder, uncertainty-aware prompt, wire to 3 dashboards) | D-02: Cohort Dashboard (6 charts + calibration coverage KPIs) |
| End of day | **D-07: Joint end-to-end testing (9 scenarios)** |
