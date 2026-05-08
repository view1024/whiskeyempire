# Whisky Empire 仕様書 v1.0

LINE Mini App（LIFF）として動作する単一HTML完結型の放置クリッカー＋経営シミュレーションゲーム。本書は本作の完成版仕様を網羅し、同種の経営ゲームを企画・実装する際の参照資料として利用できる構成にしています。

---

## 目次

1. プロダクト概要
2. ゲーム設計
3. システムアーキテクチャ
4. データ構造
5. 主要メカニクスと計算式
6. 経済バランス設計
7. UI/UX設計
8. 永続化とライフサイクル
9. LIFF（LINE Mini App）統合
10. 拡張性の指針
11. デプロイ・運用
12. 用語集

---

## 1. プロダクト概要

### 1.1 ジャンルとコンセプト

放置クリッカー（Idle Clicker）に経営シミュレーション要素を融合した、ブラウザ完結型の単一HTMLゲーム。プレイヤーはウィスキー蒸留事業を「自宅キット」から始めて「多国籍コングロマリット」まで成長させ、25種類のウィスキー銘柄をすべて図鑑に解禁することを目指す。

### 1.2 想定プレイ時間

フルコンプリートまで約1か月（1日2〜4時間のアクティブプレイ＋オフライン補正最大8時間/日）。中盤以降は施設拡張コストとTier解禁コストが指数関数的に増加するように設計されており、序盤は1〜2時間で1ティアを抜けるテンポ感、終盤は数日かけて1ティアを抜けるペース感に調整されている。

### 1.3 想定プラットフォーム

第一義はLINE Mini App（LIFF）。LIFF SDKが利用できない環境（PCブラウザ直接アクセス・Claude等のサンドボックス起動）では自動的にゲストモードへフォールバックして動作する。

### 1.4 コアループ

1. タップで現金獲得（タップ単価は名声Lvで増加）
2. 現金で製造施設を購入＆レベルアップ → 受動収益が増加
3. 全6施設をLv10まで拡張すると蒸留所アップグレード解禁
4. アップグレードで施設リセット（ただし大幅な収益倍率を獲得）
5. 蒸留所Tierが上がるごとに同時製造可能銘柄数が増加（Tier=並列ライン数）
6. ウィスキー図鑑から銘柄を製造販売、累計売上と銘柄別販売数で次の銘柄が解禁
7. 全25銘柄解禁＋Tier5到達でフルコンプリート

---

## 2. ゲーム設計

### 2.1 ゲーム要素サマリー

| カテゴリ | 数量 | 概要 |
|---|---|---|
| 蒸留所Tier | 5段階 | ホーム/クラフト/地方/大手/コングロマリット |
| 製造施設 | 6種類 | 製麦/糖化/発酵/蒸留/熟成/ボトリング |
| ウィスキー銘柄 | 25種類 | スコッチ12/バーボン4/ジャパニーズ5/アイリッシュ2/カナディアン1/特殊1 |
| 称号 | 16種類 | 累計タップ/売上/解禁数/Tier到達/並列製造数 |
| 施設レベル | 各10段階 | 各レベルに固有の「導入設備」テキスト |
| 自動製造スロット | Tier番号と同数 | Tier1=1、Tier5=5 |

### 2.2 進行フロー

序盤（Tier1）: タップ中心、最初の数施設を購入・低レベル拡張で資金蓄積。基本的に総売上 $5K〜$1M 程度で完走。

中盤（Tier2–Tier3）: 受動収益が主軸に。施設レベルの上げ方とアップグレード判断がプレイヤーの戦略。クラフト蒸留所到達でブレンダー研究室解禁、地方蒸留所到達で地域限定銘柄ライン解禁。

終盤（Tier4–Tier5）: 自動製造を3〜5銘柄並列で稼働させて指数的に資金を伸ばす。大手蒸留所で欧米プレミアム流通網、コングロマリットで全世界販売網と限定ヴィンテージが解禁される。

### 2.3 解禁チェーン設計

