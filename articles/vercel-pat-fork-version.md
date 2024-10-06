---
title: "【Vercel】フォークされたリポジトリでフォーク元のバージョンを表示する"
emoji: "🔢"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "vercel", "githubapi"]
published: true
---

## 実装内容

VercelとGitHubを使って
「Build with 〇〇 ver.XXX」の「XXX」を作る。

## 概要

リポジトリをフォークして自分用に改良し、main(master)とは異なるブランチを使ってデプロイしたとします。
※ 新しいブランチを`visual`とします。

mainブランチはフォーク元のパッチを受け付けるようにしておき、取り入れた際は「`visual`←`main`」でマージして自身の要素を残しつつ最新情報を取り入れます。

そして、`visual`によるデプロイメントでフォーク元`main`ブランチのバージョン情報を載せます。

これによって各々のデプロイメントでフォーク元のシステムのバージョン〇〇が適用されている、と分かるようになります。

本記事はその取得・表示を行います。

## 環境

- **Next.js**: 14.2.10
- **Vercel**

自身のGitHubアカウントでPersonal Access Tokenをご準備ください。

## 実装

### 必要な環境変数

まず、Vercelにあるデプロイ単位の環境変数より

- デプロイしてるリポジトリの所有者：`VERCEL_GIT_REPO_OWNER`
- デプロイしてるリポジトリ名：`VERCEL_GIT_REPO_SLUG`

を得られます。Vercelからはコミットに関する情報が環境変数として提供されているので活用していきましょう。

@[card](https://vercel.com/docs/projects/environment-variables/system-environment-variables)

### どのように実現するか

リポジトリには`main`と`visual`があり、Vercelからは`visual`の情報しか得ることができません。そこで、GitHub APIでそのリポジトリの`main`の最新コミットを取得します。想定内であれば同じコミットがフォーク元にもある筈なので、後はその情報を基にリンクを作成するだけです。

### 取得部分

```ts:最新コミットがフォーク元のコミットに含まれるか判定する
const ownerRepoUser = process.env.VERCEL_GIT_REPO_OWNER;
const ownerRepoName = process.env.VERCEL_GIT_REPO_SLUG;
const ownerRepoMainSha =
  ownerRepoUser && ownerRepoName
    ? (
        await fetch(`https://api.github.com/repos/${ownerRepoUser}/${ownerRepoName}/commits?per_page=1`, {
          headers: {
            Authorization: `token ${process.env.GIT_TOKEN!}`,
          },
        }).then((res) => (res.ok ? res.json() : undefined))
      )?.[0]?.sha
    : undefined;
const baseRepoCommitLink = `https://api.github.com/repos/BASE_USER_NAME/BASE_REPO_NAME/commits/${ownerRepoMainSha}`;
const baseRepoCommitIsPresenting = await fetch(baseRepoCommitLink, {
  headers: {
    Authorization: `token ${process.env.GIT_TOKEN!}`,
  },
}).then((res) => res.ok);
```

`ownerRepoMainSha`はユーザー名とレポジトリ名を無事取得できた場合のみfetchします。また、GitHub APIによるコミット履歴取得はクエリパラメータとして表示数(`per_page`)を指定しないと`SHA-1ハッシュ値=""`で検索することになるようです。今回は1つ取得するだけで十分です。
そして、取得できたならSHA-1ハッシュ値を取り出して、フォーク元のリポジトリで同様に検索をします。クエリパラメータは不要です。

### 表示部分

```tsx
{baseRepoCommitIsPresenting ? (
  <Link
    target='_blank'
    rel='noopener noreferrer'
    href={`https://github.com/BASE_USER_NAME/BASE_REPO_NAME/tree/${ownerRepoMainSha}`}
  >
    {String(ownerRepoMainSha).slice(0, 7)}
  </Link>
) : (
  <>Unknown</>
)}
```

`baseRepoCommitIsPresenting`(フォーク元でコミットが見つかったか)が`true`の場合はtreeのリンクを作り、それ以外は`Unknown`を出して完成です。表示テキストは本来長いものなのでよく見る7文字表記で問題ないと思います。
