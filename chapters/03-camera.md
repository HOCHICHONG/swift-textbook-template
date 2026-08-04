# 第3章：カメラの利用

> 執筆者：（氏名）
> 最終更新：2026-05-24

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、PhotosPickerでフォトライブラリから写真を選択し、UIImagePickerControllerでカメラ撮影した画像を扱う方法を学ぶ。具体的には非同期で画像データを読み込み、UIViewControllerRepresentableを使ってUIKitをSwiftUIに統合し、Coordinatorパターンを使ってカメラ機能と連携するアプリを題材にする。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ここに模範コード全体を貼る

// ============================================
// 第3章（基本）：写真を選択・撮影して表示するアプリ
// ============================================
// PhotosPickerを使ってフォトライブラリから写真を選択し、画面に表示します。
// 「カメラ」ボタンで撮影もできます。
//
// 【動作環境】
//   - フォトライブラリから選択：シミュレータでも動作します。
//   - カメラ撮影：実機（iPhone / iPad）専用。シミュレータでは
//     カメラボタンが自動的に無効化されます。
//
// 【注意】実機でカメラを使う場合は Info.plist に以下を追加してください：
//   - NSCameraUsageDescription
//     値: "撮影した写真を表示するためにカメラを使用します"
// ============================================

import SwiftUI
import PhotosUI

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImage: Image?
    @State private var isShowingCamera = false
    @State private var capturedUIImage: UIImage?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                // 画像表示エリア
                imageDisplayArea

                // ボタンエリア
                HStack(spacing: 20) {
                    // フォトライブラリから選択
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    // カメラで撮影（シミュレータには未搭載のため自動的に無効化）
                    Button {
                        isShowingCamera = true
                    } label: {
                        Label("カメラ", systemImage: "camera")
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(!UIImagePickerController.isSourceTypeAvailable(.camera))
                }
                .padding()
            }
            .navigationTitle("写真アプリ")
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    await loadImage(from: newItem)
                }
            }
            .fullScreenCover(isPresented: $isShowingCamera) {
                CameraView(capturedImage: $capturedUIImage)
            }
            .onChange(of: capturedUIImage) { _, newImage in
                if let uiImage = newImage {
                    selectedImage = Image(uiImage: uiImage)
                }
            }
        }
    }

    // MARK: - 画像表示エリア

    @ViewBuilder
    private var imageDisplayArea: some View {
        if let image = selectedImage {
            image
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(maxHeight: 400)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 4)
                .padding()
        } else {
            RoundedRectangle(cornerRadius: 16)
                .fill(.gray.opacity(0.1))
                .frame(height: 300)
                .overlay {
                    VStack(spacing: 8) {
                        Image(systemName: "photo")
                            .font(.system(size: 48))
                            .foregroundStyle(.gray)
                        Text("写真を選択または撮影してください")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                .padding()
        }
    }

    // MARK: - 画像の読み込み

    func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
}

// MARK: - カメラビュー（UIKit連携）

struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

SwiftUIで「写真を選ぶ / カメラで撮る → 画面に表示する」アプリです。

（アプリの動作を自分の言葉で説明する。スクリーンショットを貼ってもよい。）

---
## コードの詳細解説

### PhotosPickerによる写真選択

```swift
// 該当部分のコードを抜粋して貼る
struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImage: Image?
    @State private var isShowingCamera = false
    @State private var capturedUIImage: UIImage?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                // 画像表示エリア
                imageDisplayArea

                // ボタンエリア
                HStack(spacing: 20) {
                    // フォトライブラリから選択
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    // カメラで撮影（シミュレータには未搭載のため自動的に無効化）
                    Button {
                        isShowingCamera = true
                    } label: {
                        Label("カメラ", systemImage: "camera")
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(!UIImagePickerController.isSourceTypeAvailable(.camera))
                }
                .padding()
            }
            .navigationTitle("写真アプリ")
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    await loadImage(from: newItem)
                }
            }
            .fullScreenCover(isPresented: $isShowingCamera) {
                CameraView(capturedImage: $capturedUIImage)
            }
            .onChange(of: capturedUIImage) { _, newImage in
                if let uiImage = newImage {
                    selectedImage = Image(uiImage: uiImage)
                }
            }
        }
    }

    // MARK: - 画像表示エリア

    @ViewBuilder
    private var imageDisplayArea: some View {
        if let image = selectedImage {
            image
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(maxHeight: 400)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 4)
                .padding()
        } else {
            RoundedRectangle(cornerRadius: 16)
                .fill(.gray.opacity(0.1))
                .frame(height: 300)
                .overlay {
                    VStack(spacing: 8) {
                        Image(systemName: "photo")
                            .font(.system(size: 48))
                            .foregroundStyle(.gray)
                        Text("写真を選択または撮影してください")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                .padding()
        }
    }
```

