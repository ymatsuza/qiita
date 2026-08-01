---
title: Androidタブレットを Mac の拡張ディスプレイにする OSS を fork して、複数台接続とUSB再接続のバグを直した
tags:
  - macOS
  - Android
  - Go
  - adb
  - OSS
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---
手元に検証用の Android 端末が数台あり、そのうちの1台を Mac のサブディスプレイに使いたかった。Sidecar は iPad 専用で、市販の有償アプリは会社の端末に入れるまでの手続きが重い。探していたところ [LukeLogix/android-display](https://github.com/LukeLogix/android-display)（Apache License 2.0）を見つけた。Mac に**本物の仮想ディスプレイ**を生やしてタブレットへ H.264 で飛ばす、という素直な作りで、単体では期待どおり動いた。

ただし実際に使い始めると、自分の使い方（**端末を2台つなぐ / USB ケーブルを抜き差しする**）で壊れた。fork して直したのがこの記事の内容。

- 本家: [LukeLogix/android-display](https://github.com/LukeLogix/android-display)
- fork: [ymatsuza/android-display](https://github.com/ymatsuza/android-display)

この記事の主題は fork 側で加えた4つの修正・1つの機能追加で、ベースの実装は本家のもの。

---

## 何をするツールか

Mac 側が Go + CGo のサーバー、Android 側が Kotlin のクライアント。

```
Mac (Go + CGo Server)                    Android (Kotlin Client)
┌──────────────────────┐    WiFi/UDP    ┌──────────────────────┐
│ CGVirtualDisplay     │───────────────▶│ MediaCodec H.264     │
│ (private API)        │                │ デコード             │
│                      │                │                      │
│ CGDisplayStream      │                │ SurfaceView          │
│ (画面キャプチャ)     │                │ (全画面表示)         │
│                      │                │                      │
│ VideoToolbox H.264   │◀───────────────│ TouchCollector       │
│ (HWエンコード)       │      TCP       │ (指 + S Pen)         │
│                      │                │                      │
│ CGEvent / CGTablet   │                │                      │
│ (入力インジェクト)   │◀──── mDNS ────▶│ NSD Discovery        │
└──────────────────────┘                └──────────────────────┘
```

肝は `CGVirtualDisplay`。macOS の **非公開 API** で、これを叩くと OS からは実在の外部モニタと同じに見えるディスプレイが生える。ミラーリングではなく本物の拡張ディスプレイなので、Mac 側からウィンドウをドラッグして持っていける。

非公開なのでヘッダもリンク先もない。`CoreDisplay.framework` を `NSBundle` で動的ロードし、`NSClassFromString` でクラスを取り、プロパティは KVC で埋める、という書き方になっている。

```objc
// Try public Frameworks first (macOS 14+), then PrivateFrameworks (older)
NSBundle *bundle = [NSBundle bundleWithPath:
    @"/System/Library/Frameworks/CoreDisplay.framework"];
if (!bundle) {
    bundle = [NSBundle bundleWithPath:
        @"/System/Library/PrivateFrameworks/CoreDisplay.framework"];
}
```

映像は WiFi モードなら UDP で分割送信、USB モードなら `adb reverse` 経由の TCP。タッチ・S Pen の筆圧は常に TCP で Mac に返し、`CGEvent` / `CGTablet` でシステムに注入する。

---

## 踏んだ不具合

使い方はシンプルなのに、自分の環境では3つ引っかかった。いずれも**「1台をWiFiで1回つなぐ」という前提から外れた瞬間**に出るもので、単体で試している限り絶対に踏まない類のバグだった。

1. 2台目を接続した瞬間、サーバーが SIGSEGV で落ちる
2. USB で2台つなぐと `adb reverse` が失敗し、接続できない
3. USB ケーブルを抜き差しすると、以降ずっと Connection Failed

順に見ていく。

---

## 1. 2台目の仮想ディスプレイで SIGSEGV

Pixel 9 Pro を WiFi でつないだ状態で Moto G66j 5G をつなぐと、サーバープロセスが `CreateVirtualDisplay` の中で落ちる。Go 側のスタックトレースには何も残らない（CGo の向こう側で死んでいる）。

原因は仮想ディスプレイの**識別子が全インスタンスで同じ**だったこと。descriptor のプロパティがこう固定されていた。

```objc
[descriptor setValue:@(0x1234) forKey:@"vendorID"];
[descriptor setValue:@(0x5678) forKey:@"productID"];
[descriptor setValue:@"AndroidMac Virtual Display" forKey:@"name"];
[descriptor setValue:@(0) forKey:@"serialNum"];
```

1台目が生きたまま2台目を作ると、CoreDisplay からは**同一デバイスがもう一度登録された**ように見えるらしく、そこでプロセスごと死ぬ。インスタンスごとに一意な `serial` を配り、identity 系のフィールドを派生させて解決した。

```objc
[descriptor setValue:@(0x1234) forKey:@"vendorID"];
[descriptor setValue:@(0x5678 + config.serial) forKey:@"productID"];
[descriptor setValue:[NSString stringWithFormat:@"AndroidMac Virtual Display %d", config.serial]
              forKey:@"name"];
// serialNum は存在しないバージョンがあるので @try で包む
[descriptor setValue:@(config.serial) forKey:@"serialNum"];
```

Go 側は `atomic.Int32` で払い出すだけ。

```go
// nextSerial hands out a unique per-instance identifier so each virtual
// display gets distinct descriptor identity fields (see display_bridge.m).
var nextSerial atomic.Int32
```

**これだけでは直らなかった**のがもう一段の落とし穴で、identity を分けてもなお同時呼び出しで落ちることがあった。1台目が完全に安定してから4秒後に2台目を作っても再現したので、起動時レースではなく `CGVirtualDisplay` の Create / Destroy そのものが並行呼び出しに耐えない、と判断して mutex で直列化した。

```go
// createMu serializes all calls into the CGVirtualDisplay private API.
// The framework crashed with SIGSEGV on a second concurrent
// CreateVirtualDisplay call even 4s after the first display had fully
// stabilized, so this is not just a startup race — Create/Destroy calls
// must never overlap.
var createMu sync.Mutex
```

ついでにフレームワークのロード自体も `static BOOL sFrameworkLoaded` フラグを見て `[bundle load]` する非アトミックな実装だったので、`dispatch_once` に置き換えた。

実機2台（Pixel 9 Pro + Moto G66j 5G, WiFi モード）で再検証し、クラッシュ再発なし・両機がそれぞれ別の仮想ディスプレイを持って pipeline active まで到達することを確認した。

---

## 2. USB で2台つなぐと `adb reverse` が失敗する

USB モードは `adb reverse tcp:PORT tcp:PORT` で「Android 側の localhost:PORT → Mac の localhost:PORT」を張り、アプリは常に `127.0.0.1` を見るだけ、という構成になっている。これが2台つないだ途端に動かなくなる。理由は2つあった。

### 2-1. `-s <serial>` がない

すべての adb 呼び出しがシリアル指定なしだった。デバイスが2台以上見えている状態では adb は当然

```
error: more than one device/emulator
```

を返す。全呼び出しに `-s <serial>` を付け、対象デバイスを列挙して回すよう変更した。

```go
func (m *Manager) SetupReverse(serial string, port int) error {
	portStr := fmt.Sprintf("tcp:%d", port)
	cmd := exec.Command(m.adbPath, "-s", serial, "reverse", portStr, portStr)
	...
}
```

### 2-2. touch / video ポートがグローバル固定だった

元の実装は起動時に touch サーバーと TCP video サーバーを**1組だけ**立て、そのポートを全クライアントに配っていた。1台なら問題ないが、2台目のクライアントが同じポートに繋ぐと映像とタッチが混線する。

ポートを**クライアントごとに動的割り当て**へ変更した。コントロールサーバーに割り当てフックを渡し、ハンドシェイク中（`ServerHello` を返す前）にそのクライアント専用の touch サーバー（USB なら video サーバーも）を立てて、実際に割り当たったポート番号を `ServerHello` に載せて返す。

```go
ctrlServer.SetPortAllocator(allocateClientPorts)
```

USB クライアントの場合、そのポートに対して `adb reverse` を張る必要がある。ここには実装上の妥協が1つ入っていて、**コントロール接続からデバイスのシリアルを特定する手段がない**（TCP の向こう側は `127.0.0.1` なので、どの USB デバイス経由で来たか分からない）。そのため接続中の全シリアルに対して同じ forward を張っている。各クライアントは自分の `ServerHello` で受け取ったポートしか叩かないので実害はないが、綺麗ではない。

切断時は pipeline の `ctx.Done()` を待つ goroutine で touch / video サーバーを閉じ、張った reverse forward を `--remove` で個別に外す。`--remove-all` はそのデバイス上の全ルールを消すので、まだ生きているコントロールポートの forward（と、同じデバイスに他クライアントが張っている分）まで巻き添えになる。ここは個別削除でなければならない。

実機2台（USB モード）で `adb reverse --list` の内容が期待どおり一致し、両機が別々の仮想ディスプレイで pipeline active に到達することを確認した。

---

## 3. USB を抜き差しすると以降ずっと Connection Failed

これが一番厄介だった。一度接続を切ってケーブルを挿し直すと、アプリが Connection Failed になり、**サーバーを再起動するまで直らない**。

原因は `adb reverse` の寿命。reverse forward はデバイス側の adbd が保持しているので、**USB リンクが切れた時点で消える**。元の実装はサーバー起動時に一度張るだけだったため、セッション中に挿し直したデバイスはコントロールポートの forward を失ったままになる。

### 3-1. デバイス出現のポーリング

`adb devices` を2秒間隔でポーリングし、新しく現れたシリアルに対して `SetupReverse` をやり直す Watcher を追加した。

```go
watcher := adb.NewWatcher(adbManager.ListDeviceSerials, serials, func(serial string) {
	log.Printf("ADB device attached: %s — setting up reverse forwarding", serial)
	if err := adbManager.RemoveAllReverse(serial); err != nil { ... }
	if err := adbManager.SetupReverse(serial, controlPort); err != nil { ... }
})
go watcher.Run(2 * time.Second)
```

ここで1つ注意したのは **adb の一時的なエラーで known セットを壊さないこと**。`adb devices` がエラーを返したときに一覧を空として扱うと、次のポーリングで全デバイスが「新規」に見え、切断していない端末の forward まで張り直してしまう。エラー時は known を触らずに return する。

```go
func (w *Watcher) poll() {
	serials, err := w.list()
	if err != nil {
		return // 既知セットを保持する
	}
	...
}
```

### 3-2. ポーリング間隔に開く穴

これで大半は直ったが、**抜き差しが2秒のポーリング1回分の中で完結すると再現した**。デバイス一覧の差分には現れないので `onNew` が発火しない。しかし adbd 側の forward はリンク断で確かに消えている。つまり「一覧上は何も起きていないのに forward だけ死んでいる」状態になる。

差分検知では原理的に拾えないので、**既知デバイスに対して毎ポーリング実際の状態を検証する**フックを足した。

```go
// A disconnect→reconnect completing within one poll interval never
// changes the device listing, so the diff above misses it. Verify the
// control-port forward every cycle and re-add it if the link drop took
// it away. Additive only: per-client touch/video forwards are set up
// during the connect handshake and must not be cleared here.
watcher.EnsureKnown(func(serial string) {
	ok, err := adbManager.HasReverse(serial, controlPort)
	if err != nil || ok {
		return
	}
	log.Printf("ADB reverse control port missing on %s — re-establishing", serial)
	if err := adbManager.SetupReverse(serial, controlPort); err != nil { ... }
})
```

**追加のみ**にしてあるのが重要な点。クライアント接続時に張られる touch / video の forward はここでは検証も削除もしない。ここで `--remove-all` → 張り直しをやると、接続中のクライアントの映像が落ちる。

### 3-3. `adb reverse --list` のパースで踏んだ罠

`HasReverse` の実装で一度間違えた。出力はこういう形式をしている。

```
UsbFfs tcp:9000 tcp:9000
```

第1フィールドを**シリアルだと思い込んで**マッチさせようとしたが、これは transport 名（`UsbFfs` など）でシリアルではない。`adb -s <serial>` を付けている以上そもそも出力は対象デバイスに絞られているので、見るべきは第2フィールドのリッスン仕様だけだった。

```go
// reverseListHas parses `adb reverse --list` output. Each line looks like
// "UsbFfs tcp:9000 tcp:9000" — the first field is the transport name (not
// the serial), the second is the device-side listen spec.
func reverseListHas(out string, port int) bool {
	portStr := fmt.Sprintf("tcp:%d", port)
	for _, line := range strings.Split(out, "\n") {
		fields := strings.Fields(line)
		if len(fields) >= 3 && fields[1] == portStr {
			return true
		}
	}
	return false
}
```

Watcher と adb パーサはどちらも外部コマンドを関数で差し替えられる形にしてあるので、ユニットテストで挙動を固定できる。3-2 の「間隔内の抜き差し」も実機なしでテストに落ちている。

---

## 追加した機能：画面の向き選択

タブレットを縦置きにしたりスタンドで180度回して置いたりしたかったので、接続前の画面に**縦 / 横のラジオボタン**と**上下反転（180°）のチェックボックス**を足した。

<img src="https://raw.githubusercontent.com/ymatsuza/qiita/main/images/android-display-orientation.png" width="600">

接続前画面。上段の WiFi / USB と下の解像度・画質は本家のもので、真ん中の「横向き / 縦向き」と「上下反転（180°）」が今回足した部分。

Android 側は `requestedOrientation` にマップするだけ。

```kotlin
val isPortraitOrientation = intent.getStringExtra("orientation") == "portrait"
val isReversed = intent.getBooleanExtra("reverse", false)
requestedOrientation = when {
    isPortraitOrientation && isReversed -> ActivityInfo.SCREEN_ORIENTATION_REVERSE_PORTRAIT
    isPortraitOrientation               -> ActivityInfo.SCREEN_ORIENTATION_PORTRAIT
    isReversed                          -> ActivityInfo.SCREEN_ORIENTATION_REVERSE_LANDSCAPE
    else                                -> ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE
}
```

面倒だったのは**サーバーに要求する解像度**のほう。元は `metrics.widthPixels` / `heightPixels` をそのまま送っていたが、これは端末の現在の向きに依存する値なので、縦向きを選んでも横長の仮想ディスプレイが生えてしまう。長辺・短辺に正規化してから、選択した向きに応じて割り当てるようにした。

```kotlin
val rawLongSide  = maxOf(metrics.widthPixels, metrics.heightPixels) * scale
val rawShortSide = minOf(metrics.widthPixels, metrics.heightPixels) * scale
val longSide  = rawLongSide.toInt() and -2  // round down to even
val shortSide = rawShortSide.toInt() and -2
val scaledWidth  = if (isPortrait) shortSide else longSide
val scaledHeight = if (isPortrait) longSide  else shortSide
```

`and -2` で偶数に切り捨てているのは H.264 の要件（幅・高さが偶数でないとエンコーダが受け付けない）。

このとき副次的に2つ直した。

- UI 項目が増えて Connect ボタンが画面外に押し出されたので、`activity_main.xml` を `ScrollView` でラップ
- Android 14 でフォアグラウンドサービス起動時にクラッシュしていたので `CHANGE_WIFI_STATE` 権限を追加

---

## 非公開 API に乗るということ

このプロジェクトの中核である `CGVirtualDisplay` は Apple の非公開 API で、以下はすべてリスクとして受け入れる前提になる。

- **macOS のアップデートで動かなくなりうる**。実際 framework の置き場所は macOS 14 で `PrivateFrameworks` から `Frameworks` に移っており、コードは両方を順に試している
- **KVC で設定しているプロパティ名が変わりうる**。`serialNum` は存在しないバージョンがあるため `@try` / `@catch` で包まれている
- **クラッシュがプロセス全体を殺す**。今回の SIGSEGV のように、Go 側のエラーハンドリングでは何も守れない
- **App Store 配布はできない**

自分用のツールとして割り切って使う分には十分実用的だが、業務端末で常用する何かの土台にはしないほうがいい、というのが触ってみた結論。

## テストの状態

Go 側は `go test ./...` が全パスする。

```
ok  	.../internal/adb
ok  	.../internal/control
ok  	.../internal/discovery
ok  	.../internal/input
ok  	.../internal/stream
ok  	.../internal/touch
```

ただし `display` / `capture` / `encoder` の3パッケージは CGo で macOS の API を直接叩いており、テストが1本もない（`[no test files]`）。今回一番危なかった SIGSEGV がまさにこの範囲で、実機2台をつないで手で再現させるしか検証手段がなかった。CGo 境界の向こう側は、テストで守るのが本当に難しい。

---

## まとめ

- 元の実装（[LukeLogix/android-display](https://github.com/LukeLogix/android-display)）は1台をWiFiで使う分には完成度が高い
- 自分の使い方（2台同時 / USB 抜き差し）で3つ壊れたので fork して直した
  - `CGVirtualDisplay` の descriptor identity 重複と、Create/Destroy の並行呼び出しによる SIGSEGV
  - `adb -s` 抜けとグローバル固定ポートによる複数台 USB 接続の失敗
  - USB リンク断で消える reverse forward の再確立（差分検知＋毎周期の実状態検証）
- 副産物として、画面の向き選択と Android 14 対応

fork: [ymatsuza/android-display](https://github.com/ymatsuza/android-display)
