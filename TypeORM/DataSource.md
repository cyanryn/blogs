## 什么是DataSource

只有在设置`DataSource`后，才能与数据库进行交互。TypeORM的`DataSource`保存了数据库连接设置，并且根据使用的RDBMS（关系数据库管理系统）建立初始数据库连接或连接池。

为了建立初始连接/连接池，必须调用`DataSource`实例的`initialize`方法。

调用`destory`方法时断开连接（关闭连接池中的所有连接）。

## 创建新的DataSource

```js
import { DataSource } from "typeorm"

const AppDataSource = new DataSource({
    type: "mysql",
    host: "localhost",
    port: 3306,
    username: "test",
    password: "test",
    database: "test",
})

AppDataSource.initialize()
    .then(() => {
        console.log("Data Source has been initialized!")
    })
    .catch((err) => {
        console.error("Error during Data Source initialization", err)
    })
```

如果您将在整个应用程序中使用此实例，最好将`AppDataSource`导出并设置为全局可用。

`DataSource`接受`DataSourceOptions`，这些选项根据使用数据库类型而异。

## 如何使用DataSource

设置`DataSource`后，可以在应用中的任何位置使用它，例如：
```js
import { AppDataSource } from "./app-data-source"
import { User } from "../entity/User"

export class UserController {
    @Get("/users")
    getAll() {
        return AppDataSource.manager.find(User)
    }
}
```

使用`DataSource`实例，您可以对实体执行数据库操作，尤其是使用`.manager`和`.getRepository()`属性。


## DataSourceOptions

`DataSourceOptions` 是您在创建新的 `DataSource` 实例时传递的数据源配置。

### 常用的数据源选项

- `type` - RDBMS type.
	- eg: `"mysql", "postgres", "cockroachdb", "sap", "spanner", "mariadb", "sqlite", "cordova", "react-native", "nativescript", "sqljs", "oracle", "mssql", "mongodb", "aurora-mysql", "aurora-postgres", "expo", "better-sqlite3", "capacitor".`
	- This option is **required**.
- `extra` - Extra options to be passed to the underlying driver.
- `entities` - 接受实体类、实体架构类和目录路径。
	- eg: `entities: [Post, Category, "entity/*.js", "modules/**/entity/*.js"]`
- `subscribers` - 要加载并用于此数据源的订阅服务器。
	- eg: `subscribers: [PostSubscriber, AppSubscriber, "subscriber/*.js", "modules/**/subscriber/*.js"]`.
- `migrations` - 要加载并用于此数据源的迁移。
	- eg: `migrations: [FirstMigration, SecondMigration, "migration/*.js", "modules/**/migration/*.js"]`
- `logging` - 指示是否启用日志记录。如果设置为 `true` ，则将启用查询和错误日志记录。
	- 您还可以指定要启用的不同类型的日志记录，例如 `["query", "error", "schema"]` 
- `logger` - 用于日志记录目的的记录器。
	-  Possible values are `"advanced-console", "simple-console" and "file".`
	- Default is `"advanced-console".`
- `maxQueryExecutionTime` - 如果查询执行时间超过此给定的最大执行时间（以毫秒为单位），则记录器将记录此查询。
- `poolSize` - 配置连接池的最大活动连接数。
- `namingStrategy` - 用于命名数据库中的表和列的命名策略。
- `entityPrefix` - 此数据源上所有表（或集合）的给定字符串前缀。
- `entitySkipConstructor` - 指示在从数据库中反序列化实体时，TypeORM 是否应跳过构造函数。请注意，如果不调用构造函数，私有属性和默认属性都不会按预期运行。
- `dropSchema` - 每次初始化数据源时删除架构。请注意此选项，不要在生产中使用它 - 否则将丢失所有生产数据。此选项在调试和开发期间很有用。
- `synchronize` - 指示是否应在每次应用程序启动时自动创建数据库架构。请注意此选项，不要在生产中使用它 - 否则可能会丢失生产数据。此选项在调试和开发期间很有用。作为替代方案，您可以使用 CLI 并运行 schema：sync 命令。请注意，对于MongoDB数据库，它不会创建模式，因为MongoDB是无模式的。相反，它仅通过创建索引进行同步。
- `migrationsRun` - 指示是否应在每次应用程序启动时自动运行迁移。作为替代方法，您可以使用 CLI 并运行 migration：run 命令。
- `migrationsTableName` - 数据库中将包含有关已执行迁移的信息的表的名称。默认情况下，此表称为“migrations”。
- `migrationsTransactionMode` - 控制迁移事务（默认值： `all` ），可以是 `all` | `none` |`each`
- `metadataTableName` - 数据库中将包含有关表元数据信息的表的名称。默认情况下，此表称为“typeorm_metadata”。
- `cache` - 启用实体结果缓存。您还可以在此处配置缓存类型和其他缓存选项。

