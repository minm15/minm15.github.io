# Template

泛型程式設計（Generic Programming）是一種程式設計方法，目的是將型別抽離資料結構和演算法，讓同一份程式可以重複用在不同型別上。C++ 提供 template 機制，把型別參數化，以此在編譯期根據實際使用的型別產生對應的程式碼。

## function template

以下程式碼宣告一個名為 `func` 的函數模板（function template），這個函數模板有一個型別模板參數（type template parameter） `T`，對於某個型別 `T`，函式接收兩個型別皆為 `T` 的函式參數 a 和 b，並回傳一個型別為 `T` 的值。

```cpp
template <typename T>
T func(T a, T b) {
    // do something
}
```

以下是一個具體的範例，注意在呼叫函式時傳入的參數型別必須是一致的，如果改成 `add(3, 5.4)` 的話，就會出現編譯錯誤的提示說明編譯器在做型別推導的時候會遇到衝突。

```text
candidate template ignored: deduced conflicting types for parameter 'T' ('int' vs. 'double')
```

```cpp
template <typename T>
T add (T a, T b) {
    return a + b;
}

int main() {
    cout << add(3, 5) << endl;                                // 8
    cout << add(string("hello"), string(" world")) << endl;   // hello world
    cout << add(4.0, 3.2) << endl;                            // 7.2
    cout << add(3, 5.4) << endl;                              // compile error
}
```

若是需要不只一個型別，則額外加入 `typename type` 來定義另外的型別模板參數。

```cpp
template <typename T1, typename T2>
auto add (T1 a, T2 b) {
    return a + b;
}
```

在上面的例子中，也可以用 `add<int, double>` 指定 T1 和 T2 的型別，一個常見的例子是 class template 的 `std::vector<int>`。`<type1, type2, ..., typen>` 稱為 template argument list，其中內部元素被稱為 type template argument，被用來指定 type template parameter 的型別。

## class template

class template 一樣在一開始透過 `template <typename T>` 來定義 type template parameter，定義的型別可以在 class 內部任意使用，例如成員或是成員函式回傳的類型。

和 function template 比較大的差異是在於使用的時候需要明確指定 template argument，例如 `Box<int>`。會導致這個差異在於建構 class 物件時，若是沒有指定類型，編譯器看不到足夠的資訊來推導型別，例如 `Box b` 並不包含任何關於型別的資訊。而 `add(3, 5)` 則透過 arguments 讓編譯器知道推導的型別。

```cpp
template <typename T>
class Box {
public:
    Box(T v) : value(v) {}

    T get() {
        return value;
    }

private:
    T value;
};

int main() {
    Box<int> box1(3);
    Box<string> box2(string("hello"));
    cout << box1.get() << endl;    // 3
    cout << box2.get() << endl;    // hello 
}
```

## class template member function

class template 的 member function 在外部定義時，和一般 class 的差異是要在前面加上 `template <typename T>` 以及用 `className<T>::` 來指定 template class 的作用域。

```cpp
template <typename T>
class Box {
public:
    Box(T v);
    T get();

private:
    T value;
};

template <typename T>
Box<T>::Box(T v) : value(v) {}

template <typename T>
T Box<T>::get() {
    return value;
}

int main() {
    Box<int> box1(3);
    Box<string> box2(string("hello"));
    cout << box1.get() << endl;    // 3
    cout << box2.get() << endl;    // hello
}
```

## template specialization

上面的 class template 例子無論建構什麼型別的 Box 物件都是採用相同的實作，但是在設計上可能會遇到要針對不同型別實作的情況，這時候就會用到 template specialization。

在 template specialization 的機制下，包含一個像先前例子的泛型版本以及特化版本，泛型版本的語法不變，而針對 `type` 的型別實作的特化版本則寫成下方的形式。泛型版本稱為 primary template，特定版本則稱為 explicit specialization 或是 full specialization。

```cpp
template <>
class ClassName<type> {
    // class members
};
```

下方是一個具體的例子，展示 template specialization，可以注意到 explicit specialization 的 class 可以和 primary template 有不同的 member。

