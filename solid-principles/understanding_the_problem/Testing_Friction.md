# 1.2.4 Testing Friction (SRP, ISP, DIP)

Fast, reliable tests are the feedback loop that keeps changes safe. When production code leans on concrete services and do-everything classes, each test spins up databases, queues, and HTTP calls. The result is brittle suites nobody trusts. Single Responsibility, Interface Segregation, and Dependency Inversion carve seams for lightweight test doubles instead.

## Plain idea

- Split behaviour into narrow roles so tests only exercise the logic they care about.
- Depend on interfaces or protocols that callers can swap for fakes in tests.
- Push slow integrations behind a few contract tests; keep coverage fast with pure units.

## Scenario

- **QA complaint**: “Checkout tests take 15 minutes and still fail randomly. We hit rate limits on the payment sandbox and have to reset test data daily.”
- **Reality**: Every test calls a monolithic service that connects to Postgres, Redis, and the live payment sandbox. Even validation-only tests need the whole stack.

## Before — monolithic service, real infrastructure everywhere

```python
# order_service.py
import psycopg2
import redis
import requests


class OrderService:
	def __init__(self):
		self._db = psycopg2.connect(dsn="postgres://checkout")
		self._cache = redis.Redis.from_url("redis://checkout")
		self._payment_url = "https://sandbox.payments.example/charge"

	def place_order(self, order):
		cur = self._db.cursor()
		cur.execute("INSERT INTO orders (id, total) VALUES (%s, %s)", (order["id"], order["total"]))
		self._db.commit()

		invoice = self._build_invoice_html(order)
		requests.post(self._payment_url, json={"id": order["id"], "amount": order["total"]}, timeout=2)
		self._cache.set(f"invoice:{order['id']}", invoice)
		return invoice

	def _build_invoice_html(self, order):
		return f"<h1>Order {order['id']}</h1><p>Total: {order['total']}</p>"


# test_order_service.py
from uuid import uuid4


def test_invoice_contains_total():
	service = OrderService()
	order = {"id": str(uuid4()), "total": 1299.0}
	invoice = service.place_order(order)
	assert "1299.0" in invoice


def test_payment_failure_rolls_back():
	service = OrderService()
	order = {"id": str(uuid4()), "total": 1299.0}
	# When sandbox rate limits, this test hangs or fails.
	service.place_order(order)
```

### What goes wrong

- **Slow suites**: Every test boots Postgres, Redis, and makes HTTP calls.
- **Flaky results**: Third-party sandbox hiccups cause random failures.
- **Opaque seams**: No way to inject fakes because everything is hard-coded in the constructor.

## After — slice responsibilities, depend on protocols

```python
# ports.py
from dataclasses import dataclass
from typing import Protocol


class OrderRepo(Protocol):
	def save(self, order_id: str, total: float) -> None: ...


class PaymentGateway(Protocol):
	def charge(self, order_id: str, amount: float) -> str: ...


class InvoiceWriter(Protocol):
	def render(self, order: dict) -> str: ...
	def cache(self, order_id: str, invoice: str) -> None: ...


@dataclass
class CheckoutService:
	repo: OrderRepo
	payments: PaymentGateway
	invoices: InvoiceWriter

	def place_order(self, order: dict) -> str:
		self.repo.save(order["id"], order["total"])
		invoice = self.invoices.render(order)
		self.invoices.cache(order["id"], invoice)
		receipt_id = self.payments.charge(order["id"], order["total"])
		return receipt_id


# tests/test_checkout_service.py
class InMemoryRepo:
	def __init__(self):
		self.saved = {}

	def save(self, order_id, total):
		self.saved[order_id] = total


class FakeGateway:
	def __init__(self):
		self.charges = []

	def charge(self, order_id, amount):
		self.charges.append((order_id, amount))
		return "receipt-123"


class StubInvoice:
	def render(self, order):
		return f"Invoice for {order['id']}"

	def cache(self, order_id, invoice):
		pass


def test_checkout_writes_repo_and_charges_gateway():
	repo = InMemoryRepo()
	gateway = FakeGateway()
	invoices = StubInvoice()
	service = CheckoutService(repo, gateway, invoices)
	result = service.place_order({"id": "O-1", "total": 1299.0})

	assert repo.saved["O-1"] == 1299.0
	assert gateway.charges == [("O-1", 1299.0)]
	assert result == "receipt-123"
```

### Why it works better

- **Single Responsibility**: Checkout orchestration only orchestrates; storage, payments, and invoices each have dedicated collaborators.
- **Interface Segregation**: Tests depend on slim `OrderRepo`, `PaymentGateway`, and `InvoiceWriter` protocols, not 300-method service classes.
- **Dependency Inversion**: Production wiring provides real adapters; tests inject fakes without touching the orchestrator.

### Rollout checklist

1. Extract the protocols your service truly needs and adapt existing implementations to them.
2. Thread dependencies via constructors (or a container) instead of instantiating concrete clients inside methods.
3. Add focused unit tests around the orchestration logic using simple fakes, then keep a small suite of contract tests for the real adapters.

## Navigation

- ⬅️ Previous: [1.2.3 Variant Explosion](./Variant_Explosion.md)
- ➡️ Next: [1.2.5 Team Drag & Merge Conflicts](./Team_Drag_And_Merge_Conflicts.md)
- 🔙 Back to overview: [Index](./index.md)
