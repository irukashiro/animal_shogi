# 仕様書 09: Controller(手の入力)

- 実装ファイル:`dobutsu/controller.py`
- テストファイル:`tests/test_controller.py`
- 対象クラス:`Controller`(抽象基底)、`HumanController`、`AIController`

## 1. 役割
手番のプレイヤーから **指し手(`Move`)を1つ受け取る**。

- `Controller` は共通のインターフェース(`choose_move`)を定める。
- `HumanController` は人間のキーボード入力を `Move` に変換する。
- `AIController` はコンピュータが `Move` を選ぶ(最初はランダム)。

`Game` は `Controller` の種類を区別せず、`choose_move` を呼ぶだけ。したがって、人間同士・人間対AI・AI同士の対局が `Game` を変更せずにできる。

## 2. 前提知識

### 返す手の条件
`choose_move` は **必ず `legal_moves` に含まれる `Move` を返す**。`legal_moves` は `Rule.legal_moves` が作った合法手のリストで、`Controller` が自分で手を作る必要はない。

### Move の構造(必要部分)
- `move.from_pos`:移動元の `Position`。持ち駒を打つ手では `None`。
- `move.to_pos`:移動先(または打つ場所)の `Position`。
- `move.piece.kind`:動かす(打つ)駒の種類(`"chick"` など)。
- `move.is_drop`:打つ手なら `True`。
- `Position(row, col)`:`row` は 0〜3、`col` は 0〜2。左上が `(0, 0)`、先手は下(row 3 側)から上へ進む。

### 入力形式(HumanController)

| 手の種類 | 入力形式 | 例 |
|---|---|---|
| 移動 | `行,列 行,列`(移動元と移動先) | `2,1 1,1` |
| 持ち駒を打つ | `drop 駒の種類 行,列` | `drop chick 2,0` |

- 区切りは半角スペース。大文字小文字は区別しない。前後の余分な空白は無視する。
- 駒の種類は `kind` の文字列:`chick`、`elephant`、`giraffe`。

## 3. 依存するもの

| 依存先 | 使うもの |
|---|---|
| `Board` | 型ヒントのみ(`AIController` の拡張時は `copy`) |
| `Player` | 型ヒント、`name` |
| `Move` | 属性の読み取り |
| `Position` | `Position(row, col)` の生成(入力の解析用) |
| `View` | `show_message(text)`(`HumanController` のみ) |

## 4. Controller(抽象基底クラス)

`abc.ABC` を継承する。

### `choose_move(self, board: Board, player: Player, legal_moves: list[Move]) -> Move` (抽象メソッド)
- **引数:**
  - `board`:現在の盤面(**変更しない**)
  - `player`:手番のプレイヤー
  - `legal_moves`:合法手のリスト(空でない。空の局面は `Game` が事前に処理する)
- **戻り値:** `legal_moves` の要素のうちの1つ(同一オブジェクト)
- `@abstractmethod`。`Controller()` を直接生成すると `TypeError`。

## 5. HumanController

### 属性

| 名前 | 型 | 説明 |
|---|---|---|
| `_view` | `View` | メッセージ表示用 |
| `_input` | `Callable[[str], str]` | 入力関数。既定は組み込みの `input` |

### `__init__(self, view: View, input_func: Callable[[str], str] = input) -> None`
- **引数:**
  - `view`:メッセージ表示に使う `View`
  - `input_func`:1行入力を得る関数。テスト時に `input` の代わりに偽物を渡せるようにするためのもの。
- **内部処理:** `self._view = view`、`self._input = input_func`

### `parse(self, text: str, legal_moves: list[Move]) -> Optional[Move]`
- **引数:**
  - `text`:ユーザーが入力した1行
  - `legal_moves`:合法手のリスト
- **戻り値:** 入力に対応する `legal_moves` の要素。**解析できない、または対応する合法手がなければ `None`**(例外は出さない)。
- **内部処理:**
  1. `tokens = text.strip().lower().split()`
  2. `tokens[0] == "drop"` で `len(tokens) == 3` のとき(打つ手):
     1. `kind = tokens[1]`
     2. `to = self._parse_pos(tokens[2])`、`None` なら `None` を返す
     3. `legal_moves` の中で `m.is_drop and m.piece.kind == kind and m.to_pos == to` の最初の `m` を返す。なければ `None`
  3. `len(tokens) == 2` のとき(移動):
     1. `frm = self._parse_pos(tokens[0])`、`to = self._parse_pos(tokens[1])`、どちらかが `None` なら `None` を返す
     2. `legal_moves` の中で `m.from_pos == frm and m.to_pos == to` の最初の `m` を返す。なければ `None`
  4. それ以外の形式なら `None`

### `_parse_pos(self, text: str) -> Optional[Position]`
- **引数:** `text`:`"2,1"` のような文字列
- **戻り値:** `Position(2, 1)`。形式が不正なら `None`。
- **内部処理:**
  1. `","` で2つに分割する。2つにならなければ `None`
  2. それぞれを `int` に変換する。変換できなければ(`ValueError`)`None`
  3. `Position(row, col)` を返す(**盤内かの検査は不要**。盤外なら `parse` で合法手が見つからず `None` になる)

### `choose_move(self, board, player, legal_moves) -> Move`
- **内部処理:**
  1. 無限ループにする
  2. `text = self._input(f"{player.name}の手を入力してください(移動: 2,1 1,1 / 打つ: drop chick 2,0): ")`
  3. `move = self.parse(text, legal_moves)`
  4. `move` が `None` でなければ返す
  5. `None` なら `self._view.show_message("入力が不正か、指せない手です。もう一度入力してください。")` を表示してループに戻る
