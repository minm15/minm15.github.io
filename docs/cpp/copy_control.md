# Copy Control

這一篇文章是基於 c++ primer 第十三章的閱讀筆記。
Copy Control 指的是一些傳入相同類型的物件來初始化的特殊 constructor。

## Copy, Assign And Destroy

### Copy Constructor

第一個參數接收相同類型物件的 reference 以及其他預設值的 constructor 被稱為 copy constructor。

初始化的方式分成 direct initialization 和 copy initialization。前者讓編譯器透過傳入的參數找到適合的 constructor 來建構物件，後者則是先產生 rvalue 再拿來建構物件。一個簡單但不精確的說法是，辨別這兩種初始化的方式可以看有沒有 `=`。

當 copy initialization 發生的時候，要嘛會呼叫 copy constructor，不然就是 move constructor。

```cpp
String dots(10, '.');            // direct
String d(dots);                  // direct  
String dots2 = ".........";      // copy  
String dots3 = String(10, '.')   // copy  
```

除了用 `=` 會觸發 copy initialization 之外，還有其他時機

1. 將 non-reference 的物件作為參數傳入時
2. 函式回傳 non-reference 的物件時
3. class物件在大括號內用來初始化陣列的元素或是作為初始化 aggregate class 的成員時

第一個觸發時機有趣的地方是，他隱含地說明 copy ctor 的參數必須是 reference，因為如果它是接受 non-reference 的話，這個參數傳入時會觸發 copy ctor，然後觸發的 copy ctor 的參數又觸發一次 copy ctor，就永無止境的循環下去了。

### Copy initialization 的限制

當 copy initialization 中被用來初始化的物件型別和目標型別不一樣的話會進行隱式轉型。但是當遇到 explicit constructor 的時候，因為 explicit ctor 禁止隱式轉型，所以會引發錯誤。

```cpp
vector<int> vec(10);
vector<int> vec = 10; // error: implicit vector<int> vec = vector<int>(10);
```

### Destructor

destructor 是一個前綴帶有 `~` 而且沒有 parameter 的成員函式，會先執行 function body，再按照 constructor 建立成員的反向順序銷毀 non-static 成員。
注意如果成員是指標的話，destructor 會銷毀指標本身，而不會銷毀指標指向的物件，這會導致潛在的 memory leak。因此比較安全的做法是透過 smart pointer 來表示成員指標，因為當它被銷毀的時候會執行 smart pointer 的 destructor 來刪除指向的物件。

#### 有自己實作的 destructor 也會需要 copy 和 assignment

對於一個指標變數，編譯器生成的 destructor 只會刪除指標本身而不會動到指向的物件。那如果我們自己實作 destructor，其中一個原因是想要一併釋放掉指向的物件。下方展示了一個範例是 ctor 在 heap 動態配置記憶體，因此這個物件看成是這個 class 擁有的資源，所以會希望在 destructor 手動清除指向的物件。

```cpp
class HasPtr {
public:
    HasPtr(const int v): p(new int(v)) {}
    
    ~HasPtr() {
        delete p;
    }

private:
    int *p;
};
```

接著討論 copy ctor 和 copy-assignment operator，考慮是否要手動實作。在編譯器產生的 ctor 中，僅會對 class 內的成員進行 copy 或是 assignment。
在 synthesized copy constructor 對於 `HasPtr` class 只複製成員 `p`，產生的效果是 `p` 和 `rhs.p` 會指向同一個記憶體位址。因此當兩個物件都被銷毀，執行 destructor 時，指向的物件會被釋放兩次，導致未定義行為。

```cpp
class HasPtr {
public:
    // compiler synthesized copy constructor
    HasPtr(const HasPtr& rhs) : p(rhs.p) {}
};
```

### Delete function

前面提到若沒有手動實作 copy ctor 和 copy-assignment operator，編譯器會自動產生。但是有沒有那種不適合 copy 的行為呢？有的，例如 `iostream` 不希望有多個物件可以同時讀寫同一個 buffer，因此會利用 delete function 來禁止這些 function。
寫法就是在宣告的時候加上 `= delete`，以下是 class `basic_ios` 的範例。

