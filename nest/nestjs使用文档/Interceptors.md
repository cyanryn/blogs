拦截器是使用 `@Injectable()` 装饰器注释的类，实现 `NestInterceptor` 接口。

每个拦截器实现 `intercept()` 方法，该方法接受两个参数。

第一个是 `ExecutionContext` 实例

第二个参数是 `CallHandler` 

## intercept()方法

### Execution context


### Call handler

`CallHandler` 接口实现 `handle()` 方法，您可以使用该方法在侦听器中的某个时刻调用路由处理程序方法。如果在 `intercept()` 方法的实现中不调用 `handle()` 方法，则根本不会执行路由处理程序方法。


## 示例

第一个用例是使用拦截器来记录用户交互（例如，存储用户调用、异步调度事件或计算时间戳）。我们在下面显示一个简单的 `LoggingInterceptor` ：

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```

## 绑定拦截器

为了设置拦截器，我们使用从 `@nestjs/common` 包导入的 `@UseInterceptors()` 装饰器。与管道和保护一样，侦听器可以是控制器范围的、方法范围的或全局范围的。


```typescript
@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```


设置全局拦截器

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

设置依赖注入

```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```


## 响应映射

## 异常映射 

## 流覆盖

