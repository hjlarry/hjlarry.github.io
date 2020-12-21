---
title: "面向对象编程"
draft: false
---

# 面向对象编程

面向对象编程的英文全称是Object Oriented Programming，它是一种编程范式或者说编程风格。具体来说，就是以类或对象作为组织代码的基本单元，并将封装、继承、抽象、多态这四种特性作为代码设计和实现的基石。

四大特性
-------

### 封装
封装是对信息的隐藏，类通过暴露有限的接口，授权外部只能通过类本身提供的方式来访问或者修改数据。

例如，我们要做一个钱包系统，它的基础代码可能是这样的:
{{< highlight java>}}
public class Wallet {
    private String id;
    private long createTime;
    private BigDecimal balance;
    private long balanceLastModifiedTime; 
    public Wallet() {
        this.id = IdGenerator.getInstance().generate(); 
        this.createTime = System.currentTimeMillis();
        this.balance = BigDecimal.ZERO; 
        this.balanceLastModifiedTime = System.currentTimeMillis();
    }
    public String getId() { return this.id; }
    public long getCreateTime() { return this.createTime; }
    public BigDecimal getBalance() { return this.balance; }
    public long getBalanceLastModifiedTime() { return this.balanceLastModifiedTime; }
    public void increaseBalance(BigDecimal increasedAmount) { 
        if (increasedAmount.compareTo(BigDecimal.ZERO) < 0) {
            throw new InvalidAmountException("..."); 
        }
        this.balance.add(increasedAmount);
        this.balanceLastModifiedTime = System.currentTimeMillis(); 
    }
    public void decreaseBalance(BigDecimal decreasedAmount) { 
        if (decreasedAmount.compareTo(BigDecimal.ZERO) < 0) {
            throw new InvalidAmountException("..."); 
        }
        if (decreasedAmount.compareTo(this.balance) > 0) { 
            throw new InsufficientAmountException("...");
        }
        this.balance.subtract(decreasedAmount); 
        this.balanceLastModifiedTime = System.currentTimeMillis();
    }
}
{{< /highlight >}}

对于钱包的四个属性，从业务角度讲，id和createTime应该是创建钱包的时候就确定好不会变的，所以只在构造函数中确定下来，且没有相应的方法去修改它们。而对于余额balance属性，只能增或者减，而不会重新设置，所以我们提供了increaseBalance和decreaseBalance方法。对于balanceLastModifiedTime属性，只有在余额增减时，它才会变动，所以也是绑定在余额属性修改的方法中的。

这段代码用Java实现，能很好的利用语言提供的private、public关键字来达到访问权限控制的目的。而对于Python等语言，就没办法完全意义上的控制访问权限了。

如果我们对类中的属性不加以访问控制，那么外部的所有代码都可以访问、修改这些属性。这样看起来虽然更灵活，但过度灵活的代价就是代码不可控。属性被随意修改并且隐蔽在各个代码的角落，必然会影响代码的可读性和可维护性。

### 抽象
抽象是对方法的具体实现的隐藏，让调用者只需要关心方法做了什么事情，而不需要关心是怎么做的。面向对象语言中，通常使用接口(如java中的interface)或抽象类来实现抽象特性。

实际上，抽象这个概念是很通用的设计思想，在面对复杂系统的时候，人脑能承受的信息复杂度是有限的，因此我们需要过滤掉一些非关键的实现细节，抽象的设计思想能很好的帮助我们。所以抽象有时也会被排除在面向对象的特性之外，正是因为它几乎应用在所有的编程场景下，函数本身就是一种抽象。

### 继承
继承用来表示类之间is-a的关系，分为单继承和多继承，它最大的好处就是代码复用。假设两个类有一些相同的属性或方法，我们可以考虑把这些共有的东西提取到父类中，并让这两个类继承父类。

但是，过度的使用继承也会导致代码的可读性、可维护性变差。过多的代码层级，导致查找一个属性时可能要不断的追查其父类；子类和父类的高度耦合，导致修改父类时可能影响到子类等都是问题。所以也有少用继承、多用组合的说法。

### 多态
同一操作作用于不同的对象，可以有不同的解释，产生不同的执行结果，这就是多态性。

