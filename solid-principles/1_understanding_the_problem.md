# SOLID Principles

## Chapter 1: Understanding the Problem

### 1.1 The Blast Radius

A small change should be small. In real projects, it often isn’t.
When pricing, payments, or shipping rules change, you end up touching many files: cart, checkout, emails, refunds, analytics, and tests.
The more places you edit, the higher the chance of bugs and delays. Deploys feel risky, reviews take longer, and teams block each other.
This is the **blast radius**—how far a single change spreads across your code and your team.

---

### 1.2 The pain shows up in six predictable ways

**1.2.1 Change amplification (SRP, DIP)**
One small request forces edits in many modules because responsibilities are mixed and dependencies are hard-wired.

**1.2.2 Ripple effects & regressions (LSP)**
Swapping one part for another breaks callers because the new part behaves differently than expected.

**1.2.3 Variant explosion (OCP)**
Adding a new “kind” (payment, promotion, report type) spreads `if/else` logic across the codebase.

**1.2.4 Testing friction (SRP, ISP, DIP)**
Unit tests are slow or flaky because code depends on real services (DBs, networks) and giant interfaces.

**1.2.5 Team drag & merge conflicts (SRP, ISP)**
Large “god” classes become hotspots. Many people edit the same files, causing conflicts and hidden side effects.

**1.2.6 Vendor/tech lock-in & ops risk (OCP, DIP)**
High-level code depends on low-level details. You can’t safely A/B a new provider or roll out changes with low risk.

---

### 1.2.1 One small change makes you edit many places (SRP + DIP)

**Plain idea**
A small request from the **Marketing Team** should need a small code change.
If you edit many files for one request, your design is mixed up.

**Our app**
Think Myntra/Flipkart/Amazon.

**New request**
Marketing Team: “Add **Buy X Get Y Free**. Start with **Buy 2 Get 1**. Later we may try **Buy 3 Get 1** or **Buy 1 Get 2**. Also allow **SAVE200**, **SAVE300**, and change VIP discount from **10%** to **15%** if needed.”

---

## Before — logic mixed across files

**Problem in one line:**
Logic for calculating offers is added in multiple places with `if/else` or switch statements.
Cart, checkout, receipt, returns, and analytics each do their own math.

> Demo code is long on purpose so you can see the pain. (We skip taxes/international rules to keep it focused.)

```python
# ===========================
# ui_cart.py  (Cart page)
# ===========================
def render_cart_page(items, user_tier, coupon):
    # items: list of {sku, price, qty}
    # 1) calculate subtotal
    subtotal = sum(i["price"] * i["qty"] for i in items)

    # 2) VIP percent off (hard-coded 10% for GOLD)
    if user_tier == "GOLD":
        subtotal = subtotal * 0.90

    # 3) flat coupon (only checks SAVE200 here)
    if coupon == "SAVE200":
        subtotal -= 200

    # 4) Buy 2 Get 1 only for TSHIRT (cart page forgets SOCKS)
    free_discount = 0
    for i in items:
        if i["sku"].startswith("TSHIRT"):
            free_discount += (i["qty"] // 3) * i["price"]
    subtotal -= free_discount

    # 5) shipping
    shipping = 0 if subtotal >= 999 else 49

    total = max(0, round(subtotal + shipping, 2))

    # 6) show numbers on UI
    return {
        "subtotal": round(subtotal, 2),
        "shipping": shipping,
        "total": total
    }


# ===========================
# checkout.py  (Place order)
# ===========================
def calculate_checkout_total(items, user_tier, coupon):
    subtotal = sum(i["price"] * i["qty"] for i in items)

    # same logic again, but order is different (can change rounding)
    # 1) coupon: this one checks SAVE200 or SAVE300
    if coupon == "SAVE200":
        subtotal -= 200
    elif coupon == "SAVE300":
        subtotal -= 300

    # 2) VIP 10% for GOLD only (hard-coded again)
    if user_tier == "GOLD":
        subtotal = subtotal * 0.90

    # 3) Buy 2 Get 1 free for TSHIRT and SOCKS (not same as cart page!)
    free_discount = 0
    for i in items:
        if i["sku"].startswith("TSHIRT") or i["sku"].startswith("SOCKS"):
            free_discount += (i["qty"] // 3) * i["price"]
    subtotal -= free_discount

    # 4) shipping
    shipping = 0 if subtotal >= 999 else 49

    total = max(0, round(subtotal + shipping, 2))
    return total


# ===========================
# receipt.py  (Email receipt)
# ===========================
def build_receipt_text(order):
    items = order["items"]
    user_tier = order["user_tier"]
    coupon = order["coupon"]

    # recompute AGAIN (third version)
    subtotal = sum(i["price"] * i["qty"] for i in items)

    # here we forgot the VIP discount completely (oops)
    if coupon == "SAVE200":
        subtotal -= 200
    elif coupon == "SAVE300":
        subtotal -= 300

    # here we forgot Buy 2 Get 1 completely (oops again)
    shipping = 0 if subtotal >= 999 else 49

    total = max(0, round(subtotal + shipping, 2))

    lines = [
        f"Subtotal: ₹{round(subtotal, 2)}",
        f"Shipping: ₹{shipping}",
        f"Total:    ₹{total}",
    ]
    return "\n".join(lines)


# ===========================
# returns.py  (Refund math)
# ===========================
def calculate_refund_amount(original_items, returned_items, user_tier, coupon):
    # Goal: refund proportional amounts based on final charged price.
    # But we re-implement pricing logic a fourth time.

    # 1) build map for easier use
    price_map = {i["sku"]: i["price"] for i in original_items}
    qty_map = {i["sku"]: i["qty"] for i in original_items}

    # 2) compute subtotal again
    subtotal = sum(price_map[sku] * qty_map[sku] for sku in qty_map)

    # 3) apply offers again (yet another variation)
    if coupon == "SAVE200":
        subtotal -= 200
    elif coupon == "SAVE300":
        subtotal -= 300

    # VIP 10% if GOLD (we remembered here)
    if user_tier == "GOLD":
        subtotal *= 0.90

    # Buy 2 Get 1 only for TSHIRT (we forgot SOCKS again)
    free_discount = 0
    for sku, qty in qty_map.items():
        if sku.startswith("TSHIRT"):
            free_discount += (qty // 3) * price_map[sku]
    subtotal -= free_discount

    # 4) shipping
    shipping = 0 if subtotal >= 999 else 49
    total_paid = max(0, round(subtotal + shipping, 2))

    # 5) now try to compute refund for returned_items
    # (this code does not know how much discount was applied per line,
    # so it guesses; refunds become inconsistent)
    returned_value = sum(price_map[i["sku"]] * i["qty"] for i in returned_items)

    # naive ratio refund
    original_value = sum(price_map[sku] * qty for sku, qty in qty_map.items())
    ratio = returned_value / original_value if original_value else 0
    refund = round(total_paid * ratio, 2)
    return refund


# ===========================
# analytics.py  (Revenue numbers)
# ===========================
def track_revenue_event(items, user_tier, coupon):
    # We compute a "total" again for analytics dashboards.
    subtotal = sum(i["price"] * i["qty"] for i in items)

    # different rule set again (only coupon; no VIP; no BxGy)
    if coupon == "SAVE200":
        subtotal -= 200
    elif coupon == "SAVE300":
        subtotal -= 300

    shipping = 0 if subtotal >= 999 else 49
    total = max(0, round(subtotal + shipping, 2))

    # send to analytics provider
    return {"event": "ORDER_PLACED", "amount": total}
```

### What goes wrong here

* You change **VIP 10% → 15%**. You must edit **cart**, **checkout**, and **returns**. It is easy to miss **receipt** and **analytics**.
* You change **SAVE200 → SAVE300**. You must edit **4–5 places**.
* You switch **Buy 2 Get 1** to **Buy 3 Get 1**. You must search all files and hope you catch every `// 3`.
* Different files apply rules in a different order. Rounding can change. Totals don’t match.
* Refunds become messy because discounts are not tracked clearly.
* Your team wastes time in reviews and hotfixes.

---

## After — one pricing engine, rules are plug-ins

**Simple plan**

* Put **all pricing math in one place**: `PricingEngine`. (This is **Single Responsibility**.)
* The engine uses small **Rule** classes. Each rule has one `apply()` method. We plug rules into the engine. (This is **Dependency Inversion**.)
* Rules are **parameterized**: you can change numbers without touching UI, checkout, receipt, returns, or analytics.
* Everything (cart, checkout, receipt, returns, analytics) calls the **same engine**. One source of truth.

