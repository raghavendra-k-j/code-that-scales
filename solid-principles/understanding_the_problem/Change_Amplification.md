# 1.2.1 Change Amplification (SRP, DIP)
One small change should require one small edit. When price logic is copy-pasted across flows, every tweak fans out to carts, checkouts, receipts, refunds, analytics, and tests. This section shows how the Single Responsibility and Dependency Inversion principles reduce the blast radius.

## Plain idea
- Keep pricing rules in one place with one owner.
- Let every flow consume the same pricing engine instead of re-implementing math.
- Parameterise rules (percentages, coupon codes, thresholds) so tweaks are data, not code edits.
## Scenario

- **Business request**: “Launch *Buy 2 Get 1 Free*. Later we might try *Buy 3 Get 1* or *Buy 1 Get 2*. Add `SAVE200`, maybe `SAVE300`, and bump the VIP discount from 10% to 15%.”
- **Symptoms today**: The tiny promo forces edits across five modules and a handful of tests. Rounding and refund behaviour drift apart, and regressions sneak in.
## Before — logic mixed across surfaces

Each surface re-codes the same rules. Order matters, rounding differs, and some flows forget entire features (VIP discounts, coupons, shipping, BxGy). Refunds have no idea how much discount was applied per line.

```python
# ui_cart.py (Cart page)
def render_cart_page(items, user_tier, coupon):
	subtotal = sum(i["price"] * i["qty"] for i in items)

	if user_tier == "GOLD":
	if coupon == "SAVE200":
		subtotal -= 200

	free_discount = 0
	for i in items:
		if i["sku"].startswith("TSHIRT"):
			free_discount += (i["qty"] // 3) * i["price"]

	subtotal -= free_discount
	shipping = 0 if subtotal >= 999 else 49
	total = max(0, round(subtotal + shipping, 2))
	return {"subtotal": round(subtotal, 2), "shipping": shipping, "total": total}


# checkout.py (Place order)
def calculate_checkout_total(items, user_tier, coupon):
	subtotal = sum(i["price"] * i["qty"] for i in items)

	if coupon == "SAVE200":
	elif coupon == "SAVE300":
		subtotal -= 300

	if user_tier == "GOLD":
	free_discount = 0
	for i in items:
		if i["sku"].startswith("TSHIRT") or i["sku"].startswith("SOCKS"):

			free_discount += (i["qty"] // 3) * i["price"]
	subtotal -= free_discount

	shipping = 0 if subtotal >= 999 else 49
	return max(0, round(subtotal + shipping, 2))


# receipt.py (Email receipt)
def build_receipt_text(order):
	items = order["items"]
	coupon = order["coupon"]

	subtotal = sum(i["price"] * i["qty"] for i in items)
	if coupon == "SAVE200":
		subtotal -= 200
	elif coupon == "SAVE300":
		subtotal -= 300

	shipping = 0 if subtotal >= 999 else 49
	total = max(0, round(subtotal + shipping, 2))

	lines = [
		f"Subtotal: ₹{round(subtotal, 2)}",
		f"Shipping: ₹{shipping}",
		f"Total:    ₹{total}",
	]
	return "\n".join(lines)
```

Other flows (returns, analytics) contain yet more variations. Every edit becomes a treasure hunt.

### What goes wrong

- **Change amplification**: Adjusting a VIP discount demands updates in four or five files.
- **Inconsistent totals**: Different rule order and rounding produce mismatched numbers that frustrate QA and finance.
- **Refund guesswork**: Without allocations, returns prorate totals incorrectly.
- **Slow tests**: Duplication means touching dozens of tests to change one rule.

## After — one pricing engine, pluggable rules

Design a `PricingEngine` that owns all price math. It depends on lightweight rule classes that can be composed, reordered, or configured without touching the consumers.

