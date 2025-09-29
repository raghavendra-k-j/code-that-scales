# 1.2.2 Ripple Effects & Regressions (LSP)

Swapping one implementation for another should not break callers. When subclasses or providers change behaviour, the rest of the app pays the price in unexpected crashes, broken flows, and frantic hotfixes. The Liskov Substitution Principle keeps replacements compatible.

## Plain idea

- Document the contract your app expects (inputs, outputs, units, error shapes).
- Ensure every implementation honours that contract—no stricter inputs, no weaker outputs.
- Test replacements against the same contract tests before shipping.

## Scenario

- **Logistics request**: “Switch from Courier A to Courier B—better price. Checkout ETA, tracking, returns, and notifications must keep working.”
- **Reality today**: App code talks directly to each courier API. Inputs, units, and errors differ, so flipping the toggle leaves a trail of breakage.

## Before — hidden contracts and surprises

Each flow hard-codes courier logic, assumes certain states, and handles errors inconsistently.

```python
# shipping_service.py
class CourierAClient:
	def create_shipment(self, address: dict, weight_kg: float) -> dict:
		if not address.get("pincode"):
			raise ValueError("bad address")
		if weight_kg > 30:
			raise ValueError("too heavy")
		return {"tracking_id": "A123", "eta_hours": 36.0, "status": "CREATED"}

	def track(self, tracking_id: str) -> dict:
		return {"status": "IN_TRANSIT", "last_seen": "Hub-7"}


class CourierBClient:
	class BApiError(Exception): ...

	def create(self, dest: dict, kg: float):
		if not dest.get("pincode") or not dest.get("phone"):
			return None
		if kg > 10:
			raise CourierBClient.BApiError("over 10kg not allowed")
		return {"id": "B999", "eta_minutes": 1800, "state": "init"}

	def status(self, tid: str):
		return {"state": "moving", "seen_at": 1699999999}


USE_COURIER = "A"


def place_shipment(address: dict, weight_kg: float):
	if USE_COURIER == "A":
		rec = CourierAClient().create_shipment(address, weight_kg)
		return {"tracking_id": rec["tracking_id"], "eta_hours": rec["eta_hours"]}
	else:
		rec = CourierBClient().create(address, weight_kg)
		tracking_id = rec["id"]
		eta_hours = rec["eta_minutes"] / 60.0
		return {"tracking_id": tracking_id, "eta_hours": eta_hours}

## After — explicit contract with adapters

Define the courier contract you need, then wrap each provider with an adapter that honours it. Callers depend only on the contract.

```python
# courier_contract.py
from dataclasses import dataclass
from typing import Protocol, Literal, Dict


class CourierError(Exception):
	def __init__(self, code: Literal["INVALID_ADDRESS", "OVERWEIGHT", "TEMPORARY"], message: str = ""):
		super().__init__(message)
		self.code = code


@dataclass
class ShipmentLabel:
	tracking_id: str
	eta_hours: float
	status: Literal["CREATED", "IN_TRANSIT", "DELIVERED", "RETURNED"]


@dataclass
class TrackingInfo:
	status: Literal["IN_TRANSIT", "DELIVERED", "RETURNED"]
	last_seen: str


class Courier(Protocol):
	def create_shipment(self, address: Dict, weight_kg: float) -> ShipmentLabel: ...
	def track(self, tracking_id: str) -> TrackingInfo: ...
```

Adapters normalise inputs, outputs, and errors.

```python
# courier_b_adapter.py
class CourierBAdapter(Courier):
	def __init__(self, client):
		self.client = client

	def _map_state(self, state: str) -> str:
		return {"init": "CREATED", "moving": "IN_TRANSIT", "done": "DELIVERED", "rto": "RETURNED"}[state]

	def create_shipment(self, address, weight_kg) -> ShipmentLabel:
		addr = dict(address)
		addr.setdefault("phone", "NA")

		if weight_kg > 30:
			raise CourierError("OVERWEIGHT", "Over app limit 30kg")

		remaining = weight_kg
		tracking_id = None
		eta_hours = 0.0
		while remaining > 0:
			kg = min(remaining, 10.0)
			rec = self.client.create(addr, kg)
			if rec is None:
				raise CourierError("INVALID_ADDRESS", "Missing/invalid address for Courier B")
			tracking_id = tracking_id or rec["id"]
			eta_hours = max(eta_hours, rec["eta_minutes"] / 60.0)
			remaining -= kg
		return ShipmentLabel(tracking_id, eta_hours, self._map_state(rec["state"]))

	def track(self, tracking_id) -> TrackingInfo:
		status = self.client.status(tracking_id)
		return TrackingInfo(status=self._map_state(status["state"]), last_seen=str(status.get("seen_at", "")))
```

The rest of the app calls the contract via a small factory.

```python
def make_courier(name: str) -> Courier:
	if name == "A":
		return CourierAAdapter(CourierAClient())
	if name == "B":
		return CourierBAdapter(CourierBClient())
	raise ValueError("Unknown courier")


def place_shipment(address: dict, weight_kg: float, courier_name: str):
	courier = make_courier(courier_name)
	label = courier.create_shipment(address, weight_kg)
	return {"tracking_id": label.tracking_id, "eta_hours": label.eta_hours, "status": label.status}
```

### Why it works

- **LSP compliance**: Adapters never tighten preconditions or weaken postconditions. Every courier honours the same promises.
- **Normalised errors**: Callers see predictable `CourierError` codes and handle fallbacks uniformly.
- **Central wiring**: Switching providers is a config change, not a cross-cutting refactor.

## Contract tests

```python
def contract_tests(courier):
	label = courier.create_shipment({"pincode": "560001"}, 29.5)
	assert label.tracking_id and label.eta_hours > 0
	assert label.status in {"CREATED", "IN_TRANSIT", "DELIVERED", "RETURNED"}

	try:
		courier.create_shipment({"pincode": ""}, 1)
		assert False
	except CourierError as err:
		assert err.code in {"INVALID_ADDRESS", "TEMPORARY"}

	try:
		courier.create_shipment({"pincode": "560001"}, 31)
		assert False
	except CourierError as err:
		assert err.code == "OVERWEIGHT"


contract_tests(CourierAAdapter(CourierAClient()))
contract_tests(CourierBAdapter(CourierBClient()))
```

## Navigation

- ⬅️ Previous: [1.2.1 Change Amplification](./Change_Amplification.md)
- ➡️ Next: [1.2.3 Variant Explosion](./Variant_Explosion.md)
- 🔙 Back to overview: [Index](./index.md)
