# 1.2.6 Vendor & Ops Lock-In (OCP, DIP)

If high-level code is welded to a specific vendor SDK, experimenting with alternatives or reacting to outages becomes a weekend project. Open/Closed and Dependency Inversion principles let you hot-swap providers, run dual writes, or fail over with confidence.

## Plain idea

- Depend on your own abstractions (ports) instead of vendor-specific clients.
- Keep wiring in one place so adding a new provider is registering a new adapter, not editing every call site.
- Capture fallbacks, retries, and observability in the adapter instead of scattering it across the app.

## Scenario

- **Leadership ask**: “We need WhatsApp notifications next quarter. Also, experiment with a local SMS gateway for costs.”
- **Today**: Notification code calls the Twilio SDK directly in five modules. There’s no easy way to dual-write, log failures, or fall back when Twilio hiccups.

## Before — direct SDK usage everywhere

```python
# notification.py
from twilio.rest import Client as TwilioClient


twilio = TwilioClient(account_sid="...", auth_token="...")


def send_otp(phone: str, otp: str) -> None:
	message = f"Use {otp} to login"
	twilio.messages.create(to=phone, from_="+12025550000", body=message)


def send_order_update(phone: str, order_id: str, status: str) -> None:
	body = f"Order {order_id} is now {status}"
	twilio.messages.create(to=phone, from_="+12025550000", body=body)


def send_marketing_blast(numbers: list[str], text: str) -> None:
	for phone in numbers:
		twilio.messages.create(to=phone, from_="+12025550000", body=text)
```

### What goes wrong

- **Outage pain**: When Twilio has issues, you have zero fallback and must patch every caller.
- **Experiment tax**: Trying a local provider means rewriting five modules and dozens of tests.
- **Ops blind spots**: No central place to add logging, retries, or rate limiting.

## After — ports plus adapters with feature flag wiring

```python
# messaging_ports.py
from dataclasses import dataclass
from typing import Protocol, Iterable


class MessageGateway(Protocol):
	def send(self, to: str, body: str) -> None: ...


class BulkGateway(Protocol):
	def send_many(self, messages: Iterable[tuple[str, str]]) -> None: ...


@dataclass
class Notifier:
	primary: MessageGateway
	bulk: BulkGateway

	def send_otp(self, phone: str, otp: str) -> None:
		self.primary.send(phone, f"Use {otp} to login")

	def send_order_update(self, phone: str, order_id: str, status: str) -> None:
		self.primary.send(phone, f"Order {order_id} is now {status}")

	def send_marketing_blast(self, numbers: list[str], text: str) -> None:
		self.bulk.send_many((num, text) for num in numbers)
```

Adapters encapsulate vendor specifics and resilience policy.

```python
# adapters.py
class TwilioAdapter(MessageGateway, BulkGateway):
	def __init__(self, client, sender_id: str, logger):
		self.client = client
		self.sender_id = sender_id
		self.logger = logger

	def send(self, to: str, body: str) -> None:
		try:
			self.client.messages.create(to=to, from_=self.sender_id, body=body)
		except Exception as err:
			self.logger.error("twilio_send_failed", to=to, error=str(err))
			raise

	def send_many(self, messages):
		for to, body in messages:
			self.send(to, body)


class LocalSmsAdapter(MessageGateway, BulkGateway):
	def __init__(self, http_client, api_key: str):
		self.http = http_client
		self.api_key = api_key

	def send(self, to, body):
		self.http.post("https://sms.local/send", json={"to": to, "body": body}, headers={"X-API-Key": self.api_key})

	def send_many(self, messages):
		payload = [{"to": to, "body": body} for to, body in messages]
		self.http.post("https://sms.local/bulk", json=payload, headers={"X-API-Key": self.api_key})
```

One wiring module decides which adapter to expose.

```python
# wiring.py
from feature_flags import flag_value


def make_notifier(logger, http_client, twilio_client) -> Notifier:
	registry = {
		"twilio": lambda: TwilioAdapter(twilio_client, sender_id="+12025550000", logger=logger),
		"local": lambda: LocalSmsAdapter(http_client, api_key="secret"),
	}

	preferred = flag_value("sms_provider", default="twilio")
	adapter = registry[preferred]()
	return Notifier(primary=adapter, bulk=adapter)
```

### Why it works better

- **Closed for modification**: Adding WhatsApp means writing `WhatsAppAdapter` and registering it—callers stay untouched.
- **Inverted dependencies**: High-level flows depend on `Notifier`, not Twilio. Production chooses the adapter at the edges.
- **Operational flexibility**: Central wiring enables circuit breakers, dual writes, or logging without sweeping edits.

### Rollout checklist

1. List all places calling the vendor SDK; funnel them through a single port plus adapter.
2. Add feature flags or configuration to pick adapters so experiments and failovers are config changes.
3. Write contract tests against the port to ensure every adapter honours the same success and failure semantics.

## Navigation

- Previous: [1.2.5 Team Drag & Merge Conflicts](./Team_Drag_And_Merge_Conflicts.md)
- Back to overview: [README](./README.md)
