# 6. Dependency Inversion Principle (DIP)

The Dependency Inversion Principle states that high-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details; details should depend on abstractions.

## Why is it important?

- **Decoupling**: Reduces tight coupling between modules.
- **Testability**: Easier to test components in isolation.
- **Flexibility**: Allows for easy substitution of implementations.

## Example

### Bad Example (Violates DIP)

```python
class LightBulb:
    def turn_on(self):
        print("LightBulb turned on")

    def turn_off(self):
        print("LightBulb turned off")

class Switch:
    def __init__(self):
        self.bulb = LightBulb()

    def operate(self):
        self.bulb.turn_on()
```

Switch depends directly on LightBulb.

### Good Example (Follows DIP)

```python
from abc import ABC, abstractmethod

class Switchable(ABC):
    @abstractmethod
    def turn_on(self):
        pass

    @abstractmethod
    def turn_off(self):
        pass

class LightBulb(Switchable):
    def turn_on(self):
        print("LightBulb turned on")

    def turn_off(self):
        print("LightBulb turned off")

class Fan(Switchable):
    def turn_on(self):
        print("Fan turned on")

    def turn_off(self):
        print("Fan turned off")

class Switch:
    def __init__(self, device: Switchable):
        self.device = device

    def operate(self):
        self.device.turn_on()
```

Switch depends on the abstraction Switchable, not on concrete implementations.

## Benefits in Scalable Code

DIP enables building systems that are more modular and easier to extend. As applications scale, this principle helps in managing dependencies and allows for better separation of concerns across different layers of the application.