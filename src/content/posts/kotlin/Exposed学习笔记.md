---
title: Exposed框架学习笔记
published: 2025-06-12 15:11:00
description: ''
tags: [ORM, Kotlin]
category: 学习笔记
draft: false
---
# What
[官方文档链接](https://www.jetbrains.com/help/exposed/home.html)

Exposed是Kotlin的一个SQL库，他是：
- 用于构建查询的类型安全DSL
- ORM风格交互
- DAO框架

# Why
- 轻量级ORM，将数据库模式直观的映射到Kotlin对象（类似Spring-Data-JPA）
  ```kotlin
  // Create
  object Passenger : IntIdTable("passengers") {
      val name = text("name")
      val email = text("email").uniqueIndex()
  }
  
  object TaxiRide : Table("taxi_rides") {
  
    val rideId = integer("ride_id").autoIncrement()
    val passengerId = reference("passenger_id", Passenger.id)
    
    val pickupLocation = text("pickup_location")
    val price = decimal("price", 10, 2)
    
    val status = text("status").nullable().default("requested")
    
    override val primaryKey = PrimaryKey(rideId)
  }
  ```
- 类型安全的查询，防止一些错误的产生
  ```kotlin
  // Total Revenue by Passenger
  Passenger.join(
    otherTable = TaxiRide,
    joinType = JoinType.INNER,
    onColumn = Passenger.id,
    otherColumn = TaxiRide.passengerId
  )
    .select(Passenger.name, TaxiRide.price.sum().alias("total_revenue"))
    .where(TaxiRide.status eq "completed")
    .groupBy(Passenger.name)
    .orderBy(TaxiRide.price.sum() to SortOrder.DESC)
    .limit(3)
  ```
- CRUD操作，开箱即用的进行CRUD操作
  ```kotlin
  // Create
  val id = Passenger.insertAndGetId {
    it[name] = "John Doe"
    it[email] = "john.doe@example.com"
  }
    
  // Read
  val allPassengerIds = Passenger.selectAll().map { it[Passenger.id] }
    
  // Update
  TaxiRide.upsert(TaxiRide.passengerId, TaxiRide.pickupLocation) {
    it[passengerId] = id
    it[pickupLocation] = "917 Cedar Ave"
    
    
    it[price] = BigDecimal("24.40")
    it[status] = "accepted"
  }
    
  // Delete
  TaxiRide.deleteWhere { passengerId inList allPassengerIds }
  ```
- 查询构建器，安全、可重用、动态的构建查询
  ```kotlin
  fun buildQuery(): Query {
   val query = TaxiRide.selectAll()
   query.addRideFilters("completed", "John Doe")
   return query
  }
    
  fun Query.addRideFilters(status: String?, passengerName: String?) {
    status?.let {
    this.andWhere { TaxiRide.status eq status }
  }
    
  passengerName?.let {
    this.adjustColumnSet { // add a JOIN to Passenger
      innerJoin(Passenger, { Passenger.id }, { TaxiRide.passengerId })
      }.andWhere { Passenger.name eq passengerName }
    }
  }
  ```
- 框架友好，Exposed可与任何框架配合使用，并内置支持Spring Boot和Ktor
- 开箱即用，不受基本数据库类型和函数限制，支持JSON等流行类型，并允许自定义类型和和函数
- 数据库无关，支持多种流行数据库：PostgreSQL、MySQL、SQLite、Oracle、H2等
# How
## 引入dependencies
- `org.jetbrains.exposed:exposed-core`
  包含类型安全操作数据库所有的基础组建和抽象，并包含DSL API
- `org.jetbrains.exposed:exposed-jdbc`
  是exposed-core的一个扩展，添加了对JDBC的支持
- `com.h2database:h2`
  只是一个数据库驱动的例子，根据实际使用的数据库替换
## 配置数据库连接
使用Database.connect()函数
```kotlin
package org.example

import org.jetbrains.exposed.v1.jdbc.Database

fun main() {
    // jdbc:h2:mem:test为数据库连接字符串，org.h2.Driver为驱动ClassName
    Database.connect("jdbc:h2:mem:test", driver = "org.h2.Driver")
}
```
## 定义表对象
在Exposed中，通过集成Table类来定义。
```kotlin
import org.jetbrains.exposed.v1.core.Table

const val MAX_VARCHAR_LENGTH = 128

object Tasks : Table("tasks") {
    val id = integer("id").autoIncrement()
    val title = varchar("name", MAX_VARCHAR_LENGTH)
    val description = varchar("description", MAX_VARCHAR_LENGTH)
    val isCompleted = bool("completed").default(false)
}
```
## 创建和查询表
```kotlin
package org.example

import Tasks
import org.jetbrains.exposed.v1.core.*
import org.jetbrains.exposed.v1.core.SqlExpressionBuilder.eq
import org.jetbrains.exposed.v1.jdbc.Database
import org.jetbrains.exposed.v1.jdbc.SchemaUtils
import org.jetbrains.exposed.v1.jdbc.addLogger
import org.jetbrains.exposed.v1.jdbc.deleteWhere
import org.jetbrains.exposed.v1.jdbc.insert
import org.jetbrains.exposed.v1.jdbc.select
import org.jetbrains.exposed.v1.jdbc.selectAll
import org.jetbrains.exposed.v1.jdbc.transactions.transaction
import org.jetbrains.exposed.v1.jdbc.update

fun main() {
    Database.connect("jdbc:h2:mem:test", driver = "org.h2.Driver")

    transaction {
        SchemaUtils.create(Tasks)

        val taskId = Tasks.insert {
            it[title] = "Learn Exposed"
            it[description] = "Go through the Get started with Exposed tutorial"
        } get Tasks.id

        val secondTaskId = Tasks.insert {
            it[title] = "Read The Hobbit"
            it[description] = "Read the first two chapters of The Hobbit"
            it[isCompleted] = true
        } get Tasks.id

        println("Created new tasks with ids $taskId and $secondTaskId.")

        Tasks.select(Tasks.id.count(), Tasks.isCompleted).groupBy(Tasks.isCompleted).forEach {
            println("${it[Tasks.isCompleted]}: ${it[Tasks.id.count()]} ")
        }
    }
}
```
## 更新和删除
```kotlin
transaction {
    // ...

    // Update a task
    Tasks.update({ Tasks.id eq taskId }) {
        it[isCompleted] = true
    }

    val updatedTask = Tasks.select(Tasks.isCompleted).where(Tasks.id eq taskId).single()

    println("Updated task details: $updatedTask")

    // Delete a task
    Tasks.deleteWhere { id eq secondTaskId }

    println("Remaining tasks: ${Tasks.selectAll().toList()}")
}
```
## 启用日志
```kotlin
transaction {
    // print sql to std-out
    addLogger(StdOutSqlLogger)

    // ...
}
```
## 模块
### Core modules
| Module       | Function  |
|--------------|-----------|
| exposed-core | 基础组建和抽象   |
| exposed-dao  | DAO API   |
| exposed-jdbc | JDBC支持    |
### Extension modules
| Module                      | Function                          |
|-----------------------------|-----------------------------------|
| exposed-crypt               | 增加列类型以存储encrypted数据               |
| exposed-java-time           | 基于Java8时间API的日期时间扩展               |
| exposed-jodatime            | 基于Joda-Time库的日期时间扩展               |
| exposed-json                | JSON和JSONB数据类型扩展                  |
| exposed-kotlin-datetime     | 基于kotlinx-datetime库的日期时间扩展        |
| exposed-money               | 扩展支持JavaMoney API中的MonetaryAmount |
| exposed-spring-boot-starter | 使用Exposed替代Hibernate              |
| spring-transaction          | spring事物管理器                       |
| exposed-migration           | 提供支持数据库迁移的工具                      |
# 连接池
`Database.connect`方法可以传入一个`javax.sql.DataSource`，并设置参数。
```kotlin
val config = HikariConfig().apply {
    jdbcUrl = "jdbc:mysql://localhost/dbname"
    driverClassName = "com.mysql.cj.jdbc.Driver"
    username = "username"
    password = "password"
    maximumPoolSize = 6
    // as of version 0.46.0, if these options are set here, they do not need to be duplicated in DatabaseConfig
    isReadOnly = false
    transactionIsolation = "TRANSACTION_SERIALIZABLE"
}

val dataSource = HikariDataSource(config)

Database.connect(
    datasource = dataSource,
    databaseConfig = DatabaseConfig {
        // set other parameters here
    }
)
```
# Schema相关
## 表定义
### 表名
默认情况，Exposed会根据完整的类名生成表名.
```kotlin
object StarWarsFilmsTable : Table() {
    val id = integer("id").autoIncrement()
    val sequelId = integer("sequel_id").uniqueIndex()
    val name = varchar("name", MAX_VARCHAR_LENGTH)
    val director = varchar("director", MAX_VARCHAR_LENGTH)
}
```
```sql
CREATE TABLE IF NOT EXISTS STARWARSFILMS
(ID INT AUTO_INCREMENT NOT NULL,
 SEQUEL_ID INT NOT NULL,
 "name" VARCHAR(50) NOT NULL,
 DIRECTOR VARCHAR(50) NOT NULL)
```
如果需要指定，可以传递给Table构造函数的name参数.
```kotlin
object CustomStarWarsFilmsTable : Table("all_star_wars_films") {
    val id = integer("id").autoIncrement()
    val sequelId = integer("sequel_id").uniqueIndex()
    val name = varchar("name", MAX_VARCHAR_LENGTH)
    val director = varchar("director", MAX_VARCHAR_LENGTH)
}
```
```sql
CREATE TABLE IF NOT EXISTS ALL_STAR_WARS_FILMS
(ID INT AUTO_INCREMENT NOT NULL,
SEQUEL_ID INT NOT NULL,
"name" VARCHAR(50) NOT NULL,
DIRECTOR VARCHAR(50) NOT NULL)
```
## 约束
### Nullable
```kotlin
// SQL: POPULATION INT NULL
val population: Column<Int?> = integer("population").nullable()
```
### Default
```kotlin
// SQL: "NAME" VARCHAR(50) DEFAULT 'Unknown'
val name: Column<String> = varchar("name", 50).default("Unknown")
```
```kotlin
val name: Column<String> = varchar("name", 50).databaseGenerated()
```
### Index
```kotlin
val name = varchar("name", 50).index()
```
```kotlin
val indexName = index("indexName", false, *arrayOf(name, population))
// or inside an init block within the table object
init {
    index("indexName", isUnique = false, name, population)
}
```
### Unique
```kotlin
val name = varchar("name", 50).uniqueIndex()
```
### Primary Key
```kotlin
override val primaryKey = PrimaryKey(name, name = "Cities_name")
```
```sql
CONSTRAINT Cities_name PRIMARY KEY ("name")
```
如果继承IdTable，而不是Table的类会自动定义primaryKey字段。

对于联合主键，可以继承CompositeIdTable，然后使用`.entityId()`来标识。
```kotlin
object Towns : CompositeIdTable("towns") {
    val areaCode = integer("area_code").autoIncrement().entityId()
    val latitude = decimal("latitude", 9, 6).entityId()
    val longitude = decimal("longitude", 9, 6).entityId()
    val name = varchar("name", 32)

    override val primaryKey = PrimaryKey(areaCode, latitude, longitude)
}
```
如果需要引用另外一个IdTable的主键，可以通过`addIdColumn()`的方式实现。
```kotlin
object AreaCodes : IdTable<Int>("area_codes") {
    override val id = integer("code").entityId()
    override val primaryKey = PrimaryKey(id)
}

object Towns : CompositeIdTable("towns") {
    val areaCode = reference("area_code", AreaCodes)
    val latitude = decimal("latitude", 9, 6).entityId()
    val longitude = decimal("longitude", 9, 6).entityId()
    val name = varchar("name", 32)

    init {
        addIdColumn(areaCode)
    }

    override val primaryKey = PrimaryKey(areaCode, latitude, longitude)
}
```
### 外键
```kotlin
object Citizens : IntIdTable() {
    val name = varchar("name", 50)
    val city = optReference("city", Cities.name, onDelete = ReferenceOption.CASCADE)
}
```
对于多列主键：
```kotlin
object Citizens : IntIdTable() {
    val name = varchar("name", 50)
    val cityId = integer("city_id")
    val cityName = varchar("city_name", 50)

    init {
        foreignKey(cityId, cityName, target = Cities.primaryKey)
    }
}
```
### Check
```kotlin
// SQL: CONSTRAINT check_Cities_0 CHECK (REGEXP_LIKE("NAME", '^[A-Z].*', 'c')))
val name = varchar("name", 50).check { it regexp "^[A-Z].*" }
```
## 数据类型
### 基础数据类型
- integer()
- short()
- long()
- float()
- decimal()
- bool
- char()
- varchar()
- text()
### Array
基础使用
```kotlin
// Simple array table definition
object SimpleArrays : Table("teams") {
    val memberIds = array<UUID>("member_ids")
    val memberNames = array<String>("member_names")
    val budgets = array<Double>("budgets")
}

// Array with explicit column type table definition
object AdvancedArrays : Table("teams") {
    val memberNames = array<String>("member_names", VarCharColumnType(colLength = MEMBER_NAME_LENGTH))
    val deadlines = array<LocalDate>("deadlines", KotlinLocalDateColumnType()).nullable()
    val expenses = array<Double?>("expenses", DoubleColumnType()).default(emptyList())
}
// insert array data
SimpleArrays.insert {
    it[memberIds] = List(TEAM_SIZE) { UUID.randomUUID() }
    it[memberNames] = List(TEAM_SIZE) { i -> "Member ${('A' + i)}" }
    it[budgets] = listOf(DEFAULT_BUDGET)
}
```
多维数组
```kotlin
// Multidimensional array table definition
object MultiDimArrays : Table("teams") {
    val nestedMemberIds = array<UUID, List<List<UUID>>>(
        "nested_member_ids", dimensions = 2
    )
    val hierarchicalMemberNames = array<String, List<List<List<String>>>>(
        "hierarchical_member_names",
        VarCharColumnType(colLength = MEMBER_NAME_LENGTH),
        dimensions = 3
    )
}
```
数组索引
```kotlin
val firstMember = SimpleArrays.memberIds[FIRST_MEMBER_INDEX]
SimpleArrays
    .select(firstMember)
    .where { SimpleArrays.budgets[FIRST_MEMBER_INDEX] greater MIN_BUDGET }
```
数组切片
```kotlin
SimpleArrays.select(SimpleArrays.memberNames.slice(SLICE_START, SLICE_END_SMALL))
```
ANY和ALL运算法
```kotlin
SimpleArrays
    .selectAll()
    .where { SimpleArrays.budgets[FIRST_MEMBER_INDEX] lessEq allFrom(SimpleArrays.budgets) }

SimpleArrays
    .selectAll()
    .where { stringParam("Member A") eq anyFrom(SimpleArrays.memberNames.slice(SLICE_START, SLICE_END_LARGE)) }
```
### 二进制
- blob()
- binary()
```kotlin
object Files : Table() {
    val id = integer("id").autoIncrement()
    val name = varchar("name", 255)
    val content = blob("content")
    val simpleData = binary("simple_data") // simple version without length
    val thumbnail = binary("thumbnail", 1024) // length-specified version

    override val primaryKey = PrimaryKey(id)
}
```
```kotlin
SchemaUtils.create(Files)
        // Store binary data
        Files.insert {
            it[name] = "example.txt"
            it[content] = ExposedBlob("Hello, World!".toByteArray())
            it[simpleData] = "simple binary data".toByteArray() // using simple binary version
            it[thumbnail] = "thumbnail data".toByteArray() // using length-specified version
        }

        // Read binary data
        val file = Files.selectAll().first()
        val content = file[Files.content]
        val text = String(content.bytes) // for small files
        // or
        val text2 = content.inputStream.bufferedReader().readText() // for large files

        // Read both types of binary data
        val simpleBytes = file[Files.simpleData] // returns ByteArray from simple binary
        val thumbnailBytes = file[Files.thumbnail] // returns ByteArray from length-specified binary
        println("$text, $text2")
}
```
### 枚举
- enumeration() 映射到数据库的`INT`类型
- enumerationByName() 映射到数据库的`VARCHAR`，使用枚举的名称
- customEnumeration() 处理数据库特定的枚举类型，支持Mysql、PostgreSQL和H2

```kotlin
enum class Foo { BAR, BAZ }

private const val ENUM_NAME_COLUMN_LENGTH = 10

object BasicEnumTable : Table() {
    val enumOrdinal = enumeration("enumOrdinal", Foo::class)
}

object NamedEnumTable : Table() {
    val enumName = enumerationByName("enumName", ENUM_NAME_COLUMN_LENGTH, Foo::class)
}
```
```kotlin
// Insert enum value
BasicEnumTable.insert {
    it[enumOrdinal] = Foo.BAR
}

// Insert enum value
NamedEnumTable.insert {
    it[enumName] = Foo.BAZ
}
```
### 日期和时间
- date()
- datetime()
- time()
- timestamp()
- timestampWithTimeZone()

```kotlin
import org.jetbrains.exposed.v1.core.Table

// For exposed-kotlin-datetime (recommended)
import org.jetbrains.exposed.v1.kotlin.datetime.*

// For exposed-java-time
import org.jetbrains.exposed.v1.javatime.*

// For exposed-jodatime
import org.jetbrains.exposed.v1.jodatime.*

object Events : Table() {
    val id = integer("id").autoIncrement()
    val name = varchar("name", 50)
    val startDate = date("start_date")                                    // maps to LocalDate
    val startTime = time("start_time")                                    // maps to LocalTime
    val createdAt = datetime("created_at")                               // maps to LocalDateTime
        .defaultExpression(CurrentDateTime)
    val updatedAt = timestamp("updated_at")                              // maps to Instant
    val scheduledAt = timestampWithTimeZone("scheduled_at")              // maps to OffsetDateTime

    override val primaryKey = PrimaryKey(id)
}
```
### JSON
- json()
- jsonb()
```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json
import org.jetbrains.exposed.v1.core.Table
import org.jetbrains.exposed.v1.jdbc.update

const val GROUP_ID_LENGTH = 32

@Serializable
data class Project(val name: String, val language: String, val active: Boolean)

val format = Json { prettyPrint = true }

object TeamsTable : Table("team") {
    val groupId = varchar("group_id", GROUP_ID_LENGTH)
    val project = json<Project>("project", format) // equivalent to json("project", format, Project.serializer())
} 
```
### 列转换
列转换允许定义数据列类型和应用程序数据类型的转换。
```kotlin
enum class Meal {
    BREAKFAST,
    LUNCH,
    DINNER
}

object Meals : Table() {
    val mealTime: Column<Meal> = time("meal_time")
        .transform(
            wrap = {
                when {
                    it.hour < 10 -> Meal.BREAKFAST
                    it.hour < 15 -> Meal.LUNCH
                    else -> Meal.DINNER
                }
            },
            unwrap = {
                when (it) {
                    Meal.BREAKFAST -> LocalTime(8, 0)
                    Meal.LUNCH -> LocalTime(12, 0)
                    Meal.DINNER -> LocalTime(18, 0)
                }
            }
        )
}
```
Transformation也可以定义为ColumnTransformer接口的实现。
```kotlin
class MealTimeTransformer : ColumnTransformer<LocalTime, Meal> {
    override fun wrap(value: LocalTime): Meal = when {
        value.hour < 10 -> Meal.BREAKFAST
        value.hour < 15 -> Meal.LUNCH
        else -> Meal.DINNER
    }

    override fun unwrap(value: Meal): LocalTime = when (value) {
        Meal.BREAKFAST -> LocalTime(8, 0)
        Meal.LUNCH -> LocalTime(12, 0)
        Meal.DINNER -> LocalTime(18, 0)
    }
}

object Meals : Table() {
    val mealTime: Column<Meal> = time("meal_time").transform(MealTimeTransformer())
}
```
## sql函数
Exposed支持classic SQL函数
```kotlin
val lowerCaseTitle = FilmBoxOfficeTable.title.lowerCase()

val upperCaseRegion = FilmBoxOfficeTable.region.upperCase().alias("reg_all_caps")

val fullFilmTitle = Concat(separator = " ", FilmBoxOfficeTable.region, stringLiteral("||"), FilmBoxOfficeTable.title)
    .trim()
    .lowerCase()
```
## 处理Schema
### 定义
```kotlin
val schema = Schema("my_schema") // my_schema is the schema name.

val schema = Schema("my_schema", authorization = "owner")
```
## 创建
```kotlin
SchemaUtils.createSchema(schema)
```
## 默认Schema
```kotlin
SchemaUtils.setSchema(schema)
```
## 删除
```kotlin
SchemaUtils.dropSchema(schema)
```
# Migrations
与Hibernate，exposed也提供了迁移功能，需要引入`org.jetbrains.exposed:exposed-migration`。

## 同步schema
- 仅生成缺少的列
  ```kotlin
  val missingColStatements = SchemaUtils.addMissingColumnsStatements(
    UsersTable
  )
  ```
- 生成数据库迁移所需的所有语句
  ```kotlin
  val statements = MigrationUtils.statementsRequiredForDatabaseMigration(
    UsersTable
  )
  ```
- 生成迁移脚本
  ```kotlin
  MigrationUtils.generateMigrationScript(
    UsersTable,
    scriptDirectory = MIGRATIONS_DIRECTORY,
    scriptName = "V2__Add_primary_key",
  )
  ```
## 验证
### 存在
- Table.exists()
- Sequence.exists()
- Schema.exists()
## 结构
- SchemaUtils.checkExcessiveIndices()
- SchemaUtils.checkExcessiveForeignKeyConstraints()
## 元数据
- currentDialect.tableColumns()
- currentDialect.existingIndices()
- currentDialect.existingPrimaryKeys()
# 事务
在Exposed中，CRUD操作必须用`transaction`函数块里面。
## 使用返回值
```kotlin
val jamesList = transaction {
    Users.selectAll().where { Users.firstName eq "James" }.toList()
}
// jamesList is now a List<ResultRow> containing Users data
```
`Blob`和`text`类型字段，如果不加载，是不能在事务外使用的。
但对于`text`字段，可以在定义表时设置`eagerLoading`参数，使字段在事务之外可用。
```kotlin
// without eagerLoading
val idsAndContent = transaction {
   Documents.selectAll().limit(10).map { it[Documents.id] to it[Documents.content] }
}

// with eagerLoading for text fields
object Documents : Table() {
  //...
  val content = text("content", eagerLoading = true)
}

val documentsWithContent = transaction {
   Documents.selectAll().limit(10)
}
```
## 使用多个数据库
```kotlin
val db1 = connect("jdbc:h2:mem:db1;DB_CLOSE_DELAY=-1;", "org.h2.Driver", "root", "")
val db2 = connect("jdbc:h2:mem:db2;DB_CLOSE_DELAY=-1;", "org.h2.Driver", "root", "")
transaction(db1) {
   //...
   val result = transaction(db2) {
      Table1.selectAll().where { }.map { it[Table1.name] }
   }

   val count = Table2.selectAll().where { Table2.name inList result }.count()
}
```
## 设置默认数据库
```kotlin
val db = Database.connect()
TransactionManager.defaultDatabase = db
```
## 使用嵌套事务
默认情况，内部事务发生回归，父事务也会回滚。
```kotlin
val db = Database.connect()
db.useNestedTransactions = false // Default setting

transaction {
    println("Transaction # ${this.id}") // Transaction # 1
    FooTable.insert{ it[id] = 1 }
    println(FooTable.selectAll().count()) // 1

    transaction {
        println("Transaction # ${this.id}") // Transaction # 1
        FooTable.insert{ it[id] = 2 }
        println(FooTable.selectAll().count()) // 2

        rollback()
    }

    println(FooTable.selectAll().count()) // 0
}
```
## 独立嵌套事务
设置`Database`的`useNestedTransactions=true`允许嵌套事务独立运行。
```kotlin
val db = Database.connect(
    // connection parameters
)
db.useNestedTransactions = true

transaction {
    println("Transaction # ${this.id}") // Transaction # 1
    FooTable.insert{ it[id] = 1 }
    println(FooTable.selectAll().count()) // 1

    transaction {
        println("Transaction # ${this.id}") // Transaction # 2
        FooTable.insert{ it[id] = 2 }
        println(FooTable.selectAll().count()) // 2

        rollback()
    }

    println(FooTable.selectAll().count()) // 1
}
```
## 保存点
```kotlin
transaction {
    TestTable.insert { it[amount] = 99 }
    val firstInsert = connection.setSavepoint("first_insert")
    TestTable.insert { it[amount] = 100 }
    connection.rollback(firstInsert)
}

transaction {
    // If the savepoint was not set in above tx, all inserts would roll back, returning an empty list
    println(TestTable.selectAll().map { it[TestTable.amount] }) // [99]
}
```
# 使用SQL字符串
```kotlin
val secretCode = "abc"
exec("CREATE USER IF NOT EXISTS GUEST PASSWORD '$secretCode'")
exec("GRANT ALL PRIVILEGES ON ${FilmsTable.nameInDatabaseCase()} TO GUEST")
```
## 参数化语句
```kotlin
val toWatch = exec(
    stmt = "SELECT FILMS.ID, FILMS.TITLE FROM FILMS WHERE (FILMS.NOMINATED = ?) AND (FILMS.RATING >= ?)",
    args = listOf(BooleanColumnType() to true, DoubleColumnType() to GOOD_RATING)
) { result ->
    val films = mutableListOf<Pair<Int, String>>()
    while (result.next()) {
        films += result.getInt(1) to result.getString(2)
    }
    films
}
```
## 显式指定语句类型
可以指定语句的执行类型，来防止非预期执行的发生。
```kotlin
exec(
    stmt = "DROP USER IF EXISTS GUEST",
    explicitStatementType = StatementType.DROP
)
```
类型包括：
- StatementType.SELECT
- StatementType.EXEC
- StatementType.SHOW
- StatementType.PRAGMA
# DSL
## 联表
### join
```kotlin
ActorsIntIdTable.join(
    StarWarsFilmsIntIdTable,
    JoinType.INNER,
    additionalConstraint = { StarWarsFilmsIntIdTable.sequelId eq ActorsIntIdTable.sequelId }
)
    .select(ActorsIntIdTable.name.count(), StarWarsFilmsIntIdTable.name)
    .groupBy(StarWarsFilmsIntIdTable.name)
```
对于外键，使用innerJoin更简洁
```kotlin
(ActorsIntIdTable innerJoin RolesTable)
    .select(RolesTable.characterName.count(), ActorsIntIdTable.name)
    .groupBy(ActorsIntIdTable.name)
    .toList()
```
### union
```kotlin
val lucasDirectedQuery =
    StarWarsFilmsIntIdTable.select(StarWarsFilmsIntIdTable.name).where { StarWarsFilmsIntIdTable.director eq "George Lucas" }

val abramsDirectedQuery =
    StarWarsFilmsIntIdTable.select(StarWarsFilmsIntIdTable.name).where { StarWarsFilmsIntIdTable.director eq "J.J. Abrams" }

val filmNames = lucasDirectedQuery.union(abramsDirectedQuery).map { it[StarWarsFilmsIntIdTable.name] }
```
包含重复项
```kotlin
val lucasDirectedQuery =
    StarWarsFilmsIntIdTable.select(StarWarsFilmsIntIdTable.name).where { StarWarsFilmsIntIdTable.director eq "George Lucas" }

val originalTrilogyQuery =
    StarWarsFilmsIntIdTable.select(StarWarsFilmsIntIdTable.name).where { StarWarsFilmsIntIdTable.sequelId inList (3..5) }

val allFilmNames = lucasDirectedQuery.unionAll(originalTrilogyQuery).map { it[StarWarsFilmsIntIdTable.name] }
```
## CRUD
- Create
  - .insert()
  - .insertAndGetId()
  - .insert(Query)
  - .insertIgnore()
  - .insertIgnoreAndGetId()
  - .batchInsert()
- Read
  - .select()
  - .selectAll()
  - .insertReturning()
  - .upsertReturning()
  - .updateReturning()
  - .deleteReturning()
- Update
  - .update()
  - .upsert() 插入或更新
  - .replace() 插入违反唯一约束，先删除
- Delete
  - .deleteWhere()
  - .deleteIgnoreWhere()
  - .deleteAll()
## 序列
- 定义 `val myseq = Sequence("my_sequence")`
- 创建 `SchemaUtils.createSequence(myseq)`
- 删除 `SchemaUtils.dropSequence(myseq)`
- 获取下一个值 `val nextVal = myseq.nextIntVal()`
## 查询条件
### Where
- 基本条件
  - eq
  - neq
  - isNull()
  - isNotNull()
  - less
  - lessEq
  - greater
  - greaterEq
  - exists
  - notExists
  - isDistinctFrom
  - isNotDistinctFrom
- 逻辑条件
  - and
  - or
  - not
  - andIfNotNull
  - orIfNotNull
  - compoundAnd()
  - compoundOr()
- 模式匹配条件
  - like
  - notLike
  - regexp
  - match
- 范围条件
  - between()
- 集合条件
  - inList
  - anyFrom()
  - allFrom()
### 条件Where
```kotlin
fun findWithConditionalWhere(directorName: String?, sequelId: Int?) {
    val query = StarWarsFilmsTable.selectAll()

    // Add conditions dynamically
    directorName?.let {
        query.andWhere { StarWarsFilmsTable.director eq it }
    }
    sequelId?.let {
        query.andWhere { StarWarsFilmsTable.sequelId eq it }
    }
}
```
```kotlin
fun findWithConditionalJoin(actorName: String?) {
    transaction {
        val query = StarWarsFilmsTable.selectAll() // Base query

        // Conditionally adjust the query
        actorName?.let { name ->
            query.adjustColumnSet {
                innerJoin(ActorsTable, { StarWarsFilmsTable.sequelId }, { ActorsTable.sequelId })
            }
                .adjustSelect {
                    select(StarWarsFilmsTable.columns + ActorsTable.columns)
                }
                .andWhere { ActorsTable.name eq name }
        }
    }
}
```
### 聚合和排序
- count()
- orderBy()
- groupBy()
- sum()
- average()
- min()
- max()
### 限制
- limit()
### 别名
- alias()
# DAO
## 表类型
- 自增ID
  - IntIdTable
  - LongIdTable
  - UIntIdTable
  - ULongIdTable
  - UUIDTable
- 联合Id
  - CompositeIdTable
## CRUD
### Create
```kotlin
val movie = StarWarsFilmEntity.new {
    name = "The Last Jedi"
    sequelId = MOVIE_SEQUEL_ID
    director = "Rian Johnson"
}
val movie2 = StarWarsFilmEntity.new(id = 2) {
    name = "The Rise of Skywalker"
    sequelId = MOVIE2_SEQUEL_ID
    director = "J.J. Abrams"
}
```
### Read
- all()
- find()
- findById()
读取联表
```kotlin
val query = UsersTable.innerJoin(UserRatingsTable).innerJoin(StarWarsFilmsTable)
    .select(UsersTable.columns)
    .where {
        StarWarsFilmsTable.sequelId eq MOVIE_SEQUELID and (UserRatingsTable.value greater MIN_MOVIE_RATING.toLong())
    }.withDistinct()
```
包装
```kotlin
val users = UserEntity.wrapRows(query).toList()
```
排序
- sortedBy()
- sortedByDescending()
### Update
- 直接修改值，Exposed会在“Flushing”到数据库时更新
- findByIdAndUpdate()
- findSingleByAndUpdate()
### Delete
- delete()
### 子查询
需要将查询转换为表达式
```kotlin
val expression = wrapAsExpression<Int>(
    UsersTable.select(UsersTable.id.count())
        .where { CitiesTable.id eq UsersTable.cityId }
)

val cities = CitiesTable.selectAll()
    .orderBy(expression, SortOrder.DESC)
    .toList()
```
### 自动填充
```kotlin
package org.example.entities

import kotlinx.datetime.Clock
import kotlinx.datetime.LocalDateTime
import kotlinx.datetime.TimeZone
import kotlinx.datetime.toLocalDateTime
import org.example.tables.BaseTable
import org.jetbrains.exposed.v1.core.Column
import org.jetbrains.exposed.v1.core.exposedLogger
import org.jetbrains.exposed.v1.dao.*

abstract class BaseEntityClass<out E : BaseEntity>(
    table: BaseTable
) : IntEntityClass<E>(table) {
    init {
        EntityHook.subscribe { change ->
            val changedEntity = change.toEntity(this)
            when (val type = change.changeType) {
                EntityChangeType.Updated -> {
                    val now = nowUTC()
                    changedEntity?.let {
                        if (it.writeValues[table.modified as Column<Any?>] == null) {
                            it.modified = now
                        }
                    }
                    logChange(changedEntity, type, now)
                }
                else -> logChange(changedEntity, type)
            }
        }
    }

    private fun nowUTC() = Clock.System.now().toLocalDateTime(TimeZone.UTC)

    private fun logChange(entity: E?, type: EntityChangeType, dateTime: LocalDateTime? = null) {
        entity?.let {
            val entityClassName = this::class.java.enclosingClass.simpleName
            exposedLogger.info(
                "$entityClassName(${it.id}) ${type.name.lowercase()} at ${dateTime ?: nowUTC()}"
            )
        }
    }
}
```
## 关系
### Many-to-one
```kotlin
object UsersTable : IntIdTable() {
    val name = varchar("name", MAX_USER_NAME_LENGTH)
}
class UserEntity(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<UserEntity>(UsersTable)

    val ratings by UserRatingEntity referrersOn UserRatingsTable.user orderBy UserRatingsTable.value
}

object UserRatingsTable : IntIdTable() {
    val value = long("value")
    val film = reference("film", StarWarsFilmsTable)
    val user = reference("user", UsersTable)
}
class UserRatingEntity(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<UserRatingEntity>(UserRatingsTable)

    var value by UserRatingsTable.value
    var film by StarWarsFilmEntity referencedOn UserRatingsTable.film // use referencedOn for normal references
    var user by UserEntity referencedOn UserRatingsTable.user
}
```
- Accessing data `val film = filmRating.film`
- Reverse access  `val ratings by UserRatingEntity referrersOn UserRatingsTable.film`
- Back reference `val rating by UserRatingEntity backReferencedOn UserRatingsTable.user`
### Optional
```kotlin
object UserRatingsWithOptionalUserTable : IntIdTable() {
    val value = long("value")
    val film = reference("film", StarWarsFilmsTable)
    val user = optReference("user", UsersTable)
}
class UserRatingWithOptionalUserEntity(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<UserRatingWithOptionalUserEntity>(UserRatingsWithOptionalUserTable)

    var value by UserRatingsWithOptionalUserTable.value
    var film by StarWarsFilmEntity referencedOn UserRatingsWithOptionalUserTable.film // use referencedOn for normal references
    var user by UserEntity optionalReferencedOn UserRatingsWithOptionalUserTable.user
}
```
### Ordered reference
```kotlin
class UserEntity(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<UserEntity>(UsersTable)

    var name by UsersTable.name
    val ratings by UserRatingEntity referrersOn UserRatingsTable.user orderBy UserRatingsTable.value
}
```
```kotlin
class UserOrderedEntity(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<UserOrderedEntity>(UsersTable)

    var name by UsersTable.name
    val ratings by UserRatingEntity referrersOn UserRatingsTable.user orderBy listOf(
        UserRatingsTable.value to SortOrder.DESC,
        UserRatingsTable.id to SortOrder.ASC
    )
}
```
### Many-to-many
```kotiln
object ActorsTable : IntIdTable() {
    val firstname = varchar("firstname", MAX_NAME_LENGTH)
    val lastname = varchar("lastname", MAX_NAME_LENGTH)
}
class ActorEntity(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<ActorEntity>(ActorsTable)

    var firstname by ActorsTable.firstname
    var lastname by ActorsTable.lastname
}
object StarWarsFilmActorsTable : Table() {
    val starWarsFilm = reference("starWarsFilm", StarWarsFilmsTable)
    val actor = reference("actor", ActorsTable)
    override val primaryKey = PrimaryKey(starWarsFilm, actor, name = "PK_StarWarsFilmActors_swf_act") // PK_StarWarsFilmActors_swf_act is optional here
}

class StarWarsFilmEntity(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<StarWarsFilmEntity>(StarWarsFilmsTable)

    var sequelId by StarWarsFilmsTable.sequelId
    var name by StarWarsFilmsTable.name
    var director by StarWarsFilmsTable.director
    val ratings by UserRatingEntity referrersOn UserRatingsTable.film // make sure to use val and referrersOn
    var actors by ActorEntity via StarWarsFilmActorsTable
}
```
### Parent-child
```koktlin
object StarWarsFilmRelationsTable : Table() {
    val parentFilm = reference("parent_film_id", StarWarsFilmsWithDirectorTable)
    val childFilm = reference("child_film_id", StarWarsFilmsWithDirectorTable)
    override val primaryKey = PrimaryKey(parentFilm, childFilm, name = "PK_FilmRelations")
}

class StarWarsFilmWithParentAndChildEntity(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<StarWarsFilmWithParentAndChildEntity>(StarWarsFilmsWithDirectorTable)

    var name by StarWarsFilmsWithDirectorTable.name
    var director by DirectorEntity referencedOn StarWarsFilmsWithDirectorTable.director

    // Define hierarchical relationships
    var sequels by StarWarsFilmWithParentAndChildEntity.via(StarWarsFilmRelationsTable.parentFilm, StarWarsFilmRelationsTable.childFilm)
    var prequels by StarWarsFilmWithParentAndChildEntity.via(StarWarsFilmRelationsTable.childFilm, StarWarsFilmRelationsTable.parentFilm)
}

val director1 = DirectorEntity.new {
    name = "George Lucas"
    genre = Genre.SCI_FI
}

val film1 = StarWarsFilmWithParentAndChildEntity.new {
    name = "Star Wars: A New Hope"
    director = director1
}

val film2 = StarWarsFilmWithParentAndChildEntity.new {
    name = "Star Wars: The Empire Strikes Back"
    director = director1
}

val film3 = StarWarsFilmWithParentAndChildEntity.new {
    name = "Star Wars: Return of the Jedi"
    director = director1
}

// Assign parent-child relationships
film2.prequels = SizedCollection(listOf(film1)) // Empire Strikes Back is a sequel to A New Hope
film3.prequels = SizedCollection(listOf(film2)) // Return of the Jedi is a sequel to Empire Strikes Back
film1.sequels = SizedCollection(listOf(film2, film3)) // A New Hope has Empire Strikes Back as a sequel
film2.sequels = SizedCollection(listOf(film3)) // Empire Strikes Back has Return of the Jedi as a sequel
```
# 名词解释
| 名词  | 解释                        |
|-----|---------------------------|
| DSL | Domain-Specific Language  |
| ORM | Object-relational Mapping |
| DAO | Data Access Object        |
