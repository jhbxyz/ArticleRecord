# File&Database

### 1.File

**字节流** 

- InputStream->FileInputStream->BufferInputStream
- OutputStream->FileOutputStream->BufferOutputStream

**字符流**

- Reader->InputStreamReader->BufferReader->FileReader
- Writer->OutputStreamWriter->BufferWriter->FileWriter

### 2.Java随机访问文件

**RandomAccessFile**

- 随机访问文件，我们可以在文件中的任何位置读取或写入。
- 一个对象可以进行随机文件访问。我们可以读/写字节和所有原始类型的值到一个文件。
- 可以使用其readUTF()和writeUTF()方法处理字符串。
- 不在InputStream和OutputStream类的类层次结构中

### 3.Java 序列化

Serializable  使用 ObjectInputStream 和 ObjectOutputStream 进行对象的读写

#### 3.1.序列化 ID 问题

**注意：**
**虚拟机是否允许反序列化，不仅取决于类路径和功能代码是否一致，一个非常重要的一点是两个类的序列化 ID 是否一致（就是 private static final long serialVersionUID = 1L）**

#### 3.2.静态变量序列化

**注意：**序列化保存的是对象的状态，静态变量属于**类**的状态，因此 **序列化并不保存静态变量**

#### 3.3.父类的序列化与 transient 关键字

**要想将父类对象也序列化，就需要让父类也实现**Serializable 接口**。如果父类不实现的话的，就 **需要有默认的无参的构造函数。

**transient** 关键字的作用是控制变量的序列化，在**变量声明前**加上该关键字，可以**阻止**该变量被序列化到文件中，在被反序列化后，transient 变量的值被设为初始值，如 int 型的是 0，对象型的是 null。

#### 3.4.序列化存储规则

Java 序列化机制为了节省磁盘空间，具有特定的存储规则，当写入文件的为**同一对象**时，并不会再将对象的内容进行存储，而只是再次存储一份引用，上面增加的 5 字节的存储空间就是新增引用和一些控制信息的空间。反序列化时，恢复引用关系，该存储规则极大的节省了存储空间



