---
title: "jpa/querydsl 支持json_build_object 和 jsonb_agg"
categories: Java
---

### postgresql jsonb 相关函数
postgresq支持jsonb类型，并且提供了一些函数，如：
[初始化sql][init.sql]
```sql

select json_build_object('author', author, 'title', title) from book;
--      json_text
-- "{""author"" : ""罗贯中"", ""title"" : ""三国演义""}"
-- "{""author"" : ""曹雪芹"", ""title"" : ""红楼梦""}"
-- "{""author"" : ""施耐庵"", ""title"" : ""水浒传""}"
-- "{""author"" : ""吴承恩"", ""title"" : ""西游记""}"
-- json_build_object 函数可以用来构建json对象

select t1.id, t1.name, jsonb_agg(json_build_object('author', t2.author, 'title', t2.title)) as books
from library t1
         left join book t2 on t1.id = t2.library_id
group by t1.id;

- id, name , books
-- 2,海珠图书馆,"[{""title"": ""水浒传"", ""author"": ""施耐庵""}, {""title"": ""三国演义"", ""author"": ""罗贯中""}]"
-- 1,天河图书馆,"[{""title"": ""红楼梦"", ""author"": ""曹雪芹""}, {""title"": ""西游记"", ""author"": ""吴承恩""}]"
-- jsonb_agg 函数可以用来聚合json对象,从而实现一对多可以一条sql获取到

```
json_build_object 和 jsonb_agg 都时postgresql所特有的函数，jpa里面时不支持支持这些非标准函数的。

### 事件起因
我们公司的系统是需要做多语言，然后表设计的时候，经常有一些1对多的查询, 每次写的时候都需要像下面这样:
``` 
  // TestBuildJson3
  String template = String.format("CAST(json_object_agg(COALESCE({0}, ''), COALESCE({1}, '')) as text)");
  StringTemplate tp = Expressions.stringTemplate(template, QBook.book.author, QBook.book.title);
  JPAQuery<Tuple> query = new JPAQuery<>(entityManager)
            .select(QLibrary.library, tp)
            .from(QLibrary.library)
            .leftJoin(QBook.book)
            .on(QLibrary.library.eq(QBook.book.library))
            .groupBy(QLibrary.library.id);
```
每次需要指定需要的字段, 需要用模板来拼接。但字段一段或这加多字段的时候，就需要修改，比较麻烦，我们想的是想下面的代码,可以直接想select一样
可以自动展开所有字段，同时也可以支持自定义字段和别名。期望代码如下
```
StringTemplate tp = Expressions.jsonbAggTemplate(QBook.book);
StringTemplate tp = Expressions.jsonbAggTemplate(QBook.book.author);
StringTemplate tp = Expressions.jsonbAggTemplate(QBook.book.title.as("标题"), QBook.book.author);

JPAQuery<Tuple> query = new JPAQuery<>(entityManager)
                .select(QLibrary.library, tp)
                .from(QLibrary.library)
                .leftJoin(QBook.book)
                .on(QLibrary.library.eq(QBook.book.library))
                .groupBy(QLibrary.library.id);
                
可以通过三面种，分类生成如下sql
select t1.id, t1.name, jsonb_agg(json_build_object('author', t2.author, 'title', t2.title))
from library t1 left join book t2 on t1.id = t2.library_id group by t1.id;

select t1.id, t1.name, jsonb_agg(json_build_object('author', t2.author))
from library t1 left join book t2 on t1.id = t2.library_id group by t1.id;

select t1.id, t1.name, jsonb_agg(json_build_object('author', t2.author, '标题', t2.title))
from library t1 left join book t2 on t1.id = t2.library_id group by t1.id;
```

### 排查问题 
一开始以为是querydsl的问题, 但debug了一段时间代码发现，输出的HQL其实是对的。是hql在解析成sql的时候，**book没有像 select的时候自动展开**
可以看出问题是出在了hibernate的HQL解析上，而不是querydsl上。