ウィスキー解禁条件は単純な総売上閾値だけではなく「他銘柄を◯本販売」を組み合わせている。これにより総売上だけが大きい状態では先に進めず、各銘柄を一定数製造販売するプロセスを通過する必要がある。例えば Glenfiddock は「総売上 $100K + Glenlivid を20本販売」で解禁、Pappy Van Rinkle（シークレット）は「Tier4 + 総売上 $5B + Buffalo Tracks を200本販売」で解禁される。

### 2.4 シークレット枠

最後尾3銘柄（Hibique / Pappy Van Rinkle / Imperial Reserve）は解禁条件を非開示とし、🌑🥃 アイコンと「★ シークレット銘柄 ★」表示で意図的に情報を隠す。Imperial Reserve は他24銘柄の全解禁＋Tier5到達がトリガーで、コンプリート報酬として位置付けられている。

---

## 3. システムアーキテクチャ

### 3.1 ファイル構成

```
whisky_empire.html  (単一ファイル、約2500行)
├── <head>          メタ情報・LIFF設定・スタイル
├── <body>
│   ├── #ld         ローディング画面
│   ├── #app        メインUI
│   │   ├── #hdr    ヘッダー（ロゴ・残高）
│   │   ├── #content  4タブの切替コンテナ
│   │   │   ├── #td   蒸留タブ（タップ画面）
│   │   │   ├── #tb   事業タブ（施設管理）
│   │   │   ├── #te   図鑑タブ（銘柄一覧）
│   │   │   └── #tp   プロフィールタブ
│   │   └── #nav    下部ナビゲーション
│   ├── #mo         汎用モーダル
│   ├── #confirmDlg 確認ダイアログ
│   └── #toast      トースト通知
└── <script>
    ├── BALANCE     経済パラメータ
    ├── DL/FD/WL/TT/UNLOCK_REQ/FAC_UPG  ゲームデータ
    ├── G           実行時状態
    ├── 計算関数群   (cv/cuc/fi/ti/wp/dm/tierBonus 等)
    ├── 描画関数群   (rdist/rbus/renc/rprof/rall 等)
    ├── ライフサイクル (init/load/save/offline/markActive)
    └── イベントハンドラ
```

### 3.2 タイマー設計（全4本）

| タイマー | 周期 | 役割 | hidden時 |
|---|---|---|---|
| TICK | 250ms | 受動収益累積、G.ls更新、ギャップ検知 | スキップ |
| UI_REFRESH | 2000ms | 表示更新、ボタン活性、解禁検知 | スキップ |
| AUTO_SELL | 500ms | 自動製造ループ | スキップ |
| SAVE | 30000ms | localStorage永続化（G.ls触らない） | 動作可 |

`document.hidden` ガードによりバックグラウンド時の経済処理を停止し、復帰時に offline() でまとめて補正する設計。

### 3.3 イベントハンドラ

| イベント | 処理 |
|---|---|
| DOMContentLoaded | init() |
| visibilitychange(hidden) | markActive() → save() |
| visibilitychange(visible) | offline() → chkUnlocks() → rall() |
| pagehide / beforeunload | markActive() → save() |
| pageshow (BFCache復帰) | offline() → chkUnlocks() → rall() |
| window.blur | markActive() → save() |
| window.focus | offline() |
| TICK gap > 5000ms | offline() を強制呼び出し |

複数のフォールバックを多層に並べることで、LIFF/iOS/Android のどの閉じ方をしても最低1経路でオフライン補正がかかる構成になっている。

---

## 4. データ構造

### 4.1 実行時状態 G

```javascript
G = {
  cash:    Number,     // 現金残高
  te:      Number,     // 累積総売上（タップ・受動・craft全てgrossで加算）
  tt:      Number,     // タップ累計回数
  dl:      Number,     // 蒸留所Tier (1..DL.length)
  cl:      Number,     // タップ強化レベル (0..MAX_LV)
  fac:     Object,     // 施設状態 {[id]: {l: 0..MAX_LV, p: bool}}
  uw:      Array,      // 解禁済みウィスキーIDリスト
  cw:      Object,     // 銘柄ごとの累計販売本数 {[id]: count}
  am:      Object,     // 自動製造ON銘柄 {[id]: bool}
  inv:     Object,     // 在庫 {[id]: count}（製造済み未販売）
  tl:      Array,      // 獲得済み称号IDリスト
  ls:      Number,     // 最終能動時刻 (ms)
  ver:     Number,     // セーブスキーマバージョン
  dbgMul:  Number      // 検証モード倍率（1=通常、10000=デバッグ）
}
```

