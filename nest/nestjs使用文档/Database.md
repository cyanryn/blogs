Nest与数据库无关，允许您轻松地与任何SQL或NoSQL数据库集成

## TypeORM 集成

安装所需的依赖

```bash
$ pnpm install --save @nestjs/typeorm typeorm mysql2
```

导入TypeORM到模块中

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

```ad-warning
不应在生产中使用设置 `synchronize: true` - 否则可能会丢失生产数据。
```

`forRoot()` 方法支持 `TypeORM` 包中的 [[DataSource]] 构造函数公开的所有配置属性。此外，下面还介绍了几个额外的配置属性。

| Options          | 描述                                           |
| ---------------- | ---------------------------------------------- |
| retryAttempts    | 尝试连接到数据库的次数（默认值： `10` ）       |
| retryDelay       | 尝试连接之间的延迟（毫秒）（默认值： `3000` ） |
| autoLoadEntities | 如果为 `true` ，则将自动加载实体（默认值： `false` ）                                               |
完成此操作后，TypeORM `DataSource` 和 `EntityManager` 对象将可用于注入整个项目（无需导入任何模块），例如：

```typescript
import { DataSource } from 'typeorm';

@Module({
  imports: [TypeOrmModule.forRoot(), UsersModule],
})
export class AppModule {
  constructor(private dataSource: DataSource) {}
}
```

### 存储库模式

TypeORM 支持存储库设计模式，因此每个实体都有自己的存储库。可以从数据库数据源获取这些存储库。

要继续这个例子，我们至少需要一个实体。让我们定义 `User` 实体。

```typescript
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;
}
```

要开始使用 `User` 实体，我们需要通过将它插入模块 `forRoot()` 方法选项中的 `entities` 数组来让 TypeORM 知道它（除非您使用静态 glob 路径）：
```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './users/user.entity';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [User],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

接下来，让我们看一下 `UsersModule` ：

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

此模块使用 `forFeature()` 方法定义在当前范围内注册的存储库。有了这个，我们可以使用 `@InjectRepository()` 装饰器将 `UsersRepository` 注入到 `UsersService` 中：

```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  findAll(): Promise<User[]> {
    return this.usersRepository.find();
  }

  findOne(id: number): Promise<User | null> {
    return this.usersRepository.findOneBy({ id });
  }

  async remove(id: number): Promise<void> {
    await this.usersRepository.delete(id);
  }
}
```

```ad-tip
`@InjectRepository` 方式注入 Repository 有个问题，就是它注入的Repository 不是自定义的，比如 UserRepository，所以要想自定义的Repository 也有自动注入依赖功能，就需要进一步处理。
实际环境中不会直接这样使用。
```

如果要在导入 `TypeOrmModule.forFeature` 的模块之外使用存储库，则需要导出：
```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  exports: [TypeOrmModule]
})
export class UsersModule {}
```

### Relations

Relations是在两个或多个表之间建立的关联。
有三种类型的关系：

|           类型                |  描述   |
| ------------------------- | --- |
| One-to-One                |  使用 `@OneToOne()` 修饰   |
| One-to-many / Many-to-one |   使用 `@OneToMany()` 和 `@ManyToOne()` 修饰符  |
| Many-to-many                          |  使用 `@ManyToMany()` 修饰符   |

若要定义每个 `User` 可以有多个照片，请使用 `@OneToMany()` 修饰器。
```typescript
import { Entity, Column, PrimaryGeneratedColumn, OneToMany } from 'typeorm';
import { Photo } from '../photos/photo.entity';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;

  @OneToMany(type => Photo, photo => photo.user)
  photos: Photo[];
}
```

### 自动加载实体 

手动将实体添加到数据源选项的 `entities` 数组可能很乏味。此外，从根模块引用实体会破坏应用程序域边界，并导致实现详细信息泄漏到应用程序的其他部分。为了解决此问题，提供了另一种解决方案。若要自动加载实体，请将配置对象的 `autoLoadEntities` 属性（传递到 `forRoot()` 方法）设置为 `true` ，如下所示：

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...
      autoLoadEntities: true,
    }),
  ],
})
export class AppModule {}
```

定该选项后，通过 `forFeature()` 方法注册的每个实体将自动添加到配置对象的 `entities` 数组中。

### EntitySchema


