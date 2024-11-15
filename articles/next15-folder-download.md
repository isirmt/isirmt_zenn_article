---
title: "Next.js 15でGitHubのフォルダ ダウンローダーを作った"
emoji: "📁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "githubapi", "oauth"]
published: false
---

GitHubからOAuth認証を通じて任意のリポジトリ内のフォルダをダウンロードするウェブアプリをNext.js 15を通じて制作した記事です。

## はじめに

GitHubでリポジトリ内のフォルダだけをローカルに取り込みたいってことが偶にありますよね！手法はいくつかありますが、Codespacesを使った際にフォルダをコンテキストメニューからダウンロードしようとすると以下のアラートが...

![alt text](/images/next15-folder-download/cannot-open.png)

これを出さずにダウンロードすることが難しいですよね...

ダウンロードする際にAPIキーがあればコンソール上からもダウンロード可能ですが、発行する手間が大変...という動機で今回、「SourceSnap」という名称でウェブアプリを作成しました。

- OAuth認証によりプライベートリポジトリにも対応
- URLのコピペでクイックアクセス

等に意識しております。

## 該当サイト

@[card](https://ssnap.isirmt.com/)

https://github.com/isirmt/SourceSnap

## 選定した技術

- Next.js: ついにv15へメジャーアップデート。未知数ではあるものの体験として使用。
- TailwindCSS: 最近お気に入りのスタイリングフレームワーク。`className`に直接書く以上、コンポーネントとして後から独立させる処置にも向いている。プレーンからも移行しやすく開発コストが低くなると思い導入。
- Vercel: Next.js開発元のサービス。いつの間にかProductionのビルドにもミニツールバーが表示されるようになり、複数人開発における表示エラーがある際も対応しやすくなっている。

## 認証部分

[NextAuth](https://next-auth.js.org/)の**v5**を利用します。v15でも特に事故は発生せず導入可能でした。

```bash
npm i next-auth@beta
```

### アクセストークンとユーザー名の取得

NextAuthで提供されるSessionのデフォルトオブジェクト内にユーザー名(login)やOAuthのアクセストークンは含まれていません。

これらをTypeScriptでエラーなくオブジェクトへ追加することを目指します。

```ts:/src/lib/auth.ts
import NextAuth, { Session } from 'next-auth';
import { AdapterUser } from 'next-auth/adapters';
import { JWT } from 'next-auth/jwt';
import GitHubProvider from 'next-auth/providers/github';

declare module 'next-auth' {
  // eslint-disable-next-line no-unused-vars
  interface Session {
    access_token?: string;
    user: AdapterUser;
  }
}

export const { handlers, auth, signIn, signOut } = NextAuth({
  trustHost: true,
  providers: [
    GitHubProvider({
      authorization: { params: { scope: 'repo read:org' } },
    }),
  ],
  callbacks: {
    // eslint-disable-next-line no-unused-vars
    async jwt({ token, user, account, profile }) {
      if (profile && account) {
        token.username = profile.login;
        token.access_token = account.access_token;
      }
      return token;
    },
    async session({ session, token }: { session: Session; token: JWT }) {
      session.user.id = token.username as string;
      session.access_token = token.access_token as string;
      return session;
    },
  },
});
```

まずは、

```ts
declare module 'next-auth' {
  interface Session {
    access_token?: string;
    user: AdapterUser;
  }
}
```

Sessionという型定義に`access_token`と`user`が追加で入るように宣言します。これでeslint環境下も問題なく記述できます。

```ts
providers: [
  GitHubProvider({
    authorization: { params: { scope: 'repo read:org' } },
  }),
],
```

authorizationオプションには、リポジトリと組織への読み取り権限でアクセスできるようにします。これでprivateも権限があれば取得が可能になります。

```ts
callbacks: {
  async jwt({ token, user, account, profile }) {
    if (profile && account) {
      token.username = profile.login;
      token.access_token = account.access_token;
    }
    return token;
  },
  async session({ session, token }: { session: Session; token: JWT }) {
    session.user.id = token.username as string;
    session.access_token = token.access_token as string;
    return session;
  },
},
```

ここで、在るべき場所からパラメータをtokenへ一度渡し、sessionに記録します。

```ts:/src/app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/lib/auth';

export const { GET, POST } = handlers;
```

## コンテンツの取得

取得したアクセストークンでリポジトリ内のリストを取得します。GitHub APIとのやり取りにはOctokitを使います。

> JavaScript を使用して GitHub の REST API と対話するスクリプトを記述する場合、GitHub では、Octokit.js SDK を使用することをお勧めします。 Octokit.js は GitHub によって管理されます。 SDK によってベスト プラクティスが実装されており、JavaScript を使用して REST API を簡単に操作できます。Octokit.js は、最新のあらゆるブラウザー、Node.js、Deno で動作します。 Octokit.js について詳しくは、Octokit.js の README を参照してください。
> *(引用: <https://docs.github.com/ja/rest/guides/scripting-with-the-rest-api-and-javascript#about-octokitjs>)*

fetch APIでも可能ですが、typesが定義されていることもあり非常におすすめです。

@[card](https://www.npmjs.com/package/@octokit/rest)

Sessionからアクセストークンを取得し、octokitのインスタンスを作成するまでを記します。

```tsx
import { Session } from 'next-auth';
import { Octokit } from '@octokit/rest';

export default async function Page() {
  const session: Session | null = await auth();
  if (!session || !session.access_token) return <>failed to auth</>;

  const octokit =  new Octokit({ auth: session.access_token });

  return <>check console tab</>;
}
```

Octokitは`auth`オプションなしでもインスタンス作成可能ですが、そのまま利用すると同一IPに対する60回/hの制限に引っかかるので注意。

@[card](https://docs.github.com/ja/rest/using-the-rest-api/rate-limits-for-the-rest-api)

次に取得するまでを書きます。

```ts
const response = await octokit.repos.getContent({
  owner, repo, path, ref,
});

console.log(response.data);
```

これだけでリストとして出力が可能です。型情報の宣言については`@octokit/types`パッケージを導入した後、

```ts
import { Octokit } from '@octokit/rest';
import { GetResponseTypeFromEndpointMethod } from '@octokit/types';

type GitHubTreeContent = GetResponseTypeFromEndpointMethod<Octokit['repos']['getContent']>['data']
```

これで指定が可能になります。ここの指定方法はメソッドと同じ名前で型を呼び出せます。

### コンテンツの表示

表示は、配列か否かを条件分岐してから`React.ReactNode`を返すことで書きやすくなります。

```tsx
// リスト形式で返ってくる場合は基本同じ
const data = response.data;

return (
  <section>
    {Array.isArray(data) && (
      data.map((item) => (
        <div key={item.id}>{item.name}</div>
      ))
    )}
  </section>
);
```

## ダウンロード

ダウンロードを行う過程でCORSとぶつかってしまうため、APIエンドポイントを作成して回避しつつ認証を維持します。

### ファイルの場合

`content.data`の要素内には次のようなものが定義されています。

```ts
type GitHubReposContent = {
  type: 'dir' | 'file' | 'submodule' | 'symlink';
  size: number;
  name: string;
  path: string;
  content?: string;
  sha: string;
  url: string;
  git_url: string | null;
  html_url: string | null;
  download_url: string | null;
  _links: {
    git: string | null;
    html: string | null;
    self: string;
  };
};
```

したがって、`download_url`を渡すことで要件は満たせます。まずは、API側から

```ts:/src/app/api/get-file/route.ts
import { Octokit } from '@octokit/rest';
import { NextResponse } from 'next/server';
import { Session } from 'next-auth';
import { auth } from '@/lib/auth';

export async function GET(request: Request) {
  const session: Session | null = await auth();

  if (!session) {
    return NextResponse.json({ error: 'Not authenticated' }, { status: 401 });
  }
  const { searchParams } = new URL(request.url);
  const downloadUrl = searchParams.get('download_url');

  if (!downloadUrl) {
    return NextResponse.json({ error: 'Missing required parameters' }, { status: 400 });
  }

  try {
    const octokit = new Octokit({
      auth: session.access_token,
    });

    const response = await octokit.request('GET ' + downloadUrl);

    if (response.status !== 200) {
      throw new Error(`Failed to fetch the file: ${response.status}`);
    }

    const buffer = Buffer.from(response.data);

    return new NextResponse(buffer, {
      headers: {
        'Content-Type': 'application/octet-stream',
        'Content-Disposition': `attachment; filename="downloaded_file"`,
      },
    });
  } catch (error) {
    console.error('Error fetching the file:', error);
    return NextResponse.json({ error: 'Error Occurred' }, { status: 500 });
  }
}
```

`try...catch`までが認証の有無とURLが渡されたかの確認、それ以降が`octokit`によるcontentの取得とバッファの作成です。
ここで、NextAuthのv5要素が現れており、`req`からセッション確認するのではなく、`await auth()`とサーバーサイド共通の記述な部分です。

@[card](https://authjs.dev/getting-started/migrating-to-v5#authentication-methods)

そして、クライアント側でファイルとして保存させます。保存には、`file-saver`を使います。

```bash
npm i file-saver
npm i --save @types/file-saver
```

```ts
import saveAs from 'file-saver';

try {
  if (download_url) {
    const response = await fetch(`/api/get-file?download_url=${encodeURIComponent(download_url)}`);
    if (!response.ok) {
      throw new Error('Failed to download file');
    }
    const blob = await response.blob();
    saveAs(blob, name);
  } else {
    throw new Error('File does not have a download URL.');
  }
} catch (error) {
  console.error('Failed to download file:', error);
}
```

### フォルダの場合

#### フォルダを再帰的に取得する

タイトルの通りコンテンツの配列を取得し、typeがdirの場合は再帰的に配列へ追加していく手法です。先述の発展版なので操作はシンプルです。

zip圧縮まで行い、パッケージは`npm i jszip`で導入します。

```ts:/src/app/api/get-folder/route.ts
import { Octokit } from '@octokit/rest';
import JSZip from 'jszip';
import { NextResponse } from 'next/server';
import { Session } from 'next-auth';
import { auth } from '@/lib/auth';
// 配列部分の型の抽出
import { GitHubReposContext } from '@/types/GitHubReposContext';

export async function GET(request: Request) {
  const session: Session | null = await auth();

  if (!session) {
    return NextResponse.json({ error: 'Not authenticated' }, { status: 401 });
  }

  const { searchParams } = new URL(request.url);
  const owner = searchParams.get('owner');
  const repo = searchParams.get('repo');
  const path = searchParams.get('path');
  const ref = searchParams.get('ref');

  if (!owner || !repo || !path) {
    return NextResponse.json({ error: 'Missing required parameters' }, { status: 400 });
  }

  try {
    const octokit = new Octokit({ auth: session.access_token });

    const files = await getFolderContents(octokit, owner, repo, path, ref);

    const zip = new JSZip();

    for (const file of files) {
      const fileContent = await octokit.request('GET ' + file.download_url);

      zip.file(file.path, fileContent.data);
    }

    const zipBlob = await zip.generateAsync({ type: 'blob' });

    return new NextResponse(zipBlob, {
      headers: {
        'Content-Type': 'application/zip',
        'Content-Disposition': `attachment; filename="${repo}.zip"`,
      },
    });
  } catch (error) {
    console.error('Error fetching the file:', error);
    return NextResponse.json({ error: 'Error Occurred' }, { status: 500 });
  }
}

async function getFolderContents(
  octokit: Octokit,
  owner: string,
  repo: string,
  path: string,
  ref: string,
): Promise<GitHubReposContext[]> {
  const contents = await octokit.repos.getContent({
    owner, repo, path, ref,
  });

  if (contents && Array.isArray(contents.data)) {
    const files: GitHubReposContext[] = [];
    for (const item of contents.data) {
      if (item.type === 'file') {
        files.push(item);
      } else if (item.type === 'dir') {
        const subfolderFiles = await getFolderContents(octokit, owner, repo, item.path, ref);
        files.push(...subfolderFiles);
      }
    }

    return files;
  } else return [];
}
```
