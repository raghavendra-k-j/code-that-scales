# 2. Single Responsibility Principle (SRP)

The Single Responsibility Principle states that a class should have only one reason to change, meaning it should have only one job or responsibility.

## Why is it important?

- **Maintainability**: Easier to understand and modify.
- **Testability**: Simpler to test individual responsibilities.
- **Reusability**: Components can be reused independently.

## Example

### Bad Example (Violates SRP)

```python
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary

    def calculate_pay(self):
        # Calculate pay logic
        pass

    def save_to_database(self):
        # Database save logic
        pass

    def generate_report(self):
        # Report generation logic
        pass
```

This class has multiple responsibilities: pay calculation, database operations, and report generation.

### Good Example (Follows SRP)

```python
class Employee:
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary

class PayCalculator:
    def calculate_pay(self, employee):
        # Calculate pay logic
        pass

class EmployeeRepository:
    def save(self, employee):
        # Database save logic
        pass

class ReportGenerator:
    def generate(self, employee):
        # Report generation logic
        pass
```

Each class now has a single responsibility.

## Benefits in Scalable Code

In large applications, adhering to SRP prevents classes from becoming bloated and hard to maintain. It promotes modular design, making it easier to scale and refactor code as requirements evolve.