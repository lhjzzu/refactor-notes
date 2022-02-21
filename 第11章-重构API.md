## 重构API

+ 好的 API 会把更新数据的函数与只是读取数据的函数清晰分开

### 11.1 将查询函数和修改函数分离（Separate Query from Modifier）

```js
function getTotalOutstandingAndSendBill() {
const result = customer.invoices.reduce((total, each) => each.amount + total, 0);
sendBill();
return result;
}
```

```js
function totalOutstanding() {
return customer.invoices.reduce((total, each) => each.amount + total, 0);
}
function sendBill() {
emailGateway.send(formatBill(customer));
}
```

#### 动机

+ 如果某个函数只是提供一个值，没有任何看得到的副作用
  - 我可以任意调用这个函数
  - 也可以把调用动作搬到调用函数的其他地方
+ 明确表现出“有副作用”与“无副作用”两种函数之间的差异，是个很好的想法。 下面是一条好规则：
  - 任何有返回值的函数，都不应该有看得到的副作用--命令与查询分离

+ 如果遇到一个“既有返回值又有副作用”的函数，我就会试着将查询动作从修改动作中分离出来。

#### 做法

+ 从新建的查询函数中去掉所有造成副作用的语句。

####  范例

```js
//检查一群人（people）里是否混进了恶棍。如果发现了恶棍，该函数会返回恶棍的名字，并拉响警报
function alertForMiscreant(people) {
  for (const p of people) {
    if (p === "Don") {
      setOffAlarms();
      return "Don";
    }
    if (p === "John") {
      setOffAlarms();
      return "John";
    }
  }
  return "";
}
```

```js
function alertForMiscreant(people) {
  if (findMiscreant(people) !== "") setOffAlarms();
}

function findMiscreant(people) {
  for (const p of people) {
    if (p === "Don") {
      return "Don";
    }
    if (p === "John") {
      return "John";
    }
  }
  return "";
}
```

### 11.2 函数参数化（Parameterize Function）

#### 名字

+ 曾用名：令函数携带参数（Parameterize Method）

```json
function tenPercentRaise(aPerson) {
  aPerson.salary = aPerson.salary.multiply(1.1);
}
function fivePercentRaise(aPerson) {
  aPerson.salary = aPerson.salary.multiply(1.05);
}
```

```js
function raise(aPerson, factor) {
  aPerson.salary = aPerson.salary.multiply(1 + factor);
}
```



#### 做法

+ 运用改变函数声明（124），把需要作为参数传入的字面量添加到参数列表中。



#### 范例

```js
function baseCharge(usage) {
 if (usage < 0) return usd(0);
 const amount =
    bottomBand(usage) * 0.03
    + middleBand(usage) * 0.05
    + topBand(usage) * 0.07;
 return usd(amount);
}

function bottomBand(usage) {
 return Math.min(usage, 100);
}

function middleBand(usage) {
 return usage > 100 ? Math.min(usage, 200) - 100 : 0;
}

function topBand(usage) {
 return usage > 200 ? usage - 200 : 0;
}
```

```js
function withinBand(usage, bottom, top) {
 return usage > 100 ? Math.min(usage, 200) - 100 : 0;
}

function baseCharge(usage) {
 if (usage < 0) return usd(0);
 const amount =
    withinBand(usage, 0, 100) * 0.03
    + withinBand(usage, 100, 200) * 0.05
    + withinBand(usage, 200, Infinity) * 0.07;
 return usd(amount);
}
```

### 11.3 移除标记参数（Remove Flag Argument）

+ 曾用名：以明确函数取代参数（Replace Parameter with Explicit Methods）

```js
function setDimension(name, value) {
  if (name === "height") {
    this._height = value;
    return;
  }
  if (name === "width") {
    this._width = value;
    return;
  }
}

function setHeight(value) {
  this._height = value;
}
function setWidth(value) {
  this._width = value;
}
```

#### 动机

+ “标记参数”是这样的一种参数：调用者用它来指示被调函数应该执行哪一部分逻辑。

  - 只有参数值影响了函数内部的控制流，这才是标记参数

  ````js
  function bookConcert(aCustomer, isPremium) {
    if (isPremium) {
      // logic for premium booking
    } else {
      // logic for regular booking
    }
  }
  ````

  + 要预订一场高级音乐会（premium concert），就得这样发起调用：

    ```js
    bookConcert(aCustomer, true);
    ```

  + 标记参数也可能以枚举的形式出现：

    ```js
    bookConcert(aCustomer, CustomerType.PREMIUM);
    ```

  + 或者是以字符串（或者符号，如果编程语言支持的话）的形式出现：

    ```js
    bookConcert(aCustomer, "premium");
    ```

