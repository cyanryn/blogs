控制器是负责处理传入请求，并将响应返回给客户端。

通过类和装饰器，使Nest能够创建路由映射（将请求绑定到相应的控制器）

## 控制器定义

```typescript
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```



`@Controller('cats')` 指定了路由前缀 `cats`

## 请求方法

- `@Get`用于获取数据
- `@Post`用于新增数据
- `@Patch`用于更新部分数据
- `@Put`用于更新全部数据
- `@Delete`用于删除数据
- `@Options`用于对cors的跨域预检(一般用不到)
- `@Head`用于自定义请求头,常用于下载,导出excel文件等


## 请求对象

- `@Request(), @Req()`: 请求数据
- `@Response(), @Res()`: 响应数据
- `@Next()`: 执行下一个中间件(一般用不到)
- `@Session()`: session对象(一般用不到)
- `@Param(key?: string)` 获取url中的params参数，比如 `posts/1`
- `@Query(key?: string)` 获取url中的查询数据，比如`posts?page=1
- `@Body(key?: string)`  请求主体数据,一般结合DTO使用,用于新增或修改数据


## 路由通配符

```ts
@Get('ab*cd')
findAll() {
  return 'This route uses a wildcard';
}
```

- `'ab*cd'` 路由路径将匹配 `abcd` 、 `ab_cd` 、 `abecd` 等。
- 字符 `?` 、 `+` 、 `*` 和 `()` 可以在路由路径中使用

## 状态码

- `@HttpCode(200)`  方法装饰器
- `response.status(200).send()`

## 请求头

- `@Header('Cache-Control', 'none')` 方法装饰器
- `res.header()`

## 重定向

`@Redirect()` 有两个参数， `url` 和 `statusCode` ，都是可选的。如果省略，默认值 `statusCode` 为 `302`

```typescript
@Get()
@Redirect('https://nestjs.com', 301)
```

如果要动态确定 HTTP 状态代码或重定向 URL，则需要返回一个对象
```json
{
  "url": string,
  "statusCode": number
}
```

返回的值将覆盖传递给 `@Redirect()` 装饰器的任何参数。例如：

```typescript
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

## 路由参数

```typescript
@Get(':id')
findOne(@Param() params: any): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

```typescript
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
```

## DTO与数据验证

dto是用于对请求数据结构进行定义的一个类，常用于对`body`,`query`等请求数据进行验证

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

```typescript
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
```

`ValidationPipe` 可以过滤掉方法处理程序不应接收的属性。

```ts
  @Get()
    list(
        @Query(
            new ValidationPipe({
                transform: true,
                forbidUnknownValues: true,
                validationError: { target: false },
            }),
        )
        options: QueryUserDto,
    ) {
        return this.service.query(options);
    }
```


## 完整示例

```typescript
import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
```


## 启动与运行

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';

@Module({
  controllers: [CatsController],
})
export class AppModule {}
```


## 使用@res

```typescript
import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Res() res: Response) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  findAll(@Res() res: Response) {
     res.status(HttpStatus.OK).json([]);
  }
}
```

在上面的示例中，拦截器和 `@HttpCode()` / `@Header()` 装饰器会失效，要解决此问题，您可以将 `passthrough` 选项设置为 `true` ，如下所示：
```typescript
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return [];
}
```

