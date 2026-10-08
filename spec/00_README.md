# どうぶつしょうぎ 詳細仕様書 目次

全体設計は [../dobutsu_design.md](../dobutsu_design.md) を参照。
本フォルダの仕様書は **1クラス1ファイル** で、担当者がその1ファイルだけ読めば実装とテストができるように書いてある。

## 仕様書一覧

| ファイル | 対象クラス | 実装ファイル | 他クラスへの依存 |
|---|---|---|---|
| [01_Position.md](01_Position.md) | `Position` | `dobutsu/position.py` | なし |
| [02_Piece.md](02_Piece.md) | `Piece`, `Lion`, `Elephant`, `Giraffe`, `Chick`, `Hen` | `dobutsu/pieces.py` | Player(型のみ) |
| [03_Player.md](03_Player.md) | `Player` | `dobutsu/player.py` | Piece(型のみ) |
| [04_Move.md](04_Move.md) | `Move` | `dobutsu/move.py` | Position, Piece, Player(型のみ) |
| [05_Board.md](05_Board.md) | `Board` | `dobutsu/board.py` | Position, Piece, Player |
| [06_Rule.md](06_Rule.md) | `Rule` | `dobutsu/rule.py` | Board, Move, Position, Piece, Player |
| [07_Game.md](07_Game.md) | `Game`, `main.py` | `dobutsu/game.py`, `main.py` | 全クラス |
| [08_View.md](08_View.md) | `View` | `dobutsu/view.py` | Board, Player |
| [09_Controller.md](09_Controller.md) | `Controller`, `HumanController`, `AIController` | `dobutsu/controller.py` | Board, Player, Move, View |

担当の割り振りは [10_assignment.md](10_assignment.md) を参照。

## フォルダ構成

```
software/
├── dobutsu/
│   ├── __init__.py        （空ファイルでよい）
│   ├── position.py
│   ├── pieces.py
│   ├── player.py
│   ├── move.py
│   ├── board.py
│   ├── rule.py
│   ├── game.py
│   ├── view.py
│   └── controller.py
├── main.py
└── tests/
    ├── __init__.py        （空ファイルでよい）
    └── test_<モジュール名>.py
```

- Python 3.9 以上、型ヒントを付ける。
- import は `from dobutsu.position import Position` の形式に統一する。
- テストは標準の `unittest` で書く。実行は `software` フォルダで `python -m unittest discover -s tests -t .`。

## 全クラス共通の約束事

### 座標系
- 盤面は **4行 × 3列**。座標は `Position(row, col)`、`row` は 0〜3、`col` は 0〜2。
- 左上が `(0, 0)`、右下が `(3, 2)`。
- 先手は盤の下側(row 3 側)にいて **上(row が減る方向)へ進む**。後手は上側にいて **下(row が増える方向)へ進む**。

| | `direction`(前方向の `dr`) | ゴール段(成り・トライの段) |
|---|---|---|
| 先手 | -1 | row 0 |
| 後手 | +1 | row 3 |

### 駒の識別子

| 駒 | クラス | `kind` | `symbol` |
|---|---|---|---|
| ライオン | `Lion` | `"lion"` | 獅 |
| ゾウ | `Elephant` | `"elephant"` | 象 |
| キリン | `Giraffe` | `"giraffe"` | 麒 |
| ヒヨコ | `Chick` | `"chick"` | 雛 |
| ニワトリ | `Hen` | `"hen"` | 鶏 |

### 初期配置

| 座標 | 先手 | 座標 | 後手 |
|---|---|---|---|
| (3,0) | キリン | (0,0) | ゾウ |
| (3,1) | ライオン | (0,1) | ライオン |
| (3,2) | ゾウ | (0,2) | キリン |
| (2,1) | ヒヨコ | (1,1) | ヒヨコ |

### 勝敗(ルール)
- **キャッチ**:相手のライオンを取ったら勝ち。
- **トライ**:自分のライオンが相手陣のゴール段に入り、次の相手の手で取られなければ勝ち。
- 王手の概念はない。自分のライオンが取られる位置に動かす手も合法(その結果負けるだけ)。
- 取った駒は持ち駒になる。ニワトリを取るとヒヨコとして持ち駒になる。
- 持ち駒は自分の手番で、動かす代わりに空きマスへ打てる。打つ場所の制限は設けない(班で確認済みの場合のみ変更)。
- ヒヨコがゴール段に入ると自動でニワトリに成る(選択はない)。

### オブジェクトの同一性
- `Player` と `Piece` は **同一性(`is`)で比較する**。`__eq__` は定義しない。
- `Position` は値で比較する(`dataclass` の `==`)。
- `Piece` は作成後に属性を書き換えない(所有者を変えるときは新しい駒を作る)。

## 推奨する実装の進め方

1. 全員で本README・全体設計書を読み、疑問点を出して合意する。
2. 各自、自分のクラスの **シグネチャだけを書いた骨組み**(中身は `raise NotImplementedError`)を最初に作って共有する。これで他の担当が import できる。
3. 各自、仕様書の「テスト項目」を `tests/` に実装しながら本体を書く。
4. 依存の少ない順に結合する:`Position` → `Piece` / `Player` / `Move` → `Board` → `Rule` → `View` / `Controller` → `Game`。
5. 人間対人間で1局通して動作確認する。

## 仕様書の読み方
各仕様書は次の順で書かれている。

1. 役割
2. 前提知識(このクラスに関係する決まりごと)
3. 依存するもの(使ってよい外部のクラスとメソッド)
4. 属性
5. メソッド仕様(引数、戻り値、例外、内部処理の手順)
6. 使われ方(誰がどう呼ぶか)
7. テスト項目(具体的な入力と期待値)
8. 実装上の注意
