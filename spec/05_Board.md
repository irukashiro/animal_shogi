# 仕様書 05: Board(盤面)

- 実装ファイル:`dobutsu/board.py`
- テストファイル:`tests/test_board.py`

## 1. 役割
4行×3列のマスと、そこに置かれた駒を管理する。
**ルールは一切知らない。** 「駒が動けるか」「取れるか」「勝ったか」は判定せず、置く・取り除く・調べる、だけを行う。

## 2. 前提知識
- 盤面は 4行×3列。座標は `Position(row, col)`、`row` が 0〜3、`col` が 0〜2。左上が `(0, 0)`。
- 各マスには駒が1枚、または空(`None`)。
- 駒(`Piece`)には所有者 `owner`(`Player`)と種類 `kind`(`"lion"` など)がある。
- 持ち駒は `Board` ではなく `Player` が持つ。盤上の駒だけを扱う。
- 先手・後手の初期配置は `Game.setup` が `set` を使って行う。`Board` 自身は空の盤で作られる。

## 3. 依存するもの

| 依存先 | 使うもの |
|---|---|
| `dobutsu.position` | `Position`、定数 `BOARD_ROWS`(4)、`BOARD_COLS`(3) |
| `Piece` | `kind`、`owner`(型ヒントと `find_lion` / `pieces_of` で使用) |
| `Player` | 型ヒントと、`pieces_of` での所有者の比較 |

## 4. 内部データ
盤は 4×3 の二次元リスト `self._grid` で持つ(先頭の `_` は「外から直接触らない」の意味)。

- `self._grid[row][col]` が `Piece` または `None`。
- 作成: `[[None] * BOARD_COLS for _ in range(BOARD_ROWS)]`
  - 注意:`[[None] * 3] * 4` と書かない(4行が同じリストを共有してしまう)。
- 外部のクラスは `_grid` を直接触らず、必ずメソッドを通す。

## 5. メソッド仕様

### `__init__(self) -> None`
- **処理:** 全マスが `None` の盤を作る。
- **引数:** なし

### `in_bounds(self, pos: Position) -> bool`
- **引数:** `pos`
- **戻り値:** 盤内なら `True`
- **処理:** `pos.is_valid()` を返す。

### `get(self, pos: Position) -> Optional[Piece]`
- **引数:** `pos`:調べるマス
- **戻り値:** そのマスの駒。空なら `None`。
- **処理:**
  1. `pos` が盤外なら `ValueError`
  2. `self._grid[pos.row][pos.col]` を返す
- **例外:** 盤外のとき `ValueError`

### `set(self, pos: Position, piece: Piece) -> None`
- **引数:** `pos`:置くマス、`piece`:置く駒
- **戻り値:** なし
- **処理:**
  1. `pos` が盤外なら `ValueError`
  2. `self._grid[pos.row][pos.col] = piece`(**既にある駒は無条件に上書き**する)
- **例外:** 盤外のとき `ValueError`
- 上書きを許す理由:盤面の更新の順序は `Game.apply_move` が管理する。`Board` は検査しない。

### `remove(self, pos: Position) -> Optional[Piece]`
- **引数:** `pos`:取り除くマス
- **戻り値:** 取り除いた駒。空だったなら `None`。
- **処理:**
  1. `pos` が盤外なら `ValueError`
  2. そのマスの駒を取り出して控える
  3. そのマスを `None` にする
  4. 控えた駒を返す
- **例外:** 盤外のとき `ValueError`

### `move_piece(self, from_pos: Position, to_pos: Position) -> None`
- **引数:** `from_pos`:移動元、`to_pos`:移動先
- **戻り値:** なし
- **処理:**
  1. どちらかが盤外なら `ValueError`
  2. `from_pos` が空なら `ValueError`
  3. `to_pos` が空でなければ `ValueError`(**取る処理はしない**。取る場合は呼び出し側が先に `remove(to_pos)` する)
  4. `piece = remove(from_pos)`、`set(to_pos, piece)`
- **例外:** 上記の3条件で `ValueError`

### `is_empty(self, pos: Position) -> bool`
- **戻り値:** そのマスが空なら `True`
- **処理:** `self.get(pos) is None`(盤外なら `get` が `ValueError`)

### `pieces_of(self, player: Player) -> list[tuple[Position, Piece]]`
- **引数:** `player`
- **戻り値:** `player` が所有する盤上の駒を、`(座標, 駒)` のタプルのリストで返す。
- **処理:**
  1. row 0→3、各行で col 0→2 の順(**行優先**)にすべてのマスを走査する
  2. 駒があり `piece.owner is player` ならリストに追加する
- 順序を固定するのは、`Rule.legal_moves` の結果の順序を毎回同じにするため。

