---
title: "【Next.js + GitHub API】シリーズ・コメント有りのブログを作成し運用する"
emoji: "🎲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "githubapi", "recaptcha", "blog"]
published: true
---

技術等に対してちょっとした記事を書き残せるブログサイトを構築したく、作成・リリースしたため、ソースコードと共に実装方法についてまとめました。設計から実装、運用まで書くため長くなります。

## 結論

本記事の結論となるサイト・リポジトリです。

- HP

@[card](https://blog.isirmt.com)

- リポジトリ

@[card](https://github.com/isirmt/NextjsBlogWithGitPAT)

## 目標・設計

### 目標

- PCで記述しやすい Markdown(MD) 形式ブログ
- ファイル配置に外部サービスを利用し、画像も対応可
- 投稿のシリーズ化対応
- コメントフォームの設営
- 低コストなサイト運営

これらを1つのサービスで完結させることが目標です。

### 利用する技術・サービスと理由

- **Vercel**

PaaS です。GitHub アカウントとの連携やプレビュー機能の点でとても利用しやすいためです。

@[card](https://vercel.com)

- **Next.js**

React ベースの WEB アプリが構築可能です。Vercel 管理な点や動的ページ生成が容易な点が主な理由です。

@[card](https://nextjs.org/)

- **GitHub API**

自身の GitHub リポジトリへ発行されたトークンで Public / Private に問わずアクセスが可能になります。既にアクセス管理がある点とバージョン管理が可能なためです。

@[card](https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)

- **reCAPTCHA**

[コメント機能の設計](#コメント機能の設計)にて後述。

@[card](https://www.google.com/recaptcha)

- [**Tailwind CSS**](https://tailwindcss.com/): SCSS を使わず試しに使ってみました。

### 記事の管理方法について

GitHub のプライベートリポジトリを作成しました。

```tree:記事のディレクトリ構造
├── posts/
│   ├── [シリーズ名]/
│   │   ├── (meta.json)
│   │   ├── 1.md
│   │   ├── 2.md
│   │   └── ...
│   └── 記事名.md
├── img/
│   ├── [任意]
│   └── 任意
└── profile/
    └── index.md
```

[目標](#目標)よりシリーズの実現はディレクトリ構造を反映すれば解決だと考えました。ただし、`title`や`description`はあると嬉しいです。`meta.json`を任意で配置可能にします。

プロフィールは同じ MD 形式で統一します。画像配置は`posts`と同じ階層で、記述時にルート`/`から参照することで VS Code 等のプレビュー機能が対応可能な状態にします。

### サイト上の記事管理

同じトピックはタグ付けします。まず、今回の`.md`の上部に入る YAML ヘッダは以下の通りです。

```md:example.md
---
title: "あけましておめでとうございます"
date: "2024/1/1"
tags: ["ブログ","Next.js","Markdown","Tailwind CSS"]
---
ここから本文
```

`title`だけは必須化します。他は`undefined`でも対応できるようにします。今回のタグ管理は全角文字やスペースにも対応させます。でないと対応表を作る必要があったためです。

- **単発の場合**

`/posts/example.md`とした場合、URLは`[host]/post/example`となるようにします。

- **シリーズの場合**

`/posts/making-website/example.md`とした場合、URLは`[host]/post/making-website/example`となるようにします。

ここで、シリーズの識別子は`making-website`となります。

`/posts/making-website/meta.json`がある場合、ファイル内の`name`、`description`に従って表示されます。

```json:/posts/making-website/meta.json
{
  "name": "ページ作ってみた",
  "description": "Next.jsとGitHub APIでサイト構築"
}
```

### コメント機能の設計

コメントの管理はプライベートリポジトリ内の Issue を使うことにしました。同じリポジトリで管理可能な点と Issue 自体の機能でフォームを制御可能なためです。

コメントの投稿にはアカウントが無くても可能にしました。ただし、

- IPアドレス単位の投稿間隔制限
- reCAPTCHAによる認証

を設けることにします。両者ともにスパム対策を兼ねています。

しかしコメント欄の機能が不十分なため、Issue の

- ロック機能で**コメントの投稿許可・不許可**
- オープン/クローズで**コメント欄自体の有効・無効**

を設定可能にします。

### ページのレイアウト構造

ブログサイトを構築するのでレイアウトも少し触れます。

![ページレイアウト](/images/next-github-blog-launch/layout-over.webp)
*ページレイアウト*

画像のように`display: flex`と`flex-direction`を組み合わせました。`fixed`を使わないため、レスポンシブデザインを構築し易いです。

以下より実装に入ります。

## 実装に入る前に

今回の実装は後の`/src/static/constant.ts`と`.env`系統以外に自身を示すコードが入らないようにしました。
ソースは省略しつつ掲載するため、もし不明な場合コードブロックのファイル名を基に参照したいただけると幸いです。

CSS は横サイズ指定とアイテム配置方法のプロパティを示しますが、それ以外は省略いたします。

## 実装: サイトとして

`$ pnpm create next-app`で作成します。

```tree:サイトリポジトリのディレクトリ構造
├── public
├── src/
│   ├── api/
│   │   └── comment-submission/
│   │       └── route.ts
│   ├── app/
│   │   ├── post/
│   │   │   ├── [slug]/
│   │   │   │   └── page.tsx
│   │   │   └── page.tsx
│   │   ├── feed/
│   │   │   └── route.ts
│   │   ├── page.tsx
│   │   ├── layout.tsx
│   │   ├── not-found.tsx
│   │   ├── sitemap.ts
│   │   └── [...]
│   ├── components/
│   │   └── [...]
│   └── styles/
│       ├── global.css
│       └── [...]
├── .env
└── [...]
```

### レイアウトの実装

まずは、`layout.tsx`を整えます。

```tsx:/src/app/layout.tsx(簡略)
export default function RootLayout({ children }) {
  return (
    <html lang="ja">
      <head />
      <body>
        <Header />
        <div className="md:flex justify-center">
          <Menu />
          {children}
        </div>
        <Footer />
      </body>
    </html>
  );
}
```

`<body>`タグの中にヘッダー、フッター、メインコンテンツを置きました。メインコンテンツは[ページのレイアウト構造](#ページのレイアウト構造)の通り`flex`で配置し、各コンポーネントで設計します。左側は共通メニューなので同じ内容が表示されます。

例としてトップページの`page.tsx`を整えます。

```tsx:/src/app/page.tsx
export default async function Blogs() {
  return <Main>
    <Side>
      {/* 右メニュー */}
    </Side>
    <Section>
      <Title>タイトル</Title>
      {/* コンテンツ */}
    </Section>
  </Main>
}
```

`<Main>`、`<Side>`、`<Section>`、`<Title>`は全て共通コンポーネントとして呼び出します。

これにより、各ページでレイアウトが崩れなくなります。宣言は`src`配下の`components`で行います。

```tsx:/src/components/layout/PageLayout.tsx(簡略)
export function Main({ children }: { children?: React.ReactNode }) {
  return <main className="md:flex flex-row-reverse flex-grow md:flex-grow-0">
    {children}
  </main>
}

export function Side({ children }: { children?: React.ReactNode }) {
  return <div className="hidden md:block">
    <div className="w-full sticky">
      {children}
    </div>
  </div>
}

export function Section({ children }: { children?: React.ReactNode }) {
  return <section className="w-full mx-auto xl:m-0">
    {children}
  </section>
}

export function Title({ children }: { children: React.ReactNode }) {
  return <h1 className="text-3xl">
    {children}
  </h1>
}
```

`{ children: React.ReactNode }`に`undefined`を許すかでコンポーネントの調整が可能です。

## GitHub API を利用するために

### トークンの発行

Personal access token(PAT) を発行して fetch ができるようにします。ユーザ側へログインを要求しないので OAuth は利用しません。

[GitHub](https://github.com/) にアクセスし、`メニュー`→`Settings`→`Developer settings`→`Personal access tokens`→`Tokens (classic)`を選択して Generate New Token で repo 権限を与えて発行します。

### 環境変数へ PAT を追加

環境変数を設定します。

React 製 App はルート直下の`.env`に値を書くとソース内で呼び出せます。例えば`.env.local`を追加すれば git に追加されない非公開の環境変数を設定可能です。このファイルはパブリックリポジトリで非常に重要です。

```bash:.env.local
GIT_TOKEN=(YOUR TOKEN HERE)
```

他にもユーザ名、リポジトリ名、投稿フォルダ名等も環境変数で宣言します。

```bash:.env.local(例)
GIT_USERNAME=(YOUR USER NAME HERE)
GIT_REPO=(YOUR REPOSITORY NAME HERE)
GIT_POSTS_DIR=posts
GIT_IMAGES_DIR=img
GIT_PROFILE_PATH=profile/index.md
```

[記事の管理方法について](#記事の管理方法について)のディレクトリ構成に揃えました。

環境変数を TypeScript で利用するとき、

```ts
// string | undefined
const name = process.env.GIT_USERNAME
```

では、型が`string | undefined`になります。そこで、次のように書くと型を一意に設定可能です。

```ts
// string
const name = process.env.GIT_USERNAME!
```

:::message
もし、`pnpm dev`と`pnpm build && pnpm start`で参照するフォルダを変える場合は

```bash:.env.development.local
GIT_POSTS_DIR=dev_posts
```

```bash:.env.production.local
GIT_POSTS_DIR=posts
```

と宣言をすることで本番環境に影響なく開発が可能です。
:::

## 実装: 記事の取得・表示

### API による呼び出し

API のURLは以下の通りです。

```ts
const gitContentPath = `https://api.github.com/repos/${process.env.GIT_USERNAME!}/${process.env.GIT_REPO!}/contents`
```

`gitContentPath`を宣言しました。次にトークンを fetch 時のヘッダーに載せると API によるアクセスとなり、プライベートリポジトリの参照ができます。

今回複数の方法で`api.github.com`にアクセスするので`init?`用に複数の関数を作成しました。

- **ヘッダー情報**

```ts:/src/lib/fetchingFunc.ts(抜粋)
export function getHeaders() {
  return {
    headers: {
      "Authorization": `token ${process.env.GIT_TOKEN!}`,
      "Content-Type": "application/json",
    }
  };
};
```

`Authorization`はトークンの文字列だけでなく、`token "YOUR TOKEN HERE"`の形が必要です。

- **Next.js Caching対策**

Next.js では特定の URL へ fetch した際、結果がキャッシュされます。デフォルトでは永続的にキャッシュされ、解消する方法は「dev中の`Ctrl + F5`やVercelの`Clear Caching Data`」と望ましくない状況になります。
そこで、`init?`に`next`を含ませることでキャッシュ時間の秒指定が可能になります。本記事ではこれを多用します。

```ts:/src/lib/fetchingFunc.ts(抜粋)
export type FetchOptions = RequestInit & {
  next?: { revalidate: number };
};

export function getNext(revalidate: number): FetchOptions {
  if (revalidate === 0) {
    return { cache: 'no-store' };
  } else {
    return { next: { revalidate } };
  }
}
```

`getNext`関数で`revalidate: number`を受け取り、0 以外はキャッシュ時間を設定します。0 の場合は最新の状態が必要だと判断し、`cache: 'no-store'`を返すようにします。そのため、`FetchOptions`の定義が必要になりました。

### GitHub APIのリスト取得について

記事リストを取得しましょう。まず、取得だけを考えると次のようになります。

上2つの関数も使うと以下のようになります。

```ts
const data = await fetch(`${gitContentPath}/${process.env.GIT_POSTS_DIR!}`, {
    ...getHeaders(), ...getNext(3600)
  }).then(res => res.json()).catch(err => console.error(err));
```

`getNext(3600)`より、3600 秒 = 1 時間のキャッシュ期間を設けました。
`data`には配列でファイル名やディレクトリ名、URL等が入っています。

ここで、とある問題が発生します。**リストは30件までしか取得できない**点です。パラメータを変更しても最大100件なので、全部取得できない可能性があります。修正方法はレスポンスのヘッダ内にある`Link`から取得します。

そのため以下のコードを書きました。

```ts:/src/lib/fetchingFunc.ts(抜粋)
export async function fetchAllData(url: string, revalidate: number): Promise<any[]> {
  let results: any[] = [];
  let nextUrl: string | null = url;

  while (nextUrl) {
    const response: Response = await fetch(nextUrl, {
      ...getHeaders(), ...getNext(revalidate),
    });

    const data = await response.json();
    results = results.concat(data);

    const linkHeader = response.headers.get('Link');
    if (linkHeader) {
      const nextLinkMatch = linkHeader.match(/<([^>]+)>;\s*rel="next"/);
      nextUrl = nextLinkMatch ? nextLinkMatch[1] : null;
    } else {
      nextUrl = null;
    }
  }

  return results;
}
```

リストを得る際は次の情報を取得することにします。

- `slug`: ページ ID
- `excerpt`: 記事の抜粋(100文字程度)
- `data`: YAML ヘッダ(タイトル・日付・タグ)

ページIDは[サイト上の記事管理](#サイト上の記事管理)に従い、`/posts/making-website/example.md`がリポジトリにある場合、`slug = "making-website/example"`と管理します。

```ts:/src/static/postType.ts
export type PostData = {
  title: string;
  tags?: string[];
  date?: string;
  series?: string;
  [key: string]: any;
};

export type Post = {
  slug: string;
  excerpt: string;
  data: PostData;
};
```

本実装では、シリーズを設けるのでフォルダが混入しています。`data[i].type === 'dir'`の場合、そのディレクトリへ fetch する必要があります。`dir?: string`を引数にシリーズのみの取得も対応させるため、関数は以下のようになります。

```ts:/src/lib/getPosts.ts(抜粋)
async function createPostFromFile(item: any, dir: string): Promise<Post | null> {
  const { data, excerpt } = await getPostContent(`${dir}/${item.name}`);
  if (data.title) {
    return {
      slug: item.path.replace(`${process.env.GIT_POSTS_DIR}/`, "").replace('.md', ''),
      data,
      excerpt,
    };
  }
  return null;
}

async function createPostsFromDirectory(item: any): Promise<Post[]> {
  const dirPath = `${process.env.GIT_POSTS_DIR}/${item.name}`;
  const dirContent = await fetchAllData(`${gitContentPath}/${dirPath}`, 3600);

  const markdownFiles = dirContent.filter((subItem) => subItem.type === "file" && subItem.name.endsWith('.md'));
  const dirFiles = await Promise.all(markdownFiles.map(subItem => createPostFromFile(subItem, dirPath)));

  return dirFiles.filter((post): post is Post => post !== null);
}

export const getPostsProps = cache(async (dir?: string): Promise<Post[]> => {
  const targetDir = dir ? `${process.env.GIT_POSTS_DIR}/${dir}` : process.env.GIT_POSTS_DIR!;
  const data = await fetchAllData(`${gitContentPath}/${targetDir}`, 3600);

  const posts: Post[] = [];

  for (const item of data) {
    if (item.type === "file" && item.name.endsWith('.md')) {
      const post = await createPostFromFile(item, targetDir);
      if (post) {
        posts.push(post);
      }
    } else if (!dir && item.type === "dir") {
      const dirPosts = await createPostsFromDirectory(item);
      posts.push(...dirPosts);
    }
  }

  return posts.sort(comparePosts);
});
```

`Post[]`配列に追加する時に`data.title`が無い場合は弾きます。レンダーで事故が発生しないようにします。

`getPostContent`関数も記します。ファイルを直接指定すると`content`に base64 エンコーディングの中身があるのでデコーディングします。

```ts:/src/lib/getPosts.ts(抜粋)
const getPostContent = cache(async (path: string): Promise<{ data: PostData; content: string; excerpt: string }> => {
  const fileJson = await fetch(`${gitContentPath}/${path}`, {
    ...getHeaders(), ...getNext(3600)
  }).then(res => res.json()).catch(err => console.error(err))

  if (fileJson?.message === 'Not Found' || fileJson?.status === 404) {
    notFound();
  }

  // デコーディング
  const buf = Buffer.from(fileJson.content, 'base64');
  const fileContent = buf.toString("utf-8");
  const { data, content } = matter(fileContent);

  // サブディレクトリがある場合はシリーズに名前を追加
  const pathParts = path.split('/');
  const outputData = pathParts.length > 2 ? { ...data, series: pathParts[1] } : data;

  // 抜粋を作成
  const plainText = await MarkdownToPlainText(content);
  const excerpt = makeExcerpt(plainText, 128);

  return {
    data: outputData as PostData,
    content,
    excerpt,
  }
})
```

ここで、`notFound`関数を用意しておくと予期しないページ ID の場合に 404 ステータスを返すことができます。

:::message

`notFound`関数は/src/app/で存在する場合も、`fetch`で 404 ステータスの際に利用することで`/_not-found`へ転送することが可能です。

```ts:notFound 関数の利用
import { notFound } from 'next/navigation';
```

:::

### 記事の表示

取得ができたので表示します。

- **記事一覧の表示**

`/src/app/post/page.tsx`にて投稿一覧を作成します。
[レイアウトの実装](#レイアウトの実装)で作ったファイルをインポートして統一したレイアウトも作成します。

```tsx:/src/app/post/page.tsx(簡略)
export default async function PostList() {
  const posts = await getPostsProps();

  return <Main>
    <Side />
    <Section>
      <Title>投稿一覧</Title>
      <div className='flex flex-col'>
        {posts.map((post, i) => <PostCard post={post} key={i} />)}
      </div>
    </Section>
  </Main>
}
```

![/post の表示](/images/next-github-blog-launch/posts-over.png)
*/post の表示*

メインの記述がスッキリ書けました。`<PostCard />`を定義することで他のコンポーネントでも再利用できるようにします。

```tsx:/src/components/post/PostCard.tsx(簡略)
export default function PostCard({ post }: { post: Post }) {
  return <div>
    <div><Link href={`/post/${post.slug}`}>
      <div className="flex items-center">
        <DateCard date={post.data.date} />
        <div>
          <p>
            {post.data.series ? <span>シリーズ</span> : <></>}
            <span>{post.data.title}</span>
          </p>
          <p>{makeExcerpt(post.excerpt, 64)}</p>
        </div>
      </div></Link>
    </div>
    {post.data.tags || post.data.series ?
      <div className='flex flex-wrap'>
        {post.data.series ? <Link href={`/series/${post.data.series}`}>
          <div>
            <span>シリーズを表示</span>
          </div>
        </Link> : <></>}
        {post.data.tags?.map((tag, i) => <TagBanner tag={tag} key={i} />)}
      </div> : <></>}
  </div>
}
```

- **記事の表示**

ブログとして記事を表示します。`/src/app/post/[...slug]/page.tsx`にて投稿一覧を作成します。
`[ ]`がディレクトリ名にあるため Dynamic Routes が有効になります。

公式ドキュメントにある通り、`[ ]`の書き方次第で TypeScript における型が異なります。`slug`で既に`/`が入るため多重構造のディレクトリであり、指定方法は`[...slug]`となります。

`[[...slug]]`は`slug?: string[]`と`undefined`も対象となります。この場合、記事一覧のページも拾われるので利用しません。

@[card](https://nextjs.org/docs/app/building-your-application/routing/dynamic-routes#typescript)

```tsx:/src/app/post/[...slug]/page.tsx(簡略)
const getFileContent = cache(async (path: string) => {
  const decodedSlug = decodeURIComponent(path);
  const postPath = `${process.env.GIT_POSTS_DIR!}/${decodedSlug}.md`
  return await getPost(postPath);
})

export default async function Post({ params }: { params: { slug: string[] } }) {
  const slug = params.slug.join('/')
  const { data, content } = await getFileContent(slug);
  const issue = await getCommentList(slug);

  return <Main>
    <SideMDShown>
      <PostIndex content={content} title={data.title} />
      <div>
        <div>共有</div>
        <ShareButtons path={`/post/${slug}`} text={data.title} />
      </div>
    </SideMDShown>
    <Article data={data} content={content} issue={issue} slug={slug} />
  </Main>
}
```

`getFileContent`関数は Metadata の作成時に再利用します。記事の本編は`<Article />`でコンポーネント化しました。`page.tsx`をスッキリさせる目的と`/profile`で同じレイアウトを出力したいためです。

```tsx:/src/components/layout/ArticlePage.tsx
export default async function Article({ data, content, issue, slug }: { data: PostData, content: string, issue?: Issue, slug?: string }) {
  const series = data.series ? await getSeries(data.series) : undefined;

  return <article className="w-full">
    <div className="flex items-center">
      <DateCard date={data.date} />
      <h1>{data.title}</h1>
    </div>
    {data.tags ?
      <div className="flex flex-wrap">
        {data.tags?.map((tag, i) =>
          <TagBanner tag={tag} key={i} />)}
      </div> :
      <></>}
    {series ?
      <div>
        <SeriesCard
          slug={data.series}
          index={series.posts.findIndex((item) => item.slug === slug)}
        />
      </div> : <></>}
    <PostMarkdown content={content} />
    {/* ここにコメント欄 */}
  </article>
}
```

![/post/[...slug]の表示](/images/next-github-blog-launch/posts-slug-over.png)
*/post/[...slug]の表示*

`getPost`関数で返される`data.series`を引数にして`getSeries`関数でシリーズを取得しました。

このように`<PostCard />`や`<Article />`のように`data.*`で参照することが確定しているパラメータが多いため[GitHub APIのリスト取得について](#github-apiのリスト取得について)の`/src/static/postType.ts`では一部変数で型を宣言し、`[key: string]: any`で未定義にも対応させました。

`issue`は次章で利用します。

### 記事内の画像表示

`<PostMarkdown content={content} />`で MD 形式で書かれた文字列を変換しています。`<ReactMarkdown />`を今回は使います。

@[card](https://www.npmjs.com/package/react-markdown)

```tsx
<ReactMarkdown
  disallowedElements={["h1"]}
  components={components}
  remarkPlugins={[remarkMath, remarkGfm]}
  rehypePlugins={[rehypeRaw]}>
  {content}
</ReactMarkdown>
```

`h1`だけは表示の都合上困るので弾きます。`components`はHTMLタグで任意のものに変換された場合、設定した関数コンポーネントを呼び出します。

```tsx:/src/components/post/MarkdownElements.tsx
function getMimeType(path: string) {
  const ext = path.split('.').pop()?.toLowerCase();
  switch (ext) {
    case 'jpg':
    case 'jpeg':
      return 'image/jpeg';
    case 'png':
      return 'image/png';
    case 'gif':
      return 'image/gif';
    case 'webp':
      return 'image/webp';
    case 'bmp':
      return 'image/bmp';
    default:
      return 'application/octet-stream';
  }
}

async function ExImg({ path, alt }: { path: string, alt?: string }) {
  const image64 = await getImage(path);
  const mimeType = getMimeType(path);
  return <img alt={alt} src={`data:${mimeType};base64,${image64}`} />
}

// h2, h3は省略

const Img = ({ node, ...props }:
  ClassAttributes<HTMLImageElement> &
  HTMLAttributes<HTMLImageElement> &
  ExtraProps & { src?: string, alt?: string }) => {
  const src = props.src as string || '';
  const alt = props.alt as string || '';
  if (src.startsWith(`/${process.env.GIT_IMAGES_DIR!}/`)) {
    return <ExImg path={src} alt={alt} />
  } else
    return (
      <img {...props}>{props.children}</img>
    );
};

const Pre = ({ children, ...props }:
  ClassAttributes<HTMLPreElement> &
  HTMLAttributes<HTMLPreElement> &
  ExtraProps) => {
  if (!children || typeof children !== 'object') {
    return <code {...props}>{children}</code>
  }
  const childType = 'type' in children ? children.type : ''
  if (childType !== 'code') {
    return <code {...props}>{children}</code>
  }

  const childProps = 'props' in children ? children.props : {}
  const { className, children: code } = childProps
  const classList = className ? className.split(':') : []
  const language = classList[0]?.replace('language-', '')
  const fileName = classList[1]

  return (
    <div className="post_codeblock w-full">
      {fileName && (
        <div className="post_fname">
          <span>{fileName}</span>
        </div>
      )}
      <SyntaxHighlighter language={language} style={atomOneDark}>
        {String(code).replace(/\n$/, '')}
      </SyntaxHighlighter>
      <CopyToClipboard text={String(code).replace(/\n$/, '')} />
    </div>
  )
}

const A = ({ href, ...props }:
  ClassAttributes<HTMLAnchorElement> &
  HTMLAttributes<HTMLAnchorElement> &
  { href?: string }) => {
  const isInternalLink = href?.startsWith("/");
  return isInternalLink ? (
    <a href={href} {...props}>{props.children}</a>
  ) : (
    <a href={href} target="_blank" rel="noopener noreferrer" {...props}>{props.children}</a>
  );
};

export const components: Partial<Components> = {
  pre: Pre, h2: H2, h3: H3, img: Img, a: A
}
```

`components`にて JSON 形式で対応関係を作っています。

`<a>`タグは`/`から始まる場合は内部リンクと判定しそのまま転送、それ以外の場合は新規タブで開くために`target="_blank" rel="noopener noreferrer"`を挟みました。

`<img>`タグは`/posts(任意の名前)/`から始らないパスの場合、そのままの形で出力します。それ以外の場合、GitHub から得る必要があると判断したため、fetch を行います。しかし画像が 1MB を超えた際は記事と同じ方法で取得ができないため、同時に得られる`git_url`でもう一度 fetch します。base64 エンコーディングで得られ、識別用のタグを共に src へ挿入することで表示が可能です。`ExImg`関数にて複数の画像拡張子に対応しておきました。`getImage`関数を示します。

```ts:/src/lib/getPosts.ts(抜粋)
export const getImage = cache(async (path: string) => {
  const fileJson = await fetch(`${gitContentPath}${path}`, {
    ...getHeaders(), ...getNext(3600 * 24 * 30)
  }).then(res => res.json()).catch(err => console.error(err));

  // もう一度git_urlで取得
  const imageJson = await fetch(fileJson.git_url, {
    ...getHeaders(), ...getNext(3600 * 24)
  }).then(res => res.json()).catch(err => console.error(err));
  return imageJson.content as string;
})
```

## 実装: コメントフォーム

![コメントフォームの全体像](/images/next-github-blog-launch/posts-comments.png)
*コメントフォームの全体像*

### コメントの取得

MD ファイル置かれているリポジトリの Issue を使ってコメント欄を作ります。[記事と同じ要領](#github-apiのリスト取得について)で取得します。

Issue と コメントで必要な型を決めます。

1. **Comment**: コメント

- **date**: 投稿された日付
- **content**: 内容(MD 形式)

2. **Issue**: Issue

- **comments**: コメント配列
- **locked**: Issue がロックされているか
- **state**: Issue が`Open`か`Closed`か

```ts:/src/static/issueType.ts
export type Comment = {
  date: string,
  content: string
}

export type Issue = {
  comments: Comment[],
  locked: boolean,
  state: "open" | "closed"
}
```

`getCommentList`関数を作ります。Issue の探し方はタイトル名の完全一致です。タイトルを`slug`とするので、シンプルな検索が可能です。

```ts:/src/lib/commentIssueManager.ts(抜粋)
export const getCommentList = cache(async (slug: string): Promise<Issue> => {
  const targetIssue = await getIssue(slug);

  if (targetIssue) {
    const data = await fetch(targetIssue.commentsURL, {
      ...getHeaders(), ...getNext(20),
    })
      .then((res) => res.json())
      .catch((e) => console.error(e));

    const comments: Comment[] = [];
    for (const item of data) {
      const date = new Date(item.created_at as string);
      comments.push({
        date: date.toLocaleString("ja-JP"),
        content: item.body as string,
      });
    }
    return {
      comments,
      locked: targetIssue.locked,
      state: targetIssue.state
    };
  } else {
    createIssue(slug);
    return {
      comments: [],
      locked: false,
      state: "open"
    }
  }
});
```

- **getIssue** 関数

`fetchAllData`関数とタイトル名検索(`${gitIssuePath}?q=${encodeURIComponent(slug)}+in:title&state=all`)を組み合わせます。

```ts:/src/lib/commentIssueManager.ts(抜粋)
const gitIssuePath = `https://api.github.com/repos/${process.env.GIT_USERNAME!}/${process.env.GIT_REPO!}/issues`;

const getFilteredIssuePath = (slug: string) =>
  `${gitIssuePath}?q=${encodeURIComponent(slug)}+in:title&state=all`;

async function getIssue(slug: string, revalidate: number = 120) {
  const data = await fetchAllData(getFilteredIssuePath(slug), revalidate);

  if (data && data.length > 0) {
    const issue = data.find((item: any) => item.title === slug);
    if (issue) {
      return {
        slug: issue.title as string,
        commentsURL: issue.comments_url as string,
        locked: issue.locked as boolean,
        state: issue.state as "open" | "closed"
      };
    }
  }
  return null;
}
```

- **createIssue** 関数

`issueCreationMap`というマップ変数を作成しました。キーを`string`にし、値を`Promise<void>`にすると、関数の記述を代入と同時に行えます。終了後にキーを削除することで、Issue が削除された後に再度作成することが可能です。

この手法の理由は、`getIssue`で null だった場合に`createIssue`が呼ばれますがアクセス等の問題で同時実行され、同名 Issue が発生するためです。そのため、キー値の有無で同時実行が発生しないようにします。

```ts
const issueCreationMap: Record<string, Promise<void> | undefined> = {};

const createIssue = cache(async (slug: string) => {
  if (!issueCreationMap[slug]) {
    issueCreationMap[slug] = (async () => {
      const targetIssue = await getIssue(slug, 0);
      if (targetIssue) return;
      const data = {
        title: slug,
        body: `${process.env.NEXT_PUBLIC_URL!}/post/${slug}`,
        labels: ["user-comment"],
      };

      await fetch(gitIssuePath, {
        method: "POST",
        body: JSON.stringify(data),
        ...getHeaders(),
      })
        .then((res) => {
          if (res.status !== 201) {
            console.error(`Error creating issue: ${res.status} - ${res.statusText}`);
          } else {
            console.log(`Issue created successfully for slug: ${slug}`);
          }
        })
        .catch((e) => console.error(e));
    })();

    issueCreationMap[slug].finally(() => {
      delete issueCreationMap[slug];
    });

    return issueCreationMap[slug];
  } else {
    return issueCreationMap[slug];
  }
});
```

### reCAPTCHA の導入

コメントフォームを表示する前に reCAPTCHA を導入します。

@[card](https://www.google.com/recaptcha/about/)

ここでは、サイトキーとシークレットキーの発行手順は割愛します。

```bash:導入方法
pnpm add react-google-recaptcha-v3
```

ここで、`.env.local`に環境変数を追加します。

```bash:.env.local(抜粋)
NEXT_PUBLIC_RECAPTCHA_SITE_KEY=(YOUR KEY HERE)
RECAPTCHA_SECRET_KEY=(YOUR KEY HERE)
```

### コメントの表示

- **コメント一覧の表示とreCAPTCHAの配置**

クライアントコンポーネントに`<GoogleReCaptchaProvider />`を配置します。

```tsx:/src/components/post/CommentForm.tsx(簡略)
"use client";
import { GoogleReCaptchaProvider } from "react-google-recaptcha-v3";

function CommentsView({ comments }: { comments: Comment[] }) {
  return <div className="flex flex-col">
    {comments.length > 0 ? comments.map((comment, i) => (
      <div key={i} className="block items-start lg:flex">
        <DateCard date={comment.date} />
        <div className="flex-grow min-w-0">
          <CommentMarkdown content={comment.content} />
        </div>
      </div>
    )) :
      <ExplainingBanner>
        コメントはまだありません
      </ExplainingBanner>}
  </div>
}

export function CommentForm({ comments, slug }: { comments: Comment[], slug: string }) {
  return <GoogleReCaptchaProvider
    reCaptchaKey={process.env.NEXT_PUBLIC_RECAPTCHA_SITE_KEY!}
    language="ja">
    <section>
      <h2>コメント</h2>
      <CommentsView comments={comments} />
      <PostingForm slug={slug} />
    </section>
  </GoogleReCaptchaProvider>
}
```

`NEXT_PUBLIC_RECAPTCHA_SITE_KEY`のように、`NEXT_PUBLIC_`から始まる環境変数はクライアントサイドで使えるようにインライン化されます。もし無い場合は undefined となり、reCAPTCHA が起動せず右下にバナーは表示されません。

- **フォームの作成**

こちらもクライアントコンポーネントで作成します。先に全部載せた後、説明します。

```tsx:/src/components/post/CommentPostingBox.tsx(簡略)
'use client';
import { useGoogleReCaptcha } from "react-google-recaptcha-v3";

export default function PostingForm({ slug }: { slug: string }) {
  const [inputtedValue, setInputtedValue] = useState<string>("");
  const [isInputting, setIsInputting] = useState<boolean>(true);
  const [isSubmitting, setIsSubmitting] = useState<boolean>(false);
  const [notificationBanner, setNotificationBanner] = useState<React.ReactNode>()
  const { executeRecaptcha } = useGoogleReCaptcha();

  const handleSubmit = async () => {
    setNotificationBanner(null);
    if (inputtedValue.trim() === "") {
      setNotificationBanner(GenerateNotificationBanner("コメントの入力が必須です", false));
      return;
    }
    if (!executeRecaptcha) {
      setNotificationBanner(GenerateNotificationBanner("ReCaptcha を実行できませんでした", false));
      return;
    }
    const token = await executeRecaptcha('submitComment');
    setIsSubmitting(true);

    try {
      const response = await fetch('/api/comment-submission', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ slug, comment: inputtedValue, token }),
      });
      if (response.ok) {
        setNotificationBanner(GenerateNotificationBanner("コメントが送信されました", true));
        setInputtedValue("");
      } else {
        const data = await response.json();
        setNotificationBanner(GenerateNotificationBanner(`コメントの送信に失敗しました: ${data.error}`, false));
      }
    } catch (e) {
      setNotificationBanner(GenerateNotificationBanner(`コメントの送信に失敗しました: ${e}`, false));
    } finally {
      setIsSubmitting(false);
    }
  }

  return (
    <div>
      <div className="flex">
        <button onClick={() => setIsInputting(true)}>コメント入力</button>
        <button onClick={() => setIsInputting(false)}>プレビュー</button>
      </div>
      {isInputting ?
        <textarea
          onChange={(e: React.ChangeEvent<HTMLTextAreaElement>) => setInputtedValue(e.target.value)}
          value={inputtedValue}
          placeholder="Markdown形式で入力可" /> :
        <div>
          {inputtedValue.trim() !== "" ? <CommentMarkdown content={inputtedValue} /> :
            <ExplainingBanner>
              「コメント入力」で入力後、プレビューをお試しください
            </ExplainingBanner>}
        </div>
      }
      <div className="items-center flex flex-row-reverse">
        <button onClick={handleSubmit} disabled={isSubmitting}>
          {isSubmitting ? "送信中..." : "送信"}
        </button>
        <div className="flex-1">
          This site is protected by reCAPTCHA and the Google <a href="https://policies.google.com/privacy">Privacy Policy</a> and <a href="https://policies.google.com/terms">Terms of Service</a> apply.
        </div>
      </div>
      {notificationBanner ? notificationBanner : <></>}
    </div>
  )
}
```

プレビュー機能も実装しました。`setInputtedValue`関数が`<textarea>`の onChange で動作し、inputtedValue が変化します。`inputtedValue.trim() !== ""`の時のみプレビュー表示するので空白は表示されません。

送信ボタンを押すと`handleSubmit`関数が動作します。
予め、空文字の場合は弾きます。そして`executeRecaptcha`が実体を持つか確認したのち関数を実行します。引数の文字列は、恐らくイベントとして区別が可能なら問題ないと思います。

API Routeにある`/api/comment-submission`へ POST リクエストを送信します。リンク先で GitHub API を利用して Issue へコメントを投げます。body には、記事のID、コメント本文、`executeRecaptcha`関数で得られたトークンを含めます。

そして`response.ok`を得られた場合、成功のステータスを表示します。リクエストを終えたタイミングで inputtedValue を空にします。

API に認証・送信処理を任せるため、コンポーネントの記述量は少なめです。

:::message alert

コンポーネント読み込み時に`executeRecaptcha`を取得しますが、コンポーネントが`<GoogleReCaptchaProvider />`内で呼ばれることが前提です。満たさない場合、

```ts
const token = await executeRecaptcha('submitComment');
```

でクライアント側のエラーが返されます。

```tsx:動作する場合のイメージ
return <GoogleReCaptchaProvider>
  {/* executeRecaptchaを取得するコンポーネント */}
  <ComponentUsingExecuteRecaptcha />
</GoogleReCaptchaProvider>
```

:::

### 認証付きコメント送信 API の作成

`/src/app/api/comment-submission/route.ts`を作成します。App Routerになり、関数名を method で書かないといけなくなりました。

```ts:/src/app/api/comment-submission/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { LRUCache } from 'lru-cache';

const rateLimit = new LRUCache<string, number>({
  max: 1000,
  ttl: 1000 * 60 * 60,
});

const getHeaders = () => {
  return {
    "Authorization": `token ${process.env.GIT_TOKEN!}`,
    "Content-Type": "application/json",
  };
};

const authorize = async (token: string) => {
  const secretKey = `secret=${process.env.RECAPTCHA_SECRET_KEY!}&response=${token}`;
  const data = await fetch("https://www.google.com/recaptcha/api/siteverify", {
    method: "POST",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: secretKey,
  }).then(res => res.json());

  return data;
}

export async function POST(req: NextRequest) {
  const { slug, comment, token } = await req.json();
  const ip = req.ip || req.headers.get('x-forwarded-for') || 'unknown';

  if (!slug || !comment || !token) {
    return NextResponse.json({ error: 'Missing slug or comment body' }, { status: 400 });
  }

  const requestCount = rateLimit.get(ip) || 0;
  if (requestCount > 10) {
    return NextResponse.json({ error: 'Too many requests' }, { status: 429 });
  }
  rateLimit.set(ip, requestCount + 1);

  const recaptchaJson = await authorize(token);

  if (!recaptchaJson.success) {
    return NextResponse.json({ error: "ReCAPTCHA verification failed" }, { status: 401 })
  }

  try {
    const issueResponse = await fetch(`https://api.github.com/repos/${process.env.GIT_USERNAME!}/${process.env.GIT_REPO!}/issues`, {
      headers: getHeaders(),
    });
    const issues = await issueResponse.json();
    const issue = issues.find((issue: any) => issue.title === slug);

    if (!issue) {
      return NextResponse.json({ error: `Issue with slug "${slug}" not found` }, { status: 404 });
    }

    // コメントの追加
    const commentsURL = issue.comments_url;
    const data = {
      body: comment,
    };

    const response = await fetch(commentsURL, {
      method: 'POST',
      headers: getHeaders(),
      body: JSON.stringify(data),
    });

    if (response.ok) {
      return NextResponse.json({ message: 'Comment added successfully' }, { status: 200 });
    } else {
      const errorText = await response.text();
      return NextResponse.json({ error: `Failed to add comment: ${errorText}` }, { status: response.status });
    }

  } catch (e) {
    console.error('Error adding comment:', e);
    return NextResponse.json({ error: 'Internal Server Error' }, { status: 500 });
  }
}
```

`req.json()`で body からデータを得ます。

```ts
const { slug, comment, token } = await req.json();
```

[コメント機能の設計](#コメント機能の設計)にある通り、IP アドレス単位の投稿間隔制限を設けるので、`req`より取得します。
また、この時点で body に書かれるべきデータが足りない場合は 400 エラーを返します。

IP アドレスのリクエスト回数を問い合わせます。ここで`lru-cache`を使い、1 時間に 10 回までの投稿リクエスト権を最大 1000 人に付与します。上限オーバーの場合は 429 エラーを返します。上限以内ならカウントを +1 して処理を続行します。

```ts
const requestCount = rateLimit.get(ip) || 0;
if (requestCount > 10) {
  return NextResponse.json({ error: 'Too many requests' }, { status: 429 });
}
rateLimit.set(ip, requestCount + 1);
```

トークンから reCAPTCHA による認証結果を問い合わせます。`RECAPTCHA_SECRET_KEY`は API Route で呼ぶため、クライアントサイドで表示されません。`data.success = true`の場合は処理を続行し、`false`の場合は不正な送信なため 401 エラーを返します。

```ts
const authorize = async (token: string) => {
  const secretKey = `secret=${process.env.RECAPTCHA_SECRET_KEY!}&response=${token}`;
  const data = await fetch("https://www.google.com/recaptcha/api/siteverify", {
    method: "POST",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: secretKey,
  }).then(res => res.json());

  return data;
}
```

対象の Issue を見つけたら、そのページへ POST リクエストを送信します。ok が無事返されたら成功です。

問題が発生した場合に対応できるようスタータスの分岐を用意しました。

### コメントへのアクセス管理

[コメント機能の設計](#コメント機能の設計)にもあった、アクセス権の設定です。

注意点としてコメント欄を封鎖することは自由ですが、**クライアントサイドに送信/閲覧機能を与えない**ことが重要です。したがって、`<Article />`のようなサーバーサイドで提供コンポーネントを切り替えます。

- ロック機能で**コメントの投稿許可・不許可**
→ reCAPTCHA と投稿フォームのコンポーネントが無いコメント欄コンポーネントを呼ぶ

- オープン/クローズで**コメント欄自体の有効・無効**
→ サーバーサイドでコメント欄コンポーネント自体を呼ばない

`/src/components/layout/ArticlePage.tsx`の`{/* ここにコメント欄 */}`へ以下を追加します。

```diff tsx:/src/components/layout/ArticlePage.tsx
export default async function Article({ data, content, issue, slug }: { data: PostData, content: string, issue?: Issue, slug?: string }) {
  const series = data.series ? await getSeries(data.series) : undefined;

  return <article className="w-full">
    <div className="flex items-center">
      <DateCard date={data.date} />
      <h1>{data.title}</h1>
    </div>
    {data.tags ?
      <div className="flex flex-wrap">
        {data.tags?.map((tag, i) =>
          <TagBanner tag={tag} key={i} />)}
      </div> :
      <></>}
    {series ?
      <div>
        <SeriesCard
          slug={data.series}
          index={series.posts.findIndex((item) => item.slug === slug)}
        />
      </div> : <></>}
    <PostMarkdown content={content} />
-   {/* ここにコメント欄 */}
+   {issue && slug ?
+     issue.state === "closed" ?
+       <ExplainingBanner>
+         コメントは無効です
+       </ExplainingBanner> :
+       issue.locked ?
+         <CommentFormNoPosting comments={issue.comments} /> :
+         <CommentForm comments={issue.comments} slug={slug} /> :
+     <></>}
  </article>
}
```

`closed`の場合を最優先として条件分岐を作りました。

![コメント欄が無効化された状態(closed)](/images/next-github-blog-launch/posts-comments_closed.png)
*コメント欄が無効化された状態(closed)*

![投稿フォームが無効化された状態(locked)](/images/next-github-blog-launch/posts-comments_locked.png)
*投稿フォームが無効化された状態(locked)*

## 実装: ブログとして

ブログなのでサイトマップと RSS を導入します。また、動的に Metadata を生成します。

### Metadata の実装

`/src/lib/SEO.ts`に共通で付与するプロパティを関数で定義します。

```ts:/src/lib/SEO.ts
interface Props {
  title?: string,
  description?: string,
  url: string,
  imageURL?: string,
  keywords?: string[],
  type?: OpenGraphType,
}

export function generateMetadataTemplate(props: Props): Metadata {
  const { title, description, url, keywords, type } = props;
  const outputTitle = title
    ? `${title} - ${siteName}`
    : siteName;
  const outputDescription = description
    ? description
    : siteDescription;
  const outputType: OpenGraphType = type ? type : "website";

  const metadata: Metadata = {
    metadataBase: new URL(process.env.NEXT_PUBLIC_URL!),
    authors: { name: author.name, url: author.url },
    title: outputTitle,
    description: outputDescription,
    icons: "/favicon.ico",
    keywords,
    openGraph: {
      title: title ? title : siteName,
      description: outputDescription,
      url: url,
      siteName,
      type: outputType,
    },
  };
  return metadata;
}
```

`metadataBase`は相対パスを補うために利用されます。

> metadataBase は、完全修飾 URL を必要とするメタデータフィールドのベース URL プレフィックスを設定する便利なオプションです。 metadataBase を使うと、現在のルートセグメント以下で定義されている URL ベースのメタデータフィールドで、そうでなければ必須の絶対 URL の代わりに相対パスを使うことができます。 フィールドの相対パスは metadataBase と組み合わされて、完全修飾 URL になります。 (機械翻訳)

@[card](https://nextjs.org/docs/app/api-reference/functions/generate-metadata#metadatabase)

各 page.tsx で次のように呼び出します。`/src/app/post/[...slug]/page.tsx`の場合は以下の通りです。

```ts:/src/app/post/[...slug]/page.tsx(抜粋)
export async function generateMetadata({ params }: { params: { slug: string[] } }): Promise<Metadata> {
  const slug = decodeURIComponent(params.slug.join('/'));
  const { data, excerpt } = await getFileContent(slug);

  return generateMetadataTemplate({
    title: `${data.title}`,
    description: `${excerpt}`,
    url: `/post/${params.slug.join('/')}`,
    type: "article",
  });
}
```

### RSS の実装

`/feed`でアクセスが可能な RSS を作成しました。Google によると RSS もサイトマップ送信に利用可能とのことです。

@[card](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap?hl=ja#rss)

```ts:/src/app/feed/route.ts(抜粋)
import Rss from 'rss';

export const dynamic = "force-dynamic";
export const revalidate = 1200;

const baseURL = process.env.NEXT_PUBLIC_URL!;

export async function GET() {
  const feed = new Rss({
    title: `${siteName}の新着投稿`,
    description: `「${siteName}」の投稿フィード`,
    feed_url: `${baseURL}/feed`,
    site_url: baseURL,
    language: 'ja'
  });

  const posts = await getPostsProps();

  posts.forEach((post) => feed.item({
    title: post.data.title,
    description: post.excerpt,
    url: `${baseURL}/post/${encodeURIComponent(post.slug)}`,
    date: post.data.date ? new Date(post.data.date).toISOString() : new Date(lastModified).toISOString()
  }))

  return new Response(feed.xml(), {
    headers: {
      'Content-Type': 'application/xml',
      'Cache-Control': `s-maxage=${revalidate}, stale-while-revalidate`
    }
  });
}
```

このように、投稿を全てアイテムとして追加しました。

### Sitemap の実装

```ts:/src/app/sitemap.ts
export const dynamic = "force-dynamic";
export const revalidate = 0;

const staticPaths = [
  "/post",
  "/profile",
  "/series",
  "/tags"
]

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await getPostsProps();
  const tagsWithDate = getTagsWithLatestDate(posts);

  const baseURL = process.env.NEXT_PUBLIC_URL!;

  const staticPages: MetadataRoute.Sitemap = [
    {
      url: baseURL,
      priority: 1.0,
    }
  ];

  staticPaths.forEach((page) => {
    staticPages.push({
      url: baseURL + page,
      priority: 0.9
    })
  })

  const dynamicPages: MetadataRoute.Sitemap = [];

  tagsWithDate.forEach((tagWithDate) => {
    dynamicPages.push({
      url: baseURL + "/tags/" + encodeURIComponent(tagWithDate.tag),
      lastModified: tagWithDate.latestDate ? new Date(tagWithDate.latestDate) : lastModified,
      changeFrequency: "monthly",
      priority: 0.8
    })
  })

  return [...staticPages, ...dynamicPages];
}
```

`staticPaths`の配列を回すことで固定ページの追加を行います。`/tags/[slug]`だけ動的生成されるページとして追加しました。その際にタグの中で最新の投稿の日時を取得します。

> 一部のサイトのソフトウェアは、サイト上の他のページを集約しているにすぎないため、ホームページやカテゴリページの最終更新日を判断するのが難しい場合があります。そのような場合は、そのページの lastmod を除外してもかまいません。

@[card](https://developers.google.com/search/blog/2023/06/sitemaps-lastmod-ping?hl=ja#the-lastmod-element)

とあるので、`staticPages`に lastMod は付与しないことにしました。

## Vercel へのデプロイ

デプロイ前に環境変数を一度見直すことをお勧めします。例えば`.env.production`の`NEXT_PUBLIC_URL`へ自身の割り当てる予定のドメインが入っていることを確認します。そして、`pnpm build`を実行してローカルでビルドし、成功することも確認しましょう。

Vercel へデプロイすると同時に環境変数を決定します。

@[card](https://vercel.com)

GitHub にあげた自身のプロジェクト選択時に Environment Value を登録可能なフィールドがあります。以下の環境変数を追加しましょう。

```bash:.env*.local
GIT_USERNAME=(YOUR GITHUB USERNAME HERE)
GIT_REPO=(YOUR GITHUB REPOSITORY NAME HERE)
GIT_POSTS_DIR=posts
GIT_IMAGES_DIR=img
GIT_PROFILE_PATH=profile/index.md
GIT_TOKEN=(YOUR PAT HERE)
NEXT_PUBLIC_RECAPTCHA_SITE_KEY=(YOUR TOKEN HERE)
RECAPTCHA_SECRET_KEY=(YOUR TOKEN HERE)
```

`NEXT_PUBLIC_RECAPTCHA_SITE_KEY`は`NEXT_PUBLIC_`と`KEY`が両方書かれているけど公開されますよ？というメッセージが出るかもしれません。

あとはデプロイするだけです。

## おわりに

CSS は今回の主題ではないため、割愛いたします。作成したブログでシリーズとしてまとめたので是非参照ください。

@[card](https://blog.isirmt.com/series/nextjs-gh-blog-css)

何度か Next.js でサイトを作りました。動的ページの要素をどのサーバーやサービスに置けばいいかと試行錯誤している現状です。GitHub API の呼び出し上限は 5000 回 / hour との記述があるため、revalidate を 0 にしておくと、多重アクセスで直ぐに利用できなくなります。キャッシュ時間を調節しつつ適切な更新頻度が実現したいですね。ありがとうございました。
