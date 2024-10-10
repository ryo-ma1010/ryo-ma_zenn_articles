---
title: "pdfmeというライブラリにカスタマイズしてみた"
emoji: "🐰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["pdfme"]
published: true
published_at: 2024-10-10 12:00 # 未来の日時を指定する
---

# はじめに
OSSライブラリを個人的にカスタマイズすることになったので、どういう感じでカスタマイズしたのかを備忘録として記録します。
ローカル環境のpdfmeで再現できるところまでをゴールとしています。

# pdfというライブラリについて
自身の業務で、あるサービスに帳票をPDF出力する機能を作るにあたり、以下のライブラリを採用しています。
激推ししました。
https://pdfme.com/

pdfmeはTypeScriptで書かれたオープンソースの無料の帳票エンジンで、Kyohei Fukuda氏が開発しています。
https://zenn.dev/hand_dot

WYSIWYGエディターを有しており、帳票以外にもあらゆる書類が作成可能、PDF出力できます。
めちゃくちゃ使い勝手がよく、なんならライブラリをぶち込むだけで帳票機能の開発は終了するのではと思っていたのですが、どうしてもカスタマイズする必要がでてきました。

:::message
本記事ではpdfmeのバージョンは5.0.0を使っています。
:::

# なぜカスタマイズすることになったのか
サービス内の管理画面の受注詳細画面において、あるボタンを押すと帳票が出力されるという機能を開発しました。
出力される帳票には受注テーブルが表示されており、商品名や数量、価格が記載されています。
受注データは動的であり、ヘッダーに書かれた文字列("item"、"quantity"、"price")を頼りに、データが挿入されます。
サービス利用者は出力された帳票をなにかしらの業務に活かすことになるのですが、利用者によっては、出力された帳票のフォーマットを独自のものにしたくなるかもしれません。
帳票自体のフォーマットはpdfmeのエディター機能は現在利用しないことになっているので、編集不可としていますが、受注テーブルのヘッダーくらいは変更できるようにしたいです。
例えば、"quantity"ではなく"商品名"、"内容"、"quantity"ではなく"数量"、"price"ではなく"金額"、"単価"といった具合です。

つまり、受注データを挿入するためのたよりとなるヘッダーの文字列(item、quantity、price)とは別に、PDF出力用のヘッダーを用意する必要がでてきたのです。

pdfmeのgithubにてissueを投稿しようとも考えたのですが、あまり需要がなさそうだったため、独自でカスタマイズすることにしました。
そして、ローカルでカスタマイズできることを確認してから、開発中のサービスに展開しようと考えました。

# ローカル環境構築
pdfmeはgithubで公開されており、ローカル環境構築も簡単です。
https://github.com/pdfme/pdfme

pdfmeのリポジトリをローカルにcloneします。

```
$ git clone https://github.com/pdfme/pdfme.git
```

pdfmeのルートディレクトリで以下のコマンドを実行します。

```
$ npm install
$ npm run build
```

ルートディレクトリからplaygroundディレクトリに移動し、以下コマンドを実行します。

```
$ npm install
$ npm run dev
```

ブラウザにアクセスすると、pdmeのエディター画面が起動します。

![](/images/1ed3a34ecddaba_playground.png)


# 実際にカスタマイズしてみる
## フォーマットについて
pdfmeではPDFを出力するために、テキストやテーブル、矩形、さらにはQRコードといった形式が用意されています。
また、それらをPDF出力するためのフォーマットがJSON形式であらかじめ決められています。
今回は既存のテーブル機能を継承する形でcustomizeTableというスキーマを作り、そこにPDF出力用のヘッダーに相当する項目を設定できるようにしました。
おおよそ予想されるフォーマットは以下の形になります。

```typescript
{
    "schemas": [
        {
            "orders": {
                // カスタマイズ用のtype
                "type": "customizeTable",
                "head": [
                    "Item",
                    "Quantity",
                    "Unit Price",
                    "Total"
                ],
                "headStyles": {
                    "fontName": "NotoSerifJP-Regular",
                    "fontSize": 13,
                    // ...
                    // PDF表示用ヘッダーを追加
                    "displayHeaderNames": {
                        "Item": "商品名",
                        "Quantity": "数量",
                        "Unit Price": "金額",
                        "Total": "合計"
                    }
                },
            },
            // ...
        }
    ],
    // ...
}
```

## Plugin
pdfmeにスキーマを追加するために、Pluginを作成する必要があります。
そこで、 `schemas/src` 配下に追加スキーマ用ディレクトリを作ります。
そして、index.tsにスキーマを定義します。

```typescript:packages/schemas/src/customizeTables/index.ts
import type { Plugin } from '@pdfme/common';
import type { CustomizeTableSchema } from './types.js';
import { propPanel } from './propPanel.js';
import tableSchema from '../tables/index.js';
import { pdfRender } from './pdfRender.js';

const customizeTableSchema: Plugin<CustomizeTableSchema> = {
  pdf: pdfRender,
  ui: tableSchema.ui,
  propPanel,
  icon: '<svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-table"><path d="M12 3v18"/><rect width="18" height="18" x="3" y="3" rx="2"/><path d="M3 9h18"/><path d="M3 15h18"/></svg>',
};
export default customizeTableSchema;
```

以下に、Pluginの引数を解説します。

### pdf
PDFを出力するための処理を書きます。
0から作るのはしんどいので、ベースは既存のtableスキーマを使用しています。
ヘッダーをPDF出力用のヘッダーに変換してtableスキーマ内のpdfRender()を呼びます。