### TypeORM 事务

有许多不同的策略来处理TypeORM事务。我们建议使用 `QueryRunner` 类，因为它可以完全控制事务。

首先，我们需要以正常方式将 `DataSource` 对象注入到类中：

```typescript
@Injectable()
export class UsersService {
  constructor(private dataSource: DataSource) {}
}
```

现在，我们可以使用此对象来创建事务。

```typescript
async createMany(users: User[]) {
  const queryRunner = this.dataSource.createQueryRunner();

  await queryRunner.connect();
  await queryRunner.startTransaction();
  try {
    await queryRunner.manager.save(users[0]);
    await queryRunner.manager.save(users[1]);

    await queryRunner.commitTransaction();
  } catch (err) {
    // since we have errors lets rollback the changes we made
    await queryRunner.rollbackTransaction();
  } finally {
    // you need to release a queryRunner which was manually instantiated
    await queryRunner.release();
  }
}
```

或者，您可以将回调样式方法与 `DataSource` 对象的 `transaction` 方法一起使用 

```typescript
async createMany(users: User[]) {
  await this.dataSource.transaction(async manager => {
    await manager.save(users[0]);
    await manager.save(users[1]);
  });
}
```

Subscribers

使用 TypeORM 订阅者，您可以侦听特定的实体事件。

```typescript
import {
  DataSource,
  EntitySubscriberInterface,
  EventSubscriber,
  InsertEvent,
} from 'typeorm';
import { User } from './user.entity';

@EventSubscriber()
export class UserSubscriber implements EntitySubscriberInterface<User> {
  constructor(dataSource: DataSource) {
    dataSource.subscribers.push(this);
  }

  listenTo() {
    return User;
  }

  beforeInsert(event: InsertEvent<User>) {
    console.log(`BEFORE USER INSERTED: `, event.entity);
  }
}
```

```ad-warning
事件订阅者不能是 request-scoped
```

现在，将 `UserSubscriber` 类添加到 `providers` 数组中：

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UserSubscriber } from './user.subscriber';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService, UserSubscriber],
  controllers: [UsersController],
})
export class UsersModule {}
```

### Migrations
迁移提供了一种以增量方式更新数据库架构的方法，以使其与应用程序的数据模型保持同步，同时保留数据库中的现有数据。为了生成、运行和还原迁移，TypeORM 提供了一个专用的 CLI。

迁移类独立于 Nest 应用程序源代码。它们的生命周期由TypeORM CLI维护。因此，您<span style="background:rgba(240, 107, 5, 0.2)">无法通过迁移利用依赖注入和其他特定于 Nest 的功能</span>。


### 多个数据库 

```typescript
const defaultOptions = {
  type: 'postgres',
  port: 5432,
  username: 'user',
  password: 'password',
  database: 'db',
  synchronize: true,
};

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...defaultOptions,
      host: 'user_db_host',
      entities: [User],
    }),
    TypeOrmModule.forRoot({
      ...defaultOptions,
      name: 'albumsConnection',
      host: 'album_db_host',
      entities: [Album],
    }),
  ],
})
export class AppModule {}
```

```ad-tip
如果未为数据源设置 `name` ，则其名称设置为 `default` 。请注意，您不应有多个没有名称或具有相同名称的连接，否则它们将被覆盖。
```


```ad-tip
如果使用 `TypeOrmModule.forRootAsync` ，则还必须将数据源名称设置为 `useFactory` 之外。例如：
```typescript
TypeOrmModule.forRootAsync({
  name: 'albumsConnection',
  useFactory: ...,
  inject: ...,
}),

```

此时，您已将 `User` 和 `Album` 实体注册到其自己的数据源。通过此设置，您必须告诉 `TypeOrmModule.forFeature()` 方法和 `@InjectRepository()` 修饰器应使用哪个数据源。如果不传递任何数据源名称，则使用 `default` 数据源。

```typescript
@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    TypeOrmModule.forFeature([Album], 'albumsConnection'),
  ],
})
export class AppModule {}
```

您还可以为给定数据源注入 `DataSource` 或 `EntityManager`

```typescript
@Injectable()
export class AlbumsService {
  constructor(
    @InjectDataSource('albumsConnection')
    private dataSource: DataSource,
    @InjectEntityManager('albumsConnection')
    private entityManager: EntityManager,
  ) {}
}
```