而对于多态的实现，有多种方式，常见的有继承加方法重写、接口类、ducking-type三种。

{{< highlight java>}}
public class Animal {
    public void eat() {
        System.out.println("动物吃饭");
    }
}

public class Cat extends Animal {
    public void eat() {
        System.out.println("猫吃饭");
    }
}

public static void main(){
    Animal am = new Cat();
    am.eat();
}
{{< /highlight >}}
上面的例子中，父类对象可以引用子类对象，也就是说可以将Cat传递给Animal，然后就会使用子类的eat方法，从而实现了多态。

{{< highlight java>}}
public interface Iterator {
    String hasNext();
    String next();
}
public class Array implements Iterator {
    private String[] data;
    public String hasNext() { ... } 
    public String next() { ... } 
}
public class LinkedList implements Iterator {
    private LinkedListNode head;
    public String hasNext() { ... } 
    public String next() { ... } 
}
public class Demo {
    private static void print(Iterator iterator) {
        while (iterator.hasNext()) {
            System.out.println(iterator.next());
        }
    }
    public static void main(String[] args) {
        Iterator arrayIterator = new Array();
        print(arrayIterator);
        Iterator linkedListIterator = new LinkedList();
        print(linkedListIterator);
    }
}
{{< /highlight >}}
这个例子中，Iterator是一个接口类，定义了一个可以遍历集合数据的迭代器。Array和LinkedList都实现了它，然后通过传递不同的实现类到print函数中，来支持动态调用不同的next()和hasNext()实现。

{{< highlight python>}}
class Logger:
    def record(self):
        print("I write a log into file.")
class DB:
    def record(self):
        print("I insert data into db.")
def test(recorder):
    recorder.record()
def demo():
    logger = Logger()
    db = DB()
    test(logger)
    test(db)
{{< /highlight >}}
鸭子类型实现多态的方法非常简单，只要他们都定义了record方法，传入test方法时就可以调用相应的record方法。

多态特性主要能提升代码的可扩展性和复用性，例如Iterator这个例子，我们可以不改变原有代码的情况下，添加其他的数据结构像HashMap等，也同样能够传入print()函数。如果不使用多态特性，我们可能要定义print(Array array)、print(LinkedList linkedList)等。


SOLID
-------
随着面向对象编程的发展，人们总结出了很多设计原则，其中最出名的当属SOLID，它代表五条原则。如果遵循这5条设计原则，就更可能写出可扩展、易于修改的代码。相反，如果不断违反其中的一条或多条原则，那么很快你的代码就会变得不可扩展、难以维护。

### 单一职责
即Single Responsibility Principle，一个类或者模块只负责完成一个职责(或功能)。这个原则看似不太难用，但在实际情况中，往往很难判断一个类的职责是否是单一的。比如在一个社交产品中，我们用一个UserInfo类来记录用户的信息:
{{< highlight java>}}
public class UserInfo{
    private long userId;
    private String username;
    private String email;
    private String telephone;
    private String avatarUrl;
    private String provinceOfAddress; // 省
    private String cityOfAddress; // 市
    private String regionOfAddress; // 区
    private String detailedAddress; // 详细地址
    // ... 省略其他属性和方法...
}
{{< /highlight >}}
我们发现看上去这个类都是和用户相关的信息，所有的属性和方法都是基于用户这个业务模型的，应该是满足单一职责原则的。但还有一种观点认为，地址信息在UserInfo中所占比重高，可以继续拆分为独立的UserAddress类，拆分之后职责更加单一。哪种观点是对的呢？实际上还是要根据业务来确定，如果地址信息和其他信息一样只是展示用，那放在UserInfo中没有问题，如果地址还要用于后续的收货等场景，就应该独立成一个类。

如果违反单一职责，有什么坏处呢？假设某个类包含了10多个功能点，那意味着维护成本的增高，我们可能经常去因为各种原因修改它，导致不同的功能之间相互影响，此外，也不利于代码的复用。

那是否为了满足单一职责原则，把类拆的越细越好？例如当前有一个Serialization类实现一个协议的序列化和反序列化功能:
{{< highlight java>}}
/**
 * Protocol format: identifier-string;{gson string}
 * For example: UEUEUE;{"a":"A","b":"B"}
 */
