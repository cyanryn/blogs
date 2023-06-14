
## 一、基本使用
```js
import {
  validate,
  validateOrReject,
  Contains,
  IsInt,
  Length,
  IsEmail,
  IsFQDN,
  IsDate,
  Min,
  Max,
} from 'class-validator';

export class Post {
  @Length(10, 20)
  title: string;

  @Contains('hello')
  text: string;

  @IsInt()
  @Min(0)
  @Max(10)
  rating: number;

  @IsEmail()
  email: string;

  @IsFQDN()
  site: string;

  @IsDate()
  createDate: Date;
}

let post = new Post();
post.title = 'Hello'; // should not pass
post.text = 'this is a great post about hell world'; // should not pass
post.rating = 11; // should not pass
post.email = 'google.com'; // should not pass
post.site = 'googlecom'; // should not pass

validate(post).then(errors => {
  // errors is an array of validation errors
  if (errors.length > 0) {
    console.log('validation failed. errors: ', errors);
  } else {
    console.log('validation succeed');
  }
});

validateOrReject(post).catch(errors => {
  console.log('Promise rejected (validation failed). Errors: ', errors);
});
// or
async function validateOrRejectExample(input) {
  try {
    await validateOrReject(input);
  } catch (errors) {
    console.log('Caught promise rejection (validation failed). Errors: ', errors);
  }
}
```


## 二、ValidatorOptions
`validate`函数的第二个参数
```js
export interface ValidatorOptions {
  skipMissingProperties?: boolean;
  whitelist?: boolean;
  forbidNonWhitelisted?: boolean;
  groups?: string[];
  dismissDefaultMessages?: boolean;
  validationError?: {
    target?: boolean;
    value?: boolean;
  };

  forbidUnknownValues?: boolean;
  stopAtFirstError?: boolean;
}
```


> [!NOTE] 注意
> `forbidUnknownValues` 默认值为true


### Whitelisting

如果对象是验证类的实例，不希望对象包含未定义的其他属性时，可以这样设置：
```js
import { validate } from 'class-validator';
// ...
validate(post, { whitelist: true });
```

此时，对没有任何修饰器的属性赋值时，也会报错
```js
import {validate, Allow, Min} from "class-validator";

export class Post {

    @Allow()
    title: string;

    @Min(0)
    views: number;

    nonWhitelistedProperty: number;
}

let post = new Post();
post.title = 'Hello world!';
post.views = 420;

post.nonWhitelistedProperty = 69;
(post as any).anotherNonWhitelistedProperty = "something";

validate(post).then(errors => {
  // post.nonWhitelistedProperty is not defined
  // (post as any).anotherNonWhitelistedProperty is not defined
  ...
});
```

如果希望存在任何非白名单属性时引发错误，可以这样设置：
```js
import { validate } from 'class-validator';
// ...
validate(post, { whitelist: true, forbidNonWhitelisted: true });
```


### skipMissingProperties

如果希望跳过对验证对象不存在的属性的验证，可以这样设置：
```js
import { validate } from 'class-validator';
// ...
validate(post, { skipMissingProperties: true });
```


跳过缺少属性时，如果有些属性时必须的，则使用 `@IsDefined()`。`@IsDefined()` 是唯一忽略 `skipMissingProperties` 选项的装饰器

### groups

在不同情况下，您可能希望使用同一对象的不同验证架构。在这种情况下，您可以使用验证组。

```js
import { validate, Min, Length } from 'class-validator';

export class User {
  @Min(12, {
    groups: ['registration'],
  })
  age: number;

  @Length(2, 20, {
    groups: ['registration', 'admin'],
  })
  name: string;

  @IsEmail(undefined, { always: true })
  email: string;
}

let user = new User();
user.age = 10;
user.name = 'Alex';

validate(user, {
  groups: ['registration'],
}); // this will not pass validation

validate(user, {
  groups: ['admin'],
}); // this will pass validation

validate(user, {
  groups: ['registration', 'admin'],
}); // this will not pass validation

validate(user, {
  groups: undefined, // the default
}); // this will not pass validation since all properties get validated regardless of their groups

validate(user, {
  groups: [],
}); // this will not pass validation, (equivalent to 'groups: undefined', see above)
```

`always: true` 。此标志表示无论使用哪个组，都必须始终应用此验证。

## 三、ValidationError
`validate` 方法返回一个包含 `ValidationError` 个对象的数组。每个 `ValidationError` 个是：
```js
{   
	// Object that was validated.
    target: Object;
    
    // Object's property that haven't pass validation.
    property: string; 
    
    // Value that haven't pass a validation.
    value: any; 
    
	// Constraints that failed validation with error messages.
    constraints?: { 
        [type: string]: string;
    };
    
    // Contains all nested validation errors of the property
    children?: ValidationError[]; 
}
```


