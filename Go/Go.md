# Go语言教程

## 数据类型

### bool类型

```go
var success bool = true
```

### 数字类型

#### 整形

1. `unit8`：无符号 8 位整型 (0 到 255)
2. `unit16`：无符号 16 位整型 (0 到 65535)
3. `unit32`：无符号 32 位整型 (0 到 4294967295)
4. `unit64`：无符号 64 位整型 (0 到 18446744073709551615)
5. `int8`：有符号 8 位整型 (-128 到 127)
6. `int16`：有符号 16 位整型 (-32768 到 32767)
7. `int32`：有符号 32 位整型 (-2147483648 到 2147483647)
8. `int64`：有符号 64 位整型 (-9223372036854775808 到 9223372036854775807)

#### 浮点型

1. `float32`
2. `float64`
3. `complex64`
4. `complex128`

#### 字符串

```go
var name string = "hxy"
```

## 变量声明

1. 通用形式

   ```go
   var num int16 = 23
   ```

2. 省略类型或者初始值，如果省略类型则初始化表达式决定其类型，如果省略初始化表达式则其值默认为该类型的初始值

   ```go
   var num int
   var num = 23
   ```

3. 省略var关键字，使用 **:=** 完成初始化声明，变量类型通过系统自行推断

   ```go
   name := "hxy"
   ```

### 常量声明

1. 通过const声明常量，类型可以写也可以省略

   ```go
   const PI = 3.14
   const NAME string = "GO"
   ```

2. 也可以声明一组常量，类型可以不同

   ```go
   const (
       name = "go"
       age  = 18
   )
   ```

3. 声明一组常量时，除过第一个常量后面的可以省略类型和初始化值，会复用前一个常量的值

   ```go
   const (
       name = "go"
       age
   )
   // 打印name和age会输出 go go
   ```

4. iota常量生成器，iota会从0开始取值，后续常量如果没有指定类型和进行赋值，那么他的值会是前面一个常量的值加1

   ```go
   const (
       Sunday = iota
       Monday
       Tuesday
   )
   ```

## 条件语句

### if

```go
func main() {
	a := 20
    // 不需要像java一样括起来
	if a > 20 {
		fmt.Println(a)
	}
}
```

### if else

```go
func main() {
	a := 20
	if a > 20 {
		fmt.Println("a大于20")
	} else if a == 20 {
		fmt.Println("a等于20")
	} else {
		fmt.Println("a小于20")
	}
}
```

### switch

switch不像java需要写break语句，默认支持break，如果要执行下面的case，需要用`fallthrough`关键字强制执行，但是这个关键字不管后面的case是否满足，都会执行，如果想退出也可以用`break`。

```go
switch marks {
    case 90:
    	grade = "A"
    case 80:
    	grade = "B"
    case 50, 60, 70:
    	grade = "C"
    default:
    	grade = "D"
}

switch {
    case grade == "A":
    	fmt.Println("优秀")
    case grade == "B", grade == "C":
    	fmt.Println("良好")
    case grade == "D":
    	fmt.Println("及格")
    default:
    	fmt.Println("差")
}
```

### select

## 循环语句

### for循环

```go
sum := 0
for i := 0; i < 10; i++ {
    sum += i
}

fmt.Println(sum)
```

go语言没有while循环语句，`for condition { }`类似于`while`，`for {}`类似于`for(;;)`表示无限循环

### for range循环

如果通过下面的方式遍历数组，index表示索引，item表示该索引位置的值

```go
numbers := [6]int{1, 2, 3, 5}
for index, item := range numbers {
    fmt.Printf("第 %d 位 x 的值 = %d\n", index, item)
}
```

如果不需要索引，可以通过下面的方式忽略索引

```go
for _, item := range slice {
    fmt.Printf("%s\n", item)
}
```

如果只需要索引，可以忽略第二个变量

```go
for index := range slice {
    fmt.Printf("%d\n", index)
}
```

## 函数

### 函数定义

go语言中的函数用func关键字来定义，返回值类型写在形参列表后面。看起来怪怪的。

```go
func main() {
	max := max(2, 3)
	fmt.Println(max)

}

func max(num1, num2 int) int {
	if num1 > num2 {
		return num1
	}
	return num2
}
```

