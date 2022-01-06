# Python/C++ 接口

参考：[Python 接口](https://developers.google.cn/protocol-buffers/docs/pythontutorial) 与 [C++ 接口](https://developers.google.cn/protocol-buffers/docs/cpptutorial)。

本教程为 Python/C++ 程序员提供了关于使用协议缓冲区的基本介绍。通过创建一个简单的示例程序，它告诉你如何

- 在一个 `.proto` 文件中定义消息格式。
- 使用协议缓冲区编译器。
- 使用 Python 协议缓冲区 API 来写和读消息。

这并不是一份在 Python 中使用协议缓冲区的全面指南。对于更详细的参考信息，请参见 [协议缓冲区语言指南（proto2）](https://developers.google.cn/protocol-buffers/docs/proto)、[协议缓冲区语言指南（proto3）](https://developers.google.cn/protocol-buffers/docs/proto3)、[Python API 参考](https://googleapis.dev/python/protobuf/latest/)、[Python 生成的代码指南](https://developers.google.cn/protocol-buffers/docs/reference/python-generated) 和 [编码参考](https://developers.google.cn/protocol-buffers/docs/encoding)。

## 为什么使用协议缓冲区？

我们将使用的例子是一个非常简单的 "地址簿" 应用程序，它可以从一个文件中读写人们的联系信息。地址簿中的每个人都有一个名字、一个 ID、一个电子邮件地址和一个联系电话。

你如何序列化和检索这样的结构化数据？有几种方法可以解决这个问题。

- （Python）使用 Python pickling。这是默认的方法，因为它是内置于语言中的，但是它不能很好地处理模式的演变，而且如果你需要与用 C++ 或 Java 编写的应用程序共享数据，也不能很好地工作。
- （C++）原始的内存数据结构可以以二进制形式发送/保存。随着时间的推移，这是一个脆弱的方法，因为接收/读取代码必须以完全相同的内存布局、编码方式等进行编译。另外，随着文件积累了原始格式的数据，以及为该格式布线的软件副本四处传播，要扩展该格式是非常困难的。
- 你可以发明一种特别的方式将数据项编码成一个字符串——比如将 4 个整数编码为 "12:3:-23:67"。这是一种简单而灵活的方法，尽管它确实需要编写一次性的编码和解析代码，而且解析会带来少量的运行时间成本。这对编码非常简单的数据来说效果最好。
- 将数据序列化为 XML。这种方法可能非常有吸引力，因为 XML 是（某种程度上）人类可读的，而且有很多语言的绑定库。如果你想与其他应用程序/项目分享数据，这可能是一个不错的选择。然而，XML 是出了名的空间密集型，对它进行编码/解码会给应用程序带来巨大的性能损失。另外，浏览 XML DOM 树比浏览一个类中的简单字段要复杂得多。

协议缓冲区正是解决这一问题的灵活、高效、自动化的解决方案。使用协议缓冲区，你可以写一个你希望存储的数据结构的 `.proto` 描述。由此，协议缓冲区编译器创建了一个类，实现了自动编码和解析协议缓冲区数据的高效二进制格式。生成的类为构成协议缓冲区的字段提供了获取器和设置器，并负责处理作为一个单元的协议缓冲区的读写细节。重要的是，协议缓冲区格式支持随着时间的推移扩展格式的想法，这样代码仍然可以读取用旧格式编码的数据。

## 哪里可以找到示例代码

示例代码包含在源代码包的 `examples/` 目录下。在[这里下载](https://developers.google.cn/protocol-buffers/docs/downloads)。

## 定义协议格式

要创建你的地址簿应用程序，你需要从一个 `.proto` 文件开始。`.proto` 文件中的定义很简单：在需要序列化的每个数据结构添加一个 *message*，然后为消息中的每个字段指定一个名称和一个类型。这里是定义你的消息的 `.proto` 文件（`addressbook.proto`）：

```proto
// 注意：START 和 END 标签在注释中用来定义教程中使用的部分。
// 它们不是协议缓冲区语法的一部分。
//
// 要深入了解这个文件和相关的例子，请看
// https://developers.google.com/protocol-buffers/docs/tutorials

// [START declaration]
syntax = "proto3";
package tutorial;

import "google/protobuf/timestamp.proto";
// [END declaration]

// [START messages]
message Person {
  string name = 1;
  int32 id = 2;  // Unique ID number for this person.
  string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    string number = 1;
    PhoneType type = 2;
  }

  repeated PhoneNumber phones = 4;

  google.protobuf.Timestamp last_updated = 5;
}

// Our address book file is just one of these.
message AddressBook {
  repeated Person people = 1;
}
// [END messages]
```

让我们浏览一下文件的每个部分，看看它有什么作用。

`.proto` 文件以 `package` 声明开始，这有助于防止不同项目之间的命名冲突。

```{tab} Python
在 Python 中，包通常由目录结构决定，所以你在 `.proto` 文件中定义的 `package` 对生成的代码没有影响。然而，你仍然应该声明一个包，以避免在协议缓冲区的名称空间以及非 Python 语言中出现名称冲突。
```

```{tab} C++
在 C++ 中，你生成的类将被放在与包名匹配的命名空间中。
```

接下来，是你的消息（`message`）定义。一个消息只是一个包含一组类型字段的集合。许多标准的简单数据类型可以作为字段类型使用，包括 `bool`、`int32`、`float`、`double` 和 `string`。你也可以通过使用其他消息类型作为字段类型来为你的消息添加进一步的结构——在上面的例子中，`Person` 消息包含 `PhoneNumber` 消息，而 `AddressBook` 消息包含 `Person` 消息。你甚至可以定义嵌套在其他消息中的消息类型——如你所见，`PhoneNumber` 类型被定义在 `Person` 中。如果你想让你的一个字段有一个预定义的值列表，你也可以定义 `enum` 类型——这里你想指定一个电话号码可以是下列电话类型之一：`MOBILE`、`HOME` 或 ` WORK`。

每个元素上的 `=1`、`=2` 标记标识了该字段在二进制编码中使用的唯一 "标签"。标签数字 1-15 或者更高的数字需要少一个字节的编码，所以作为一种优化，你可以决定将这些 1-15 标签用于常用的或 `repeated` 的元素，而将标签 16 和更高的数字留给不太常用的可选元素。`repeated` 字段中的每个元素都需要重新编码标签号，所以 `repeated` 字段是这种优化的特别好的候选者。

每个字段都必须用以下修饰语之一进行注解：

- `optional`：该字段可以设置也可以不设置。如果一个可选字段的值没有设置，就会使用一个默认值。调用访问器（accessor）来获取一个没有明确设置的可选（或 `required`）字段的值，总是返回该字段的默认值。
- `repeated`：该字段可以重复任何次数（包括零）。`repeated` 值的顺序将在协议缓冲区中被保留下来。可以把 `repeated` 字段看作是动态大小的数组。

你可以在 [协议缓冲区语言指南](https://developers.google.cn/protocol-buffers/docs/proto) 中找到编写 `.proto` 文件的完整指南——包括所有可能的字段类型。不过，不要去寻找类似于类继承的设施——协议缓冲区不做这个。

## 编译协议缓冲区

现在有了一个 `.proto`，你需要做的下一件事是生成你需要用来读写 `AddressBook` （以及 `Person` 和 `PhoneNumber`）信息的类。要做到这一点，你需要在你的 `.proto` 上运行协议缓冲区编译器 `protoc`。

- 如果你还没有安装编译器，请 [下载该软件包](https://developers.google.cn/protocol-buffers/docs/downloads) 并按照 [README](https://daobook.github.io/protobuf/src/README.html) 中的说明进行操作。
- 现在运行编译器，指定源目录（你的应用程序的源代码所在的地方——如果你不提供一个值，就使用当前目录），目标目录（你想让生成的代码去哪里；通常与 `$SRC_DIR` 相同），以及你的 `.proto` 的路径。在这种情况下：

  ````{tab} Python
  ```sh
  protoc -I=$SRC_DIR --python_out=$DST_DIR $SRC_DIR/addressbook.proto
  ```

  因为你想要 Python 类，所以你使用 `--python_out` 选项——类似的选项也提供给其他支持的语言。

  这将在你指定的目标目录下生成 `addressbook_pb2.py`。
  ````

  ````{tab} C++
  ```sh
  protoc -I=$SRC_DIR --cpp_out=$DST_DIR $SRC_DIR/addressbook.proto
  ```

  因为你想要 C++ 类，所以你使用 `--cpp_out` 选项——类似的选项也提供给其他支持的语言。

  这将在你指定的目标目录下生成：

  - `addressbook.pb.h`：声明你生成的类的头文件。
  - `addressbook.pb.cc`：它包含了你的类的实现。
  ````

  

## Protocol Buffer API

````{tab} Python
与生成 Java 和 C++ 协议缓冲区代码时不同，Python 协议缓冲区编译器不会直接为你生成数据访问代码。相反（如果你看一下 `addressbook_pb2.py` 就会发现）它为你所有的消息、枚举和字段生成特殊的描述符，还有一些神秘的空类，每种消息类型都有一个。

```python
_PERSON = DESCRIPTOR.message_types_by_name['Person']
_PERSON_PHONENUMBER = _PERSON.nested_types_by_name['PhoneNumber']
_ADDRESSBOOK = DESCRIPTOR.message_types_by_name['AddressBook']
_PERSON_PHONETYPE = _PERSON.enum_types_by_name['PhoneType']
Person = _reflection.GeneratedProtocolMessageType('Person', (_message.Message,), {

    'PhoneNumber': _reflection.GeneratedProtocolMessageType('PhoneNumber', (_message.Message,), {
        'DESCRIPTOR': _PERSON_PHONENUMBER,
        '__module__': 'addressbook_pb2'
        # @@protoc_insertion_point(class_scope:tutorial.Person.PhoneNumber)
    }),
    'DESCRIPTOR': _PERSON,
    '__module__': 'addressbook_pb2'
    # @@protoc_insertion_point(class_scope:tutorial.Person)
})
_sym_db.RegisterMessage(Person)
_sym_db.RegisterMessage(Person.PhoneNumber)

AddressBook = _reflection.GeneratedProtocolMessageType('AddressBook', (_message.Message,), {
    'DESCRIPTOR': _ADDRESSBOOK,
    '__module__': 'addressbook_pb2'
    # @@protoc_insertion_point(class_scope:tutorial.AddressBook)
})
_sym_db.RegisterMessage(AddressBook)
```

每个类中重要的一行是 `__metaclass__ = reflection.GeneratedProtocolMessageType`。虽然 Python 元类如何工作的细节超出了本教程的范围，但你可以把它们看成是创建类的模板。在加载时，`GeneratedProtocolMessageType` 元类使用指定的描述符来创建所有你需要的 Python 方法来处理每个消息类型，并将它们添加到相关的类中。然后你可以在你的代码中使用完全填充的类（fully-populated classes）。

所有这些的最终效果是，你可以使用 `Person` 类，就像它把 `Message` 基类的每个字段定义为一个普通字段一样。例如，你可以这样写：

```python
import addressbook_pb2
person = addressbook_pb2.Person()
person.id = 1234
person.name = "John Doe"
person.email = "jdoe@example.com"
phone = person.phones.add()
phone.number = "555-4321"
phone.type = addressbook_pb2.Person.HOME
```

注意，这些赋值并不只是向一个通用的 Python 对象添加任意的新字段。如果你试图赋值一个没有在 `.proto` 文件中定义的字段，将会产生一个 `AttributeError`。如果你把一个字段分配给一个错误的类型的值，将会产生一个 `TypeError`。另外，在一个字段被设置之前读取它的值会返回默认值。

```python
person.no_such_field = 1  # raises AttributeError
person.id = "1234"        # raises TypeError
```

关于协议编译器为任何特定字段定义生成的具体成员的更多信息，请参阅 [Python 生成的代码参考](python-generated)。
````

````{tab} C++
让我们看一下生成的一些代码，看看编译器为你创建了哪些类和函数。如果你看一下 `addressbook.pb.h`，你可以看到你在 `addressbook.proto` 中指定的每个消息都有一个类。仔细看一下 `Person` 类，你可以看到编译器已经为每个字段生成了访问器。例如，对于 `name`、`id`、`email` 和 `phones` 字段，你有这些方法：

```cpp
  // name
  inline bool has_name() const;
  inline void clear_name();
  inline const ::std::string& name() const;
  inline void set_name(const ::std::string& value);
  inline void set_name(const char* value);
  inline ::std::string* mutable_name();

  // id
  inline bool has_id() const;
  inline void clear_id();
  inline int32_t id() const;
  inline void set_id(int32_t value);

  // email
  inline bool has_email() const;
  inline void clear_email();
  inline const ::std::string& email() const;
  inline void set_email(const ::std::string& value);
  inline void set_email(const char* value);
  inline ::std::string* mutable_email();

  // phones
  inline int phones_size() const;
  inline void clear_phones();
  inline const ::google::protobuf::RepeatedPtrField< ::tutorial::Person_PhoneNumber >& phones() const;
  inline ::google::protobuf::RepeatedPtrField< ::tutorial::Person_PhoneNumber >* mutable_phones();
  inline const ::tutorial::Person_PhoneNumber& phones(int index) const;
  inline ::tutorial::Person_PhoneNumber* mutable_phones(int index);
  inline ::tutorial::Person_PhoneNumber* add_phones();
```

正如你所看到的，getters 的名字和字段的名字完全一样，都是小写的，setter 方法以 `set_` 开头。每个单数（可选）字段也有 `has_` 方法，如果该字段已被设置，则返回 `true`。最后，每个字段都有一个 `clear_` 方法，可以将字段解设置为空状态。
````

### 枚举

```{tab} Python
Enums 被元类扩展为一组具有整数值的符号常数。因此，例如，常量 `addressbook_pb2.Person.PhoneType.WORK` 的值为 `2`。
```

```{tab} C++
生成的代码包括一个 PhoneType 枚举，与你的 `.proto` 枚举相对应。你可以把这个类型称为 `Person::PhoneType`，把它的值称为 `Person::MOBILE`、`Person::HOME` 和 `Person::WORK` （实现细节有点复杂，但你不需要理解它们来使用这个枚举）。

编译器还为你生成了一个名为 `Person::PhoneNumber` 的嵌套类。如果你看一下代码，你可以看到 "真正的" 类实际上叫做 `Person_PhoneNumber`，但是在 `Person` 内部定义的类型化允许你把它当作一个嵌套类。唯一有区别的情况是你想在另一个文件中前置声明这个类——在 C++ 中你不能前置声明嵌套类型，但你可以前置声明 `Person_PhoneNumber`。
```

### 标准信息方法

````{tab} Python
每个消息类还包含一些其他方法，让你检查或操作整个消息，包括：

- `IsInitialized()`：检查是否所有需要的字段都已设置。
- `__str__()`：返回一个人类可读的消息表示，对于调试特别有用。（通常作为 `str(message)` 或 `print(message)` 来调用。）
- `CopyFrom(other_msg)`：用给定的消息的值覆盖消息。
- `Clear()`：将所有的元素清除到空状态。

这些方法实现了 `Message` 接口。欲了解更多信息，请参见 {class}`Message 接口 <google.protobuf.message.Message>`。
````

### 解析和序列化

最后，每个协议缓冲区类都有方法用于使用协议缓冲区的 [二进制格式](https://developers.google.cn/protocol-buffers/docs/encoding) 写入和读取你选择的信息。这些方法包括：

- `SerializeToString()`：将消息序列化，并将其作为字符串返回。注意，字节是二进制的，而不是文本；我们只使用 `str` 类型作为一个方便的容器。
- `ParseFromString(data)`：从给定的字符串中解析出一条信息。

这些只是为解析和序列化提供的几个选项。同样，完整的清单请参见 {class}`Message 接口 <google.protobuf.message.Message>`。

```{admonition} 协议缓冲区和面向对象的设计
:class: warning

协议缓冲区类基本上是 dumb 数据持有者（像 C 语言中的结构）；它们在对象模型中不是很好的一等公民。如果你想在生成的类中添加更丰富的行为，最好的方法是将生成的协议缓冲器类包裹在一个特定的应用类中。如果你不能控制 `.proto` 文件的设计，包裹协议缓冲区也是一个好主意（比如，你从另一个项目中重复使用）。在这种情况下，你可以使用封装类来制作一个更适合你的应用程序的独特环境的接口：隐藏一些数据和方法，公开方便的函数，等等。你不应该通过继承来给生成的类添加行为。这将破坏内部机制，而且无论如何都不是面向对象的好做法。
```

## 编写 `Message`

现在让我们试着使用你的协议缓冲器类。你希望你的地址簿应用程序能够做的第一件事是将个人信息写入你的地址簿文件中。要做到这一点，你需要创建并填充你的协议缓冲区类的实例，然后将它们写入输出流中。

这里有一个程序，它从文件中读取一个 `AddressBook`，根据用户的输入添加一个新的 `Person`，并将新的 `AddressBook` 再次写回文件。直接调用或引用由协议编译器生成的代码的部分被突出显示。
````{tab} Python
```python
import addressbook_pb2
import sys


def PromptForAddress(person):
    '''该函数根据用户输入的信息填写 Person 信息
    '''
    person.id = int(input("输入 person ID 数字："))
    person.name = input("输入名字：")

    email = input("输入邮箱地址（空白设置为 none）：")
    if email != "":
        person.email = email

    while True:
        number = input("输入手机号码（可为空，用于终止输入）")
        if number == "":
            break

        phone_number = person.phones.add()
        phone_number.number = number

        type = input("这是个手机，家庭，还是工作电话？")
        if type == "手机":
            phone_number.type = addressbook_pb2.Person.MOBILE
        elif type == "家庭":
            phone_number.type = addressbook_pb2.Person.HOME
        elif type == "工作":
            phone_number.type = addressbook_pb2.Person.WORK
        else:
            print("未知电话类型；留作默认值。")


# 主程序。
# 从一个文件中读取整个地址簿，根据用户的输入添加一个人，然后把它写回同一个文件。
if len(sys.argv) != 2:
    print("Usage:", sys.argv[0], "ADDRESS_BOOK_FILE")
    sys.exit(-1)

address_book = addressbook_pb2.AddressBook()

# 读取现有的地址簿
try:
    with open(sys.argv[1], "rb") as f:
        address_book.ParseFromString(f.read())
except IOError:
    print(sys.argv[1] + ": File not found.  Creating a new file.")

# 添加地址
PromptForAddress(address_book.people.add())

# 将新的地址簿写回磁盘。
with open(sys.argv[1], "wb") as f:
    f.write(address_book.SerializeToString())
```
````

## 阅读 `Message`

当然，如果你不能从地址簿中得到任何信息，那么地址簿就没有什么用处了！这个例子读取了上面例子创建的文件，并打印了其中的所有信息。

````{tab} Python
```python
import addressbook_pb2
import sys


# Iterates though all people in the AddressBook and prints info about them.
def ListPeople(address_book):
  for person in address_book.people:
    print("Person ID:", person.id)
    print("  Name:", person.name)
    if person.email != "":
      print("  E-mail address:", person.email)

    for phone_number in person.phones:
      if phone_number.type == addressbook_pb2.Person.MOBILE:
        print("  Mobile phone #:", end=" ")
      elif phone_number.type == addressbook_pb2.Person.HOME:
        print("  Home phone #:", end=" ")
      elif phone_number.type == addressbook_pb2.Person.WORK:
        print("  Work phone #:", end=" ")
      print(phone_number.number)


# Main procedure:  Reads the entire address book from a file and prints all
#   the information inside.
if len(sys.argv) != 2:
  print("Usage:", sys.argv[0], "ADDRESS_BOOK_FILE")
  sys.exit(-1)

address_book = addressbook_pb2.AddressBook()

# Read the existing address book.
with open(sys.argv[1], "rb") as f:
  address_book.ParseFromString(f.read())

ListPeople(address_book)
```
````

## 扩展协议缓冲区（proto2）

在发布了使用协议缓冲区的代码后，迟早你会毫无疑问地想要 "改进" 协议缓冲区的定义。如果你希望你的新缓冲区是向后兼容的，而你的旧缓冲区是向前兼容的——你几乎肯定希望如此——那么有一些规则你需要遵循。在新版本的协议缓冲区中：

- 你不能改变任何现有字段的标签号。
- 你不能添加或删除任何 `required` 字段。
- 你可以删除 `optional` 或 `repeated` 的字段。
- 你可以添加新的 `optional` 或 `repeated` 的字段，但你必须使用新的标签号（即在这个协议缓冲区中从未使用过的标签号，甚至被删除的字段也没有）。

（这些规则有一些[例外](https://developers.google.cn/protocol-buffers/docs/proto#updating)，但很少使用）。

如果你遵循这些规则，旧的代码会很高兴地读取新的信息，并简单地忽略任何新字段。对旧代码来说，被删除的可选字段将仅仅具有其默认值，而被删除的重复字段将为空。新的代码也会透明地读取旧的消息。然而，请记住，新的可选字段不会出现在旧消息中，所以你需要明确地检查它们是否用 `has_` 设置，或者在你的 `.proto` 文件中用标签号后面的 `[default = value]` 提供一个合理的默认值。如果没有为一个可选元素指定默认值，就会使用一个特定类型的默认值：对于字符串，默认值是空字符串。对于布尔运算，默认值是 `false`。对于数字类型，默认值是 `0`。还要注意的是，如果你添加了一个新的重复字段，你的新代码将无法判断它是留空（由新代码）还是根本没有设置（由旧代码），因为它没有 `has_` 标志。

## 高级用法

协议缓冲区的用途超出了简单的访问器和序列化。一定要探索 [Python API 参考](https://googleapis.dev/python/protobuf/latest/)，看看你还能用它们做什么。

协议消息类提供的一个关键功能是反射（reflection）。你可以遍历一个消息的字段并操作它们的值，而不需要针对任何特定的消息类型编写代码。使用反射的一个非常有用的方法是将协议消息转换为其他编码，如 XML 或 JSON。反射的一个更高级的用途可能是寻找同一类型的两个消息之间的差异，或者开发一种 "协议消息的正则表达式"，在其中你可以编写与某些消息内容相匹配的表达式。如果你发挥你的想象力，就有可能将协议缓冲区应用到比你最初预期的更广泛的问题上！

反射是作为 {class}`Message 接口 <google.protobuf.message.Message>` 的一部分提供的。
