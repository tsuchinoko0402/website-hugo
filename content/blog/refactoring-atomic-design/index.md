---
title: Atomic Design を参考に Gatsby のサイトのコンポーネント分割を考える
date: 2021-05-16
slug: refactoring-atomic-design
tags:
  - UI/UX
  - MaterialUI
---
とりあえず作ったサイトですが、いろいろ直したいところが山積しているので一つずつ片付けていこうという算段です。  
  
まずは **単体テストが書けてない** というのと、 **デザインをなんとかできるようにしたい** という問題を解決すべく、コンポーネント分割を見直すことにしました。  
世の中のトレンドが分かっていなさそうなのですが、 Atomic Design の考え方が合っている気がしたので、色々と見てみました：

-   [Atomic Design](https://atomicdesign.bradfrost.com/)
-   [Atomic Design を分かったつもりになる](https://design.dena.com/design/atomic-design-%E3%82%92%E5%88%86%E3%81%8B%E3%81%A3%E3%81%9F%E3%81%A4%E3%82%82%E3%82%8A%E3%81%AB%E3%81%AA%E3%82%8B)
-   [React Hooks ✕ AtomicDesignで画像投稿webアプリのフロントエンドを実装する](https://qiita.com/stranger1989/items/74b23062fa25e975af94)

React の UI フレームワークとして、 [Material UI](https://material-ui.com/) も使っているので、これらを使いながら Atomic Design の考え方に沿ってコンポーネント分割を再考する、という作業を始めています。  
  
厳密に Atomic Design に従おうとすると「おや？」というところが出てきてしまうと思いますが、まずはちょっと大目に見ながらぼちぼち進めていきます。