**何をしているか：**
（この部分が果たしている役割を説明する）

「画面を表示して、ユーザーが写真を選んだり撮影したりしたら、その画像を表示する」という処理をしています。

---
**なぜこう書くのか：**
（別の書き方ではなく、この書き方が選ばれている理由を説明する）

1.
```
.onChange(of: selectedItem)
```
onChangeを使うことによって値の変化を検知を行っている

2.なぜ非同期処理を行っている？　アプリがフレーズのを防ぐ

fullScreenCover と isShowingCameraはセットになります。
```
@State private var isShowingCamera = false
```
```
.fullScreenCover(isPresented: $isShowingCamera)
```
これがないとカメラ画面は出ません。

仕組みは：
```
ボタン押す

↓

isShowingCamera = true

↓

fullScreenCoverが反応

↓

CameraView表示
```
---

**もしこう書かなかったら：**
（この部分を省略したり変えたりすると何が起きるか。実際に試した結果があればここに書く）

1.SwiftUIは状態が変わったら画面を再描画する仕組みなので @State がないと画面更新されない。

---
2.onChange がないと写真が選ばれたことを検知できなくて、画像が表示されない。

```
.disabled(!UIImagePickerController.isSourceTypeAvailable(.camera))
```

これを外すとシミュレータでカメラ押したとき問題が出やすい


---

### 画像の非同期読み込み

```swift
// 該当部分のコードを抜粋して貼る

    // MARK: - 画像の読み込み

    func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
}
```

**何をしているか：**

この部分は、「選ばれた写真を実際に読み込んで、画面に表示できる形に変換する」処理です。

---

**なぜこう書くのか：**

1.PhotosPickerItem?が重要

```
func loadImage(from item: PhotosPickerItem?) async
```

? があるので、
```
・写真がある

・写真がない（nil）
```
両方の可能性があります。
---
2.asyncが必要

時間のかかる処理をする関数、写真の読み込みはすぐ終わらないことがあるので、非同期で処理します。

```
try await item.loadTransferable(...)
```
---
3.guard letが必要

```
guard let item = item else { return }
```
item が nil なら処理をやめる、クラッシュを防ぐ

---

**もしこう書かなかったら：**

1.guard letが書かなかったら、処理が始まる前にエラーの発生が防ぐことができなくなる

---
2.PhotosPickerItem?に?を付けないとデータに空のデータがはいてしまいアプリがクラッシュしてしまう可能性がある

---

### UIViewControllerRepresentableによるカメラ連携

```swift
// 該当部分のコードを抜粋して貼る

struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }
}
```

**何をしているか：**

ここは 「SwiftUIでUIKitのカメラ機能を使うための橋渡し」 をしています。

・SwiftUIだけでは直接カメラを開けないので、UIKitのカメラ機能を借りている

・iPhoneの標準カメラを開く機能は主にUIKitのUIImagePickerControllerが担当しています。

---
**なぜこう書くのか：**

1.SwiftUIとUIKitをつなぐ役としてUIViewControllerRepresentableが必要。

流れそして：
```
SwiftUIでCameraViewを表示
      ↓
UIKitのカメラを作る
      ↓
ユーザーが撮影
      ↓
撮影した画像を受け取る
      ↓
ContentViewに渡す
      ↓
画面更新
```
```
struct ○○: UIViewControllerRepresentable
```
ここにおいて「UIKitのViewControllerをSwiftUIに変換します」という意味です。

---
2.dismiss
```
@Environment(\.dismiss) private var dismiss
```
この画面を閉じるための機能

---
3.picker
```
let picker = UIImagePickerController()

picker.sourceType = .camera
```
これは
UIImagePickerControllerを作っています。

これはApple標準のカメラ、写真選択用画面です。そしてpickerをカメラモードにする

---
4.delegate
```
picker.delegate = context.coordinator
```
これは撮影が終わったら誰に知らせるかを設定しています。

delegate は「イベントを受け取る担当」です。かなりUIKitでよく使います。

---
5.makeCoordinator
```
func makeCoordinator() -> Coordinator {
    Coordinator(self)
}
```
これはイベントを受け取る担当を作る処理です。

役割：
```
・撮影完了を受け取る

・キャンセルを受け取る

・画像を渡す

・画面を閉じる
```
---

**もしこう書かなかったら：**

1,SwiftUIとUIKitをつなぐ役としてUIViewControllerRepresentableが必要。

書かなかったらカメラ機能を使えない

---

2.picker作らないとカメラモードに変換できない

