- [Copying and Copy Constructor](#copying-and-copy-constructor)
- [Arrow Operator -\>](#arrow-operator--)
- [Dynamic Arrays `std::vector`](#dynamic-arrays-stdvector)
- [Local Static](#local-static)
- [Multiple-Return Values](#multiple-return-values)
		- [1. Pass Arguments by References](#1-pass-arguments-by-references)
		- [2. Vector](#2-vector)
		- [3. Tuples and Pairs](#3-tuples-and-pairs)
		- [4. Using Structs](#4-using-structs)
- [Templates](#templates)
- [Stack and Heap](#stack-and-heap)
- [Macros](#macros)
- [`auto`](#auto)
- [Function Pointers and Lambdas](#function-pointers-and-lambdas)
- [Threads](#threads)


## Copying and Copy Constructor

```cpp
struct Vector2
{
        float x, y;
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


## Local Static

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

## Multiple-Return Values

C++ functions can't return multiple values of different types. There are solutions to achieve such a behavior.

#### 1. Pass Arguments by References

```cpp
#include <iostream>
#include <string>

void function(const std::string& src, std::string& src_png, std::string& src_jepg)
{
	src_png = src + ".png";
	src_jepg = src + ".jpeg";
}

int main()
{
	std::string file_name = "image";
	std::string png_path, jpeg_path;
	function(file_name, png_path, jpeg_path);
	std::cout << png_path << std::endl << jpeg_path << std::endl;
}
```

#### 2. Vector

This will of course require that the contents of the vector of the same type, or pointer to a common base class.

```cpp
#include <iostream>
#include <string>
#include <vector>

std::vector<std::string> function(const std::string& src)
{
	std::vector<std::string> v;
	v.reserve(2);
	std::string src_png = src + ".png";
	std::string src_jpeg = src + ".jpeg";
	v.push_back(src_png);
	v.push_back(src_jpeg);
	return v;
}

int main()
{
	std::string file_name = "image";
	std::vector<std::string> v = function(file_name);
	std::cout << v[0] << std::endl << v[1] << std::endl;
}
```

#### 3. Tuples and Pairs

Tuples can be used to store any amount of variables of different types. Pairs can be used to store only 2 variables.

```cpp
#include <iostream>
#include <string>
#include <tuple>
#include <utility>

std::tuple<std::string, std::string> function(const std::string& src)
{
	std::string src_png = src + ".png";
	std::string src_jpeg = src + ".jpeg";
	std::tuple<std::string, std::string> t = std::make_tuple(src_png, src_jpeg);
	return t;
}

int main()
{
	std::string file_name = "image";
	std::tuple<std::string, std::string> t = function(file_name);
	std::string png_path = std::get<0>(t);
	std::string jpeg_path = std::get<1>(t);
	std::cout << png_path << std::endl << jpeg_path << std::endl;
}
```

same for tuples but different functions and accessing.

```cpp
#include <iostream>
#include <string>
#include <tuple>
#include <utility>

std::pair<std::string, std::string> function(const std::string& src)
{
	std::string src_png = src + ".png";
	std::string src_jpeg = src + ".jpeg";
	std::pair<std::string, std::string> p = std::make_pair(src_png, src_jpeg);
	return p;
}

int main()
{
	std::string file_name = "image";
	std::pair<std::string, std::string> p = function(file_name);
	const std::string& png_path = p.first;
	const std::string& jpeg_path = p.second;
	std::cout << png_path << std::endl << jpeg_path << std::endl;
}
```

#### 4. Using Structs

This is the best way, for me, because we can avoid the ambiguity of trying to guess in which order the variables were stored and easily, and minimally, update our code if we want to add more variables. Also, it makes our code cleaner from the name of the struct members. 

```cpp
#include <iostream>
#include <string>

struct Paths
{
	std::string png_path;
	std::string jpeg_path;
};

Paths function(const std::string& src)
{
	Paths p;
	p.png_path = src + ".png";
	p.jpeg_path = src + ".jpeg";
	return p;
}

int main()
{
	std::string file_name = "image";
	Paths p = function(file_name);
	std::cout << p.png_path << std::endl << p.jpeg_path << std::endl;
}
```

## Templates

Templates can be used to create a "blueprint" or "template" for your code in which it can support different types. The complier generates a version of that code for each type you used in your code.

For example, if I want to write a function that simply prints a given value in a new line. I can do this,

```cpp
#include <iostream>
#include <string>

void print(int value)
{
	std::cout << value << std::endl;
}

void print(std::string value)
{
	std::cout << value << std::endl;
}

void print(float value)
{
	std::cout << value << std::endl;
}

int main()
{
	print(1);
	print("Hello");
	print(5.5f); // without the print(float _), it will be called by print(int _) and cast 5.5f to int and remove the decimal
}
```

of course, this is difficult to maintain and hard to expand for unsupported types. A solution for this is templates.

```cpp
#include <iostream>
#include <string>

template <typename T>
void print(T value)
{
	std::cout << value << std::endl;
}

int main()
{
	print(1); // creates a print(int value)
	print("Hello"); // creates a print(std::string value)
	print(5.5f); // creates a print(float value)
}
```

the templates are evaluated at compile-time. So the generated code will include `print (int value)`, `print (std::string value)`, and `print (float value)`.


`template` is a keyword to define that the following block is a blueprint of which a version will be created in compile-time. `<typename T>` is a template parameter, `typename` tells that `T`, which is the common name, of this template block which will be replaced when a different version of the template is used or called.

***NOTE:*** writing `template <class T>` is exactly the same as `template <typename T>`.


When `print(1)` is called, `T` is deduced from the type of the first argument which will be `int`. If the deduction of the template argument isn't trivial like this, it's better to explicit define what `T` is.

```cpp
#include <iostream>
#include <string>

template <typename T>
void print(T value)
{
	std::cout << value << std::endl;
}

int main()
{
	print<int>(1); // creates a print(int value)
	print<std::string>("Hello"); // creates a print(std::string value)
	print<float>(5.5f); // creates a print(float value)
}
```

When explicitly define the typename and contradict this explicit definition,

```cpp
print<float>("Hello"); // creates a print(float value)
```

we get this error

```
no instance of function template "print" matches the argument list
```
this is because it's expected when using `print<float>` to
pass `val` as `float` not `std::string`.

---
We know that the template function is only to create a blueprint of which version are created in compile time depending on what types you used and called. If you didn't use that template function at all, the template function absolutely doesn't exist at all in your code. A way to prove that by creating an error in the template function and not call that function at all.

```cpp
#include <iostream>

template <typename T>
void print(T value)
{
	const int x = 3;
	x = 5; // this should raise an error in runitme
}

int main()
{
}
```

this doesn't raise an error as if the code that is compiled was simply

```cpp
#include <iostream>

int main()
{
}
```

while this code

```cpp
#include <iostream>

template <typename T>
void print(T value)
{
	const int x = 3;
	x = 5; // this should raise an error in runitme
}

int main()
{
	print(3);
}
```
raises an error

```
'x': you cannot assign to a variable that is const
```
because `print (int value)` actually exists.

***NOTE:*** The above discussion about error in templates is true for some compilers including vsc, the one used for the notes. Other compilers like clang will raise an error anyways.


---

We can define template classes as well as template functions. Also, we can define more template parameters other than a template type.

```cpp
#include <iostream>
#include <string>

template<typename T, int SIZE>
class Array
{
private:
	T m_arr[SIZE];
public:
	int get_size()
	{
		return SIZE;
	}

	void print()
	{
		for (int i = 0; i < SIZE; ++i)
		{
			std::cout << T{} << ", ";
		}
		std::cout << std::endl;
	}
};

int main()
{
	Array<int, 3> int_arr;
	std::cout << int_arr.get_size() << std::endl;
	int_arr.print();
	Array<std::string, 5> str_arr;
	std::cout << str_arr.get_size() << std::endl;
	str_arr.print();
}
```

Here, `typename T` tells that T is a template type that can be used anywhere in the code and can be replaced by the type provided when using this class. And `int SIZE` is an integer that is explicitly passed and any `SIZE` is replaced with that value when using this class.

since the templates are evaluated at compile time, the `SIZE` is defined before statically allocating. Otherwise, it wouldn't be compiled in the first place.


the above code will be compiled as something like this,

```cpp
#include <iostream>
#include <string>

class Array<int, 3>
{
private:
	int m_arr[3];
public:
	int get_size()
	{
		return 3;
	}

	void print()
	{
		for (int i = 0; i < 3; ++i)
		{
			std::cout << int{} << ", ";
		}
		std::cout << std::endl;
	}
};

class Array<std::string, 5>
{
private:
	std::string m_arr[5];
public:
	int get_size()
	{
		return 5;
	}

	void print()
	{
		for (int i = 0; i < 5; ++i)
		{
			std::cout << std::string{} << ", ";
		}
		std::cout << std::endl;
	}
};

int main()
{
	Array<int, 3> int_arr;
	std::cout << int_arr.get_size() << std::endl;
	int_arr.print();
	Array<std::string, 5> str_arr;
	std::cout << str_arr.get_size() << std::endl;
	str_arr.print();
}
```

## Stack and Heap

For the program to start, the exec0utable needs to be loaded to the memory as well as allocating physical RAM. The **stack** and **heap** are 2 areas that we have in our RAM. Stack has a predefined size, usually around 2 MB, and the heap an area that has a initial size but can grow and change as our application goes on. The actual loaction of these areas is always the same in our RAM.

Memory is used to store the data. Stack and heap are 2 areas in our memory in which we are allow to store. We can ask C++ to give us some memory from either stack or heap. But the way C++ allocates the requested memory, i.e. memory allocation, is different.

```cpp
#include <iostream>

struct Vec3
{
	int x, y, z;

	Vec3() : x(10), y(11), z(12) {}
};

int main()
{
	int value = 5; // var is integer alloacted on stack
	int arr[5] = { 1, 2, 3, 4, 5 }; // arr is array of 5 integers allocated on stack
	Vec3 v; // v is Vec3 alloacted on stack
}
```

when looking at the memory of each of these stack-allocated objects. We get this.

<img src="./assets/stack-heap/stack-allocation-memory.png" height="100">

The objects are allocated near each other. Actually, when allocating on stack, the stack pointer moves *backwards* with an amount of the size of the object to be allocated. `value` gets allocated (third rectangle), then the stack pointer moves 20 bytes (5 integers * 4 bytes) and allocate `arr` (second rectangle), then `v` gets allocated `first rectangle` after the stack pointer moves by `12 bytes` which is the size of `Vector3` struct. You can see that there are bytes filled with `cc` between our objects, these are safeguards because we are on debug mode, these bytes are helpful to avoid accessing wrong location and helps us debugging our bugs.

---

For heap, the allocation is different.

```cpp
#include <iostream>

struct Vec3
{
	int x, y, z;

	Vec3() : x(10), y(11), z(12) {}
};

int main()
{
	int* h_value = new int; // h_value points to an int allocated on heap
	*h_value = 5;
	int* h_arr = new int[5]{ 1, 2, 3, 4, 5 }; // h_arr points to an int allocated on heap
	Vec3* h_v = new Vec3(); // h_v points to an Vec3 allocated on heap

	delete h_value;
	delete[] h_arr;
	delete h_v;
}
```

If we look at the locations of each object created on the heap. Each will be allocated in separate place not close to each other unlike the stack.

`new` calls `malloc` which in turns according to the OS allocate on heap the size required. When starting the application, we get a certain amount of physical RAM and the program will maintain something called **free list** which keeps track of the blocks of memory that are free to use when you want to allocate something on it. It will check on its free list which block have a size enough for your requirement and return a pointer to that block. If it can't find a block with enough space, it will ask the OS to give it some additional blocks. While this is a heavy task, allocating on stack only requires one CPU instruction.

---

So, you should allocate on stack when needed. You should only allocate on heap if you can't allocate on stack due to its lifetime. Objects allocated on heap don't get destructed at the end of the scopes of their definition, they get destroyed when you explicitly do so by calling `delete`. Or if you want a large amount of data, e.g. 50 MB, that can't be allocated on the small stack.


## Macros

Macros are preprocessors that are evaluated in preprocessing before compilation starts. Macros are text-replacers. They follow this syntax,

```
#define text-to-replace text-to-be-replaced-with
```

For example,

```cpp
#include <iostream>

#define WAIT std::cin.get()

int main()
{
	WAIT;
}
```

this code after preprocessing will be

```cpp
#include <iostream>

int main()
{
	std::cin.get();
}
```

which will be compiled.

This is possible for any text

```cpp
#include <iostream>

#define OPEN_SCOPE {
#define WAIT std::cin.get()
#define CLOSE_SCOPE }

int main()
OPEN_SCOPE
	WAIT;
CLOSE_SCOPE
```
which will be preprocessed to

```cpp
#include <iostream>

int main()
{
	std::cin.get();
}
```

---

Macros can recieve arguments as well

```cpp
#include <iostream>

#define MULTIPLY(x, y) x * y

int main()
{
	std::cout << MULTIPLY(3, 4) << std::endl;
	std::cout << MULTIPLY(2 + 1, 5 - 1) << std::endl;
}
```

this prints out

```
12
6
```

but wait, the first statement is correct but the second one should get the same result because 2 + 1 is 3 and 5 - 1 is 4. So why did we get 6 instead of 12 ?

Becuase macros only replaces text, so the actual code will be

```cpp
#include <iostream>

#define MULTIPLY(x, y) x * y

int main()
{
	std::cout << 3 * 4 << std::endl;
	std::cout << 2 + 1 * 5 - 1 << std::endl;
}
```

So yes 3 * 4 is 12 but due to C++ operators precednce, 1 * 5 is evaulated first to 5 then 2 + 5 - 1 is 6. A solution to get the expected behavior is wrap the arguments in braces.

```cpp
#include <iostream>

#define MULTIPLY(x, y) (x) * (y)

int main()
{
	std::cout << MULTIPLY(3, 4) << std::endl;
	std::cout << MULTIPLY(2 + 1, 5 - 1) << std::endl;
}
```

this results in

```
12
12
```

becuase the code after preprocessing was

```cpp
#include <iostream>

#define MULTIPLY(x, y) (x) * (y)

int main()
{
	std::cout << (3) * (4) << std::endl;
	std::cout << (2 + 1) * (5 - 1) << std::endl;
}
```

So 2 + 1 and 5 - 1 will be evaluated before multiplying.

---

Let's assume we want to build a logger in which it logs in console to us, develpers, in **debug** mode and doesn't log anything in **release** mode. We can do this behavior without changing the code of the logger using macros with the following:


First, we have to ***define preprocessor definitions*** for debug and release configurations. I'll do this in Visual Studio but of course this can be done in any IDE or cmake easily. In my project properties -> C/C++ -> Preprocessor. I added **AK_DEBUG** and **AK_RELEASE**. *AK* are just my name's initials to tell you that these are my custom definitions.

<img src="./assets/macros/debug_definition.png">

<img src="./assets/macros/release_defintion.png">

Then in our code we can do this

```cpp
#include <iostream>

#ifdef AK_DEBUG
#define LOG(msg) std::cout << msg << std::endl
#else
#define LOG(msg)
#endif

int main()
{
	LOG("Message");
}
```

`AK_DEBUG` is defined in debug mode. So compiling this in debug mode prints out `Message`. But, it isn't defined in release mode, so compiling this in release mode results in not printing anything since `LOG("Message")` was replaced with nothing as if the code was

```cpp
// debug mode

#include <iostream>

int main()
{
	std::cout << "Message" << std::endl;
}
```


```cpp
// release mode

#include <iostream>

int main()
{

}
```

In macros we can replace text with a multi-line block. Just make sure that after every single line, add `\` to escape the newline.

```cpp
#include <iostream>

#define MAIN_FN int main()\
{\
	std::cout << "Hello from Macro\n";\
}

MAIN_FN
```

And this works fine.

## `auto`

`auto` is a keyword that tells the compiler to deduce the type automatically without explicitly writing it from the user.

```cpp
#include <iostream>

int main()
{
	auto a = 5;		// a is int 
	auto b = a;		// b is int
	auto c = 1L;		// c is long
	auto d = 5.5f;		// d is float
	auto e = "hello";	// e is const char*
}
```

`auto` can be very useful for long types. For example this code,

```cpp
std::vector<std::string> strings;
strings.push_back("apple");
strings.push_back("orange");

for (std::vector<std::string>::iterator it = strings.begin(); it != strings.end(); ++it)
{
	std::cout << *it << std::endl;
}
```

can be reduced with `auto` to

```cpp
std::vector<std::string> strings;
strings.push_back("apple");
strings.push_back("orange");

for (auto it = strings.begin(); it != strings.end(); ++it)
{
	std::cout << *it << std::endl;
}
```

and this code

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <map>
#include <unordered_map>

class Device {};

class DeviceManager
{
private:
	std::unordered_map<std::string, std::vector<Device*>> m_devices;
public:
	const std::unordered_map<std::string, std::vector<Device*>>& get_devices() const
	{
		return m_devices;
	}
};

int main()
{
	DeviceManager device_manager;
	const std::unordered_map<std::string, std::vector<Device*>>& devices = device_manager.get_devices();
}
```

can be reduced to


```cpp
#include <iostream>
#include <string>
#include <vector>
#include <map>
#include <unordered_map>

class Device {};

class DeviceManager
{
private:
	std::unordered_map<std::string, std::vector<Device*>> m_devices;
public:
	const std::unordered_map<std::string, std::vector<Device*>>& get_devices() const
	{
		return m_devices;
	}
};

int main()
{
	DeviceManager device_manager;
	auto devices = device_manager.get_devices();
}
```

>right? Nope. But why?!

Here `auto` is considered as `std::unordered_map<std::string, std::vector<Device*>>` not as `const std::unordered_map<std::string, std::vector<Device*>>&`. This will make a copy of `m_devices` not a reference to it. To use the latter type, you need to specify the const and reference qualifiers like this,

```cpp
const auto& auto devices = device_manager.get_devices();
```

You can check more on that in this [article](https://www.learncpp.com/cpp-tutorial/type-deduction-with-pointers-references-and-const/).

You can also use `auto` for function return types.

```cpp
class Device {};

class DeviceManager
{
private:
	std::unordered_map<std::string, std::vector<Device*>> m_devices;
public:
	const auto& get_devices() const
	{
		return m_devices;
	}
};

```


## Function Pointers and Lambdas

A function pointer is a way to assign a function to a variable, and pass functions to other functions as arguments. Functions are stored in the memory in the text segment, thus have an address. And we can assign this address to a variable to use it elsewhere. Let's consider the following example,

```cpp
#include <iostream>

void print_msg(const std::string &msg)
{
	std::cout << msg << std::endl;
}

int main()
{
	auto print_msg_p = print_msg;
}
```

Here, `print_msg_p` is a pointer to function. But the specific type can be found when hovering on `print_msg_p` to be `void(*print_msg_p)(const std::string &)`. When translating this it becomes, `print_msg_p` is a pointer to a function that receives a `const` reference `&` to a `std::string` and return `void` (nothing). So, we can replace `auto` with

```cpp
void(*print_msg_p)(const std::string &) = print_msg;
// here print_msg_p is the name of the pointer to function, can be named anything else
```

We can use `print_msg_p` as if it was the function name itself by passing arguments.

```cpp
#include <iostream>

void print_msg(const std::string &msg)
{
	std::cout << msg << std::endl;
}

int main()
{
	void(*print_msg_p)(const std::string &) = print_msg;
	print_msg_p("Hello World");
	print_msg_p("Hello From Function Pointer");
}
```

will result in

```
Hello World
Hello From Function Pointer
```

To reduce the need to write `void(*p)(const std::string &)` for each function pointer we create, we can do one of the following (or both):

* Use `auto`
* Use `typedef`

```cpp
#include <iostream>

void print_msg(const std::string &msg)
{
	std::cout << msg << std::endl;
}

typedef void(*Print_Msg_P)(const std::string &);

int main()
{
	Print_Msg_P p = print_msg;
	p("Hello World");
	p("Hello From Function Pointer");
}
```

The function pointers can be very useful when passing to other functions.

```cpp
#include <iostream>
#include <vector>

int multiply_by_2(int val)
{
	return val * 2;
}

void apply_for_each(std::vector<int>& v, int(*func)(int))
{
	for (auto& val : v)
		val = func(val);
}

void print_vector(const std::vector<int>& v)
{
	for (const auto& val : v)
	{
		std::cout << val << std::endl;
	}
}

int main()
{
	auto v = std::vector<int>({ 1, 2, 3, 4, 5 });
	print_vector(v);
	apply_for_each(v, multiply_by_2);
	std::cout << "-------------------------" << std::endl;
	print_vector(v);
}
```

Here, we have `apply_for_each` is a function that receives a vector `v` of integers by reference "modifiable" and `function` that will be applied on each value. Here we expect `function` to be `int(*function)(int)` which is what we have with `multiply_by_2` that is a function that takes an `int` as argument and returns `int`. `apply_for_each` modify each value in `v` with the return of `multiply_by_2`. We print the vector before and after applying `apply_for_each`. The above code results in,


```
1
2
3
4
5
-------------------------
2
4
6
8
10
```

When passing functions to other functions this way, we need to define the function separately and then pass it. Then can be useful if the function will be used in other times in your code. However, if you want to apply this specific function only few times, instead of writing the function, you can write a ***lambda expression***.

A ***lambda expression*** is a way to write these anonymous functions, those passed to higher-order functions, to define that function in the time(line) you use it. A very simple syntax for a function that takes `int` and returns `int` is 

```cpp
[](int x){return x * 2;};
```

this can be used instead of function pointers in the previous example as following,

```cpp
#include <iostream>
#include <vector>

void apply_for_each(std::vector<int>& v, int(*func)(int))
{
	for (auto& val : v)
		val = func(val);
}

void print_vector(const std::vector<int>& v)
{
	for (const auto& val : v)
	{
		std::cout << val << std::endl;
	}
}

int main()
{
	auto v = std::vector<int>({ 1, 2, 3, 4, 5 });
	print_vector(v);
	apply_for_each(v, [](int val){return val * 2;});
	std::cout << "-------------------------" << std::endl;
	print_vector(v);
	apply_for_each(v, [](int val){return val * 3;});
	std::cout << "-------------------------" << std::endl;
	print_vector(v);
	apply_for_each(v, [](int val){return val / 6;});
	std::cout << "-------------------------" << std::endl;
	print_vector(v);
}
```

results in

```
1
2
3
4
5
-------------------------
2
4
6
8
10
-------------------------
6
12
18
24
30
-------------------------
1
2
3
4
5
```

No need to define functions explicitly to multiply by 2, 3 or divide by 6.

The lambda follows the following syntax :
1. `[ ]` is called the **catpure** which are outside objects captured from your code into your lambda expression to be used inside your function.
   * `[a, &b]` captures `a` by copy and `b` by reference.
   * `[=]` captures everything by copy
   * `[&]` captures everything by reference

***VERY IMPORTANT NOTE:*** A lambda expression with an empty capture clause is convertible to a function pointer. So, for the above example,

This works

```cpp
apply_for_each(v, [](int val){return val * 2;});
```


while these don't

```cpp
int x = 0;
apply_for_each(v, [=](int val){return val * 2;});
apply_for_each(v, [&](int val){return val * 2;});
apply_for_each(v, [x](int val){return val * 2;});
```

Solution ? Convert

```cpp
void apply_for_each(std::vector<int>& v, int(*func)(int))
```

to 

```cpp
#include <functional>

void apply_for_each(std::vector<int>& v, const std::function<int(int)> &func)
```

this works for both c-style raw function pointers and lambda expression with non-empty capture clauses.

2. `()` are filled with parameters that the function (lambda expression) should expect.

	* `[](int x)` function takes an `int x`.
	* `[]()` function takes nothing.
	* `[](int x, float y)` function takes an `int x` and `float y`.

3. `{}` which is filled with the body of the actual function utilizing the paramaters passed and objects captured.

>There is more than that for the lambda expression syntax which can be explained easily [here](https://learn.microsoft.com/en-us/cpp/cpp/lambda-expressions-in-cpp?view=msvc-170).


## Threads
Multithreading is a feature that allows concurrent execution of two or more parts of a program for maximum utilization of the CPU. Each part of such a program is called a thread. So, threads are lightweight processes within a process. So, we want a specific part (a function) to be executed in another thread and some main program to be executed in the main thread.

For example, let's say that a function prints a message continously until an input from the user is received, e.g., enter button pressed. The input function `std::cin.get();` would block the code until receiving the input while we want to print the message continously until the input is received.

We can do this by making the function that prints the message in a separate thread and keep the one that waits for the input in the main thread. This can be done like this,

1. Make the function that prints in a separate thread.

First, we define that function,

```cpp
static bool s_Finished = false;

void print_msg()
{
	while (!s_Finished)
	{
		std::cout << "Working...." << std::endl;
	}
	std::cout << "Finished..." << std::endl;
}
```

this function continously print a message until a condition `!s_Finished` is `false`. This happens when `s_Finished` is assigned to `true`.

2. In our main function, we define that we want to execute `print_msg` in a separate thread. This can be done by instanting `std::thread` by passing `print_msg`

```cpp
int main()
{
	std::thread worker(DoWork);
}
```

This instantly begins the thread, thus priniting continously as nothing sets `s_Finished` to `true`. Now, we need to set `s_Finished` to `ture` in the main function after `std::cin.get();` is executed.

```cpp
int main()
{
	std::thread worker(print_msg);
	std::cin.get(); // block the main thread until input is received
	s_Finished = true; // the loop of print_msg will stop 
}
```

To avoid [full CPU load](https://stackoverflow.com/a/1616409), we can add time sleep between each 2 iterations in the infinite loop. [Check also this link](https://stackoverflow.com/questions/1616392/why-does-an-empty-loop-use-so-much-processor-time)

This can be achieved by doing this

```cpp
void print_msg()
{
	using namespace std::literals::chrono_literals; // to use the s user-define literal operator
	while (!s_Finished)
	{
		std::cout << "Working...." << std::endl;
		std::this_thread::sleep_for(1s); // for that specific thread wait 1 second
	}
	std::cout << "Finished..." << std::endl;
}
```

So our final program will be something like this

```cpp
#include <iostream>
#include <thread>

static bool s_Finished = false;

void print_msg()
{
	using namespace std::literals::chrono_literals;
	while (!s_Finished)
	{
		std::cout << "Working...." << std::endl;
		std::this_thread::sleep_for(1s);
	}
	std::cout << "Finished..." << std::endl;
}

int main()
{
	std::thread worker(print_msg);
	std::cin.get();
	s_Finished = true;
}
```

when we run the program, it crashes when input is received. This happens because our thread wasn't joined or detached before it was destroyed, because it was created on the stack thus destroyed at the end of the scope. [Check this link](https://leimao.github.io/blog/CPP-Ensure-Join-Detach-Before-Thread-Destruction/).

When the destructor of `std::thread` is called, it checks if the object is still associated with a thread or not. If it does while calling the destructor, it calls `std::terminate()` which kills the runtime of the program. So you should call either `join` or `detach` depending on the behavior you want. [Check this link](https://stackoverflow.com/a/37021767).

So, we should modify the code to be

```cpp
#include <iostream>
#include <thread>

static bool s_Finished = false;

void print_msg()
{
	using namespace std::literals::chrono_literals;
	while (!s_Finished)
	{
		std::cout << "Working...." << std::endl;
		std::this_thread::sleep_for(1s);
	}
	std::cout << "Finished..." << std::endl;
}

int main()
{
	std::thread worker(print_msg);
	std::cin.get();
	s_Finished = true;
	worker.join();
}
```

Here, `worker.join();` waits until the thread of execution finishes completely and the thread object can be destroyed safely.
