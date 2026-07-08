# 第8章：ウィジェット

> 執筆者： HO CHI CHONG
> 最終更新：2026-07-08

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、WidgetKitを使ってホーム画面やロック画面に表示できるウィジェットを実装する方法を学ぶ。具体的には毎日異なる名言を表示するウィジェットを題材にして、TimelineProviderの仕組み、ウィジェットビューの構成、複数サイズへの対応、そしてメインアプリとの連携方法を学ぶ。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ここに模範コード全体を貼る

// MARK: - メインアプリのContentView

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                // 今日の名言（ハイライト）
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)

                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)

                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                // 全名言リスト
                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)
                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

簡単に言えば、今日の名言と名言一覧を表示するアプリ

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

<img width="314" height="684" alt="Image" src="https://github.com/user-attachments/assets/b0610848-883c-495d-9695-8b64cb6d3a63" />
<img width="321" height="674" alt="Image" src="https://github.com/user-attachments/assets/859b6be9-37c8-480a-ad2c-69c066d05850" />

## コードの詳細解説

このコードはデータ管理クラスQuoteStoreから今日の名言を呼び出し表示する
```
let todaysQuote = QuoteStore.todaysQuote()
```
```
QuoteStore
   │
   ├─ 名言1
   ├─ 名言2
   ├─ 名言3
   │
   └─ todaysQuote()
          ↓
      今日の名言
```

全体のイメージはこのように
```
ContentView
│
├── QuoteStoreから今日の名言を取得
│
├── QuoteStoreから全名言を取得
│
└── NavigationStack
      │
      └── VStack
            │
            ├── 今日の名言カード
            │      ├─ タイトル
            │      ├─ 名言
            │      └─ 作者
            │
            └── List
                  ├─ 名言①
                  ├─ 名言②
                  ├─ 名言③
                  └─ …
```

### TimelineProviderの仕組み

```swift
// 該当部分のコードを抜粋して貼る
//MARK: - タイムラインプロバイダ

struct QuoteProvider: TimelineProvider {
   //  プレースホルダー（読み込み中の仮表示）
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    // スナップショット（ウィジェットギャラリーでのプレビュー）
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

//  タイムライン（実際のウィジェット更新スケジュール）
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

      //   次の日の0時にウィジェットを更新
        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}
```

**何をしているか：**
（この部分が果たしている役割を説明する）

TimelineProviderは**「ウィジェットのスケジュール管理者」**です。

「何を表示するか（QuoteEntry）」と「いつ更新するか（Timelineとpolicy）」を決めてWidgetKitに渡し、WidgetKitがその予定に従ってホーム画面のウィジェットを表示・更新します。

仕組みとしてはこんな感じ:
```
ホーム画面
      │
      ▼
 WidgetKit
      │
      ▼
 QuoteProvider
      │
      ├── placeholder()
      │      ↓
      │   読み込み中
      │
      ├── getSnapshot()
      │      ↓
      │   見本を1枚作る
      │
      └── getTimeline()
             │
             ├── 今日の名言取得
             ├── Entry作成
             ├── 更新時刻を設定
             └── Timelineを返す
                    │
                    ▼
              WidgetKitが表示・更新
```

**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）

1.TimelineProviderを採用する場合は、この3つを実装しないとコンパイルエラーになります。

・placeholder()　→ データがまだ準備できていないときに呼ばれ、こういう状態になります。
```
ホーム画面

□□□□□□□□□□

読み込み中...

□□□□□□□□□□
```
だからAppleは「最低限の仮画面を返してください」と決めています。

・getSnapshot() → ウィジェットを追加するときに呼ばれ、こういう状態になります。

```
ウィジェット一覧

──────────

今日の名言

努力は才能を超える

──────────
```
Snapshot スナップショット＝写真という意味です。

・getTimeline() → 実際にホーム画面へ表示するときに呼ばれ、こういう状態になります。
```
今日

努力は才能を超える

↓

明日

失敗は成功のもと
```
表示内容と更新タイミングを教える必要があります。

WidgetKitは
```
今日表示

↓

明日の0時になった

↓

またgetTimeline()を呼ぶ
```
という動きをします。

→この3つのメソッドは、それぞれ「使われる場面」が違うからです。Apple（WidgetKit）が状況に応じて呼び分けています。


**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

1.もしplaceholderを使わなかったら、データがまだ準備できていないとき、何も表示できない

イメージはこうなります：
```
ホーム画面

□□□□□□□□□□

（何も表示できない）

□□□□□□□□□□
```

2.getSnapshot()を使わないと、ウィジェットに何も表示できなくなる
```
□□□□□□

？？？

□□□□□□
```

3.getTimeline()を使わないと、名言が更新できなくなる。

---

### TimelineEntryとウィジェットビュー

```swift
 //MARK: - タイムラインエントリ

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

//MARK: - ウィジェットのビュー

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }


```

**何をしているか：**

この部分は**「ウィジェットに表示するデータ」と「そのデータをどう画面に表示するか」をつなぐ役割**をしています。

**なぜこう書くのか：**

1.QuoteEntryとは？
```
struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}
```
・ウィジェットが1回表示するために必要なデータをまとめた箱です。

例えば今日が7月8日なら、この箱の中身は
```
QuoteEntry
├── date
│     2026/7/8
│
└── quote
      ├── text
      │     努力は才能を超える
      │
      └── author
            イチロー
```

2.TimelineEntryとは?


・「このデータはタイムラインで使うデータですよ」という約束（プロトコル）です。


3.QuoteWidgetEntryView

・やっていることの流れ：
```
QuoteProvider

↓
「今日の名言はこちらです！」
↓
QuoteEntry
・日付
・名言
↓
QuoteWidgetEntryView
↓
画面に表示
```

Viewの流れ：
```
QuoteProvider
      │
      │ 今日の名言を取得
      ▼
QuoteEntry
（日付＋名言）
      │
      ▼
QuoteWidgetEntryView
      │
      ├── family = systemSmall
      │         │
      │         ▼
      │    smallWidget
      │
      └── family = systemMedium
                │
                ▼
           mediumWidget
```


**もしこう書かなかったら：**

1.QuoteEntryにlet date: Dateをつけないとどうなる？

・TimelineEntryというプロトコルを使用しているので、それをつけないとエラーになる。



---

### ウィジェットサイズごとのレイアウト

```swift
// 該当部分のコードを抜粋して貼る
 
    // 小サイズ
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // 中サイズ
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}

```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

### メインアプリとの連携

```swift
// 該当部分のコードを抜粋して貼る
// MARK: - ウィジェット定義

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

 //MARK: - プレビュー

#Preview(as: .systemMedium) {
    QuoteWidget()
} timeline: {
    QuoteEntry(date: .now, quote: QuoteStore.todaysQuote())
}
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`TimelineProvider` | ウィジェットを更新するタイミングとコンテンツを定義 | `struct QuoteProvider: TimelineProvider { ... }` |
| 例：`@main` + `WidgetConfiguration` | ウィジェットのエントリーポイント | `@main struct QuoteWidget: Widget { ... }` |
| | | |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：
- 結果：
- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
