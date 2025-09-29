# 1.2.5 Team Drag & Merge Conflicts (SRP, ISP)

When every feature funnels through the same 1,500-line module, teams trip over each other. Reviews stall, merge conflicts explode, and “just one more flag” tweaks land in the wrong place. Single Responsibility and Interface Segregation shrink hotspots so teams can ship in parallel.

## Plain idea

- Split giant service classes into cohesive components with clear owners.
- Expose narrow interfaces so each team touches only the capabilities they own.
- Encapsulate wiring in composition roots or factories instead of scattering imports.

## Scenario

- **Weekly standup**: Checkout, pricing, and loyalty squads all edit `order_service.py`. Pull requests queue behind each other because everyone touches the same methods.
- **Accidental regressions**: A quick change for loyalty accidentally breaks marketplace orders because the method handled both concerns.

## Before — one god class to rule them all

```python
# order_service.py
class OrderService:
	def __init__(self, db, analytics, loyalty_client, inventory_client):
		self.db = db
		self.analytics = analytics
		self.loyalty_client = loyalty_client
		self.inventory = inventory_client

	def place_order(self, payload):
		self._reserve_inventory(payload)
		self._apply_loyalty(payload)
		self._persist(payload)
		self._notify_analytics(payload)
		return {"status": "PLACED"}

	def _reserve_inventory(self, payload):
		# Marketplace team tweaks this block weekly
		for line in payload["items"]:
			self.inventory.reserve(line["sku"], line["qty"])

	def _apply_loyalty(self, payload):
		# Loyalty squad edits here
		if payload.get("loyalty_id"):
			points = payload["total"] * 2
			self.loyalty_client.award(payload["loyalty_id"], points)

	def _persist(self, payload):
		# Core checkout team edits this
		self.db.execute("INSERT INTO orders ...", payload)

	def _notify_analytics(self, payload):
		self.analytics.track("order", payload)
```

### What goes wrong

- **Conflicts**: Every squad pushes to the same file, so even unrelated changes collide.
- **Hidden coupling**: Private helper methods reach into shared state, so edits require understanding unrelated logic.
- **Risky reviews**: Reviewers see 30-file diffs because one tweak touches the mega-class plus scattered tests.

## After — cohesive collaborators with clear seams

```python
# ports.py
from typing import Protocol, Iterable


class InventoryPort(Protocol):
	def reserve(self, lines: Iterable[dict]) -> None: ...


class LoyaltyPort(Protocol):
	def award(self, loyalty_id: str, order_total: float) -> None: ...


class OrderWriter(Protocol):
	def save(self, payload: dict) -> None: ...


class AnalyticsPort(Protocol):
	def order_placed(self, payload: dict) -> None: ...


class OrderWorkflow:
	def __init__(self, inventory: InventoryPort, loyalty: LoyaltyPort, writer: OrderWriter, analytics: AnalyticsPort):
		self.inventory = inventory
		self.loyalty = loyalty
		self.writer = writer
		self.analytics = analytics

	def place(self, payload: dict) -> dict:
		self.inventory.reserve(payload["items"])
		if payload.get("loyalty_id"):
			self.loyalty.award(payload["loyalty_id"], payload["total"])
		self.writer.save(payload)
		self.analytics.order_placed(payload)
		return {"status": "PLACED"}
```

Each squad owns its adapter without editing the workflow class.

```python
# inventory_adapter.py (Marketplace team)
class InventoryAdapter:
	def __init__(self, client):
		self.client = client

	def reserve(self, lines):
		for line in lines:
			self.client.block(line["sku"], line["qty"])


# loyalty_adapter.py (Loyalty team)
class LoyaltyAdapter:
	def __init__(self, service):
		self.service = service

	def award(self, loyalty_id, order_total):
		points = int(order_total // 100) * 10
		self.service.record_award(loyalty_id, points)


# composition.py
def make_order_workflow():
	return OrderWorkflow(
		inventory=InventoryAdapter(real_inventory_client()),
		loyalty=LoyaltyAdapter(real_loyalty_service()),
		writer=SqlOrderWriter(real_db()),
		analytics=SegmentAnalytics(real_segment()),
	)
```

### Why it works better

- **Single Responsibility**: The workflow coordinates; adapters own integration details. Each team edits its file, not the shared core.
- **Interface Segregation**: Ports only expose methods a collaborator needs. Loyalty doesn’t see inventory methods, so interfaces stay stable.
- **Ownership clarity**: Code owners map to files, enabling parallel work and targeted reviews.

### Rollout checklist

1. Identify hot files causing conflicts; slice responsibilities into separate ports/adapters.
2. Move integration code (SQL, HTTP) into dedicated adapters owned by each squad.
3. Introduce a composition module that wires the workflow once, so new features don’t reopen core files.

## Navigation

- ⬅️ Previous: [1.2.4 Testing Friction](./Testing_Friction.md)
- ➡️ Next: [1.2.6 Vendor & Ops Lock-In](./Vendor_Tech_Lock_In_And_Ops_Risk.md)
- 🔙 Back to overview: [Index](./index.md)
