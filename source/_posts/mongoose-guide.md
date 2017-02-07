---
title: Mongoose - 在 NodeJs 中优雅地建立 MongoDb 对象模型
date: 2016-08-13 17:07:15
tags: [nodejs, mongodb, javascript]
---
在 NodeJs 中写 MongoDb 的校验，构造和业务逻辑简直是一种拖累。这就是为什么我们写了 Mongoose。
<!-- more -->
##  什么是 MongoDb 和 Mongoose

MongoDb

>MongoDB is an open-source document database that provides high performance, high availability, and automatic scaling.

>MongoDb 是一个开源的文档数据库，可提供高性能，高可用性和自动缩放。

MongoDB 是一个基于分布式文件存储的数据库。由 C++ 语言编写。旨在为 WEB 应用提供可扩展的高性能数据存储解决方案。

MongoDB 是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。

Mongoose

>Mongoose provides a straight-forward, schema-based solution to model your application data. It includes built-in type casting, validation, query building, business logic hooks and more, out of the box.

>Mongoose 为你的应用程序数据提供了一个直接的，基于模型的解决方案。他包括内置的类型转换，校验，建立查询，业务逻辑的钩子等。

>Mongoose是一个提供了MongoDB地相映射的Node.js库，它在Node.js中与ORM（Object Relational Mapping）有着类似的接口。如果你不熟悉ORM或者Mongoose中的Object Data Mapping（ODM），意思是Mongoose将数据库中的数据转换为JavaScript对象以供你在应用中使用。

##  为什么使用 Mongoose

>Let’s face it, writing MongoDB validation, casting and business logic boilerplate is a drag

##  安装和使用 Mongoose
1.  安装
    `$ npm install mongoose --save-dev`
2.  使用
    `$ touch index.js`

    在 `index.js` 文件中引入 `Mongoose` ，并连接数据库

    ```
    var mongoose = require('mongoose'); mongoose.connect('mongodb://localhost/myblog');//连接上 myblog 数据库
    ```

**开始使用 Mongoose 操作 MongoDb**

**注：本文默认已经搭好 Nodejs 和 MongoDb 环境，若未搭建环境，请移步至 -> 还在编写当中**

假设我们在构建一个博客，需要将博文存储至数据库中。
而每一篇博文的结构如下：
```json
{
    title: '标题',
    author: '作者',
    date: '时间',
    comments: '评论',
    tags: '标签',
    body: '正文'
}
```
**OK, Go！**

**Mongoose 中的 Schema 和 Model**

在使用 Mongoose 前，先了解一下 Mongoose 中的 Schema 和 Model。

