# mybatis-generator-core-oracle

## mybatis gererator 针对 Oracle 定制，实现分页，ID自增，返回ID等功能。

主要针对Oracle数据库定制，添加功能列表如下：
- 分页
- 自增
- 返回自增ID

### 添加分页功能
**1** 修改`Mapping`文件（文件位置：src\main\java\org\mybatis\generator\codegen\mybatis3\xmlmapper\elements\）

- SelectByExampleWithBLOBsElementGenerator.java
- SelectByExampleWithoutBLOBsElementGenerator.java

**1.1** 添加内容以`SelectByExampleWithBLOBsElementGenerator.java`文件为例，`SelectByExampleWithoutBLOBsElementGenerator.java`文件的修改相同：

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

**1.2** 修改`' as QUERYID,` 为 `' as QUERYID,ROWNUM as RN,`，使用ROWNUM进行分页
```java
        StringBuilder sb = new StringBuilder();
        if (stringHasValue(introspectedTable
                .getSelectByExampleQueryId())) {
            sb.append('\'');
            sb.append(introspectedTable.getSelectByExampleQueryId());
            sb.append("' as QUERYID,ROWNUM as RN,"); //$NON-NLS-1$
            answer.addElement(new TextElement(sb.toString()));
        }
```

**1.3** 生成Mapping.xml文件中select语句为：

```xml
<select id="selectByExampleWithBLOBs" resultMap="ResultMapWithBLOBs" parameterType="com.cmcc.emp.model.AlgoExample" >
    <if test="start != 0 or limit != 0" >
      select * from ( select * from (
    </if>
    select
    <if test="distinct" >
      distinct
    </if>
    'true' as QUERYID,ROWNUM as RN,
    <include refid="Base_Column_List" />
    ,
    <include refid="Blob_Column_List" />
    from EMP_ALGO
    <if test="_parameter != null" >
      <include refid="Example_Where_Clause" />
    </if>
    <if test="orderByClause != null" >
      order by ${orderByClause}
    </if>
    <if test="start != 0 or limit != 0" >
      ) A where A.RN &lt;= #{limit} ) B where B.RN &gt; #{start}
    </if>
  </select>
```

**2** 修改`model`文件（文件位置：src\main\java\org\mybatis\generator\codegen\mybatis3\model\）

- ExampleGenerator.java

**2.1** 添加构造方法`public XxxExample(int start, int limit)`

```java
#73行，请参考具体文件内容
        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setConstructor(true);
        method.setName(type.getShortName());
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getIntInstance(), "start"));
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getIntInstance(), "limit"));
        method.addBodyLine("this.start = start;\n" +
                "        this.limit = limit;\n" +
                "        oredCriteria = new ArrayList<Criteria>();");

        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);
```

**2.2** 添加参数：`protected int start;` 和 `protected int limit;`

```java
#140行，请参考具体文件内容
        field = new Field();
        field.setVisibility(JavaVisibility.PROTECTED);
        field.setType(FullyQualifiedJavaType.getIntInstance());
        field.setName("start");
        commentGenerator.addFieldComment(field, introspectedTable);
        topLevelClass.addField(field);

        field = new Field();
        field.setVisibility(JavaVisibility.PROTECTED);
        field.setType(FullyQualifiedJavaType.getIntInstance());
        field.setName("limit");
        commentGenerator.addFieldComment(field, introspectedTable);
        topLevelClass.addField(field);
```

**2.3** 生成Model文件中内容为，和上面的select语句中的start、limit关联上了：

```java
...
protected int start;

    protected int limit;

    protected List<Criteria> oredCriteria;

    public AlgoExample() {
        oredCriteria = new ArrayList<Criteria>();
    }

    public AlgoExample(int start, int limit) {
        this.start = start;
        this.limit = limit;
        oredCriteria = new ArrayList<Criteria>();
    }
...
```

### 添加自增功能

Oracle数据库同SQL Server 、MySQL等数据库不一样，不能直接设置某个字段为自增的，需要手动创建序列（Sequence）和触发器（Trigger），实现列的自增

由于是使用Mybatis操作Oracle数据库，所以在进行建表的时候只需要建立序列（Sequence）就行，触发器（Trigger)不用创建，触发器的创建将整合到Mybatis Mapping文件中去，详情如下：