### 4.2 主要定数

#### DL（蒸留所Tier定義）

```javascript
{
  name: String,        // 表示名
  mult: Number,        // 収益倍率（dm()）
  unlock: Number,      // 次Tier解禁コスト
  region: String,      // 販売地域名
  slots: Number,       // 同時製造可能銘柄数
  bonus: Number,       // tierBonus（cv/fiに乗る追加倍率）
  desc: String         // Tier解説（拡張特典含む）
}
```

各エントリの数値:
- Tier1 ホームキット: mult=1, unlock=0, slots=1, bonus=1.0
- Tier2 クラフト蒸留所: mult=50, unlock=$500M, slots=2, bonus=1.1
- Tier3 地方蒸留所: mult=2500, unlock=$50B, slots=3, bonus=1.25
- Tier4 大手蒸留所: mult=125000, unlock=$5T, slots=4, bonus=1.5
- Tier5 コングロマリット: mult=6250000, unlock=$500T, slots=5, bonus=2.0

#### FD（製造施設定義）

```javascript
{
  id: String,          // 内部ID
  name: String,        // 表示名
  icon: String,        // 絵文字
  bi: Number,          // base income/h（Lv1相当時の収益）
  bc: Number,          // base upgrade cost（Lv0→1のコスト基準）
  pur: Number,         // 初期購入コスト（malting=0）
  uc: Object|null,     // 解禁条件 {f: 親施設ID, l: 親の必要Lv}
  ul: String           // 解禁条件の表示テキスト
}
```

施設は前段の Lv3 で次が解禁される線形チェーン: 製麦 → 糖化 → 発酵 → 蒸留 → 熟成 → ボトリング。

#### WL（ウィスキー銘柄定義）

```javascript
{
  id: String,
  name: String,
  rg: String,          // 産地（regional）
  tp: String,          // タイプ（スコッチ等）
  yr: Number,          // 熟成年数
  ck: Number,          // 樽ボーナス係数
  bs: Number,          // base sell price
  st: 1|2|3,           // 星評価
  em: String,          // 色絵文字
  fl: Object,          // フレーバープロファイル {フルーティ:0..100, ...}
  ds: String           // 解説
}
```

#### TT（称号定義）

```javascript
{
  id: String,
  lb: String,          // ラベル
  ch: Function         // 達成判定 (state)=>bool
}
```

#### UNLOCK_REQ（銘柄解禁条件）

```javascript
{
  id: String,          // 対応銘柄ID
  req: Function,       // 解禁判定 ()=>bool
  label: String,       // 表示用テキスト
  secret: bool         // シークレット枠（条件非開示）
}
```

#### FAC_UPG（施設レベルアップ詳細）

```javascript
{
  [facilityId]: [
    {eq: String, desc: String},  // Lv1の設備と説明
    ... 計10エントリ ...
  ]
}
```

### 4.3 BALANCE（経済パラメータ）

```javascript
BALANCE = {
  TAP_BASE: 2,                       // タップ基本単価
  TAP_MUL: 1.5,                      // タップ強化倍率
  TAP_COST_BASE: 50,                 // タップ強化Lv0→1のコスト
  TAP_COST_MUL: 3,                   // タップ強化コスト倍率
  FAC_INC_MUL: 2,                    // 施設1Lvあたり収益倍率
  FAC_COST_MUL: 3.5,                 // 施設1Lvあたりコスト倍率
  CRAFT_COST_RATIO: 0.3,             // 製造原価率（30%）
  YEAR_EXP: 0.6,                     // 熟成年数の販売価格指数
  OFFLINE_CAP_H: 8,                  // オフライン補正上限時間
  TICK_MS: 250,                      // tick間隔
  SAVE_INTERVAL_MS: 30000,
  UI_REFRESH_MS: 2000,
  MAX_LV: 10,                        // 各種レベル上限
  SCHEMA_VERSION: 5,
  STORAGE_PREFIX: 'we_v4:',
  AM_BASE_INTERVAL_MS: 30000,        // 自動製造Lv1時の間隔
  AM_AUTO_SELL_THRESHOLD: 10,        // 在庫到達で自動販売
  TICK_GAP_DETECT_MS: 5000           // ギャップ検知の閾値
}
```