go语言中的函数还支持返回多个值，这是比较厉害的

```go
func main() {
	str1, str2 := swap("hxy", "lmm")
	fmt.Println(str1, str2)

}

func swap(x, y string) (string, string) {
	return y, x
}

```

### 匿名函数

### 闭包

## 数组

```go
var arr = [5]string{"java", "go", "c", "c++", "python"}
arr := [5]string{"java", "go", "c", "c++", "python"}
// 长度不知道的话，可用...表示
arr := [...]string{"java", "go", "c", "c++", "python"}
// 初始化指定索引的元素的值
arr := [5]string{1: "java", 3: "go"}

for i, x := range arr {
    fmt.Printf("第 %d 位 x 的值 = %s\n", i, x)
}

for i := 0; i < len(arr); i++ {
    fmt.Printf("第 %d 位 x 的值 = %s\n", i, arr[i])
}
```

## 切片

数组的长度是固定的，不能动态扩展。并且在Go语言中，数组是值传递的方式，这一点和C语言不同。在C语言中，数组名是指向数组中索引为0的元素的地址，也就是人们说的引用。而go语言中则不是。例如下面这段代码，我们将一个数组传给函数modify，在modify中修改了他的值，但是在main函数中并没有体现。因为数组arr在传给modify的时候，其实是在内存中拷贝了一个新的数组传给了modify。

```go
func main() {
	arr := [5]string{"java", "go", "c", "c++", "ruby"}
	modify(arr)
	fmt.Println("main(), arr values:", arr)

}

func modify(arr [5]string) {
	arr[4] = "Golang"
	fmt.Println("modify(), arr values:", arr)
}

// 输出
modify(), arr values: [java go c c++ Golang]
main(), arr values: [java go c c++ ruby]
```

基于以上两点，go语言的解决方案是提供了切片（slice）这种数据结构，也就是动态数组。切片支持动态扩展，并且是引用传递，更符合C中数组的设定。我们可以通过定义一个没有长度的数组来定义一个切片。例如

```go
var arr = [5]string{"java", "go", "c", "c++", "python"}
arr := []string{"java", "go", "c", "c++", "ruby"}
```

我们也可以通过对一个已知数组**切片（动词）**的方式来定义和初始化数组。例如

```go
arr := [5]string{"java", "go", "c", "c++", "ruby"}
slice := arr[:]
```

这段代码的意思是对arr做了一个切片，切的范围是从0号索引到`len(arr)-1`。包含了arr中的所有元素，那么如果不想切所有元素，可以在 **:** 前后指定切片开始位置和结束位置。例如执行下面这行代码后，slice中的值将是：[c c++]

```go
arr := [5]string{"java", "go", "c", "c++", "ruby"}
slice := arr[2:4]
```

通过对数组切片的方式创建一个切片后，其实是相当于为原来的数组创建了一个引用，以上面的代码为例，slice其实可以看做指向arr的一个指针，他们是共享一块内存的，如下代码：我们在第四行修改了arr[3]的值，那么在打印slice，发现输出的是修改后的值。

```go
arr := [5]string{"java", "go", "c", "c++", "ruby"}
slice := arr[2:4]

arr[3] = "python"
fmt.Println("main(), arr values:", slice)

// 输出
main(), arr values: [c python]
```

切片还有两个属性，一个是len、一个是cap。len是切片中包含元素的个数，cap是切片的容量。这两个属性可以通过`len()`和`cap()`方法获取。因此一个切片可以看做一个包含指向对应数组的指针、切片长度、切片容量这么三个属性的结构体。

除过对已知数组切片的方式之外，还可以通过make方法和new方法来创建切片。`make()`支持三个参数，切片类型、len和cap，其中cap是可选参数

```go
slice := make([]string, 10, 15)
```

## map

### map的声明和初始化

map类似于java中的map，python中的字典，redis中的hash，其实都是对Hash表的一种实现。Go语言中，生命初始化一个map也很简单，代码如下：

```go
func main() {
    // map[keyType]valueType
	provice := make(map[string]string)

	provice["shanxi"] = "xian"
	provice["gansu"] = "lanzhou"

	fmt.Println(provice["shanxi"])
}
```

