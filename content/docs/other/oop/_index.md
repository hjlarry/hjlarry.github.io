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

### 多态