---

## 5. 主要メカニクスと計算式

### 5.1 タップ単価

```
cv = TAP_BASE × dm() × TAP_MUL^cl × tierBonus()
```

- TAP_BASE = 2
- dm() = DL[dl-1].mult（Tier1=1, Tier5=6250000）
- cl: 名声レベル (0..10)
- tierBonus(): Tierボーナス倍率（1.0..2.0）

### 5.2 タップ強化コスト

```
cuc = round(TAP_COST_BASE × TAP_COST_MUL^cl × dm())
```

cl=10 で Infinity を返す（最大レベル）。

### 5.3 施設収益と総時間収益

```
fi(facility) = bi × dm() × FAC_INC_MUL^l × tierBonus() × dbgMul
            (l=0 または未購入なら 0)

ti() = Σ fi(facility) for facility in FD
```

### 5.4 施設アップグレードコスト

```
fuc = round(bc × dm() × FAC_COST_MUL^l)
fpc = round(pur × dm())  // 初回購入コスト
```

### 5.5 ウィスキー販売価格

```
wp(whisky) = round(bs × (yr/3)^YEAR_EXP × ck × dm())
```

例: Glenlivid (bs=100, yr=8, ck=1.0) at Tier1 → wp = round(100 × (8/3)^0.6 × 1.0 × 1) ≈ $179

### 5.6 製造原価

```
craft_cost = round(wp × CRAFT_COST_RATIO)  // CRAFT_COST_RATIO = 0.3
```

### 5.7 自動製造間隔

```
am_interval(ms) = max(2000, AM_BASE_INTERVAL_MS / botLv)
                ≈ Lv1: 30s, Lv5: 6s, Lv10: 3s
```

ボトリング未購入時は Infinity。

### 5.8 同時製造可能銘柄数

```
maxAMConcurrent = G.dl  // Tier番号と同数
```

### 5.9 オフライン収益計算

```
elapsedMs = max(0, now - G.ls)
hours = min(elapsedMs/3600000, OFFLINE_CAP_H)
G.ls = now  // 関数冒頭で即時更新（再呼出時の二重補正防止）

facEarned = ti() × hours

// 自動製造シミュレート（各銘柄ごと）
maxManu = floor(cappedMs / am_interval)
fundConstrain = floor((G.cash + totalEarned) / cc)
manuCount = min(maxManu, fundConstrain)
totalStock = G.inv[id] + manuCount
soldCount = floor(totalStock / threshold) × threshold
remainingStock = totalStock - soldCount
netGain = (wp × soldCount) - (cc × manuCount)

totalEarned = facEarned + Σ netGain
```

### 5.10 銘柄解禁判定

```
chkUnlocks(): 各 UNLOCK_REQ.req() を評価し true なら G.uw に追加
```

イベント駆動（tap, craft, upgFac, upgDist, 2秒UI更新ループ、可視化復帰）で実行。

---

## 6. 経済バランス設計

### 6.1 全Tierクリア所要金額の概算

各Tier「全6施設をLv10まで拡張」に必要な総額（FAC_COST_MUL=3.5、Lv1〜10の総和）:

- Tier1: ≒ $190M
- Tier2: ≒ $9.5B
- Tier3: ≒ $475B
- Tier4: ≒ $24T
- Tier5: ≒ $1200T

各Tierの次Tier解禁コストは「現Tierでの全施設拡張総額」を意図的に上回るように設定（2.6〜20倍）。これによりプレイヤーは「全施設拡張 → 最後にunlockコストを貯める」の2段階を必ず通過する。

### 6.2 進行ペース見積もり

ピーク時（各Tierで全施設Lv10）の時間収益:
- Tier1: ≒ $890K/h
- Tier2: ≒ $44.5M/h
- Tier3: ≒ $2.2B/h
- Tier4: ≒ $111B/h
- Tier5: ≒ $5.5T/h

