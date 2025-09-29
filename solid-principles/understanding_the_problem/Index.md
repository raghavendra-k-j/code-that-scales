# SOLID Principles

## Chapter 1: Understanding the Problem

### 1.1 The Blast Radius

A small change should be small. In real projects, it often isn’t.
When pricing, payments, or shipping rules change, you end up touching many files: cart, checkout, emails, refunds, analytics, and tests.
The more places you edit, the higher the chance of bugs and delays. Deploys feel risky, reviews take longer, and teams block each other.
This is the **blast radius**—how far a single change spreads across your code and your team.

Continue to **[1.2.1 Change amplification (SRP, DIP)](./Change_Amplification.md)** to see how small requests balloon into multi-file edits.

---

### 1.2 The pain shows up in six predictable ways

1. **[1.2.1 Change amplification (SRP, DIP)](./Change_Amplification.md)** — One small request forces edits in many modules because responsibilities are mixed and dependencies are hard-wired.
2. **[1.2.2 Ripple effects & regressions (LSP)](./Ripple_Effect_And_Regressions.md)** — Swapping one part for another breaks callers because the new part behaves differently than expected.
3. **[1.2.3 Variant explosion (OCP)](./Variant_Explosion.md)** — Adding a new “kind” (payment, promotion, report type) spreads `if/else` logic across the codebase.
4. **[1.2.4 Testing friction (SRP, ISP, DIP)](./Testing_Friction.md)** — Unit tests are slow or flaky because code depends on real services (DBs, networks) and giant interfaces.
5. **[1.2.5 Team drag & merge conflicts (SRP, ISP)](./Team_Drag_And_Merge_Conflicts.md)** — Large “god” classes become hotspots. Many people edit the same files, causing conflicts and hidden side effects.
6. **[1.2.6 Vendor/tech lock-in & ops risk (OCP, DIP)](./Vendor_Tech_Lock_In_And_Ops_Risk.md)** — High-level code depends on low-level details. You can’t safely A/B a new provider or roll out changes with low risk.