* Schema
    在使用 Mongoose 前，先了解一下 Mongoose 中的 Schema 和 Model。

    >Everything in Mongoose starts with a Schema. Each schema maps to a MongoDB collection and defines the shape of the documents within that collection.

    >在 Mongoose 中，所有东西都从一个 Schema 开始。每一个 schema 都映射到一个 MongoDb 的集合，并定义了该集合中的文档的形式。

    定义一个 Schema：
    ```javascript
    var Schema = mongoose.Schema;
    var userSchema = new Schema({
        name: String,
        pass: String,
        email: String,
        createTime: Date,
        lastLogin: Date
    });
    ```
    Schema 中的每一个键都定义了一个文档的一个属性。在上面的例子中，我们定义了用户名 `name` ，它会被映射为 String 的 Schema 类

    型，注册时间 `createTime` 会被映射为 Date 的 Schema 类型。

    允许的 Schema 类型有：（[了解更多](http://mongoosejs.com/docs/schematypes.html)）

    * String
    * Number
    * Date
    * Buffer
    * Boolean
    * Mixed
    * ObjectId
    * Array

    用法：
    * 自定义方法
        模型的实例都是一个个的文档，文档中自带许多方法。同时，我们也可以定义我们自己的方法。

        ```javascript
        var userSchema = new Schema({
            name: String,
            pass: String,
            email: String,
            createTime: Date,
            lastLogin: Date，
            type: String
        });
        //根据该用户的类型区查找该类型下的所有用户
        userSchema.methods.findUsersByType = function(name, cb){
            return this.find({type: this.type}, cb);
        }
        //新建模型
        var User = mongoose.model('User', userSchema);
        //使用
        var newUser = new User({...});
        newUser.findUsersByType(function(err, users){
            err && return console.error(err);
            console.log(users);
        })
        ```
        这样就向 `User` 的实例添加了一个自定义的方法。
    * 静态方法
        同样的，向模型中添加自定义方法也是很简单。

        ```javascript
        userSchema.statics.findUsersByName = function(name, cb){
            return this.find({name: new RegExp(name, "ig")}, cb);
        }
        //使用
        User.findUsersByName('leung', function(err, users){
            err && return console.error(err);
            console.log(users);
        })
        ```
    * 查询辅助
        可以自定义一个查询的辅助函数，它和实体的方法类似，但是供 Mongoose 查询使用。

        ```javascript
        userSchema.query.byName = function(name){
            return this.find({name: new RegExp(name, "ig")});
        }
        //使用
        userSchema.find().byName('leung').exec(function(err, users){
            err && return console.error(err);
            console.log(users);
        })
        ```
    * 索引
        MongoDb 支持第二个索引，在使用 Mongoose 的时候，可以在定义 Schema 的时候定义索引。

        ```javascript
        //定义方法1
        var userSchema = new Schema({
            name: String,
            pass: String,
            email: String,
            createTime: {type: Date, index: true},
            lastLogin: {type: Date, index: true},
            type: String
        });
        //定义方法2
        userSchema.index({ createTime: 1, lastLogin: -1 });
        ```

        Mongoose 会在程序启动的时候，对于每个定义了索引的字段自动调用 `ensureIndex` 函数。当在不需要这些索引的时候，可以使用下列 4 种方式关闭索引。

        ```javascript
        mongoose.connect('mongodb://user:pass@localhost:port/database', { config: { autoIndex: false } });
        // or
        mongoose.createConnection('mongodb://user:pass@localhost:port/database', { config: { autoIndex: false } });
        // or
        userSchema.set('autoIndex', false);
        // or
        new Schema({..}, { autoIndex: false });
        ```
    * 虚拟字段
        虚拟字段可以让你很方便的在文档中存取，但是不会写入数据库中。getter 方法在格式化或者合并字段的时候很有用，而 setter 方法则在反格式化或者时将多个值合并的时候有用。
        ```javascript
        var personSchema = new Schema({
            name:{
                firstName: String,
                lastName: String
            }
        });
        var Person = mongoose.model('Person', personSchema);
        //定义虚拟字段 fullName
        personSchema.virtual('name.fullName').get(function(){
            console.log(this);
            return this.name.firstName + ' ' + this.name.lastName;
        })
        var me = new Person({name: {firstName: 'zhong', lastName: 'Lueng'}});
        console.log(me);
        console.log(me.name.fullName) //zhong Lueng
        ```
        虚拟字段的 setter 方法会在其他校验前使用，因此，即使字段时必须的，虚拟字段也会正常执行。

        **只用非虚拟字段才可以在查询或者字段选择中使用。**
    * 配置项
        Schema 有许多可配置的配置项，可以在新建 Schema 时或者直接设置。

        ```javascript
        new Schema({..}, options);
        //or
        var schema = new Schema({..});
        schema.set(option, value);
        ```
        有效的配置项：
        * `autoIndex`
        * `capped`
        * `collection`
        * `emitIndexErrors`
        * `id`
        * `_id`
        * `minimize`
        * `read`
        * `safe`
        * `shardKey`
        * `strict`
        * `toJSON`
        * `toObject`
        * `typeKey`
        * `validateBeforeSave`
        * `versionKey`
        * `skipVersioning`
        * `timestamps`
* Model
    在 Mongoose 中，模型是通过已定义的 Schema 生成，这些模型的实例可以保存并检索数据库中的文档。数据库中的所有文档都是通过模型来创建和检索。
    * 创建文档
        文档是模型的实例，创建文档和将其保存到数据库中是非常简单的：
        ```javascript
        var animalSchema = new Schema({
            type: String
        });
        var Animal = mongoose.model('Animal', animalSchema);
        var cat = new Animal({type: 'cat'});
        cat.save((err, doc) =>{
            err && console.error(err);
            console.log('doc is saved');//saved
        });
        // or
        Animal.create({type: 'cat'}, (err, doc) =>{
            err && console.error(err);
            //saved!
        })
        ```
    * 查询文档
        在 Mongoose 中查询文档是很方便的，同时支持 MongoDb 原生语法，模型中有许多内置的静态函数： `find` , `where` , `findById` , `findOne` 等等， 也支持链式调用。
        ```javascript
        Animal.find({type: 'dog'}).where('age').gt(2).limit(10).exec();
        ```
    * 移除文档
        每个模型都有一个默认的移除方法，可以移除所有符合搜索条件的文档。
        ```javascript
        Animal.remove({type: 'dongbeihu'}, (err, res){
            err && console.error(err);
            //removed!
        })
        ```
    * 更新文档
        每个模型都有他自己的更新方法，供更新数据库中所有符合条件的文档，同时不返回任何东西。

        **如果需要更新单个文档，并且返回更新后的文档，请使用 `findOneAndUpdate`**

下面可开始使用 Mongoose 来操作 MongoDb 数据库了。

1. 创建博文的 Schema
    ```javascript
    var Schema = mongoose.Schema;
    var blogSchema = new Schema({
        title: String,          //标题
        author: String,         //作者
        body: String,           //正文
        comments: [{            //评论
            author: String,
            date: Date,
            body: String
        }],
        date: {type: Date, default: Date.now},  //发布时间
        isDelete: Boolean,                        //是否删除
        tags: [{type: String}]                  //标签
    }, {collection: 'blog'});   //选择 blog 集合
    ```
2. 创建一个 Blog 模型
    创建了一个模型之后，我们就可以通过模型去创建新的文档或者去操作数据库。
    ```javascript
    var Blog = mongoose.model('Blog', blogSchema);
    ```
3. 创建一个新的实体对象
    **这里的所有数据都是通过 Mockjs 进行模拟。详情移步至 [Mockjs 官网](http://mockjs.com/)**
    使用 mockjs 模拟真实数据
    ```javascript
    var Mock = require('mockjs');
    var Random = Mock.Random;
    exports.getBlog = function(){
        var blog = {
            title: Random.ctitle(3,10),         //随机生成一个长度3-10之间的中文标题
            author: Random.cword(2,3),          //随机生成一个长度2-3之间的中文单词
            body: Random.cparagraph(1,3),       //随机生成1-3段中文段落
            comments: [],
            date: Random.datetime('yyyy-MM-dd HH:mm'),  //随机生成一个时间点
            tags: []
        }
        var length =~~(Math.random()*10) +5;
        for (var i = 0; i < length; i++) {
            blog.comments.push(Mock.mock({
                        author: Random.cword(2,3),      //随机生成一条评论的作者名字
                        body: Random.cparagraph(1,3),   //随机生成一条评论的内容
                        date: Random.datetime('yyyy-MM-dd HH:mm')   //随机生成评论的时间点
                    }))
        }
        var length2 =~~(Math.random()*4) +2;
        for (var i = 0; i < length2; i++) {             //随机加入几个标签
            var tag = Random.pick(['nodejs','js','html','css','java','php','python']);
            if(blog.tags.indexOf(tag) == -1) blog.tags.push(tag)
        }
        return blog;
    }
    ```
    生成一个 blog 实体对象
    ```javascript
    var newBlog = require('./blog')；        //导入随机生成数据的模块
    var blogObj = newBlog.getBlog();
    var blog = new Blog(blogObj)  //将随机生成的数据转换成为 blog 模型的实体。
    //生成实体对象的另一种方法.
    //此方法相当于创建了一个实体并写入数据库，再返回一个 promise 对象。
    //var blog2 = Blog.create(blogObj);
    ```

4. 数据库的 CRUD
    * create
        基于实体对象: `Entity.save(callback)`;

        Example:
        ```javascript
        var blogObj = newBlog.getBlog();
        var blog = new Blog(blogObj)
        blog.save((err, doc) => {
            if(err) return console.error(err);
            console.log(doc);
        })
        //{title: '...', author: '...'}
        ```
        结果：
        ![](mongoose-guide/entity,save.png)
        基于模型： `Model.create(Obj, callback)`

        Example:
        ```javascript
        Blog.create(blogObj, (err, doc) => {
            if(err) return console.error(err);
            console.log(doc);
        })
        //{title: '...', author: '...'}
        ```
        结果：
        ![](mongoose-guide/model.create.png)
    * delete
        基于实体对象：`Entity.remove(cb)`

        基于模型：`Model.remove(query, cb)`
    * update
        基于实体对象：`Entity.update(cb)`

        Example:
        ```javascript
        var blog = new Blog(blogObj);
        blog.save((err, doc) => {   //先添加新的博文
            if(err) console.error(err);
            console.log('Entity.save()');
            console.log(doc);
            //通过实体直接更新博文
            blog.update({$set: {date: new Date(), title: 'Update From Mongoose'}}, (err2, doc) => {
                if(err2) return console.error(err2);
                console.log(doc);
            })
        })
        ```
        结果：
        ![](mongoose-guide/entity.update.png)
        更新后在 Mongobooster 中的数据
        ![](mongoose-guide/entity.update2.png)
        基于模型：
        ```javascript
        Model.update(query, docs, options, cb)
        ```
        参数：
        * `query`：查询条件。
        * `docs`：需要更新的数据。
        * `options`：配置项。
        * `cb`：回调。
        返回：

        `<Qeury> this`

        其中，配置项中有效的配置如下：
        * `safe` (boolean) safe mode (defaults to value set in schema (true))
        * `upsert` (boolean) whether to create the doc if it doesn’t match (false)
        * `multi` (boolean) whether multiple documents should be updated (false)
        * `runValidators`: if true, runs update validators on this command. Update validators validate the update operation against the model’s schema.
        * `setDefaultsOnInsert`: if this and upsert are true, mongoose will apply the defaults specified in the model’s schema if a new document is created. This option only works on MongoDB * >= 2.4 because it relies on MongoDB’s $setOnInsert operator.
        * `strict` (boolean) overrides the strict option for this update
        * `overwrite` (boolean) disables update-only mode, allowing you to overwrite the doc (false)
        * `context` (string) if set to ‘query’ and runValidators is on, this will refer to the query in custom validator functions that update * validation runs. Does nothing if runValidators is false.

        **只有传入 `callback` 时，所有的更新操作才会被执行**

        Example：
        ```javascript
        //以下操作均不会执行
        var q = Blog.where({title: /del/g}).update({$set: {comments: []}});
        //等同于
        //var q = where({title: /del/g}).update({comments: []});
        //下面三条命令均会执行更新操作。
        //q.exec()
        //q.update(true)
        var q = Blog.where({title: /del/g}).update({$set: {comments: []}}, (err) => {
            //executes
        });
        //将文档重写为空文档
        var q = Blog.where({ _id: id }).setOptions({ overwrite: true })
        q.update({ }, callback); // executes
        //or
        //var q = Blog.update({_id: id}, {}, {overwrite: true}, callback);
        //将所有博文的创建日期设置为今天
        Blog.where()
         .setOptions({ multi: true })
         .update({ $set: { date: new Date() }}, callback)
        //or
        //Blog.update({}, { $set: { date: new Date() }}, { multi: true }, callback);
        ```
        批量更新结果：
        ![](mongoose-guide/model.update.png)
    * read(query)
        **find 操作包含了许多查询的操作，在这里只介绍最常用的几种查询方法，具体的可以查阅官网的 [API Guide](http://mongoosejs.com/docs/api.html#query-js)**

        * $where **(注意前面的 $)**
        可以传入一个 javascript 函数或者表达式作为查询条件。

        用法：$where(js)
        ```javascript
        //在数据库中找出作者名字长度为 2 且评论大于 10 条的博文：
        Blog
        .$where('this.comments.length >= 10 && this.author.length === 3')
        //.$where(function(){return this.comments.length >= 10 && this.author.length === 3;})
        .select('title author commentSize')
        .find((err, docs)=>{
            if(err) return console.error(err);
            console.log(docs);
        })
        ```
        查询结果：
        ![](mongoose-guide/find.$where.png)
        * where
        指定一个查询字段,供后续的链式调用.字段可以是字符串,也可以时一个完整的查询对象。

        用法：`where(path, val)`，`path` 可以是字符串或是对象

        ```javascript
        //查找评论数大于8条,且标题含有叫字的博文
        Blog
        .where('commentSize').gte(8)
        //.where({'commentSize':{gte: 8}})
        .where('title', /叫/g)
        .select('title author commentSize')
        .find((err, docs)=>{
            if(err) return console.error(err);
            console.log(docs);
        })
        ```
        查询结果：
        ![](mongoose-guide/find.where.png)
    * all
        指定一个 all 的查询条件。all 的使用方法可以查阅 MongoDb [官方文档](https://docs.mongodb.com/manual/reference/operator/query/all/)

        用法： `all([path], val)`,当只传递一个参数时，查询字段已经在 `where` 中指定。
        ```javascript
        //查询出同时含有 html 和 js 标签的博文
        Blog
        // .where('tags').all(['html', 'js'])
        .find().all('tags', ['html', 'js'])
        .select('title author tags')
        .find((err, docs)=>{
            if(err) return console.error(err);
            console.log(docs);
        })
        ```
        查询结果：
        ![](mongoose-guide/find.all.png)
    * and
        指定一个 and 的查询条件。

        用法：`and(array)`
        ```javascript
        //查找出标题带有叫字，且含有 js 标签的博文
        Blog.find()
        .and([{title: /叫/g}, {tags: 'js'}])
        .select('title author tags')
        .find((err, docs)=>{
            if(err) return console.error(err);
            console.log(docs);
        })
        ```
        查询结果：
        ![](mongoose-guide/find.and.png)
    * count
        返回符合指定条件的文档个数。

        用法：`count(query, callback)`
        ```javascript
        //查找出标题带有叫字，且含有 js 标签的博文
        Blog.find()
        .and([{title: /叫/g}, {tags: 'js'}])
        .select('title author tags')
        .count((err, docs)=>{
            if(err) return console.error(err);
            console.log(docs);
        })
        ```
        查询结果：1
    * limit
        指定返回查询条数。

        用法：`limit(Number)`。
    * or / nor
        指定一组查询条件，只要符合/不符合其中一个条件即可。

        用法：`or(array) / nor(array)`
        ```javascript
        //查询所有标签包含 js, 或者标题有“叫”字的博文
        Blog.find()
        .or([{title: /叫/g}, {tags: 'css'}])
        .select('title author tags')
        .find((err, docs)=>{
            if(err) return console.error(err);
            console.log(docs);
        })
        ```
        查询结果：
        ![](mongoose-guide/find.or.png)
        ```javascript
        //查询所有标签不包含 js或者标题不含有“叫”字的博文
        Blog.find()
        .nor([{title: /叫/g}, {tags: 'js'}])
        .select('title author tags')
        .find((err, docs)=>{
            if(err) return console.error(err);
            console.log(docs);
        })
        ```
        查询结果：
        ![](mongoose-guide/find.nor.png)
    * slice
        返回符合指定条件的文档个数。

        用法：`slice([path], value)`
        ```javascript
        //查出所有的博文，并显示每个博文的第一个标签。
        Blog
        .where('tags').slice(1)
        .select('title')
        .limit(4)
        .exec((err, doc) => {
            console.log(doc);
        })
        ```
        查询结果：
        ![](mongoose-guide/find.slice.png)
    * sort
        设定排序规则。

        用法：`sort(arg)`.参数可以是对象，可以是字符串。
        ```javascript
        //查出所有的博文，并按照标题排序。
        Blog.find()
        .sort('title')
        //.sort({title: 1})     //传入对象参数
        //.sort({title: -1})    //传入对象参数，标题倒叙
        //.sort('-title')       //传入对象字符串，标题倒叙
        .limit(4)
        .exec((err, doc) => {
            console.log(doc);
        })
        ```
        查询结果：
        ![](mongoose-guide/find.sort.png)
5. validate
    Mongoose 中，在定义 Schema 的时候，允许用户自定义字段的校验规则。当在 Schema 中定义校验规则的时候，需要遵守下列规则：
    * 校验规则需要在 Schema 中定义。
    * 校验会在 `save` 操作前调用。
    * 可以通过 `doc.validate(callback)` 和 `doc.validateSync()` 手动校验。
    * 未定义校验规则的字段将不会校验，除非是 `required` 必要字段。
    * 校验时异步递归的，当使用 `Model.save` 时，子文档的校验规则将会立即调用，有错误时，执行的 `Model.save` 会接收到错误。
    * 校验时自定义的。
    普通的校验定义：
    ```javascript
    var personSchema = new Schema({
    name:{
        type: String,
        validate:{
            validator: function(value){
                //只能以字母开头，只能包含英文数字下划线。
                return /^[a-zA-Z][a-zA-Z0-9_]*$/.test(value);
            },
            message: '{VALUE} is not a valid username. It must begin with a letter.'
        },
        required: [true, 'Username is required']
    },
    phone: {
        type: String,
        validate: {
        validator: function(v) {
            //11位电话号码
            return /\d{11}/.test(v);
            },
            message: '{VALUE} is not a valid phone number!'
        },
        required: [true, 'User phone number required']
    },
    gender:{
        type: String,
        enum: {
            //只能是男女
            values: ['female', 'male'],
            message: '{VALUE} is not a valid gender'
        },s
    }
    },{collection: 'person'})
    var Person = mongoose.model('Person', personSchema);
    //测试数据
    var one = new Person({
        name: '123',
        phone:'11111',
        gender:' male'
    })
    ```
    我们使用 `one.validateSync()` 对数据进行验证，其验证结果如下：
    ![](mongoose-guide/validate-result.png)
    Mongoose 中内置的验证如下：

    >* 所有的 Schema 都有 required 校验规则，必要校验器会调用 Schema 的 chenckRequired() 函数去校验该字段的值是否符合规则。
    >* Number 类型有 min 和 max。
    >* String 类型有 enum, minlength, maxlength, match的校验器。

    * `required`：必填校验。
	可以传入的参数为：`boolean[，String]`。当只传 `true` 或者 `false` 时，用于定义该字段是否为必填；当传入第二个参数 String 时，该信息作为校验失败的错误提示语。
    * `enum`：枚举类型校验。
    	接受参数类型为：`Array/ Object`。当类型为 `Array` 时，指定该字段的值必须在该数组中；当传入类型为 `Object` 时，可以定义两个键值队：`values` 和 `message`，`values` 的类型为 Array，指定字段的值必须在数组 `values` 中，`message` 指定校验失败的错误提示语。
    * `validator`：校验的对象。
    	该对象包含 2 个键值队：
    	* `validate` 可以是正则表达式，也可以是一个函数。
    	* `message` 则是错误信息提示语。
    * `min ,max`：对数字类型的进行大小校验。第二个参数可以为字符串，作为错误提示语。
    * `minlength` ,`maxlength`：对字符串类型的进行长度校验。第二个参数可以为字符串，作为错误提示语。
    * `match`：接受一个正则表达式进行验证。第二个参数可以为字符串，作为错误提示语。
    * `trim`：字符串在校验时，是否去除前后字符串。默认为 `false`。

使用 Mongoose 的自动校验功能，可以省下许多校验操作。

以前，我们直接使用 Mongodb 去操作数据，每次读取操作都要打开，关闭数据库，但是当网页刷新/访问过于频繁时，就会出现数据库来不及关闭，又开始新的查询，就可能会出现 error。

现在，我们使用 Mongoose，打开数据库的连接之后，db 就会一直处于连接状态，不需要在访问时才打开连接，操作完后关闭连接，Error 也不再出现了。而且，代码的易读性也提高了不少。