オフライン補正含めた1日12時間相当の収益で、各Tier解禁コストを賄うのに5〜30日程度。フルコンプリートまで約1か月の設計。

### 6.3 検証モード（dbgMul=10000）

開発・検証用に時間収益のみを10000倍する隠しトグル。OWNER_LINE_IDS に含まれるユーザーまたはゲストモードでのみボタンが表示される。タップ単価には乗らないため「タップ操作の動作確認」と「中盤・終盤の挙動確認」を分離して検証可能。

---

## 7. UI/UX設計

### 7.1 タブ構造

蒸留タブ: タップ画面。樽SVGをタップして現金獲得。Tier別装飾オーバーレイ（王冠・追加樽列・PREMIUMプレート・星）が視覚的進行感を演出。

事業タブ: 製造施設の購入とアップグレード。各施設に履歴ボタン（📖）があり、これまでに導入した設備の詳細を参照可能。

図鑑タブ: 25銘柄のグリッド表示。タップ詳細モーダルで販売価格・フレーバープロファイル・在庫・自動製造トグル・販売地域を確認。

プロフィールタブ: ユーザー情報、累計統計、称号一覧、セーブのエクスポート/インポート、データリセット、検証モード（権限ありのみ）。

### 7.2 モーダル設計

汎用モーダル `#mo` に動的にHTMLを差し込んで使い回す。data-act 属性で操作種別を識別し、イベントデリゲーションで一括処理。サポートする act:
- craft / upgFac / closeM / autosell / sellInv / distConfirm / hist / ioCopy / ioApply

### 7.3 トースト通知

`#toast` 要素1個を使い回し。新着通知が古い通知を即座に上書きする仕様。重要通知（オフライン収益・新Tier到達）は `setTimeout` で 150〜200ms 遅延させ、後続のトーストで上書きされにくいよう調整。

### 7.4 視覚的進行感

タップ画面の樽SVGに `rTapVisuals()` でTier別装飾レイヤを追加:
- Tier2: 王冠
- Tier3: 背景の追加樽列
- Tier4: PREMIUMプレート + スポットライト
- Tier5: 金色の星4箇所

### 7.5 データ可視化

ヘッダーに残高常時表示。蒸留タブに「収益/h・総売上・タップ単価・銘柄コンプ」の2x2グリッド。プロフィールに「現在残高・総売上・受動収益/h・図鑑コンプ率・蒸留所Lv・タップ累計」の2x3グリッド。

---

## 8. 永続化とライフサイクル

### 8.1 保存設計

ストレージキー: `we_v4:<userId>`。LIFFユーザーごとに完全分離した保存領域を持つ。ゲストモード（LIFF未接続）は `we_v4:guest` キー。

`save()` は localStorage への JSON.stringify のみを行う。`G.ls`（最終能動時刻）の更新は `markActive()` で別管理。これは「30秒間隔のsaveが背景throttle中も発火し、`G.ls` を上書きしてオフライン補正を破壊する」バグ（v4で発覚し v5で修正）への対応。

### 8.2 サニタイズと移行

`sanitize(s)` でロード時に各フィールドの型・境界を検証:
- 数値フィールドは `Math.max/min` で範囲クリップ
- 配列・オブジェクトは型検査して既定値補完
- 未知の銘柄ID/施設IDは無視
- 旧フィールド `G.aw`（自動販売）→ `G.am`（自動製造）への移行ロジック

`migrate(s)` でバージョン番号の昇格を行う。現在は単純な version bump のみだが将来的なフィールド変換に備えた拡張点。

### 8.3 オフライン補正の信頼性確保

LIFF特有の閉じ方（X ボタン・別アプリ切替・BFCache復帰・タブ非表示）すべてをカバーするため、6つの経路を多層化:
1. `init()` 起動時
2. `visibilitychange(visible)`
3. `pageshow(persisted)` でBFCache復帰
4. `window.focus`
5. tickのギャップ検知（5秒以上の停止検出）
6. 定期UIループでの diff チェック（軽微）

`offline()` 関数の冒頭で `G.ls = now` を即時更新するため、複数経路で同時呼び出しされても二重補正は発生しない。

### 8.4 検証モード制御

