# Understanding Golang Interfaces

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2015-09-11
- Tags: Go
- Canonical: https://kunalkushwaha.github.io/2015/09/11/understanding-golang-interfaces/

Even after writing Go code for a while, there have been couple of time, when I get confused about Interfaces in golang.
So I think, it may help people, who have started with Golang, or don’t use much Interfaces in there code.

### Interfaces

By definition, Interfaces are named collections of method signatures. But usage of Interfaces in go is little confusing as it is used a in context of data type also and defining the behavior of methods also.

Lets try to understand each of them.

### 1. interface{} , interface as data-type

This is commonly mistaken as ( void type of C/C++), But it’s not! It still holds its property from the above definition.
Since, `interface{}` is empty interface i.e. No methods associated with it, all data type satisfy the behavior and hence, all data types can be passed/assigned to `interface{}` type.

Lets take a example,

```golang

type Stack struct {
       data []interface{}
}

func NewStack() *Stack {
	s := Stack{data: make([]interface{},0)}
	return &s
}

func (s *Stack) Push(data interface{}) {
	s.data = append(s.data, data)
}

func (s *Stack) Pop() interface{} {
	if len(s.data) == 0 {
		return nil
	}
	data := s.data[len(s.data) -1]
	s.data = s.data[0:len(s.data)-1]
	return data
}
```

To use the above stack, a simple example shows retrieval of values from interface type.

```golang
s := NewStack()
s.Push(10)
s.Push("Hello")
obj1 := s.Pop()
obj2 = s.Pop()
// To retrieve int value, use type assertion.
elem1 := obj.(string)
elem2 := obj.(int)
```

Here are few things needs to be understood about interfaces to be used as type.

- Whenever variables of any datatype is assigned to interface type, it is converted into interface type and stored.
  - So properties of original data-type cannot be retrieved until, it is converted again back to original data-type.
- conversion to data-type from interfaces cannot be achieved using typecasting, Here it required, [type assertion](http://golang.org/ref/spec#Type_assertions).

So use interface{}, wherever you wish to add generic data-type and remember at retrieval time, use type assertion!

### 2. Interface as set of methods.

Unlike above type, here interface have some function prototype.

If you come from “C/C++” background, you can relate it with function pointers or abstract functions/pure virtual functions. These functions don’t have any implementation.

But, to implement these functions, no need to inherit in structs or assign to function pointers.

In Go, if some structs or functions have same behavior as of interface, it automatically can be used in interface. Here behavior mean, all functions in interface should be implemented with same signature of functions as of interface.

Lets take a simple example.

```golang
type Plugin interface {
    Execute(string, int) bool
}
```

Here, a Plugin interface is defined with one function. Any Structure, which have this one function, will automatically implements the interface.

i.e. Any structure, which has same behavior means the Plugin interface can represent that structure. No explicitly need to show via any keyword like implement or inherit.

I feel, it is very simple approach to build OOP design. Such simple approaches, to create design patterns, make Go even more interesting.

Lets see examples which explains the behavior of interfaces

### 1. Example #1

```golang
type aPlugin struct {
    name string
}

func (p aPlugin) Execute(name string, count int) bool {
    fmt.Println("aPlugin executing", name, count)
    return 0
}
```

Here, structure `aPlugin` have exactly same function as of `Plugin` interface, so this `aPlugin` can be assigned to Plugin interface.

So main function will look like this.

```golang
func main() {
   // Create a Plugin array
   obj := []Plugin{aPlugin{}, bPlugin{}}
   obj[0].Execute("Hello", 1)
   obj[1].Execute("World", 2)
}

$ go run main.go
aPlugin executing Hello 1
bPLugin executing World 2
```

### 2. Example #2

```golang
type bPlugin struct {
    name string
}

func (p bPlugin) Execute(name string, count int) bool {
    fmt.Println("bPlugin executing", name, count)
    return 0
}

func (p bPlugin) DumpInfo() {
    fmt.Println(p.name)
}
```

Here, another structure `bPlugin` have one additional function then of `Plugin` interface. So this `bPlugin` can be assigned to Plugin interface.

Only, if it is assigned to Plugin object, the additional function DumpInfo() cannot be invoked from Plugin object.
- So a Structure can have additional functions, data members and still it can assigned to interface object.

### 3. Example #3

```golang
type cPlugin struct {
    name string
}

func (p cPlugin) Execute() bool {
    fmt.Println("cPlugin executing")
    return 0
}
```

Here, structure `cPlugin` have function with same name `Execute`, but with different signatures. So cPlugin object cannot be assigned to `Plugin` object.

go compiler will raise error if it is tried to do so.

```golang
obj := []Plugin{aPlugin{}, bPlugin{}, cPlugin{}}
```

```
$ go run main.go
./main.go:22: cannot use cPlugin literal (type cPlugin) as type Plugin in array element:
       cPlugin does not implement Plugin (wrong type for Execute method)
               have Execute() bool
               want Execute(string, int) bool
```

## How should these interfaces should be used.

Interfaces are one of most important building blocks of golang. Also, its usage make golang as language very simpler to use.
for example, you can look into [“io” package](http://golang.org/src/io/io.go).

If you implement any kind of framework or as simple as muti-platform program, where you wish functionalities, differ for each OS, interfaces are perfect candidates to use.

I hope this blog would have helped to understand the interfaces in Golang.
Please comment in case you have any feedback.

Happy Coding :)