如果要声明一个有容量的map，可以用下面的代码：`provice := make(map[string]string, 32)`

从map中根据key获取value，可以用这种写法：`provice["shanxi"]`，这行代码存在多个返回值，除了value，还返回一个表示value是否存在的bool值。如下

```go
func main() {
	provice := make(map[string]string, 32)

	provice["shanxi"] = "xian"
	provice["gansu"] = "lanzhou"

	var value string
	var exists bool
	value, exists = provice["shanxi"]
	if exists {
		fmt.Println(value)
	}
	
    // 上面的写法还可以简化成下面这样
	if value, exists := provice["shanxi"]; exists {
		fmt.Println(exists)
		fmt.Println(value)
	}
    
    // 如果忽略value，可以用_省去value值
    if _, exists := provice["shanxi"]; exists {
		fmt.Println(exists)
	}

}
```

如果要从map中删除一个key，可通过delelte方法删除`delete(provice, "shanxi")`。

### map的遍历

map的遍历可以用for range循环去做。

```go
func main() {
	provice := make(map[string]string, 32)

	provice["shanxi"] = "xian"
	provice["gansu"] = "lanzhou"

	for key, value := range provice {
		fmt.Printf("key is %s, value is %s\n", key, value)
	}

}
```

map的遍历中，key是一个可选参数，如果不关心key的话，可以用下面的写法只取value

```go
for _, value := range provice {
    fmt.Printf("key is %s\n", value)
}
```

如果只想取key，那么也可以用下面的写法省去value

```go
for key := range provice {
    fmt.Printf("key is %s\n", key)
}
```

## 结构体

### 结构体的定义和初始化

除了int、float、字符串类型之外，实际开发过程中还会遇到很多更复杂的类型，比如一个学生、老师等等，就不能用基本类型来表示了。在C中用结构体来表示，在java中用类来定义，在Go中也用结构体来表示。比如我们定义一个学生结构体，代码如下：

在main方法中，我们通过`new`关键字来分配一块内存，用来承载一个student结构体，stu可以看做一个指向这块内存的指针，`stu := new(student)`这行代码也可以写成`var stu *student = new(student)`这样，类似于C语言中常见的这种写法：`student* stu = malloc(sizeof(struct student));`。其含义就是定义 一个studnet类型的指针stu，并开辟一块内存空间，让stu指向这块空间。那么在java中，则是这样的写法`Studnet stu = new Studnet()`。意思是一样的。

```go
// 定义一个结构体，和c的语法很相似
type student struct {
	name string
	age  int
	love []string
}

func main() {
	stu := new(student)
	stu.age = 12
	stu.name = "hxy"
	stu.love = make([]string, 5)
	stu.love = append(stu.love, "sing", "pingpang")

	fmt.Println(*stu)
}

```

在上面的代码片段中，我们通过`stu.age`等这样的表达式给一个结构体实体的属性进行了赋值，着实是有点麻烦的？有没有省事的办法呢。在C语言中我们可以通过下面的方式`struct student stu = {.name = "hxy"}`的方式初始化，java中可以通过构造函数的方式初始化，那么在Go语言中也有类似的语法，比如：

```go
love := []string{"java", "go"}
// 按照结构体中属性定义的顺序初始化
stu := student{"hxy", 12, love}

// 为指定属性赋值，没有指定的为初始值
stu := student{name: "hxy", age: 12}
```

**注意：**

1. 通过`stu := new(student)`这种方式创建一个结构体，我们说这里的stu是一个指针
2. 通过`stu := student{"hxy", 12, love}`这种方式呢，stu却不是一个指针，作为参数传递是值传递，具体参考下面几段代码

```go
func main() {
	stu := student{name: "hxy", age: 12}
	modify(stu)
	fmt.Println(stu)
}

func modify(stu student) {
	stu.age = 20
}
// 输出  age并没有改变，传的不是地址
{hxy 12 []}
```

```go
func main() {
	stu := student{name: "hxy", age: 12}
	modify(&stu) // 将stu的地址传过去，才是引用传递
	fmt.Println(stu)
}

// 这里应该接受一个student类型的指针
func modify(stu *student) {
	stu.age = 20
}

// 输出  age并没有改变，传的不是地址
{hxy 20 []}
```