```cpp
template <typename T>
class Printer {
public:
    Printer(T v) : value(v) {}

    void print() {
        cout << "this is a primary template with data: " << value << endl;
    }

private:
    T value;
};

template <>
class Printer<string> {
public:
    Printer(string v) : value(v) {}

    void printExplicit() {
        cout << "this is an explicit template for string with data: " << value << endl;
    }

private:
    string value;
};

int main() {
    Printer<int> p1(3);
    Printer<double> p2(4.5);
    Printer<string> p3(string("hello"));
    p1.print();    // this is a primary template with data: 3
    p2.print();    // this is a primary template with data: 4.5
    p3.printExplicit();    // this is an explicit template for string with data: hello
    return 0;
}
```

## partial specialization

partial specialization 針對符合特定樣式的型別提供不同的實作。例如下方的範例提供 `T*` 這種指標類型的實作，讓`int*`、`double*` 或 `int**` 都能採用這個特化版本。

```cpp
template <typename T>
class Printer {
public:
    void print() {
        cout << "this is a primary template."  << endl;
    }
};

template <typename T>
class Printer<T*> {
public:
    void print() {
        cout << "this is a partial template for pointer" << endl;
    }
};

int main() {
    Printer<int> p1;
    Printer<double> p2;
    Printer<int*> p3;
    p1.print();
    p2.print();
    p3.print();
    return 0;
}
```

除了 pointer 之外，以下列出幾個常見的 partial specialization

- `<T&>` 針對 reference
- `<T[]>` 針對 array
- `const T` 針對 const 型別
- `<vector<T>>` 針對 vector<T> 型別

## non-type template parameter

前面例子看到的 template parameters 型別，但其實 list 內也可以放**編譯期就能決定的值**，稱為 non-type template parameter。

這種 `Array<int, 5>` 的寫法表示 class template 加上 template argument list 建立的型別，和 `Array<int, 10>` 是不同的型別。這和 `vector<int> arr(5)` 不同，這裡的 5 是 constructor argument，表示在 runtime 建構物件時傳入大小為 5 的參數，而前者是編譯期就必須決定的值。

```cpp
template <typename T, int Size>
class Array {
public:
    int size() {
        return Size;
    }

    void set_value(int index, T val) {
        data[index] = val;
    }

    T get_value(int index) {
        return data[index];
    }

private:
    T data[Size];
};

int main() {
    Array<int, 5> arr1;
    arr1.set_value(0, 12);
    cout << arr1.size() << endl;        // 5
    cout << arr1.get_value(0) << endl;  // 12

    Array<double, 10> arr2;
    arr2.set_value(3, 23.4);
    cout << arr2.size() << endl;        // 10
    cout << arr2.get_value(3) << endl;  // 23.4
    return 0;
}
```

從下方的範例可以看出 non-type template parameter 確實會影響型別。

```cpp
vector<int> v1(5);
vector<int> v2(10);
vector<double> v3(5);

cout << boolalpha;
cout << is_same_v<Array<int, 5>, Array<int, 10>> << endl; // false
cout << is_same_v<Array<int, 5>, Array<int, 5>> << endl;  // true
cout << is_same_v<decltype(v1), decltype(v2)> << endl;    // true
cout << is_same_v<decltype(v1), decltype(v3)> << endl;    // false
```

## type traits

這個技巧是 partial specialization 的延伸，type traits 能夠在編譯期查詢、判斷或轉換型別，通常會接收一或多個型別作為 arguments，會透過 value 或是 type 提供結果，value 常用於判斷是否為某個型別的結果，type 常用於表示轉換後的型別結果。

```cpp
template <typename T>
struct IsPointer {
    static constexpr bool value = false;
};

template <typename T>
struct IsPointer<T*> {
    static constexpr bool value = true;
};

int main() {
    cout << IsPointer<int>::value << endl;
    cout << IsPointer<int*>::value << endl;
    return 0;
}
```

上面的程式範例用來判斷型別是否為指標，透過 partial specialization 對 pointer 型別實作特化版本。

```cpp
template <typename T>
struct RemovePointer {
    using type = T;
};

template <typename T>
struct RemovePointer<T*> {
    using type = T;
};

template <typename T, typename S>
struct IsSameType {
    static constexpr bool value = false;
};

template <typename T>
struct IsSameType<T, T> {
    static constexpr bool value = true;
};

int main() {
    cout << IsSameType<RemovePointer<int*>::type, int*>::value << endl; // false
    cout << IsSameType<RemovePointer<int*>::type, int>::value << endl;  // true
    return 0;
}    
```

這裡的範例展示了判斷是否相同類型以及將指標移除的轉換功能。
