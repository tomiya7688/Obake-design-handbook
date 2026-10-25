# UPD DesignとUPD Commander Base Designの違い

[トップページ](../README.md) / [比較一覧](README.md) / UPD DesignとUPD Commander Base Designの違い

**両者は同じ設計ではありません。** UI・Process・Dataという3層を使う点は共通していますが、どこまで構造を決めるかが異なります。

- [UPD Design](../methods/upd-design.md) は、**どの種類の仕事を、どの層が担当するか**を決めます。
- [UPD Commander Base Design（原典）](https://github.com/tomiya7688/upd-commander-base-design) は、同じ3層を前提として、**誰が処理を呼び出し、どう層間で情報を受け渡すか**まで定めます。

## 違いの一覧

| 観点 | UPD Design | UPD Commander Base Design |
| --- | --- | --- |
| UI・Process・Data | 仕事の種類で3層に分ける | 同じ3層を使う |
| UIとDataの直接接続 | 禁止 | 禁止 |
| Commander | 必須ではない | 各層で処理の呼び出しを指揮する役割 |
| Messenger | 必須ではない | 層間で要求と結果を運ぶ役割 |
| Processing | 固有の構成要素として定めない | 各層の実際の仕事を担当する役割 |
| 層をまたぐ通信の実現 | 特定の方法に固定しない | Messengerを介して行う。同期・非同期や通信技術自体は固定しない |
| 導入時の主な確認事項 | 仕事の担当と層の境界を守っているか | 層の境界に加えて、呼び出し・通信・実処理の役割も分かれているか |

## 同じ「データを取得する」でも実装上の規則は違う

### UPD Designの場合

仕事の受け渡しは、基本的に次の形で表せます。

```text
UI → Process → Data
UI ← Process ← Data
```

ここでの矢印は、**どの層が、どの層に仕事を依頼し、結果を受け取るか**を表します。特定のクラスや関数の呼び出し方法までは指定しません。

### UPD Commander Base Designの場合

Processの実処理がデータを必要とした場合、Process ProcessingはDataへ直接要求を送りません。

```text
Process Processing
  ↓ 必要なデータを結果として伝える
Process Commander
  ↓ 送信を依頼する
Process Messenger
  ↓ 層間で要求を運ぶ
Data Messenger
  ↓ 受信内容を渡す
Data Commander
  ↓ 取得処理を呼び出す
Data Processing
```

取得した結果は、Data側のCommanderとMessengerを経由してProcess側へ戻ります。Commanderは処理の呼び出しを担当し、データ取得などの実作業を持ちません。

これは、3層の責務だけを定めるUPD Designにはない、**UPD Commander Base Designの追加の規則**です。

なお、Messengerという名前であってもメッセージキューや非同期通信を必須にするわけではありません。実際の通信手段は選べますが、役割の境界は守る必要があります。

## どちらを採用するか

**まずUI・Process・Dataの仕事を分けたい**なら、UPD Designの責務分離が目的に合うか確認します。CommanderやMessengerを導入しなくても、UPD Designの規則を守ることはできます。

**仕事の担当だけでなく、処理の呼び出し役と通信役も分けたい**なら、UPD Commander Base Designを検討します。その分、構成要素や受け渡しの規則が増えるため、追加の複雑さに見合う理由があるか考えます。

両者の関係は、独立した手法を単純に「組み合わせる」というよりも、**基本的な責務分離を、さらに具体的な構造へ発展させる関係**です。

## 原典

- [UPD Design — README](https://github.com/tomiya7688/upd-design/blob/main/README.md)
- [UPD Design — 必ず守る境界](https://github.com/tomiya7688/upd-design/blob/main/docs/developers/境界.md)
- [UPD Commander Base Design — README](https://github.com/tomiya7688/upd-commander-base-design/blob/main/README.md)
- [UPD Commander Base Design — CommanderとMessengerの役割](https://github.com/tomiya7688/upd-commander-base-design/blob/main/docs/commander-messenger.md)
- [UPD Commander Base Design — Data Flow](https://github.com/tomiya7688/upd-commander-base-design/blob/main/docs/data-flow.md)

---

[比較一覧へ戻る](README.md) · [UPD Designを読む](../methods/upd-design.md) · [トップページへ戻る](../README.md)
