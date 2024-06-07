---
title: microCMS で作成した記事に Gatsby で目次を付ける
date: 2021-06-14
slug: microcms-toc
tags:
  - Gatsby
---

microCMS + Gatsby なブログに目次を付けました。  
前に[シンタックスハイライトを行なった](https://qiita.com/tsuchinoko0402/items/632f095b7ae7f1c8b117)ように、HTML パーサーである [cheerio](https://www.npmjs.com/package/cheerio) を使用しました。  
また、スタイルの適用にあたっては、 [styled components](https://styled-components.com/) を用いました。  

## 準備：styled components のインストール

-   これまで、スタイルの適用には、外部のCSS module を使って、 TypeScript のファイルと CSS ファイルを分けて管理して適用していましたが、目次にスタイルを適用するにあたって上手く行かなかったので、styled components を使用しました。

### styled components の特徴

-   JavaScript でスタイルを記述する CSS in JS のライブラリ
    -   コンポーネントとstyleのマッピングが無くなる
    -   ローカルスコープができる
-   書き方の例としては以下のようになります：

```
const Button = styled.a`
  /* This renders the buttons above... Edit me! */
  display: inline-block;
  border-radius: 3px;
  padding: 0.5rem 0;
  margin: 0.5rem 1rem;
  width: 11rem;
  background: transparent;
  color: white;
  border: 2px solid white;

  /* The GitHub button is a primary button
   * edit this to target it specifically! */
  ${props => props.primary && css`
    background: white;
    color: black;
  `}
`

render(
  <div>
    <Button
      href="https://github.com/styled-components/styled-components"
      target="_blank"
      rel="noopener"
      primary
    >
      GitHub
    </Button>

    <Button as={Link} href="/docs">
      Documentation
    </Button>
  </div>
)
```

### Gatsby での styled components の利用

-   Gatsby で styled components を使用するために、以下のようにプラグインをインストールします：

```
$ yarn add gatsby-plugin-styled-components styled-components babel-plugin-styled-components
```

-   `gatsby-config.js` に以下のように記述します：

```
module.exports = {
  plugins: [`gatsby-plugin-styled-components`],
}
```

-   再ビルドすると使用できるようになります。

## 目次の作成方法

-   [microCMS のブログ記事](https://blog.microcms.io/create-table-of-contents/)を参考に HTML パーサーである [cheerio](https://www.npmjs.com/package/cheerio) を使用します。
-   具体的には…
    -   microCMS の API 経由で取得した記事（リッチエディタで作成）の HTML から `h1`, `h2`, `h3` の要素を抜き出します。
    -   スタイルを適用するために、抜き出した要素にクラス名を付与します。
    -   インデントをするために Styled Components を用います。

### 要素の抜き出し

-   以下のようにして要素を抜き出します
    -   `htmlString` には、 microCMS の API 経由で取得した記事の内容（HTML）をそのまま入れます
    -   配列形式で取得できるので、整形してテキスト、ID、タグ名の配列に変換します。

```
const $ = cheerio.load(htmlString)
  const toc = $("h1, h2, h3")
    .toArray()
    .map(data => ({
      text: data.children[0].data,
      id: data.attribs.id,
      name: data.name,
    }))
```

-   以下のような形で取得できます：

```
[
  { text: '目次の作り方', id: 'hkY7o3DlYB2', name: 'h1' },
  { text: '目次機能のオンオフ', id: 'h4AL9B4qAV6', name: 'h2' },
  { text: 'おわりに', id: 'hBwwL6rm8B7', name: 'h1' }
]
```

### 表示方法

-   以下のように要素を返して目次を表示します。
    -   スタイルの適用を見越して、リストの項目一つ一つに `list h1`, `list h2`, `list h3` といったクラス名を付与しています。

```
return (
    <>
      {toc.length ? (
        <div id="create-table-of-contents">
          <h4>目次</h4>
            <ul id="lists">
              {toc.map((toc, index) => {
                return (
                  <li className={"list " + toc.name} key={toc.id}>
                    <a href={"#" + toc.id}>{toc.text}</a>
                  </li>
                )
              })}
            </ul>
        </div>
      ) : (
        ""
      )}
    </>
  )
```

### 字下げを行う

-   h2, h3 の要素に対しては字下げを行うよう、スタイルを適用します。

```
import styled from "styled-components"

const StyledToc = styled.div`
  & .list.h2 {
    margin-left: 1em;
  }

  & .list.h3 {
    margin-left: 2em;
  }
`
```

-   最終的に、目次のコンポーネントは以下のように作成できました：

```
import React from "react"
import cheerio from "cheerio"
import styled from "styled-components"

export type TocTypes = {
  text: string
  id: string
  name: string
}

interface Props {
  htmlString: string
}

const StyledToc = styled.div`
  & .list.h2 {
    margin-left: 1em;
  }

  & .list.h3 {
    margin-left: 2em;
  }
`

const Toc: React.FC<Props> = props => {
  const { htmlString } = props
  const $ = cheerio.load(htmlString)
  const toc = $("h1, h2, h3")
    .toArray()
    .map(data => ({
      text: data.children[0].data,
      id: data.attribs.id,
      name: data.name,
    }))

  return (
    <>
      {toc.length ? (
        <div id="create-table-of-contents">
          <h4>目次</h4>
          <StyledToc>
            <ul id="lists">
              {toc.map((toc, index) => {
                return (
                  <li className={"list " + toc.name} key={toc.id}>
                    <a href={"#" + toc.id}>{toc.text}</a>
                  </li>
                )
              })}
            </ul>
          </StyledToc>
        </div>
      ) : (
        ""
      )}
    </>
  )
}

export default Toc
```

## 参考資料

-   [microCMSで目次を作成する](https://blog.microcms.io/create-table-of-contents/)
-   [【Next.js】microCMSで繰り返しフィールドから目次を作成する](https://ru-blog.com/nextjs-microcms-create-table-of-contents/)
-   [styled-componentsを使ったCSS設計](https://qiita.com/taneba/items/4547830b461d11a69a20)
-   [GatsbyJsでStyled-componentを使う](https://www.corylog.com/gatsby/gatsby028/)