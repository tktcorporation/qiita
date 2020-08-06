# 前提条件

- 画像の保存処理はデコレータで行われる
- 画像保存等の処理には multer というモジュールが使われる
- デコレータでは保存するファイル名の指定、保存先の指定などが行える
- デコレータでは、形式、サイズ等によって受け取るファイルのフィルタリングが行える
- 既存のデコレータの実装では、関数の中の処理は画像の保存後に行われる

# 方針

- 保存処理はデコレータに任せ、オプションを渡してファイル名の指定、フィルタリングなどを行う
- デコレータでは実現できない部分はファイルの保存後に追加で実施する

# ひながた

これだけ書けば、`POST HOST/somathing/images` の file に画像を乗せることで保存自体はできるはず。

```ts
@Controller('images')
class SampleController {
    @Post()
    @UseInterceptors(FilesInterceptor('files'))
    async uploadFile(
        @UploadedFile() file: Express.Multer.File,
    ) {}
}
```

# 細かい実装

## 追加で必要になったモジュール

multer, @nestjs/platform-express

```bash
$ yarn install multer @nestjs/platform-express
```

## Intercepter

`@UseInterceptors()` に渡すことで、保存するファイル名の指定や、フィルタリングを行える。

```ts
import { FileInterceptor } from '@nestjs/platform-express';
import { diskStorage } from 'multer';
import { BadRequestException } from '@nestjs/common';

export const customFileIntercept = (options: {
    fieldname: string;        // file が乗っている body の field
    dest: string;             // 保存したい場所
    maxFileSize: number;      // 最大画像サイズ
    fileCount: number;        // 最大画像枚数
    allowFileTypes: string[]; // 許可する画像形式
}) =>
    FileInterceptor(options.fieldname, {
        storage: diskStorage({
            destination: options.dest,
        }),
        limits: {
            fileSize: options.maxFileSize,
            files: options.fileCount,
        },
        fileFilter(
            req: any,
            file: {
                fieldname: string;
                originalname: string;
                encoding: string;
                mimetype: string;
                size: number;
                destination: string;
                filename: string;
                path: string;
                buffer: Buffer;
            },
            callback: (error: Error | null, acceptFile: boolean) => void,
        ) {
            if (!options.allowFileTypes.includes(file.mimetype))
                return callback(
                    new BadRequestException('invalid file type.'),
                    false,
                );
            return callback(null, true);
        },
    });
```

## Controller

```ts
import {
    Post,
    UseInterceptors,
    Controller,
    UploadedFile,
    BadRequestException,
    Body,
} from '@nestjs/common';
import { PostImagesDto } from '../dto/post-images.dto';
import { PostImagesClass } from '../class/post-images.class';
import { Image } from 'src/domain/image/image.domain';
// 上で作ったもの
import { customFileIntercept } from './fileIntercepter';

@Controller('images')
export class ImagesController {
    @Post()
    @UseInterceptors(customFileIntercept({
        fieldname: 'file',
        dest: './uploads',
        maxFileSize: 200000,
        fileCount: 1,
        allowFileTypes: ['image/png'],
    }))
    async uploadFile(
        @UploadedFile() file: Express.Multer.File | undefined,
        @Body() dto: PostImagesDto,
    ): Promise<PostImagesClass> {
        if (!file) throw new BadRequestException('file field must be set.');
        
        // body で受け取った値を使った処理
        // db に問い合わせて、その結果ファイル名を変えるとか、ファイル削除するとか
        something(file, body)

        return new PostImagesClass(file);
    }
}
```
