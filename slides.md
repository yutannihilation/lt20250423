---
title: HTTP/3なんて簡単さ、おれは少なくとももう10回は入門に成功しているぜ。
format:
  gfm:
    variant: +yaml_metadata_block
theme: default
background: /background.png
fonts:
  sans: Noto Sans JP
  mono: Fira Code
  weights: 600,900
class: text-center
drawings:
  persist: false
mdc: true
---

# HTTP/3なんて簡単さ、おれは少なくとももう10回は入門に成功しているぜ。

2025/04/23

@yutannihilation

---
layout: image-right
image: "/icon.jpg"
---

# ドーモ！

## 名前:

湯谷啓明 (@yutannihilation)

## 自己紹介:

- 株式会社 MIERUNE で GIS エンジニア見習いをしています。

---

# tl;dr

- また入門に失敗しました。
- 何もわからないので教えてください。

---

# きっかけ

- うちのCTOが書いた本

<img src="/books.jpg" class="h-100 pb-10" />

---

# 感想

- バックエンドで GeoJSON に変換して、それをブラウザ上でデコードして、という処理がやたらある
- Apache Arrow でなんとかなったりしない？

---

# Apache Arrow

- 高速なデータのやりとりとインメモリのデータ分析のためのデータフォーマット
- 列指向
- 様々な言語のライブラリが用意されている

---

# GeoArrow

- Apache Arrow 形式で GIS データを扱う際の仕様
- Apache Arrow 本体と比べると実装の数は少ないが、`geoarrow-wasm` があるのでブラウザ上でも扱えるはず

---

# まだよくわかっていないところ

- 最終的に、GPU のバッファに直接入れて、MapLibre や deck.gl といったライブラリが描画するところも変換なしでいけるとうれしいけど、内部実装がどうなってるかわからない

---

# 本題

- Apache Arrow のデータは、テーブル全体を一気に送るのではなく、**record batch** というチャンクで少しづつ送られる。

```rust
let mut db = driver.new_database()?;
let mut conn = db.new_connection()?;

let mut stmt = conn.new_statement()?;
stmt.set_sql_query("SELECT * FROM tbl")?;

let record_batch_reader = stmt.execute()?;
```

---

- ナイーブなやり方としては、いったんバッファに溜めて、まとめて返せばいい。

```rust
let mut result: Vec<u8> = Vec::new();
let mut writer = StreamWriter::try_new(
    &mut result,
    &*schema_ref
).unwrap();

for batch in record_batch_reader {
    writer.write(batch)?;
}
result
```

---

- でも、せっかく小分けになってきてるから、ストリームで返したい。
- ストリームで返すということは...、よく知らないけど HTTP/2 とか HTTP/3 ってやつか？？（雑な発想）
- その後、Rust むずすぎ問題（？）でできないことが判明するが、それはまた別の日の話にしましょう...

---

# HTTP/2・HTTP/3 とは

* ※ここから先は素人の説明なので話半分で聴いてください。

---

# HTTP/2・HTTP/3 とは

<table>
  <thead>
    <tr>
      <th scope="col"></th>
      <th scope="col">HTTP/2</th>
      <th scope="col">HTTP/3</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">下位プロトコル</th>
      <td>TCP+TLS</td>
      <td>QUIC</td>
    </tr>
    <tr>
      <th scope="row">TLS</th>
      <td>実質必須</td>
      <td>必須</td>
    </tr>
    <tr>
      <th scope="row">IETF標準化</th>
      <td>2015年</td>
      <td>2022年</td>
    </tr>
  </tbody>
</table>

---

# HTTP/3 のメリット

- 0-RTT
- パケットロスが起こっても影響が少ない
- 接続するネットワークが変わっても通信を継続できる（例：スマホ持ち歩き）
- セキュリティの向上

---

# 留意点

- TCP で安定して通信できる環境なら、HTTP/2 より速いわけではない
- HTTP/3（というか QUIC）にまだ対応していないプロダクトも多い

---

# 個人的に大変だと思ったところ

- 世の中のプロダクトの QUIC 対応がまだまだ
- ブラウザで確認しづらい
- end to end で暗号化

---

# 例：curl

``` console
❯ curl.exe --http3
```

`curl: option --http3: the installed libcurl version does not support this`

---

# 例：OpenSSL

- 4月8日にリリースされたバージョン3.5.0でようやくserver-side QUICをサポート
  - 今は「HTTP/3を有効にしたいならBoringSSLをビルドしてね」というプロダクトが多いけど、これで徐々に状況が変わるのかも...？

---

# Go
- [quic-go](https://github.com/quic-go/quic-go)
- 標準ライブラリでは、[net/internal](https://pkg.go.dev/golang.org/x/net/internal/http3) には入っているが、まだ net/http3 はできていない

---

# Rust
- [quiche](https://github.com/cloudflare/quiche): Cloudflare。最も古くからある？
- [h3](https://github.com/hyperium/h3): hyper の拡張。いずれは hyper に取り込まれるらしい。
- [s2n-quic](https://github.com/aws/s2n-quic): AWS。CloudFront で使われているらしい。

---

# ブラウザで確認しづらい

- TLS通信必須なので証明書が必要
  - 自己署名証明書だと通信できない
- ブラウザが HTTP/3 を選ぶ際の挙動がよくわからない

---

# 証明書

- これはなんとかなった。

<a href="https://github.com/FiloSottile/mkcert" target="_blank">
  <img src="/mkcert.png" class="h-100 pb-10" />
</a>

---

# ブラウザが HTTP/3 を選ぶ際の挙動

- 仕様上は、以下の2つの方法がある。
  1. DNS HTTPS レコードを設定する
  2. Alt-Svc ヘッダをつける（HTTP/1、HTTP/2 のレスポンスに対して）
- ローカルでの開発なので DNS レコードはちょっと使いづらい。

---

# ブラウザの挙動

- とりあえず、Chrome の挙動は、
  - 初手はまず HTTP/2 を試すっぽい
  - そこで Alt-Svc ヘッダがあると HTTP/3 にアップグレードしようとしてくれる。が...
  - その HTTP/3 の通信に失敗すると、その失敗をしばらく覚えているらしく、リロードしても HTTP/3 では通信してくれなくなる

---

# ブラウザの挙動

- HTTP/3 を優先するようにオプションを付けて起動すると当然 HTTP/3 で通信してくれるけど、それだとふつうのユーザーがアクセスしたときも同じなのか確信が持てない。。
- 自分の実装は、なぜか HTTP/2 から HTTP/3 に切り替わるところで TLS 通信エラーになってしまった。Wireshark 力がないので、調査できず断念...

---

# end to end で暗号化ということは

<img src="/e2e.jpg" class="h-100 pb-10" />

---

# end to end で暗号化ということは

<img src="/e2e-2.jpg" class="h-100 pb-10" />

---

# end to end で暗号化ということは

<img src="/e2e-3.jpg" class="h-100 pb-10" />


---

# end to end で暗号化ということは

- HTTP/3 のサーバーを作るのであれば、「TLS はリバースプロキシに任せた！」みたいなことはできず、そのサーバー自身が TLS 通信できないといけない
  - CPU 負荷が不安
- たぶん、サーバー自身は h2c（平文の HTTP/2 通信）で喋って、HTTP/3 はリバースプロキシに任せる、とかが正解な気がする

---

# 感想

- HTTP/3 の入門にまた失敗しました。
- とはいえ、地図アプリとかって外でスマホで見るし、大量のファイルをやりとりするので、HTTP/3 の恩恵自体はある気がする。たぶん GIS エンジニアとしてはもうちょっと分かっといた方がよさそう。
