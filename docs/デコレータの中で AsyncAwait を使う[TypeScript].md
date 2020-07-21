# 関数に付与したデコレータの中の処理で非同期処理をいい感じに完了させる

関数用のデコレータ(Method Decorator)で、非同期処理を実行しようとして詰まったため、実装方法を置いておく。

## 環境
- typescript: `3.9.7`
- target: `es2019`

## 確認用関数

以降、動作確認のため断りなくこの関数を使います。

```ts
// ms 待ってからログ出力
const delayLog = async (str: string, ms: number) => {
    await delay(ms);
    console.log(str);
};
const delay = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));
```

# テンプレート

こんな感じに書いてあげると、async/await が効いてくれる。

```ts
export function AsyncDecorator(): MethodDecorator {
    return function (
        target: Object,
        methodName: string | symbol,
        descriptor: PropertyDescriptor,
    ) {
        const originalMethod = descriptor.value;
        descriptor.value = function (...args: any[]) {
            const originalMethodFunc = async () => {
                return originalMethod.apply(this, [...args]);
            };
            // このメソッド内に自由に記述していく。
            const asyncFunc = async () => {
                // デコレータ を付与した関数本体
                await originalMethodFunc();
            };
            return asyncFunc();
        };
    };
}
```

# 使ってみる

## 実装

### デコレータ

トランザクション管理をしている想定。

```ts
export function TransactionDecorator(): MethodDecorator {
    return function (
        target: Object,
        methodName: string | symbol,
        descriptor: PropertyDescriptor,
    ) {
        const originalMethod = descriptor.value;
        descriptor.value = function (...args: any[]) {
            const originalMethodFunc = async () => {
                return originalMethod.apply(this, [...args]);
            };
            // このメソッド内に自由に記述していく。
            const asyncFunc = async () => {
                delayLog("DBコネクション確立", 2000);
                delayLog("トランザクション処理開始", 1000);
                // デコレータ を付与した関数本体
                await originalMethodFunc();
                delayLog("トランザクション処理完了", 3000);
                delayLog("DBコネクション廃棄", 1000);
            };
            return asyncFunc();
        };
    };
}
```

### クラス

```ts
class UserRepository {
    @TransactionDecorator()
    async registerNew() {
        delayLog("ユーザー登録処理", 2000);
    }
}
```

## 実行

```ts
await new UserRepository().registerNew()
// =>
//    DBコネクション確立
//    トランザクション処理開始
//    ユーザー登録処理
//    トランザクション処理完了
//    DBコネクション廃棄
```


# 実行タイミング管理テンプレート

よく使う感じのものを用意してみました。  
下記の引数を設定すれば、そのタイミングで関数が実行されます。

- beforeFunc: 関数本体の実行前に実行する関数
- afterFunc: 関数本体の実行後に実行する関数
- catchFunc: エラー発生時に実行する関数

```ts
export type VoidFunc = (() => Promise<void>) | (() => void);
export type ErrorArgVoidFunc = ((error: any) => Promise<void>) | (() => void);
/**
 *
 * @param beforeFunc no args void func
 * @param afterFunc no args void func
 * @param catchFunc error arg void func
 */
export function AsyncDecorator(
    beforeFunc?: VoidFunc,
    afterFunc?: VoidFunc,
    catchFunc?: ErrorArgVoidFunc,
): MethodDecorator {
    return function (
        target: Object,
        methodName: string | symbol,
        descriptor: PropertyDescriptor,
    ) {
        const originalMethod = descriptor.value;
        descriptor.value = function (...args: any[]) {
            const originalMethodFunc = async () => {
                return originalMethod.apply(this, [...args]);
            };
            const asyncFunc = async () => {
                try {
                    await beforeFunc?.();
                    const result = await originalMethodFunc();
                    await afterFunc?.();
                    return result;
                } catch (e) {
                    if (catchFunc === undefined) throw e;
                    await catchFunc?.(e);
                }
            };
            return asyncFunc();
        };
    };
}
```

## 使用サンプル

### テスト実装

```ts
describe('AsyncDecorator', () => {
    class User {
        @AsyncDecorator(
            async () => delayLog('まえ', 4000),
            async () => delayLog('あと', 2000),
            async () => delayLog('しっぱい', 1000),
        )
        async success() {
            console.log('こんにちは');
        }

        @AsyncDecorator(
            async () => delayLog('まえ', 4000),
            async () => delayLog('あと', 2000),
            async () => delayLog('しっぱい', 1000),
        )
        async throw() {
            console.log('こんにちは');
            throw 'えらー';
        }
    }
    it('せいこう', async () => {
        await new User().success();
    }, 10000);
    it('しっぱい', async () => {
        await new User().throw();
    }, 10000);
});
```

### 実行結果

```
まえ
こんにちは
あと

まえ
こんにちは
しっぱい
```


## 失敗例

この実装だとうまく実行してくれない。不思議。

```ts
export function AsyncDecorator(
    beforeFunc?: VoidFunc,
    afterFunc?: VoidFunc,
    catchFunc?: ErrorArgVoidFunc,
): MethodDecorator {
    return function (
        target: Object,
        methodName: string | symbol,
        descriptor: PropertyDescriptor,
    ) {
        const originalMethod = descriptor.value;
        descriptor.value = function (...args: any[]) {
            const originalMethodFunc = async () => {
                return originalMethod.apply(this, [...args]);
            };
            // 関数を定義せず直接返すようにしている
            return async () => {
                try {
                    await beforeFunc?.();
                    const result = await originalMethodFunc();
                    await afterFunc?.();
                    return result;
                } catch (e) {
                    if (catchFunc === undefined) throw e;
                    await catchFunc?.(e);
                }
            };
        };
    };
}
```

### テスト実行すると

```ts
// 出力なし
```

# 参考

- https://github.com/typeorm/typeorm/blob/c6336aa4c3bbdc4c7907db605c5bd497c0eb42a3/src/connection/ConnectionManager.ts

これでトランザクション周りの処理もすっきり書ける！
