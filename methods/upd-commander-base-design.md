# UPD Commander Base Design

[トップページ](../README.md) / [設計手法の一覧](README.md) / UPD Commander Base Design

**UPD Commander Base Designは、アプリケーションの仕事を3つの層に分け、さらに「処理を呼び出す役」「層の間で情報を運ぶ役」「実際に処理する役」を分担させる設計手法です。**

このページでは、手法の目的、仕組み、守るべき規則、具体例、実装の進め方を説明します。

## 1. なぜこの手法が必要なのか

アプリケーションの機能が増えると、1つのクラスやファイルに多くの仕事が集まることがあります。例えば、購入処理を開始する部分が、入力の検証、金額の計算、データの保存、結果の表示まで引き受けてしまう状態です。

この状態では、処理の順序を変えたいだけなのに計算の内容にも触れる必要があり、どの部分を変更すればよいか分かりにくくなります。

UPD Commander Base Designは、**仕事を進める順序の管理と、実際の仕事を分けること**を重視します。大きな機能も小さな処理に分け、決められた担当を通して協力させることを目指します。

## 2. UI・Process・Dataという3つの層

「層」とは、仕事の種類ごとに分けたまとまりです。

| 層 | 担当する仕事 | 具体例 |
| --- | --- | --- |
| **UI** | 利用者の入力を受け取り、結果を表示する | 検索欄の読み取り、商品一覧の表示 |
| **Process** | アプリのルールに基づいて判断・計算する | 購入条件の判定、割引額の計算 |
| **Data** | データを取得・保存する | 商品の読み込み、注文の保存 |

**UIがDataを直接呼び出すことは認めません。** データが必要ならProcessを通して依頼します。Processは画面の表示方法を決めず、Dataはアプリのルールを判断しません。

## 3. 各層の中で分ける3つの役割

3つの層のそれぞれで、処理を次の役割に分けます。

| 役割 | 担当すること | 担当しないこと |
| --- | --- | --- |
| **Commander** | 次にどの処理を呼ぶか、どの順序で呼ぶかを決める | 金額計算、描画、ファイルの読み書きなどの実作業 |
| **Messenger** | 隣り合う層へ要求と結果を届ける | アプリのルールの判断、表示内容の決定、データの保存 |
| **Processing** | 自分の層が担当する実際の仕事を行う | 別の層への直接の要求や、その層の処理の呼び出し |

例えばUIのProcessingは入力欄を読み、ProcessのProcessingは入力内容を検証し、DataのProcessingはデータを保存します。

**Commanderは処理そのものを実行せず、適切なProcessingを呼び出します。** 別の層へ依頼する必要があれば、Messengerに送信を頼みます。Messengerは要求を運ぶ役であって、要求の内容から購入の可否などを決める役ではありません。

### 層と役割は別の分け方

層は「どの種類の仕事か」、役割は「その仕事をどのように受け持つか」を表します。例えば、UIにもProcessにもDataにも、必要なProcessingがあります。

```text
UI層
├── Commander   次に行うUI処理を選ぶ
├── Messenger   Processとの要求・結果を運ぶ
└── Processing  入力と表示を扱う

Process層
├── Commander   判断・計算の呼び出し順を決める
├── Messenger   UI・Dataとの要求・結果を運ぶ
└── Processing  アプリのルールを扱う

Data層
├── Commander   取得・保存の処理を選ぶ
├── Messenger   Processとの要求・結果を運ぶ
└── Processing  データを読み書きする
```

これは役割の説明図です。**必ず9個のクラスやファイルを作るという規則ではありません。** どの言語でも、役割と層の境界が守られるように構成します。Messengerも、層間通信が必要な場所に設けます。

## 4. どのように処理を受け渡すのか

例えば、検索画面で入力されたキーワードを使って商品情報を取得する場合を考えます。

1. **UI Processing**が検索欄からキーワードを読み、UI Commanderに返します。
2. **UI Commander**がUI Messengerに検索要求の送信を頼みます。UI MessengerはProcess Messengerへ要求を運びます。
3. **Process Messenger**が要求をProcess Commanderに渡します。Process CommanderはProcess Processingを呼び、検索条件を確かめさせます。
4. 商品情報が必要になったら、**Process Processing**はその必要性を結果としてProcess Commanderに返します。Process ProcessingからDataへ直接要求しません。
5. **Process Commander**がProcess Messengerにデータの要求を依頼します。要求はData Messengerへ届き、Data CommanderがData Processingを呼んで商品を取得します。
6. 取得結果はData CommanderとData MessengerからProcess側へ戻ります。Process Commanderは必要な処理を再開し、最終結果をUI側へ返します。
7. **UI Commander**はUI Processingを呼び、受け取った商品一覧を画面に表示させます。

特に重要なのは、**Processingが別の層のProcessingを直接呼ばないこと**です。層を越えてやり取りするときはMessengerを通し、必要な処理をCommanderが呼び出します。

この流れは仕事と受け渡しの関係を表すものです。必ず同期的に処理するという意味ではありません。

### Messengerは特別な通信技術を要求しない

Messengerという名前でも、メッセージキューや非同期処理を必ず使うわけではありません。通常の関数呼び出しでも実装できます。