```typescript:packages/schemas/src/customizeTables/pdfRender.ts
import { PDFRenderProps } from "@pdfme/common";
import { CustomizeTableSchema } from "./types.js";
import { pdfRender as parentPdfRender } from "../tables/pdfRender.js";

export const pdfRender = async (arg: PDFRenderProps<CustomizeTableSchema>) => {
const {schema, ...rest} = arg;

// ヘッダー書き換え
if (schema.headStyles.displayHeaderNames !== undefined) {
    for (const [key, head] of schema.head.entries()) {
    if (schema.headStyles.displayHeaderNames[head] === undefined) continue;
    if (schema.headStyles.displayHeaderNames[head] === '') continue;
    schema.head[key] = schema.headStyles.displayHeaderNames[head];
    }
}

const renderArgs = {schema, ...rest};

await parentPdfRender(renderArgs);
};
```

### ui
pdfmeのエディター上での処理を書きます。
今回はtableスキーマから変更はないので `tableSchema.ui` を指定します。

### propPanel
ここではエディターの右側に表示されるスキーマ特有の入力項目を定義します。
こちらも引数pdfと同様で、ベースはtableスキーマのまま、customizeTable用の項目を追加します。

```typescript:packages/schemas/src/customizeTables/propPanel.ts
import { propPanel as parentPropPanel } from "../tables/propPanel.js";
import { PropPanel, PropPanelWidgetProps } from '@pdfme/common';
import { CustomizeTableSchema } from './types.js';
import tableSchema from '../tables/index.js';


export const propPanel: PropPanel<CustomizeTableSchema>  = {
schema: ({ activeSchema, options, i18n }) => {
    const propPanelProps = {activeSchema, options, i18n};
    // @ts-ignore
    const parentSchema = parentPropPanel.schema(propPanelProps);
    const parentHeadStyles = parentSchema.headStyles;
    const headProperties = parentSchema.headStyles.properties;
    // @ts-ignore
    const head = activeSchema.head;
    return {
    ...parentSchema,
    headStyles: {
        ...parentHeadStyles,
        properties: {
        ...headProperties,
        // これを追加したかった
        '---': { type: 'void', widget: 'Divider' },
        displayHeaderNames: {
            title: 'display header name',
            type: 'object',
            widget: 'lineTitle',
            column: 3,
            properties: getDisplayHeaderNamesSchema(head),
        },
        }
    },
    }
},
// custmizeTableの初期表示
defaultSchema: {
    ...tableSchema.propPanel.defaultSchema,
    // CustomizeTableSchema extends TableSchema extends Schema
    // @ts-ignore
    type: 'customizeTable',
    content: JSON.stringify([
    ['Apple', '1', '100'],
    ['Banana', '10', '2000'],
    ['Chocolate', '2', '300'],
    ]),
    head: ['Name', 'Quantity', 'Price'],
    headStyles: {
    ...tableSchema.propPanel.defaultSchema.headStyles,
    displayHeaderNames: {
        'Name': '商品名',
        'Quantity': '数量',
        'Price': '価格',
    },
    },
},
};

const getDisplayHeaderNamesSchema = (head: string[]) => {
return head.reduce((acc, cur, i) => Object.assign(acc, {
    [cur || 'Column ' + String(i + 1)]: {
    title: cur || 'Column ' + String(i + 1),
    type: 'string',
    props: {},
    },
}), {});
};
```
### icon
エディター上での左側に表示されるアイコンを指します。

型については、tableスキーマの型をextendsした上でcustomizeTable固有のプロパティを追加しました。

## table系スキーマ固有の処理
PDF出力される際、スキーマごとの位置や高さを計算する処理があるのですが、tableスキーマは高さの計算のみ、他のスキーマとは違う処理を経由します。
詳細は `packages/schemas/src/tables/dynamicTemplate.ts` を参照してほしいのですが、今回のカスタマイズでcustomizeTableスキーマにもまったく同じものを作りました。

## そのほか
他、 `packages/schemas/src/` 配下のファイルをいくつか編集しました。

# 実装
ローカル環境のplaygroundで期待通りに動作していることが確認できました。

![](/images/1ed3a34ecddaba_customizeTable.png)
*customizeTableスキーマとその設定項目がエディター上で表示できている(赤枠)*

![](/images/1ed3a34ecddaba_customizeTable_pdf_rendar.png)
*PDF出力時にヘッダー名が置き換えられている(赤枠)*

# 開発中のサービスに適用する
今度は開発中のサービスに適用させます。
pdfmeはnpmで提供されているパッケージなので、パッケージ本体は `node_modules` 配下にあります。
しかし、node_modulesはgit管理されていない、また、 `npm install` してしまうとnode_modules内のパッケージは上書きされてしまうため、node_modules配下を編集することはできてもそれをもとに運用することは厳しいです。

## patch-package
そこで、 `patch-package` というパッケージをインストールします。
こちらのパッケージは、npmの編集したいパッケージを編集してそのときの差分をパッチファイルとして記録することができます。その上で `npm install` すると、該当のパッケージが上書きされた状態で利用することができます。

https://www.npmjs.com/package/patch-package

patch-packageの詳細の解説は別記事化予定です。

# おわりに
以上が、pdfmeのカスタマイズの解説でした。
pdfmeは大変使いやすく、PDF機能を実装したい方はまず試してほしいです。
Kyohei Fukuda氏には大変感謝しています！！