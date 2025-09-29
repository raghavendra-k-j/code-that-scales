# 4. Liskov Substitution Principle (LSP)

The Liskov Substitution Principle states that objects of a superclass should be replaceable with objects of its subclasses without affecting the correctness of the program.

## Why is it important?

- **Correctness**: Ensures that inheritance hierarchies are properly designed.
- **Polymorphism**: Allows for safe use of polymorphism.
- **Maintainability**: Prevents unexpected behavior when using subclasses.

## Example

### Bad Example (Violates LSP)

```python
class Bird:
    def fly(self):
        print("Flying")

class Penguin(Bird):
    def fly(self):
        raise NotImplementedError("Penguins can't fly")
```

Using Penguin where Bird is expected will cause errors.

### Good Example (Follows LSP)

```python
class Bird:
    pass

class FlyingBird(Bird):
    def fly(self):
        print("Flying")

class Penguin(Bird):
    def swim(self):
        print("Swimming")

# Usage
def make_bird_fly(bird):
    if isinstance(bird, FlyingBird):
        bird.fly()

birds = [FlyingBird(), Penguin()]
for bird in birds:
    make_bird_fly(bird)
```

Penguin doesn't inherit fly method, and the code checks appropriately.

## Benefits in Scalable Code

LSP ensures that as your codebase grows and you add more subclasses, they can be used interchangeably with their base classes without breaking existing functionality. This is essential for building robust, scalable systems.