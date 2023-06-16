为了自动验证传入的请求，Nest 提供了几个开箱即用的管道：
- `ValidationPipe`
- `ParseIntPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`

`ValidationPipe` 使用了`class-validator`包及其声明性验证装饰器。`ValidationPipe` 提供了一种方便的方法，用于对所有传入客户端请求数据强制实施验证规则，其中特定规则在每个模块的本地类/DTO 声明中使用简单注释进行声明。

## 使用内置的ValidationPipe

安装所需的依赖项
```bash
$ pnpm i --save class-validator class-transformer
```

由于此管道使用 `class-validator` 和 `class-transformer` 库，因此有许多可用选项。您可以通过传递给管道的配置对象来配置这些设置。以下是内置选项：

```typescript
export interface ValidationPipeOptions extends ValidatorOptions {
  transform?: boolean;
  disableErrorMessages?: boolean;
  exceptionFactory?: (errors: ValidationError[]) => any;
}
```

除此之外，所有 `class-validator` 选项（继承自 `ValidatorOptions` 接口）都可用：

| Option                  | Type     | Description                                                                                           |
| ----------------------- | -------- | ----------------------------------------------------------------------------------------------------- |
| enableDebugMessages     | boolean  | 如果设置为 true，验证器将在出现问题时向控制台打印额外的警告消息                                       |
| skipUndefinedProperties | boolean  | 如果设置为 true，则验证程序将跳过验证对象中未定义的所有属性的验证。                                   |
| skipNullProperties      | boolean  | 如果设置为 true，则验证程序将跳过验证对象中所有为 null 的属性的验证。                                 |
| skipMissingProperties   | boolean  | 如果设置为 true，则验证程序将跳过验证对象中所有为 null 或未定义的属性的验证。                         |
| whitelist               | boolean  | 如果设置为 true，验证器将去除未使用任何验证修饰器的任何属性的已验证（返回）对象。                     |
| forbidNonWhitelisted    | boolean  | 如果设置为 true，则验证程序不会剥离未列入白名单的属性，而是会引发异常。                               |
| forbidUnknownValues     | boolean  | 如果设置为 true，则验证未知对象的尝试将立即失败。(dto上属性都没有使用class-validator进行验证才会报错) |
| disableErrorMessages    | boolean  | 如果设置为 true，则不会向客户端返回验证错误。                                                         |
| errorHttpStatusCode     | number   | 此设置允许您指定在发生错误时使用的异常类型。默认情况下，它抛出 `BadRequestException` 。               |
| exceptionFactory        | Function | 获取验证错误的数组并返回要引发的异常对象。                                                            |
| groups                  | string[] | 验证对象期间要使用的组。                                                                              |
| strictGroups            | boolean  | 如果未给出 `groups` 或为空，请忽略至少具有一个组的修饰器。                                            |
| dismissDefaultMessages  | boolean  | 如果设置为 true，则验证将不使用默认消息。如果未显式设置，则错误消息将始终为 `undefined` 。            |
| validationError.target  | boolean  | 指示目标是否应在 `ValidationError` 中公开。                                                           |
| validationError.value   | boolean  | 指示是否应在 `ValidationError` 中公开验证值。                                                         |
| stopAtFirstError        | boolean  | 设置为 true 时，给定属性的验证将在遇到第一个错误后停止。默认为 false。                                |


## 测试管道

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

```typescript
@Post()
create(@Body() createUserDto: CreateUserDto) {
  return 'This action adds a new user';
}
```

使用`class-validator` 在 `CreateUserDto` 中添加一些验证规则
```typescript
import { IsEmail, IsNotEmpty } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsNotEmpty()
  password: string;
}
```

使用这些规则后，如果验证不通过，则：
```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": ["email must be an email"]
}
```

## 禁用详细信息

生产环境上，可能不需要显示错误信息，则：
```typescript
app.useGlobalPipes(
  new ValidationPipe({
    disableErrorMessages: true,
  }),
);
```

## 剥离特性

 `ValidationPipe` 还可以过滤掉方法处理程序不应接收的属性。在这种情况下，我们可以将可接受的属性列入白名单，并且任何未包含在白名单中的属性都会自动从生成的对象中删除。例如，如果我们的处理程序需要 `email` 和 `password` 属性，但请求还包含 `age` 属性，则可以从生成的 DTO 中自动删除此属性。若要启用此类行为，请将 `whitelist` 设置为 `true`。
```js
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
  }),
);
```
设置为 true 时，这将自动删除未列入白名单的属性（验证类中没有任何修饰器的属性）。

或者，您可以在存在未列入白名单的属性时停止处理请求，并向用户返回错误响应。若要启用此功能，请将 `forbidNonWhitelisted` 选项属性设置为 `true` ，并将设置 `whitelist` 设置为 `true` 。

## Transform

 `ValidationPipe` 可以自动将有效负载转换为 DTO 实例。要启用自动转换，请将 `transform` 设置为 `true`

```js
@Post()
@UsePipes(new ValidationPipe({ transform: true }))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```


如果是在全局上使用，则：
```typescript
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,
  }),
);
```

启用自动转换选项后， `ValidationPipe` 还将执行基本类型的转换。
```typescript
@Get(':id')
findOne(@Param('id') id: number) {
  console.log(typeof id === 'number'); // true
  return 'This action returns a user';
}
```

在上面的示例中，我们将 `id` 类型指定为 `number` （在方法签名中）。因此， `ValidationPipe` 将尝试自动将字符串标识符转换为数字。

## 显示转换
可以使用 `ParseIntPipe` 或 `ParseBoolPipe` 显式强制转换值

```typescript
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('sort', ParseBoolPipe) sort: boolean,
) {
  console.log(typeof id === 'number'); // true
  console.log(typeof sort === 'boolean'); // true
  return 'This action returns a user';
}
```

## 映射类型

### PartialType
`PartialType()` 函数返回一个类型（类），其中输入类型的所有属性都设置为 optional。例如，假设我们有一个创建类型，如下所示：
```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

默认情况下，所有这些字段都是必填字段。要创建具有相同字段但每个字段都是可选的类型，请使用 `PartialType()` 将类引用 （ `CreateCatDto` ） 作为参数传递：

```typescript
export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

### PickType
```typescript
export class UpdateCatAgeDto extends PickType(CreateCatDto, ['age'] as const) {}
```

### OmitType
`OmitType()` 函数通过从输入类型中选取所有属性，然后删除一组特定的键来构造类型。

```typescript
export class UpdateCatDto extends OmitType(CreateCatDto, ['name'] as const) {}
```


### IntersectionType
将两种类型组合成一个新类型（类）。例如，假设我们从两种类型开始，例如：

```typescript
export class CreateCatDto {
  name: string;
  breed: string;
}

export class AdditionalCatInfo {
  color: string;
}
```

我们可以生成一个新类型，该类型结合了两种类型中的所有属性。

```typescript
export class UpdateCatDto extends IntersectionType(
  CreateCatDto,
  AdditionalCatInfo,
) {}
```

