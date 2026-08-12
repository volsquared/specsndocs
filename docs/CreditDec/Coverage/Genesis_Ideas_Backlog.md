# Genesis — Running Ideas & Detail Backlog

**Purpose:** A running log for ideas, mechanics and detail that come up during Genesis discussions but are too granular for the Product Capability Model or too early for a formal Deep Dive. Nothing here is committed scope — it's raw material, captured with enough detail to be useful whenever we come back to it.

**How this file works:** New entries get appended, most recent at the bottom of each section. When an idea graduates into an actual document (Capability Model, a Deep Dive, an Architecture doc), it should be marked **Promoted** here with a pointer to where it landed, not deleted — so the history of how an idea evolved isn't lost.

---

## Entry: Strategy Test Engineering — Detailed Mechanics

**Captured:** During Genesis Product Capability Model v0.1 development, discussing Runtime Testing.
**Status:** Open — the capability itself (Strategy Test Engineering / Test Execution) is now in the Capability Model at the L2 level. Everything below is the mechanic-level detail that was deliberately kept out of the Capability Model and belongs in the future **Strategy Engineering Workbench Deep Dive**.

### The core mental model

A strategy shouldn't just be a DSL file — it's an engineering asset with associated tests, versioned together:

> **Strategy + Test Suite + Test Data + Expected Outcomes = Versioned Strategy Package**

Illustrative packaging shape (exact packaging is an architecture decision, not a product one):

```text
strategy/
  credit-limit-increase/
    strategy.dsl
    tests/
      happy-path.yaml
      arrears-customer.yaml
      insufficient-tenure.yaml
      affordability-referral.yaml
    test-data/
      customer-001.json
      customer-002.json
    metadata/
      strategy.yaml
```

### Authoring experience — Given / When / Then

The Workbench should let a user build a test case visually, without writing YAML or code. For each test case:

**Given**
- input/customer/application data
- mocked external data if required
- model outputs if those need isolating
- execution context

**When**
- this strategy/version is executed

**Then**
- expected final decision
- expected reason codes
- optionally, expected path/nodes executed
- optionally, expected intermediate outputs

Example scenarios:

> Given customer tenure = 18 months, no arrears, affordability = PASS
> When strategy `CLI-v12` executes
> Then decision = APPROVE
> And reason code = ELIGIBLE

> Given customer tenure = 4 months
> Then decision = DECLINE
> And rule `MIN_TIME_ON_BOOK` evaluates false

The second example matters: tests shouldn't only assert on the final decision. Behavioural assertions against specific rules/nodes let a test catch *why* a decision changed, not just *that* it changed.

### Workbench UX on test run

Editing a node should immediately surface test impact:

```text
Tests
✓ 47 passed
✕ 2 failed
○ 3 not run
```

Clicking a failed test opens the **Decision Inspector** against that specific execution:

```text
Expected: APPROVE
Actual: REFER

Difference:
AffordabilityRule
Expected: PASS
Actual: UNKNOWN

Input:
monthly_income = £3,500
monthly_commitments = null
```

### Tests as first-class strategy assets

Tests should be: named, saved, versioned, cloned, shared, attached to a strategy, executed as a suite, included in change history, and available to CI/CD. When a strategy moves from v12 to v13, its test suite and fixtures travel with it as one unit — Strategy v13 + Test Suite v13 + Test Fixtures + Expected Assertions.

This does **not** depend on the Git-authority open question being resolved first — the conceptual model holds whether Genesis owns its own strategy repository or Git becomes the backing store; Git integration would simply mirror/export the same artefacts.

### CI/CD pipeline triggered by commit/publish/PR

1. Compile DSL
2. Static validation
3. Execute unit-test suite
4. Produce test results
5. Execute mandatory regression suite
6. Optionally launch replay against historical population
7. Compare behaviour against current champion
8. Generate evidence
9. Apply governance gate

### Turning production decisions into regression tests

A powerful specific feature: while investigating a production issue in the Decision Inspector, an engineer finds a historical decision that should have referred but approved due to a specific input combination. A **"Create test case from this decision"** action copies the relevant execution snapshot into a test fixture; the engineer corrects the expected assertion and names it, e.g.:

> `REG-18472 — Missing affordability response must refer`

Once saved alongside the strategy, that defect becomes part of the automated test suite and can never silently reappear. This is one of the more distinctive ideas to come out of this project — it operationalises Genesis's existing "everything should be testable" principle rather than just asserting it.

---
