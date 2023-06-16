Nest 带有一个内置的异常层，负责处理整个应用程序中的所有未经处理的异常。当应用程序代码未处理异常时，该层会捕获该异常，然后自动发送适当的用户友好响应。

开箱即用，此操作由内置的全局异常筛选器执行，该筛选器处理类型 `HttpException` （及其子类）的异常。当异常无法识别（既不是 `HttpException` 也不是从 `HttpException` 继承的类）时，内置异常筛选器会生成以下默认 JSON 响应：

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

## 抛出标准异常

Nest 提供了一个内置的 `HttpException` 类，从 `@nestjs/common` 包中公开。对于典型的基于 HTTP REST/GraphQL API 的应用程序，最佳做法是在发生某些错误情况时发送标准 HTTP 响应对象。

```typescript
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

响应数据如下所示：
```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

`HttpException` 构造函数采用两个确定响应的必需参数：
- `response` 参数定义 JSON 响应正文。它可以是 `string` 或 `object` ，如下所述。
- `status` 参数定义 HTTP 状态代码。

默认情况下，JSON 响应正文包含两个属性：
- `statusCode` ：默认为 `status` 参数中提供的 HTTP 状态代码 
- `message` ：基于 `status` 的 HTTP 错误的简短描述


## 自定义异常

```typescript
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

```typescript
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

## 内置HTTP异常 

Nest 提供了一组从基数 `HttpException` 继承的标准异常。
- `BadRequestException`
- `UnauthorizedException`
- `NotFoundException`
- `ForbiddenException`
- `NotAcceptableException`
- `RequestTimeoutException`
- `ConflictException`
- `GoneException`
- `HttpVersionNotSupportedException`
- `PayloadTooLargeException`
- `UnsupportedMediaTypeException`
- `UnprocessableEntityException`
- `InternalServerErrorException`
- `NotImplementedException`
- `ImATeapotException`
- `MethodNotAllowedException`
- `BadGatewayException`
- `ServiceUnavailableException`
- `GatewayTimeoutException`
- `PreconditionFailedException`

所有内置异常还可以使用 `options` 参数提供错误 `cause` 和错误描述：

```typescript
throw new BadRequestException('Something bad happened', { cause: new Error(), description: 'Some error description' })
```

使用上述内容，响应的数据为：

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400,
}
```

## 异常过滤器

我们创建一个异常筛选器，负责捕获作为 `HttpException` 类实例的异常，并为它们实现自定义响应逻辑。为此，我们需要访问底层平台 `Request` 和 `Response` 对象。我们将访问 `Request` 对象，以便提取原始的 `url` 并将其包含在日志记录信息中。我们将使用 `Response` 对象直接控制发送的响应，使用 `response.json()` 方法。

```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

```ad-warning
如果使用 `@nestjs/platform-fastify` ，则可以使用 `response.send()` 而不是 `response.json()` 。不要忘记从 `fastify` 导入正确的类型。
```

`@Catch(HttpException)` 装饰器将所需的元数据绑定到异常过滤器，告诉 Nest 此特定过滤器正在查找类型 `HttpException` 的异常，而不是其他任何异常。 `@Catch()` 装饰器可以采用单个参数或逗号分隔的列表。这使您可以一次为多种类型的异常设置筛选器。

### Arguments host

`catch()` 方法的参数。 `exception` 参数是当前正在处理的异常对象。 `host` 参数是一个 `ArgumentsHost` 对象。

在此代码示例中，我们使用它来获取对传递给原始请求处理程序的 `Request` 和 `Response` 对象的引用。


## 绑定过滤器 

```typescript
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

或者

```typescript
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

```ad-hint
如果可能，最好使用类而不是实例来应用筛选器。它减少了内存使用，因为 Nest 可以轻松地在整个模块中重用同一类的实例。
```

### 设置范围

作用域可以位于不同的级别：方法范围、控制器范围或全局范围。

控制器范围：

```typescript
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```


全局范围：
```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```


### 依赖注入

在依赖注入方面，从任何模块外部注册的全局过滤器（如上例所示为 `useGlobalFilters()` ）无法注入依赖关系，因为这是在任何模块的上下文之外完成的。为了解决此问题，您可以使用以下构造直接从任何模块注册全局范围的筛选器：

```typescript
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

## Catch everything

为了捕获每个未处理的异常（无论异常类型如何），请将 `@Catch()` 装饰器的参数列表留空，例如 `@Catch()` 。

在下面的示例中，我们有一个与平台无关的代码，因为它使用 HTTP 适配器来传递响应，并且不直接使用任何特定于平台的对象（ `Request` 和 `Response` ）：

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // In certain situations `httpAdapter` might not be available in the
    // constructor method, thus we should resolve it here.
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

## 继承 

通常，您将创建专为满足应用程序要求而精心制作的完全自定义的异常筛选器。但是，在某些情况下，您可能只想扩展内置的默认全局异常筛选器，并根据某些因素覆盖行为。

```typescript
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```

```ad-warning
扩展 `BaseExceptionFilter` 的方法范围和控制器范围的筛选器不应使用 `new` 进行实例化。相反，让框架自动实例化它们。
```


全局筛选器可以扩展基本筛选器。这可以通过以下两种方式之一完成。

第一种方法是在实例化自定义全局筛选器时注入 `HttpAdapter` 引用：

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(3000);
}
bootstrap();
```


第二种方法是使用 `APP_FILTER` 令牌