```cpp
basic_ios(const basic_ios&) = delete;
basic_ios& operator=(const basic_ios&) = delete;
```

另外 destructor 通常不會定義成 delete function，因為若是這樣做，代表物件離開結束生命週期時沒辦法自動呼叫 destructor，導致這個物件不會被自動釋放，這個情況下，編譯器會禁止定義這個類型的變數。
但是動態配置這樣的變數是合法的，不過仍然沒辦法透過 `delete` 銷毀這個物件，因此通常不會定義 destructor 為 delete function。

```cpp
// define A's destructor as a delete function
A* p;           // failed
A* p = new A(); // successful
delete p;      // failed
```

要判斷一個 copy control member 由編譯器生成的版本是不是 deleted，原則上是看處理 member 時會不會遇上不合法操作。如果某個 member 沒有 destructor，那麼 copy ctor 就會被定義為 delete funtion；如果 member 有 const 或是 reference 類型，那麼 copy-assignment operator 就會被定義為 delete funtion

## Copy Control and Resource Management

當我們要自己定義 copy ctor 和 copy-assignment operator 這些 copy control member 的時候，需要考慮 copy 這個行為是在 value 還是 pointer 的層面。前者是當 copy 發生後，各自就是獨立的，而後者則是相依。
例如 `vector` 或是 `string` 的 copy 是 value 層面的，而 `shared_ptr` 則是 pointer 層面。另外 `unique_ptr` 這種不允許 copy 則是不屬於任何一種。

### value-like class

```cpp
class HasPtr {
public:
    HasPtr(const std::string &s = std::string()) :
        sptr(new std::string(s)), i(0) {}
    HasPtr(const HasPtr &obj):
        sptr(new std::string(*obj.sptr)), i(obj.i) {}
    HasPtr& operator=(const HasPtr &);
    ~HasPtr() { delete sptr; }

private:
    int i;
    std::string *sptr;
};

HasPtr& HasPtr::operator=(const HasPtr &rhs) {
    auto new_sptr = new std::string(*rhs.sptr);
    delete sptr;
    sptr = new_sptr;
    i = rhs.i;
    return *this;
}
```

上述是一個 valuelike class 的範例，如果沒有定義 copy ctor，而採用編譯器生成的版本會是 memberwise copy，因此 `sptr = obj.sptr`，這個效果是兩個指標指向同一個物件，並不符合 valuelike class 的語意，而且如果兩個都結束生命週期時，會導致這個物件被重複釋放。

### pointer-like class

```cpp
class HasPtr {
public:
    HasPtr(const std::string &s = std::string()) :
        i(0), used(new std::size_t(1)), sptr(new std::string(s)) {}
    HasPtr(const HasPtr &obj) :
        i(obj.i), used(obj.used), sptr(obj.sptr) { ++*used; }
    HasPtr& operator=(const HasPtr &);
    ~HasPtr() {
        if (--*used == 0) {
            delete used;
            delete sptr;
        }
    }
private:
    int i;
    std::size_t *used;
    std::string *sptr;
};

HasPtr& HasPtr::operator=(const HasPtr &rhs) {
    ++*rhs.used;

    if (--*used == 0) {
        delete used;
        delete sptr;
    }
    
    i = rhs.i;
    used = rhs.used;
    sptr = rhs.sptr;
    return *this;
}
```

在 pointer-like 的實作中，當不同物件共享同一個 `string` 指標時，要避免 destructor 的重複釋放資源，所以採用類似 `shared_ptr` 的 reference count，當計數爲 0 的時候，才需要釋放資源。

## Moving Objects

要使用 move 而避免 copy 通常有兩個原因

1. 目標物件很快就會被銷毀，因此為了省資源，會避免多餘的 copy
2. 對於 IO 或是 `unique_ptr` 維護的物件，copy 本身是被禁止的，所以只能做 move

### rvalue reference

在講 move 之前，要先理解 rvalue reference，rvalue reference 利用 `&&` 宣告，來綁定 rvalue。簡單區分 lvalue 和 rvalue 的方法是，lvalue 是有名稱的物件；而 rvalue 是一個暫時物件，通常是臨時產生的值。

