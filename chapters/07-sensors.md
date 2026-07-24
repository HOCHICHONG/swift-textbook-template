# 第7章：センサーの活用

> 執筆者： HO CHI CHONG
> 最終更新：2026-07-24

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、iPhoneに搭載されている加速度・ジャイロなどのセンサーにアクセスして、デバイスの動きや姿勢を検出する方法を学ぶ。具体的にはCoreMotionを使った水平器アプリ、CMPedometerとCoreLocationを組み合わせた活動トラッカーを題材にして、センサーデータの読み取りと処理の実装を学ぶ。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ここに模範コード全体を貼る
// ============================================
// 第7章（基本）：加速度センサーで動く水平器アプリ
// ============================================
// CoreMotionを使って端末の傾きをリアルタイムで取得し、
// 水平器（水準器）として表示するアプリです。
//
// 【注意】シミュレータではセンサーが使えません。
//         実機（iPhone / iPad）でテストしてください。
// ============================================

import SwiftUI
import CoreMotion

// MARK: - モーションマネージャー

@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool

    init() {
        // 初回 body 評価時点で正しい値を返すよう、init で同期的にセット
        isAvailable = motionManager.isDeviceMotionAvailable
    }

    func startUpdates() {
        guard isAvailable else { return }

        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var motionManager = MotionManager()

    var body: some View {
        NavigationStack {
            if motionManager.isAvailable {
                VStack(spacing: 30) {
                    // 水平器の円
                    LevelIndicator(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll
                    )

                    // 数値表示
                    DataDisplay(
                        pitch: motionManager.pitch,
                        roll: motionManager.roll,
                        yaw: motionManager.yaw
                    )
                }
                .padding()
                .navigationTitle("水平器")
            } else {
                ContentUnavailableView(
                    "センサーが利用できません",
                    systemImage: "iphone.slash",
                    description: Text("このアプリは実機（iPhone）で動作します。\nシミュレータではセンサーが使えません。")
                )
            }
        }
        .onAppear {
            motionManager.startUpdates()
        }
        .onDisappear {
            motionManager.stopUpdates()
        }
    }
}

// MARK: - 水平器インジケーター

struct LevelIndicator: View {
    let pitch: Double
    let roll: Double

    private let maxOffset: CGFloat = 100

    private var xOffset: CGFloat {
        CGFloat(roll) * maxOffset
    }

    private var yOffset: CGFloat {
        CGFloat(pitch) * maxOffset
    }

    private var isLevel: Bool {
        abs(pitch) < 0.03 && abs(roll) < 0.03
    }

    var body: some View {
        ZStack {
            // 外側の円
            Circle()
                .stroke(.gray.opacity(0.3), lineWidth: 2)
                .frame(width: 250, height: 250)

            // 中心の十字線
            Path { path in
                path.move(to: CGPoint(x: 125, y: 0))
                path.addLine(to: CGPoint(x: 125, y: 250))
                path.move(to: CGPoint(x: 0, y: 125))
                path.addLine(to: CGPoint(x: 250, y: 125))
            }
            .stroke(.gray.opacity(0.2), lineWidth: 1)
            .frame(width: 250, height: 250)

            // 中間の円
            Circle()
                .stroke(.gray.opacity(0.2), lineWidth: 1)
                .frame(width: 125, height: 125)

            // バブル（傾きに応じて移動）
            Circle()
                .fill(isLevel ? .green : .red)
                .frame(width: 40, height: 40)
                .opacity(0.8)
                .shadow(color: isLevel ? .green : .red, radius: 8)
                .offset(
                    x: max(-maxOffset, min(maxOffset, xOffset)),
                    y: max(-maxOffset, min(maxOffset, yOffset))
                )
                .animation(.spring(duration: 0.1), value: xOffset)
                .animation(.spring(duration: 0.1), value: yOffset)

            // 水平時の表示
            if isLevel {
                Text("水平!")
                    .font(.headline)
                    .foregroundStyle(.green)
                    .offset(y: 140)
            }
        }
    }
}

// MARK: - 数値データ表示

struct DataDisplay: View {
    let pitch: Double
    let roll: Double
    let yaw: Double

    var body: some View {
        VStack(spacing: 12) {
            DataRow(
                label: "前後の傾き（Pitch）",
                value: pitch,
                icon: "arrow.up.and.down"
            )
            DataRow(
                label: "左右の傾き（Roll）",
                value: roll,
                icon: "arrow.left.and.right"
            )
            DataRow(
                label: "水平回転（Yaw）",
                value: yaw,
                icon: "arrow.triangle.2.circlepath"
            )
        }
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(.gray.opacity(0.05))
        )
    }
}

