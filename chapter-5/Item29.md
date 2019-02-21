# 优先考虑泛型类型

通常，对声明进行参数化并使用JDK提供的泛型类型和方法并不太难。编写自己的泛型类型有点困难，但是值得学习如何编写泛型类型。

考虑条款7中的简单(玩具)堆栈实现：

```java
// Object-based collection - a prime candidate for generics
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    
    public Stack() {
    	elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }
    
    public void push(Object e) {
    	ensureCapacity();
    	elements[size++] = e;
    }
    
    public Object pop() {
        if (size == 0)
        	throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        	return result;
    }
    
    public boolean isEmpty() {
        return size == 0;
    }
    
    private void ensureCapacity() {
        if (elements.length == size)
        	elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

这个类一开始就应该被参数化，但是因为它不是参数化的，所以我们可以在事后对它进行泛化。换句话说，我们可以在不损害原始非参数化版本的客户端的情况下对其进行参数化。按照目前的情况，客户端必须强制转换堆栈中弹出的对象，而这些强制转换可能在运行时失败。泛化一个类的第一步是向其声明中添加一个或多个类型参数。在这种情况下，有一个类型参数，表示堆栈的元素类型，该类型参数的常规名称是`E `(条款68)。

下一步是用恰当的类型参数来替换类型`Object`的所有用法，然后尝试编译得到的程序：

```java
// Initial attempt to generify Stack - won't compile!
public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    
    public Stack() {
    	elements = new E[DEFAULT_INITIAL_CAPACITY];
    }
    
    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }
    public E pop() {
        if (size == 0)
        	throw new EmptyStackException();
        
        E result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }
    ... // no changes in isEmpty or ensureCapacity
}
```

通常你至少会得到一个错误或警告，这个类也不例外。幸运的是，这个类只生成一个错误：

```java
Stack.java:8: generic array creation
        elements = new E[DEFAULT_INITIAL_CAPACITY];
                   ^
```

正如第28项中所解释的，你不能创建不可具体化类型的数组，如`E`。你每次写下一个泛型类型数组的时候，这个问题就会出现。有两种合理的方法可以解决这个问题。第一个解决方案直接绕过了创建泛型数组的禁令：创建Object数组并将其转换为泛型数组类型。现在，编译器将发出一个警告，而不是错误。一般来说，这种用法是合法的，但它不是类型安全：

```java
Stack.java:8: warning: [unchecked] unchecked cast
found: Object[], required: E[]
       elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
             	      ^
```

编译器可能无法证明你的程序是类型安全的，但你可以。你必须说服自己，未经检查的强制转换不会损害程序的类型安全。问题中的数组(`elements`)存储在私有字段中，并且从未返回给客户端或传递给任何其他方法。存储在数组中的元素仅仅传递给了`push`方法，方法中的元素类型也是`E`，所以未检查的强制转换不会造成任何危害。

一旦你证明了未经检查的强制转换是安全的，请将警告限制在尽可能小的范围内(条款27)。在这种情况下，构造函数只包含了未检查的数组创建，因此在整个构造函数中去抑制警告是合适的。通过添加注解来抑制异常，`Stack`可以干净地编译，并且你可以使用它而不用显式强制转换或担心`ClassCastException`。

```java
// The elements array will contain only E instances from push(E).
// This is sufficient to ensure type safety, but the runtime
// type of the array won't be E[]; it will always be Object[]!
@SuppressWarnings("unchecked")
public Stack() {
	elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
}
```

消除`Stack`中泛型数组创建错误的第二种方法是将字段`elements`的类型从`E[]`更改为`Object[]`。如果你这样做，你会得到一个不同的错误：

```java
Stack.java:19: incompatible types
found: Object, required: E
       E result = elements[--size];
                          ^
```

你可以通过将从数组中检索到的元素转换为`E`来将这个错误转换为一个警告，但是你会得到一个警告：

```java
Stack.java:19: warning: [unchecked] unchecked cast
found: Object, required: E
       E result = (E) elements[--size];
                              ^
```