当我们验证`post`对象时，会得到返回结果：
```js
[{
    target: /* post object */,
    property: "title",
    value: "Hello",
    constraints: {
        length: "$property must be longer than or equal to 10 characters"
    }
}, {
    target: /* post object */,
    property: "text",
    value: "this is a great post about hell world",
    constraints: {
        contains: "text must contain a hello string"
    }
},
// and other errors
]
```



如果不希望在`ValidationError`中暴露`target`，可以设置`ValidatorOptions`为：
```js
validator.validate(post, { validationError: { target: false } });
```

当你通过http发送错误，并且不想公开整个目标对象时，这尤其有用。

## 四、验证消息

可以在修饰器选项中指定验证消息，该消息会在`ValidationError`中返回（如果字段验证失败）

```js
import { MinLength, MaxLength } from 'class-validator';

export class Post {
  @MinLength(10, {
    message: 'Title is too short',
  })
  @MaxLength(50, {
    message: 'Title is too long',
  })
  title: string;
}
```

可以在消息中使用一些特殊标识：
- `$value`
- `$property`：验证对象属性的名称
- `$target`：验证对象的类的名称
- `$constraint1`, `$constraint2`, ... `$constraintN`

用法示例：
```js
import { MinLength, MaxLength } from 'class-validator';

export class Post {
  @MinLength(10, {
    // here, $constraint1 will be replaced with "10", and $value with actual supplied value
    message: 'Title is too short. Minimal length is $constraint1 characters, but actual is $value',
  })
  @MaxLength(50, {
    // here, $constraint1 will be replaced with "50", and $value with actual supplied value
    message: 'Title is too long. Maximal length is $constraint1 characters, but actual is $value',
  })
  title: string;
}
```

也可以使用函数返回消息：
```js
import { MinLength, MaxLength, ValidationArguments } from 'class-validator';

export class Post {
  @MinLength(10, {
    message: (args: ValidationArguments) => {
      if (args.value.length === 1) {
        return 'Too short, minimum length is 1 character';
      } else {
        return 'Too short, minimum length is ' + args.constraints[0] + ' characters';
      }
    },
  })
  title: string;
}
```


该消息函数接收的参数`ValidationArguments`：
- `value`
- `constraints`：特定类型定义的约束数组
- `targetName`：对象类的名称
- `object`：验证的对象
- `property`：对象属性的名称


## 五、验证集合

如果要对`数组/Set/Map`的每一项进行验证，则使用`each: true` 修饰器选项：
```js
import { MinLength, MaxLength } from 'class-validator';

export class Post {
  @MaxLength(20, {
    each: true,
  })
  tags: string[] | Set<string> | Map<string, string>;
}
```

## 六、验证嵌套对象
```js
import { ValidateNested } from 'class-validator';

export class Post {
  @ValidateNested()
  user: User;
}
```

嵌套对象必须是类的实例。

也可用于验证多维数组
```js
import { ValidateNested } from 'class-validator';

export class Plan2D {
  @ValidateNested()
  matrix: Point[][];
}
```

## 七、验证Promise

如果对象包含应验证的属性的返回值为 `Promise` ：
```js
import { ValidatePromise, Min } from 'class-validator';

export class Post {
  @Min(0)
  @ValidatePromise()
  userId: Promise<number>;
}
```

```js
import { ValidateNested, ValidatePromise } from 'class-validator';

export class Post {
  @ValidateNested()
  @ValidatePromise()
  user: Promise<User>;
}
```


## 八、继承

子类继承父类的装饰器
```js
import { validate } from 'class-validator';

class BaseContent {
  @IsEmail()
  email: string;

  @IsString()
  password: string;
}

class User extends BaseContent {
  @MinLength(10)
  @MaxLength(20)
  name: string;

  @Contains('hello')
  welcome: string;

  @MinLength(20)
  password: string;
}

let user = new User();

user.email = 'invalid email'; // inherited property
user.password = 'too short'; // password wil be validated not only against IsString, but against MinLength as well
user.name = 'not valid';
user.welcome = 'helo';

validate(user).then(errors => {
  // ...
}); // it will return errors for email, title and text properties
```

## 九、条件验证

`@ValidateIf`返回`false`时，会忽略属性上的其他验证
```js
import { ValidateIf, IsNotEmpty } from 'class-validator';

export class Post {
  otherProperty: string;

  @ValidateIf(o => o.otherProperty === 'value')
  @IsNotEmpty()
  example: string;
}
```



## 十一、使用上下文

将自定义对象传递给装饰器

