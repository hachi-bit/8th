# 00. 共通仕様 — 8th Wall AR検証プロジェクト

このドキュメントは、以下の3つのデモアプリすべてに共通するルール・技術方針をまとめたものです。
Claude Codeは各デモの実装前に必ずこのファイルを読んでください。

- `01-floor-placement.md` — 案1: マーカーレス床置きオブジェクト配置デモ
- `02-ar-ruler.md` — 案2: ARバーチャル巻尺デモ
- `03-proposal-tower-ar.md` — 案3: 提案評価タワーのAR実寸配置デモ

## 1. プロジェクトの目的

8th Wall(2026年2月にオープンソース化されたAR/XRエンジン)が、既存のAR.js(マーカーベース)+ WebXR hit-test(Androidのみ)構成の代替になり得るか検証する。特に **iOS Safariでマーカーレスの環境認識ARが動くか** が最大の検証ポイント。

検証済み事実(2026年7月時点、公式ソースで確認済み):
- 8th Wallは2026年2月28日に有料ホスティングプラットフォームを終了し、`8thwall.org` で無償のオープンソースツールセットとして提供されている。
- **MITライセンスで完全オープン**: Image Targets, Face Effects, Sky Effects, 各種ユーティリティ
- **バイナリ限定ライセンス(ソース非公開・改変不可)**: World Effects(SLAM/環境認識), Absolute Scale。ただし無償・商用利用は可。リバースエンジニアリング・改変・再配布は禁止。
- アカウント登録・appKey取得は不要(旧プラットフォーム時代の名残の情報が検索に混ざるので注意)。
- VPS・Lightship Maps・Hand Trackingは同梱されておらず、2027年2月28日以降は完全に利用不可になる予定。

## 2. 制約条件(ハード制約、必ず守ること)

- **登録不要**: ユーザーがアカウント作成やAPIキー取得をせずに動作すること。
- **無償のみ**: 有償サービス・有償ライブラリを追加で組み込まない。
- **GitHub Pages対応**: 静的ファイルのみで完結すること。サーバーサイド処理(Node.jsサーバーなど)を必須にしない。
- **単一HTMLファイル推奨**: 可能な限り1ファイルで完結させる(CSS/JSはインライン)。ビルドツール(webpack等)を必須にしない。
- **HTTPS必須**: カメラアクセスにはHTTPSが必要。GitHub Pagesはデフォルトでhttps配信なのでここは自動的に満たされる。

## 3. 技術スタックの固定方針

| 項目 | 選定 | 理由 |
|---|---|---|
| 3Dレンダリング | Three.js r128 | 既存プロジェクト(PLATEAU洪水シミュレーション等)との統一。**r128固定**。`THREE.CapsuleGeometry`はr142以降の機能なので使用禁止。代わりに`CylinderGeometry`, `SphereGeometry`等の組み合わせで代替すること。 |
| Three.js CDN | `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js` | 既存プロジェクトで実績あり。unpkg.comはCSPの問題で不可、jsdelivr ESM形式は3Dモジュール系との相性を要検証のため、まずはcdnjsのUMD版を使う。 |
| ARエンジン | 8th Wall Engine Binary | `https://cdn.jsdelivr.net/npm/@8thwall/engine-binary@1/dist/xr.js` を `async crossorigin="anonymous" data-preload-chunks="slam"` 属性付きで読み込む。**appKey等のクエリパラメータは付与しない**(新方式では不要)。 |
| ARエンジン連携 | XR8 JS API(Three.jsパイプライン) | A-Frameは使わず、素のXR8 JS APIを使用する。既存プロジェクトがThree.jsベースであるため統一する。 |

### 3.1 8th Wallエンジンの読み込みテンプレート(全デモ共通、変更しないこと)

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@8thwall/engine-binary@1/dist/xr.js" async crossorigin="anonymous" data-preload-chunks="slam"></script>
```

three.jsは必ずエンジンバイナリより先に読み込むこと(`XR8.Threejs.pipelineModule()`がTHREE本体を必要とするため)。

### 3.2 エンジン初期化テンプレート(全デモ共通の起動シーケンス)

```javascript
const onxrloaded = () => {
  XR8.addCameraPipelineModules([
    XR8.GlTextureRenderer.pipelineModule(),   // カメラ映像描画
    XR8.Threejs.pipelineModule(),             // three.jsシーン管理
    XR8.XrController.pipelineModule(),        // World Tracking(SLAM)有効化
    sceneModule,                              // 各デモ固有のシーンロジック(下記参照)
  ])
  XR8.run({ canvas: document.getElementById('camerafeed') })
}