- 不正な入力で例外を出さず、正しい入力が得られるまで繰り返す。

## 6. AIController

### 属性

| 名前 | 型 | 説明 |
|---|---|---|
| `_rng` | `random.Random` | 乱数生成器 |

### `__init__(self, seed: Optional[int] = None) -> None`
- **引数:** `seed`:乱数の種。同じ値を渡すと同じ手を選ぶ(テストで再現できる)。
- **内部処理:** `self._rng = random.Random(seed)`

### `choose_move(self, board, player, legal_moves) -> Move`
- **基本版(必須):** `self._rng.choice(legal_moves)` を返す。
- **戻り値:** `legal_moves` の要素

### 拡張案(任意課題)
難しくする場合は、次の順に拡張する。いずれも `choose_move` のシグネチャを変えない。

1. **ライオンを取れる手を優先:** `legal_moves` に `m.captured` が `Lion` の手があればそれを返す。
2. **取られる手を避ける:** 手を指した後、自分の駒が相手に取られる位置に動かす手を避ける(`Rule.is_attacked` を使う場合は `Rule` を引数で渡す設計に変える)。
3. **先読み(ミニマックス法):** `board.copy()` と、持ち駒の複製が必要。`Game.apply_move` と同じ処理が `Controller` 側にも要るため、設計変更が必要になる。取り組む場合は班で相談する。

## 7. 使われ方

| 呼び出し元 | 使うもの |
|---|---|
| `Game.__init__` | `Controller` のインスタンスを2つ受け取り、辞書に保持する |
| `Game.play_turn` | `controllers[current].choose_move(board, current, moves)` |
| `main.py` | `HumanController(view)`、`AIController()` を生成する |

```python
view = View()
ctrl = HumanController(view)
move = ctrl.choose_move(board, first, legal_moves)   # 入力待ちになる
```

## 8. テスト項目

### 8.1 Controller

| # | 操作 | 期待結果 |
|---|---|---|
| 1 | `Controller()` | `TypeError` |
| 2 | `choose_move` を実装しないサブクラスを生成 | `TypeError` |

### 8.2 HumanController.parse
テスト用に、次の `legal_moves` を用意する(`Move` と `Position`、`Piece` を使って作る)。

- `A`:ヒヨコ (2,1)→(1,1)
- `B`:ライオン (3,1)→(2,0)
- `C`:ヒヨコを打つ手 (2,2)
- `D`:ゾウを打つ手 (2,2)

| # | 入力 | 期待結果 |
|---|---|---|
| 3 | `"2,1 1,1"` | `A` |
| 4 | `"3,1 2,0"` | `B` |
| 5 | `"drop chick 2,2"` | `C` |
| 6 | `"drop elephant 2,2"` | `D` |
| 7 | `"  2,1   1,1  "`(余分な空白) | `A` |
| 8 | `"DROP Chick 2,2"`(大文字) | `C` |
| 9 | `"2,1 0,1"`(合法手にない) | `None` |
| 10 | `"drop giraffe 2,2"`(持っていない種類) | `None` |
| 11 | `"abc"` | `None`(例外を出さない) |
| 12 | `""` | `None` |
| 13 | `"2,1"`(トークンが1つ) | `None` |
| 14 | `"2,1 1,1 0,0"`(トークンが3つで先頭が drop でない) | `None` |
| 15 | `"a,b 1,1"` | `None` |
| 16 | `"2,1 9,9"`(盤外) | `None` |
| 17 | `"drop chick"` | `None` |
| 18 | `parse` の戻り値 | `legal_moves` の中の同一オブジェクト(`is`) |

### 8.3 HumanController.choose_move
`input_func` に、あらかじめ決めた文字列を順に返す偽物を渡す。`View` は `show_message` の呼び出しを記録するスタブ。

| # | 偽の入力列 | 期待結果 |
|---|---|---|
| 19 | `["2,1 1,1"]` | `A` が返り、`show_message` は呼ばれない |
| 20 | `["abc", "2,1 1,1"]` | `A` が返り、`show_message` が1回呼ばれる |
| 21 | `["x", "y", "drop chick 2,2"]` | `C` が返り、`show_message` が2回呼ばれる |
| 22 | プロンプトの文字列 | `player.name` を含む |

### 8.4 AIController

| # | 操作 | 期待結果 |
|---|---|---|
| 23 | `choose_move` の戻り値 | `legal_moves` に含まれる(100回繰り返しても) |
| 24 | `legal_moves` が1つだけ | その1つを返す |
| 25 | `AIController(seed=1)` を2つ作り同じ入力で呼ぶ | 同じ手を返す |
| 26 | 呼び出し前後の `board` | 変化しない |
| 27 | 多数回呼んだとき | 複数の異なる手が選ばれる(偏りの確認) |

## 9. 実装上の注意
- `input` を `HumanController` の中で直接呼ばず、`self._input` を通す(テストできるようにするため)。
- `choose_move` が返す `Move` は **`legal_moves` の中の同じオブジェクト**。新しく `Move` を作って返さない(打つ手の `piece` の不一致を避けるため)。
- `Controller` は盤面や持ち駒を変更しない。
- 入力の誤りは例外ではなく、`View.show_message` による通知と再入力で扱う。`Ctrl+C` による中断など、ユーザーの意図による終了は処理しなくてよい。
- 手の選択に必要な情報(ルール)を `Controller` に持たせない。合法手は引数で受け取る。