### 经过多天的查看hibernate的源码（挺有趣）
最后找到的解决办法是，PostgreSQLDialect方法找到，其实一些postgresql 自己的函数，他其实是通过 initializeFunctionRegistry
注册到hibernate里面的。其实就是模仿 PostgreSQLMinMaxFunction的写法。然后在自定义方言(CustomPostgreSQLDialect)。
public void render(SqlAppender sqlAppender, List<? extends SqlAstNode> sqlAstArguments, SqlAstTranslator<?> walker);
SqlAppender 其实就是最终要生成的sql
sqlAstArguments 当前节点的参数列表
walker:  sqlAstArgument.accept(walker);  每个节点自己处理
这里就是一个典型的visitor模式。对自己感兴趣的节点进行操作，不感兴趣的，调用默认处理

#### 源码解析
- CustomPostgreSQLDialect这个很简单，就是覆盖initializeFunctionRegistry 注册更多的函数
- JsonbAggFunction (jsonb_agg) 函数也很简单
```
// 当遇到jsonb_agg()的时候，往sql里面加拼接上 jsonb_agg(_)  中间的参数,让中间的自己处理也就是sqlAstArgument.accept(walker);
sqlAppender.append("jsonb_agg(");
    for (SqlAstNode sqlAstArgument : sqlAstArguments) {
        sqlAstArgument.accept(walker);
    }
sqlAppender.append(")");
```
- JsonBuildFunction (json_build_object)
  这个函数相对复杂点, 它需要将 json_build_object(book)的HQL ->  json_build_object('author', author, 'title', title)  这样的sql
  也可以是是 json_build_object(book.author) 的 HQL ->  json_build_object('author', author)  这样的sql 
   还可以支持as如 json_build_object(book.author as 作者) -> json_build_object('作者', author)  这样的sql
```
        sqlAppender.append("json_build_object(");
        SqlAstNode sqlAstNode = sqlAstArguments.get(0);
        if (sqlAstNode instanceof EntityValuedPathInterpretation<?> entityValue) {    // 这种的话是第一种
            String qualifier = entityValue.getSqlExpression().getColumnReference().getQualifier();
            ModelPartContainer modelPart = entityValue.getTableGroup().getModelPart();
            if (modelPart instanceof EntityMappingType mapping) {
                AttributeMappingsList attributeMappings = mapping.getAttributeMappings();
                for(int i = 0; i < attributeMappings.size(); i++){
                    if (attributeMappings.get(i) instanceof BasicAttributeMapping basicAttribute) {
                        new QueryLiteral<>(basicAttribute.getAttributeName(), stringType).accept(walker);
                        sqlAppender.append(",");
                        String value = qualifier + "." + basicAttribute.getSelectableName();
                        sqlAppender.append(value);
                        if(i != attributeMappings.size() - 1){
                            sqlAppender.append(",");
                        }
                    }
                }
            }
        } else {            //这种的话是第二种和第三种
            boolean hasAlias = false;  // 记录是否有别名
            for(int i = 0; i < sqlAstArguments.size(); i++){
                SqlAstNode sqlAstArgument = sqlAstArguments.get(i);
                if (sqlAstArgument instanceof QueryLiteral) {
                    sqlAstArgument.accept(walker);
                    hasAlias = true;
                }
                if (sqlAstArgument instanceof BasicValuedPathInterpretation<?> basicValue) {
                    if (hasAlias) {
                        hasAlias = false;  // 重置为没有别名
                    }else {                 // 没有别名的话，使用实体字段名
                        new QueryLiteral<>(basicValue.getExpressionType().getPartName(), stringType).accept(walker);
                        sqlAppender.append(",");
                    }
                    basicValue.accept(walker);
                }
                if(i != sqlAstArguments.size() - 1){
                    sqlAppender.append(",");
                }
            }
        }
        sqlAppender.append(")");

```
-  CustomExpressions 这个是querydsl 支持，方便每次使用,也比较简单


### 总结 
- [源码的地址] [querydsl-postgresql-json] 
- 学习过程中，了解很多hibernate的东西,也了解到了 antlr4,有空也写点好玩的
- 写这个的过程其实花的时间不多，更多的时间是花在了找到问题，和修改加入逻辑的地点。 这种sql转换hibernate的代码确实很复杂，一点点debug看得头都晕了。

[init.sql]: https://github.com/happen-w/querydsl-postgresql-json/blob/main/src/test/resources/init.sql
[querydsl-postgresql-json]: https://github.com/happen-w/querydsl-postgresql-json



