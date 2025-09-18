---
title: "はじめての React Router v7 - layout で実現する超簡素な認可制御"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true # trueを指定する
published_at: 2025-09-19 08:00 # 未来の日時を指定する
---

# はじめに

React Router v7 で以下を実装してみました.

- Cookie に含まれたトークンの検証によるユーザー認証
- 認証したユーザーが各 route にアクセスできるかの認可制御

React Router v7 の仕様の詳細説明や思想の解説は行いません.

# 成果物

https://github.com/virtual-hippo/hello-react-router/tree/v0.0.0

# 実装のポイント

- 認可制御用の layout を用意する
- 認可が必要な route は認可制御用の layout 配下に route を設定する

## 認可制御用の layout を用意する

「レイアウト」という言葉からは, UI を規定するものというイメージを持ちがちですが, React Router v7 では `loader` や `action` などの [Route Module API](https://reactrouter.com/start/framework/route-module) も「レイアウト」のスコープに入ります.

以下のコンポーネントでは, `loader()` において, リクエストヘッダに含まれた Cookie から token を取得し検証しています.

※トークン検証失敗時に, `Error` ではなくて `redirect` を throw しているのも React Router っぽいです.

```tsx
// app/routes/_layouts/auth.tsx

export async function loader({ request }: Route.LoaderArgs) {
  const cookieHeader = request.headers.get("Cookie");
  const cookie = (await idTokenCookie.parse(cookieHeader)) || {};

  // トークンが存在しない場合はログイン画面へリダイレクト
  if (!cookie["id-token"]) {
    throw redirect("/login");
  }

  try {
    const user = verifyIdToken(cookie["id-token"]);
    return { userName: user?.name ?? "Unknown" };
  } catch {
    // トークンの検証に失敗した場合はログイン画面へリダイレクト
    // 単に有効期限の場合は refresh トークンを使って再発行する処理を入れたいが今回は実装をサボる
    cookie["id-token"] = "";
    throw redirect("/login", {
      headers: {
        "Set-Cookie": await idTokenCookie.serialize(cookie, { maxAge: 0 }),
      },
    });
  }
}

export default function Layout({ loaderData }: Route.ComponentProps) {
  return (
    <>
      <Outlet />
    </>
  );
}
```

:::message
認可制御については [middleware](https://reactrouter.com/start/framework/route-module#middleware) も利用できそうです (というかこちらも利用すべきかもしれません).

しかし, 検証できてないです.

また, middleware はフレームワークモードでは安定板としてリリースされていなさそうです.

https://reactrouter.com/how-to/middleware
:::

## 認可が必要な route は認可制御用の layout 配下に route を設定する

`/dashboard`, `/dashboard/settings` の route を 認可制御用の layout 配下にネストさせます.

この構造により

- `/dashboard` へのアクセス → auth layout の loader が実行される
- `/dashboard/settings` へのアクセス → 同様に auth layout の loader が実行される
- `/login` へのアクセス → auth layout を通らないため認証不要

token 検証に失敗した場合はログイン画面にリダイレクトされます.

```ts
// app/routes.ts
import {
  type RouteConfig,
  index,
  layout,
  route,
} from "@react-router/dev/routes";

export default [
  layout("routes/_layouts/index.tsx", [
    index("routes/home.tsx"),
    route("login", "routes/login.tsx"),

    layout("routes/_layouts/auth.tsx", { id: "auth" }, [
      route("dashboard", "routes/dashboard/_layout.tsx", [
        index("routes/dashboard/index.tsx"),
        route("settings", "routes/dashboard/settings.tsx"),
      ]),
    ]),
  ]),
] satisfies RouteConfig;
```

# 参考資料

### りあクト！ TypeScript で始めるつらくない React 開発【③ React 実践編】

わたしが React に触れる際にはお世話になっている本です (第 5 版と第 4 版を持っています).

認証/認可制御の実装例を探せなかったので, 試しに自分で実装してみました.

https://oukayuka.booth.pm/items/2367992

### React Router Home

公式ドキュメントです.

https://reactrouter.com/home