### `find_lion(self, player: Player) -> Optional[Position]`
- **引数:** `player`
- **戻り値:** `player` のライオンの座標。盤上にいなければ(取られていれば)`None`。
- **処理:** `pieces_of(player)` から `piece.kind == "lion"` のものを探して座標を返す。

### `empty_positions(self) -> list[Position]`
- **戻り値:** 空きマスの座標のリスト(行優先の順)
- **用途:** `Rule` が「打つ手」を作るとき

### `copy(self) -> Board`
- **戻り値:** 同じ配置の新しい `Board`
- **処理:**
  1. 空の `Board` を作る
  2. 各マスの駒を新しい盤にコピーする(**`Piece` 自体はコピーせず共有する**。`Piece` は不変として扱うため)
  3. 新しい盤を返す
- 複製した盤を変更しても、元の盤に影響しないこと(リストは別々に作る)。

## 6. 使われ方

| 呼び出し元 | 使うメソッド | 目的 |
|---|---|---|
| `Game.setup` | `set` | 初期配置 |
| `Game.apply_move` | `remove`, `set`, `move_piece` | 手の反映 |
| `Rule.legal_moves` | `pieces_of`, `get`, `in_bounds`, `empty_positions` | 合法手の生成 |
| `Rule.is_attacked` | `pieces_of` | 攻撃されているかの判定 |
| `Rule.judge_winner` | `find_lion`, `pieces_of` | 勝敗判定 |
| `View.show_board` | `get` | 表示 |
| `AIController`(拡張時) | `copy` | 先読みの試行 |

```python
board = Board()
board.set(Position(3, 1), Lion(first))
board.get(Position(3, 1))            # Lion
board.find_lion(first)               # Position(3, 1)
board.move_piece(Position(3, 1), Position(2, 1))
board.pieces_of(first)               # [(Position(2, 1), Lion)]
```

## 7. テスト項目
先手 `first`、後手 `second`、駒クラス(未実装ならダミー)を用意する。

| # | 操作 | 期待結果 |
|---|---|---|
| 1 | `Board()` を作り、全12マスを `get` | すべて `None` |
| 2 | `set(Position(1,1), chick)` → `get(Position(1,1))` | `chick`(同一オブジェクト) |
| 3 | `set` した後、別のマスを `get` | `None` |
| 4 | 駒がある `(1,1)` に別の駒を `set` | 上書きされる |
| 5 | `remove(Position(1,1))`(駒あり) | その駒が返り、以後 `get` は `None` |
| 6 | `remove(Position(0,0))`(空) | `None` |
| 7 | `get(Position(4,0))` | `ValueError` |
| 8 | `get(Position(0,-1))` | `ValueError` |
| 9 | `set(Position(4,0), chick)` | `ValueError` |
| 10 | `remove(Position(0,3))` | `ValueError` |
| 11 | `move_piece((3,1),(2,1))`(移動元に駒あり・移動先は空) | `(3,1)` が空、`(2,1)` に駒 |
| 12 | `move_piece` で移動元が空 | `ValueError` |
| 13 | `move_piece` で移動先に駒あり | `ValueError`(盤面は変化しない) |
| 14 | `move_piece` で盤外 | `ValueError` |
| 15 | `in_bounds(Position(3,2))` / `(4,0)` | `True` / `False` |
| 16 | `is_empty` を空きマスと駒ありマスで | `True` / `False` |
| 17 | 先手の駒を (3,1) (2,1) に、後手の駒を (0,1) に置いて `pieces_of(first)` | `[(Position(2,1), …), (Position(3,1), …)]`(行優先の順で2要素) |
| 18 | 同じ盤で `pieces_of(second)` | 1要素 |
| 19 | 駒が無いプレイヤーの `pieces_of` | `[]` |
| 20 | 先手のライオンを (3,1) に置いて `find_lion(first)` | `Position(3,1)` |
| 21 | ライオンを置いていないとき `find_lion(first)` | `None` |
| 22 | ライオンを先手と後手に1頭ずつ置き `find_lion(second)` | 後手のライオンの座標 |
| 23 | 2駒を置いた盤の `empty_positions()` | 10個、置いた2マスを含まない、行優先の順 |
| 24 | 空の盤の `empty_positions()` の長さ | 12 |
| 25 | `b2 = board.copy()` の後 `b2.remove(pos)` | `board` の同じマスは変化しない |
| 26 | `b2 = board.copy()` | すべてのマスで `b2.get(p) is board.get(p)` |

## 8. 実装上の注意
- 盤外の座標に対して `IndexError` や負のインデックスによる逆側アクセスが起きないよう、`get` `set` `remove` の最初で必ず盤内を確認する(Python のリストは `-1` で末尾を指してしまう)。
- `Board` はルールを知らない。`set` の中で取る処理や勝敗判定をしない。
- `pieces_of` の所有者の比較は `piece.owner is player`(`==` ではなく `is`)。
