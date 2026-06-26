# AI質問ログ：第7章 センサーの活用

## 使用した生成AIツール

（例：ChatGPT 無料版 / Claude 無料版 / Gemini など）

## 質問と回答の記録

### Q1

**質問：weak selfの仕組みについて**

（生成AIに聞いた質問をそのまま書く）

**AIの回答の要点：**
（長い回答の場合、要点だけ自分の言葉でまとめる）

Swift の weak self は、**クロージャ（closure）の中で self を参照するときに**、強い参照（strong reference）による**メモリリークを防ぐため**の仕組みです。

そもそもメモリリークってなに？メモリリークすると具体的に何が困るの？

**1. 画面を閉じても ViewController が消えない**
```
class ViewController: UIViewController {

    var timer: Timer?

    override func viewDidLoad() {
        super.viewDidLoad()

        timer = Timer.scheduledTimer(
            withTimeInterval: 1,
            repeats: true
        ) { _ in
            self.update()
        }
    }

    func update() {
        print("更新")
    }
}
```
この場合：
```
ViewController
      |
      | strong
      ↓
Timer
      |
      | strong
      ↓
closure
      |
      | strong
      ↓
ViewController(self)
```
という輪ができます。ユーザーが画面を閉じても：
```
navigationController?.popViewController()
```
しても ViewController が解放されません。

->deinit が呼ばれない

->メモリに残り続ける


**2.メモリ使用量が徐々に増える**

**3.アプリが重くなる・落ちる**

OSは使えるメモリに限界があります。

メモリが圧迫されると：

・動作がカクつく

・画像読み込み失敗

・バックグラウンドから戻れない

・最悪アプリ強制終了


例えば:
```
アプリ
 ├─ 今表示している画面
 ├─ 過去の画面
 ├─ 古い画像
 ├─ 古い通信結果
 └─ 古いTimer
```


**自分の理解：**
（回答を受けて自分がどう理解したか。納得できたか、さらに疑問が生まれたか）

### Q2

**質問：**

**AIの回答の要点：**

**自分の理解：**

### Q3

**質問：**

**AIの回答の要点：**

**自分の理解：**

（質問は何個でも追加してください。多ければ多いほど良いです。）

## 今日の質問を振り返って

（どんな質問が良い質問だったか。生成AIの回答で間違いや不正確な部分はあったか。次回はどんな質問をしてみたいか。）