struct DataRow: View {
    let label: String
    let value: Double
    let icon: String

    var body: some View {
        HStack {
            Image(systemName: icon)
                .frame(width: 30)
                .foregroundStyle(.blue)

            Text(label)
                .font(.caption)

            Spacer()

            Text(String(format: "%.3f rad", value))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)

            Text(String(format: "(%.1f°)", value * 180 / .pi))
                .font(.system(.caption, design: .monospaced))
                .foregroundStyle(.secondary)
                .frame(width: 60, alignment: .trailing)
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

**このアプリはiPhoneの加速度センサー（正確にはCore Motionのデバイスモーション）を使って、端末の傾きをリアルタイム表示する水平器アプリです。**

## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
// 該当部分のコードを抜粋して貼る
@Observable
class MotionManager {
    private let motionManager = CMMotionManager()

    var pitch: Double = 0    // 前後の傾き
    var roll: Double = 0     // 左右の傾き
    var yaw: Double = 0      // 水平方向の回転
    var isAvailable: Bool

    init() {
        // 初回 body 評価時点で正しい値を返すよう、init で同期的にセット
        isAvailable = motionManager.isDeviceMotionAvailable
    }

    func startUpdates() {
        guard isAvailable else { return }

        motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

        motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
            guard let self = self, let motion = motion else { return }

            self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
        }
    }

    func stopUpdates() {
        motionManager.stopDeviceMotionUpdates()
    }
}
```

**何をしているか：**
（この部分が果たしている役割を説明する）

iPhoneの姿勢センサーを起動して、端末の傾きをSwiftUIに渡す管理クラス

仕組みとしては：
```
ユーザーがiPhoneを傾ける
          ↓
加速度センサー・ジャイロ
          ↓
CMMotionManager
          ↓
motion.attitude
          ↓
pitch / roll / yaw更新
          ↓
@Observableが変更検知
          ↓
SwiftUI再描画
          ↓
バブルが動く
```

**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）

1.@Observableをつけることによって値が変更された時画面が更新する。

流れとして：
```
センサー
   ↓
MotionManagerの値変更
   ↓
SwiftUIが検知
   ↓
画面再描画
```

2.CMMotionManager 

-> CMMotionManager はAppleが用意しているセンサー管理クラス

これを使うことで：

・加速度

・ジャイロ

・磁気センサー

・端末姿勢

を取得できます。

3.センサーの値の保存
```
var pitch: Double = 0 //前後
var roll: Double = 0 //左右
var yaw: Double = 0 //回転方向
```

4.更新の頻度

```
motionManager.deviceMotionUpdateInterval = 1.0 / 60.0

//つまり
//1 ÷ 60 = 0.016秒
//毎秒60回センサーを読む。
```

5.センサー開始の処理
```
//DeviceMotionのデータを取得して、mainスレッドで返す
motionManager.startDeviceMotionUpdates(
    to: .main 
)
```
-> なせ.mainなのか？

・それはSwiftUIの画面更新はメインスレッドで行うからです。

6.weak self を使う理由
stopUpdates() が絶対に呼ばれる保証はないからです。

例えば：

・アプリが強制終了

・View構造が変わる

・エラー発生

・別のコード変更


**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

1.メモリ管理でweak selfを使用
「なくなっていたら無視して」を指定できる。

weak selfは循環参照の防止のためつけたもの。（クロージャが長時間 self を保持）

それをつけないと
```
MotionManager
 ↓
CMMotionManager
 ↓
クロージャ
 ↓
MotionManager
```
weak selfを使わないとメモリリーク（循環参照） が起きる可能性がある
```
motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
    guard let self = self else { return }

    self.pitch = motion.attitude.pitch
}
```
例えばここにweak selfを抜けると

オブジェクト同士の関係はこうなる
```
MotionManager
      |
      | owns
      ↓
CMMotionManager
      |
      | owns
      ↓
クロージャ
      |
      | owns
      ↓
MotionManager
```
つまり：
```
MotionManager → CMMotionManager → closure → MotionManager
```
という輪ができます。これを 循環参照（retain cycle） と呼びます。

何故それが問題になる？
Swiftのメモリ管理（ARC）は、「誰からも参照されなくなったオブジェクトを削除する」ことから

循環がある場合：
```
MotionManager
 ↑          ↓
 └──────────┘
