---
title: "【Next.js + GAS】研究室のホームページをスプレッドシートを使って動的に生成し運用した"
emoji: "🏠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "gas", "googlespreadshe"]
published: true
---

自身の所属する研究室にまだウェブサイトがなく、活動が報告できるプラットフォームが増えればとなり作成・リリースした件について、実際のソースコードと共に仕様と実装方法についてまとめました。

今回の記事は、設計を考慮した実装方法～リリース、初期運用までを書くため長くなります。

## 結論

はじめに、この記事の結論となるサイト・リポジトリを提示します。

### ホームページ

@[card](https://sai.ac)

### GitHub

https://github.com/kcct-rtakada/sai_lab_web

## 目的

### 達成したい内容

- 研究室のメンバーや論文誌等、またはニュースの更新を動的に行う
- スプレッドシートのように表で管理することでメンテナンス性を上げる
- ユーザビリティを高める

この3点を重視したサイトを構築するため、タイトルの通り Next.js と Google Apps Script を利用しました。

## 利用した技術

- **Next.js**

@[card](https://nextjs.org/)

- **Google Apps Script**

@[card](https://workspace.google.co.jp/intl/ja/products/apps-script/)

- **TypeScript・SCSS**

CSS(SCSS) は直書き。アイコンは Font Awesome の Free Icons を用いました。アイコンを使う場合は Bootstrap Icons を使うといった、統一感を持たせることが重要と考えます。

@[card](https://fontawesome.com/icons)

- **Vercel**

技術というよりサービスですが。今回のデプロイするためのサービスです。

@[card](https://vercel.com/)

:::message
この記事はv1.0.0を基に執筆しています。
:::

https://github.com/kcct-rtakada/sai_lab_web/tree/v1.0.0

### 選定理由

- Next.jsはWebページのフレームワークとして知名度が高く、サーバーサイドのレンダリングや静的なサイトを作る両面において優れているため
- 自身が製作した後に他のコントリビューターが機能追加・修正を行うにあたり、ドキュメントが多いフレームワークである
- キャッシュ時間の調整が容易
- GASはGoogleのサービスでスプレッドシートと連携可能なため、新たに学習するコストが低く、表として全体を見ながら管理が可能

## 設計

設計の初期段階、夢を語るフェーズではホワイトボード等を使って何ができたら嬉しいか？を考えます。何のページが必要でどんな情報を表示すべきかを考えます。運用方法は特に重要で、記事更新のコストをできるだけ下げることも重要です。

## 開発(TypeScript)

※本研究室の教員より許可を頂き、以降の画像・文章を掲載しています。

v1.0.0を発行するまでの開発手順は、

1. TypeScriptによるGASの動作確認
2. プロジェクト一覧の表示
3. プロジェクト詳細ページの設計
4. ニュースの設計
5. その他ページの作成
6. クライアントコンポーネントのパーツを設計

で行いました。

```:最終的なディレクトリ構造
├─public
└─src
    ├─app
    │  ├─award
    │  ├─contact
    │  ├─member
    │  ├─news
    │  │  └─[slug]
    │  ├─project
    │  │  └─[slug]
    │  ├─publication
    │  ├─thesis
    │  └─[...notfound]
    ├─components
    │  └─... (略)
    └─styles
        ├─app
        │  ├─award
        │  ├─contact
        │  └─... (略)
        ├─components
        └─global
```

### GASのセットアップ

今回は4種類の情報を別々に取得するようにしました。

- メンバー
- プロジェクト
- ニュース
- 表彰

操作は同じなため、ニュースを使って説明します。

#### ニュース用JSONの作成

スプレッドシートで次のタイトル行を作り、2行目から実際のデータを記述しました。

| a   | b    | c        | d                | e             | f                        | g                         |
| --- | ---- | -------- | ---------------- | ------------- | ------------------------ | ------------------------- |
| id  | 日付 | タイトル | 内容(HTML記法可) | サムネイルURL | 関連リンク (区切り「;」) | 追加画像URL (区切り「;」) |

idは数値以外にも自由に文字列が設定できるようにします。

本文(内容)はユーザが`<a>`タグや`<detail>`タグを挟めるよう、文字列のまま記入するようにします。

目的はスプレッドシートの内容をJSONの形式に変換することです。リボンから`拡張機能`→`Apps Script`の順にアクセスします。
`コード.js`が生成されたら以下のコードを記述しました。

```js:コード.js
function readData() {
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const target_sheet = spreadsheet.getSheetByName('News');
  const last_row = target_sheet.getLastRow();
  const range = target_sheet.getRange(`A2:G${last_row}`);
  const data_values = range.getValues();

  const json_data = data_values.map((data_row) => {
    // ";"で分割しリスト化
    const formatToObjects = (str) => str.split(';').map(name => ({ name }));

    let linksArray = formatToObjects(data_row[5]); // F列
    if (linksArray.length === 1 && linksArray[0].name === "")
    linksArray = []

    let imagesArray = formatToObjects(data_row[6]); // G列
    if (imagesArray.length === 1 && imagesArray[0].name === "") 
    imagesArray = []

    return {
      id: data_row[0], // A列
      date: data_row[1], // B列
      title: data_row[2], // C列
      article: data_row[3], // D列
      thumbnailURL: data_row[4], // E列
      links: linksArray,
      additionalImageURL: imagesArray,
    }
  });
  console.log(json_data) 
  return json_data;
}
 
function doGet() {
    const data = readData();
    const response = ContentService.createTextOutput();
    response.setMimeType(ContentService.MimeType.JSON);
    response.setContent(JSON.stringify(data));
    return response;  
}
```

`linksArray` や `imagesArray` で使っている `formatToObjects()` 関数は、文字列を `;` で分割し、リスト化しています。
そして、配列の長さが1・インデックスが0番目の文字列が空文字の場合、すなわち空セルの場合は要素を0個と書き換えます。

動作確認は`▷ 実行`で可能です。
準備が完了したら`デプロイ`よりウェブアプリとしてデプロイします。
ニュース以外も同様に作ります。
[ニュースの生成結果はこちら](https://script.google.com/macros/s/AKfycbwZdJd0gov7nPMMXXBh_q7btSZuS-K93LAy5sjRFEGgnpK3DlrRYee2st2Wv7GVjwCo/exec)(デプロイ結果)

### Next.jsの準備

`create-next-app` から Next.js プロジェクトを立ち上げます。パッケージ管理システムは `pnpm` で、 [App Router](https://nextjs.org/docs/app) を使います。

@[card](https://nextjs.org/docs/app/api-reference/create-next-app)

### GASからJSONデータの読み出し

実際にデータを読み出します。

```tsx:src/components/GASFetch.tsx(抜粋)
// 5分ごと
export async function fetchNews() {
  const response = await fetch(sai_news, {
    next: { revalidate: 300 },
  });
  return response;
}
```

`fetch` 関数に、`sai_news=(先ほどのURL)`を第一引数として代入し、第二引数に、

```json
{
  next: { revalidate: 300 },
}
```

を代入しました。第二引数の意図について次節で解説します。

### (補足)Cachingの対策

Next.js では Vercel にデプロイした後、一度キャッシュしたデータは永続的に保持されます。

> **Duration**
The Data Cache is persistent across incoming requests and deployments unless you revalidate or opt-out.
(https://nextjs.org/docs/app/building-your-application/caching#duration-1)

> **持続時間(訳)**
データキャッシュは再検証またはオプトアウトしない限り、リクエストの受信やデプロイメントに渡り永続的だ。

今回の目的である動的な更新にはニュースも含まれています。
ニュースを投稿して10分もあれば反映されていたいと筆者は考えます。

- **オプトアウト**

引用のように、オプトアウトすれば常に最新の状態を得ることが出来ます。キャッシュを保持しないことで新しいデータを探しにいこうとするからです。
ただ、GAS の api 応答時間や呼び出し回数制限が問題となり、ページ描画までの快適さが失われます。

@[card](https://developers.google.com/apps-script/guides/services/quotas?hl=ja)

- **再検証**

引用に、もう一方の手法も提示されていました。
時間に基づいた再検証とオンデマンド再検証の2パターンがあり、特に**時間に基づいた再検証**は目的に近い実現方法です。

![How Time-based Revalidation Works](https://nextjs.org/_next/image?url=%2Fdocs%2Flight%2Ftime-based-revalidation.png&w=1920&q=75&dpl=dpl_EtFYoHX2pVHk4GSBAfEPDev3kkHk)
*How Time-based Revalidation Works
(引用: https://nextjs.org/docs/app/building-your-application/caching)*

初回取得はData Sourceより読み出し、キャッシュに格納します。次回以降のアクセスはキャッシュにあるデータを表示します。
そして、revalidate する際のアクセスにはキャッシュの内容を表示させ、裏でデータを取得しキャッシュへ格納します。
したがって、自然な流れでデータを更新することが可能です。

この方法を有効にするため、[GASからJSONデータの読み出し](#gasからjsonデータの読み出し)では第二引数に JSON 形式を代入していました。
今回の場合は300とあるので300s=5分となり、5分おきに最新のデータが反映されます。

### プロジェクト一覧の取得

データ取得が可能になったので、プロジェクトページの方を進めます。
プロジェクト一覧での要件は以下の通りです。

- 部分一致による検索機能(OR: 論理和)
- 年度ごとに表示分け
- 類似性の高いプロジェクトのグループ化

表示結果が動的に変化するよう、クライアントコンポーネント化します。

https://github.com/kcct-rtakada/sai_lab_web/blob/v1.0.0/src/app/project/page.tsx#L18-L44

まず、`/project/`自体はサーバーコンポーネントなのでプロジェクトの JSON を取得し、リスト化します。リストに空のデータが入っている可能性を解決すべく、`item.id !== ""`でフィルターの条件を設定します。

そして`<ProjectViewer />`を設け、描画部を設計します。

### プロジェクト一覧の表示

まずは、プロジェクトの JSON データの型を示します。

```js:src/components/DefaultStructure.tsx(抜粋)
export interface Author {
  name: string; // 著者名
}
export interface Tag {
  name: string; // キーワード
}
export interface AdditionalImage {
  name: string; // 追加で表示する画像のURL
}
export interface Project {
  id: string; // URLの一部にもなる識別文字列
  classification: string; // 論文誌, 国際会議等
  type: string; // 国際査読あり/なし等
  title: string; // 研究名など
  authors: Author[]; // 著者名のリスト
  tags: Tag[]; // キーワードのリスト
  bookTitle: string; // 論文誌(予稿集)の情報はじめ
  volume: string;
  number: string;
  pageStart: number;
  pageEnd: number; // 論文誌(予稿集)の情報おわり
  date: Date; // 公開日
  abstract: string; // 概要説明
  url: string; // 関連リンク(1件)
  citation: string; // 引用文
  paperUrl: string; // ドキュメントURL
  thumbnailURL: string; // サムネイル画像URL
  presentationHTML: string; // プレゼンテーション用HTML記述欄
  documentHTML: string; // ドキュメント用HTML記述欄
  posterHTML: string; // ポスター等用HTML記述欄
  freeHTML: string; // その他HTML記述欄
  additionalImageURL: AdditionalImage[]; // 追加画像の配列
}
```

以降の記事で多く出現するため、どんな内容が入るかコメントで記述しました。

https://github.com/kcct-rtakada/sai_lab_web/blob/v1.0.0/src/components/project_list/ProjectsViewer.tsx#L387-L511

埋め込み表示部がリストを使って描画する記述です。

#### グループ化

グループ化について、
**サムネイルの画像URLが同一なものをグループとして扱う**ようにします。

![グループ化したプロジェクト](/images/next-gas-laboratory/zenn-sai-2.png)
*グループ化したプロジェクト*

```tsx:src/components/project_list/ProjectsViewer.tsx(抜粋)
function groupByThumbnail(array: Project[]) {
  var grouped: { [name: string]: Project[] } = {};
  // サムネイルパスが一致した場合にグループ化する
  array.forEach((item) => {
    if (item.thumbnailURL !== "" || item.thumbnailURL) {
      // キーが無い場合は作成
      if (!grouped[item.thumbnailURL]) {
        grouped[item.thumbnailURL] = [];
      }
      grouped[item.thumbnailURL].push(item);
    } else {
      // デフォルトサムネイルの場合
      if (!grouped[item.id]) {
        grouped[item.id] = [];
      }
      grouped[item.id].push(item);
    }
  });
  return grouped;
}
```

:::message
「デフォルト値 = 初期サムネイル表示状態」は空文字でグループ化されないように分岐が必要です。
:::

このように辞書型で作成することで
`{文字列キー: プロジェクト配列}`
が実現できます。

### (補足)表示年度の算出

学校内の研究室なので、年度ごとに表示をしたいです。

まず、年度の計算は次のとおりです。

```tsx
const fiscalYear: number = time.getMonth() + 1 > 3 ?
                           time.getFullYear() :
                           time.getFullYear() - 1;
```

1月～12月が0～11と割り当てられているため、4月の場合はgetMonth メソッドは3を返します。

そのため、特定の年度でフィルタリングする場合は次のようになります。

```tsx:src/components/project_list/ProjectsViewer.tsx(抜粋)
const matchedDataWithYear = displayArray?.filter((item) => {
  const japanTime = new Date(
    new Date(item.date).toLocaleString("en-US", {
      timeZone: "Asia/Tokyo",
    })
  );
  // 年度を計算
  return (
    (japanTime.getMonth() + 1 > 3
      ? japanTime.getFullYear()
      : japanTime.getFullYear() - 1) === year
  );
});
```

#### `japanTime`について

スプレッドシートによって日付が ISOフォーマット(ISO 8601) に変換されたものは、`new Date()`でもう一度 Date 型にできますが、スプレッドシートの時点でGMT+0になっており、日本標準時になっていません。

```tsx
const japanTime = new Date(
  new Date(item.date).toLocaleString("en-US", {
    timeZone: "Asia/Tokyo",
  })
);
```

このコードでタイムゾーンを無理やり変更することで、日本で閲覧した際のズレを修正します。

### プロジェクト一覧における検索

検索システムの要件は

- スペースで区切ってキーワード化して、論理和で結果を表示する
- 入力後、エンターキーで検索
- 検索条件を設定できるようにする

その中で、多くの人が直感的に分かっていただけるように、

- 検索条件はドロップダウンリストを用意する
- 入力フィールドとドロップダウン、検索ボタン(クリアボタン)で構成する

リストはOS依存のドロップダウンで表示して、選択中を表示するボタンだけ`appearance: none;`でCSSを設計します。

#### 検索時にエンターキーを使う

入力フィールド選択中にエンターキーを押すことで検索を実行させます。

```tsx:src/components/project_list/ProjectsViewer.tsx(抜粋)
<input
  title="検索条件を入力"
  value={searchWord}
  placeholder={
    initialQ
      ? `クリアはXを${isUsingPhone ? "タップ" : "クリック"}`
      : `${isUsingPhone ? "タップ" : "クリック"}して入力`
  }
  type={"text"}
  className={`${styles.search_input}`}
  onInput={triggerSearchInput}
  onKeyDown={handleEnterKeyPress}
/>
```

`onKeyDown={handleEnterKeyPress}`内で`Enter`キーか判別し、空文字でない時のみ検索します。

#### 検索を実行する関数

https://github.com/kcct-rtakada/sai_lab_web/blob/v1.0.0/src/components/project_list/ProjectsViewer.tsx#L75-L195

検索は`mode`というドロップダウンリストから選択された条件で分岐し、それぞれフィルター関数を記述します。

##### 入力フィールドから検索キーワードを作る

```tsx:src/components/project_list/ProjectsViewer.tsx(抜粋)
const filterKeywords = _searchWord.split(/[ 　]+/);
```

`[]`の内部には半角スペースと全角スペースがあり、入力フィールド内のそれら全てを区切り文字としてリスト化します。

```tsx:src/components/project_list/ProjectsViewer.tsx(抜粋)
if (filterKeywords.every((keyword) => keyword === "")) {
  setUserFiltered(false);
  router.push(`/project/`);
  setDisplayingSearchCondition(null);
  return;
}
```

:::message
フィールドにスペースしか入力されていない時はリストが全て空文字のはずです。
この場合は、上のようにリセットします。
:::

##### タイトルで検索

```tsx:src/components/project_list/ProjectsViewer.tsx(抜粋)
filteredArray = lists?.filter((project) =>
  filterKeywords.some(
    (keyword) =>
      keyword.toLowerCase() !== "" &&
      project.title.toLowerCase().includes(keyword.toLowerCase())
  )
);
```

キーワードと各プロジェクトで二重ループを作り、その中で1つでも真が返ることでフィルタ対象となります。

- キーワードが空でないか
- タイトル(Lower Case)とキーワード(Lower Case)が部分一致するか

この2条件の論理積を判定しています。

##### 著者名で検索

```tsx:src/components/project_list/ProjectsViewer.tsx(抜粋)
filteredArray = lists?.filter((project) =>
  filterKeywords.some((keyword) =>
    project.authors.some(
      (author) =>
        keyword.toLowerCase() !== "" &&
        (author.name
          .toLowerCase()
          .replace(/[ 　]+/, "")
          .includes(keyword.toLowerCase()) ||
          author.name
            .toLowerCase()
            .split(/[ 　]+/)
            .reverse()
            .join("")
            .includes(keyword.toLowerCase()))
    )
  )
);
```

検索原理は「タイトル」の時と同じです。異なる点は、

- `authors`も配列なため、三重ループで検索
- `author`は First name と Last name の検索順序が不明なため、スペース区切りの配列で入れ替えた文字列も検索対象
- `author`文字列は検索に問題がないとして、スペースを切り詰め

の3つです。

これ以外はキーワードと公開年ですが、authors の切り詰めない版 & 数値一致であるため省略します。

### (補足)ページジャンプ時の初期検索

ページ読み込み時に検索結果を表示したい為(リンク共有や著者検索など)、URL パラメータを2つ設けページロード時に処理します。

```tsx:src/components/project_list/ProjectsViewer.tsx(抜粋)
const router = useRouter();
const params = useSearchParams();
const initialMode = params.get("mode");
const initialQ = params.get("q");
```

例：`/project?mode=name&q=センシング,メガネ` ↓
`「センシング」または「メガネ」で研究題目を検索する`

:::details useSearchParams の使用について
useSearchParams を使用する場合は、`<Suspense>`で囲む必要があるとされています。
実際、私が`next build`した際にエラーが出力されました。

@[card](https://nextjs.org/docs/messages/missing-suspense-with-csr-bailout)

ここで、`<Suspense />`のプロパティに`fallback?`を設定できるため、
[プロジェクト一覧の取得](#プロジェクト一覧の取得)ではLoadingアニメーションが出力されるように設定していました。
:::

```tsx:src/components/project_list/ProjectsViewer.tsx(抜粋)
useEffect(() => {
  // 前略
  if (projects) {
    setLoaded(true);

    if (initialQ) {
      const initialSearchWord = initialQ.replace(/,/g, " ");
      setSearchWord(initialSearchWord);
      let mode: string | null = null;
      if (initialMode === "mode") {
        mode = "research_name";
      } else if (initialMode === "author") {
        mode = "research_author";
      } else if (initialMode === "keyword") {
        mode = "research_tag";
      } else if (initialMode === "year") {
        mode = "research_year";
      }
      searchProjects(initialSearchWord, mode, projects);
      setUserFiltered(true);
    }
  }
}, []);
```

例にある通り、`q`はカンマ区切りにしたため、カンマをスペースに置き換えることで、1つの文字列へと変換します。`mode`も分岐で対応させ、`searchProjects`:(検索関数)へ引数を渡します。

ここで引数にわざわざセットしているのは、レンダリング毎に検索されるわけではなく、関数によって1度だけ検索されるようにしているためです。

### プロジェクト詳細の表示

一覧から飛んできた先、詳細ページを作っていきます。

プロジェクトの検索は`find`で探します。

```tsx:src/app/project/[slug]/page.tsx(抜粋)
const project: Project | undefined = filteredProjects.find(
  (c: { id: string }) => {
    const cid = String(c.id);
    return cid === slug;
  }
);
```

ファイルパスにもあるように、`[slug]`であるため、「`slug`と一致するか」でプロジェクトを決定します。( string にキャストしておかないと、id が数値の時に比較できなくなります。)

#### `string`からHTML要素へ変換する

基本的に、`project.abstract ? 真の内容 : <></>`等を中心に書いています。
ただ、`documentHTML`や`presentationHTML`のような～HTMLの変数はHTMLタグが含まれる可能性があります。

今回は、`html-react-parser`パッケージを使ってHTML要素に変換しました。

@[card](https://www.npmjs.com/package/html-react-parser)

```tsx:src/app/project/[slug]/page.tsx(抜粋)
import parse from "html-react-parser";
```

を記述したファイルで、

```tsx:src/app/project/[slug]/page.tsx(抜粋)
{project.posterHTML ? (
  <>
    <h2 id="poster" className={styles.section_name}>
      Poster
    </h2>
    <div className={`${styles.slide_box} ${styles.poster}`}>
      {parse(project.posterHTML)}
    </div>
  </>
) : (
  <></>
)}
```

このように記述することで実際に変換されます。

### (補足)ニュース画面の設計

ニュース画面は、

- [プロジェクト一覧の表示](#プロジェクト一覧の表示)
- [プロジェクト一覧における検索](#プロジェクト一覧における検索)
- [プロジェクト詳細の表示](#プロジェクト詳細の表示)

とアルゴリズム等は全く同じなため記事では省略いたします。

### 学位論文・研究業績・表彰といったリストの表示

この3つのページは構造がかなり似ているため、まとめて記述します。代表して`publication: 研究業績`のページを基にします。

基本的にサーバーコンポーネントで問題ないです。

#### プロジェクトからリストする対象を絞り込む

```tsx:src/app/publication/page.tsx(抜粋)
const sortedConferencePapers = filteredProjects?.filter(
  (element) =>
    element.classification.toLowerCase().includes("国内会議") ||
    element.classification.toLowerCase().includes("国際会議") ||
    element.classification.toLowerCase().includes("論文誌")
);
```

表示対象はいずれかの単語が入っている(と信じている)ので、`includes`の戻り値、boolean で判定します。

#### 年度リストを作成する

年度ごとに表示を分けます。今から作るリストは[目次の設計](#補足目次の設計)でも活用します。

```tsx:src/app/publication/page.tsx(抜粋)
const uniqueYears = Array.from(
  new Set(
    sortedConferencePapers?.flatMap((item) => {
      const japanTime = new Date(
        new Date(item.date).toLocaleString("en-US", {
          timeZone: "Asia/Tokyo",
        })
      );
      return japanTime.getMonth() + 1 > 3
        ? japanTime.getFullYear()
        : japanTime.getFullYear() - 1;
    })
  )
);
```

年度の判定方法は[表示年度の算出](#補足表示年度の算出)と同じです。`flatMap`関数が 1次元配列を作成するので一致したら配列に追記するようにします。

あとは表示年度でループを回して種類ごとに表示するだけで目標の表示まで辿り着きます。

#### (補足)目次の設計

学位論文・研究業績・表彰ページで表示される年度ごとの目次です。(プロジェクト詳細ページの左サイドバーも仕様は似ています)
[年度リストを作成する](#年度リストを作成する)で作成した`uniqueYears: number[]`をコンポーネントを渡して表示させます。

https://github.com/kcct-rtakada/sai_lab_web/blob/v1.0.0/src/components/client_parts/YearListSidebar.tsx

### メンバーの表示

研究室のメンバーを表示します。要件は以下の通りです。

- 名前と英語名を表示する
- 個人のホームページや GitHub のリンクがあるなら表示する
- 異体字等があるなら示す
- 各個人のプロジェクトが直接検索できるようにする

ちなみにリストは3つに分けて表示しますが、条件は

1. 教員: 所属(`belonging`)に「教員」が入っているか
2. 在籍中: 1 と 3 を除いたもの
3. 卒業/修了: 所属(`belonging`)に「卒」または「修」が入っているか

そのため、何にも当てはまらなかった場合にも **2** へ振り分けられます。

```tsx:在籍中のメンバーを抽出する：src/app/member/page.tsx(抜粋)
const sortedEnrolledMember = members?.filter((element) => {
  return (
    !sortedMemberWithGraduation?.some(
      (sortedMember) => sortedMember.id === element.id
    ) &&
    !sortedMemberWithTeacher?.some(
      (sortedMember) => sortedMember.id === element.id
    )
  );
});
```

3つのグループに分けた後はHTML要素を戻り値とする関数で効率的に描画するよう試みました。(`/publication/`,`/thesis/`も同様)

https://github.com/kcct-rtakada/sai_lab_web/blob/v1.0.0/src/app/member/page.tsx

### (補足)異体字等の対策とプロジェクト検索

メンバー表示から検索する際に、1つの問題が発生しました。異体字等がプロジェクトのauthorとして記載される場合、検索結果に出てこれなくなる点です。

例: 髙田/高田 (「髙」は環境依存文字)

リンクの自動生成を考慮して表の列に「異体字等」を追加しました。

| 名前 | 異体字等 |
| ---- | -------- |
| 髙田 | (高田)   |

上のように記入して、検索のパラメータに含ませるようにしました。

- Q. なぜカンマを使わずにカッコを使ったのか。
A. スプレッドシートで一覧を眺めるときに、カッコを使えば別表記として用意していることが判別しやすいため。

#### カッコからの抽出

`(高田)(John)`のような文字列から抽出するため、以下の方法で文字列を結合しました。

```tsx:src/app/member/page.tsx(抜粋)
item.otherName
.match(/\([^()]+\)/g)  // 括弧内のテキストを抽出
?.flatMap((match) => match.split(",")) // カンマで区切る(対策)
.flatMap((match) => match.slice(1, -1)) // カッコを取り除く
.map((match) => match.replace(/[ 　]+/g, " ")) // スペースを切り詰める
.join(",") // 文字列を結合
```

#### 検索リンクの作成

```tsx:src/app/member/page.tsx(抜粋)
<Link
  href={`/project?mode=author&q=${
    item.name.replace(/[ 　]+/, "") // 名前
    },${item.englishName.replace(/[ 　]+/, "") // 英語名
    },${item.englishName.split(/[ 　]+/).reverse().join("")}${ // 英語名(順序入れ替え)
    item.otherName // 異体字等
      ? `,${item.otherName // 真なら
          .match(/\([^()]+\)/g)
          ?.flatMap((match) => match.split(","))
          .flatMap((match) => match.slice(1, -1))
          .map((match) => match.replace(/[ 　]+/g, ""))
          .join(",")}`
      : "" // 偽なら
  }`}
  className={styles.search_link}
  title="プロジェクトを検索"
>
{/* 中略 */}
</Link>
```

この文字列結合を href 内で展開すると少し見にくくなりますが、
1つのリンクでほぼ全てのプロジェクトが検索結果に含められます。

:::message
このリンクはプロジェクト検索で名前のスペースを切り詰める経緯でもあります
:::

### トップページの作成

ついにトップページです。要件は次の通りです。

- 新着ニュース5件の表示
- 言語切り替えボタンの設置(日<=>EN)

ニュースが5件に満たない場合の対策を記述すると、

```tsx:src/components/client_page/HomeContent.tsx(抜粋)
const listingNum = newsList ? (newsList.length > 5 ? 5 : newsList.length) : 0;
```

このようになります。

#### 言語切り替え

Next.js は i18n のサポートがされており、言語ごとのページ作成が容易です。しかし、プロジェクトやニュースを自動翻訳させると訳が意図に反する可能性も潜在するため、2つの固定ページにボタンを作成してトグルする運びとなりました。

@[card](https://nextjs.org/docs/pages/building-your-application/routing/internationalization)

ボタンには`onClick`プロパティを付与して関数と結びつけます。

```tsx:src/components/client_page/HomeContent.tsx(抜粋)
const displayString = (japaneseString: string, englishString: string) => {
  return <>{usingJapanese ? japaneseString : englishString}</>;
};
```

これをレンダリング内に差し込むことで実現しました。

### 右のサイドバーをスクロールと被らない位置に設置・設計

個人的に困っていた点の1つです。プロジェクト詳細ページの右サイドバーに関連プロジェクトを表示するまでは決定しており、背景色も用意することにしていました。

```css:サイドバーのイメージ
.sidebar {
  position: fixed;
  right: 0;
  background-color: #fafafa;

  /*その他プロパティを記述*/
}
```

一部のブラウザー(Firefox・Opera・Safari)では問題ありませんが、スクロールバーが埋もれてしまいます。
`right: 17px`のように、実測値で記述しても拡大率の影響でズレが発生してしまいます。

スクロールバーが見えないとページに続きがあるのか分からなく、UX の低下に繋がります。

そのため、**リサイズするたび、スクロールバーの幅を計算してずらす機構**が必要だと判明しました。
クライアントコンポーネント化し、`useEffect`内でイベントリスナーと関数を実装しました。

```tsx:src/components/project_detail/ProjectRightSidebar.tsx(抜粋)
useEffect(() => {
  // スクロールバーの幅を実際に要素を生成して算出する
  function getScrollbarWidth() {
    const outer = document.createElement("div");
    outer.style.visibility = "hidden";
    outer.style.overflow = "scroll";
    document.body.appendChild(outer);

    const inner = document.createElement("div");
    outer.appendChild(inner);

    const scrollbarWidth = outer.offsetWidth - inner.offsetWidth;
    outer.parentNode?.removeChild(outer);
    return scrollbarWidth;
  }

  // リサイズした場合に更新
  function updateScrollbarWidth() {
    const width = getScrollbarWidth();
    setScrollbarWidth(width);

    const outerDiv = document.getElementById("top_main");
    if (outerDiv) {
      const scrollbarVisible = outerDiv.scrollHeight > outerDiv.clientHeight;
      setScrollbarAppeared(scrollbarVisible);
    }
  }

  updateScrollbarWidth();
  window.addEventListener("resize", updateScrollbarWidth);

  return () => {
    window.removeEventListener("resize", updateScrollbarWidth);
  };
}, []);
```

実装にあたり、2つの`useState`フックを利用しました。

- scrollbarWidth: スクロールバーの幅
- scrollbarAppeared: ページにスクロールバーが出現しているかどうか

#### `scrollbarWidth`

スクロールバーの幅の計算は次の手順で行いました。

1. div要素の作成：`outer`
2. スタイルを編集し、要素が見えない状態にする・スクロールバーが`outer`で常に表示されるようにする
3. bodyへ`outer`を追加
4. `inner`というdiv要素を作成・`outer`の子要素に
5. `outer`と`inner`の`offsetWidth`の差分がスクロールバーの幅になる

#### `scrollbarAppeared`

ページにスクロールバーが出現しているかどうかは次の手順で行いました。

1. スクロール可能要素(メインコンテンツ)を取得：`outerDiv`
2. 取得できたなら、メインコンテンツの`scrollHeight`が`clientHeight`より大きいか求める

ここまで記述したら、後は div 要素に style プロパティを直接記述するだけです。

```tsx:サイドバーのズレを適用
<div style={{ right: `${scrollbarAppeared ? scrollbarWidth : 0}px` }} />
```

![Chromeの場合](/images/next-gas-laboratory/zenn-sai-4.png)
*Chromeの場合(ずらした結果)*

![Firefoxの場合](/images/next-gas-laboratory/zenn-sai-5.png)
*Firefoxの場合(ずらした結果)*

#### 関連プロジェクトの表示

次は右サイドバーのコンテンツ部分です。関連プロジェクトは以下の基準で決定しました。

- タイトルが部分一致するかどうか
- キーワードのいずれかが部分一致するかどうか

これらの内1つでも満たしたものを関連プロジェクトとしました。

```tsx:src/app/project/[slug]/page.tsx(抜粋)
const filteredRelativeProject = projects.filter(
  (item) =>
    project?.tags.some(
      (tag) =>
        item.tags.some(
          (itemTag) =>
            tag.name !== "" &&
            itemTag.name.toLowerCase().includes(tag.name.toLowerCase())
        ) || project?.title.toLowerCase().includes(item.title.toLowerCase())
    ) && project?.id !== item.id
);
```

### メタデータの設定

https://github.com/kcct-rtakada/sai_lab_web/blob/v1.0.0/src/components/common/SEO.tsx

Next.js は Metadata 型を`generateMetadata`関数で返すことで`<head>`内の要素を設定できます。

@[card](https://nextjs.org/docs/app/building-your-application/optimizing/metadata)

`SEO.tsx`を作成し、各ページで作成されるメタデータの展開を担います。

#### 動的生成ページのメタデータ

プロジェクト(ニュース)一覧からヒットした対象のプロジェクト(ニュース)を使って割り当てます。

```tsx:src/app/project/[slug]/page.tsx(抜粋)
export async function generateMetadata({
  params,
}: {
  params: { slug: string };
}): Promise<Metadata> {
  const { project } = await getProject(params.slug);
  if (!project)
    return SEO({
      title: "Undefined",
      description: "No Project",
      url: `https://sai.ac/project/${params.slug}`,
      imageUrl: undefined,
    });
  else
    return SEO({
      title: project.title,
      description: project.abstract,
      url: `https://sai.ac/project/${params.slug}`,
      imageUrl: undefined,
    });
}
```

生成対象外のページは notFound ページに飛ばされるわけでなく、リンクはそのままになるため「Undefined」としてメタデータを設定します。

### サイトマップの作成

サイトマップは Google のクローラで監視してくれるファイルでもあるため、ニュースやプロジェクトの更新も記載したいところです。

Next.js ではサイトマップを TypeScript から生成する場合は、`app`ディレクトリ直下に`sitemap.tsx`を置くとのこと。

@[card](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/sitemap)

https://github.com/kcct-rtakada/sai_lab_web/blob/v1.0.0/src/app/sitemap.tsx

固定ページと動的ページで分けて変数を定義しました。

:::message
サーバーでレンダリングするファイルとして認識させるため、

```tsx:src/app/sitemap.tsx(抜粋)
export const dynamic = "force-dynamic";
export const revalidate = 0;
```

を最初に記述して動的生成可能にしています。

:::

### (補足)OSによって表示テキストを変える

[検索時にエンターキーを使う](#検索時にエンターキーを使う)の`<input />`ではプレースホルダーに「タップして入力」または「クリックして入力」が表示されます。

```tsx:OSがスマホ用か判定する
setIsUsingPhone(/iPhone|iPad|iPod|Android/i.test(navigator.userAgent));
```

`useState`フックのセッターに直接 boolean 型で代入しています。
`navigator`はクライアントならではのインターフェイスなので`useEffect`内で実行します。

![デスクトップ版](/images/next-gas-laboratory/zenn-sai-6.png)
*デスクトップ版*

![スマホ版](/images/next-gas-laboratory/zenn-sai-7.png)
*スマホ版*

### (補足)bodyを参照しない上へ戻るボタンの設置

今回作成するページは等しくスクロールバーがヘッダーから下にしか表示されないようにしました。(画像参照)

![スクロールバーがヘッダーより下にある様子](/images/next-gas-laboratory/zenn-sai-3.png)
*スクロールバーがヘッダーより下にある様子*

トップへ戻るボタンはよく見かけますが、div 内を監視する必要があるので別のアプローチが必要でした。

```tsx:src/app/layout.tsx(抜粋)
const containerRef = useRef(null);
```

`useRef`フックを用いて以下のように割り当てました。

```tsx:src/app/layout.tsx(抜粋)
<body>
  {/* 前略 */}
  <Header />
  <main ref={containerRef} id="top_main">
    {children}
    {/* フッターはスクロール対象 */}
    <Footer />
    <ScrollToTopButton containerRef={containerRef} />
  </main>
</body>
```

注目すべき点は`<main ref={containerRef}>`で、main タグ(ページのスクロール要素)に ref を付与していることです。

```tsx:src/components/common/ScrollToTopButton.tsx(抜粋)
const [isVisible, setIsVisible] = useState(false);

useEffect(() => {
  const handleScroll = () => {
    const scrollTop = containerRef.current.scrollTop;
    // 300px下がるまで表示させない
    if (scrollTop > 300) {
      setIsVisible(true);
    } else {
      setIsVisible(false);
    }
  };

  containerRef.current.addEventListener("scroll", handleScroll);

  return () => {
    // eslint-disable-next-line react-hooks/exhaustive-deps
    containerRef.current.removeEventListener("scroll", handleScroll);
  };
}, [containerRef]);
```

`containerRef`が props で渡された useRef フックであり、`scrollTop`を取得しスクロール量で判定することでトップへ戻るボタンの出現を管理しています。

## 開発(SCSS)

UX に直結する場所です。向上を目的とする上で気が抜けない箇所になります。
今回、スタイルに関しては SCSS を使ったスクラッチ開発を行いました。

視線誘導やボタンやリンクをぱっと見で判別できる点を意識したいです。
今回の製作は基本単位を`rem`として開発しました。

- ヘッダーの高さは 4rem
- はみ出し部分は`overflow: hidden;`で切り取られる

### トップページの画像・アニメーション部分

画面全体(`height: 100vh;`)を覆うのではなく、スクロールできると思わせることが重要です。

```scss:src/styles/app/page.module.scss(抜粋)
.img_box {
  background-color: #e6e6e6;
  width: 100%;
  height: 30rem;
  max-height: calc(100vh - 4rem);
  // 略
}
```

高さが非常に低い時のみ全画面で覆います。

#### 文字アニメーションで全て同じ位置に出現させる

```scss:src/styles/app/page.module.scss(抜粋)
.animation_box {
  position: absolute; // アニメーションのあるボックスを固定配置
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  background-color: #232323;
  z-index: 1500;

  // 子要素のすべてのdivをanimation_boxと同じサイズで同じ場所に配置
  & > div {
    position: absolute; 
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    justify-content: center; // 要素中央に配置
    align-items: center;
    text-align: center;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    // 略
  }
  // 略
}
```

あとは animation-delay を調節するだけで表示が可能です。最後に`display: none;`を遅延実行し、後ろの画像を表示させます。

### ページ下ボタン

言語切り替えボタン・トップへ戻るボタンの2つです。
下側で揃えるのは勿論ですが、どれだけで浮かして揃えるかです。

最初は`1rem`程度にしていましたが、iPhone 11 で PWA として開くとフレームで表示がかなり切れていました。この結果を踏まえ、`2rem`となりました。

### 各メンバーの検索ボタン

スマホ幅でない時、ボタン幅は`width: 2.5rem;`にしています。しかし、スマホ幅の時のみ`width: 3.6rem;`に設定しました。

これは、ホバーするタイプのスクロールバーの時に問題が生じます。

右端はスクロールバーの領域なためスクロール時に表示が被り、スクロールバーの方を触る可能性を感じさせ、触れそうな範囲が縮小します。この対策として幅が狭い場合のボタン面積拡大があります。

### (補足)`position: fixed;` でセクションタイトルを固定する際の問題

今回の実装中、`position: fixed;`された h タグと id によるページ内リンクの相性が非常に悪く、上から下へのジャンプができても下から上へのジャンプができませんでした。この問題の解消についてはこちらを参照してください。

@[card](https://qiita.com/isirmt/items/bfa657c02612541d2d6e)

### 目次を上から表示させる

ヘッダーまでは付いてこず、スクロール時は常に左上に位置する目次メニューを作りました。

```scss:src/styles/components/YearListSidebar.module.scss
.l_sidebar {
  width: 3.5rem;
  height: 3.5rem;
  background-color: $c-y_list_bg;
  margin-top: 1rem;
  top: 1rem;
  left: 1rem;
  position: sticky;
  overflow: visible;
  // 略
}
```

`overflow: visible`と設定したことで飛び出す要素も表示され、拡張した描画が可能になります。今回の設計は左サイドバーをレスポンシブデザインによって変化させるため、設定する必要がありました。

メニューの高さは、ヘッダーとフッターが重ならないように、

```scss:src/styles/components/YearListSidebar.module.scss
&.open {
  height: auto;
  max-height: calc(100vh - 12rem);
}
```

最大の高さから`12rem`減らすことで調整しました。

## リリース・運用

Vercel から GitHub のリポジトリをリンクしました。パッチをあてる時はブランチを作成しプレビュー機能を使うことで、OpenGraph 以外は確認できるようになるので問題ないことを確認した後、マージします。

運用時のキャッシュ更新時間は以下のように設定しました。

- メンバー: 6時間
- プロジェクト: 1時間
- ニュース: 5分
- 表彰: 8時間

更新が既に確認できており、現在は教員を中心に運用中です。

## 注意点

製作する上で、何点か念頭に置いた内容も記述します。

### スプレッドシートの文字数上限

1つのセルあたり、50000 文字までです。簡単に超えられる数字ではありませんが、
`<svg>`を入れる際に気を付けたいところです。

同時に GAS のレスポンス制限も確認しました。[(補足)Cachingの対策](#補足cachingの対策)でも述べたサイトから確認しましたが、恐らく`URL Fetch POST size`に引っかかると推測します。1アクション`50 MB`とあるので、この値が超えない限りは大丈夫だと信じています。

### Google Driveから直接画像を持ってくるか

今回、画像の参照元を研究室で契約しているサーバのディレクトリにしました。Google Drive の共有リンクを使ってどうこうするという手法があるらしいですが、規約等の観点からもやめておきました。

## こだわり(工夫した点)・苦労した点

### こだわり(工夫した点)

CSS(SCSS) については全面的にこだわったつもりです。ボタンのサイズ設計、画面の占有率はユーザビリティに直結し、満足度が変わる場所です。機能も勿論重要視されますが、見て頂くページである以上は丁寧に振舞う必要があると考えました。

HTML 要素の設計においても、目的の機能まで何ステップでアクセス可能かを重視しました。ステップの幅もできるだけ小さくすると同時に、初回アクセスでも目的の機能まで素早くたどり着くにはどんなデザインで配置が良いか迷いました。
このコンテンツを見る人が次にどんなコンテンツを見るかという点を、周囲の人に質問しながら調整を行いました。意見をしてくださった方々に感謝を申し上げます。

### 苦労した点

Caching とサイドバーのずらしが印象に残っています。

記事更新を実装している途中でテスト環境にデプロイしている際、記事が何故か更新されないという状況に陥っていました。
App Router に対して慣れない状態でドキュメントを読んでおり、中々解決しませんでした。中盤は妥協案として Vercel で Data Cache をパージし、リデプロイすることで更新を実施していましたが、更新のコストが非常に高いと感じていました。
最終的に解決しましたが、しなかった場合は Fastly も前向きに検討していました。

スクロールバーは Web ページの中で、スクロール可能だと教えてくれる貴重な存在なので活かしたかったです。

@[card](https://www.fastly.com/)

## 最後に

Next.js を使ったサイト構築は今回が4回目となり、開発スピードも安定してきました。

ページの利用者と管理者双方のUX(User Experience)を重視し、構築するために第三者の意見を常に聞ける体制があることが良いと感じます。ありがとうございました。
