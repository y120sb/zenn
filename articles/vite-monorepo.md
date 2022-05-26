---
title: "React & ReactNativeのmonorepo環境にViteを導入する"
emoji: "✌🏻"
type: "tech"
topics: ["react", "reactnative", "typescript", "vite", "monorepo"]
published: false
---

# はじめに

こんにちは、[Unlace](https://www.unlace.net/)を運営している株式会社Unlaceの岩下です。

Unlaceでは、ユーザー向けにカウンセリング機能を提供する[Unlace](https://unlace.net/)のほか、登録カウンセラーの方が利用するUnlace for counselor、企業が従業員に対してUnlaceの利用料金を負担する仕組みを提供する[Unlace for business](https://www.unlace.net/business)があり、全部で3つのサービスを提供しています。
この内、UnlaceとUnlace for counselorはWebのほかiOS/Android向けのアプリでも提供しています。

全サービス共通でWebはReact、アプリはReactNativeを使用しており、[react-native-web](https://necolas.github.io/react-native-web/)を用いて実装の大部分を共通化しています。
また、共通部分の実装作業を効率化するため、これら全てを一つのリポジトリで管理する、いわゆるmonorepo構成となっています。
今回は、このmonorepo構成においてWeb部分の開発サーバーを[Vite](https://ja.vitejs.dev/)で構築した事例について紹介します。

# 背景

もともとWeb部分には[create-react-app](https://create-react-app.dev/)を使用していましたが、次の様な課題があり他のビルドツールへの移行を検討していました。

- コードベースの増加に伴ってdevサーバーの起動に時間がかかるようになった。
- ReactNativeと実装の共通化を図るにあたって、create-react-app特有の制限に躓くことが増えた。
    - @babel/plugin-proposal-class-propertiesがnode_modules以下に効かないとか。
        - ES2022に入ったようなので、最新のCRAでは使えるのかもしれませんが...
- Webを前提に作られたパッケージはReactNativeで利用できない事が多いため、どうしてもパッケージを選定する際の軸足はReactNative側にとなってしまい、そういったときにWeb側で色々と吸収できる自由度が欲しくなった。

これら全てを解消してくれるビルドツールは見つかりませんでしたが、開発の活発さや将来性を加味してViteへ移行することにしました。

# 前提

- monorepoの管理には[yarn workspaces](https://yarnpkg.com/features/workspaces#gatsby-focus-wrapper)を使用
- 可能な限りWebとAppの実装を共通化するため、共有コードはパッケージとして切り出し、それぞれのプロジェクトからimportする形を取る
- 高速化のため、共通コードのバンドル処理を避け、typescriptのまま扱う

## 環境

- yarn@3.2.1
- vite@2.9.9
- typescript

# 最終的な構成

以降の説明はこちらのディレクトリ構造を例に進めます。

```shell
$ tree .
.
├── node_modules
├── package.json
├── unlace-app # ReactNativeプロジェクト
│   ├── android
│   ├── app.json
│   ├── babel.config.js
│   ├── index.js
│   ├── ios
│   ├── metro.config.js
│   ├── node_modules
│   │   └── unlace-lib # ./unlace-libへのシンボリックリンク
│   ├── package.json
│   ├── src
│   ├── tsconfig.json
│   └── tsconfig.json
├── unlace-web # Webプロジェクト
│   ├── index.html
│   ├── node_modules
│   │   └── unlace-lib # ./unlace-libへのシンボリックリンク
│   ├── package.json
│   ├── public
│   ├── src
│   ├── tsconfig.json
│   └── vite.config.ts
├── unlace-web-server # Webサーバー
│   ├── build
│   ├── node_modules
│   ├── package.json
│   ├── src
│   ├── tsconfig.json
│   └── webpack.config.ts
├── unlace-lib # WebとReactNativeで共通利用するパッケージ
│   ├── node_modules
│   ├── package.json
│   ├── src
│   └── yarn-error.log
└── yarn.lock
```

:::details unlace-web/vite.config.ts

```typescript:vite.config.ts
import * as fs from 'fs';
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react';
import requireTransform from 'vite-plugin-require-transform';
import viteSentry from 'vite-plugin-sentry';

export default defineConfig(({ command, mode }) => {
  const env = loadEnv(mode, process.cwd(), 'REACT_APP_');
  const release = fs.readFileSync('../.version').toString();
  const packageJson: { dependencies: Record<string, string> } = JSON.parse(
          fs.readFileSync('package.json').toString()
  );

  return {
    server: {
      port: 3000,
    },
    build: {
      outDir: '../unlace-web-server/build',
      emptyOutDir: true,
      sourcemap: command === 'build' && env.REACT_APP_ENV === 'production' ? 'hidden' : true,
    },
    envPrefix: 'REACT_APP_',
    plugins: [
      (() => {
        const plugin = requireTransform();
        return {
          ...plugin,
          async transform(code: string, id: string) {
            const { code: newCode } = await plugin.transform(code, id);
            return {
              code: newCode,
              map: null,
            };
          },
        };
      })(),
      react({
        exclude: /\.stories\.tsx?$/,
        babel: {
          parserOpts: {
            plugins: ['decorators-legacy', 'classProperties'],
          },
        },
      }),
      !!process.env.SENTRY_AUTH_TOKEN &&
      command === 'build' &&
      viteSentry({
        debug: true,
        skipEnvironmentCheck: true,
        release: release,
        authToken: process.env.SENTRY_AUTH_TOKEN,
        org: XXX,
        project: XXX,
        deploy: {
          env: env.REACT_APP_ENV,
        },
        sourceMaps: {
          include: ['../unlace-web-server/build/assets'],
          ignore: ['node_modules'],
        },
      }),
    ],
    resolve: {
      dedupe: Object.keys(packageJson.dependencies),
      alias: [
        { find: 'src/', replacement: `${__dirname}/src/` },
        { find: 'lib/', replacement: `${__dirname}/node_modules/unlace-lib/src/` },
        { find: /^react-native$/, replacement: 'react-native-web' },
        {
          find: 'entities/maps/entities.json',
          replacement: `${__dirname}/node_modules/entities/lib/maps/entities.json`,
        },
        {
          find: 'entities/maps/legacy.json',
          replacement: `${__dirname}/node_modules/entities/lib/maps/legacy.json`,
        },
        { find: 'entities/maps/xml.json', replacement: `${__dirname}/node_modules/entities/lib/maps/xml.json` },
      ],
      extensions: ['.web.js', '.js', '.ts', '.jsx', '.tsx'],
    },
    optimizeDeps: {
      esbuildOptions: {
        resolveExtensions: ['.web.js', '.js', '.ts', '.jsx', '.tsx'],
      },
    },
    define: {
      __DEV__: command === 'serve',
      __SENTRY_RELEASE__: JSON.stringify(release),
      'import.meta.env.variables': JSON.stringify(env),
    },
  };
});

```

:::

:::details unlace-web/tsconfig.json

```json:tsconfig.json
{
    "compilerOptions": {
        "target": "ESNext",
        "useDefineForClassFields": true,
        "lib": ["DOM", "DOM.Iterable", "ESNext"],
        "allowJs": false,
        "skipLibCheck": true,
        "esModuleInterop": false,
        "allowSyntheticDefaultImports": true,
        "strict": true,
        "forceConsistentCasingInFileNames": true,
        "module": "ESNext",
        "moduleResolution": "Node",
        "resolveJsonModule": true,
        "isolatedModules": true,
        "noEmit": true,
        "jsx": "react-jsx",
        "baseUrl": ".",
        "experimentalDecorators": true,
        "paths": {
            "src/*": ["src/*"],
            "lib/*": ["../unlace-lib/src/*"],
        }
    },
    "include": ["src"]
}

```

:::

# ワークスペースの準備

`unlace-lib`へlinkを設定します。
これで`unlace-lib`を`add`したときにシンボリックリンクが貼られるようになります。

```shell
$ yarn link --private --relative ./unlace-lib
```

次に、`unlace-web`から`unlace-lib`を参照します。

```shell
$ yarn workspace unlace-web add unlace-lib
```

# Viteの導入作業

## パッケージをインストールする

Vite本体の他に、React用の公式のプラグインがあるためこちらも併せてインストールします。

```shell
$ yarn workspace unlace-web add vite @vitejs/plugin-react
```

## vite.config.tsを作成する

```shell
$ touch unlace-web/vite.config.ts
```

```typescript:vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig(({ command, mode }) => {
    return {};
});
```

## npm scriptsの起動コマンドを変更する

```diff json:package.json
"scripts": {
+    "dev": "vite dev",
},
```

## index.htmlをプロジェクトルートへ移動する

```shell
$ mv ./unlace-web/public/index.html ./unlace-web/index.html
```

craでは`index.html`は`public/`にありましたが、Viteではプロジェクトルートがデフォルトなのでこちらへ移動します。`public/`自体はViteでもそのまま利用可能なので、favicon等はそのまま残しておきます。

## index.htmlを編集する

`public/`に置いたファイルはルートの下に直接生えるので、craであった`%PUBLIC_URL%`は不要になります。
例は一部ですが他にも記述があるところは全て削除します。

```diff html:index.html
- <link rel="icon" href="%PUBLIC_URL%/favicon.ico" />
+ <link rel="icon" href="/favicon.ico" />
```

また、Viteではmoduleのエントリポイントが必要となるためこちらもbodyへ追加します。

```diff html:index.html
  <body>
    <div id="root"></div>
+   <script type="module" src="/src/index.tsx"></script>
  </body>
```

## @vitejs/plugin-reactを設定する

babelに追加したいものがあればここで一緒に指定できます。

```typescript:vite.config.ts
...
plugins: [
  ...
  react({
      exclude: /\.stories\.tsx?$/,
      babel: {
          parserOpts: {
              plugins: ['decorators-legacy', 'classProperties'],
          },
      },
  }),
  ...
],
...
```

## 環境変数にアクセスできる状態を作る

craでは`REACT_APP`というprefixを付けた環境変数にアクセスすることができましたが、Viteでは同様の仕組みを`VITE_`というprefixで提供しています。
ですが、今回はApp側になるべく変更を加えたくないので、prefixを`REACT_APP`に変更し、変更箇所を最小限に抑えています。

```typescript:vite.config.ts
...
envPrefix: 'REACT_APP_',
...
```

定義した環境変数へは`import.meta.env`というオブジェクトを通してアクセスすることが出来ます。
しかし、[ガイド](https://ja.vitejs.dev/guide/env-and-mode.html#production-replacement)によると
> プロダクションでは、これらの環境変数は、静的に置換されます。

ということで、indexerを用いて`import.meta.env['REACT_APP_ENV']`といった形で参照することは出来ないようです。

これまでは、環境変数の参照方法をWeb/Appで共通化する為ラッパーを挟んだ上でindexerを用いて`process.env[key]`という形でアクセスしていましたが、この形が使えないとなると他の手段を取るほかありません。
少々力技ですが、`define`も静的に置換されることを利用してビルド時に環境変数をオブジェクトにして置いてやることにしました。

```typescript:vite.config.ts
const env = loadEnv(mode, process.cwd(), 'REACT_APP_');
...
define: {
    'import.meta.env.variables': JSON.stringify(env),
},
```

これをアプリケーションコードで

```typescript
const env: any = import.meta.env.variables;
```

のように宣言しておくと、

```typescript
const env: any = {
    REACT_APP_ENV: "development",
    REACT_APP_BASE_URL: "https://...",
    ...
};
```

といった形に展開してくれます。

## 共通パッケージを読み込む

まずTypeScriptから参照できるように`tsconfig.json`に`paths`を追加します。

```json:tsconfig.json
...
"paths": {
    "src/*": ["src/*"],
    "lib/*": ["../unlace-lib/src*"],
}
...
```

次にViteでビルドする際のエイリアスを設定します。

```typescript:vite.config.ts
...
resolve: {
  alias: [
    { find: 'src/', replacement: `${__dirname}/src/` },
    { find: 'lib/', replacement: `${__dirname}/node_modules/unlace-lib/src/` },
...
```

これで`import MyComponent from 'lib/component/MyComponent';` という風に共通パッケージを参照できるようになりました。

## Web用のコードを読み込む

ReactNative向けに提供されているパッケージには、[react-native-gesture-handler](https://github.com/software-mansion/react-native-gesture-handler)のようにreact-native-web向けのコードが[同梱されている](https://unpkg.com/browse/react-native-gesture-handler@2.4.2/lib/commonjs/RNGestureHandlerModule.web.js)ことがあります。
create-react-appは[標準](https://github.com/facebook/create-react-app/blob/main/packages/react-scripts/config/paths.js#L37)でこういったコードを読み込んでくれますが、Viteにはそういった仕組みは無いため、`resolve.extensions`に追加しweb向けコードがある場合は優先的に読むようします。
また、`resolve.extensions`に追加するだけでは`vite dev`時に読み込んでくれないため、`optimizeDeps.esbuildOptions.resolveExtensions`にも同様の指定を追加しています。

```typescript:vite.config.ts
...
resolve: {
    ...
    extensions: ['.web.js', '.js', '.ts', '.jsx', '.tsx'],
},
...
optimizeDeps: {
    esbuildOptions: {
        resolveExtensions: ['.web.js', '.js', '.ts', '.jsx', '.tsx'],
    },
},
...
```

## 画像を読み込む為にrequireを書き換える

ReactNativeでは次のように`require`を使用して画像を読み込みます。

```typescript jsx
<Image source={require('icon.png')}/>
```

一方、Viteは[静的アセットの取り扱い](https://ja.vitejs.dev/guide/assets.html)にあるように`import`を使って最終的な画像のURLを得る仕組みになっています。
この違いを吸収するため、ビルド時に`require`を`import`に書き換えます。

書き換えにはこちらのプラグインを使用しました。
https://github.com/WarrenJones/vite-plugin-require-transform#readme

`plugins`に追加します。

```typescript:vite.config.ts
import requireTransform from 'vite-plugin-require-transform';
...
plugins: [
  (() => {
      const plugin = requireTransform();
      return {
          ...plugin,
          async transform(code: string, id: string) {
              const { code: newCode } = await plugin.transform(code, id);
              return {
                  code: newCode,
                  // https://rollupjs.org/guide/en/#thisgetcombinedsourcemap
                  map: null,
              };
          },
      };
  })(),
  ...
]
...
```

こちらのプラグインをそのまま使用するとSourceMapの出力時に次のエラーが発生したため、ワークアラウンドとして`map: null`を追加しています。
> Sourcemap is likely to be incorrect: a plugin (_vite_plugin_require_transform_) was used to transform files, but
> didn't generate a sourcemap for the transformation. Consult the plugin documentation for help

## パッケージの重複を解消する

[こちら](https://github.com/facebook/react/issues/13991)で言及されているように、`react`や`react-dom`には**アプリケーション全体でインスタンスは1つ**
という制限があります。 同様の制限は他のライブラリにも存在し、Unlaceでは状態管理に使用している`recoil`などがこれに該当しました。
この問題は`node_modules`に同じパッケージが複数箇所でインストールされてしまうことで発生しますが、発生の条件は様々で一概には言えません。
今回Unlaceでは`yarn link`によって`unlace-web/node_modules`に`unlace-lib`へのシンボリックリンクを置いた事でこちらの問題にあたりました。

`unlace-lib`は`unlace-web`の依存パッケージであると同時に、ワークスペース全体で見るとルート直下に存在する一つのパッケージでもあります。
そのため`yarn installl`すると、

- `unlace-web/node_modules/recoil`
- `unlace-lib/node_modules/recoil`

の2つがインストールされ、`unlace-web/node_modules/unlace-lib/src/.+`に置いたコードからは`unlace-lib/node_modules/recoil`の方が参照されてしまい、
冒頭で述べた問題が生じてしまいます。
:::message
`node_modules`のルックアップはrequireをしたファイルから親へ親へと遡るため、`unlace-lib`内からrequireすると先に`unlace-lib/node_modules/recoil`
の方が見つかってしまいます。
:::

こちらの問題の対応として、`resolve.dedupe`に重複を許可しないパッケージを指定しました。

```typescript:vite.config.ts
...
const packageJson: { dependencies: Record<string, string> } = JSON.parse(
    fs.readFileSync('package.json').toString()
);
...
resolve: {
...
  dedupe: Object.keys(packageJson.dependencies),
...
},

```

一つ一つ調べるのが面倒だったので、明示的にインストールしてあるパッケージは全て突っ込んでいます...
:::message
`react`と`react-dom`は`@vitejs/plugin-react`であらかじめdedupe指定されているため改めて追加する必要はありません。
:::

細かい部分は省略しましたが、概ねこちらの流れで移行が完了しました。

# その他にハマったところ

Unlaceではカウンセリングのチャット部分に[stream](https://getstream.io/chat/)を使用しているのですが、こちらのSDKが依存する`entities`
というパッケージがビルドの際に次のエラーを吐いていました。
> Uncaught TypeError: Failed to resolve module specifier "entities/maps/entities.json?commonjs-external". Relative
> references must start with either "/", "./", or "../".


`entities`は中でjsonをインポートしているのですが、これがViteのcjsの変換処理と相性が悪いようでした。
こちらは`resolve.alias`で読み込み先を絶対パスに変更することで回避できました。

```typescript:vite.config.js
...
resolve: {
  ...
  alias: [
    {
      find: 'entities/maps/entities.json',
      replacement: `${__dirname}/node_modules/entities/lib/maps/entities.json`,
    },
    ...
  ],
},
...
```

# 性能比較

移行を終えて、devサーバーの起動に掛かる時間がどのように変化したのか計測しました。
| | create-react-app | Vite |
|------|------------------|--------|
| 初回起動後ページが表示されるまで | 32.44秒 | 16.98秒 |
| HMR | 2.76秒 | 一瞬（！） |

Viteは開発サーバーの立ち上げは1秒程度完了しますが、ブラウザ側でESModuleの読み込みが完了してページが表示されるまでに少し時間が掛かります。 それでもcraの約半分の時間で済むので大分速くなりました。
また、HMRに関してはViteが圧倒的で、体感では保存した瞬間に書き換わっています。craではよっこらせという感じだったのですごい。