```
お互いが「まだ必要」と思ってしまう

結果:
```
参照数 0にならない
↓
削除されない
↓
メモリに残り続ける
```
これがメモリリークです。

このアプリにおいて：
```
画面消える
↓
MotionManagerは残る
↓
センサー更新継続
↓
メモリ使用継続
```
になる可能性があります。


2.停止処理
```
func stopUpdates()
{
    motionManager.stopDeviceMotionUpdates()
}
```
これでセンサー停止。
```
View消える
 ↓
センサー停止
 ↓
バッテリー節約
```
になる

それをしないとセンサー停止が停止しないことになり、メモリーが重くなりiPhoneが固まってしまう、あとはバッテリーの消耗も激しい。


---

### デバイスの姿勢データ（pitch/roll/yaw）

```swift
// 該当部分のコードを抜粋して貼る

motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
    guard let self = self, let motion = motion else { return }

self.pitch = motion.attitude.pitch
            self.roll = motion.attitude.roll
            self.yaw = motion.attitude.yaw
```

**何をしているか：**

水平器アプリの「センサー → 角度データ」への変換部分

**なぜこう書くのか：**

1.センサー開始の処理
```
//DeviceMotionのデータを取得して、mainスレッドで返す
motionManager.startDeviceMotionUpdates(
    to: .main 
)
```
-> なせ.mainなのか？

・それはSwiftUIの画面更新はメインスレッドで行うからです。

2.weak self を使う理由
stopUpdates() が絶対に呼ばれる保証はないからです。

例えば：

・アプリが強制終了

・View構造が変わる

・エラー発生

・別のコード変更


**もしこう書かなかったら：**

1.メモリ管理でweak selfを使用
「なくなっていたら無視して」を指定できる。

weak selfは循環参照の防止のためつけたもの。（クロージャが長時間 self を保持）

それをつけないと
```
MotionManager
 ↓
CMMotionManager
 ↓
クロージャ
 ↓
MotionManager
```
weak selfを使わないとメモリリーク（循環参照） が起きる可能性がある
```
motionManager.startDeviceMotionUpdates(to: .main) { [weak self] motion, error in
    guard let self = self else { return }

    self.pitch = motion.attitude.pitch
}
```
例えばここにweak selfを抜けると

オブジェクト同士の関係はこうなる
```
MotionManager
      |
      | owns
      ↓
CMMotionManager
      |
      | owns
      ↓
クロージャ
      |
      | owns
      ↓
MotionManager
```
つまり：
```
MotionManager → CMMotionManager → closure → MotionManager
```
という輪ができます。これを 循環参照（retain cycle） と呼びます。

何故それが問題になる？
Swiftのメモリ管理（ARC）は、「誰からも参照されなくなったオブジェクトを削除する」ことから

循環がある場合：
```
MotionManager
 ↑          ↓
 └──────────┘
```
お互いが「まだ必要」と思ってしまう

結果:
```
参照数 0にならない
↓
削除されない
↓
メモリに残り続ける
```
これがメモリリークです。

このアプリにおいて：
```
画面消える
↓
MotionManagerは残る
↓
センサー更新継続
↓
メモリ使用継続
```
になる可能性があります。


---

### 歩数計（CMPedometer） SensorAdvanceから

```swift
// 該当部分のコードを抜粋して貼る
@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    // 歩数関連
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0     // メートル
    var isPedometerAvailable: Bool = false

    // 位置関連
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0  // m/s
    var locations: [CLLocationCoordinate2D] = []

    // 状態
    var isTracking: Bool = false
    var startTime: Date?
    var endTime: Date?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }
```

**何をしているか：**

役割を一言でいうと、「運動データ（歩数・距離・速度・位置）を取得して管理する司令塔」です。

流れとして：
```
ActivityTracker作成
        │
        ├─ CMPedometer（歩数計）を準備
        │
        ├─ CLLocationManager（GPS）を準備
        │
        ├─ GPSのDelegateを設定
        │
        ├─ GPSの精度を設定
        │
        ├─ 位置情報の使用許可を要求
        │
        └─ 歩数計が使えるか確認
```

**なぜこう書くのか：**

まずはクラス宣言の部分から説明：

1.@Observable -> 値が変化するとSwiftUIへ通知します

例えば:
```
stepCountが変更
      ↓
