**安装MongDB：**

1. 配置环境变量
2. 创建存放数据的文件夹
3. 打开cmd命令，输入mongod启动mongodb服务器
   * 自定义数据库存储路径和端口
   * mongod --dpath 数据库路径 --port 端口号 
4. 在打开一个cmd窗口，输入mongo命令，连接mongodb（出现<，表示连接成功）



### 一、mongoDB基础

**基本命令：**

* show dbs（显示所有数据库=show databases）

* use db1（切换到某数据库，db1为数据库名，如果db1不存在，也可使用该命令，不会报错，当在改数据库创建文档时，会自动创建该数据库）

* show collections（查看数据库中的集合）

* db（查看当前所在数据库）

**数据库的CRUD：(假设表为stus，数据库为db1)**

* **插入文档：**

  db.<collection>.insert(doc)（此处db表示当前数据库）

  eg：向db1数据库，stus集合插入一个学生对象{name:"小明", gender:"男"}

  db.stus.insert({name:"小明", gender:"男"})

  **插入多条数据：**

  db.stus.insert([{name:"小明", gender:"男"},

  ​                           {name:"小红", gender:"女"}])

  **_id属性可以自己指定**，指定后数据库就不会自动生成了：

  db.stus.insert({_id:"232dad", name:"小明", gender:"男"})

* **查询文档：**

  db.stus.find()
  
  **条件查询：**find()可以接受一个对象作为条件参数，{}表示查询集合中所有文档
  
  db.stus.find({_id:"10086abc"})  //__id="10086abc"
  
  db.stus.find({age:16, name:"小明"}) //查询age=16且姓名为小明 
  
  db.stus.findOne({条件}) //查询集合中符合条件的第一个文档
  
  **find()返回的是一个数组，findOne()返回的是一个文档，find()后面可以跟索引取指定文档：find({})[1]**
  
  db.stus.find({}).count() //查询文档数量
  
  **如果要通过内嵌文档对文档进行查询，此时属性名必须使用引号**
  
  ```json
  db.users.find({"hobby.movies":"hero"});
  ```
  
  
  
* **修改文档：**

  **添加：**

  ```json
  db.users.update({username:"小明"},{$push:{"hobby.movies":"I"}})
  ```

  **addToSet也是添加元素，但是如果数组中存在该元素，则会添加失败**

  **替换：**

  ```json
  db.stus.replaceOne({username:"小明"}, {username:"小狗"})
  ```

  

  db.collections.update(查询条件，新对象) //会替换原有对象

  eg：db.stus.update({name="小明"}, {新对象})

  **注意：update默认只改一个，要修改多个可以使用updateMany()或者选择update第三个参数中的multi=true**

  eg:

  ```json
  db.stus.update(
  {"name":"小明"},
  {$set:{age:24}},
  {multi:true},
  )
  ```

  

  **如果需要修改指定属性，而不是完全替换，需要使用"修改操作符"**

  * $set 可以用来修改文档中的指定属性

    ```json
    db.stus.update(
    {"name":"小明"},//查询条件
    {$set:{name:"小黄"}}
    )
    ```

  * $unset 删除文档中的指定属性

* **删除文档：**

  db.collections.remove()

  db.collections.deleteOne()

  db.collections.deleteMany()

  ```json
  db.stus.remove({_id:"10086abc"})
  //有多个满足条件时，默认删除多个
  ```

  ```json
  db.stus.remove() //会报错，必须有参数
  
  db.stus.remove({}) //清空集合（集合还在），一行一行删除，性能较差
  db.stus.drop() //删除集合
  ```

  db.dropDatabase() //删除数据库

**当我们创建文档时，如果文档所在集合或数据库不存在，会自动创建数据库和集合**



**mongodb也可以使用for循环**

例如需要插入两万条数据（数据库的方法调用越少性能越好）

```java
// 一条条添加性能很差，耗时7.2s
for(var i=1;i<=20000;i++){
    db.numbsers.insert({num:i});
}

// 先创建一个数组，一块添加，耗时0.4s
var arr = [];
for(var i=1;i<=20000;i++){
    arr.push({num:i});
}
db.numbers.insert(arr);
```

* 显示前10条数据

  ```json
  db.numbers.find().limit(10)
  ```

* 显示第10到20的数据

  ```json
  db.numbers.skip(10).limit(10)
  // mongoDB会自动调整skip和limit的顺序，谁写前谁写后都可以
  ```

  

**自定义排序：（1是升序，2是降序）**

```json
// 按照sal升序
db.emp.find().sort({sal:1});

// 按照sal降序
db.emp.find().sort({sal:-1});

// 先按照sal升序排列，sal相等在按照empno降序
db.emp.find().sort({sal:1,empno:-1})
```



**limit, skip, sort可以按任意顺序调用，mongoDB会自动先sort->skip->limit**



#### 在查询时，第二个参数可以设置查询结果的投影

```json
//只查询姓名和工资（_id默认会查询出来，需要指定为0）
db.emp.find({},{name:1,sal:1,_id:0})
```



### 二、mongoose

Mongoose是一个可以通过Node来操作MongDB的模块

Mongoose是一个对象文档模型(ODM)库，它对Node原生的MongDB模块进行了进一步的优化封装，并提供了更多的功能

**mongoose的好处：**

* 可以为文档创建一个模式结构Schema（约束）
* 可以对模型中的对象/文档进行验证
* 数据可以通过类型转换转换为对象模型
* 可以使用中间件来应用业务逻辑挂钩
* 比Node原生的MongoDB驱动更容易



**mongoose提供的新的对象：**

* **Schema（模式对象）：**

  Schema对象定义约束了数据库中的文档结构

* **Model：**

  Model对象作为集合中的所有文档的表示，相当于MongoDB数据库的集合collection

* **Document：**

  Document表示集合中的具体文档，相当于集合中的一个具体文档

**创建顺序：Schema->Model->Document**



