# Polymorphic Iteration Unleashed: Storing Objects of Different Type Contiguously #

Imagine you have several classes with different sizeof, but all have the same interface
```cpp
struct A
{
  int value1;
  void DoAction() { /* */ }
};


struct B
{
  int value1;
  float value2;
  A value3;
  void DoAction() { /* */ }
}
```
What if you want to make a data structure to store all that types and iterate them all? First and simpliest thing we all come up with is to create base class with interface (virtual methods), derive from it, make vector and iterate

```cpp

struct Base
{
  virtual void DoAction() const = 0;
};

struct A : Base
{
  int value1;

  A(int value) : value1(value) {}
  void DoAction() const override { std::cout << "A " << value1 << '\n'; }
};


struct B : Base
{
  int value1;
  float value2;
  A value3;

  B(int value) : value1(value), value3(value) {}
  void DoAction() const override { std::cout << "B " << value1 << '\n'; }
};

// Somewhere...
std::vector<Base*> vec{new A{5}, new B{4}, new B{3}, new A{2}};
for(const Base* BasePtr: vec)
{
  BasePtr->DoAction();
}
```
This method is easy, familiar to us, but have some costs.
When we iterate base class pointer, we cannot utilize cache in most cases, spending a lot of time waiting to fetch RAM at different places.

What we might want, is to have a data structure that could contain objects of different types laid out contiguously in memory. Here we would use Type Erasure technique to place object contiguously but save their ability to be polymorphic. There are dozens of article about Type Erasure, and I will just explain the basic idea and provide example implementation.
Basically Type Erasure happens when we have a common classic virtual interface, templated interface implementation with object as member, which calls object method in it's virtual method. When we use interface, we do not know actual object type, but the difference is that we don't need to derive our from that interface, it just must have all methods from interface.

```cpp
class VirtualDuckInterface {
public:
    virtual ~VirtualDuckInterface() = default;
    virtual void VirtualDuck() const = 0;
};

template <typename T>
class DuckInterfaceImpl final : public VirtualDuckInterface {
public:
    explicit DuckInterfaceImpl(const T& obj) : object(obj) {}

    void VirtualDuck() const override {
        object.Duck();
    }

private:
    T object;
};

class MyClass {
public:
    int value1;
    
    void Duck() const {
        std::cout << "MyClass value1 = " << value1 << std::endl;
    }
};

// Somewhere...
MyClass obj{5};
DuckInterfaceImpl<MyClass> objWrapper(obj);
VirtualDuckInterface* ptr = &objWrapper;
ptr->VirtualDuck(); 
```

Now, using that technique you can do many things:
* Create simple Allocator class that would write each object to contiguous buffer, keeping them in ```std::vector<DoActionInterface*>```, so you can iterate objects one after another close in memory.
* Create Allocator more smarter than above, with optimization based on data processed.
* Create Pool Manager that would keep objects of different type in different pools (vectors) (ECS similar technique)
* And others...

Full example, showing how to iterate objects of different types packed in single vector:

```cpp
#include <iostream>
#include <vector>

// Virtual interface
class VirtualDuckInterface {
public:
    virtual ~VirtualDuckInterface() = default;
    virtual void VirtualDuck() const = 0;
};

// Simple memory allocator for objects of different sizes
class ObjectAllocator {
public:
    explicit ObjectAllocator(size_t size) : buffer(size), offset(0) {}

    template <typename T>
    T* Allocate(const T& obj) {
        if (offset + sizeof(T) > buffer.size()) {
            throw std::bad_alloc(); // You may want to handle this more gracefully
        }

        T* ptr = new (&buffer[offset]) T(obj);
        
        offset += sizeof(T);
        objects.push_back(ptr);
        return ptr;
    }

    const std::vector<const VirtualDuckInterface*>& GetObjects() const {
        return objects;
    }

private:
    std::vector<char> buffer;
    size_t offset;
    std::vector<const VirtualDuckInterface*> objects;
};

// Template implementation of the interface for each type
template <typename T>
class DuckInterfaceImpl final : public VirtualDuckInterface {
public:
    explicit DuckInterfaceImpl(const T& obj) : object(obj) {}

    void VirtualDuck() const override {
        object.Duck();
    }

private:
    T object;
};

// Container for managing objects of different types
class ObjectManager {
public:
    ObjectManager(size_t Size = 500) : allocator(Size) {}

    template <typename T>
    void AddObject(const T& obj) {
        allocator.Allocate(DuckInterfaceImpl<T>(obj));
    }

    void IterateObjects() const {
        for (const auto& obj : allocator.GetObjects()) {
            obj->VirtualDuck();
        }
    }

private:
    ObjectAllocator allocator;
};

// Example usage
class MyClass1 {
public:
    int value1;
    
    void Duck() const {
        std::cout << "MyClass1 value1 = " << value1 << std::endl;
    }
};

class MyClass2 {
public:
    int value1;
    int value2;
    int value3[5];
    void Duck() const {
        std::cout << "MyClass2 value1 = " << value1 << ", value2 = " << value2 << ", value3 = ";
        std::cout << value3[0];
        for(int i = 1; i < 5; ++ i)
        {
            std::cout << ", " << value3[i];
        }
        std::cout << std::endl;
    }
};

class MyClass3 {
public:
    bool value1;
    
    void Duck() const {
        std::cout << "MyClass3 value1 = " << value1 << std::endl;
    }
};

int main() {

    MyClass1 obj{5};
    DuckInterfaceImpl<MyClass1> objWrapper(obj);
    VirtualDuckInterface* ptr = &objWrapper;
    std::cout << "---\n";
    ptr->VirtualDuck();
    std::cout << "---\n";
    
    ObjectManager objectManager;

    objectManager.AddObject(MyClass1{5});
    objectManager.AddObject(MyClass2{6, 6, {6, 6}});
    objectManager.AddObject(MyClass1{7});
    objectManager.AddObject(MyClass3{1});
    objectManager.AddObject(MyClass2{9, 9, {9}});
    objectManager.AddObject(MyClass3{0});

    objectManager.IterateObjects();

    return 0;
}
```

Of course, many things could be tuned and optimized, you can always point that things in comments!

Now we will benchmark our approaches.
* First approach - create ```std::vector<Base*>``` and fill it with Derived type objects with overriden virtual methods. In the measured loop will call simple virtual function that return int member.
* Second approach - in first approach we create new objects and they may lay contiguous, depending on allocator. So in this approach we will shuffle ```Base*``` pointers to eliminate any chances of this.
* Third approach - we will fill our ObjectManager with the same creation data as in 1 and 2 approach, but using non-derived types (their counterparts have the same layout). Using new method, we will use the same but not function that returns int member that is called via virtual function in Type Erasure implementation.

[Benchmark link](https://quick-bench.com/q/XzHAqhaslSCkeodLqVRzrhbvcXw)

IMAGE HERE

As we can see, 1 and 3 approaches looks like the same, but the second, the shuffled one, is much worse. In fact in real software second situatuion is more like to happen because different object are created in different times. Third approach is guaranteed to be cache-friendly and could be further optimized.