直接綁定 rvalue reference 到 lvalue 是不被允許的，反之亦然。特別的是，將 rvalue 可以綁定到 const lvalue reference，關鍵就在於不可修改這個特性，一個 non-const lvalue reference 通常綁定一個可以修改的物件，但是 rvalue 通常是臨時結果，若是用 `T&` 綁定的話會產生不明確的語意。

```cpp
int a = 42;
int &&b = a; // error, bind the lvalue to rvalue reference

int &&c = 42;      // ok
int &d = 42;       // error, bind the rvalue to lvalue reference
const int &e = 42; // ok
```

### move

c++ 11 提供 `std::move` 函式來把 expression 轉型成 rvalue，這個情況下，這個物件就要當作 rvalue 來看待，也就是當物件所有權被轉移之後，不應該再讀取原先的物件（因為讀出來的內容不可靠）。

```cpp
std::string s = "hello";
std::string t = std::move(s);
// should not rely on s after the move
```

### move constructor

move ctor 和 copy ctor 有類似的規則，第一個參數是該 class 型別的 reference，且若有其他參數必須有預設值，唯一不一樣的地方是 copy ctor 第一個參數通常是 `const T&`，而 move ctor 是 `T&&`。

另外 move ctor 在實作慣例上要讓 move-from object 在 move 後是一個可以安全被釋放的狀態，也就是 move 之後，object 的 member 不能再擁有原本的資源。從下面範例可以看到 `room_number` 和 `students` 的所有權從 rvalue 轉移，因此不像 copy ctor 需要配置新的記憶體。

一般來說，傳入的 rvalue 即將會被銷毀，如果沒有清掉 move-from object 的所有權，當它的 destructor 被呼叫，那剛轉移的資源就會被釋放，因此把原本的指標設成 nullptr 讓存取舊物件仍然有效，但是不會動到已經轉移的資源。

```cpp
class Room {
public:
    Room(Room &&room) noexcept
    : room_number(room.room_number), students(room.students) {
        // the member shouldn't point to the original resource after the move
        room.room_number = nullptr;
        room.students = nullptr;
    }
private:
    char* room_number
    Student** students;
};    
```

可以注意到，parameter list 後面有 `noexcept`，這是用來明確告知這個 function 不會丟出例外，原因是這裡實作的 move ctor 不會配置任何額外記憶體，因此保證不拋出例外。

明確告知 `noexcept` 有一個好處，以 vector `push_back` 為例，vector 保證在做這個操作若遇到 exception，原本容器內的資料會保持不變。當 vector 需要 reallocate 並且呼叫沒有 noexcept 的 move ctor 的時候，若是 vector 搬移部分資料到新記憶體位址的時候發出例外，容器內有些資料的所有權已經轉移了，所以失去保證，因此 vector 會傾向選擇不會動到原本容器資料的 copy ctor。所以明確指出 `noexcept` 能讓 vector 放心地使用 move ctor，而不是成本較高的 copy。

### move assignment operator

基本的操作和 copy ctor 一樣先 move 再把 move-from object 的 member 清空。只是要多做兩件事情

1. 先釋放掉 move-to object 的資源
2. 明確檢查 self-assignemnt，和 copy assignment 不一樣是因為，後者可以先把 rhs 的值複製下來再釋放 lhs 的資源，即便是 self-assignment 也能保證資料存在。但是 move assignment 不做 copy，因此若是在這種情況 free 的話，rhs 的資料也被釋放了。

### synthesized move operations

沒有定義 copy ctor 和 copy assignment 時，編譯器會自動生成，但是 move 操作不一樣，即便沒有定義 move 系列的操作，只要有定義 copy ctor、copy assignment 或 destructor，編譯器就不會自動生成 move 系列操作。

只有在沒有定義 copy control member 且所有 non-static 的成員（包含可以 move 的 built-in type 以及有定義 move operation 的 class 物件） 都可以被 move 的情況，編譯器才會自動生成 move ctor 和 move-assignemnt。