`OWNER_LINE_IDS` 配列にLINE userIdを登録した特定ユーザーまたは LIFF未接続ゲストモードでのみ検証ボタンを表示。第三者がインポート機能で検証ON状態のセーブを取り込んでも `syncDebugBtn()` で強制 OFF にする保険を実装。

---

## 9. LIFF（LINE Mini App）統合

### 9.1 設定

ファイル冒頭の `<script id="liff-config">` ブロックで `LIFF_ID` を定数として保持。`'__LIFF_ID__'` プレースホルダ時はゲストモード固定。

### 9.2 認証フロー

1. `liff.init({liffId})` で SDK 初期化（最大2秒タイムアウト）
2. `liff.isLoggedIn()` で確認、未ログインなら `liff.login()` でリダイレクト
3. `liff.getProfile()` で `userId / displayName / pictureUrl` 取得
4. 取得失敗時は console.warn でログ後、ゲストモードへフォールバック

### 9.3 共有機能

`liff.shareTargetPicker(['type:text', text])` でゲーム進捗を友だちにシェアするボタンをプロフィール画面に提供。`liff.isApiAvailable()` で対応環境のみ表示。

### 9.4 セキュリティ考慮

`pictureUrl` は `^https://` の正規表現で検証してから `<img src>` に挿入（javascript: 等のスキーム排除）。動的 HTML 挿入箇所は `escapeHtml()` を経由。

---

## 10. 拡張性の指針

他の経営ゲームと接続・連携しやすくするための設計指針を以下にまとめる。本作の現コードはまだ全てを実装していないため、参考として後続実装での適用を想定する。

### 10.1 テーマレイヤとエンジンレイヤの分離

現コードの DL/FD/WL/TT/UNLOCK_REQ/FAC_UPG はすべて「ウィスキー」というテーマ固有データ。これらを `theme` オブジェクトに集約し、エンジンコード（cv/fi/ti/wp/dm/upgDist/craft 等）は theme をパラメータとして受け取る形に再構築すれば、テーマ差し替えだけで「茶屋」「ワイナリー」「コーヒー焙煎」等の業態に変更可能。

```javascript
// 推奨設計
const theme = {
  id: 'whiskyEmpire',
  tiers: DL,          // 蒸留所Tier
  facilities: FD,     // 施設
  items: WL,          // 銘柄
  achievements: TT,
  unlocks: UNLOCK_REQ,
  upgrades: FAC_UPG,
  balance: BALANCE
};
const engine = createEngine(theme);
```

### 10.2 ストレージキーのネームスペース

複数ゲームを 1つのLIFFアプリに統合する場合、ストレージキーは `gameHub:<userId>` をルートにし、内部で `{whiskyEmpire: {...}, teaShop: {...}}` の階層構造を持たせると、ゲーム間で `cash` を共有したり、別ゲームの達成を参照可能になる。

```javascript
const root = JSON.parse(localStorage.getItem('gameHub:' + userId) || '{}');
root.whiskyEmpire = G;
localStorage.setItem('gameHub:' + userId, JSON.stringify(root));
```

### 10.3 イベントバスの導入

ゲーム間でイベントを通知するための pub/sub を入れる。例: 「ウィスキー帝国でTier5達成」を別ゲームで「ボーナス称号付与」のフラグに利用。

```javascript
const bus = {
  listeners: {},
  on(ev, fn){(this.listeners[ev]=this.listeners[ev]||[]).push(fn);},
  emit(ev, data){(this.listeners[ev]||[]).forEach(fn=>fn(data));}
};
// 使用例
bus.emit('game:tierReached', {gameId:'whiskyEmpire', tier:5});
```

### 10.4 共通APIサーフェス

各ゲームが実装すべき共通インターフェース例:

```javascript
interface GameModule {
  id: string;
  init(ctx: AppContext): Promise<void>;
  render(container: HTMLElement): void;
  save(): SaveData;
  load(data: SaveData): void;
  destroy(): void;
  events?: EventBus;
}
```

`AppContext` には共通の userId、ストレージ、トースト関数、LIFFインスタンスを含めて DI する。

### 10.5 通貨と称号の共有設計