```python
from dataclasses import dataclass
from typing import List, Optional, Dict
from abc import ABC, abstractmethod

# ---------- Core models ----------
@dataclass
class Item:
    sku: str
    price: float
    qty: int
    category: Optional[str] = None  # optional, helps for category promos

@dataclass
class Adjustment:
    label: str
    amount: float  # negative = discount, positive = fee
    meta: Optional[Dict] = None     # extra info (e.g., which items were free)

@dataclass
class Breakdown:
    subtotal: float
    adjustments: List[Adjustment]

    @property
    def total(self) -> float:
        return round(self.subtotal + sum(a.amount for a in self.adjustments), 2)

# ---------- Rule interface ----------
class Rule(ABC):
    @abstractmethod
    def apply(self, items: List[Item], context: dict) -> List[Adjustment]:
        ...

# ---------- Reusable rules (parameterized) ----------
class FlatCoupon(Rule):
    def __init__(self, code: str, amount: float):
        self.code = code
        self.amount = amount
    def apply(self, items, context):
        return [Adjustment(f"Coupon {self.code}", -self.amount)] if context.get("coupon") == self.code else []

class PercentOffForTier(Rule):
    def __init__(self, tier: str, percent: float):
        self.tier = tier
        self.percent = percent  # e.g., 10 → 10%
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
        adj: List[Adjustment] = []
        group = self.x + self.y
        for it in items:
            if self.only_skus and it.sku not in self.only_skus:
                continue
            if self.only_categories and (it.category not in self.only_categories):
                continue
            free_units = (it.qty // group) * self.y
            if free_units > 0:
                adj.append(Adjustment(
                    f"Buy {self.x} Get {self.y} Free ({it.sku})",
                    -free_units * it.price,
                    meta={"sku": it.sku, "free_units": free_units}
                ))
        return adj

class ShippingFlatOrFreeOver(Rule):
    def __init__(self, threshold: float, flat_fee: float):
        self.threshold = threshold
        self.flat_fee = flat_fee
    def apply(self, items, context):
        subtotal = sum(i.price * i.qty for i in items)
        fee = 0.0 if subtotal >= self.threshold else self.flat_fee
        return [Adjustment("Shipping", fee)]

# Optional: keep per-line allocations for refund logic
class AllocateDiscountsPerLine(Rule):
    """Turns overall discounts into per-item allocations for fair refunds/returns."""
    def apply(self, items, context):
        # This rule does not change the total; it just adds metadata.
        # In a real system you would store allocations line by line.
        return []

# ---------- Engine ----------
class PricingEngine:
    def __init__(self, rules: List[Rule]):
        self.rules = rules
    def price(self, items: List[Item], context: dict) -> Breakdown:
        subtotal = round(sum(i.price * i.qty for i in items), 2)
        adjustments: List[Adjustment] = []
        for rule in self.rules:
            adjustments.extend(rule.apply(items, context))
        return Breakdown(subtotal=subtotal, adjustments=adjustments)

# ---------- Wiring (easy to change; could come from config) ----------
rules = [
    PercentOffForTier("GOLD", 10.0),                # change to 15.0 later
    FlatCoupon("SAVE200", 200.0),                   # switch to SAVE300 by changing code/amount
    BuyXGetYFree(2, 1, only_categories=["TOPS"]),   # switch to (3,1) or (1,2); target by category or SKU
    ShippingFlatOrFreeOver(999.0, 49.0),
    AllocateDiscountsPerLine(),
]
engine = PricingEngine(rules)

# ---------- All flows call the same engine ----------
def cart_view(items: List[Item], user_tier: str, coupon: str):
    b = engine.price(items, {"user_tier": user_tier, "coupon": coupon})
    return {"subtotal": b.subtotal,
            "adjustments": [(a.label, a.amount) for a in b.adjustments],
            "total": b.total}

def checkout_place_order(items: List[Item], user_tier: str, coupon: str):
    # same call → same numbers
    b = engine.price(items, {"user_tier": user_tier, "coupon": coupon})
    return {"total": b.total}

def email_receipt(order):
    items = order["items"]
    b = engine.price(items, {"user_tier": order["user_tier"], "coupon": order["coupon"]})
    lines = [f"Subtotal: ₹{b.subtotal:.2f}"] + \
            [f"{a.label}: ₹{a.amount:.2f}" for a in b.adjustments] + \
            [f"Total:    ₹{b.total:.2f}"]
    return "\n".join(lines)

def compute_refund(original_items: List[Item], returned_items: List[Item], user_tier: str, coupon: str):
    # use the same engine to know the real charged total
    b = engine.price(original_items, {"user_tier": user_tier, "coupon": coupon})
    total_paid = b.total

    # simple proportional refund; could use Allocation rule metadata
    orig_value = sum(i.price * i.qty for i in original_items)
    returned_value = sum(i.price * i.qty for i in returned_items)
    ratio = (returned_value / orig_value) if orig_value else 0
    return round(total_paid * ratio, 2)

def analytics_track_order(items: List[Item], user_tier: str, coupon: str):
    b = engine.price(items, {"user_tier": user_tier, "coupon": coupon})
    return {"event": "ORDER_PLACED", "amount": b.total}
```

### What improves now

* Change **VIP 10% → 15%**: update **one place** where rules are wired.
* Add **SAVE300**: add **one** `FlatCoupon("SAVE300", 300.0)` line (or load from config).
* Switch **Buy 2 Get 1** → **Buy 3 Get 1** → **Buy 1 Get 2**: change parameters `(3,1)` or `(1,2)`.
* Cart, checkout, receipt, returns, analytics all show **the same numbers**, because they all call the **same engine**.
* Tests are simple: test each rule alone; test the engine once.


---

### 1.2.2 Ripple effects & bugs when you swap a part (LSP)

**Plain idea**
You should be able to replace one part with another, and the rest of the app should still work.
If switching a provider breaks pages and jobs, the replacement is not truly compatible.

**Our app**
Think Myntra/Flipkart/Amazon.

**New request (Logistics Team)**
“Switch from **Courier A** to **Courier B** (better price). Everything else should work as before: checkout ETA, tracking page, returns, and notifications.”

---

## Before — hidden contract, many surprises

**Problem in one line:**
Different couriers return different shapes, units, errors, and rules. Code talks to them directly.
When we swap A → B, checkout, tracking, and returns break in different ways.

> Demo code is long so you can see the pain. (Simple types; no real APIs.)

```python
# ===========================
# shipping_service.py
# ===========================
class CourierAClient:
    # Returns: {"tracking_id": str, "eta_hours": float, "status": "CREATED"}
    def create_shipment(self, address: dict, weight_kg: float) -> dict:
        if not address.get("pincode"):
            raise ValueError("bad address")  # simple error
        if weight_kg > 30:
            raise ValueError("too heavy")
        return {"tracking_id": "A123", "eta_hours": 36.0, "status": "CREATED"}

    # Returns: {"status": "IN_TRANSIT" | "DELIVERED" | "RETURNED", "last_seen": "str"}
    def track(self, tracking_id: str) -> dict:
        return {"status": "IN_TRANSIT", "last_seen": "Hub-7"}

class CourierBClient:
    # Different shape: {"id": "...", "eta_minutes": int, "state": "init|moving|done|rto"}
    class BApiError(Exception): ...
    def create(self, dest: dict, kg: float):
        if not dest.get("pincode") or not dest.get("phone"):   # requires phone (stricter)
            return None  # returns None on error (not exception)
        if kg > 10:  # stricter limit than A
            raise CourierBClient.BApiError("over 10kg not allowed")
        return {"id": "B999", "eta_minutes": 1800, "state": "init"}  # 1800 min = 30 hours

    def status(self, tid: str):
        return {"state": "moving", "seen_at": 1699999999}  # unix time

# App code directly depends on the concrete courier:
USE_COURIER = "A"  # later someone flips it to "B"

def place_shipment(address: dict, weight_kg: float):
    if USE_COURIER == "A":
        client = CourierAClient()
        rec = client.create_shipment(address, weight_kg)
        tracking_id = rec["tracking_id"]
        eta_hours = rec["eta_hours"]
    else:
        client = CourierBClient()
        # different method name, different returns, different error style
        rec = client.create(address, weight_kg)          # may return None
        tracking_id = rec["id"]                          # different key
        eta_hours = rec["eta_minutes"] / 60.0            # unit conversion

    return {"tracking_id": tracking_id, "eta_hours": eta_hours}

# ===========================
# checkout.py (uses ETA at checkout)
# ===========================
def show_checkout_eta(cart, address):
    # assume total weight based on cart
    weight_kg = sum(i["kg"] * i["qty"] for i in cart)
    info = place_shipment(address, weight_kg)  # creates a real shipment just to estimate!
    return f"ETA: ~{round(info['eta_hours'])} hours"

# ===========================
# tracking_page.py (tracking UI)
# ===========================
def tracking_view(tracking_id):
    if USE_COURIER == "A":
        s = CourierAClient().track(tracking_id)
        status = s["status"]  # IN_TRANSIT / DELIVERED / RETURNED
    else:
        s = CourierBClient().status(tracking_id)
        # different states: init|moving|done|rto
        status = s["state"]
    return {"status": status}

# ===========================
# returns_flow.py (allow returns if delivered)
# ===========================
def can_return(tracking_id):
    if USE_COURIER == "A":
        st = CourierAClient().track(tracking_id)["status"]
        return st == "DELIVERED"
    else:
        st = CourierBClient().status(tracking_id)["state"]
        # BUG: we forgot to map "done" → DELIVERED; returns get stuck
        return st == "DELIVERED"
```

### What goes wrong here

* **Stricter inputs (tightened rules):** Courier B requires `phone` and allows only **≤10 kg**. Our app expected **≤30 kg** and phone optional. Some orders now fail.
* **Different outputs:** B returns `eta_minutes` and `state` (“moving/done/rto”), not `eta_hours` and “IN_TRANSIT/DELIVERED/RETURNED”. Pages break or show wrong status.
* **Different error style:** A raises `ValueError`; B returns `None` or `BApiError`. Callers don’t handle both styles; we get crashes.
* **Side effect at checkout:** `show_checkout_eta` creates a real shipment just to estimate. With B, that may charge or reserve shipments wrongly.

Result: Checkout crashes for heavy orders; tracking shows “moving” when UI expects “IN_TRANSIT”; returns get blocked because code checks for “DELIVERED”, not “done”.

---

## After — one clear contract + adapters that keep promises

**Simple plan**

* Define a **Courier** interface (contract) for our app.

  * Inputs it accepts (address must have `pincode`; `phone` is optional).
  * Weight in **kg**, up to **30 kg**.
  * Outputs (tracking fields) and **units** (ETA in **hours**).
  * **Standard errors** we raise (`CourierError` with `code`: `"INVALID_ADDRESS"`, `"OVERWEIGHT"`, `"TEMPORARY"`).
* Write **adapters** for Courier A and B that **fit this contract**.

  * Map units and states.
  * Normalize errors.
  * If B only supports 10 kg, **the adapter splits** into multiple parcels to still accept up to 30 kg (so we don’t tighten inputs).
* App talks only to the **Courier** interface. Swaps don’t ripple.

