# 1.2.3 Variant Explosion (OCP)

Adding a new “kind” of payment, promotion, or report should mean adding a plug-in, not sprinkling `if/else` across the codebase. The Open/Closed Principle keeps core flows stable while letting new variants extend behaviour.

## Plain idea

- Extract a narrow interface for the behaviour that varies.
- Register handlers instead of branching on strings everywhere.
- Make flows call the registry so new variants are just new handlers + config.

## Scenario

- **Payments team request**: “Ship PayLater this quarter. GiftCards will follow soon. Checkout, refunds, risk checks, receipts, ledger, and webhooks must keep working.”
- **Today’s reality**: Every flow has `if method == ...` branches that duplicate logic. Adding one method means editing six files and dozens of tests.

## Before — switch statements everywhere

Checkout, refunds, risk, receipts, ledger, and webhooks all hard-code payment logic.

```python
# checkout.py
CARD = CardSDK()
UPI = UpiSDK()
WALLET = WalletSDK()


def process_payment(order, method):
	amount = total_amount(order)
	if method == "CARD":
		res = CARD.charge(amount, order["id"])
		return {"status": "PAID", "payment_id": res["payment_id"]} if res["status"] == "CAPTURED" else {"status": "FAILED"}
	elif method == "UPI":
		res = UPI.pay(amount, order["id"], order.get("vpa"))
		return {"status": "PAID", "payment_id": res["txn"]} if res["state"] == "SUCCESS" else {"status": "FAILED"}
	elif method == "WALLET":
		res = WALLET.debit(order["user_id"], amount)
		return {"status": "PAID" if res["ok"] else "FAILED", "payment_id": res["txn"]}
	else:
		raise ValueError("Unknown method")


# refunds.py
def refund_payment(order, method, payment_id, amount):
	if method == "CARD":
		return CARD.refund(payment_id, amount)["ok"]
	elif method == "UPI":
		return UPI.refund(payment_id, amount)["state"] == "SUCCESS"
	elif method == "WALLET":
		return WALLET.credit(order["user_id"], amount)["ok"]
	else:
		raise ValueError("Unknown method")
```

Every other flow mirrors these branches, so a new method means touching all of them.

### What goes wrong

- **Change amplification**: Adding PayLater means editing checkout, refunds, risk, receipts, ledger, and webhook code paths.
- **Drift**: Forgetting one branch leaves inconsistent behaviour (e.g., refunds failing while checkout works).
- **Test sprawl**: Each flow copies test cases for every method, so adding one variant multiplies the suite.
- **Rollbacks**: Disabling a problematic method requires yet another full sweep of edits.

## After — one payment contract and registry

Define a narrow contract for payment handlers, register concrete implementations, and make every flow resolve handlers from the registry.

```python
from dataclasses import dataclass
from typing import Dict, List, Protocol


@dataclass
class Order:
	id: str
	user_id: str
	items: List[Dict]
	vpa: str = ""


def total_amount(order: Order) -> float:
	return round(sum(i["price"] * i["qty"] for i in order.items), 2)


@dataclass
class PayResult:
	ok: bool
	payment_id: str | None = None
	message: str = ""


class PaymentHandler(Protocol):
	def pay(self, order: Order) -> PayResult: ...
	def refund(self, order: Order, payment_id: str, amount: float) -> bool: ...
	def risk_ok(self, order: Order) -> bool: ...
	def receipt_text(self, payment_id: str) -> str: ...
	def ledger_lines(self, order: Order, result: PayResult) -> List[tuple[str, float]]: ...
	def handle_webhook(self, payload: Dict) -> Dict: ...
```

Concrete handlers adapt provider SDKs and encapsulate method-specific logic.

