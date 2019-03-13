## 1. 底层数据结构——建立连接时建立的什么。
读者可先大致浏览一下`qobject_p.cpp`中添加连接的实现，回头再细看：
```
void QObjectPrivate::addConnection(int signal, Connection *c)
{
    /*
    * 中心数据结构是二维广义表
    * 不妨看作一个二维数组：
    *  A B C D E...
    * 0
    * 1
    * 2
    * 3
    * 4
    * ...
    * 纵向序号0123..为signal序号，每一个signal对应一个connectionList（无s)单链表。
    * 所有conectionList 存放在类的 connectionLists这个向量中。
    * 横向序号为接收者reciver,相应每一列为该reciver对应接收signal的connection节点（有的话才分配，顺序不一定按0123)
    * 每列组成相应纵向单链表senders（含义为该reciver的senders）。
    * 每个对象自己保存着自己的senders单链表
    * 该二维表为全局数据结构，包含以signal索引的行链表connectionLists和每个对象自己保存的相应senders链表
    * 
    */
    Q_ASSERT(c->sender == q_ptr);
    if (!connectionLists)
        connectionLists = new QObjectConnectionListVector();
    if (signal >= connectionLists->count())
        connectionLists->resize(signal + 1);

    ConnectionList &connectionList = (*connectionLists)[signal];
    if (connectionList.last) {
        connectionList.last->nextConnectionList = c;
    } else {
        connectionList.first = c;
    }
    connectionList.last = c;

    cleanConnectionLists();
    /*
    * 插入节点的技巧：
    * prev为二级指针，指向本对象的senders列表头的地址addr(addr中保存的是表头节点的地址)
    * senders表本身为一个单链表。
    * 但链表中每一个节点的prev都指向表头地址addr，即访问某一个节点，也可以继续访问整个senders表。
    * 更新senders表时，直接将c插到原表头，原表头变为c的next。addr内容变为c的地址，但addr本身地址不变。
    */
    c->prev = &(QObjectPrivate::get(c->receiver)->senders);
    c->next = *c->prev;
    *c->prev = c;
    if (c->next)
        c->next->prev = &c->next;

    if (signal < 0) {
        connectedSignals[0] = connectedSignals[1] = ~0;
    } else if (signal < (int)sizeof(connectedSignals) * 8) {
        connectedSignals[signal >> 5] |= (1 << (signal & 0x1f));
    }
}
```
先看看传入参数：
传入参数`signal`就是所谓的信号，是一个整数。
Connection 类的定义，在`qobject_p.h`中。
它是在QObjectPrivate类中定义的一个结构体：
```
struct Connection
    {
        QObject *sender;
        QObject *receiver;
        union {
            StaticMetaCallFunction callFunction;
            QtPrivate::QSlotObjectBase *slotObj;
        };
        // The next pointer for the singly-linked ConnectionList
        Connection *nextConnectionList;
        //senders linked list
        Connection *next;
        Connection **prev;
        QAtomicPointer<const int> argumentTypes;
        QAtomicInt ref_;
        ushort method_offset;
        ushort method_relative;
        uint signal_index : 27; // In signal range (see QObjectPrivate::signalIndex())
        ushort connectionType : 3; // 0 == auto, 1 == direct, 2 == queued, 4 == blocking
        ushort isSlotObject : 1;
        ushort ownArgumentTypes : 1;
        Connection() : nextConnectionList(nullptr), ref_(2), ownArgumentTypes(true) {
            //ref_ is 2 for the use in the internal lists, and for the use in QMetaObject::Connection
        }
        ~Connection();
        int method() const { Q_ASSERT(!isSlotObject); return method_offset + method_relative; }
        void ref() { ref_.ref(); }
        void deref() {
            if (!ref_.deref()) {
                Q_ASSERT(!receiver);
                delete this;
            }
        }
    };
```
可以看到，与链表相关的有三个指针，其中`prev`是个二级指针。为什么这么用，在`addConnection`函数我写的注释里可以看到一些解释。其他的结构体参数在这里不作深究，有些也可以见名知意。

采用二维表的目的有二：
- 能根据发出的信号找到所有接受者
- 能将某一个接受者接收的所有信号统一管理

采用链表的目的，自然就是方便动态管理了。

既然知道了建立的是什么数据结构，那么下一步就是理清是怎么建立的。
另外，我们都知道Qt中基本上所有的类都继承于QObject这个类，这个类包含些什么？为什么连接的数据结构是存在QObjectPrivate这个类中？QObjectPrivate和QObject是什么关系？为什么要这样定义？这是我们最后要讨论的问题。