public class Serialization {
    private static final String IDENTIFIER_STRING = "UEUEUE;";
    private Gson gson;
    public Serialization() {
        this.gson = new Gson();
    }
    public String serialize(Map<String, String> object) {
        StringBuilder textBuilder = new StringBuilder();
        textBuilder.append(IDENTIFIER_STRING);
        textBuilder.append(gson.toJson(object));
        return textBuilder.toString();
    }
    public Map<String, String> deserialize(String text) {
        if (!text.startsWith(IDENTIFIER_STRING)) {
            return Collections.emptyMap();
        }
        String gsonStr = text.substring(IDENTIFIER_STRING.length());
        return gson.fromJson(gsonStr, Map.class);
    }
}
{{< /highlight >}}
为了让类的职责更加单一，我们对它进一步拆分:
{{< highlight java>}}
public class Serializer {
    private static final String IDENTIFIER_STRING = "UEUEUE;";
    private Gson gson;
    public Serializer() { 
        this.gson = new Gson();
    }
    public String serialize(Map<String, String> object) {
        StringBuilder textBuilder = new StringBuilder();
        textBuilder.append(IDENTIFIER_STRING);
        textBuilder.append(gson.toJson(object));
        return textBuilder.toString();
    }
}

public class Deserializer {
    private static final String IDENTIFIER_STRING = "UEUEUE;";
    private Gson gson;
    public Deserializer() {
        this.gson = new Gson();
    }
    public Map<String, String> deserialize(String text) {
        if (!text.startsWith(IDENTIFIER_STRING)) {
            return Collections.emptyMap();
        }
        String gsonStr = text.substring(IDENTIFIER_STRING.length());
        return gson.fromJson(gsonStr, Map.class);
    }
}
{{< /highlight >}}
虽然拆分之后，类的职责更加单一了，但也带来新的问题，比如我们如果修改数据标识"UEUEUE"改为"DFDFDF"，或者序列化方式从JSON改为XML，那么两个类都需要做修改，代码的内聚性没有原来的高了，可维护性变差了。

### 开闭原则
即Open Closed Principle，表示对扩展开放、对修改关闭，当我们添加一个功能的时候，应该在已有代码的基础上扩展(新增模块、类、方法等)，而非修改已有代码(修改模块、类、方法等)。

比如我们在写一个爬虫的小程序，用户对爬取结果中关于github的内容感兴趣，因此代码变成了这样:
{{< highlight python>}}
class HNTopPostsSpider:
    # <... 已省略 ...>
    def fetch(self) -> Generator[Post, None, None]:
        # <... 已省略 ...>
        for item in items:
            # <... 已省略 ...>
            link = node_title.get('href')
            # 只关注来自 github.com 的内容
            if 'github' in link.lower():
                yield Post(... ...)
{{< /highlight >}}
这么做看似完成了功能，但如果用户改变了需求，想关注来自twitter的内容，就必须要修改`if 'github' in link.lower()`这段，这显然是违反开闭原则的。那么如何修改，有这样几种思路:

#### 利用继承修改
使用这种方法的关键点在于，找到父类中会变动的部分，将其抽象成新的方法或属性，最终允许子类重写来改变这些行为。
{{< highlight python>}}
class HNTopPostsSpider:
    # <... 已省略 ...>
    def fetch(self) -> Generator[Post, None, None]:
        # <... 已省略 ...>
        for item in items:
            # <... 已省略 ...>
            post = Post( ... ... )
            # 使用测试方法来判断是否返回该帖子
            if self.interested_in_post(post):
                yield post
    def interested_in_post(self, post: Post) -> bool:
        return True
class GithubOnlyHNTopPostsSpider(HNTopPostsSpider):
    def interested_in_post(self, post: Post) -> bool:
        return 'github' in post.link.lower()
{{< /highlight >}}
这时，用户如果想要添加其他网站到感兴趣的内容，只需要再写一个继承类并覆盖`interested_in_post`方法中的内容即可，而不需要修改类本身。

#### 利用组合与依赖注入修改

### 里式替换

### 接口隔离

### 依赖反转