```python
class CardHandler:
	def __init__(self, sdk: CardSDK):
		self.sdk = sdk

	def pay(self, order: Order) -> PayResult:
		res = self.sdk.charge(total_amount(order), order.id)
		ok = res.get("status") == "CAPTURED"
		return PayResult(ok=ok, payment_id=res.get("payment_id"), message="card_ok" if ok else "card_fail")

	def refund(self, order: Order, payment_id: str, amount: float) -> bool:
		return bool(self.sdk.refund(payment_id, amount).get("ok"))

	def risk_ok(self, order: Order) -> bool:
		return total_amount(order) <= 100000

	def receipt_text(self, payment_id: str) -> str:
		return f"Paid by Card (ID {payment_id})"

	def ledger_lines(self, order: Order, result: PayResult):
		amt = total_amount(order)
		return [("AR", -amt), ("Cash@CardGateway", amt)]

	def handle_webhook(self, payload: Dict) -> Dict:
		if payload.get("event") == "REFUND_SETTLED":
			return {"type": "REFUND_OK", "id": payload["ref_id"]}
		if payload.get("event") == "CHARGEBACK":
			return {"type": "CHARGEBACK", "id": payload["payment_id"]}
		return {"type": "IGNORED"}


class UpiHandler:
	def __init__(self, sdk: UpiSDK):
		self.sdk = sdk

	def pay(self, order: Order) -> PayResult:
		res = self.sdk.pay(total_amount(order), order.id, order.vpa)
		ok = res.get("state") == "SUCCESS"
		return PayResult(ok=ok, payment_id=res.get("txn"), message="upi_ok" if ok else "upi_fail")

	def refund(self, order: Order, payment_id: str, amount: float) -> bool:
		return self.sdk.refund(payment_id, amount).get("state") == "SUCCESS"

	def risk_ok(self, order: Order) -> bool:
		return total_amount(order) <= 50000

	def receipt_text(self, payment_id: str) -> str:
		return f"Paid by UPI (Txn {payment_id})"

	def ledger_lines(self, order: Order, result: PayResult):
		amt = total_amount(order)
		return [("AR", -amt), ("Cash@UPI", amt)]

	def handle_webhook(self, payload: Dict) -> Dict:
		if payload.get("status") == "SUCCESS" and payload.get("kind") == "REFUND":
			return {"type": "REFUND_OK", "id": payload.get("refund_txn")}
		return {"type": "IGNORED"}


class PaymentRegistry:
	def __init__(self):
		self._handlers: Dict[str, PaymentHandler] = {}

	def register(self, method: str, handler: PaymentHandler) -> None:
		self._handlers[method] = handler

	def resolve(self, method: str) -> PaymentHandler:
		if method not in self._handlers:
			raise ValueError(f"Unknown payment method: {method}")
		return self._handlers[method]


registry = PaymentRegistry()
registry.register("CARD", CardHandler(CardSDK()))
registry.register("UPI", UpiHandler(UpiSDK()))


def checkout(order: Order, method: str) -> PayResult:
	handler = registry.resolve(method)
	if not handler.risk_ok(order):
		raise RuntimeError("risk_failed")
	result = handler.pay(order)
	write_ledger(handler.ledger_lines(order, result))
	dispatch_webhooks(handler.handle_webhook({"event": "PAYMENT", "id": result.payment_id}))
	return result


def refund(order: Order, method: str, payment_id: str, amount: float) -> bool:
	handler = registry.resolve(method)
	return handler.refund(order, payment_id, amount)


def configure_handlers():
	registry.register("PAYLATER", PayLaterHandler(PayLaterSDK()))
	registry.register("GIFTCARD", GiftCardHandler(GiftCardSDK()))
	# new handlers only touch this registration spot + tests


def write_ledger(entries: List[tuple[str, float]]) -> None:
	...


def dispatch_webhooks(payload: Dict) -> None:
	...


def test_checkout_isolated_from_variants(card_order: Order):
	result = checkout(card_order, "CARD")
	assert result.ok
	assert registry.resolve("CARD").receipt_text(result.payment_id) == f"Paid by Card (ID {result.payment_id})"


```

### Why it works better

- New variants register themselves; core flows never change.
- Tests for each handler live with the handler, so suites don't multiply by flows.
- Disabling a method is removing a registration or toggling config, not editing six modules.
- You can build dashboards off `registry` to see enabled methods, owner contacts, or rollout state.

### Rollout checklist

1. Extract the protocol and move existing logic into handlers behind it.
2. Introduce a registry (dict or DI container) and update flows to resolve handlers there.
3. Flesh out handler-level tests (pay/refund/risk/webhook) plus a narrow integration test per flow.
4. When adding a new method, add its handler + configuration and you're done.

---

- Previous: [Ripple Effect and Regressions](./Ripple_Effect_And_Regressions.md)
- Next: [Testing Friction](./Testing_Friction.md)
