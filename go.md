## 1. 通用
### 1.1 缩进
```
代码 必须使用4个空格符的缩进，一定不可用 tab键。
```
> 备注：使用空格而不是「tab键缩进」的好处在于，
避免在比较代码差异、打补丁、重阅代码以及注释时产生混淆。
并且，使用空格缩进，让对齐变得更方便。

### 1.2 行长
```
每一行代码字符数不超过 80.
```

### 1.3 空格
Don't:
```
a:=1 // 赋值时
if x==1 // 写结构化语句时
func main(){ // 写函数时
```
Do
```
a := 1
if x == 1
func main() {
```
不要小看一个空格的区别

### 1.4 注释
Do
```
只保留必要的注释，没有用的注释删掉
```

## 2. 错误处理
### 2.1 给错误添加上下文
Don't:
```
file, err := os.Open("foo.txt")
if err != nil {
	return err
}
```
这种处理方式会导致错误信息不清晰，因为丢失了错误本来的上下文。

Do:
```
file, err := os.Open("foo.txt")
if err != nil {
	return errors.New("open foo.txt failed")
}
```

## 3. 测试

### 3.1 使用assert库
Don't:
```
func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	if (actual != expected) {
		t.Errorf("Expected %d, but got %d", expected, actual)
	}
}
```
Do:
```
func TestAdd(t *testing.T) {
	actual := 2 + 2
	expected := 4
	assert.Equal(t, expected, actual)
}
```
使用 assert 库使测试代码更可读，节省冗余的代码并提供稳定的错误输出。

### 3.2 使用表驱动的测试
Don't:
```
func TestAdd(t *testing.T) {
	assert.Equal(t, 1+1, 2)
	assert.Equal(t, 1+-1, 0)
	assert.Equal(t, 1, 0, 1)
	assert.Equal(t, 0, 0, 0)
}
```
上面的程序看着还算简单，但是想找一个 fail 掉的 case 却非常麻烦，特别是有几百个 test case 的时候尤其如此。

Do:
```
func TestAdd(t *testing.T) {
	cases := []struct {
		A, B, Expected int
	}{
		{1, 1, 2},
		{1, -1, 0},
		{1, 0, 1},
		{0, 0, 0},
	}

	for _, tc := range cases {
		t.Run(fmt.Sprintf("%d + %d", tc.A, tc.B), func(t *testing.T) {
			assert.Equal(t, t.Expected, tc.A+tc.B)
		})
	}
}
```
使用表驱动的 tests 结合子测试能够让你直接看到哪些 case 被测试，哪一个 case 失败了。

## 4. 代码组织

### 4.1 分块组织import
Don't:
```
import (
	"encoding/json"
	"github.com/some/external/pkg"
	"fmt"
	"github.com/this-project/pkg/some-lib"
	"os"
)
```
Do:
```
import (
	"encoding/json"
	"fmt"
	"os"

	"github.com/some/external/pkg"

	"github.com/this-project/pkg/some-lib"
)
```
将 std，外部包和 internal 导入分开写，可以提高可读性。

### 4.2 包文件拆分
Don't:
```
- http.go 
```
Do
```
- doc.go       // package documentation
- headers.go   // HTTP headers types and code
- cookies.go   // HTTP cookies types and code
- http.go      // HTTP client implementation, request and response types, etc.
```
一个包里面因为包含了多个go文件，所以懂得如何通过逻辑关系来拆分不同的go文件能够使得代码提高可读性


## 5. 编码
### 5.1 避免全局变量
Don't:
```
var db *sql.DB

func main() {
	db = // ...
	http.HandleFunc("/drop", DropHandler)
	// ...
}

func DropHandler(w http.ResponseWriter, r *http.Request) {
	db.Exec("DROP DATABASE prod")
}
```
全局变量会使测试难度增加，会使代码的可读性降低，每一个函数都能够访问这些全局变量(即使是那些根本就不需要操作全局变量的函数)。

Do:
```
func main() {
	db := // ...
	http.HandleFunc("/drop", DropHandler(db))
	// ...
}

func DropHandler(db *sql.DB) http.HandleFunc {
	return func (w http.ResponseWriter, r *http.Request) {
		db.Exec("DROP DATABASE prod")
	}
}
```
使用高阶函数来按需注入依赖，而不是全局变量。


