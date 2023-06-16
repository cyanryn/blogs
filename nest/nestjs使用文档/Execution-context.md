Nest 提供了几个实用程序类，有助于轻松编写跨多个应用程序上下文（例如，基于 Nest HTTP 服务器、微服务和 WebSockets 应用程序上下文）运行的应用程序。这些实用程序提供有关当前执行上下文的信息，这些信息可用于构建可跨一组广泛的控制器、方法和执行上下文工作的通用防护、过滤器和拦截器。

这里介绍了两个这样的类： `ArgumentsHost` 和 `ExecutionContext` 。

## ArgumentsHost 

`ArgumentsHost` 类提供用于检索传递给处理程序的参数的方法。它允许选择适当的上下文（例如，HTTP，RPC（微服务）或WebSockets）来检索参数。

`ArgumentsHost` 只是充当处理程序参数的抽象。例如，对于 HTTP 服务器应用程序（使用 `@nestjs/platform-express` 时）， `host` 对象封装了 Express 的 `[request, response, next]` 数组，其中 `request` 是请求对象， `response` 是响应对象， `next` 是控制应用程序的请求-响应周期的函数。另一方面，对于 GraphQL 应用程序， `host` 对象包含 `[root, args, context, info]` 数组。

### getArgs 

```typescript
const [req, res, next] = host.getArgs();
```

### getArgByIndex

```typescript
const request = host.getArgByIndex(0);
const response = host.getArgByIndex(1);
```

### 上下文切换实用程序 

```typescript
/**
 * Switch context to RPC.
 */
switchToRpc(): RpcArgumentsHost;
/**
 * Switch context to HTTP.
 */
switchToHttp(): HttpArgumentsHost;
/**
 * Switch context to WebSockets.
 */
switchToWs(): WsArgumentsHost;
```

```typescript
const ctx = host.switchToHttp();
const request = ctx.getRequest<Request>();
const response = ctx.getResponse<Response>();
```

```typescript
export interface WsArgumentsHost {
  /**
   * Returns the data object.
   */
  getData<T>(): T;
  /**
   * Returns the client object.
   */
  getClient<T>(): T;
}
```

```typescript
export interface RpcArgumentsHost {
  /**
   * Returns the data object.
   */
  getData<T>(): T;

  /**
   * Returns the context object.
   */
  getContext<T>(): T;
}
```

## 当前应用程序上下文

我们需要一种方法来确定我们的方法当前正在运行的应用程序类型。

```typescript
if (host.getType() === 'http') {
  // do something that is only important in the context of regular HTTP requests (REST)
} else if (host.getType() === 'rpc') {
  // do something that is only important in the context of Microservice requests
} else if (host.getType<GqlContextType>() === 'graphql') {
  // do something that is only important in the context of GraphQL requests
}
```



## ExecutionContext 

`ExecutionContext` 扩展 `ArgumentsHost` ，提供有关当前执行过程的其他详细信息。与 `ArgumentsHost` 一样，Nest 在您可能需要的地方提供了 `ExecutionContext` 的实例，例如在守卫的 `canActivate()` 方法和拦截器的 `intercept()` 方法中。它提供以下方法：

```typescript
export interface ExecutionContext extends ArgumentsHost {
  /**
   * Returns the type of the controller class which the current handler belongs to.
   */
  getClass<T>(): Type<T>;
  /**
   * Returns a reference to the handler (method) that will be invoked next in the
   * request pipeline.
   */
  getHandler(): Function;
}
```


```typescript
const methodKey = ctx.getHandler().name; // "create"
const className = ctx.getClass().name; // "CatsController"
```

## 反射和元数据

