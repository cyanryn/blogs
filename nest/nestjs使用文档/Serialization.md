序列化是在网络响应中返回对象之前发生的过程。这是提供转换和清理要返回给客户端的数据的规则的适当位置。例如，应始终从响应中排除密码等敏感数据。或者，某些属性可能需要其他转换，例如仅发送实体属性的子集。

Nest 提供了内置功能，可帮助确保这些操作能够以直接的方式执行。 `ClassSerializerInterceptor` 拦截器使用 class-transformer包 提供转换对象的声明性和可扩展方式。

它执行的基本操作是获取方法处理程序返回的值，并应用类转换器中的 `instanceToPlain()` 函数。在这样做时，它可以在实体/DTO 类上应用 `class-transformer` 个装饰器表示的规则，如下所述。

## 排除属性 

假设我们要从用户实体中自动排除 `password` 属性。我们对实体进行注释如下：

```typescript
import { Exclude } from 'class-transformer';

export class UserEntity {
  id: number;
  firstName: string;
  lastName: string;

  @Exclude()
  password: string;

  constructor(partial: Partial<UserEntity>) {
    Object.assign(this, partial);
  }
}
```

现在考虑一个控制器，该控制器具有返回此类实例的方法处理程序。

```typescript
@UseInterceptors(ClassSerializerInterceptor)
@Get()
findOne(): UserEntity {
  return new UserEntity({
    id: 1,
    firstName: 'Kamil',
    lastName: 'Mysliwiec',
    password: 'password',
  });
}
```

客户端会收到以下数据 

```json
{
  "id": 1,
  "firstName": "Kamil",
  "lastName": "Mysliwiec"
}
```

## 公开属性 

您可以使用 `@Expose()` 装饰器为属性提供别名，或执行函数来计算属性值（类似于 getter 函数），如下所示。

```typescript
@Expose()
get fullName(): string {
  return `${this.firstName} ${this.lastName}`;
}
```

## Transform

您可以使用 `@Transform()` 修饰器执行其他数据转换。例如，下面的构造返回 `RoleEntity` 的 name 属性，而不是返回整个对象。

```typescript
@Transform(({ value }) => value.name)
role: RoleEntity;
```


## Pass options

您可能需要修改转换函数的默认行为。要覆盖默认设置，请使用 `@SerializeOptions()` 修饰器在 `options` 对象中传递它们。

```typescript
@SerializeOptions({
  excludePrefixes: ['_'],
})
@Get()
findOne(): UserEntity {
  return new UserEntity();
}
```

通过 `@SerializeOptions()` 传递的选项作为基础 `instanceToPlain()` 函数的第二个参数传递。在此示例中，我们会自动排除以 `_` 前缀开头的所有属性。