```python
# ===========================
# courier_contract.py
# ===========================
from dataclasses import dataclass
from typing import Protocol, Literal, List, Dict

class CourierError(Exception):
    def __init__(self, code: Literal["INVALID_ADDRESS","OVERWEIGHT","TEMPORARY"], message: str = ""):
        super().__init__(message)
        self.code = code

@dataclass
class ShipmentLabel:
    tracking_id: str
    eta_hours: float    # always hours
    status: Literal["CREATED","IN_TRANSIT","DELIVERED","RETURNED"]

@dataclass
class TrackingInfo:
    status: Literal["IN_TRANSIT","DELIVERED","RETURNED"]
    last_seen: str

class Courier(Protocol):
    """
    Contract:
    - Accepts weight up to 30 kg (kg units).
    - Address must have pincode; phone is optional.
    - create_shipment() returns ShipmentLabel with eta in HOURS.
    - Errors are raised as CourierError with one of the known codes.
    - track() returns TrackingInfo with status in the common set.
    """
    def create_shipment(self, address: Dict, weight_kg: float) -> ShipmentLabel: ...
    def track(self, tracking_id: str) -> TrackingInfo: ...

# ===========================
# courier_a_adapter.py
# ===========================
class CourierAAdapter(Courier):
    def __init__(self, client):
        self.client = client

    def create_shipment(self, address, weight_kg) -> ShipmentLabel:
        try:
            rec = self.client.create_shipment(address, weight_kg)
            return ShipmentLabel(
                tracking_id = rec["tracking_id"],
                eta_hours   = float(rec["eta_hours"]),
                status      = rec["status"]  # already in the common set
            )
        except ValueError as e:
            msg = str(e).lower()
            if "bad address" in msg:
                raise CourierError("INVALID_ADDRESS", str(e))
            if "too heavy" in msg:
                raise CourierError("OVERWEIGHT", str(e))
            raise CourierError("TEMPORARY", str(e))

    def track(self, tracking_id) -> TrackingInfo:
        rec = self.client.track(tracking_id)
        return TrackingInfo(status=rec["status"], last_seen=rec.get("last_seen",""))

# ===========================
# courier_b_adapter.py
# ===========================
class CourierBAdapter(Courier):
    def __init__(self, client):
        self.client = client

    def _map_state(self, s: str) -> str:
        return {"init":"CREATED","moving":"IN_TRANSIT","done":"DELIVERED","rto":"RETURNED"}[s]

    def create_shipment(self, address, weight_kg) -> ShipmentLabel:
        # Do NOT tighten inputs: phone is optional in our contract.
        # If phone is missing, we still proceed by passing blank or fallback.
        addr = dict(address)
        addr.setdefault("phone", "NA")

        # Do NOT tighten weight: if >10 kg (B's limit), split into parcels under the hood.
        if weight_kg > 30:
            raise CourierError("OVERWEIGHT", "Over app limit 30kg")
        parcels = []
        remaining = weight_kg
        while remaining > 0:
            this = min(remaining, 10.0)  # B's limit
            parcels.append(this)
            remaining -= this

        try:
            first = True
            tracking_id = None
            eta_hours_total = 0.0
            for kg in parcels:
                rec = self.client.create(addr, kg)
                if rec is None:
                    raise CourierError("INVALID_ADDRESS", "Missing/invalid address for B")
                if first:
                    tracking_id = rec["id"]  # lead tracking id
                    first = False
                eta_hours_total = max(eta_hours_total, rec["eta_minutes"]/60.0)
            return ShipmentLabel(
                tracking_id=tracking_id or "UNKNOWN",
                eta_hours=eta_hours_total,             # normalize to hours
                status=self._map_state(rec["state"])   # state of last parcel; fine for demo
            )
        except self.client.BApiError as e:
            msg = str(e).lower()
            if "over 10kg" in msg:
                # we already split; if this still fires, treat as overweight
                raise CourierError("OVERWEIGHT", str(e))
            raise CourierError("TEMPORARY", str(e))

    def track(self, tracking_id) -> TrackingInfo:
        s = self.client.status(tracking_id)
        status = self._map_state(s["state"])
        return TrackingInfo(status=status, last_seen=str(s.get("seen_at","")))

# ===========================
# shipping_facade.py (app uses only the Courier contract)
# ===========================
def make_courier(name: str) -> Courier:
    if name == "A":
        return CourierAAdapter(CourierAClient())
    elif name == "B":
        return CourierBAdapter(CourierBClient())
    else:
        raise ValueError("Unknown")

def place_shipment(address: dict, weight_kg: float, courier_name: str):
    courier = make_courier(courier_name)
    label = courier.create_shipment(address, weight_kg)
    return {"tracking_id": label.tracking_id, "eta_hours": label.eta_hours, "status": label.status}

def tracking_view(tracking_id: str, courier_name: str):
    courier = make_courier(courier_name)
    t = courier.track(tracking_id)
    return {"status": t.status, "last_seen": t.last_seen}

def can_return(tracking_id: str, courier_name: str) -> bool:
    courier = make_courier(courier_name)
    return courier.track(tracking_id).status == "DELIVERED"
```

### Why this fits LSP (simple words)

* **Do not make inputs stricter.** Our app allows up to **30 kg** and phone optional. The **B adapter** splits heavy parcels and fills a default phone so callers do not change.
* **Do not make outputs weaker.** We always return **hours** for ETA and a **common status** set.
* **Normalize errors.** Callers see `CourierError` with known codes, not random exceptions or `None`.
* **Same behavior, different provider.** Replacing A with B does not change how the rest of the app calls shipping.

---

## Contract test (small but powerful)

Run the **same tests** against any courier. If both pass, you can swap them safely.

```python
def contract_tests(courier):
    # accepts up to 30 kg
    lbl = courier.create_shipment({"pincode":"560001"}, 29.5)
    assert lbl.tracking_id and lbl.eta_hours > 0
    assert lbl.status in {"CREATED","IN_TRANSIT","DELIVERED","RETURNED"}

    # invalid address → standardized error
    try:
        courier.create_shipment({"pincode":""}, 1)
        assert False, "should fail"
    except CourierError as e:
        assert e.code in {"INVALID_ADDRESS","TEMPORARY"}

    # overweight >30 → standardized error
    try:
        courier.create_shipment({"pincode":"560001"}, 31)
        assert False, "should fail"
    except CourierError as e:
        assert e.code == "OVERWEIGHT"

# Run for both:
contract_tests(CourierAAdapter(CourierAClient()))
contract_tests(CourierBAdapter(CourierBClient()))
```

---

## What improves now

* Checkout, tracking, returns, and analytics **use one contract**.
* Switching A ↔ B is a **config change**, not a code rewrite.
* Fewer hidden bugs: same units, same status names, same error types.
* You can add **Courier C** by writing one more adapter and running the **contract tests**.

---

### 1.2.3 Variant explosion (OCP)

**Plain idea**
New “kinds” keep arriving: new payment methods, new promo types, new report formats.
If every new kind forces `if/else` edits across many files, changes are slow and risky.
We want to **add** a new kind by **adding** a small plug-in, **not** by editing old code everywhere.

**Our app**
Think Myntra/Flipkart/Amazon.

**New request (Payments Team)**
“Add **PayLater** now. Next quarter we may add **GiftCard**. Everything else should keep working: checkout, refunds, risk checks, receipts, ledgers, webhooks.”

---

## Before — logic spread with `if/else` in many files

**Problem in one line:**
Payment logic is added in multiple places with `if/else` or switch statements.
Checkout, refunds, risk, receipts, ledgers, and webhooks each do their own thing.

> Demo code is long so you can see the pain. (SDK calls are mocked.)

```python
# ===========================
# fake sdks (stubs for demo)
# ===========================
class CardSDK:
    def charge(self, amount, order_id): return {"payment_id": f"C-{order_id}", "status": "CAPTURED"}
    def refund(self, payment_id, amount): return {"ref_id": f"CR-{payment_id}", "ok": True}

class UpiSDK:
    def pay(self, amount, order_id, vpa): return {"txn": f"U-{order_id}", "state": "SUCCESS"}
    def refund(self, txn, amount): return {"refund_txn": f"UR-{txn}", "state": "SUCCESS"}

class WalletSDK:
    def debit(self, user_id, amount): return {"txn": f"W-{user_id}", "ok": True}
    def credit(self, user_id, amount): return {"txn": f"WR-{user_id}", "ok": True}


# shared helpers
def total_amount(order): return round(sum(i["price"]*i["qty"] for i in order["items"]), 2)


# ===========================
# checkout.py
# ===========================
CARD = CardSDK()
UPI  = UpiSDK()
WALLET = WalletSDK()

def process_payment(order, method):
    amount = total_amount(order)
    if method == "CARD":
        res = CARD.charge(amount, order["id"])
        if res["status"] == "CAPTURED":
            return {"status": "PAID", "payment_id": res["payment_id"]}
        else:
            return {"status": "FAILED"}
    elif method == "UPI":
        # needs VPA from order
        res = UPI.pay(amount, order["id"], order.get("vpa"))
        if res["state"] == "SUCCESS":
            return {"status": "PAID", "payment_id": res["txn"]}
        else:
            return {"status": "FAILED"}
    elif method == "WALLET":
        res = WALLET.debit(order["user_id"], amount)
        return {"status": "PAID" if res["ok"] else "FAILED", "payment_id": res["txn"]}
    else:
        raise ValueError("Unknown method")


# ===========================
# refunds.py
# ===========================
def refund_payment(order, method, payment_id, amount):
    if method == "CARD":
        r = CARD.refund(payment_id, amount)
        return r["ok"]
    elif method == "UPI":
        r = UPI.refund(payment_id, amount)
        return r["state"] == "SUCCESS"
    elif method == "WALLET":
        r = WALLET.credit(order["user_id"], amount)
        return r["ok"]
    else:
        raise ValueError("Unknown method")


# ===========================
# risk.py
# ===========================
def risk_check(order, method):
    amt = total_amount(order)
    # different checks per method, all inline here
    if method == "CARD":
        return amt <= 100000  # allow <= 1L on card
    elif method == "UPI":
        return amt <= 50000   # allow <= 50k on UPI
    elif method == "WALLET":
        return amt <= 20000
    else:
        return False


# ===========================
# receipts.py
# ===========================
def receipt_line(method, payment_id):
    if method == "CARD":
        return f"Paid by Card (ID {payment_id})"
    elif method == "UPI":
        return f"Paid by UPI (Txn {payment_id})"
    elif method == "WALLET":
        return f"Paid by Wallet (Txn {payment_id})"
    else:
        return "Payment method unknown"


# ===========================
# ledger.py
# ===========================
def post_ledger_entries(order, method, payment_result):
    lines = []
    amt = total_amount(order)
    if method == "CARD":
        lines.append(("AR", -amt))
        lines.append(("Cash@CardGateway", amt))
    elif method == "UPI":
        lines.append(("AR", -amt))
        lines.append(("Cash@UPI", amt))
    elif method == "WALLET":
        lines.append(("AR", -amt))
        lines.append(("Cash@Wallet", amt))
    else:
        raise ValueError("Unknown method")
    return lines


# ===========================
# webhooks.py
# ===========================
def handle_webhook(provider, payload):
    # provider sends different shapes
    if provider == "CARD":
        if payload.get("event") == "REFUND_SETTLED":
            return {"type": "REFUND_OK", "id": payload["ref_id"]}
        elif payload.get("event") == "CHARGEBACK":
            return {"type": "CHARGEBACK", "id": payload["payment_id"]}
    elif provider == "UPI":
        if payload.get("status") == "SUCCESS" and payload.get("kind") == "REFUND":
            return {"type": "REFUND_OK", "id": payload["refund_txn"]}
    elif provider == "WALLET":
        if payload.get("ok") and payload.get("kind") == "CREDIT":
            return {"type": "REFUND_OK", "id": payload["txn"]}
    return {"type": "IGNORED"}
```

### What goes wrong here