---
3.@Bindingを書かないと親画面（ContentView）とデータを共有することができない　→ ContentViewにも反映されない

---

### Coordinatorパターン

```swift
// 該当部分のコードを抜粋して貼る
class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
```

**何をしているか：**

ここは 「カメラで起きたイベントを受け取って処理する係」 です。

「カメラが『撮れたよ』『キャンセルされたよ』と知らせてきたときに対応する受付係」

流れとして：
```
ユーザーがカメラを開く
     ↓
写真を撮る / キャンセルする
     ↓
UIImagePickerController が通知する
     ↓
Coordinator が受け取る
     ↓
ContentView に結果を渡す
     ↓
カメラ画面を閉じる
```

**なぜこう書くのか：**

1.NSObject

これはUIKitでよく必要になる親クラスです。Appleの古い仕組み（Objective-C系）と連携するために必要。

---
2.UINavigationControllerDelegate

これもUIImagePickerControllerが必要とするお約束。

細かくはナビゲーション管理用ですが、実際はUIImagePickerControllerを使うならセットで書くことが多いです。

---
3.func imagePickerController
```
func imagePickerController(
    _ picker: UIImagePickerController,
    didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
)
```
写真の撮影が終わったとき自動で呼ばれる関数

infoは撮影結果の情報が入った辞書です

中には例えば：

・元画像

・編集後画像

・メタデータ

などが入っています。

---
4.info[.originalImage]
```
if let image = info[.originalImage] as? UIImage
```
辞書から撮影した元画像を取り出す

as? UIImage →　「UIImage型として変換できる？」

---
**もしこう書かなかったら：**

1.NSObjectを書かないとdelegateとして使えないことがあります。かなり「お約束」です。
---
2.if let 使わなければ画像がUIImage型として変換できなくてアプリが落ちる

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

### SwiftUI

| 項目 | 説明 | 使用例 |
|------|------|---------|
| `@State` | Viewの状態を管理する | `@State private var selectedImage: Image?` |
| `@Binding` | 親子Viewで値を共有する | `@Binding var capturedImage: UIImage?` |
| `NavigationStack` | ナビゲーション画面を構成する | `NavigationStack { ... }` |
| `PhotosPicker` | フォトライブラリから画像を選択する | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| `.onChange()` | 値の変化を監視する | `.onChange(of: selectedItem) { ... }` |
| `.fullScreenCover()` | 全画面モーダルを表示する | `.fullScreenCover(isPresented: $isShowingCamera)` |
| `@ViewBuilder` | 条件によって異なるViewを返す | `@ViewBuilder private var imageDisplayArea: some View` |
| `Task` | 非同期処理を開始する | `Task { await loadImage(from: newItem) }` |
| `async / await` | 非同期処理を実行・待機する | `func loadImage(...) async` / `try await item.loadTransferable(...)` |
| `guard let` | Optionalを安全に取り出す | `guard let item = item else { return }` |
| `if let` | Optionalの値がある場合のみ処理する | `if let image = selectedImage { ... }` |

---

### UIKit

| 項目 | 説明 | 使用例 |
|------|------|---------|
| `UIViewControllerRepresentable` | UIKitのViewControllerをSwiftUIで利用する | `struct CameraView: UIViewControllerRepresentable` |
| `UIImagePickerController` | カメラを起動する | `let picker = UIImagePickerController()` |
| `Coordinator` | UIKitのイベントを受け取る | `func makeCoordinator() -> Coordinator` |
| `UIImagePickerControllerDelegate` | 撮影完了・キャンセルを受け取る | `class Coordinator: NSObject, UIImagePickerControllerDelegate` |
| `UINavigationControllerDelegate` | `UIImagePickerController`で必要なDelegate | `class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate` |
| `NSObject` | UIKitと連携するための基底クラス | `class Coordinator: NSObject` |

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

### 1. Coordinatorの役割

**質問**

> Coordinatorは何のために必要なのか？

**得られた理解**

- UIKitから送られてくるイベント（撮影完了・キャンセル）を受け取る役割。
- 撮影した画像をSwiftUIへ渡す仲介役になっている。

---

### 2. UIViewControllerRepresentableとは？

**質問**

> なぜUIViewControllerRepresentableを使うのか？

**得られた理解**

- SwiftUIだけではカメラを直接扱えない。
- UIKitのViewControllerをSwiftUIで利用するための橋渡しをしている。

---

### 3. NSObjectとは？

**質問**

> NSObjectは何のために継承するのか？

**得られた理解**

- UIKitのDelegateとして動作するための基底クラス。
- UIKitの仕組みと連携するための土台になっている。

---


## この章のまとめ

SwiftUIだけでなくUIKitとの連携方法も学ぶことができる

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