ゲーム間で通貨を共通化する場合、各ゲームは `requestCash(amount)` / `addCash(amount)` をエンジン経由で呼び出し、内部の単一ウォレットに反映。称号は `unlockAchievement(id)` で全ゲーム共通リストに追加。

### 10.6 描画レイヤの分離

現コードは render 関数が DOM を直接操作（`$('hc').textContent = ...`）。この方式は単一ゲームでは軽量で十分だが、複数ゲームを切替表示する場合、各ゲームが自身の DOM サブツリーを管理し、外部（ハブ）からは「render(container)」「unmount()」だけを呼ぶ形が望ましい。

### 10.7 設定駆動とJSON分離

理想的には `theme.json` を別ファイルにして、ゲームHTMLは「エンジン + UI」のみ、テーマは外部から読み込む構造。これにより「同じエンジンで複数ゲーム」が実現する。単一HTML制約がある場合は `<script type="application/json" id="theme">` で埋め込みJSONを採用する案もある。

### 10.8 多言語対応の余地

現状は日本語ハードコード。将来的に i18n 対応する場合、テキストキーの集約（`I18N.ja.tap_to_earn` 等）と、`escapeHtml` 適用の徹底が必要。

### 10.9 サーバ連携の拡張点

現状はクライアント完結（localStorage）。クラウドセーブやランキングを実装する場合:
- LINE Login の access_token を用いて自前サーバへ認証
- セーブ JSON をサーバに POST、サーバは userId をキーに保存
- ランキング集計はサーバ側で日次バッチ
- 不正対策のため重要数値（cash, te）は署名付きで送受信

---

## 11. デプロイ・運用

### 11.1 LINE Mini App公開手順

1. LINE Developers Console で LIFF アプリを作成、LIFF ID を取得
2. ファイル冒頭の `LIFF_ID` を実IDに書換
3. 自分の LINE userId を `OWNER_LINE_IDS` に追加（プロフィール画面下部の保存先表示で確認）
4. HTTPS 配信できる任意のホスティングへ配置（GitHub Pages / Netlify / Vercel / Cloudflare Pages 等）
5. LIFF アプリのエンドポイントURLを配置先に設定
6. Scope: `profile` のみで足りる

### 11.2 検証手順

ゲストモード（PCブラウザでファイル直接オープン）でゲーム機能の挙動確認、検証モード（dbgMul=10000）で経済バランスの中盤・終盤確認、LINE実機でLIFF統合・オフライン補正の検証。

### 11.3 セーブのエクスポート/インポート

プロフィール画面からBase64エンコードのセーブ文字列を発行・取込可能。端末変更時の引継ぎや、複数デバイス間での状態移行に利用できる。`btoa(unescape(encodeURIComponent(...)))` の古典イディオムでUTF-8安全に変換。

---

## 12. 用語集

| 用語 | 説明 |
|---|---|
| Tier | 蒸留所のグレード。1〜5の整数 |
| dm | Distillery Multiplier。DL[Tier-1].mult |
| tierBonus | Tier追加倍率。1.0〜2.0 |
| Lv | レベル（タップ強化・施設）。0〜10 |
| 受動収益 | 施設由来の時間収益。tap操作なしで増える |
| 自動製造 | ボトリング購入後に有効な、定期的な銘柄製造 |
| 自動販売 | 在庫が閾値（10本）に達した時の一括売却 |
| 同時製造スロット | 同時に自動製造できる銘柄数の上限。Tier番号と同数 |
| 解禁条件 | 銘柄が図鑑に出現するための条件 |
| シークレット枠 | 解禁条件が非開示の高難度銘柄。最後尾3銘柄 |
| 検証モード | 受動収益を10000倍する開発・検証用モード |
| LIFF | LINE Front-end Framework。LINEミニアプリの基盤 |
| BFCache | Back-Forward Cache。ブラウザのページ復帰最適化機構 |
| markActive | 「最終能動時刻」を `G.ls=now` で記録するヘルパ |
| offline | バックグラウンド経過時間からの収益補正処理 |

---

本仕様書は v1.0 として完成版コードベースに基づき作成。将来のバージョンアップでは差分を別ドキュメントで追記すること。