```go
func main() {
	stu := new(student)
	modify(stu)
	fmt.Println(*stu)
}

func modify(stu *student) {
	stu.age = 20
}

// 输出
{go 20 []}
```

### 带标签的结构体

在java编程中，经常我们会遇到对Bean做一些参数校验、json序列化的问题，这个时候我们经常会写一些注解去帮助我们做，比如`@NotNull`、`@Min(5)`、`@JsonField`等。或者也可以自定义一些注解，去做定制化的操作。那么在Go语言中，也有类似的设计，叫做**结构体的Tag**。

Tag是在结构体编译阶段就关联到结构体中具体字段的一个原信息字符串，可以在运行的时候通过反射获取到，他可以由一个或者多个键值对的组成，键与值使用冒号分隔，值用双引号括起来。键值对之间使用一个空格分隔。比如：

```go
type student struct {
    // 这里的tag是binding标签的一种用法，做参数校验
	name string `form:"name" binding:"required,min=1,max=10"`
	age  int    `form:"age" binding:"lte=150,gte=0"`
	sex  string `form:"sex" binding:"oneof=male female"`
}

func main() {
	stu := student{"hxy", 12, "male"}
	for i := 0; i < 3; i++ {
		stuType := reflect.TypeOf(stu)
		stuField := stuType.Field(i)
		fmt.Printf("%v\n", stuField.Tag)
	}
}

// 输出
form:"name" binding:"required,min=1,max=10"
form:"age" binding:"lte=150,gte=0"
form:"sex" binding:"oneof=male female"
```

### 匿名字段和内嵌结构体

我们都说封装、继承、多态是面向对象的三大特性，而C语言是面向过程的，但是C语言同样也可以实现所谓的封装、继承、多态。我们今天一起看看**继承**。

java中是通过`extends`关键字来实现对父类的继承，在C语言中我们可以通过**组合**来实现继承，比如下面这段代码。

```c
struct Person {
    char[] name;
    int age;
}

struct Studnet {
    struct Person name;
    struct int id;
    struct float score;
}
```

而Go语言也借鉴了这种思想，即通过组合实现继承的特性。比如下面这段代码：

```go
type person struct {
	name string
	age  int
	sex  string
}

type student struct {
	person
	score float32
}

func main() {
	stu := student{person{"hxy", 18, "man"}, 90.5}
	fmt.Println(stu)
	fmt.Println(stu.name)
}

// 输出
{{hxy 18 man} 90.5}
hxy
```

这里呢我们就通过在student中**内嵌**person来实现了继承的效果。可以看到在第8行只定义了字段的类型，没有显示的名字，这在Go语言里面是被允许的，叫做**匿名字段**。

### 结构体和方法

在java中，我们常常见到这样的写法，也就是用类来表示用户自定义的数据结构，里面声明属性，还可以声明一些方法。而在Go里面，用结构体来定义用户自定义的数据结构，也支持为结构体定义一些方法。

```java
public class Student {
    private String name;
    private int age;
    public void printStu() {
        
    }
}
```

上面的java代码如果用Go来表示，就可以写成下面这样。我们在12行定义了一个方法，通过`(stu *student)`指定了这个方法的接受者为student。那么在18行就可以通过`stu.printStu()`的方法区进行调用了。

```go
type person struct {
	name string
	age  int
	sex  string
}

type student struct {
	person
	score float32
}

func (stu *student) printStu() {
	fmt.Printf("%v\n", *stu)
}

func main() {
	stu := &student{person{"hxy", 18, "man"}, 90.5}
	stu.printStu()
}
```

**注意：**

为结构体定义方法，他们必须要求在同一个包里才行，否则会报错。如果不在一个包里，非要定义，那么可以通过定义别名的方式去间接的实现。

## 接口

### 接口的基本使用

Go语言中的接口很简单，通过`interface`就可以定义一个接口，在接口中定义的方法和java一样，是没有实现的，只做声明。接口的名字一般是方法名加`er`组成。不适合用`er`的，也可以用`able`结束。

```go
type Flayer interface {
	fly()
}
```