**1** 修改`generatorConfig.xml`文件（文件位置：为Mybatis Generator的配置文件）

说明：Oracle中对大小写不敏感，小写一律会自动变成大写字母的

- generatorConfig.xml

```xml
<table tableName="EMP_ALGO" domainObjectName="Algo" enableCountByExample="true"
    enableUpdateByExample="true" enableDeleteByExample="true"
    enableSelectByExample="true" selectByExampleQueryId="true">
    <generatedKey column="id" sqlStatement="SELECT ALGO_SEQ_ID.nextval AS id FROM DUAL"/>
</table>
```

**[注]** `ALGO_SEQ_ID`为表`EMP_ALGO`的序列（Sequence）名称，务必保持一致

- 建表 - > `EMP_ALGO`

```sql
CREATE TABLE EMP_ALGO
(
  id          NUMBER(11) NOT NULL,
  name        VARCHAR2(32),
  path        VARCHAR2(64),
  param       CLOB,
  description VARCHAR2(64),
  at_time     TIMESTAMP(6)
)
TABLESPACE EMP
PCTFREE 10
INITRANS 1
MAXTRANS 255
STORAGE
(
INITIAL 64K
MINEXTENTS 1
MAXEXTENTS UNLIMITED
);

ALTER TABLE EMP_ALGO
  ADD CONSTRAINT EMP_ALGO_PK PRIMARY KEY (ID)
  USING INDEX
  TABLESPACE EMP
  PCTFREE 10
  INITRANS 2
  MAXTRANS 255
  STORAGE
  (
  INITIAL 64K
  MINEXTENTS 1
  MAXEXTENTS UNLIMITED
  );
```

- 创建序列（Sequence） - > `ALGO_SEQ_ID`

```sql
CREATE SEQUENCE ALGO_SEQ_ID
MINVALUE 1
MAXVALUE 9999999999999999999999999999
START WITH 1
INCREMENT BY 1
CACHE 20;
```

**2** 生成Mapping.xml文件中insert和nsertSelective（此处省）语句为：

```sql
<insert id="insert" parameterType="com.cmcc.emp.model.Algo" >
    <selectKey resultType="java.lang.Long" keyProperty="id" order="BEFORE" >
      SELECT ALGO_SEQ_ID.nextval AS id FROM DUAL
    </selectKey>
    insert into EMP_ALGO (ID, NAME, PATH, 
      DESCRIPTION, AT_TIME, PARAM
      )
    values (#{id,jdbcType=DECIMAL}, #{name,jdbcType=VARCHAR}, #{path,jdbcType=VARCHAR}, 
      #{description,jdbcType=VARCHAR}, #{atTime,jdbcType=TIMESTAMP}, #{param,jdbcType=CLOB}
      )
</insert>
```

### 获取自增后返回的ID

```java
Algo algo = new Algo();
...
//不用setId(xxx),插表操作后会自动返回ID
algo.setName("xxx");
algo.setAtTime(new Date());
...
algoService.insertSelective(algo);
//获取返回的ID，直接getID()即可
logger.info("自增后ID为：" + algo.getId());
```

## 重要

推荐工具：git、idea 

### 1.如何导入该工程
Clone该工程到项目文件夹下

```bash
git clone https://github.com/pnoker/mybatisGenerator.git
```

idea导入maven

`File` -> `New` -> `Project From Existing Sources` -> 选择`mybatisGenerator`

选择自动导入依赖的jar

### 2.如何生成mybatis-generator-core-oracle.jar和mybatis-generator-core-oracle-sources.jar

在IDE上修改完代码后，在项目根目录下命令行执行：`mvn inseall`,当出错是使用：`mvn clean`进行清空，于是会在target文件下生成两个jar包：

- mybatis-generator-core-oracle-1.3.2.jar，需要使用的jar
- mybatis-generator-core-oracle-1.3.2-sources.jar，源码

### 3.如何生成Dao、Model、Mapping

```bash
java -jar src/main/webapp/WEB-INF/lib/mybatis-generator-core-oracle-1.3.2.jar -configfile generatorConfig.xml -overwrite
```





