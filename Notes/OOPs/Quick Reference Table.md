| Concept           | Purpose                         | Example                                 |
| ----------------- | ------------------------------- | --------------------------------------- |
| **Class**         | Blueprint for objects           | `class Car { }`                         |
| **Object**        | Instance of class               | `new Car("Tesla")`                      |
| **Encapsulation** | Hide data, expose via methods   | `private balance + public getBalance()` |
| **Inheritance**   | Parent-child relationship       | `class Dog extends Animal`              |
| **Polymorphism**  | Same method, different behavior | `animal.makeSound()` (varies by type)   |
| **Abstraction**   | Hide implementation             | `abstract class Shape`                  |
| **Interface**     | 100% abstract contract          | `interface Drawable`                    |
| **Composition**   | Strong "has-a"                  | `House HAS-A Room`                      |
| **Aggregation**   | Weak "has-a"                    | `Department HAS-A Employee`             |


##  **Key Takeaways**

1. **Class** = Blueprint, **Object** = Instance
    
2. **Encapsulation** = Keep data private, expose methods
    
3. **Inheritance** = Code reuse, but be careful (IS-A)
    
4. **Polymorphism** = One interface, many implementations
    
5. **Abstraction** = Hide complexity
    
6. **Composition over Inheritance** when possible (HAS-A > IS-A)
    
7. **Interfaces** = Contract for multiple classes
    
8. **SOLID Principles** = Write maintainable code