* You add **PayLater**. You must edit **checkout**, **refunds**, **risk**, **receipts**, **ledger**, and **webhooks**. That is 6 places.
* You later add **GiftCard**. Repeat the same edits in 6 places again.
* Miss one `elif` → checkout works but refunds fail, or ledger is wrong.
* Tests are copied for each method and get out of sync.
* Business asks for **partial refunds** on UPI only → you hard-code more branches.

---

## After — one payment contract + handlers (open for extension)

**Simple plan**

* Define a **PaymentHandler** interface: `pay()`, `refund()`, `receipt_text()`, `risk_ok()`, `ledger_lines()`, `handle_webhook()`.
* Make **handlers** for Card, UPI, Wallet (and later PayLater, GiftCard).
* Keep a **registry** of handlers.
* All flows (checkout, refunds, risk, receipts, ledger, webhooks) call the **same handler** from the registry.
* Adding a new method = **add one handler class + one registry entry**. No edits to old code.

```python
from dataclasses import dataclass
from typing import Dict, List, Protocol

# ---- Shared types ----
@dataclass
class Order:
    id: str
    user_id: str
    items: List[Dict]  # [{"sku":..., "price":..., "qty":...}]
    vpa: str = ""      # UPI id if any

def total_amount(order: Order) -> float:
    return round(sum(i["price"] * i["qty"] for i in order.items), 2)

@dataclass
class PayResult:
    ok: bool
    payment_id: str | None = None
    message: str = ""

# ---- The contract (simple interface) ----
class PaymentHandler(Protocol):
    def pay(self, order: Order) -> PayResult: ...
    def refund(self, order: Order, payment_id: str, amount: float) -> bool: ...
    def risk_ok(self, order: Order) -> bool: ...
    def receipt_text(self, payment_id: str) -> str: ...
    def ledger_lines(self, order: Order, result: PayResult) -> List[tuple[str, float]]: ...
    def handle_webhook(self, payload: Dict) -> Dict: ...

# ---- Concrete handlers (adapters for each method) ----
class CardHandler:
    def __init__(self, sdk: CardSDK): self.sdk = sdk
    def pay(self, order: Order) -> PayResult:
        res = self.sdk.charge(total_amount(order), order.id)
        ok = res.get("status") == "CAPTURED"
        return PayResult(ok=ok, payment_id=res.get("payment_id"), message="card_ok" if ok else "card_fail")
    def refund(self, order: Order, payment_id: str, amount: float) -> bool:
        r = self.sdk.refund(payment_id, amount); return bool(r.get("ok"))
    def risk_ok(self, order: Order) -> bool:
        return total_amount(order) <= 100000
    def receipt_text(self, payment_id: str) -> str:
        return f"Paid by Card (ID {payment_id})"
    def ledger_lines(self, order: Order, result: PayResult) -> List[tuple[str, float]]:
        amt = total_amount(order); return [("AR", -amt), ("Cash@CardGateway", amt)]
    def handle_webhook(self, payload: Dict) -> Dict:
        if payload.get("event") == "REFUND_SETTLED": return {"type":"REFUND_OK","id":payload["ref_id"]}
        if payload.get("event") == "CHARGEBACK": return {"type":"CHARGEBACK","id":payload["payment_id"]}
        return {"type":"IGNORED"}

class UpiHandler:
    def __init__(self, sdk: UpiSDK): self.sdk = sdk
    def pay(self, order: Order) -> PayResult:
        res = self.sdk.pay(total_amount(order), order.id, order.vpa)
        ok = res.get("state") == "SUCCESS"
        return PayResult(ok=ok, payment_id=res.get("txn"), message="upi_ok" if ok else "upi_fail")
    def refund(self, order: Order, payment_id: str, amount: float) -> bool:
        r = self.sdk.refund(payment_id, amount); return r.get("state") == "SUCCESS"
    def risk_ok(self, order: Order) -> bool:
        return total_amount(order) <= 50000
    def receipt_text(self, payment_id: str) -> str:
        return f"Paid by UPI (Txn {payment_id})"
    def ledger_lines(self, order: Order, result: PayResult) -> List[tuple[str, float]]:
        amt = total_amount(order); return [("AR", -amt), ("Cash@UPI", amt)]
    def handle_webhook(self, payload: Dict) -> Dict:
        if payload.get("status") == "SUCCESS" and payload.get("kind") == "REFUND":
            return {"type":"REFUND_OK","id":payload.get("refund_txn")}
        return {"type":"IGNORED"}

class WalletHandler:
    def __init__(self, sdk: WalletSDK): self.sdk = sdk
    def pay(self, order: Order) -> PayResult:
        r = self.sdk.debit(order.user_id, total_amount(order))
        return PayResult(ok=bool(r.get("ok")), payment_id=r.get("txn"), message="wallet_ok" if r.get("ok") else "wallet_fail")
    def refund(self, order: Order, payment_id: str, amount: float) -> bool:
        r = self.sdk.credit(order.user_id, amount); return bool(r.get("ok"))
    def risk_ok(self, order: Order) -> bool:
        return total_amount(order) <= 20000
    def receipt_text(self, payment_id: str) -> str:
        return f"Paid by Wallet (Txn {payment_id})"
    def ledger_lines(self, order: Order, result: PayResult) -> List[tuple[str, float]]:
        amt = total_amount(order); return [("AR", -amt), ("Cash@Wallet", amt)]
    def handle_webhook(self, payload: Dict) -> Dict:
        if payload.get("ok") and payload.get("kind") == "CREDIT":
            return {"type":"REFUND_OK","id":payload.get("txn")}
        return {"type":"IGNORED"}

# ---- Registry (single source of truth) ----
class PaymentRegistry:
    def __init__(self): self._reg: Dict[str, PaymentHandler] = {}
    def register(self, name: str, handler: PaymentHandler): self._reg[name.upper()] = handler
    def get(self, name: str) -> PaymentHandler: return self._reg[name.upper()]

REG = PaymentRegistry()
REG.register("CARD",   CardHandler(CardSDK()))
REG.register("UPI",    UpiHandler(UpiSDK()))
REG.register("WALLET", WalletHandler(WalletSDK()))

# ---- App flows now use the registry (no if/else) ----
def checkout_pay(order: Order, method: str) -> PayResult:
    handler = REG.get(method)
    return handler.pay(order)

def refunds_do(order: Order, method: str, payment_id: str, amount: float) -> bool:
    return REG.get(method).refund(order, payment_id, amount)

def do_risk_check(order: Order, method: str) -> bool:
    return REG.get(method).risk_ok(order)

def build_receipt_line(method: str, payment_id: str) -> str:
    return REG.get(method).receipt_text(payment_id)

def ledger_post(order: Order, method: str, result: PayResult):
    return REG.get(method).ledger_lines(order, result)

def handle_payment_webhook(method: str, payload: Dict) -> Dict:
    return REG.get(method).handle_webhook(payload)
```

### Add a new method (PayLater) — no edits to old code

```python
# New SDK (stub)
class PayLaterSDK:
    def create_plan(self, amount, order_id): return {"plan_id": f"PL-{order_id}", "approved": True}
    def refund(self, plan_id, amount): return {"ok": True}

class PayLaterHandler:
    def __init__(self, sdk: PayLaterSDK): self.sdk = sdk
    def pay(self, order: Order) -> PayResult:
        r = self.sdk.create_plan(total_amount(order), order.id)
        return PayResult(ok=bool(r.get("approved")), payment_id=r.get("plan_id"), message="pl_ok" if r.get("approved") else "pl_fail")
    def refund(self, order: Order, payment_id: str, amount: float) -> bool:
        return bool(self.sdk.refund(payment_id, amount).get("ok"))
    def risk_ok(self, order: Order) -> bool:
        return total_amount(order) <= 75000
    def receipt_text(self, payment_id: str) -> str:
        return f"Paid by PayLater (Plan {payment_id})"
    def ledger_lines(self, order: Order, result: PayResult):
        amt = total_amount(order); return [("AR", -amt), ("Cash@PayLater", amt)]
    def handle_webhook(self, payload: Dict) -> Dict:
        # map provider-specific events to our common shape
        if payload.get("type") == "PLAN_REFUND_OK":
            return {"type": "REFUND_OK", "id": payload.get("ref")}
        return {"type": "IGNORED"}

# Only two lines to enable a new method:
REG.register("PAYLATER", PayLaterHandler(PayLaterSDK()))
# (Optionally from config at startup)
```

That’s it. No changes in `checkout_pay`, `refunds_do`, `do_risk_check`, `build_receipt_line`, `ledger_post`, or `handle_payment_webhook`.
You can add **GiftCard** the same way: create `GiftCardHandler`, register it.

---

### Why this fits OCP (simple words)

* **Open for extension:** You can add a new payment by **adding a new handler class**.
* **Closed for modification:** Existing flows do not need edits. No new `elif` everywhere.
* **One registry:** One place knows which methods exist.
* **One contract:** All handlers provide the same methods, so pages and jobs call them the same way.

---

### What improves now

* New payment methods land faster.
* Fewer bugs from missed branches.
* Tests are reusable: run the **same test suite** against each handler.
* Easier rollbacks: disable a method by removing its registry entry.

---
# SOLID Principles

## Chapter 1: Understanding the Problem

### 1.2.4 Testing friction (SRP, ISP, DIP)

**Plain idea**
Unit tests should be fast and simple.
If tests need a real database, real network, or the real clock, they are slow and flaky.

**Our app**
Think Myntra/Flipkart/Amazon.

**Problem in one line**
One big function does many jobs and talks to real systems directly.
So a “unit” test becomes an integration test by mistake.

---

## Before — one function does everything (hard to test)

*Logic for offers, payments, database, email, and time all live together.
Tests need a DB, API keys, network, and stable time.*

