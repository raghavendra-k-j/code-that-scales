# 3. Open-Closed Principle (OCP)

The Open-Closed Principle states that software entities (classes, modules, functions, etc.) should be open for extension but closed for modification.

## Why is it important?

- **Extensibility**: New functionality can be added without changing existing code.
- **Stability**: Reduces the risk of introducing bugs in existing code.
- **Maintainability**: Easier to add new features.

## Example

### Bad Example (Violates OCP)

```python
class Shape:
    def __init__(self, type):
        self.type = type

class AreaCalculator:
    def calculate_area(self, shapes):
        total = 0
        for shape in shapes:
            if shape.type == "circle":
                # calculate circle area
                pass
            elif shape.type == "square":
                # calculate square area
                pass
        return total
```

Adding a new shape requires modifying the AreaCalculator class.

### Good Example (Follows OCP)

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius

    def area(self):
        return 3.14 * self.radius * self.radius

class Square(Shape):
    def __init__(self, side):
        self.side = side

    def area(self):
        return self.side * self.side

class AreaCalculator:
    def calculate_area(self, shapes):
        return sum(shape.area() for shape in shapes)
```

New shapes can be added by extending the Shape class without modifying AreaCalculator.

## Benefits in Scalable Code

OCP allows systems to grow by adding new components rather than altering existing ones. This is crucial for scalable applications where new features are frequently added without disrupting the core functionality.