---
title: TypeScript の Tips
publishedDate: "2021-05-19"
updatedDate: "2021-05-19"
description: TypeScript（や Node.js）を使っていて、何度も調べてしまうことを書いておく。ある程度まとまったら、別ページに移すかも。
slug: /memo/typescript/tips
---

## 日付処理

### 日付フォーマット

- デフォルトでは変換関数みたいなのがないっぽいので、自前で用意する必要があるっぽい。
    - [https://qiita.com/osakanafish/items/c64fe8a34e7221e811d0](https://qiita.com/osakanafish/items/c64fe8a34e7221e811d0)
- 以下のようなユーティリティ関数を用意しておくと良い

```typescript
function formatDate(date: Date, format: string): string {
  if (!format) format = "YYYY-MM-DD hh:mm:ss.SSS"

  format = format.replace(/YYYY/g, date.getFullYear().toString())
  format = format.replace(/MM/g, ("0" + (date.getMonth() + 1)).slice(-2))
  format = format.replace(/DD/g, ("0" + date.getDate()).slice(-2))
  format = format.replace(/hh/g, ("0" + date.getHours()).slice(-2))
  format = format.replace(/mm/g, ("0" + date.getMinutes()).slice(-2))
  format = format.replace(/ss/g, ("0" + date.getSeconds()).slice(-2))
  if (format.match(/S/g)) {
    var milliSeconds = ("00" + date.getMilliseconds()).slice(-3)
    var length = format.match(/S/g).length
    for (var i = 0; i < length; i++)
      format = format.replace(/S/, milliSeconds.substring(i, i + 1))
  }
  return format
}
```

### 文字列から Date への変換

- `Date.parse()` でミリ秒に変換したものを `new Date()` に入れれば良い。
    - [https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Date/parse](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Date/parse)
- `Date.parse()` はある程度無茶振りが利くっぽい。

```typescript
const unixTimeZero = Date.parse('01 Jan 1970 00:00:00 GMT');
const unixTimeZeroDate = new Date(unixTimeZero);
const javaScriptRelease = Date.parse('04 Dec 1995 00:12:00 GMT');
```

```typescript
let time = Date.parse("2001");
let date = new Date(time);
console.log(date.toISOString());  // 出力：2001-01-01T00:00:00.000Z

time = Date.parse("2001-02");
date = new Date(time);
console.log(date.toISOString());  // 出力：2001-02-01T00:00:00.000Z

time = Date.parse("2001-02-03");
date = new Date(time);
console.log(date.toISOString());  // 出力：2001-02-03T00:00:00.000Z

time = Date.parse("2001-02-03T04:05");
date = new Date(time);
console.log(date.toISOString());  // 出力：2001-02-02T19:05:00.000Z
time = Date.parse("2001-02-03T04:05+09:00");
date = new Date(time);
console.log(date.toISOString());  // 出力：2001-02-02T19:05:00.000Z
time = Date.parse("2001-02-03T04:05Z");
date = new Date(time);
console.log(date.toISOString());  // 出力：2001-02-03T04:05:00.000Z

time = Date.parse("2001-02-03T04:05:06");
date = new Date(time);
console.log(date.toISOString());  // 出力：2001-02-02T19:05:06.000Z
time = Date.parse("2001-02-03T04:05:06+09:00");
date = new Date(time);
console.log(date.toISOString());  // 出力：2001-02-02T19:05:06.000Z
time = Date.parse("2001-02-03T04:05:06Z");
date = new Date(time);
console.log(date.toISOString());  // 出力：2001-02-03T04:05:06.000Z

time = Date.parse("2001-02-03T04:05:06.078");
date = new Date(time);
console.log(date.toISOString());  // 出力：2001-02-02T19:05:06.078Z
time = Date.parse("2001-02-03T04:05:06.078+09:00");
date = new Date(time);
console.log(date.toISOString());  // 出力：2001-02-02T19:05:06.078Z
time = Date.parse("2001-02-03T04:05:06.078Z");
date = new Date(time);
console.log(date.toISOString());  // 出力：2001-02-03T04:05:06.078Z

time = Date.parse("+123456-07-08T09:10:11.012");
date = new Date(time);
console.log(date.toISOString());  // 出力：+123456-07-08T00:10:11.012Z
time = Date.parse("+123456-07-08T09:10:11.012+09:00");
date = new Date(time);
console.log(date.toISOString());  // 出力：+123456-07-08T00:10:11.012Z
time = Date.parse("+123456-07-08T09:10:11.012Z");
date = new Date(time);
console.log(date.toISOString());  // 出力：+123456-07-08T09:10:11.012Z

time = Date.parse("");
console.log(time);  // 出力：NaN
```