```python
# order_flow_before.py
import os, time, sqlite3
from datetime import datetime

# pretend SDKs / external calls
def upi_pay(amount, order_id, vpa):  # hits network
    # returns {"state": "SUCCESS"|"FAIL", "txn": "U-..."}
    raise NotImplementedError

def send_email(to, subject, body):   # hits SMTP
    raise NotImplementedError

def send_sms(phone, text):           # hits SMS provider
    raise NotImplementedError

def place_order(user, items, coupon, vpa):
    """
    user: {"id": "...", "email": "...", "phone": "...", "tier": "GOLD"|"SILVER"|...}
    items: [{"sku": "TSHIRT", "price": 499.0, "qty": 3, "category": "TOPS"}, ...]
    coupon: "SAVE200" | "SAVE300" | "" ...
    vpa: "name@bank"
    """
    # 1) DB (real) — creates tight coupling
    conn = sqlite3.connect(os.getenv("DB_PATH", "/tmp/shop.db"))
    cur = conn.cursor()
    cur.execute("CREATE TABLE IF NOT EXISTS orders(id TEXT, user_id TEXT, total REAL)")
    order_id = f"O-{int(time.time())}"

    # 2) Pricing (mixed here)
    subtotal = sum(i["price"] * i["qty"] for i in items)

    # VIP percent off (hard-coded)
    if user.get("tier") == "GOLD":
        subtotal *= 0.90  # 10% off

    # flat coupon (manual)
    if coupon == "SAVE200":
        subtotal -= 200
    elif coupon == "SAVE300":
        subtotal -= 300

    # simple B2G1 only for TOPS (rigid)
    for it in items:
        if it.get("category") == "TOPS":
            free = (it["qty"] // 3) * it["price"]
            subtotal -= free

    # shipping (uses subtotal before/after? ambiguous)
    shipping = 0 if subtotal >= 999 else 49
    amount = max(0, round(subtotal + shipping, 2))

    # 3) Time-based offer (uses real clock → nondeterministic)
    now = datetime.now()
    if now.hour < 10:  # morning promo
        amount = round(amount * 0.95, 2)

    # 4) Risk check (inline)
    if amount > 50000 and vpa:  # UPI limit
        raise ValueError("UPI limit exceeded")

    # 5) Payment (network)
    pay_res = upi_pay(amount, order_id, vpa)
    if pay_res["state"] != "SUCCESS":
        send_sms(user["phone"], f"Payment failed for {order_id}")  # network again
        raise RuntimeError("Payment failed")

    # 6) Save order (real DB again)
    cur.execute("INSERT INTO orders(id, user_id, total) VALUES (?, ?, ?)", (order_id, user["id"], amount))
    conn.commit()

    # 7) Email receipt (network + sleep)
    time.sleep(1)  # avoid rate limits (slows tests)
    send_email(user["email"], "Your order", f"Order {order_id} paid {amount}")

    return {"order_id": order_id, "amount": amount, "payment_id": pay_res["txn"]}
```

**Why tests are painful here**

* Needs a **real DB** path, or the code crashes.
* Calls **real network** for UPI, email, SMS (or you have to monkeypatch everything).
* Uses the **real clock**, so totals change by time of day.
* Sleeps for a second (tests become slow).
* Any small change breaks many tests.

---

## After — split responsibilities + small interfaces + fakes (SRP, ISP, DIP)

**What we change (simple words)**

* **SRP (Single Responsibility):** Move pricing, payments, storage, notifications, and time into their **own parts**.
* **ISP (Interface Segregation):** Each part has a **small interface** (only the methods it needs). No giant “do everything” interface.
* **DIP (Dependency Inversion):** `OrderService` depends on **interfaces**, not real SDKs. In tests we pass **fakes**.

```python
# order_flow_after.py
from dataclasses import dataclass
from typing import List, Protocol, Dict, Optional

# --------- Models ---------
@dataclass
class Item:
    sku: str
    price: float
    qty: int
    category: Optional[str] = None

@dataclass
class User:
    id: str
    email: str
    phone: str
    tier: str  # "GOLD", "SILVER", ...

# --------- Small interfaces (IS P) ---------
class Clock(Protocol):
    def now_hour(self) -> int: ...

class Storage(Protocol):
    def save_order(self, order_id: str, user_id: str, total: float) -> None: ...

class Payments(Protocol):
    def pay_upi(self, amount: float, order_id: str, vpa: str) -> Dict: ...

class Notifier(Protocol):
    def email(self, to: str, subject: str, body: str) -> None: ...
    def sms(self, phone: str, text: str) -> None: ...

class Pricing(Protocol):
    def total(self, items: List[Item], user_tier: str, coupon: str) -> float: ...

# --------- Concrete pricing engine (SRP) ---------
class SimplePricing(Pricing):
    def total(self, items: List[Item], user_tier: str, coupon: str) -> float:
        subtotal = sum(i.price * i.qty for i in items)
        if user_tier == "GOLD":
            subtotal *= 0.90
        if coupon == "SAVE200":
            subtotal -= 200
        elif coupon == "SAVE300":
            subtotal -= 300
        for it in items:
            if it.category == "TOPS":
                subtotal -= (it.qty // 3) * it.price
        shipping = 0 if subtotal >= 999 else 49
        return max(0, round(subtotal + shipping, 2))

# --------- Service (depends on interfaces) ---------
class OrderService:
    def __init__(self, pricing: Pricing, clock: Clock, storage: Storage, payments: Payments, notifier: Notifier):
        self.pricing = pricing
        self.clock = clock
        self.storage = storage
        self.payments = payments
        self.notifier = notifier

    def place_order(self, user: User, items: List[Item], coupon: str, vpa: str) -> Dict:
        amount = self.pricing.total(items, user.tier, coupon)

        # time-based rule uses injected clock (deterministic in tests)
        if self.clock.now_hour() < 10:
            amount = round(amount * 0.95, 2)

        # simple risk rule (also testable)
        if amount > 50000 and vpa:
            raise ValueError("UPI limit exceeded")

        order_id = f"O-{user.id}-001"  # simplified for demo

        pay_res = self.payments.pay_upi(amount, order_id, vpa)
        if pay_res.get("state") != "SUCCESS":
            self.notifier.sms(user.phone, f"Payment failed for {order_id}")
            raise RuntimeError("Payment failed")

        self.storage.save_order(order_id, user.id, amount)
        self.notifier.email(user.email, "Your order", f"Order {order_id} paid {amount}")

        return {"order_id": order_id, "amount": amount, "payment_id": pay_res.get("txn")}
```

### Fakes for fast, reliable unit tests

```python
# fakes.py (used only in tests)
class FakeClock:
    def __init__(self, hour: int): self._h = hour
    def now_hour(self) -> int: return self._h

class FakeStorage:
    def __init__(self): self.saved = []
    def save_order(self, order_id, user_id, total): self.saved.append((order_id, user_id, total))

class FakePaymentsOK:
    def pay_upi(self, amount, order_id, vpa): return {"state": "SUCCESS", "txn": f"U-{order_id}"}

class FakePaymentsFail:
    def pay_upi(self, amount, order_id, vpa): return {"state": "FAIL"}

class FakeNotifier:
    def __init__(self): self.emails = []; self.sms_list = []
    def email(self, to, subject, body): self.emails.append((to, subject, body))
    def sms(self, phone, text): self.sms_list.append((phone, text))
```

### Example unit tests (no DB, no network, no sleep)

```python
def test_place_order_morning_discount_ok():
    pricing = SimplePricing()
    svc = OrderService(
        pricing=pricing,
        clock=FakeClock(hour=9),          # deterministic
        storage=FakeStorage(),
        payments=FakePaymentsOK(),        # no network
        notifier=FakeNotifier()           # no SMTP/SMS
    )
    user = User(id="U1", email="a@b.com", phone="9999999999", tier="GOLD")
    items = [Item("TSHIRT", 499.0, 3, "TOPS"), Item("JEANS", 1299.0, 1, "BOTTOMS")]

    res = svc.place_order(user, items, coupon="SAVE200", vpa="name@bank")

    assert res["order_id"].startswith("O-U1")
    assert res["payment_id"].startswith("U-")
    # storage got one row; notifier sent one email; no SMS
    # and test ran in milliseconds

def test_place_order_payment_fail_sends_sms():
    pricing = SimplePricing()
    store = FakeStorage()
    notify = FakeNotifier()
    svc = OrderService(pricing, FakeClock(12), store, FakePaymentsFail(), notify)

    user = User(id="U2", email="c@d.com", phone="8888888888", tier="SILVER")
    items = [Item("TSHIRT", 499.0, 1, "TOPS")]

    try:
        svc.place_order(user, items, coupon="", vpa="name@bank")
        assert False, "Should raise"
    except RuntimeError:
        pass

    assert len(store.saved) == 0              # nothing saved
    assert len(notify.sms_list) == 1          # SMS sent
```

---

## Why this solves the pain (simple words)

* **SRP:** Pricing, time, storage, payments, and notifications are **separate**. Each piece does one job.
* **ISP:** Each interface is **small**. Tests can use only the parts they need.
* **DIP:** The service uses **interfaces**, not real SDKs. Tests pass **fakes**, so no DB, no network, no real clock.

---

## What improves now

* Tests run in **milliseconds** and pass the same way on every machine.
* You can test **error paths** (payment failed) without triggering real errors.
* Refactors are safer: as long as interfaces stay the same, tests keep working.
* CI is faster; developers get feedback quickly.

---
# SOLID Principles

## Chapter 1: Understanding the Problem

### 1.2.4 Testing friction (SRP, ISP, DIP)

**Plain idea**
Unit tests should be fast and simple.
If tests need a real database, real network, or the real clock, they are slow and flaky.

**Our app**
Think Myntra/Flipkart/Amazon.

**Problem in one line**
One big function does many jobs and talks to real systems directly.
So a “unit” test becomes an integration test by mistake.

---

## Before — one function does everything (hard to test)

*Logic for offers, payments, database, email, and time all live together.
Tests need a DB, API keys, network, and stable time.*

```python
# order_flow_before.py
import os, time, sqlite3
from datetime import datetime

# pretend SDKs / external calls
def upi_pay(amount, order_id, vpa):  # hits network
    # returns {"state": "SUCCESS"|"FAIL", "txn": "U-..."}
    raise NotImplementedError

def send_email(to, subject, body):   # hits SMTP
    raise NotImplementedError

def send_sms(phone, text):           # hits SMS provider
    raise NotImplementedError

def place_order(user, items, coupon, vpa):
    """
    user: {"id": "...", "email": "...", "phone": "...", "tier": "GOLD"|"SILVER"|...}
    items: [{"sku": "TSHIRT", "price": 499.0, "qty": 3, "category": "TOPS"}, ...]
    coupon: "SAVE200" | "SAVE300" | "" ...
    vpa: "name@bank"
    """
    # 1) DB (real) — creates tight coupling
    conn = sqlite3.connect(os.getenv("DB_PATH", "/tmp/shop.db"))
    cur = conn.cursor()
    cur.execute("CREATE TABLE IF NOT EXISTS orders(id TEXT, user_id TEXT, total REAL)")
    order_id = f"O-{int(time.time())}"

    # 2) Pricing (mixed here)
    subtotal = sum(i["price"] * i["qty"] for i in items)

    # VIP percent off (hard-coded)
    if user.get("tier") == "GOLD":
        subtotal *= 0.90  # 10% off

    # flat coupon (manual)
    if coupon == "SAVE200":
        subtotal -= 200
    elif coupon == "SAVE300":
        subtotal -= 300

    # simple B2G1 only for TOPS (rigid)
    for it in items:
        if it.get("category") == "TOPS":
            free = (it["qty"] // 3) * it["price"]
            subtotal -= free

    # shipping (uses subtotal before/after? ambiguous)
    shipping = 0 if subtotal >= 999 else 49
    amount = max(0, round(subtotal + shipping, 2))

    # 3) Time-based offer (uses real clock → nondeterministic)
    now = datetime.now()
    if now.hour < 10:  # morning promo
        amount = round(amount * 0.95, 2)

    # 4) Risk check (inline)
    if amount > 50000 and vpa:  # UPI limit
        raise ValueError("UPI limit exceeded")

    # 5) Payment (network)
    pay_res = upi_pay(amount, order_id, vpa)
    if pay_res["state"] != "SUCCESS":
        send_sms(user["phone"], f"Payment failed for {order_id}")  # network again
        raise RuntimeError("Payment failed")

    # 6) Save order (real DB again)
    cur.execute("INSERT INTO orders(id, user_id, total) VALUES (?, ?, ?)", (order_id, user["id"], amount))
    conn.commit()

    # 7) Email receipt (network + sleep)
    time.sleep(1)  # avoid rate limits (slows tests)
    send_email(user["email"], "Your order", f"Order {order_id} paid {amount}")

    return {"order_id": order_id, "amount": amount, "payment_id": pay_res["txn"]}
```

