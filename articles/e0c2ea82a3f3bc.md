---
title: "react-queryをサーバーサイドで働かさせて、データ取得の最適化をする"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "react-query", "rsc", "nextjs"]
published: true
---

## はじめに

以下は、Next.js を使った例となります。

## セットアップ

```ts: app/query-client.ts
function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        refetchOnWindowFocus: false,
        staleTime: 60 * 1000, //💡 サーバーサイドで毎回データ取得したくないので、0以上のstaleTimeを指定しましょう
      },
    },
  });
}

let browserQueryClient: QueryClient | undefined = undefined;

function getQueryClient() {
  if (isServer) {
    return makeQueryClient();
  } else {
    if (!browserQueryClient) browserQueryClient = makeQueryClient();
    return browserQueryClient;
  }
}

export default getQueryClient;
```

## Provider コンポーネント

```ts:app/provider.tsx
"use client";

import Devtools from "@/src/components/ui/dev-tools";
import "@fontsource-variable/noto-sans-jp/";
import { QueryClientProvider } from "@tanstack/react-query";
import { ReactNode } from "react";
import { getQueryClient } from "./query-client";

export default function Providers({ children }: { children: ReactNode }) {
  const queryClient = getQueryClient(); //💡 インスタンスを取得します

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <Devtools />
    </QueryClientProvider>
  );
}

```

そのあと、layout コンポーネントなどに provider で children を囲むイメージ

```ts:app/layout.tsx
import Providers from "./provider";

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
      <html lang='en'>
        <body>
          <Providers>{children}</Providers>
        </body>
      </html>
  );
}
```

## サーバーサイドでデータ取得セットアップ

サイトの情報を取得する react-query を内包したカスタムフックがあるとします。
![alt text](/images/image.png)

```ts:app/_hooks/useSite.ts
import { getSiteInfo } from "@firebase/admin/getSiteInfo";
import { useQuery } from "@tanstack/react-query";

export const SITE_INFO_KEY = "siteInfo";

export default function useSite() {
  const res = useQuery({
    queryKey: [SITE_INFO_KEY],
    queryFn: getSiteInfo,
    meta: {
      errorMessage: "サイト情報の取得に失敗しました",
    },
  });

  return res;
}

```

サーバーサイドで react-query を働かさせます。

```ts: (cms)/layout.tsx
import AppBar from "@/src/components/component/page-app-bar";
import { getSiteInfo } from "@/src/services/firebase/admin/getSiteInfo";
import {
  dehydrate,
  HydrationBoundary,
  QueryClient,
} from "@tanstack/react-query";
import { Provider } from "./provider";
import { SITE_INFO_KEY } from "../_hooks/useSite";

export default async function CMSLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  const queryClient = new QueryClient();

  await queryClient.prefetchQuery({ //prefetchQueryを使います
    queryKey: [SITE_INFO_KEY],//💡 カスタムフックと同値のqueryキーを指定します
    queryFn: getSiteInfo, //💡 同様のquery関数も指定
  });

  return (
    <>
      <HydrationBoundary state={dehydrate(queryClient)}>
        <div className='overflow-auto pb-16'>{children}</div>
      </HydrationBoundary>
      <AppBar />
    </>
  );
}

```

## 結果

### ASIS

本来では、以下のように、コンポーネントが hydrate されてから、クライアントでデータ取得を行うようにしています。
![alt text](/images/image5.png)
サーバーから返ってきた bundle にも含まれません
![alt text](/images/image-3.png)

### TOBE

一方、ネットワークレスポンスを見ると、改めて取得したデータがコンポーネントファイルの bundle と一緒に返ってきていることがわかります。
![alt text](/images/image-1.png)
useSite のフックが行うクライアントサイド fetch もしません。
![alt text](/images/image4.png)

あと圧倒的にデータの描画も早くなりました 👏
ぜひ試してみてください。