```js
import { validate } from 'class-validator';

class MyClass {
  @MinLength(32, {
    message: 'EIC code must be at least 32 characters',
    context: {
      errorCode: 1003,
      developerNote: 'The validated string must contain 32 or more characters.',
    },
  })
  eicCode: string;
}

const model = new MyClass();

validate(model).then(errors => {
  //errors[0].contexts['minLength'].errorCode === 1003
});
```






## 十二、自定义验证类

### 创建约束类
```js
import { ValidatorConstraint, ValidatorConstraintInterface, ValidationArguments } from 'class-validator';

@ValidatorConstraint({ name: 'customText', async: false })
export class CustomTextLength implements ValidatorConstraintInterface {
  validate(text: string, args: ValidationArguments) {
    return text.length > 1 && text.length < 10; // for async validations you must return a Promise<boolean> here
  }

  defaultMessage(args: ValidationArguments) {
    // here you can provide default error message if validation failed
    return 'Text ($value) is too short or too long!';
  }
}
```


### 在类中使用新的验证约束
```js
import { Validate } from 'class-validator';
import { CustomTextLength } from './CustomTextLength';

export class Post {
  @Validate(CustomTextLength, {
    message: 'Title is too short or long!',
  })
  title: string;
}
```

### 使用验证器
```js
import { validate } from 'class-validator';

validate(post).then(errors => {
  // ...
});
```


### 传递约束
我们还可以将约束传递给验证器

```js
import { Validate } from 'class-validator';
import { CustomTextLength } from './CustomTextLength';

export class Post {
  @Validate(CustomTextLength, [3, 20], {
    message: 'Wrong post title',
  })
  title: string;
}
```

从 `validationArguments` 对象中使用它们：
```js
import { ValidationArguments, ValidatorConstraint, ValidatorConstraintInterface } from 'class-validator';

@ValidatorConstraint()
export class CustomTextLength implements ValidatorConstraintInterface {
  validate(text: string, validationArguments: ValidationArguments) {
    return text.length > validationArguments.constraints[0] && text.length < validationArguments.constraints[1];
  }
}
```

## 十三、自定义装饰器

这是使用自定义验证最优雅的方式

### `validator`使用简单对象

#### 创建装饰器

```js
import { registerDecorator, ValidationOptions, ValidationArguments } from 'class-validator';

export function IsLongerThan(property: string, validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      name: 'isLongerThan',
      target: object.constructor,
      propertyName: propertyName,
      constraints: [property],
      options: validationOptions,
      validator: {
        validate(value: any, args: ValidationArguments) {
          const [relatedPropertyName] = args.constraints;
          const relatedValue = (args.object as any)[relatedPropertyName];
          return typeof value === 'string' && typeof relatedValue === 'string' && value.length > relatedValue.length; // you can return a Promise<boolean> here as well, if you want to make async validation
        },
      },
    });
  };
}
```

#### 使用

```js
import { IsLongerThan } from './IsLongerThan';

export class Post {
  title: string;

  @IsLongerThan('title', {
    /* you can also use additional validation options, like "groups" in your custom validation decorators. "each" is not supported */
    message: 'Text must be longer than the title',
  })
  text: string;
}
```


### `validator`使用验证类

```js
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
} from 'class-validator';

@ValidatorConstraint({ async: true })
export class IsUserAlreadyExistConstraint implements ValidatorConstraintInterface {
  validate(userName: any, args: ValidationArguments) {
    return UserRepository.findOneByName(userName).then(user => {
      if (user) return false;
      return true;
    });
  }
}

export function IsUserAlreadyExist(validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName: propertyName,
      options: validationOptions,
      constraints: [],
      validator: IsUserAlreadyExistConstraint,
    });
  };
}
```

## 十四、使用服务容器
如果想将依赖项注入到自定义验证器约束类中，则：
```js
import { Container } from 'typedi';
import { useContainer, Validator } from 'class-validator';

// do this somewhere in the global application level:
useContainer(Container);
let validator = Container.get(Validator);

// now everywhere you can inject Validator class which will go from the container
// also you can inject classes using constructor injection into your custom ValidatorConstraint-s
```


与typeorm集成示例
```js
async function bootstrap() {
    // ...
    useContainer(app.select(AppModule), {
        fallbackOnErrors: true,
    });
    await app.listen(3000);
}
```

## 十五、手动验证

```js
import { isEmpty, isBoolean } from 'class-validator';

isEmpty(value);
isBoolean(value);
```

## 十六、验证普通对象

通过 [class-transformer](https://github.com/pleerock/class-transformer)将普通对象转成类实例，再进行验证。


## 参考链接
> [!NOTE] 参考链接
> [class-validator](https://github.com/typestack/class-validator)