**Why tests are painful here**

* Needs a **real DB** path, or the code crashes.
* Calls **real network** for UPI, email, SMS (or you have to monkeypatch everything).
* Uses the **real clock**, so totals change by time of day.
* Sleeps for a second (tests become slow).
* Any small change breaks many tests.

---

## After — split responsibilities + small interfaces + fakes (SRP, ISP, DIP)

**What we change (simple words)**

* **SRP (Single Responsibility):** Move pricing, payments, storage, notifications, and time into their **own parts**.
* **ISP (Interface Segregation):** Each part has a **small interface** (only the methods it needs). No giant “do everything” interface.
* **DIP (Dependency Inversion):** `OrderService` depends on **interfaces**, not real SDKs. In tests we pass **fakes**.

```python
# order_flow_after.py
from dataclasses import dataclass
from typing import List, Protocol, Dict, Optional

# --------- Models ---------
@dataclass
class Item:
    sku: str
    price: float
    qty: int
    category: Optional[str] = None

@dataclass
class User:
    id: str
    email: str
    phone: str
    tier: str  # "GOLD", "SILVER", ...

# --------- Small interfaces (IS P) ---------
class Clock(Protocol):
    def now_hour(self) -> int: ...

class Storage(Protocol):
    def save_order(self, order_id: str, user_id: str, total: float) -> None: ...

class Payments(Protocol):
    def pay_upi(self, amount: float, order_id: str, vpa: str) -> Dict: ...

class Notifier(Protocol):
    def email(self, to: str, subject: str, body: str) -> None: ...
    def sms(self, phone: str, text: str) -> None: ...

class Pricing(Protocol):
    def total(self, items: List[Item], user_tier: str, coupon: str) -> float: ...

# --------- Concrete pricing engine (SRP) ---------
class SimplePricing(Pricing):
    def total(self, items: List[Item], user_tier: str, coupon: str) -> float:
        subtotal = sum(i.price * i.qty for i in items)
        if user_tier == "GOLD":
            subtotal *= 0.90
        if coupon == "SAVE200":
            subtotal -= 200
        elif coupon == "SAVE300":
            subtotal -= 300
        for it in items:
            if it.category == "TOPS":
                subtotal -= (it.qty // 3) * it.price
        shipping = 0 if subtotal >= 999 else 49
        return max(0, round(subtotal + shipping, 2))

# --------- Service (depends on interfaces) ---------
class OrderService:
    def __init__(self, pricing: Pricing, clock: Clock, storage: Storage, payments: Payments, notifier: Notifier):
        self.pricing = pricing
        self.clock = clock
        self.storage = storage
        self.payments = payments
        self.notifier = notifier

    def place_order(self, user: User, items: List[Item], coupon: str, vpa: str) -> Dict:
        amount = self.pricing.total(items, user.tier, coupon)

        # time-based rule uses injected clock (deterministic in tests)
        if self.clock.now_hour() < 10:
            amount = round(amount * 0.95, 2)

        # simple risk rule (also testable)
        if amount > 50000 and vpa:
            raise ValueError("UPI limit exceeded")

        order_id = f"O-{user.id}-001"  # simplified for demo

        pay_res = self.payments.pay_upi(amount, order_id, vpa)
        if pay_res.get("state") != "SUCCESS":
            self.notifier.sms(user.phone, f"Payment failed for {order_id}")
            raise RuntimeError("Payment failed")

        self.storage.save_order(order_id, user.id, amount)
        self.notifier.email(user.email, "Your order", f"Order {order_id} paid {amount}")

        return {"order_id": order_id, "amount": amount, "payment_id": pay_res.get("txn")}
```

### Fakes for fast, reliable unit tests

```python
# fakes.py (used only in tests)
class FakeClock:
    def __init__(self, hour: int): self._h = hour
    def now_hour(self) -> int: return self._h

class FakeStorage:
    def __init__(self): self.saved = []
    def save_order(self, order_id, user_id, total): self.saved.append((order_id, user_id, total))

class FakePaymentsOK:
    def pay_upi(self, amount, order_id, vpa): return {"state": "SUCCESS", "txn": f"U-{order_id}"}

class FakePaymentsFail:
    def pay_upi(self, amount, order_id, vpa): return {"state": "FAIL"}

class FakeNotifier:
    def __init__(self): self.emails = []; self.sms_list = []
    def email(self, to, subject, body): self.emails.append((to, subject, body))
    def sms(self, phone, text): self.sms_list.append((phone, text))
```

### Example unit tests (no DB, no network, no sleep)

```python
def test_place_order_morning_discount_ok():
    pricing = SimplePricing()
    svc = OrderService(
        pricing=pricing,
        clock=FakeClock(hour=9),          # deterministic
        storage=FakeStorage(),
        payments=FakePaymentsOK(),        # no network
        notifier=FakeNotifier()           # no SMTP/SMS
    )
    user = User(id="U1", email="a@b.com", phone="9999999999", tier="GOLD")
    items = [Item("TSHIRT", 499.0, 3, "TOPS"), Item("JEANS", 1299.0, 1, "BOTTOMS")]

    res = svc.place_order(user, items, coupon="SAVE200", vpa="name@bank")

    assert res["order_id"].startswith("O-U1")
    assert res["payment_id"].startswith("U-")
    # storage got one row; notifier sent one email; no SMS
    # and test ran in milliseconds

def test_place_order_payment_fail_sends_sms():
    pricing = SimplePricing()
    store = FakeStorage()
    notify = FakeNotifier()
    svc = OrderService(pricing, FakeClock(12), store, FakePaymentsFail(), notify)

    user = User(id="U2", email="c@d.com", phone="8888888888", tier="SILVER")
    items = [Item("TSHIRT", 499.0, 1, "TOPS")]

    try:
        svc.place_order(user, items, coupon="", vpa="name@bank")
        assert False, "Should raise"
    except RuntimeError:
        pass

    assert len(store.saved) == 0              # nothing saved
    assert len(notify.sms_list) == 1          # SMS sent
```

---

## Why this solves the pain (simple words)

* **SRP:** Pricing, time, storage, payments, and notifications are **separate**. Each piece does one job.
* **ISP:** Each interface is **small**. Tests can use only the parts they need.
* **DIP:** The service uses **interfaces**, not real SDKs. Tests pass **fakes**, so no DB, no network, no real clock.

---

## What improves now

* Tests run in **milliseconds** and pass the same way on every machine.
* You can test **error paths** (payment failed) without triggering real errors.
* Refactors are safer: as long as interfaces stay the same, tests keep working.
* CI is faster; developers get feedback quickly.

---
# SOLID Principles

## Chapter 1: Understanding the Problem

### 1.2.5 Team drag & merge conflicts (SRP, ISP)

**Plain idea**
When one file or class does too many jobs, every team has to edit the same place.
People bump into each other, reviews are slow, and merge conflicts happen again and again.

**Our app**
Think Myntra/Flipkart/Amazon.

**Real situation**

* **Marketing Team** adds a new offer.
* **Payments Team** adds a refund rule.
* **Logistics Team** tweaks shipping.
  All three open the same file. Boom—conflicts.

---

## Before — one “God” module everyone edits

**Problem in one line**
Logic for pricing, payment, shipping, returns, emails, analytics sits together.
There is also a **fat interface** for payments: it has many methods most providers don’t need, so teams keep changing the interface too.

> The code is long on purpose so you can see how many teams must touch it.

