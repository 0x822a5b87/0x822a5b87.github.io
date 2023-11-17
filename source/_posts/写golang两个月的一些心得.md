# 写golang两个月的一些心得

> 自从开始负责公司后台的SQL引擎优化之后，时常需要为公司其他同事提供一些SQL解析接口。
>
> 由于之前部门大部分的接口开发都使用 cpp/golang，而我这边的接口服务主要性能瓶颈都在后台的SQL解析引擎，接口所耗费的资源在整体资源中占比非常小。cpp性能虽然更快，但是在这里并不是我需要考虑的主要因素，很自然的，我选择了golang作为我的后台接口开发语言。
>
> 这里就对我这两个月的一些开发心得做一些总结。

## referecnce

- [SOLID](https://en.wikipedia.org/wiki/SOLID)
- [Is this a violation of the Liskov Substitution Principle?](https://softwareengineering.stackexchange.com/questions/170138/is-this-a-violation-of-the-liskov-substitution-principle)

## golang实现SOLID原则

> SOLID 并不算是一个面向golang的原则，而是面向所有**面向对象语言**都可以参考的原则。
>
> 虽然从严格的角度来讲，golang没有**显示继承**，但是golang的接口可以隐式继承，也可以通过struct持有另外一个匿名struct对象来实现隐式继承。
>
> 所以SOLID原则对golang来讲仍然是适用的。

SOLID分别对应于：

1. The Single-responsibility principle
2. The Open–closed principle
3. The Liskov substitution principle
4. The Interface segregation principle
5. The Dependency inversion principle

### 单一职责原则（The Single-responsibility principle）

> A module should be responsible to one, and only one,`actor`. The term `actor` refers to a group (consisting of one or more stakeholders or users) that requires a change in the module.

对于**单一职责原则**，有一个比较难理解的术语是 `actor`。按照网络上大部分博主的说法，我们可以简单的理解为**会引起变化的原因**。

我们可以用下面的代码来描述，假设我们存在这样一个struct

```go
// 定义 Trade 结构体
type Trade struct {
    TradeID int
    Symbol string
    Quantity float64
    Price float64
}
```

我们可以声明两个结构体 `TradeRepository` 和 `TradeValidator`分别用于存储Trade，验证Trade是否合法。

```golang
// 定义 TradeRepository 结构体
type TradeRepository struct {}

// Save 方法用于将 Trade 对象保存到数据库中
func (tr *TradeRepository) Save(trade *Trade) error {
  // ...
}

// 定义 TradeValidator 结构体
type TradeValidator struct {}

// Validate 方法用于验证 Trade 对象的有效性
func (tv *TradeValidator) Validate(trade *Trade) error {
	// validate trade
}
```

从代码功能的角度讲，我们也可以这样定义，把Save和Validate都放在一个类中。这样带来的最大问题是，当存储逻辑或者验证逻辑要修改时，都会需要修改 TradeUtil，这违背了单一职责原则。

```go
type TradeUtil struct {}

// Save 方法用于将 Trade 对象保存到数据库中
func (u *TradeUtil) Save(trade *Trade) error {
  // ...
}

// Validate 方法用于验证 Trade 对象的有效性
func (u *TradeUtil) Validate(trade *Trade) error {
	// validate trade
}
```

但是，我们可以进一步的思考，如果按照这样的单一原则，那是不是我每个struct都只能绑定一个方法呢？比如我现在拥有

1. Save
2. Update
3. Validate

三个方法，难道我需要声明三个struct并为他们绑定各自的方法吗？此时我们可以将逻辑拆分为：

1. 第一个struct负责数据库操作
2. 第二个struct负责Trade验证

```go
type TradeDb struct {}

// Save 方法用于将 Trade 对象保存到数据库中
func (u *TradeDb) Save(trade *Trade) error {
  // ...
}

// Update 更新
func (u *TradeDb) Update(trade *Trade) error {
  // ...
}

// 定义 TradeValidator 结构体
type TradeValidator struct {}

// Validate 方法用于验证 Trade 对象的有效性
func (tv *TradeValidator) Validate(trade *Trade) error {
	// validate trade
}
```

> 而具体的拆分粒度需要根据应用场景来设计，毕竟拆或者不拆都会引入一些问题：
>
> 1. 拆分粒度过细会导致出现大量的struct，并且在这种情况下，绑定方法到struct完全丧失了意义，我们甚至可以声明一个全局的方法，而不是将方法绑定到struct。
> 2. 拆分粒度过粗会导致任何外部的需求变更都会有大量的接口/方法可能受到影响。

### 开闭原则

> software entities (classes, modules, functions, etc.) should be open for extension, but closed for modification

开闭原则其实可以简单的描述为，**在需求可能会发生变更的地方，尽可能的使用接口，继承（虽然golang只有隐式继承）来实现**。例如，我们现在有一个如下的交易流程：

1. 验证Trade
2. 交易Trade
3. 存储Trade

而交易流程中，这个Trade的交易可能是买入或者卖出。如果我们不遵循开闭原则，那么我们的代码实现最开始可能是这样的：我们只需要实现买

```go
// 定义 TradeRepository 结构体
type TradeDb struct {}

func (u *TradeDb) Save(trade *Trade) error {
  // ...
}

func DoTrade(td *Trade){
  td.Validate()
  td.Buy()
  td.Save()
}
```

随后发现需要新增一个卖

```go
// 定义 TradeRepository 结构体
type TradeDb struct {}

func (u *TradeDb) Save(trade *Trade) error {
  // ...
}

func DoTrade(td *Trade, tp Type){
  td.Validate()
  if tp == Buy {
    td.Buy()
  } else if tp == Sale {
    td.Sale()
  }
  td.Save()
}
```

这样明显违背了开闭原则，因为我们的接口修改了，如果我们遵循开闭原则实现则可能是这样的：

```go
// TradeProcessor 交易接口
type TradeProcessor interface {
    Process(trade *Trade) error
} 

// 定义 TradeRepository 结构体
type TradeDb struct {}

func (u *TradeDb) Save(trade *Trade) error {
  // ...
}

type TradeBuy struct {}
func (tb *TradeBuy) Process(trade *Trade) error {}

type TradeSale struct {}
func (tb *TradeSale) Process(trade *Trade) error {}

func DoTrade(td *Trade, tp *TradeProcessor){
  td.Validate()
  tp.Process(td)
  td.Save()
}
```

在这种情况下，在流程不变的情况下，我们不需要修改 `DoTrade` 方法，只需要只对与 `TradeProcessor` 进行扩展，我们即可实现买或卖。

### 里式替换原则

> 里式替换原则表明，所有引用父类的地方必须能透明的使用其子类。也就是说，程序的正确性不应该依赖于子类的实现。
>
> 由于golang没有提供继承，只是通过接口提供了一个类似于父类-子类的继承关系。

对于里式替换原则，我们可以看看他的反例（这里是基于 [Is this a violation of the Liskov Substitution Principle?](https://softwareengineering.stackexchange.com/questions/170138/is-this-a-violation-of-the-liskov-substitution-principle) 这个例子的，就懒得翻译成 go，直接使用问题中的代码了。）

> 在下面的例子中，Task 可能在任意时间被 Close，而 ProjectTask 则不一样，当他的 Status == Status.Started 时会直接抛出异常。

```java
public class Task
{
     public Status Status { get; set; }

     public virtual void Close()
     {
         Status = Status.Closed;
     }
}

public class ProjectTask : Task
{
     public override void Close()
     {
          if (Status == Status.Started) 
              throw new Exception("Cannot close a started Project Task");

          base.Close();
     }
}
```

> 第一个优化方案是，我们新增加一个CanClose函数

```cpp
public class Task {
    public Status Status { get; set; }
    public virtual bool CanClose() {
        return true;
    }
    public virtual void Close() {
        Status = Status.Closed;
    }
}

public class ProjectTask : Task {
    public override bool CanClose() {
        return Status != Status.Started;
    }
    public override void Close() {
        if (!CanClose()) 
            throw new Exception("Cannot close a started Project Task");
        base.Close();
    }
}
```

> 第二个优化方案是，这个方案下 Close 不在是 `virtual` 的。这样设计相对于之前的好处是：
>
> 1. 任何其他的类型都可以判断此任务的状态是否可以关闭；
> 2. 可以提供原因。
>
> 为什么这样设计？因为当引入了一个额外的 Status 之后，我们的流程其实已经发生了改变，新的流程是：
>
> 1. 是否可以关闭；
> 2. 如果可以关闭则进入关闭流程；
>
> 而关闭的流程（在目前而言）对于所有的Task都是一样的：就是将Status设置为Closed。这样对于用户来讲，只需要实现CanClose即可。

```cpp
public class Task {
    public Status Status { get; private set; }

    public virtual bool CanClose(out String reason) {
        reason = null;
        return true;
    }
    public void Close() {
        String reason;
        if (!CanClose(out reason))
            throw new Exception(reason);

        Status = Status.Closed;
    }
}

public class ProjectTask : Task {
    public override bool CanClose(out String reason) {
        if (Status != Status.Started)
        {
            reason = "Cannot close a started Project Task";
            return false;
        }
        return base.CanClose(out reason);
    }
}
```

### 接口隔离原则

> Clients should not be forced to depend upon interfaces that they do not use.

简单理解就是，应该建立单一接口，而不是臃肿庞大的接口，例如当我们进行跨国交易的时候，我们可能会希望交易包含一些汇率信息，那么我们可以这样实现：

```go
// 定义 Trade 接口，要求实现 Process() 方法
type Trade interface {
  // Process 交易接口  
  Process() error
  // CalculateImpliedVolatility 计算汇率
  CalculateImpliedVolatility() error
}
```

在这种情况下，**那些不依赖于汇率的交易，也必须去实现这个计算汇率的接口**。更好的实现是，我们去定义两个不同的接口。**然后再为每个不同的子类去绑定不同的实现。**

```go
// 定义 Trade 接口，要求实现 Process() 方法
type Trade interface {
    Process() error
}

// 定义 OptionTrade 接口，要求实现 CalculateImpliedVolatility() 方法
type OptionTrade interface {
    CalculateImpliedVolatility() error
}

// 定义 FutureTrade 结构体，继承自 Trade 接口
type FutureTrade struct {
    Trade
}

func (ft *FutureTrade) Process() error {
    // process future trade
    return nil
}

// 定义 OptionTrade 结构体，继承自 Trade 接口
type OptionTrade struct {
    Trade
}

// OptionTrade 实现 Process() 方法，用于处理期权交易
func (ot *OptionTrade) Process() error {
    // process option trade
    return nil
}

// OptionTrade 实现 CalculateImpliedVolatility() 方法，用于计算隐含波动率
func (ot *OptionTrade) CalculateImpliedVolatility() error {
    // calculate implied volatility
    return nil
}
```

### 依赖倒置原则

> Depend upon abstractions, [not] concretions

这个比较好理解：依赖接口，不依赖实例。假设我们已经实现了我们存储Trade的代码，这样的代码更不容易扩展，因为他依赖于 `TradeMysql` 或者 `TradeOracle`。这是一个实例。

```go
type TradeMysql struct {}

func (tr *TradeMysql) Save(trade *Trade) error {
  // save trade to mysql
}

// 定义 TradeOracle 结构体
type TradeOracle struct {}

func (tr *TradeOracle) Save(trade *Trade) error {
  // save trade to oracle
}

func main() {
  td := Trade{}
  
  // save to mysql
  tm := TradeMysql{}
  tm.Save(&td)
  
  // save to oracle
  to := TradeOracle{}
  to.Save(&td)
}
```

我们可以使用接口来将他们解耦，我们定义了 `TradeService` 接口处理 `Trade`。同时定义了 `TradeProcessor`，**依赖于`TradeService` 接口，而不是 TradeService 的具体实现。 **

```go
type TradeService interface {
    Save(trade *Trade) error
}

// 定义 TradeProcessor 结构体，它包含一个 TradeService 接口
type TradeProcessor struct {
    tradeService TradeService
}

// Process 方法接收一个 Trade 接口类型的参数，并调用 TradeService 接口的 Save 方法保存数据
func (tp *TradeProcessor) Process(trade *Trade) error {
    err := tp.tradeService.Save(trade)
    if err != nil {
        return err
    }
    // process trade
    return nil
}

// 定义 SqlServerTradeRepository 结构体，它包含一个 SqlServer 数据源的 DB 连接
type SqlServerTradeRepository struct {
    db *sql.DB
}

// Save 方法接收一个 Trade 接口类型的参数，然后使用 SqlServer 的 INSERT 语句将数据插入到 trades 表中
func (str *SqlServerTradeRepository) Save(trade *Trade) error {
}

// 定义 MongoDbTradeRepository 结构体，它包含一个 MongoDb 数据源的 Session
type MongoDbTradeRepository struct {
    session *mgo.Session
}

// Save 方法接收一个 Trade 接口类型的参数，然后使用 MongoDb 的 Insert 语句将数据插入到 trades 表中
func (mdtr *MongoDbTradeRepository) Save(trade *Trade) error {
}
```











