+ 我不喜欢标记参数，因为它们让人难以理解到底有哪些函数可以调用、应该怎么调用

  - 标记参数却隐藏了函数调用中存在的差异性
  - 用这样的函数，我还得弄清标记参数有哪些可用的值
  - 布尔型的标记尤其糟糕，因为它们不能清晰地传达其含义

+ 移除标记参数不仅使代码更整洁，并且能帮助开发工具更好地发挥作用
+ 如果一个函数有多个标记参数，可能就不得不将其保留，否则我就得针对各个参数的各种取值的所有组合情况提供明确函数。
  - 不过这也是一个信号，说明这个函数可能做得太多
  - 应该考虑是否能用更简单的函数来组合出完整的逻辑。

#### 做法

+ 针对参数的每一种可能值，新建一个明确函数。
+ 如果主函数有清晰的条件分发逻辑，可以用分解条件表达式（260）创建明确函数

#### 范例

```js
aShipment.deliveryDate = deliveryDate(anOrder, false);

function deliveryDate(anOrder, isRush) {
  if (isRush) {
    let deliveryTime;
    if (["MA", "CT"].includes(anOrder.deliveryState)) deliveryTime = 1;
    else if (["NY", "NH"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else deliveryTime = 3;
    return anOrder.placedOn.plusDays(1 + deliveryTime);
  } else {
    let deliveryTime;
    if (["MA", "CT", "NY"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else if (["ME", "NH"].includes(anOrder.deliveryState)) deliveryTime = 3;
    else deliveryTime = 4;
    return anOrder.placedOn.plusDays(2 + deliveryTime);
  }
}
```

```js
function rushDeliveryDate(anOrder) {
  let deliveryTime;
  if (["MA", "CT"].includes(anOrder.deliveryState)) deliveryTime = 1;
  else if (["NY", "NH"].includes(anOrder.deliveryState)) deliveryTime = 2;
  else deliveryTime = 3;
  return anOrder.placedOn.plusDays(1 + deliveryTime);
}
function regularDeliveryDate(anOrder) {
  let deliveryTime;
  if (["MA", "CT", "NY"].includes(anOrder.deliveryState)) deliveryTime = 2;
  else if (["ME", "NH"].includes(anOrder.deliveryState)) deliveryTime = 3;
  else deliveryTime = 4;
  return anOrder.placedOn.plusDays(2 + deliveryTime);
}

aShipment.deliveryDate = rushDeliveryDate(anOrder);
aShipment.deliveryDate = regularDeliveryDate(anOrder);//or
```

-------------------------

```js
//不容易移除isRush的情况
function deliveryDate(anOrder, isRush) {
 let result;
 let deliveryTime;
 if (anOrder.deliveryState === "MA" || anOrder.deliveryState === "CT")
  deliveryTime = isRush? 1 : 2;
 else if (anOrder.deliveryState === "NY" || anOrder.deliveryState === "NH") {
  deliveryTime = 2;
  if (anOrder.deliveryState === "NH" &amp;&amp; !isRush)
   deliveryTime = 3;
 }
 else if (isRush)
  deliveryTime = 3;
 else if (anOrder.deliveryState === "ME")
  deliveryTime = 3;
 else
  deliveryTime = 4;
 result = anOrder.placedOn.plusDays(2 + deliveryTime);
 if (isRush) result = result.minusDays(1);
 return result;
}
```

```js
//用两个新的函数来包装它，新函数的名字自解释其函数的目的
function rushDeliveryDate(anOrder) {
  return deliveryDate(anOrder, true);
}
function regularDeliveryDate(anOrder) {
  return deliveryDate(anOrder, false);
}
```

### 11.4 保持对象完整（Preserve Whole Object）

```js
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
if (aPlan.withinRange(low, high))
```

```js
if (aPlan.withinRange(aRoom.daysTempRange))
```

#### 动机

+ “传递整个记录”的方式能更好地应对变化：如果将来被调的函数需要从记录中导出更多的数据，我就不用为此修改参数列表。

#### 范例

```js
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
if (!aPlan.withinRange(low, high))
  alerts.push("room temperature went outside range");
  
  class HeatingPlan {
    withinRange(bottom, top) {
 return (bottom >= this._temperatureRange.low)&&(top <= this._temperatureRange.high);
}
  }
```