```python
# order_module.py  (one big hotspot; everyone edits this file)

import time
from typing import Dict, List, Optional

# ---- Fat payment interface (too many methods; ISP violation) ----
class PaymentProvider:
    def charge_card(self, amount: float, order_id: str, card_token: str) -> Dict: ...
    def pay_upi(self, amount: float, order_id: str, vpa: str) -> Dict: ...
    def debit_wallet(self, user_id: str, amount: float) -> Dict: ...
    def refund_card(self, payment_id: str, amount: float) -> Dict: ...
    def refund_upi(self, txn: str, amount: float) -> Dict: ...
    def refund_wallet(self, user_id: str, amount: float) -> Dict: ...
    def capture_later(self, ref: str) -> Dict: ...
    def void(self, ref: str) -> Dict: ...
    # providers that do not support a method still must implement it (with pass/raise)

# ---- One big class (SRP violation) ----
class OrderModule:
    def __init__(self, payment: PaymentProvider):
        self.payment = payment

    # MARKETING edits here (offers)
    def compute_total(self, items: List[Dict], user_tier: str, coupon: str) -> float:
        subtotal = sum(i["price"] * i["qty"] for i in items)

        # VIP % off (hard-coded)
        if user_tier == "GOLD":
            subtotal *= 0.90

        # flat coupons
        if coupon == "SAVE200":
            subtotal -= 200
        elif coupon == "SAVE300":
            subtotal -= 300

        # B2G1 for TOPS only
        for it in items:
            if it.get("category") == "TOPS":
                free = (it["qty"] // 3) * it["price"]
                subtotal -= free

        # shipping
        shipping = 0 if subtotal >= 999 else 49
        return max(0, round(subtotal + shipping, 2))

    # PAYMENTS edits here
    def take_payment(self, method: str, order: Dict) -> Dict:
        amount = order["amount"]
        if method == "CARD":
            res = self.payment.charge_card(amount, order["id"], order["card_token"])
            if res.get("status") == "CAPTURED":
                return {"ok": True, "payment_id": res["payment_id"], "method": "CARD"}
            return {"ok": False}
        elif method == "UPI":
            res = self.payment.pay_upi(amount, order["id"], order["vpa"])
            if res.get("state") == "SUCCESS":
                return {"ok": True, "payment_id": res["txn"], "method": "UPI"}
            return {"ok": False}
        elif method == "WALLET":
            res = self.payment.debit_wallet(order["user_id"], amount)
            return {"ok": res.get("ok"), "payment_id": res.get("txn"), "method": "WALLET"}
        else:
            return {"ok": False}

    # LOGISTICS edits here
    def book_shipment(self, address: Dict, weight_kg: float) -> Dict:
        # pretend logic: later someone adds new courier rules here
        if not address.get("pincode"):
            raise ValueError("bad address")
        # hard-coded limit
        if weight_kg > 30:
            raise ValueError("too heavy")
        eta_hours = 36.0
        return {"tracking_id": f"T-{int(time.time())}", "eta_hours": eta_hours}

    # CX/EMAIL edits here
    def send_email(self, to: str, subject: str, body: str) -> None:
        # later someone changes SMTP handling here
        pass

    # ANALYTICS edits here
    def track(self, event: str, payload: Dict) -> None:
        # later someone changes fields here
        pass

    # RETURNS edits here
    def refund(self, method: str, payment_id: str, amount: float, order: Dict) -> bool:
        if method == "CARD":
            return bool(self.payment.refund_card(payment_id, amount).get("ok"))
        elif method == "UPI":
            return self.payment.refund_upi(payment_id, amount).get("state") == "SUCCESS"
        elif method == "WALLET":
            return bool(self.payment.refund_wallet(order["user_id"], amount).get("ok"))
        return False

    # ORCHESTRATION (everyone depends on this)
    def place_order(self, user: Dict, items: List[Dict], coupon: str, method: str, pay_info: Dict, address: Dict) -> Dict:
        amount = self.compute_total(items, user.get("tier",""), coupon)
        order = {"id": f"O-{int(time.time())}", "user_id": user["id"], "amount": amount, **pay_info}

        pay_res = self.take_payment(method, order)
        if not pay_res["ok"]:
            self.track("PAYMENT_FAILED", {"order": order["id"], "amount": amount})
            return {"ok": False, "error": "payment_failed"}

        ship = self.book_shipment(address, weight_kg=sum(i.get("kg", 0)*i["qty"] for i in items))
        self.send_email(user["email"], "Your order", f"Order {order['id']} paid {amount}\nTracking {ship['tracking_id']}")
        self.track("ORDER_PAID", {"order": order["id"], "amount": amount, "method": method})
        return {"ok": True, "order_id": order["id"], "payment_id": pay_res["payment_id"], "tracking_id": ship["tracking_id"]}
```

### What goes wrong here

* Every small change touches **order_module.py**. Three teams change the same file → **merge conflicts**.
* The **fat PaymentProvider** forces providers to implement unused methods, or teams keep changing the interface → more conflicts.
* Reviews are slow because the file is big and risky.
* A bug in one part (say analytics) blocks a payment hotfix because it’s the same file.

---

## After — small files, small interfaces, clear owners (SRP + ISP)

**Simple plan**

* Split by responsibility: Pricing, Payments, Shipping, Notifications, Analytics.
* Each part has a **small interface** (only what the caller needs).
* A thin **OrderService** coordinates them but owns no details.
* Teams own their piece; they don’t edit each other’s files.

```python
# contracts.py  (small interfaces)

from typing import Dict, List, Protocol

class Pricing(Protocol):
    def total(self, items: List[Dict], user_tier: str, coupon: str) -> float: ...

class Payments(Protocol):
    def pay(self, amount: float, order_id: str, method: str, info: Dict) -> Dict: ...
    def refund(self, payment_id: str, amount: float, method: str, info: Dict) -> bool: ...

class Shipping(Protocol):
    def create(self, address: Dict, weight_kg: float) -> Dict: ...

class Notifier(Protocol):
    def email(self, to: str, subject: str, body: str) -> None: ...

class Analytics(Protocol):
    def track(self, event: str, payload: Dict) -> None: ...
```

```python
# pricing.py  (Marketing Team owns this file)

class PricingEngine:
    def total(self, items, user_tier, coupon) -> float:
        subtotal = sum(i["price"] * i["qty"] for i in items)
        if user_tier == "GOLD":
            subtotal *= 0.90
        if coupon == "SAVE200":
            subtotal -= 200
        elif coupon == "SAVE300":
            subtotal -= 300
        for it in items:
            if it.get("category") == "TOPS":
                subtotal -= (it["qty"] // 3) * it["price"]
        shipping = 0 if subtotal >= 999 else 49
        return max(0, round(subtotal + shipping, 2))
```

```python
# payments.py  (Payments Team owns this file)

from typing import Dict, List

def total(order_items: List[Dict]) -> float:
    return round(sum(i["price"]*i["qty"] for i in order_items), 2)

class CardSDK: ...
class UpiSDK: ...
class WalletSDK: ...

class PaymentHandlers:
    def __init__(self):
        self.card = CardSDK()
        self.upi = UpiSDK()
        self.wallet = WalletSDK()

    def pay(self, amount: float, order_id: str, method: str, info: Dict) -> Dict:
        if method == "CARD":
            r = self.card.charge(amount, order_id, info["card_token"])
            return {"ok": r.get("status") == "CAPTURED", "payment_id": r.get("payment_id")}
        if method == "UPI":
            r = self.upi.pay(amount, order_id, info["vpa"])
            return {"ok": r.get("state") == "SUCCESS", "payment_id": r.get("txn")}
        if method == "WALLET":
            r = self.wallet.debit(info["user_id"], amount)
            return {"ok": bool(r.get("ok")), "payment_id": r.get("txn")}
        return {"ok": False}

    def refund(self, payment_id: str, amount: float, method: str, info: Dict) -> bool:
        if method == "CARD":
            return bool(self.card.refund(payment_id, amount).get("ok"))
        if method == "UPI":
            return self.upi.refund(payment_id, amount).get("state") == "SUCCESS"
        if method == "WALLET":
            return bool(self.wallet.credit(info["user_id"], amount).get("ok"))
        return False
```

```python
# shipping.py  (Logistics Team owns this file)

class ShippingService:
    LIMIT_KG = 30

    def create(self, address, weight_kg):
        if not address.get("pincode"):
            raise ValueError("bad address")
        if weight_kg > self.LIMIT_KG:
            raise ValueError("too heavy")
        return {"tracking_id": "T-123", "eta_hours": 36.0}
```

```python
# notify.py  (CX/Email Team owns this file)

class EmailNotifier:
    def email(self, to, subject, body):
        # call SMTP or provider
        pass
```

```python
# analytics.py  (Analytics Team owns this file)

class Tracker:
    def track(self, event, payload):
        # send to analytics backend
        pass
```

```python
# order_service.py  (thin coordinator; rarely changes)

from typing import Dict, List

class OrderService:
    def __init__(self, pricing, payments, shipping, notifier, analytics):
        self.pricing = pricing
        self.payments = payments
        self.shipping = shipping
        self.notifier = notifier
        self.analytics = analytics

    def place_order(self, user: Dict, items: List[Dict], coupon: str, method: str, pay_info: Dict, address: Dict):
        amount = self.pricing.total(items, user.get("tier",""), coupon)
        order_id = f"O-{user['id']}"
        pay = self.payments.pay(amount, order_id, method, {**pay_info, "user_id": user["id"]})
        if not pay["ok"]:
            self.analytics.track("PAYMENT_FAILED", {"order": order_id, "amount": amount})
            return {"ok": False, "error": "payment_failed"}

        ship = self.shipping.create(address, weight_kg=sum(i.get("kg",0)*i["qty"] for i in items))
        self.notifier.email(user["email"], "Your order", f"Order {order_id} paid {amount}\nTracking {ship['tracking_id']}")
        self.analytics.track("ORDER_PAID", {"order": order_id, "amount": amount, "method": method})
        return {"ok": True, "order_id": order_id, "payment_id": pay["payment_id"], "tracking_id": ship["tracking_id"]}
```

### Why this removes team drag (simple words)

* **SRP:** Each file has **one job**. Marketing edits `pricing.py`; Payments edits `payments.py`; Logistics edits `shipping.py`.
* **ISP:** Interfaces are **small**. Payments exposes only `pay()` and `refund()`. No one is forced to implement unused methods.
* **Thin coordinator:** `order_service.py` changes rarely, so fewer conflicts.
* **Parallel work:** Teams can commit and test their piece without blocking others.

---

## A quick walk-through: two teams change things at the same time

* **Marketing Team**: changes `GOLD` from **10%** to **15%** → edit **pricing.py** only.
* **Logistics Team**: sets max weight to **25 kg** → edit **shipping.py** only.
  No merge conflict. No hunt through a 500-line file. Reviews are short and clear.

---

## What improves now

* Fewer conflicts, faster reviews.
* New features land without stepping on other teams’ work.
* Rollbacks are localized (revert one file).
* Ownership is clear; each team builds expertise in its area.


---

# SOLID Principles

## Chapter 1: Understanding the Problem

### 1.2.6 Vendor/tech lock-in & ops risk (OCP, DIP)

**Plain idea**
If your code talks to a vendor (SMS, email, search, storage) **directly**, you get locked in.
When the vendor is slow, down, or expensive, you cannot switch fast.
We want to **depend on an interface**, not on one vendor.
We want to **add** a new vendor by **adding** a small adapter, not by editing old code.

**Our app**
Think Myntra/Flipkart/Amazon.

**New request (Ops Team)**
“Move SMS from **Twilio** to **FastSMS** for India. Keep Twilio as fallback.
Next month, try **CheapSMS** (A/B test 10%).
If the primary fails, auto-fallback. Add timeouts, retries, and simple rate limit.”

---

## Before — direct vendor calls spread across the app

**Problem in one line**
Code calls vendors **everywhere** with their own SDK shapes, keys, and errors.
Each place has its own retry/timeout logic.
Changing vendor = edit **many** files.

> Long code on purpose so you can see the pain.

