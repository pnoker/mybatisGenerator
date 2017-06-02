# mybatis-generator-core-oracle

主要针对Oracle数据库定制，添加功能列表如下：
- 分页
- 自增
- 返回自增ID

### 添加分页功能
修改文件（文件位置：src\main\java\org\mybatis\generator\codegen\mybatis3\xmlmapper\elements\）
- SelectByExampleWithBLOBsElementGenerator.java
- SelectByExampleWithoutBLOBsElementGenerator.java

添加内容以`SelectByExampleWithBLOBsElementGenerator.java`文件为例，`SelectByExampleWithoutBLOBsElementGenerator.java`文件的修改相同：

```java
#48行，请参考具体文件内容
XmlElement ifElement = new XmlElement("if");
ifElement.addAttribute(new Attribute("test", "start != 0 or limit != 0"));
ifElement.addElement(new TextElement("select * from ( select * from ("));
```

```java
#85行，请参考具体文件内容
ifElement = new XmlElement("if");
ifElement.addAttribute(new Attribute("test", "start != 0 or limit != 0"));
ifElement.addElement(new TextElement(") A where A.RN &lt;= #{limit} ) B where B.RN &gt; #{start}"));
answer.addElement(ifElement);
```