如上我们定义了一个`player`的接口。Go语言中没有`implements`的关键字，如果要实现一个接口，参考下面的例子：

```go
type bird struct {
}

// 按照接口中fly方法的返回值、参数列表做声明，函数的接受者定义为bird类型的一个指针，就相当于bird实现了Player接口
func (bi *bird) fly() {
	fmt.Println("bird fly...")
}

type plane struct {
}

func (bi *plane) fly() {
	fmt.Println("plane fly...")
}

func main() {
	flayer1 := new(bird)
	flayer1.fly()

	var flayer2 Flayer
	flayer2 = new(plane)
	flayer2.fly()
}

// 输出
bird fly...
plane fly...
```

### 接口类型的检测

因为接口是一个动态类型，就像上面的例子，有可能是bird类型，也有可能是plane类型。那么我们如何在运行时检测他到底是那种类型呢？代码如下：

```go
func main() {
	var flayer2 Flayer
	flayer2 = new(plane)
	flayer2.fly()
	
    // ok是一个bool值 如果为true，则t是plane类型的值
	if t, ok := flayer2.(*plane); ok {
		fmt.Printf("%T", t)
	}
    
    if _, ok := flayer2.(*plane); ok {
		fmt.Printf("%T", t)
	}
    
}

// 输出：
bird fly...
plane fly...
*main.plane
```

除了上面的方式，也可以用`type-switch`的方式去做，代码如下：

```go
switch t := flayer2.(type) {
case *plane:
    fmt.Printf("plane %T\n", t)
case *bird:
    fmt.Printf("bird %T\n", t)
default:
    fmt.Println("unknow")
}
```

如果我们不关注t的取值，那么上面的代码也可以简写成下面的样子：

```go
switch flayer2.(type) {
case *plane:
    // todo
case *bird:
    // todo
default:
    // todo
}
```

### 空接口

在C语言中，有一个函数`malloc`函数用来开辟内存空间，参数是要开辟的空间大小，返回值是开辟出来的空间的地址。那么这个返回的地址是用来存什么类型的数据的呢？int类型，那返回值类型应该定义成`int *p`，float类型，那返回值类型应该定义成`float *p`。。。可是这个是确定不了的，是在用户使用的时候动态确定的，因此它的返回值类型不能是确定的，也应该是动态的。具体怎么表示呢？我们看看`malloc`函数的声明`void *__cdecl malloc(size_t _Size);`，他这里是用了`void *`类型的指针来指向一个空类型，一个不确定类型。在使用的时候我们可以将他强制转换成任意一个确定的类型。比如`void * p1 = malloc(100); int p2 = (int *)p1`。

在java中，类似的功能是通过`Object`类来实现的，他是所有类型的基类，也就是说`Object`类的对象可以指向任意类型的一个引用。在Ts中，类似的写法是`Any`。那么在Go语言中，类似的功能则被叫做空接口。虽然换了一个概念，但是本质解决的问题是相同的。

Go语言中的空接口是接口类型的一种特殊形式，空接口没有任何方法，因此任何类型都无须实现空接口。换句话说，任何值都满足这个接口的需求，都可以看做实现了这个空接口。因此空接口类型可以保存任何值，也可以从空接口中取出原值。如下代码片段：

```go
type Student struct {
	name  string
	age   int
	score float32
}

// 定义一个空接口
type Any interface{}

func main() {
	var val Any
	val = 1  // 将int类型赋值给空接口
	fmt.Printf("val has the value: %v\n", val)

	val = "go"  // 将string类型赋值给空接口
	fmt.Printf("val has the value: %v\n", val)

	stu := Student{"haxingyu", 12, 100.0}
	val = stu  // 将struct类型赋值给空接口
	fmt.Printf("val has the value: %v\n", val)

	switch t := val.(type) {
	case int:
		fmt.Printf("Type int %T\n", t)
	case string:
		fmt.Printf("Type int %T\n", t)
	case Student:
		fmt.Printf("Type int %T\n", t)
	default:
		fmt.Println("unknown")
	}
}

// 输出：
val has the value: 1
val has the value: go
val has the value: {haxingyu 12 100}
Type int main.Student
```