```python
# ===========================
# signup_otp.py  (OTP flow)
# ===========================
import time
from random import randint

class TwilioSDK:
    def __init__(self, sid, token): self.sid, self.token = sid, token
    def send_sms(self, to, text):
        # returns {"ok": True/False, "err": "..."}
        raise NotImplementedError

TWILIO = TwilioSDK("sid", "token")

def send_signup_otp(phone):
    otp = str(randint(100000, 999999))
    text = f"Your OTP is {otp}"

    # Hard-coded vendor
    for attempt in range(3):
        res = TWILIO.send_sms(phone, text)
        if res.get("ok"):
            return {"ok": True, "otp": otp}
        time.sleep(0.5)  # local retry logic
    return {"ok": False, "error": "twilio_failed"}


# ===========================
# order_updates.py (order status SMS)
# ===========================
class FastSmsAPI:
    def __init__(self, key): self.key = key
    def push(self, msisdn, body):
        # returns {"status": "SENT"|"FAILED", "code": "..."}
        raise NotImplementedError

FASTSMS = FastSmsAPI("key-1")

def send_order_placed(phone, order_id):
    body = f"Order {order_id} placed."
    # Different vendor here, different shape
    r = FASTSMS.push(phone, body)
    if r.get("status") != "SENT":
        # No fallback here
        return {"ok": False, "error": r.get("code", "unknown")}
    return {"ok": True}


# ===========================
# marketing_blast.py (bulk SMS)
# ===========================
def send_marketing_sms(phones, msg):
    # Uses Twilio again (inconsistent with order updates)
    failed = []
    for p in phones:
        res = TWILIO.send_sms(p, msg)  # hard-coded
        if not res.get("ok"):
            failed.append(p)
        time.sleep(0.2)  # naive rate limit, slows everything
    return {"ok": len(failed) == 0, "failed": failed}


# ===========================
# notify_email.py (email)
# ===========================
class SendGrid:
    def __init__(self, api_key): self.api_key = api_key
    def send(self, to, subject, body):
        # returns {"success": True/False}
        raise NotImplementedError

SENDGRID = SendGrid("sg-key")

def send_receipt(to, order_id, amount):
    subject = "Your order"
    body = f"Order {order_id} paid ₹{amount}"
    r = SENDGRID.send(to, subject, body)
    if not r.get("success"):
        return {"ok": False}
    return {"ok": True}
```

### What goes wrong here

* To switch SMS vendor you must edit **signup_otp.py**, **order_updates.py**, **marketing_blast.py** (and more you forgot).
* Each file has **different** retry/timeout logic. Some have fallback, some don’t.
* Rate limiting is copy-pasted. If a vendor is down, you scramble to patch many places.
* A/B test is hard: no central place to choose providers by percentage or region.
* Adding **CheapSMS** means touching all files again.

---

## After — one notifier interface + adapters + router (open for extension)

**Simple plan (plain words)**

* Make a **Notifier** interface: `sms(to, text)` and `email(to, subject, body)`.
* Write small **adapters** for Twilio, FastSMS, CheapSMS, SendGrid, SES.
* Use a **Router** that picks a provider (primary/secondary, A/B %) with a **config**.
* The Router handles **timeouts, retries, fallback, and rate limit** in one place.
* App code calls **NotifierService** only.
* Add a new vendor by **adding an adapter** and a **config entry**. No edits to flows.

```python
# notifier_contracts.py
from typing import Protocol, Dict

class SmsProvider(Protocol):
    def sms(self, to: str, text: str) -> Dict: ...  # {"ok": bool, "error": str|None}

class EmailProvider(Protocol):
    def email(self, to: str, subject: str, body: str) -> Dict: ...

class Notifier(Protocol):
    def sms(self, to: str, text: str) -> Dict: ...
    def email(self, to: str, subject: str, body: str) -> Dict: ...
```

```python
# sms_providers.py (adapters)
import time

# Stubs of vendor SDKs
class TwilioSDK: ...
class FastSmsAPI: ...
class CheapSmsAPI: ...

class TwilioAdapter:
    def __init__(self, sdk: TwilioSDK): self.sdk = sdk
    def sms(self, to, text):
        try:
            r = self.sdk.send_sms(to, text)
            return {"ok": bool(r.get("ok")), "error": None if r.get("ok") else r.get("err","unknown")}
        except Exception as e:
            return {"ok": False, "error": str(e)}

class FastSmsAdapter:
    def __init__(self, api: FastSmsAPI): self.api = api
    def sms(self, to, text):
        r = self.api.push(to, text)
        return {"ok": r.get("status") == "SENT", "error": None if r.get("status") == "SENT" else r.get("code","unknown")}

class CheapSmsAdapter:
    def __init__(self, api: CheapSmsAPI): self.api = api
    def sms(self, to, text):
        r = self.api.fire(to, text)  # suppose this method exists
        return {"ok": r.get("ok", False), "error": r.get("err")}
```

```python
# email_providers.py (adapters)
class SendGrid: ...
class SES: ...

class SendGridAdapter:
    def __init__(self, sg: SendGrid): self.sg = sg
    def email(self, to, subject, body):
        r = self.sg.send(to, subject, body)
        return {"ok": bool(r.get("success")), "error": None if r.get("success") else "sendgrid_fail"}

class SesAdapter:
    def __init__(self, ses: SES): self.ses = ses
    def email(self, to, subject, body):
        r = self.ses.send_mail(to, subject, body)
        return {"ok": r.get("ok", False), "error": None if r.get("ok") else "ses_fail"}
```

```python
# router.py  (one place for choice, timeout, retry, fallback, rate limit)
import time
from typing import Dict, List, Tuple, Callable
from random import random

class RateLimiter:
    def __init__(self, per_second: int):
        self.per_second = per_second
        self.ts = []
    def allow(self) -> bool:
        now = time.time()
        self.ts = [t for t in self.ts if now - t < 1.0]
        if len(self.ts) < self.per_second:
            self.ts.append(now)
            return True
        return False

def with_timeout(fn: Callable[[], Dict], seconds: float) -> Dict:
    # demo: pretend timeout by time.sleep check (real code uses threads/async)
    start = time.time()
    res = fn()
    if time.time() - start > seconds:
        return {"ok": False, "error": "timeout"}
    return res

class NotifierRouter:
    def __init__(self,
                 sms_primary,
                 sms_secondary,
                 email_primary,
                 email_secondary,
                 sms_ab_percent: float = 0.0,   # e.g., 0.1 for 10% to secondary
                 rate_limit_per_sec: int = 10,
                 retries: int = 2,
                 timeout_sec: float = 1.0):
        self.sms_primary = sms_primary
        self.sms_secondary = sms_secondary
        self.email_primary = email_primary
        self.email_secondary = email_secondary
        self.sms_ab_percent = sms_ab_percent
        self.retries = retries
        self.timeout_sec = timeout_sec
        self.limiter = RateLimiter(rate_limit_per_sec)

    # unified Notifier interface
    def sms(self, to: str, text: str) -> Dict:
        if not self.limiter.allow():
            return {"ok": False, "error": "rate_limited"}

        # A/B split
        provider = self.sms_secondary if random() < self.sms_ab_percent else self.sms_primary

        # try chosen provider with retries, then fallback
        for attempt in range(self.retries + 1):
            res = with_timeout(lambda: provider.sms(to, text), self.timeout_sec)
            if res.get("ok"):
                return res
            time.sleep(0.1)
        # fallback provider
        fb = with_timeout(lambda: self.sms_secondary.sms(to, text), self.timeout_sec)
        return fb

    def email(self, to: str, subject: str, body: str) -> Dict:
        if not self.limiter.allow():
            return {"ok": False, "error": "rate_limited"}
        for attempt in range(self.retries + 1):
            res = with_timeout(lambda: self.email_primary.email(to, subject, body), self.timeout_sec)
            if res.get("ok"):
                return res
            time.sleep(0.1)
        return with_timeout(lambda: self.email_secondary.email(to, subject, body), self.timeout_sec)
```

```python
# wiring.py  (one config; easy to change or load from env)
from sms_providers import TwilioAdapter, FastSmsAdapter, CheapSmsAdapter, TwilioSDK, FastSmsAPI, CheapSmsAPI
from email_providers import SendGridAdapter, SesAdapter, SendGrid, SES
from router import NotifierRouter

CONFIG = {
    "sms": {
        "primary":  "FASTSMS",
        "secondary":"TWILIO",
        "ab_percent": 0.10  # send 10% to secondary to test quality/cost
    },
    "email": {
        "primary":  "SENDGRID",
        "secondary":"SES"
    },
    "limits": {"per_sec": 20},
    "timeout_sec": 1.0,
    "retries": 2
}

# build providers
PROVIDERS = {
    "TWILIO":  TwilioAdapter(TwilioSDK("sid", "token")),
    "FASTSMS": FastSmsAdapter(FastSmsAPI("key-1")),
    "CHEAP":   CheapSmsAdapter(CheapSmsAPI("k2")),
    "SENDGRID":SendGridAdapter(SendGrid("sg-key")),
    "SES":     SesAdapter(SES("aws-key")),
}

def make_notifier():
    sms_p = PROVIDERS[CONFIG["sms"]["primary"]]
    sms_s = PROVIDERS[CONFIG["sms"]["secondary"]]
    email_p = PROVIDERS[CONFIG["email"]["primary"]]
    email_s = PROVIDERS[CONFIG["email"]["secondary"]]
    return NotifierRouter(
        sms_primary=sms_p,
        sms_secondary=sms_s,
        email_primary=email_p,
        email_secondary=email_s,
        sms_ab_percent=CONFIG["sms"]["ab_percent"],
        rate_limit_per_sec=CONFIG["limits"]["per_sec"],
        retries=CONFIG["retries"],
        timeout_sec=CONFIG["timeout_sec"],
    )

NOTIFIER = make_notifier()
```

```python
# app_flows.py  (all flows call the same Notifier; no vendor code here)

from wiring import NOTIFIER

def send_signup_otp(phone, otp):
    return NOTIFIER.sms(phone, f"Your OTP is {otp}")

def send_order_placed(phone, order_id):
    return NOTIFIER.sms(phone, f"Order {order_id} placed.")

def send_receipt(email, order_id, amount):
    return NOTIFIER.email(email, "Your order", f"Order {order_id} paid ₹{amount}")

def send_marketing_sms(phones, msg):
    results = []
    for p in phones:
        results.append(NOTIFIER.sms(p, msg))
    return results
```

### What improves now

* **Open for extension (OCP):** Add **CheapSMS** by adding its adapter + one config key. No edits to signup, order, or marketing flows.
* **Reduced risk (DIP):** App depends on **Notifier** (an idea), not on Twilio/FastSMS shapes.
* **One place** for ops rules: timeout, retry, fallback, A/B %, and rate limit live in the **Router**.
* **Faster rollouts:** Flip config to test a new vendor for 10% traffic; increase to 100% later.
* **Safer outages:** If primary fails, fallback kicks in automatically.
* **Cleaner tests:** You can pass a **FakeSmsProvider** to the Router and test flows without network.

---

