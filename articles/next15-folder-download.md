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

[NextAuth](https://next-auth.js.org/)を利用します。v15でも特に事故は発生せず導入可能でした。

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

## コンテンツの取得

取得したアクセストークンでリポジトリ内のリストを取得します。GitHub APIとのやり取りにはOctokitを使います。

> JavaScript を使用して GitHub の REST API と対話するスクリプトを記述する場合、GitHub では、Octokit.js SDK を使用することをお勧めします。 Octokit.js は GitHub によって管理されます。 SDK によってベスト プラクティスが実装されており、JavaScript を使用して REST API を簡単に操作できます。Octokit.js は、最新のあらゆるブラウザー、Node.js、Deno で動作します。 Octokit.js について詳しくは、Octokit.js の README を参照してください。
> *(引用: <https://docs.github.com/ja/rest/guides/scripting-with-the-rest-api-and-javascript#about-octokitjs>)*

fetch APIでも可能ですが、typesが定義されていることもあり非常におすすめです。

次にSessionからアクセストークンを取得し、octokitのインスタンスを作成するまでを記します。

```tsx
import { Session } from 'next-auth';

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

これだけでリストとして出力が可能です。型情報の宣言については、

```ts
type GitHubRepoContent = GetResponseTypeFromEndpointMethod<NonNullable<typeof octokit>['repos']['getContent']>['data']
```

これで指定が可能になります。
