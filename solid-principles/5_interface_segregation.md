# 5. Interface Segregation Principle (ISP)

The Interface Segregation Principle states that no client should be forced to depend on methods it does not use. Interfaces should be specific to the client's needs.

## Why is it important?

- **Decoupling**: Clients depend only on what they need.
- **Flexibility**: Easier to implement and change interfaces.
- **Maintainability**: Changes to one interface don't affect unrelated clients.

## Example

### Bad Example (Violates ISP)

```python
class Worker:
    def work(self):
        pass

    def eat(self):
        pass

class Robot(Worker):
    def work(self):
        print("Working")

    def eat(self):
        raise NotImplementedError("Robots don't eat")

class Human(Worker):
    def work(self):
        print("Working")

    def eat(self):
        print("Eating")
```

Robot is forced to implement eat method it doesn't need.

### Good Example (Follows ISP)

```python
from abc import ABC, abstractmethod

class Workable(ABC):
    @abstractmethod
    def work(self):
        pass

class Eatable(ABC):
    @abstractmethod
    def eat(self):
        pass

class Robot(Workable):
    def work(self):
        print("Working")

class Human(Workable, Eatable):
    def work(self):
        print("Working")

    def eat(self):
        print("Eating")
```

Interfaces are segregated based on functionality.

## Benefits in Scalable Code

ISP prevents the creation of bloated interfaces that force implementations to provide unnecessary methods. In large systems, this leads to more modular and adaptable code structures.