window.addEventListener('load', () => {
  if (window.XR8) {
    onxrloaded()
  } else {
    window.addEventListener('xrloaded', onxrloaded)
  }
})
```

この `window.XR8 ? 即実行 : 'xrloaded'イベント待ち` というパターンは8th Wall公式リポジトリのxrextrasパッケージのREADMEに記載された標準パターンであり、変更しないこと。

### 3.3 シーンモジュールの型

各デモの中心ロジックは以下の形の`sceneModule`オブジェクトとして実装する。

```javascript
const sceneModule = {
  name: 'アプリ固有の名前',
  onStart: ({ canvas }) => {
    const { scene, camera } = XR8.Threejs.xrScene()
    // カメラYを非ゼロにするのが公式Tips(スケール基準の安定化に寄与)
    camera.position.set(0, 1.4, 0)
    // ここにライティング・イベントリスナー登録などを書く
  },
}
```

### 3.4 タップ位置から実世界座標を取得する共通関数

床・平面検出によるタップ配置が必要な全デモで、以下の関数を共通利用すること。

```javascript
function hitTestAtScreenPoint(clientX, clientY) {
  const x = clientX / window.innerWidth
  const y = clientY / window.innerHeight
  const results = XR8.XrController.hitTest(x, y, ['FEATURE_POINT'])
  if (!results || results.length === 0) return null
  return results[0].position // {x, y, z} 実世界座標(メートル単位)
}
```

`touchstart`と`click`の両方にリスナーを張り、`touchstart`は`e.touches.length === 1`の場合のみ処理すること(マルチタッチのジェスチャー操作と衝突しないようにするため)。

## 4. リポジトリ構成とファイル命名ルール

リポジトリ名は `8th`。GitHub Pagesでの公開時、各デモは以下のフォルダ構造にすること。

```
8th/
├── index.html          ← ランディングページ(3デモへのリンク一覧のみ。凝ったデザイン不要)
├── specs/               ← 本ドキュメント一式(00〜03のmdファイル)を格納。Pages公開対象ではない
│   ├── 00-common-setup.md
│   ├── 01-floor-placement.md
│   ├── 02-ar-ruler.md
│   └── 03-proposal-tower-ar.md
├── 1/
│   └── index.html      ← 案1: 床置きオブジェクト配置デモの実体
├── 2/
│   └── index.html      ← 案2: ARバーチャル巻尺デモの実体
└── 3/
    └── index.html      ← 案3: 提案評価タワーAR配置デモの実体
```

- 各デモの実装ファイルは、開発中は`01-floor-placement.md`等の仕様書と対応が分かりやすいよう`1-floor-placement.html`のような名前で作業してよいが、**リポジトリへの最終配置時には必ず `N/index.html` の形に変換すること**(GitHub PagesはURLのデフォルト表示に`index.html`という名前を要求するため)。
- ルート直下の`index.html`は以下のような最小限のリンク一覧でよい:

```html
<a href="./1/">案1: 床置き配置デモ</a>
<a href="./2/">ARバーチャル巻尺</a>
<a href="./3/">提案評価タワーAR</a>
```

- GitHub Pagesの設定は `gh api -X PUT "/repos/OWNER/8th/pages" -f "source[branch]=main" -f "source[path]=/"` のように、`gh api`経由でブランチ`main`のルートをソースとして有効化する(`gh`コマンドにPages専用のサブコマンドは存在しないため、直接API呼び出しが必要)。

## 5. UI/UXの共通ルール

- 画面上部に**常時表示のヒント文**(何をすればいいかの1行ガイド)を置くこと。
- 画面下部に**ステータステキスト**(検出中/配置済み/エラー等の状態)を置くこと。
- HTTPSでないアクセス時、および`window.XR8`が一定時間経っても読み込まれない場合は、全画面オーバーレイでエラーメッセージを表示すること(カメラが真っ黒のまま無反応になる状態を避ける)。
- フォントは`sans-serif`、テキストには`text-shadow`を付けてカメラ映像の上でも読みやすくすること。
- ボタン類は画面右下に固定配置し、`rgba(0,0,0,.55)`程度の半透明背景+白文字で統一すること(既存プロトタイプのスタイルに合わせる)。

## 6. 未検証事項(実装前に確認、または実装後に実機検証が必要な項目)

Claude Codeは以下の項目について、断定的な実装をする前に一度立ち止まって確認すること。不明な場合はコメントで「要検証」と明記し、動作しなかった場合の代替手段(フォールバック)も併記すること。

1. `@8thwall/engine-binary@1` のバージョン固定(`@1`)が最新かどうか。npmで実際に存在するか確認すること。
2. `XR8.XrController.hitTest()` の戻り値の座標系(ワールド座標かカメラ相対座標か)。実機ログで`console.log`して確認すること。
3. iOS Safariで`XR8.XrController.recenter()`の挙動がAndroidと同一か。
4. 複数タブ・複数セッションでカメラリソースが正しく解放されるか(SPA的にページ遷移する場合)。

## 7. 参照済みプロトタイプ

以下の3ファイルは、上記方針に基づいて過去に一度組んだ動作確認用プロトタイプである(実機でAndroid/iPhone双方で起動確認済み)。実装時の参考にしてよいが、コピーではなく本仕様書の要求に沿って作り直すこと。

- `1-floor-placement.html`
- `2-ar-ruler.html`
- `3-proposal-tower-ar.html`

## 8. 受け入れ基準(共通・全デモ共通)

- [ ] GitHub Pagesにpushしただけでビルド不要で動作する
- [ ] HTTPS経由でアクセスした際、カメラ許可ダイアログが表示される
- [ ] 平面/特徴点が検出されるまでの間、ユーザーに分かる形でガイドが表示される
- [ ] Android Chrome / iOS Safari 双方で起動しクラッシュしない
- [ ] コンソールに`XR8 is not defined`等の未定義エラーが出ない
- [ ] `4. リポジトリ構成とファイル命名ルール`の通りのフォルダ構造(`1/index.html`等)になっている
- [ ] ルート直下の`index.html`から3デモすべてにリンクが張られており、クリックで正しく遷移する
- [ ] GitHub Pagesが有効化され、`https://<user>.github.io/8th/` でランディングページが表示される