也可以向提供程序注入任何 `DataSource` ：

```typescript
@Module({
  providers: [
    {
      provide: AlbumsService,
      useFactory: (albumsConnection: DataSource) => {
        return new AlbumsService(albumsConnection);
      },
      inject: [getDataSourceToken('albumsConnection')],
    },
  ],
})
export class AlbumsModule {}
```

### 测试

在对应用程序进行单元测试时，我们通常希望避免建立数据库连接，保持测试套件的独立性及其执行过程尽可能快。但是我们的类可能依赖于从数据源（连接）实例中提取的存储库。我们如何处理？解决方案是创建模拟存储库。为了实现这一目标，我们设置了自定义提供程序。每个已注册的存储库都由一个 `<EntityName>Repository` 令牌自动表示，其中 `EntityName` 是实体类的名称。

`@nestjs/typeorm` 包公开 `getRepositoryToken()` 函数，该函数基于给定实体返回准备好的令牌。

```typescript
@Module({
  providers: [
    UsersService,
    {
      provide: getRepositoryToken(User),
      useValue: mockRepository,
    },
  ],
})
export class UsersModule {}
```

现在，`mockRepository` 将用作 `UsersRepository` 。每当任何类使用 `@InjectRepository()` 装饰器请求 `UsersRepository` 时，Nest 将使用注册的 `mockRepository` 对象。

### 异步配置 

您可能希望异步传递存储库模块选项，而不是静态传递。在这种情况下，请使用 `forRootAsync()` 方法，该方法提供了几种处理异步配置的方法。

一种方法是使用工厂函数：

```typescript
TypeOrmModule.forRootAsync({
  useFactory: () => ({
    type: 'mysql',
    host: 'localhost',
    port: 3306,
    username: 'root',
    password: 'root',
    database: 'test',
    entities: [],
    synchronize: true,
  }),
});
```

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (configService: ConfigService) => ({
    type: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    entities: [],
    synchronize: true,
  }),
  inject: [ConfigService],
});
```


或者，您可以使用 `useClass` 语法：

```typescript
TypeOrmModule.forRootAsync({
  useClass: TypeOrmConfigService,
});
```

上面的构造将在 `TypeOrmModule` 中实例化 `TypeOrmConfigService` ，并通过调用 `createTypeOrmOptions()` 使用它来提供选项对象。请注意，这意味着 `TypeOrmConfigService` 必须实现 `TypeOrmOptionsFactory` 接口，如下所示：

```typescript
@Injectable()
export class TypeOrmConfigService implements TypeOrmOptionsFactory {
  createTypeOrmOptions(): TypeOrmModuleOptions {
    return {
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [],
      synchronize: true,
    };
  }
}
```

为了防止在 `TypeOrmModule` 中创建 `TypeOrmConfigService` 并使用从其他模块导入的提供程序，可以使用 `useExisting` 语法。

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

此结构的工作方式与 `useClass` 相同，但有一个关键区别 - `TypeOrmModule` 将查找导入的模块以重用现有的 `ConfigService` 而不是实例化新的模块。

```ad-hint
确保 `name` 属性的定义与 `useFactory` 、 `useClass` 或 `useValue` 属性的级别相同。这将允许 Nest 在适当的注入令牌下正确注册数据源。
```

### 自定义数据源工厂

结合使用 `useFactory` 、 `useClass` 或 `useExisting` 的异步配置，您可以选择指定 `dataSourceFactory` 函数，该函数允许您提供自己的 TypeORM 数据源，而不是允许 `TypeOrmModule` 创建数据源。

`dataSourceFactory` 接收在异步配置期间使用 `useFactory` 、 `useClass` 或 `useExisting` 配置的 TypeORM `DataSourceOptions` ，并返回解析 TypeORM `DataSource` 的 `Promise` 。

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  // Use useFactory, useClass, or useExisting
  // to configure the DataSourceOptions.
  useFactory: (configService: ConfigService) => ({
    type: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    entities: [],
    synchronize: true,
  }),
  // dataSource receives the configured DataSourceOptions
  // and returns a Promise<DataSource>.
  dataSourceFactory: async (options) => {
    const dataSource = await new DataSource(options).initialize();
    return dataSource;
  },
});
```

## Sequelize 集成

