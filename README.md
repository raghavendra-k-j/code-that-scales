# Code That Scales

For a quick overview of SOLID principles, see [Quick Reader](./Quick_Reader.md).

## Chapter 1: Understanding the Problem

### 1.1 The Blast Radius

A small change should be small. In real projects, it often isn’t.
When pricing, payments, or shipping rules change, you end up touching many files: cart, checkout, emails, refunds, analytics, and tests.
The more places you edit, the higher the chance of bugs and delays. Deploys feel risky, reviews take longer, and teams block each other.
This is the **blast radius**—how far a single change spreads across your code and your team.

Continue to **[1.2.1 Change amplification (SRP, DIP)](./Change_Amplification.md)** to see how small requests balloon into multi-file edits.

---

### 1.2 The pain shows up in six predictable ways

1. **[1.2.1 Change amplification (SRP, DIP)](./Change_Amplification.md)**
   One small request forces edits in many modules because responsibilities are mixed and dependencies are hard-wired.
   *Result:* slow releases, missed spots, and mismatched totals.

2. **[1.2.2 Ripple effects & regressions (LSP)](./Ripple_Effect_And_Regressions.md)**
   Swapping one part for another breaks callers because the new part behaves differently than expected.
   *Result:* hidden bugs when you change providers or implementations.

3. **[1.2.3 Variant explosion (OCP)](./Variant_Explosion.md)**
   Adding a new “kind” (payment, promotion, report type) spreads `if/else` logic across the codebase.
   *Result:* lots of edits, brittle code, and duplicated tests.

4. **[1.2.4 Testing friction (SRP, ISP, DIP)](./Testing_Friction.md)**
   Unit tests are slow or flaky because code depends on real services (DBs, networks) and giant interfaces.
   *Result:* noisy CI, hard-to-reproduce failures, and developer fatigue.

5. **[1.2.5 Team drag & merge conflicts (SRP, ISP)](./Team_Drag_And_Merge_Conflicts.md)**
   Large “god” classes become hotspots. Many people edit the same files, causing conflicts and hidden side effects.
   *Result:* blocked work, long reviews, and risky deploys.

6. **[1.2.6 Vendor/tech lock-in & ops risk (OCP, DIP)](./Vendor_Tech_Lock_In_And_Ops_Risk.md)**
   High-level code depends on low-level details. You can’t safely A/B a new provider or roll out changes with low risk.
   *Result:* slow vendor switches, poor fallbacks, and costly outages.

---

## Chapter 2: SOLID Principles

This chapter explains **why** these pains happen and **how** to avoid them with SOLID design.
Plain English, short examples, step-by-step refactors.

### 2.1 Introduction — What SOLID solves (and what it doesn’t)

* Why design rules matter for teams, not just for “clean code.”
* How SOLID reduces the blast radius and keeps code open to change.
* When not to over-engineer: start simple, extract when pain shows.
  Continue to **[2.1 Introduction](./SOLID_Introduction.md)**.

### 2.2 The five SOLID principles (in practice, with e-commerce examples)

**2.2.1 [SRP — Single Responsibility Principle](./SRP_Single_Responsibility.md)**
One module should do **one job**.
*Payoff:* small changes stay small; easier tests; fewer conflicts.

**2.2.2 [OCP — Open/Closed Principle](./OCP_Open_Closed.md)**
Code should be **open to extension**, **closed to modification**.
*Payoff:* add a new kind (payment, promo, report) by adding a plug-in, not editing old code.

**2.2.3 [LSP — Liskov Substitution Principle](./LSP_Liskov_Substitution.md)**
If you replace a part with another of the same type, callers should not break.
*Payoff:* safe provider swaps; fewer regressions.

**2.2.4 [ISP — Interface Segregation Principle](./ISP_Interface_Segregation.md)**
Prefer **small, focused interfaces** over one big interface.
*Payoff:* lighter tests and mocks; less coupling; clearer ownership.

**2.2.5 [DIP — Dependency Inversion Principle](./DIP_Dependency_Inversion.md)**
High-level code should depend on **interfaces**, not low-level details.
*Payoff:* easy A/B tests, vendor swaps, and clean unit tests with fakes.

---