```python
from dataclasses import dataclass
from typing import List, Optional, Dict
from abc import ABC, abstractmethod


@dataclass
class Item:
	sku: str
	price: float
	qty: int
	category: Optional[str] = None


@dataclass
class Adjustment:
	label: str
	amount: float  # negative = discount, positive = fee
	meta: Optional[Dict] = None


@dataclass
class Breakdown:
	subtotal: float
	adjustments: List[Adjustment]

	@property
	def total(self) -> float:
		return round(self.subtotal + sum(a.amount for a in self.adjustments), 2)


class Rule(ABC):
	@abstractmethod
	def apply(self, items: List[Item], context: dict) -> List[Adjustment]:
		...


class FlatCoupon(Rule):
	def __init__(self, code: str, amount: float):
		self.code = code
		self.amount = amount

	def apply(self, items, context):
		return [Adjustment(f"Coupon {self.code}", -self.amount)] if context.get("coupon") == self.code else []


class PercentOffForTier(Rule):
	def __init__(self, tier: str, percent: float):
		self.tier = tier
		self.percent = percent

	def apply(self, items, context):
		if context.get("user_tier") != self.tier:
			return []
		subtotal = sum(i.price * i.qty for i in items)
		return [Adjustment(f"{self.tier} {self.percent}% off", -(self.percent/100.0) * subtotal)]


class BuyXGetYFree(Rule):
	def __init__(self, x: int, y: int, only_skus: Optional[List[str]] = None, only_categories: Optional[List[str]] = None):
		self.x = x
		self.y = y
		self.only_skus = set(only_skus) if only_skus else None
		self.only_categories = set(only_categories) if only_categories else None

	def apply(self, items, context):
		adjustments: List[Adjustment] = []
		group = self.x + self.y
		for it in items:
			if self.only_skus and it.sku not in self.only_skus:
				continue
			if self.only_categories and it.category not in self.only_categories:
				continue
			free_units = (it.qty // group) * self.y
			if free_units:
				adjustments.append(
					Adjustment(
						f"Buy {self.x} Get {self.y} Free ({it.sku})",
						-free_units * it.price,
						meta={"sku": it.sku, "free_units": free_units},
					)
				)
		return adjustments


class ShippingFlatOrFreeOver(Rule):
	def __init__(self, threshold: float, flat_fee: float):
		self.threshold = threshold
		self.flat_fee = flat_fee

	def apply(self, items, context):
		subtotal = sum(i.price * i.qty for i in items)
		fee = 0.0 if subtotal >= self.threshold else self.flat_fee
		return [Adjustment("Shipping", fee)]


class AllocateDiscountsPerLine(Rule):
	"""Optional helper to record per-line allocations for refunds."""

	def apply(self, items, context):
		return []


class PricingEngine:
	def __init__(self, rules: List[Rule]):
		self.rules = rules

	def price(self, items: List[Item], context: dict) -> Breakdown:
		subtotal = round(sum(i.price * i.qty for i in items), 2)
		adjustments: List[Adjustment] = []
		for rule in self.rules:
			adjustments.extend(rule.apply(items, context))
		return Breakdown(subtotal=subtotal, adjustments=adjustments)


rules = [
	PercentOffForTier("GOLD", 10.0),
	FlatCoupon("SAVE200", 200.0),
	FlatCoupon("SAVE300", 300.0),
	BuyXGetYFree(2, 1, only_categories=["TOPS"]),
	ShippingFlatOrFreeOver(999.0, 49.0),
	AllocateDiscountsPerLine(),
]
engine = PricingEngine(rules)


def analytics_track_order(items: List[Item], user_tier: str, coupon: str):
	breakdown = engine.price(items, {"user_tier": user_tier, "coupon": coupon})
	return {"event": "ORDER_PLACED", "amount": breakdown.total}
```

### Why it works

- **Single Responsibility**: Pricing rules live in one module with focused tests and owners.
- **Dependency Inversion**: Cart, checkout, refunds, analytics all depend on the abstract `PricingEngine` contract, not concrete rule orderings.
- **Parameterised rules**: Adjusting a coupon or tier becomes a config change, not a cross-cutting code edit.

## What to verify

1. Unit-test each rule (VIP percent, coupon, BxGy) in isolation.
2. Smoke-test the engine end-to-end to confirm totals match the legacy numbers.
3. Update refunds to consume allocation metadata if needed.

## Navigation

- ⬅️ Previous: [Index](./index.md)
- ➡️ Next: [1.2.2 Ripple Effects & Regressions](./Ripple_Effect_And_Regressions.md)