@Observableが検知
      ↓
SwiftUIが再描画
      ↓
画面の歩数表示が更新
```

2.NSObject -> CLLocationManager はObjective-Cで作られたクラスなので、Delegateを使うには NSObject を継承する必要があります

3.CLLocationManagerDelegate -> これは「位置情報が更新されたら教えてください」という約束（プロトコル⭐️）です。

例えば後で
```
func locationManager(
    _ manager: CLLocationManager,
    didUpdateLocations locations: [CLLocation]
)
```
を書くことで、GPSが更新されるたびにこの関数が呼ばれます。


歩数センサーの流れ：
```
iPhone
   │
歩数センサー
   │
CMPedometer
   │
ActivityTracker
```

GPSに関してクラス：
```
private let locationManager = CLLocationManager()
```
イメージとして：
```
GPS
 │
CLLocationManager
 │
ActivityTracker
```

場所の管理:
```
var locations: [CLLocationCoordinate2D] = []
```
配列に管理する

イメージとしては:
```
スタート
 ↓
A
 ↓
B
 ↓
C
```
Swiftにおいては：
```
[
 A,
 B,
 C
]
```
こういうイメージになります。


**もしこう書かなかったら：**

1.NSObjectを継承しなたっらどうなる？ 

-> CLLocationManagerDelegate を正しく利用できません。

原因：CLLocationManagerDelegate は Objective-C の仕組み（Delegate）を利用するプロトコルだからです。

なせNSObjectが必要？

CLLocationManagerは
```
locationManager.delegate = self
```
によって、「位置情報が更新されたらこのクラスに知らせる」という仕組みになっています。

流れも見てみよう：
```
GPS
 ↓
CLLocationManager
 ↓
Delegate
 ↓
ActivityTracker
```
この Delegate の仕組みは Objective-C のランタイムを利用しているため、NSObject を継承したクラスであることが前提になっています。


NSObject は Apple の多くのフレームワークの基礎となるクラスです。

継承すると、

・Delegateを使える

・Objective-Cとの連携ができる

・KVO（Key-Value Observing）などの機能が使える

といった機能が利用できます。

---

### CoreLocationとの連携

```swift
// 該当部分のコードを抜粋して貼る

import CoreLocation

class ActivityTracker: NSObject, CLLocationManagerDelegate {

    private let locationManager = CLLocationManager()

    var currentSpeed: Double = 0
    var locations: [CLLocationCoordinate2D] = []

    override init() {
        super.init()

        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
    }

    func startTracking() {
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        locationManager.stopUpdatingLocation()
    }

    func locationManager(
        _ manager: CLLocationManager,
        didUpdateLocations locations: [CLLocation]
    ) {
        guard let location = locations.last else { return }

        currentSpeed = max(0, location.speed)
        self.locations.append(location.coordinate)
    }
}
```

**何をしているか：**

簡単に説明すればCoreLocationは「CLLocationManagerがGPSから位置情報を取得し、その結果をデリゲートメソッド locationManager(_:didUpdateLocations:) で受け取って利用する仕組み」です。

流れとしては:
```
① CLLocationManager を作る
          │
          ▼
② Delegate を設定する
          │
          ▼
③ 位置情報の利用許可を求める
          │
          ▼
④ startUpdatingLocation()
          │
          ▼
⑤ GPSが現在地を取得する
          │
          ▼
⑥ didUpdateLocations が自動で呼ばれる
          │
          ├── 速度を取得
          └── 座標を保存
          │
          ▼
⑦ stopUpdatingLocation() で取得終了
```

**なぜこう書くのか：**

1.CLLocation をimport

CoreLocationというフレームワークを使えるようにします。
このフレームワークには
```
現在地
緯度・経度
速度
高度
```
などを取得する機能があります。

2.位置情報の取得(GPS)
```
private let locationManager = CLLocationManager()
```
```
ActivityTracker
      │
      │
      ▼
CLLocationManager
      │
      ▼
GPS
```
位置情報を管理するオブジェクトからGPSとやり取りして、位置情報を取得してくれます

3.
**もしこう書かなかったら：**

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`CMMotionManager` | 加速度・ジャイロ・気圧などのセンサーデータを取得 | `motionManager.startDeviceMotionUpdates(to: .main) { ... }` |
| 例：`CMPedometer` | 歩数や歩行距離をカウント | `pedometer.queryPedometerData(from: startDate, to: Date())` |
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
