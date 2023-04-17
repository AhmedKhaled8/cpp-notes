- [Copying and Copy Constructor](#copying-and-copy-constructor)
- [Arrow Operator -\>](#arrow-operator--)
- [Dynamic Arrays `std::vector`](#dynamic-arrays-stdvector)
- [Local static](#local-static)


## Copying and Copy Constructor

```cpp
struct Vector2
{
        float x, y;
         // If we didn't provide a default constructor
         // it will use the instantiation constructor which set the default values for the members 0, 0
}

int main()
{
        Vector2 a {};
        Vector2 b = a; // Copying
        b.x = 3; // doesn't change a.x
}
```

This is because by default, when copying class objects, we copy the members of the source object to the destination object. So `b.x` is a copy of `a.x` and the same for `y`. Changing in `b.x` and `b.y` doesn't change in `a.x` and `a.y`.

However,
```cpp
int main()
{
        Vector2* ap = new Vector2();
        Vector2* bp = ap; // this copy the pointer (address) mot the object itself
        bp->x = 3; // this changes a->x as b and a point to the same object
}
```

But how to make a class copyable ?!

```cpp
#include <iostream>

class MyString
{
private:
    char* m_buffer;
    unsigned int m_size;
public:
    MyString(const char* string)
    {
        m_size = strlen(string);
        m_buffer = new char[m_size + 1];
        memcpy(m_buffer, string, m_size);
        m_buffer[m_size] = '\0';
    }

    ~MyString()
    {
        delete[] m_buffer;
    }

    friend std::ostream& operator<<(std::ostream& stream, const MyString& string);
};

std::ostream& operator<<(std::ostream& stream, const MyString& string)
{
    stream << string.m_buffer;
    return stream;
}


int main()
{
    MyString string("cherno");
    std::cout << string << std::endl;
}
```

It works well 👍.

However, if we tried to copy the `string` object.

```cpp
int main()
{
    MyString string("cherno");
    MyString str2 = string;
    std::cout << string << std::endl;
    std::cout << str2 << std::endl;
    std::cin.get();
}
```

it will print `cherno` correctly twice. But when clicking any key to execute `std::cin.get();` and finish the program, the program crashes. Why?

Because when copying `string` to `str2`, we assign `str2.m_buffer` to be the same as `string.m_buffer`. And when the program exits, the destructor gets called and `delete[] str2.m_bufffer` gets executed and the content of `m_buffer` has been "deleted". Now, the destructor of `string` is to be called and the `delete[] string.m_buffer` will be executed. But wait, the same content and memory has been deleted already when `str2.m_buffer` has been deleted so we can't delete it again and thus causing the crash.


We can better notice this in this example

```cpp
#include <iostream>

class MyString
{
private:
    char* m_buffer;
    unsigned int m_size;
public:
    MyString(const char* string)
    {
        m_size = strlen(string);
        m_buffer = new char[m_size + 1];
        memcpy(m_buffer, string, m_size);
        m_buffer[m_size] = '\0';

    }

    ~MyString()
    {
        delete[] m_buffer;
    }

    char& operator[](unsigned int index)
    {
        return m_buffer[index];
    }

    friend std::ostream& operator<<(std::ostream& stream, const MyString& string);
};

std::ostream& operator<<(std::ostream& stream, const MyString& string)
{
    stream << string.m_buffer;
    return stream;
}

int main()
{
    MyString string("cherno");
    MyString str2 = string;
    str2[3] = 'z';
    std::cout << string << std::endl;
    std::cout << str2 << std::endl;
    std::cin.get();
}
```

This will print `chezno` twice. So now the modification occured in `str2` reflected to `string` because we are dealing with the same pointer assigned to their `m_buffer` members.


**Solution ?**
We need to allocate the same content in another loaction and define a new pointer for `str2`. This generally should be done for any copying instruction. This can be done by writing a ***copy constructor***.

Inside the class, we can write,
```cpp
MyString(const MyString& other)
{
        m_size = other.m_size;
        m_buffer = new char[m_size + 1];
        memcpy(m_buffer, other.m_buffer, m_size + 1);
}
```

Doing that and running the previous code, when we change the string through the `[]` operator, we notice the following:

* The program prints
```
cherno
chezno
```
this tells that the `m_buffer` for the 2 objects are different addresses.

* The program doesn't crash because each destuctor deletes their object's own `m_buffer`.


## Arrow Operator ->

Arrow operator is useful to access the members of an object given a pointer to that object rather dereference that object and the access that member using `.`

Since arrow operator is, well, an operator, we can overload it and use it differently.

In this example,

```cpp
#include <iostream>

class Entity
{
private:
	int x;

public:
	void print() { std::cout << "Hello from Entity\n"; }
};

class ScopedPtr
{
private:
	Entity* m_object;
public:
	ScopedPtr(Entity* entity_p)
		: m_object(entity_p)
	{
	}

	~ScopedPtr()
	{
		delete m_object;
	}
};


int main()
{
	ScopedPtr e = new Entity();
}
```

we want to call `print` from the `Entity` pointer. Since `m_object` is a private member inside `ScopedPtr` thereore can't be accessed with `->` or `.` , we will need to write a method that returns the `m_object`,  and then use the returned pointer to acceess `print()`. Something like this,

```cpp
#include <iostream>

class Entity
{
private:
	int x;

public:
	void print() { std::cout << "Hello from Entity\n"; }
};

class ScopedPtr
{
private:
	Entity* m_object;
public:
	ScopedPtr(Entity* entity_p)
		: m_object(entity_p)
	{
	}

	~ScopedPtr()
	{
		delete m_object;
	}

	Entity* entity()
	{
		return m_object;
	}
};


int main()
{
	ScopedPtr e = new Entity();
	e.entity()->print();
}
```

This prints out as expected

```
Hello from Entity
```

This looks messy, we can do it in a better way. We can overload the arrow operator for `ScopedPtr` to do what we want properly. Inside the `ScopedPtr` class we can write,

```cpp
Entity* operator->()
{
        return m_object;
}
```

And now we can do the following

```cpp
int main()
{
	ScopedPtr e = new Entity();
	e->print();
}
```

And it produces the same output.

***Bonus:*** How can we get the offset of a data member inside a class using arrow operator ?

```cpp
#include <iostream>

class Vector3
{
public:
    float x, y, z;
};

int main()
{
    int offset_x = (int)&((Vector3*)nullptr)->x;
    int offset_y = (int)&((Vector3*)nullptr)->y;
    int offset_z = (int)&((Vector3*)nullptr)->z;
    std::cout << "Offset for x is " << offset_x << std::endl;
    std::cout << "Offset for y is " << offset_y << std::endl;
    std::cout << "Offset for z is " << offset_z << std::endl;
}
```

This results in

```
Offset for x is 0
Offset for y is 4
Offset for z is 8
```


## Dynamic Arrays `std::vector`

`std::vector` is an array that dynamically grows or shrinks if we added or removed data from it instead of the arrays that has a fixed size and can't insert or delete the array's data. Operation on `std::vector` are shown in the following example.

```cpp
#include <iostream>
#include <vector>


struct Vertex
{
	float x, y, z;
};

std::ostream& operator<<(std::ostream& stream, const Vertex& vertex)
{
	stream << vertex.x << ", " << vertex.y << ", " << vertex.z;
	return stream;
}

int main()
{
	std::vector<Vertex> vertices;
	
	// adding (pushing back) elements to the vector
	vertices.push_back({ 1, 2, 3 });
	vertices.push_back({ 4, 5, 6 });
	vertices.push_back({ 7, 8, 9 });
	vertices.push_back({ 10, 11, 12 });

	// get the size of the vector
	int size = vertices.size();

	for (int i = 0; i < size; ++i)
	{
		std::cout << vertices[i] << std::endl;
	}

	std::cout << "============\n";

	// std::vector supports range-based loops
	for (const Vertex& v : vertices)
	{
		std::cout << v << std::endl;
	}

	std::cout << "============\n";

	// removing elements
	vertices.erase(vertices.begin() + 2); // earse takes an iterator not index

	for (const Vertex& v : vertices)
	{
		std::cout << v << std::endl;
	}
}
```
the above example results in

```
1, 2, 3
4, 5, 6
7, 8, 9
10, 11, 12
============
1, 2, 3
4, 5, 6
7, 8, 9
10, 11, 12
============
1, 2, 3
4, 5, 6
10, 11, 12
```

The way vectors allow us to add elements is by reallocating a new array and copy the old elements from the previous array to the new one and add the new element to that array. This happens when the vector has reached its capacity when adding the new element. Because copying can be expensive, we need to optimize the way we use vectors to avoid unnessary copying operation.

We can check when an object is being copied by writing a copy constructor.

```cpp
#include <iostream>
#include <vector>


struct Vertex
{
	float x, y, z;

	// We added this constructor because adding the copy constructor ( or any constructor)
	// prevents the compiler from adding the instantiation  (default) constructor
	Vertex(float x, float y, float z)
		: x(x), y(y), z(z)
	{

	}

	Vertex(const Vertex& vertex)
		: x(vertex.x), y(vertex.y), z(vertex.z)
	{
		std::cout << "copied" << std::endl;
	}
};

std::ostream& operator<<(std::ostream& stream, const Vertex& vertex)
{
	stream << vertex.x << ", " << vertex.y << ", " << vertex.z;
	return stream;
}

int main()
{
	std::vector<Vertex> vertices;

	vertices.push_back(Vertex{ 1, 2, 3 });
	vertices.push_back(Vertex{ 4, 5, 6 });
	vertices.push_back(Vertex{ 7, 8, 9 });
}
```

this results it

```
copied
copied
copied
copied
copied
copied
```

mmm 6 times. Why?

Well, the vector has a capacity which is the actual size of our array. Yes `vertices.size` returns the count of the actual objects inside the vector's buffer but the actual size of the buffer can be get from `vertices.capacity()`. When we add an element when the vector has objects already equal to the capacity, the vector needs to allocate a new buffer with a larger capacity and copy all the contents from the the previous buffer to the new one. We can check the capacity at each step and see how and when the vector resizes.

```cpp
int main()
{
	std::vector<Vertex> vertices;

	std::cout << "When vertices is empty, the capcaity is " << vertices.capacity() << std::endl;
	vertices.push_back(Vertex{ 1, 2, 3 });
	std::cout << "After adding the first vertex is " << vertices.capacity() << std::endl;
	vertices.push_back(Vertex{ 4, 5, 6 });
	std::cout << "After adding the second vertex is " << vertices.capacity() << std::endl;
	vertices.push_back(Vertex{ 7, 8, 9 });
	std::cout << "After adding the third vertex is " << vertices.capacity() << std::endl;
}
```

results in

```
When vertices is empty, the capcaity is 0
copied
After adding the first vertex is 1
copied
copied
After adding the second vertex is 2
copied
copied
copied
After adding the third vertex is 3
```

So, we can see that the initial capacity when we created the vector is 0. So, when adding the first vertex, there is a copy operation occured from the vertex passed to the `push_back` method in the main function to its new location in the buffer created with a capacity of 1.

When a new vertex is added, we need to allocate a new buffer with a larger capacity since we will have 2 vertices while the capacity is just 1. So after creating the new buffer, we copy the first vertex from the old buffer to the new one and then copy the second vertex from the main function to the new buffer. The same occurs when pushing the third vertex since the capacity of the vector is 2 while we try to add a third vertex, we copy the 2 vertices from the old buffer to the new one and copy the third vertex from the main function to the new buffer.

Adding all these copy operation, we get 6 operations. Can we reduce them ? Yes.


If we knew already how many vertices we are going to push to the vector, we can tell the vector from the beginning to have a capacity of that many to avoid reallocating and copying every time we push a vertex. We can do this by writing the following.

```cpp
int main()
{
	std::vector<Vertex> vertices;
	vertices.reserve(3);

	std::cout << "When vertices is empty, the capcaity is " << vertices.capacity() << std::endl;
	vertices.push_back(Vertex{ 1, 2, 3 });
	std::cout << "After adding the first vertex is " << vertices.capacity() << std::endl;
	vertices.push_back(Vertex{ 4, 5, 6 });
	std::cout << "After adding the second vertex is " << vertices.capacity() << std::endl;
	vertices.push_back(Vertex{ 7, 8, 9 });
	std::cout << "After adding the third vertex is " << vertices.capacity() << std::endl;
}
```

results in

```
When vertices is empty, the capcaity is 3
copied
After adding the first vertex is 3
copied
After adding the second vertex is 3
copied
After adding the third vertex is 3
```

So capacity is always 3 and no reallocation occurs. Notice that we defined the capacity not the size because we don't want for the vector to hold any objects. So, if you wrote this,

```cpp
std::vector<Vertex> vertices(10);
```

you are creating a vector with 10 vertices not a vector that can hold 10 vertices but actually contatining 10 instances of the `Vertex` class. Actually, the above code won't work because this approach requires for the class to have a default constructor.

So if we wrote the following,

```cpp
#include <iostream>
#include <vector>


struct Vertex
{
	float x, y, z;

	Vertex()
		: x(10), y(10), z(10)
	{

	}
	
	Vertex(float x, float y, float z)
		: x(x), y(y), z(z)
	{

	}

	Vertex(const Vertex& vertex)
		: x(vertex.x), y(vertex.y), z(vertex.z)
	{
		std::cout << "copied" << std::endl;
	}
};

std::ostream& operator<<(std::ostream& stream, const Vertex& vertex)
{
	stream << vertex.x << ", " << vertex.y << ", " << vertex.z;
	return stream;
}

int main()
{
	std::vector<Vertex> verticies(10);
	std::cout << "Capacity is " << verticies.capacity() << std::endl;
	verticies.push_back(Vertex(1, 2, 3));
	for (const Vertex& v : verticies)
	{
		std::cout << v << std::endl;
	}
}
```

results in

```
Capacity is 10
copied
copied
copied
copied
copied
copied
copied
copied
copied
copied
copied
10, 10, 10
10, 10, 10
10, 10, 10
10, 10, 10
10, 10, 10
10, 10, 10
10, 10, 10
10, 10, 10
10, 10, 10
10, 10, 10
1, 2, 3
```

you will notice that although we pushed only one vertex, there are 11 instances of `Vertex`, because we defined `vertices` as a vector of 10 `Vertex` objects and not as vector that can hold 10 instances of `Vertex`.

So back to the correct approach. We still have 3 copies occur; the ones responsible for copying the `Vertex` object from the main function to the buffer itself. Can we construct the object directly inside the vector ? Yes, with the `emplace_back` and pass the arguments of our constructor.

```cpp
int main()
{
	std::vector<Vertex> vertices;
	vertices.reserve(3);
	vertices.emplace_back(1, 2, 3);
	vertices.emplace_back(4, 5, 6);
	vertices.emplace_back(7, 8, 9);
}
```

this doesn't print anyting thus no copying occurs 🎉


## Local static

You can declare a variable in the local scope as static. This changes its lifetime, how long it will remain before getting deleted (destructed), and its scope.

Let's say we want to count the number of times a function has been called. I can define a global variable that can be accessed from the function and increment it.


```cpp
#include <iostream>

int variable = 0;

void function()
{
	variable++;
	std::cout << "This function has been called " << variable << " times" << std::endl;
}

int main()
{
	function();
	function();
	function();
	function();
	function();
}
```

results in

```
This function has been called 1 times
This function has been called 2 times
This function has been called 3 times
This function has been called 4 times
This function has been called 5 times
```

however, since this variable is global it can be accessed from any scope.

```cpp
#include <iostream>

int variable = 0;

void function()
{
	variable++;
	std::cout << "This function has been called " << variable << " times" << std::endl;
}

void another_function()
{
	variable += 2;
}

int main()
{
	function();
	variable++;
	function();
	function();
	another_function();
	function();
	function();
}
```

results in

```
This function has been called 1 times
This function has been called 3 times
This function has been called 4 times
This function has been called 7 times
This function has been called 8 times
```

So, we want to have a variable local to the function that isn't redefined each time we call the function and keeps its state for the next call. This can be done by defining a local variable inside the function as `static`.

This makes the lifetime of that variable until the program exits and its defined in compile-time. And since its defined in the function scope. It won't be accessed by from other scopes.

```cpp
#include <iostream>

int variable = 0;

void function()
{
	static int variable = 0;
	variable++;
	std::cout << "This function has been called " << variable << " times" << std::endl;
}

void another_function()
{
	variable += 2;
}

int main()
{
	function();
	variable++;
	function();
	function();
	another_function();
	function();
	function();
}
```

results in 

```
This function has been called 1 times
This function has been called 2 times
This function has been called 3 times
This function has been called 4 times
This function has been called 5 times
```