重要なのは通信方法ではなく、**層を越える情報の受け渡しをMessengerが担当すること**です。送信内容も、画面部品やデータベース接続そのものではなく、受信先に必要な値にします。

## 5. 実装の進め方

まずは1つの機能を取り上げ、次の順序で役割を整理します。

1. **仕事を3つの層に分ける。** 入力と表示はUI、アプリの判断はProcess、データの読み書きはDataとします。
2. **実際の仕事をProcessingに置く。** 例えば、検索条件の検証と商品の取得は、それぞれProcessとDataのProcessingが担当します。
3. **呼び出し順をCommanderにまとめる。** 処理結果に応じて次の処理を選びます。Commander自身に検証や計算を書きません。
4. **層間の受け渡しをMessengerに任せる。** どの要求を送り、成功・失敗を含むどんな結果を返すか決めます。
5. **機能を1つ通して確認する。** 期待どおり動くことに加え、Processingが別層を直接呼んでいないか、CommanderやMessengerに実処理が入っていないかを調べます。

例えば「読み込み→検証→変換→保存」という長い処理がある場合、各処理は担当する層のProcessingに分け、Commanderが必要な順序で呼び出す形を考えます。こうすることで、Commanderが大きな計算や保存処理を抱え込まずに済みます。

必要ならCommanderを階層化し、上位のCommanderが下位のCommanderを呼ぶ形にできます。ただし、役割を分けたからといって、必ず階層を深くする必要はありません。

## 6. どんなプロジェクトに向いているか

次のような場合に、導入を検討できます。

- 機能の中に検証、計算、保存など多くの仕事があり、呼び出し順を整理したい
- 1つのクラスが処理の開始から終了まで何でも引き受けてしまっている
- 画面の変更、アプリのルールの変更、データの取得方法の変更を分けて確認したい
- 層の間で要求と結果がどこを通るかを明確にしたい

一方、仕事が少ない機能では、処理を呼ぶ部分と実処理を分けても、分かりやすさがあまり変わらない場合があります。

導入時には、**各役割が受け持つ処理、層の間で渡す情報、処理が失敗したときの受け渡し方**を決めます。新しく作った呼び出し経路が正しくつながっているかもテストする必要があります。こうした作業を行う意味があるか、自分のプロジェクトで確かめてください。

## 7. 守る規則と、自由に決めてよいこと

### 守る規則

- UI・Process・Dataの仕事を混ぜず、UIとDataを直接つながない
- Commanderに計算、描画、データの読み書きなどの実処理を書かない
- Messengerにアプリのルールの判断や実処理を書かない
- Processingから別の層のProcessingを直接呼ばない
- 層をまたぐ要求と結果はMessengerを通す

### 自由に決めてよいこと

- クラス、関数、モジュール、ファイルなど、各役割の実装方法
- 同期処理か非同期処理か、通信に使う技術
- 役割を分離したうえでのファイルの数や配置
- 必要に応じて値をまとめる補助部品を使うかどうか

例えば[Compresser](https://github.com/tomiya7688/upd-commander-base-design/blob/main/specification/compresser-spec.md)は、複数の値を1つの受け渡し単位にまとめるための補助です。必要な場合に検討するもので、最初から導入する必須条件ではありません。

## 8. 間違いやすい実装

**Commanderの中に計算式を書く**

「どの計算を実行するか」はCommanderが決めます。「どう計算するか」はProcessingが担当します。

**Messengerが受け取った値で購入可否を判断する**

Messengerは値を運ぶ役です。購入可否など、アプリのルールに関する判断はProcessのProcessingが担当します。

**Process Processingが直接Data Processingを呼ぶ**

層を越える要求は、Process CommanderがMessengerに送信を依頼し、Data側のMessengerとCommanderを経由して行います。

**名前だけを分ける**

Commanderという名前のクラスを作っても、その中に画面操作や保存処理が混ざっていれば、役割を分離したことにはなりません。

## 9. 原典・補助ツール

この解説は、次の日本語文書をもとにしています。詳しい規則や実装例も原典で確認できます。

- [UPD Commander Base Design（原典リポジトリ）](https://github.com/tomiya7688/upd-commander-base-design)
- [全体構造と3つの役割](https://github.com/tomiya7688/upd-commander-base-design/blob/main/docs/architecture.md)
- [CommanderとMessengerの役割](https://github.com/tomiya7688/upd-commander-base-design/blob/main/docs/commander-messenger.md)
- [要求と結果の流れ](https://github.com/tomiya7688/upd-commander-base-design/blob/main/docs/data-flow.md)
- [小さな機能を作るチュートリアル](https://github.com/tomiya7688/upd-commander-base-design/blob/main/docs/minimal-tutorial.md)
- [必須規則と推奨事項の区別](https://github.com/tomiya7688/upd-commander-base-design/blob/main/specification/guideline-levels.md)

原典には設計上の問題を探すための検査ツールもあります。ただし、ツールが問題を検出しなかったことは、すべての仕事の分け方が正しいことを保証しません。

---

[設計手法の一覧へ戻る](README.md) · [トップページへ戻る](../README.md) · [設計手法の選び方](../guides/selection.md)
