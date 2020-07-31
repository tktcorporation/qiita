# やりたいこと

デフォルトの Exception が返すこんな感じのレスポンスを。

```json
{
  "statusCode": 400,
  "message": "id must be number.",
  "error": "Bad Request"
}
```

例えばこんな感じに変更したい。

```json
{
   "message": "Bad Request",
   "errors": [
     {
       "resource": "Issue",
       "field": "id",
       "code": "must be number"
     }
   ]
 }
```

(どちらも Status Code は 400 が返る想定)

# やること

NestJS は最後まで Catch されなかった Exception を拾ってエラーレスポンスを生成してくれるので、そこで拾ってくれる Exception を独自の形で書いてあげれば良い。

# 実装

## Exception Class

### Base

```ts
import { HttpException, HttpStatus } from '@nestjs/common';

export class CustomHttpException extends HttpException {
    constructor(
        message: string,
        errors: Array<{
            resource: string,
            field: string,
            code: string,
        }>,
        statusCode: HttpStatus,
    ) {
        const response = {
            code,
            title,
            message,
        };
        super(
            HttpException.createBody(response, message, statusCode),
            statusCode,
        );
    }
}
```

### Bad Request 用

```ts
import { HttpStatus } from '@nestjs/common';
import { CustomHttpException } from './CustomHttpException';

export class BadRequestException extends CustomHttpException {
    static readonly message = 'Bad Request';
    constructor(
        errors: Array<{
            resource: string,
            field: string,
            code: string,
        }>,
    ) {
        super(
            BadRequestException.message,
            errors,
            HttpStatus.BAD_REQUEST,
        );
    }
}
```

# 使い方

こんな感じで書いてあげれば、あとは NestJS が拾って良い感じに返却してくれる。

```ts
throw new BadRequestException([
    {
       resource: "Issue",
       field: "id",
       code: "must be number"
    },
])
```

