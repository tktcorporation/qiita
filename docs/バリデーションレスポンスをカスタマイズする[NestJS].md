# やりたいこと

NestJS の Web API は class-validator を使ってリクエストを検証できる。
そして、そこで引っかかった項目をまとめて 400 のエラーを返してくれる。
こんな感じ。

```ts
{
    "statusCode": 400,
    "message": [
        "each value in {field} must be a valid enum value"
    ],
    "error": "Bad Request",
}
```

このフォーマットを好きな形に変更したい。

# やりかた

このレスポンスは class-validation が生成したクラスインスタンスからエラーメッセージ取り出して　Nest の Exception クラスを作成することによって作られている。
そして、それは ValidationPipe の中で行われている。

なので、ここの実装をいじってあげることで実現できる。

## ValidationPipe ってなに

main.ts のここに書かれてるやつ。

```ts
app = moduleFixture.createNestApplication();
app.useGlobalPipes(new ValidationPipe({ transform: true }));
await app.init();
```

# 実装する

ValidationPipe を拡張する形で作成。
（private なプロパティが多かったので、この実装方法はあまり想定されていないのかも?）

使っている BadRequestException は独自に実装したもので、ここでレスポンスの形を決めている。
BadRequestException の実装については **[NestJS でエラーレスポンスをカスタマイズしてみる](https://qiita.com/tktcorporation/items/936135b551ce555a4626)** ここで書いている。

```ts
import { ValidationPipe, HttpStatus } from '@nestjs/common';
import { ValidationError } from 'class-validator';
import { HttpErrorByCode } from '@nestjs/common/utils/http-error-by-code.util';
import { iterate } from 'iterare';
import { BadRequestException } from 'src/exception/badRequest.exception';

export class CustomValidationPipe extends ValidationPipe {
    createExceptionFactory() {
        return (validationErrors: ValidationError[] = []) => {
            if (this.isDetailedOutputDisabled) {
                return new HttpErrorByCode[this.errorHttpStatusCode]();
            }
            const errors = this.flattenValidationErrorsExtends(
                validationErrors,
            );
            // Bad Request 以外でも使いたければここらへんの実装をいじる
            if (this.errorHttpStatusCode === HttpStatus.BAD_REQUEST)
                return new BadRequestException(errors);
            return new HttpErrorByCode[this.errorHttpStatusCode](errors);
        };
    }
    private flattenValidationErrorsExtends(
        validationErrors: ValidationError[],
    ): string[] {
        return iterate(validationErrors)
            .map(error => this.mapChildrenToValidationErrorsExtends(error))
            .flatten()
            .filter(item => !!item.constraints)
            .map(item => item.constraints && Object.values(item.constraints))
            .flatten()
            .toArray()
            .filter(
                (item): item is Exclude<typeof item, undefined> =>
                    item !== undefined,
            );
    }
    private mapChildrenToValidationErrorsExtends(error: ValidationError) {
        if (!(error.children && error.children.length)) {
            return [error];
        }
        const validationErrors: ValidationError[] = [];
        for (const item of error.children) {
            if (item.children && item.children.length) {
                validationErrors.push(
                    ...this.mapChildrenToValidationErrorsExtends(item),
                );
            }
            validationErrors.push(
                this.prependConstraintsWithParentPropExtends(error, item),
            );
        }
        return validationErrors;
    }
    private prependConstraintsWithParentPropExtends(
        parentError: ValidationError,
        error: ValidationError,
    ) {
        const constraints: { [key: string]: string } = {};
        for (const key in error.constraints) {
            constraints[
                key
            ] = `${parentError.property}.${error.constraints[key]}`;
        }
        return Object.assign(Object.assign({}, error), { constraints });
    }
}
```

# 使ってみる

## main.ts

こんな感じで使ってあげれば、指定した形でレスポンスが返ってくる。

```ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';
import { Logger, INestApplication } from '@nestjs/common';
import { Config } from './app.config';
import { CustomValidationPipe } from 'src/presentation/pipe/customValidation.pipe';

async function bootstrap() {
    const app = await NestFactory.create(AppModule);
    app.useGlobalPipes(
        new CustomValidationPipe({
            transform: true,
        }),
    );

    await app.init();
    await app.listen(3000);
}
bootstrap();
```