```js
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
if (!aPlan.xxNEWwithinRange(aRoom.daysTempRange))
  alerts.push("room temperature went outside range");
  
 class HeatingPlan {
    withinRange(aNumberRange) {
 return (aNumberRange.low >= this._temperatureRange.low) &&(aNumberRange.high <= this._temperatureRange.high);
   } 
 }
```

### 11.5 以查询取代参数（Replace Parameter with Query）



#### 名字

+ 曾用名：以函数取代参数（Replace Parameter with Method）
+ 反向重构：以参数取代查询（327）

```js
availableVacation(anEmployee, anEmployee.grade);

function availableVacation(anEmployee, grade) {
  // calculate vacation...

```

```js
  availableVacation(anEmployee)

function availableVacation(anEmployee) {
  const grade = anEmployee.grade;
  // calculate vacation...
```

#### 动机

+ 参数列表应该尽量避免重复，并且参数列表越短就越容易理解
  - 如果调用函数时传入了一个值，而这个值由函数自己来获得也是同样容易，这就是重复

#### 做法

+ 如果有必要，使用提炼函数（106）将参数的计算过程提炼到一个独立的函数中。

#### 范例

```js
class Order {

  get finalPrice() {
 const basePrice = this.quantity * this.itemPrice;
 let discountLevel;
 if (this.quantity > 100) discountLevel = 2;
 else discountLevel = 1;
 return this.discountedPrice(basePrice, discountLevel);
}

discountedPrice(basePrice, discountLevel) {
 switch (discountLevel) {
  case 1: return basePrice * 0.95;
  case 2: return basePrice * 0.9;
 }
}
   
}
```

````js
class Order {

  get finalPrice() {
 const basePrice = this.quantity * this.itemPrice;
 return this.discountedPrice(basePrice);
}

discountedPrice(basePrice) {
 switch (this.discountLevel()) {
  case 1: return basePrice * 0.95;
  case 2: return basePrice * 0.9;
 }
}
  
 get discountLevel() {
 return (this.quantity > 100) ? 2 : 1;
}
}
````



### 11.6 以参数取代查询（Replace Query with Parameter）

#### 名字