## 2. 从信号和槽的定义到整数的转换
### （1） 神奇的关键字
再看信号和槽的基本定义和使用方式。
一个简单例子：
定义一个 Counter类
```
class Counter : public QObject
{
    Q_OBJECT
    int m_value;
public:
    int value() const { return m_value; }
public slots: //使用public slots声明槽函数
    void setValue(int value);
signals: //使用signal声明信号，也是函数
    void valueChanged(int newValue);
};
```
信号的发射方式。在某处发射信号的写法：
```
void Counter::setValue(int value)
{
    if (value != m_value) {
        m_value = value;
        //使用emit关键字发射信号，附带参数。
        emit valueChanged(value); 
    }
}
```
有了信号和槽，还需要将它们建立连接关系：
```
Counter a, b;
  QObject::connect(&a, SIGNAL(valueChanged(int)),
                   &b, SLOT(setValue(int)));
```
或
```
//可省略QObject::默认的是同一个函数
// 信号和对应的槽，参数要一致
connect(&a, &a->valueChanged,&b, b->setValue);
```
首先来看这几个Qt中特有的关键字的含义：slots,signals,emit
在`no-keywords.h`中：
```
#define signals Q_SIGNALS
#define slots Q_SLOTS
#define emit Q_EMIT
```
具体地：
```
# define Q_SLOTS QT_ANNOTATE_ACCESS_SPECIFIER(qt_slot)
# define Q_SIGNALS public QT_ANNOTATE_ACCESS_SPECIFIER(qt_signal)   //主要有个public
//进一步可见：
# define QT_ANNOTATE_ACCESS_SPECIFIER(x)

#define Q_EMIT  //（空）
```
简单来说，emit就只是个写程序用到的逻辑关键字，起提示作用。
而signals除了public关键字声明为公有，其他部分就只在预处理过程中起作用。
slots就是只有预处理作用。

由此可见，定义的信号和槽函数， 最终变成了整数，而定义的特殊性，仅在于多一个预处理特征（包括`#define xx`本身）。那么可以推测，是使用某种预处理机制，进行了转化。那么这是什么机制？如何运作的？
### （2） Qt 的MOC（the Meta Object Compiler）预处理器

[文档](https://woboq.com/blog/how-qt-signals-slots-work.html)翻译时间：
> Qt信号/槽和属性系统基于在运行时内省对象的能力。内省意味着能够列出对象的方法和属性，并具有关于它们的各种信息，例如它们的参数类型。
C ++本身不提供内省支持，因此Qt附带了一个提供它的工具。该工具是MOC。它是一个代码生成器（而不是像某些人所说的预处理器）。它解析头文件并生成一个额外的C ++文件，该文件与程序的其余部分一起编译。生成的C ++文件包含内省所需的所有信息。

引用说那不是预处理，但是我仍然认为，既然需要用到了宏定义，先对代码进行一遍处理（生成相关代码也是一种处理），仍然可以看作是预处理，只不过是全部预处理的一部分。

再来看每个基于QObject类都需要声明的一个宏Q_OBJECT：
```
#define Q_OBJECT \
public: \
    static const QMetaObject staticMetaObject; \
    virtual const QMetaObject *metaObject() const; \
    virtual void *qt_metacast(const char *); \
    virtual int qt_metacall(QMetaObject::Call, int, void **); \
    QT_TR_FUNCTIONS /* translations helper */ \
private: \
    Q_DECL_HIDDEN static void qt_static_metacall(QObject *, QMetaObject::Call, int, void **);
```
> Q_OBJECT定义了一堆函数和一个静态QMetaObject这些函数在MOC生成的文件中实现。

简单说，就是MOC根据代码的关键字，自动提取出信号和槽，并进行处理，生成了相应的cpp文件，相关要使用的函数即由Q_OBJECT定义，也生成在相应cpp文件中，随整个工程一同进行编译链接。

对于老版本的信号和槽：
```
Q_CORE_EXPORT const char *qFlagLocation(const char *method);
#ifndef QT_NO_DEBUG
# define QLOCATION "\0" __FILE__ ":" QTOSTRING(__LINE__)
# define SLOT(a)     qFlagLocation("1"#a QLOCATION)
# define SIGNAL(a)   qFlagLocation("2"#a QLOCATION)
#else
# define SLOT(a)     "1"#a
# define SIGNAL(a)   "2"#a
#endif
```
> 这些宏只是使用预处理器将参数转换为字符串，并在前面添加代码。在debug模式下，如果信号连接不起作用，我们还会使用文件位置注释字符串以显示警告消息。这是以兼容的方式在Qt 4.5中添加的。为了知道哪些字符串具有行信息，我们使用qFlagLocation，它将在具有两个条目的表中注册字符串地址。

也即直接将函数名翻译成字符串，在前面加1或2字符，所谓的信号和槽，也就是相应的字符串。另外只需要处理参数传递问题。

新版用法中，传递的也是函数名，但从方式上看并没有转换成字符串，而是直接传递函数指针。总之，就是两种区分名称的方法而以。函数指针方式能方便编辑器和编译器检查语法错误，使用更安全。

那么，MOC是如何识别宏定义的？生成的cpp文件包含什么具体内容呢？ 
（未完）