### 5.2 尽量使用纯函数（单一职责）
Don't:
```
func MarshalAndWrite(some *Thing) error {
	b, err := json.Marshal(some)
	if err != nil {
		return err
	}

	return ioutil.WriteFile("some.thing", b, 0644)
}
```
Do:
```
// Marshal is a pure func (even though useless)
func Marshal(some *Thing) ([]bytes, error) {
	return json.Marshal(some)
}
```
纯函数并不一定在所有场景下都适用，但保证你用到的函数尽量都是纯函数能够让你的代码更易理解，且更容易 debug。

### 5.3 避免不加修饰的return
Don't:
```
func run() (n int, err error) {
	// ...
	return
}
```
Do:
```
func run() (n int, err error) {
	// ...
	return n, err
}
```
命名返回的值对文档编写或者生成是有益的，不加任何修饰的 return 会让代码变得难读而易错。

### 5.4 避免空接口
Don't:
```
func run(foo interface{}) {
	// ...
}
```
空接口会让代码变得复杂而不清晰，只要能够不使用，就应该在任何时候避免。

### 5.5 main函数先行
Don't:
```
package main // import "github.com/me/my-project"

func test() int {
	// ...
}

func main() {
	// ...
}
```
Do:
```
package main // import "github.com/me/my-project"

func main() {
	// ...
}

func test() int {
	// ...
}
```
将 main() 函数放在你文件的最开始，能够让阅读这个文件变得更加轻松。如果有 init() 函数的话，应该再放在 main()之前。

### 5.6 使用可变参数列表
Don't:
```
func add(num1, num2, num3 int) int {
    sum := 0
    sum = num1 + num2 + num3
    return sum
}
```
Do
```
func add(nums ...int) int {
    sum := 0
    for _, num := range nums {
        sum += num
    }
    return sum
}
So(add(1, 2), ShouldEqual, 3)
So(add(1, 2, 3), ShouldEqual, 6)
```

### 5.7 使用struct作为函数参数
Don't
```
func MyFunc1(requiredStr string, str1 string, str2 string, int1 int, int2 int) {
    fmt.Println(requiredStr, str1, str2, int1, int2)
}
// 调用方法
MyFunc1("requiredStr", "defaultStr1", "defaultStr2", 1, 2)
```
这种实现比较简单，但是同时传入参数较多，对调用方来说，使用的成本就会比较高，而且每个参数的具体含义这里并不清晰，很容易出错

Do

```
type MyFuncOption struct {
    optionStr1 string
    optionStr2 string
    optionInt1 int
    optionInt2 int
}

func MyFunc1(options MyFuncOption) {
    fmt.Println(options.optionStr1, options.optionStr2, options.optionInt1, options.optionInt2)
}

var myFuncOption = MyFuncOptions{
    optionStr1: "defaultStr1",
    optionStr2: "defaultStr2",
    optionInt1: 1,
    optionInt2: 2,
}

MyFunc1(myFuncOption)
这种写法还可解决函数参数过多的问题，所谓一切皆对象
```

### 5.8 变量命名带命名空间
Don't
```
BUS_COLOR          = "#438BBF" // 公交
SHUTTLE_BUS_COLOR  = "#438BBF" // 班车
SUBWAY_COLOR       = "#438BBF" // 地铁
BICYCLE_COLOR      = "#5BC2AF" // 骑行
```

Do
```
COLOR_BUS          = "#438BBF" // 公交
COLOR_SHUTTLE_BUS  = "#438BBF" // 班车
COLOR_SUBWAY       = "#438BBF" // 地铁
COLOR_BICYCLE      = "#5BC2AF" // 骑行
```
变量命名从大到小，有命名空间的感觉

### 5.9 复杂的逻辑抽成函数
Don't
```
if (a == 'aaaa' || b == 'bbbbb' || (c == 0 && d == 1)) {
    return true
}
return false
```
一般来说，这么长的判断，已经是一个复杂的逻辑了，需要用一个方法指明具体判断了什么，而不是去读代码

Do
```
func doSomething() {
   if (a == 'aaaa' || b == 'bbbbb' || (c == 0 && d == 1)) {
     // ...
   }
   // ...
}

主程序调用doSomething()
```