### `mysql` / `mariadb` 数据源选项

- `url`
- `host`
- `port` - 数据库主机端口。默认 mysql 端口为 `3306` 。
- `username`
- `password`
- `database` - 数据库名称。
- `charset` - 连接的字符集。这在MySQL的SQL级别称为“排序规则”（如utf8_general_ci）。如果指定了 SQL 级字符集（如 utf8mb4），则使用该字符集的默认排序规则。（默认值： `UTF8_GENERAL_CI` ）。
- `timezone` - 在 MySQL 服务器上配置的时区。可以是 `local` 、 `Z` 或形式为 `+HH:MM` 或 `-HH:MM` 的偏移量。（默认值： `local` ）
- `connectTimeout` - 在与 MySQL 服务器的初始连接期间发生超时之前的毫秒数。（默认值： `10000` ）
- `acquireTimeout` - 在初始连接到 MySql 服务器期间发生超时之前的毫秒数。它与 `connectTimeout` 不同，因为它控制 TCP 连接超时，而`connectTimeout`则不控制。 （默认值： `10000` ）
- `insecureAuth` - 允许连接到要求使用旧（不安全）身份验证方法的 MySQL 实例。（默认值： `false` ）
- `supportBigNumbers` - 处理数据库中的大数字（ `BIGINT` 和 `DECIMAL` 列）时，应启用此选项（默认值： `true` ）
- `bigNumberStrings` - Enabling both `supportBigNumbers` and `bigNumberStrings` forces big numbers (`BIGINT` and `DECIMAL` columns) to be always returned as JavaScript String objects (Default: `true`). Enabling `supportBigNumbers` but leaving `bigNumberStrings` disabled will return big numbers as String objects only when they cannot be accurately represented with [JavaScript Number objects](http://ecma262-5.com/ELS5_HTML.htm#Section_8.5) (which happens when they exceed the `[-2^53, +2^53]` range), otherwise they will be returned as Number objects. This option is ignored if `supportBigNumbers` is disabled.
- `dateStrings` - 强制日期类型 （ `TIMESTAMP` ， `DATETIME` ， `DATE` ） 作为字符串返回。可以是真/假或要保留为字符串的类型名称数组。（默认值： `false` ）
- `debug` - 将协议详细信息打印到标准输出。可以是真/假或应打印的数据包类型名称数组。（默认值： `false` ）
- `trace` - 在错误时生成堆栈跟踪，以包括库入口的调用站点（“长堆栈跟踪”）。大多数调用的性能略有下降。（默认值： `true` ）
- `multipleStatements` - 每个查询允许多个 mysql 语句。请注意这一点，它可能会增加SQL注入攻击的范围。（默认值： `false` ）
- `legacySpatialSupport` - 使用在MySQL 8中删除的空间函数，如GeomFromText和AsText。（默认值：`true`）
- `flags` - 除默认标志外要使用的连接标志列表。也可以将默认的列入黑名单。
- `ssl` - 具有 SSL 参数或包含 SSL 配置文件名称的字符串的对象。See [SSL options](https://github.com/mysqljs/mysql#ssl-options).


### 示例

```js
{
    host: "localhost",
    port: 3306,
    username: "test",
    password: "test",
    database: "test",
    logging: true,
    synchronize: true,
    entities: [
        "entity/*.js"
    ],
    subscribers: [
        "subscriber/*.js"
    ],
    entitySchemas: [
        "schema/*.json"
    ],
    migrations: [
        "migration/*.js"
    ]
}
```

## 多个 DataSource

### 使用多个DataSource

连接不同数据库的多个`DataSource`，只需创建多个`DataSource`实例：

```js
import { DataSource } from "typeorm"

const db1DataSource = new DataSource({
    type: "mysql",
    host: "localhost",
    port: 3306,
    username: "root",
    password: "admin",
    database: "db1",
    entities: [__dirname + "/entity/*{.js,.ts}"],
    synchronize: true,
})

const db2DataSource = new DataSource({
    type: "mysql",
    host: "localhost",
    port: 3306,
    username: "root",
    password: "admin",
    database: "db2",
    entities: [__dirname + "/entity/*{.js,.ts}"],
    synchronize: true,
})
```


### 使用单个DataSource连接多个数据库

要在单个`DataSource`中使用多个数据库，可以指定每个实体的数据库名称：

```js
import { Entity, PrimaryGeneratedColumn, Column } from "typeorm"

@Entity({ database: "secondDB" })
export class User {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    firstName: string

    @Column()
    lastName: string
}
```

```js
import { Entity, PrimaryGeneratedColumn, Column } from "typeorm"

@Entity({ database: "thirdDB" })
export class Photo {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    url: string
}
```

从不同数据库中获取数据

```js
const users = await dataSource
    .createQueryBuilder()
    .select()
    .from(User, "user")
    .addFrom(Photo, "photo")
    .andWhere("photo.userId = user.id")
    .getMany() // userId is not a foreign key since its cross-database request
```

此代码将生成以下 SQL 查询（取决于数据库类型）：

```js
SELECT * FROM "secondDB"."user" "user", "thirdDB"."photo" "photo"
    WHERE "photo"."userId" = "user"."id"
```

您还可以指定表路径而不是实体：

```js
const users = await dataSource
    .createQueryBuilder()
    .select()
    .from("secondDB.user", "user")
    .addFrom("thirdDB.photo", "photo")
    .andWhere("photo.userId = user.id")
    .getMany() // userId is not a foreign key since its cross-database request
```

### 在单个数据源中使用多个架构
若要在应用程序中使用多个架构，只需在每个实体上设置 `schema` ：

```js
import { Entity, PrimaryGeneratedColumn, Column } from "typeorm"

@Entity({ schema: "secondSchema" })
export class User {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    firstName: string

    @Column()
    lastName: string
}
```

```js
import { Entity, PrimaryGeneratedColumn, Column } from "typeorm"

@Entity({ schema: "thirdSchema" })
export class Photo {
    @PrimaryGeneratedColumn()
    id: number

    @Column()
    url: string
}
```

从不同架构中获取数据

```js
const users = await dataSource
    .createQueryBuilder()
    .select()
    .from(User, "user")
    .addFrom(Photo, "photo")
    .andWhere("photo.userId = user.id")
    .getMany() // userId is not a foreign key since its cross-database request
```

此代码将生成以下 SQL 查询（取决于数据库类型）：

```js
SELECT * FROM "secondSchema"."question" "question", "thirdSchema"."photo" "photo"
    WHERE "photo"."userId" = "user"."id"
```

您还可以指定表路径而不是实体：

```js
const users = await dataSource
    .createQueryBuilder()
    .select()
    .from("secondSchema.user", "user") // in mssql you can even specify a database: secondDB.secondSchema.user
    .addFrom("thirdSchema.photo", "photo") // in mssql you can even specify a database: thirdDB.thirdSchema.photo
    .andWhere("photo.userId = user.id")
    .getMany()
```

### 复制
您可以使用 TypeORM 设置读/写复制。复制选项示例：

```js
{
  type: "mysql",
  logging: true,
  replication: {
    master: {
      host: "server1",
      port: 3306,
      username: "test",
      password: "test",
      database: "test"
    },
    slaves: [{
      host: "server2",
      port: 3306,
      username: "test",
      password: "test",
      database: "test"
    }, {
      host: "server3",
      port: 3306,
      username: "test",
      password: "test",
      database: "test"
    }]
  }
}
```

所有架构更新和写入操作都使用 `master` 服务器执行。查找方法或选择查询生成器执行的所有简单查询都使用随机 `slave` 实例。通过查询方法执行的所有查询都使用 `master` 实例执行。

如果要在查询生成器创建的 SELECT 中显式使用 master，可以使用以下代码：

```js
const masterQueryRunner = dataSource.createQueryRunner("master")
try {
    const postsFromMaster = await dataSource
        .createQueryBuilder(Post, "post")
        .setQueryRunner(masterQueryRunner)
        .getMany()
} finally {
    await masterQueryRunner.release()
}
```
如果要在查询中使用 `slave` ，还需要显式指定查询运行程序。

```js
const slaveQueryRunner = dataSource.createQueryRunner("slave")
try {
    const userFromSlave = await slaveQueryRunner.query(
        "SELECT * FROM users WHERE id = $1",
        [userId],
        slaveQueryRunner,
    )
} finally {
    return slaveQueryRunner.release()
}
```

请注意，由 `QueryRunner` 创建的连接需要显式释放。

mysql，postgres和sql server数据库支持复制。

Mysql 支持深度配置：
```js
{
  replication: {
    master: {
      host: "server1",
      port: 3306,
      username: "test",
      password: "test",
      database: "test"
    },
    slaves: [{
      host: "server2",
      port: 3306,
      username: "test",
      password: "test",
      database: "test"
    }, {
      host: "server3",
      port: 3306,
      username: "test",
      password: "test",
      database: "test"
    }],

    /**
    * If true, PoolCluster will attempt to reconnect when connection fails. (Default: true)
    */
    canRetry: true,

    /**
     * If connection fails, node's errorCount increases.
     * When errorCount is greater than removeNodeErrorCount, remove a node in the PoolCluster. (Default: 5)
     */
    removeNodeErrorCount: 5,

    /**
     * If connection fails, specifies the number of milliseconds before another connection attempt will be made.
     * If set to 0, then node will be removed instead and never re-used. (Default: 0)
     */
     restoreNodeTimeout: 0,

    /**
     * Determines how slaves are selected:
     * RR: Select one alternately (Round-Robin).
     * RANDOM: Select the node by random function.
     * ORDER: Select the first node available unconditionally.
     */
    selector: "RR"
  }
}
```

## DataSource API

`options` - 用于创建此数据源的选项。
```js
const dataSourceOptions: DataSourceOptions = dataSource.options
```

-  `isInitialized` - 指示数据源是否已初始化，是否与数据库建立了初始连接/连接池。
```js
const isInitialized: boolean = dataSource.isInitialized
```

- `driver` - 此数据源中使用的基础数据库驱动程序。
```js
const driver: Driver = dataSource.driver
```

- `manager` - `EntityManager` 用于处理实体。
```js
const manager: EntityManager = dataSource.manager 
// you can call manager methods, for example find: 
const users = await manager.find()
```

- `mongoManager` - `MongoEntityManager` 用于处理 MongoDB 数据源的实体。
```js
const manager: MongoEntityManager = dataSource.mongoManager
// you can call manager or mongodb-manager specific methods, for example find:
const users = await manager.find()
```

- `initialize` - 初始化数据源并打开数据库的连接池。
```js
await dataSource.initialize()
```

- `destroy` - 销毁数据源并关闭所有数据库连接。通常，在应用程序关闭时调用此方法。
```js
await dataSource.destroy()
```

- `synchronize` - 同步数据库架构。在数据源选项中设置 `synchronize: true` 时，它将调用此方法。通常，在应用程序启动时调用此方法。
```js
await dataSource.synchronize()
```

- `dropDatabase` - 删除数据库及其所有数据。在生产环境中要小心使用此方法，因为此方法将擦除所有数据库表及其数据。只能在建立与数据库的连接后使用。
```js
await dataSource.dropDatabase()
```

- `runMigrations` - 运行所有挂起的迁移。
```js
await dataSource.runMigrations()
```

- `undoLastMigration` - 还原上次执行的迁移。
```js
await dataSource.undoLastMigration()
```

- `hasMetadata` - 检查是否已注册给定实体的元数据。
```js
if (dataSource.hasMetadata(User)) 
	const userMetadata = dataSource.getMetadata(User)
```

- `getMetadata` - 获取给定实体的 `EntityMetadata` 。您还可以指定表名称，如果找到具有此类表名称的实体元数据，则将返回该表名称。
```js
const userMetadata = dataSource.getMetadata(User) 
// now you can get any information about User entity
```

- `getRepository` - 获取给定实体的 `Repository` 。您还可以指定表名，如果找到给定表的存储库，则将返回该存储库。
```js
const repository = dataSource.getRepository(User)
// now you can call repository methods, for example find:
const users = await repository.find()
```

- `getTreeRepository` - 获取给定实体的 `TreeRepository` 。您还可以指定表名，如果找到给定表的存储库，则将返回该存储库。
```js
const repository = dataSource.getTreeRepository(Category)
// now you can call tree repository methods, for example findTrees:
const categories = await repository.findTrees()
```

- `getMongoRepository` - 获取给定实体的 `MongoRepository` 。此存储库用于 MongoDB 数据源中的实体。
```js
const repository = dataSource.getMongoRepository(User)
// now you can call mongodb-specific repository methods, for example createEntityCursor:
const categoryCursor = repository.createEntityCursor()
const category1 = await categoryCursor.next()
const category2 = await categoryCursor.next()
```


- `transaction` - 提供单个事务，其中将在单个数据库事务中执行多个数据库请求。
```js
await dataSource.transaction(async (manager) => {
    // NOTE: you must perform all database operations using given manager instance
    // its a special instance of EntityManager working with this transaction
    // and don't forget to await things here
})
```

- `query` - 执行原始 SQL 查询。
```js
const rawData = await dataSource.query(`SELECT * FROM USERS`)
```

- `createQueryBuilder` - 创建可用于生成查询的查询生成器。
```js
const users = await dataSource
    .createQueryBuilder()
    .select()
    .from(User, "user")
    .where("user.name = :name", { name: "John" })
    .getMany()
```

- `createQueryRunner` - 创建用于管理和使用单个实际数据库数据源的查询运行程序。
```js
const queryRunner = dataSource.createQueryRunner()

// you can use its methods only after you call connect
// which performs real database connection
await queryRunner.connect()

// .. now you can work with query runner and call its methods

// very important - don't forget to release query runner once you finished working with it
await queryRunner.release()
```