+ 反向重构：以查询取代参数（324）

  ```js
  targetTemperature(aPlan)
  
  function targetTemperature(aPlan) {
    currentTemperature = thermostat.currentTemperature;
    // rest of function...
  ```

  ````js
   targetTemperature(aPlan, thermostat.currentTemperature)
  
  function targetTemperature(aPlan, currentTemperature) {
    // rest of function...
  ````

#### 动机

+ 我有时会发现一些令人不快的引用关系，例如，引用一个全局变量，或者引用另一个我想要移除的元素
  - 这容易造成函数的副作用
  - 为了解决这些令人不快的引用，我需要将其替换为函数参数，从而将处理引用关系的责任转交给函数的调用者。

+ 需要使用本重构的情况大多源于我想要改变代码的依赖关系
  - 为了让目标函数不再依赖于某个元素，我把这个元素的值以参数形式传递给该函数
  - 需要注意权衡：如果把所有依赖关系都变成参数，会导致参数列表冗长重复
  - 如果作用域之间的共享太多，又会导致函数间依赖过度
+ 如果一个函数用同样的参数调用总是给出同样的结果，我们就说这个函数具有“引用透明性”（referential transparency）
  - 如果一个函数使用了另一个元素，而后者不具引用透明性，那么包含该元素的函数也就失去了引用透明性
  - 只要把“不具引用透明性的元素”变成参数传入，函数就能重获引用透明性。
  - 这样的函数可以叫做**纯函数(无副作用函数)**



### 11.7 移除设值函数（Remove Setting Method）

```js
class Person {
get name() {...}
set name(aString) {...}
```

```js
class Person {
get name() {...}
```



#### 动机

+ 如果为某个字段提供了设值函数，这就暗示这个字段可以被改变
+ 如果不希望在对象创建之后此字段还有机会被改变，那就不要为它提供设值函数（同时将该字段声明为不可变）
+ 将参数以构造函数的形式传入对象中

### 范例

```js
class Person {

get name() {return this._name;}
set name(arg) {this._name = arg;}
get id() {return this._id;}
set id(arg) {this._id = arg;}
}

//创建对象
const martin = new Person();
martin.name = "martin";
martin.id = "1234";

//对象创建之后，name 字段可能会改变，但 id 字段不会
```

```js
class Person {
  constructor(id) {
  this.id = id;
  }
  
  get name() {return this._name;}
  set name(arg) {this._name = arg;}
  get id() {return this._id;}
}

//创建对象
const martin = new Person("1234");
martin.name = "martin";
```



### 11.8 以工厂函数取代构造函数（Replace Constructor with Factory Function）

+ 曾用名：以工厂函数取代构造函数（Replace Constructor with Factory Method）

```js
leadEngineer = new Employee(document.leadEngineer, "E");
```

```js
leadEngineer = createEngineer(document.leadEngineer);
```



#### 做法

+ 新建一个工厂函数，让它调用现有的构造函数。



#### 范例

```js
class Employee {

 constructor (name, typeCode) {
  this._name = name;
  this._typeCode = typeCode;
}
get name() {return this._name;}
get type() {
  return Employee.legalTypeCodes[this._typeCode];
}
static get legalTypeCodes() {
  return {"E": "Engineer", "M": "Manager", "S": "Salesman"};
}
}


//调用方
candidate = new Employee(document.name, document.empType);

```

````js
function createEmployee(name, typeCode) {
  return new Employee(name, typeCode);
}

//调用方
candidate = createEmployee(document.name, document.empType);

````

### 11.9 以命令取代函数（Replace Function with Command）

+ 曾用名：以函数对象取代函数（Replace Method with Method Object）
+ 反向重构：以函数取代命令（344）

````js
function score(candidate, medicalExam, scoringGuide) {
  let result = 0;
  let healthLevel = 0;
  // long body code
}

class Scorer {
  constructor(candidate, medicalExam, scoringGuide) {
    this._candidate = candidate;
    this._medicalExam = medicalExam;
    this._scoringGuide = scoringGuide;
  }

  execute() {
    this._result = 0;
    this._healthLevel = 0;
    // long body code
  }
}
````



#### 动机

+ 将函数封装成自己的对象，有时也是一种有用的办法。这样的对象我称之为“命令对象”（command object），或者简称“命令”（command）
+ 与普通的函数相比，命令对象提供了更大的控制灵活性和更强的表达能力
  - 除了函数调用本身，命令对象还可以支持附加的操作，例如撤销操作
  - 我可以通过命令对象提供的方法来设值命令的参数值，从而支持更丰富的生命周期管理能力
  - 可以借助继承和钩子对函数行为加以定制
+ 命令对象的灵活性也是以复杂性作为代价的



#### 范例

```js
function score(candidate, medicalExam, scoringGuide) {
  let result = 0;
  let healthLevel = 0;
  let highMedicalRiskFlag = false;

  if (medicalExam.isSmoker) {
    healthLevel += 10;
    highMedicalRiskFlag = true;
  }
  let certificationGrade = "regular";
  if (scoringGuide.stateWithLowCertification(candidate.originState)) {
    certificationGrade = "low";
    result -= 5;
  } // lots more code like this
  result -= Math.max(healthLevel - 5, 0);
  return result;
}
```

```js
class Scorer {

constructor(candidate){
 this._candidate = candidate;
 }
  execute(candidate, medicalExam, scoringGuide) {
    let result = 0;
    let healthLevel = 0;
    let highMedicalRiskFlag = false;

    if (medicalExam.isSmoker) {
      healthLevel += 10;
      highMedicalRiskFlag = true;
    }
    let certificationGrade = "regular";
    if (scoringGuide.stateWithLowCertification(candidate.originState)) {
      certificationGrade = "low";
      result -= 5;
    } // lots more code like this
    result -= Math.max(healthLevel - 5, 0);
    return result;
  }
}

function score(candidate, medicalExam, scoringGuide) {
  return new Scorer(candidate).execute(candidate, medicalExam, scoringGuide);
}
```



### 11.10 以函数取代命令（Replace Command with Function）

+ 反向重构：以命令取代函数（337）

```js
class ChargeCalculator {
  constructor(customer, usage) {
    this._customer = customer;
    this._usage = usage;
  }
  execute() {
    return this._customer.rate * this._usage;
  }
}
```

```js
function charge(customer, usage) {
  return customer.rate * usage;
}
```



#### 范例

```js
class ChargeCalculator {
  constructor(customer, usage, provider) {
    this._customer = customer;
    this._usage = usage;
    this._provider = provider;
  }
  get baseCharge() {
    return this._customer.baseRate * this._usage;
  }
  get charge() {
    return this.baseCharge + this._provider.connectionCharge;
  }
}

monthCharge = new ChargeCalculator(customer, usage, provider).charge;
```

```js
monthCharge = charge(customer, usage, provider);
function charge(customer, usage, provider) {
  const baseCharge = customer.baseRate * usage;
  return baseCharge + provider.connectionCharge;
}
```



