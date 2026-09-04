# Test Case Quality Scoring Skill

## Objective

Evaluate the quality of a generated test case consistently using the rubric below.

Do NOT assign scores based on overall impression.

Score each dimension independently using the defined criteria and evidence from the user story, acceptance criteria, domain context, and test case.

Maximum score: 20 points.

---

## 1. Requirement Coverage — 0 to 4

Evaluate how well the test case validates the intended behavior described by the requirement.

**0 — No coverage**

* Test is unrelated to the requirement.

**1 — Weak coverage**

* Test is loosely related but does not meaningfully validate the requirement.

**2 — Partial coverage**

* Test validates part of the requirement but misses an important condition, scenario, or outcome.

**3 — Good coverage**

* Test directly validates the primary requirement behavior.

**4 — Thorough coverage**

* Test directly validates the requirement and covers an important condition, variation, boundary, or outcome relevant to the requirement.

Do not award 4 merely because the test is detailed. Award 4 only when the additional coverage provides meaningful validation value.

---

## 2. Correctness — 0 to 4

Evaluate whether the test is logically and factually correct based on the available evidence.

**0 — Incorrect**

* Contradicts the requirement or known domain behavior.
* Contains fundamentally invalid logic.

**1 — Major issues**

* Contains significant incorrect assumptions or logically invalid steps.

**2 — Partially correct**

* Core scenario is plausible, but one or more steps, inputs, conditions, or outcomes are questionable.

**3 — Correct**

* Test is logically consistent with the available information.
* No significant errors identified.

**4 — Fully correct**

* Every major step, input, precondition, and expected outcome is supported by the available evidence.
* No unsupported assumptions or contradictions.

IMPORTANT:
If the available information is insufficient to determine correctness, do not assume correctness.
Mark the uncertainty and score conservatively.

---

## 3. Test Steps / Executability — 0 to 4

Evaluate whether another tester or automation framework could execute the test unambiguously.

**0 — Not executable**

* Steps are missing or unusable.

**1 — Major gaps**

* Significant information required for execution is missing.

**2 — Partially executable**

* Test can mostly be understood but requires assumptions or clarification.

**3 — Executable**

* Steps are clear, logically ordered, and sufficient to execute the test.

**4 — Highly executable**

* Steps are precise, deterministic, minimal, and contain all important preconditions/test data/actions required for execution.

Do not reward unnecessary verbosity.

---

## 4. Expected Result — 0 to 4

Evaluate whether the expected result clearly defines what must happen for the test to pass.

**0 — Missing**

* No expected result.

**1 — Vague or incorrect**

* Expected result is ambiguous, generic, or unsupported.

**2 — Partially defined**

* Expected behavior is identified but important observable outcomes are missing.

**3 — Clear**

* Expected result is specific and understandable.

**4 — Precise and testable**

* Expected result is specific, observable, deterministic, and directly traceable to the requirement or domain rule.

Prefer observable outcomes such as:

* UI state
* API response/status
* database/business state
* displayed message
* transaction status
* state transition

Avoid vague outcomes such as:

* "system works correctly"
* "operation is successful"
* "user is notified appropriately"

unless the requirement itself defines those outcomes sufficiently.

---

## 5. Test Value — 0 to 4

Evaluate whether executing this test provides meaningful defect-detection value.

**0 — No value**

* Test does not meaningfully validate behavior.

**1 — Very low value**

* Highly trivial or redundant test with little defect-detection potential.

**2 — Moderate value**

* Validates expected behavior but has limited ability to expose defects.

**3 — Good value**

* Tests meaningful behavior, condition, failure mode, or risk.

**4 — High value**

* Specifically targets an important business rule, boundary, negative scenario, state transition, risk, or realistic failure mode.

Do not automatically give 4 to negative or edge-case tests. The scenario must be meaningful.

---

# Scoring Rules

### Rule 1 — Score dimensions independently

Do not let a strong score in one dimension influence another.

A detailed test is not necessarily correct.

A correct test is not necessarily valuable.

A test with excellent steps is not necessarily good requirement coverage.

---

### Rule 2 — Use evidence, not intuition

Base the evaluation on:

1. User story
2. Acceptance criteria
3. Domain knowledge/context
4. Explicit assumptions
5. Test case itself

Do not invent missing application behavior.

---

### Rule 3 — Penalize unsupported assumptions

If the test assumes behavior that is not supported by the available evidence, reduce the Correctness score.

Examples:

* Invented error codes
* Invented UI messages
* Invented API behavior
* Invented business rules
* Invented system states

---

### Rule 4 — Missing information should reduce confidence

If a requirement is ambiguous and the test makes a specific assumption without evidence, do not reward that assumption.

---

### Rule 5 — Avoid score inflation

A test should NOT receive a high score simply because:

* it contains many steps
* it is verbose
* it sounds professional
* it uses technical terminology
* it has detailed descriptions
* the scenario appears realistic

Score only against the five defined dimensions.

---

### Rule 6 — Do not give 4 unless the highest-level criterion is demonstrated

A score of 4 requires evidence that the test satisfies the full definition of the dimension.

When uncertain between two scores, choose the lower score.

---

### Rule 7 — Duplicate tests should lose Test Value

If another generated test covers essentially the same scenario, reduce Test Value.

Do not penalize correctness or executability simply because another test is similar.

---

## Hard Constraints

Apply these constraints after scoring:

* Missing expected result → Expected Result score = 0
* Missing executable steps → Test Steps score ≤ 1
* Contradicts requirement → Correctness ≤ 1
* Unsupported critical assumption → Correctness ≤ 2
* Unrelated to requirement → Requirement Coverage = 0
* Essentially duplicate of another test → Test Value ≤ 1

---

## Final Score

Calculate:

Score = Requirement Coverage
+ Correctness
+ Test Steps
+ Expected Result
+ Test Value

Maximum = 20

Convert to percentage:

Final Score = (Score / 20) × 100

Classification:

17–20  = Excellent
14–16  = Good
10–13  = Fair
0–9    = Poor

---

## Required Output

Return:

{
"overall_score": <0-100>,
"rating": "<Excellent|Good|Fair|Poor>",
"dimensions": {
"requirement_coverage": <0-4>,
"correctness": <0-4>,
"test_steps": <0-4>,
"expected_result": <0-4>,
"test_value": <0-4>
},
"strengths": [
"<specific evidence>"
],
"weaknesses": [
"<specific evidence>"
],
"improvement": "<single most important improvement>"
}

Keep the justification concise and evidence-based.
