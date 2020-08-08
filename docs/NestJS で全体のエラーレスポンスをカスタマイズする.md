# やりたいこと

どんな例外が投げられても、同じ形式のエラーレスポンスを返す。

## 前回やったこと
[NestJS でエラーレスポンスをカスタマイズしてみる](https://qiita.com/tktcorporation/items/5d70e4dde85ae3b6eeda) の方法でエラーレスポンスをカスタマイズすることができた。
が、これだけではまだできないことがある。

### できること
- 自ら作成した例外を使用することでレスポンスの形を変えることができる

### できないこと
- 自前実装したところ以外は NestJS 標準の例外が投げられる
- NestJS 標準の例外が投げられた場合は標準形式のレスポンスが返却される
- 上記の理由から、すべてのレスポンス形式を統一させることができない

# 前提

- NestJS は最後まで catch されなかった例外を拾ってエラーレスポンスを返す仕組みがある(ExceptionFilter)
- ExceptionFilter には、自前でカスタムできる仕組みが用意されている
ref: https://docs.nestjs.com/exception-filters

```ts
import { CustomExceptionFilter } from './component/exception/filter/customException.filter';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    // こんな感じでセットすることで使える
    app.useGlobalFilters(new CustomExceptionFilter());
    await app.init();

    await app.listen(3000);
}
bootstrap();
```

# 方針

ExceptionFilter を使って、例外からレスポンスを作成する処理を実装する。

## ExceptionFilter の構成

1. すべての例外を拾う
1. 拾った例外が HttpException(StatusCode などの情報を含んでいる例外) か
  - Yes: なにもしない
  - No: InternalServerError 用の HttpException を生成する
1. HttpException からレスポンス用の Json を生成する
1. Json を返却

# 実装

ここでは、例としてレスポンスを下記の形式に統一する実装を行う。

```ts
interface ResponseJson {
    message: string;
    errors: Array<{
        resource?: string;
        field?: string;
        code: string;
    }>;
}
```

## ResponseJson

```ts
interface ResponseJson {
    message: string;
    errors: Array<{
        resource?: string;
        field?: string;
        code: string;
    }>;
}
```

## 例外 -> Response

コメントアウト部分は [NestJS でエラーレスポンスをカスタマイズしてみる](https://qiita.com/tktcorporation/items/936135b551ce555a4626) の実装に依存する。
レスポンスをしっかりと制御したい場合はコメントアウト部分のような実装が一例となるかもしれない。

```ts
import {
    HttpException,
    InternalServerErrorException,
} from '@nestjs/common';

// import { ErrorHttpStatusCode } from '@nestjs/common/utils/http-error-by-code.util';

// 特定のレスポンス生成に特化した独自例外クラス
// import { CustomHttpException } from 'src/component/exception/core/customHttp.exception';

// ステータスコードから CustomHttpException をとってくる
// import { ExceptionByCode } from 'src/component/exception/exceptionByCode';

// CustomHttpException を継承した例外
// import { InternalServerErrorException } from 'src/component/exception/internalServerError.exception';

const transformToHttpException = (
    exception: unknown,
): HttpException /* | CustomHttpException */ => {
    // const createCustomExceptionByHttpException = (exception: HttpException) => {
    //     const status: ErrorHttpStatusCode = exception.getStatus();
    //     return new ExceptionByCode[status](exception.message);
    // };
    // if (exception instanceof CustomHttpException) return exception;
    // if (exception instanceof HttpException)
    //     return createCustomExceptionByHttpException(exception);
    if (exception instanceof HttpException) return exception;
    return new InternalServerErrorException();
};

const createResponseJson = (
    exception: HttpException /* | CustomHttpException */,
): ResponseJson => {
    // const createResponseJsonByCustomHttpException = (
    //      exception: CustomHttpException,
    // ) => ({
    //      message: exception.getResponse().message,
    //      errors: exception.getResponse().errors,
    // })

    const createResponseJsonByHttpException = (exception: HttpException) => {
        return {
            message: exception.message,
            errors: [
                {
                    code: exception.getStatus().toString(),
                },
            ],
        };
    };

    // if (exception instanceof CustomHttpException)
    //     return createResponseJsonByCustomHttpException(exception);
    return createResponseJsonByHttpException(exception);
};
```

## CustomExceptionFilter

```ts
import {
    ExceptionFilter,
    Catch,
    ArgumentsHost,
} from '@nestjs/common';
import { Response } from 'express';

@Catch()
export class CustomExceptionFilter implements ExceptionFilter {
    catch(unknown: unknown, host: ArgumentsHost) {
        const httpException = transformToHttpException(unknown);

        const ctx = host.switchToHttp();
        const response = ctx.getResponse<Response>();

        const responseJson: ResponseJson = createResponseJson(httpException);
        response.status(httpException.getStatus()).json(responseJson);
    }
}
```

# 使ってみる

```ts
import { CustomExceptionFilter } from './component/exception/filter/customException.filter';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    app.useGlobalFilters(new CustomExceptionFilter());
    await app.init();

    await app.listen(3000);
}
bootstrap();
```

もしくは Module に実装することもできる。

```ts
import { APP_FILTER } from '@nestjs/core';
import { CustomExceptionFilter } from 'src/component/exception/filter/customException.filter';

@Module({
    controllers: [AnyController],
    providers: [
        {
            provide: APP_FILTER,
            useClass: CustomExceptionFilter,
        },
    ],
})
```

# 参考
- https://docs.nestjs.com/exception-filters
