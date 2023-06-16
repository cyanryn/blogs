管道是使用 `@Injectable()` 装饰器注释的类，它实现 `PipeTransform` 接口。

管道有两个典型的用例：
- **transformation**：将输入数据转换为所需的形式（例如，从字符串转换为整数）
- **validation**：评估输入数据，如果有效，只需原封不动地传递它;否则，引发异常

Nest在方法调用前插入管道，管道会对接收数据进行验证或转换后，再将数据传递给控制器的路由处理函数

## 内置管道

- `ValidationPipe`
- `ParseIntPipe`
- `ParseFloatPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `ParseEnumPipe`
- `DefaultValuePipe`
- `ParseFilePipe`

## 绑定管道

```typescript
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

如果验证失败
```bash
GET localhost:3000/abc
```

Nest会抛出异常
```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```



如果想要自定义内置管道的行为，可以传递<font color="#c0504d">管道实例</font>

```typescript
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

## 自定义管道

```typescript
// validation.pipe.ts
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

每个管道都必须实现 `transform()` 方法才能完成 `PipeTransform` 接口协定。此方法有两个参数：
- `value`
- `metadata`
`value` 参数是当前处理的方法参数（在路由处理方法接收之前）， `metadata` 是当前处理的方法参数的元数据。元数据对象具有以下属性：
```typescript
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string; // 传递给装饰器的字符串，例如 `@Body('string')` 。如果将装饰器括号留空，则为 `undefined`
}
```

## 对象架构验证


## 绑定验证管道

## class-validator

Nest可以很好集成class-validator
```bash
$ npm i --save class-validator class-transformer
```
接下来，我们可以向 `CreateCatDto` 类添加一些装饰器。
```typescript
import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```


自己实现一个 `ValidationPipe` 类
```typescript
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    // 先将普通对象转成类实例再验证
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```


绑定 `ValidationPipe`

```typescript
@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

## 全局管道

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

如果需要注入依赖关系
```typescript
import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

```ad-tip
`useGlobalPipes()` 无法注入依赖关系，因为绑定已在任何模块的上下文之外完成。
```

## transform 用例
`ParseIntPipe` ，它负责将字符串解析为整数值。
```typescript
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```
绑定管道
```typescript
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id) {
  return this.catsService.findOne(id);
}
```

## 提供默认值

```typescript
@Get()
async findAll(
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
) {
  return this.catsService.findAll({ activeOnly, page });
}
```