前面說到 synthesized copy operation 如果會導致不合法操作，會 implicitly 定義為 `= delete`，但是 move operation 會直接不宣告。若是實作上 explicitly 定義 move operation 為 `= default` 且編譯器無法合法 move 所有 member，那麼生成的 move operation 就會被定義為 delete function。

```cpp
// assume X has copy ctor but not a move ctor
struct hasX {
    hasX() = default;
    hasX(hasX&&) = default; // defined as delete function since mem cannot be moved
    X mem;
}
hasX x1, x2 = std::move(x1); // error, hasX has no move ctor
```

總結

- 在沒有定義 control member 且所有 member 可以合法 move 時，編譯器才會生成 move operation。
- 若是 move 操作會不合法，不會像 copy 操作那樣隱式定義為 deleted function，而是直接不宣告。除非實作顯式定義為 default，才會定義為 deleted function。
- 如果 class 有定義 move operation 而沒有 copy operation，則編譯器生成的 copy operation 是 deleted function。

### 編譯器如何選擇 copy 或 move

接下來說明編譯器如何根據傳入的 argument 決定要使用 copy 還是 move，當 copy 和 move 都有定義的時候，只有在傳入 non const rvalue expression 才會使用 move。另外如果只有 copy 沒有 move ctor，即便顯式傳入 `std::move(data)`，仍然會呼叫 copy ctor。

這個選擇呼應到這篇文章一開始提到 copy initialization 可以選擇呼叫 copy 或是 move ctor，抉擇點就是看來源的 expression 適合綁定哪一個 ctor。

相關用法可以看下面的例子，這裡的 assignment 操作是透過 copy and swap 完成，參數 rhs 是 pass by value。回顧 copy initialization 的一種方式是透過 non-reference function parameter，因此編譯器會判斷傳入的值適合什麼綁定方式，進而呼叫對應的 ctor，來完成對應的 copy 或是 move assignemnt。

```cpp
class Foo {
public:
    Foo() : p(nullptr) {}
    Foo(int v) : p(new int(v)) {}
    Foo(const Foo& foo) : p(foo.p ? new int(*foo.p) : nullptr) {}
    Foo(Foo&& foo) noexcept : p(foo.p) {
        foo.p = nullptr;
    }
    
    // copy and swap
    Foo& operator=(Foo rhs) {
        swap(*this, rhs);
        return *this;
    }

    friend void swap(Foo& lhs, Foo& rhs) noexcept {
        using std::swap;
        swap(lhs.p, rhs.p);
    }
private:
    int* p;
};
```

## Rvalue reference and member function

除了 copy control member 之外，member function 也可以實作 copy 和 move 的版本，方式是透過 overload function 接收 `const T&` 或是 `T&&` 的參數。

下方是一個 vector `push_back` 的例子，

```cpp
void push_back(const X&);
void push_back(X&&);       // binds only to non-const rvalue
```

所以對一個物件而言，他可能可以以 lvalue 或是 rvalue 的身分去呼叫一個 function，例如下方的範例中 `s1 + s2` 是 rvalue，但他也可以使用 assignment operator，雖然合法但是沒有意義，因為修改 `s1 + s2` 這個暫時物件馬上就會銷毀。

```cpp
string s1 = "abc", s2 = "def";
s1 + s2 = "hello";
```

所以 c++ 提供 **reference qualifier**，這個可以是 `&` 或是 `&&`，表示 `this` 指向的物件類別，例如下方是 `&` 修飾的 function，就代表只能對左值物件呼叫這個函式。

```cpp
class Foo {
public:
    Foo& operator=(const Foo&) &;  
};

Foo& Foo::operator=(const Foo& rhs) & {
    // do something
    return *this;
}

Foo &retFoo(); // returns a reference; a call to retFoo is an lvalue
Foo retVal(); // returns by value; a call to retVal is an rvalue
Foo i, j;     // i and j are lvalues
i = j;        // ok: i is an lvalue
retFoo() = j; // ok: retFoo() returns an lvalue
retVal() = j; // error: retVal() returns an rvalue
i = retVal(); // ok: we can pass an rvalue as the right-hand operand to assignment
```
