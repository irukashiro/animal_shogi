# 仕様書 06: Rule(ルール判定)

- 実装ファイル:`dobutsu/rule.py`
- テストファイル:`tests/test_rule.py`

## 1. 役割
どうぶつしょうぎの **ルール判定** を担当する。

1. 手番のプレイヤーが指せる手(合法手)をすべて求める。
2. ある手が合法かどうかを調べる。
3. あるマスが相手に取られる状態かを調べる。
4. 手を指した直後の盤面から、勝者を判定する。

**状態を持たない。** すべてのメソッドは、引数の `board` と `player` を **読むだけ** で、変更しない。盤面や持ち駒の更新は `Game` が行う。

## 2. 前提知識

### 座標と向き
- 盤面は 4行×3列。座標は `Position(row, col)`、`row` が 0〜3、`col` が 0〜2。
- 先手は `direction = -1`(上へ進む)、ゴール段は row 0。後手は `direction = +1`(下へ進む)、ゴール段は row 3。
- `Piece.get_move_offsets()` は、所有者の向きを反映済みの移動方向 `[(dr, dc), ...]` を返す。**`Rule` 側で先手・後手の反転をしてはいけない。**

### 手の種類
1. **移動:** 盤上の駒を1マス動かす。移動先が盤内で、味方の駒がなければ動かせる。相手の駒があれば取る。
2. **打つ:** 持ち駒を空きマスに置く。打てる場所に制限はない(どの空きマスでもよい)。

### 特別なルール
- **王手の概念はない。** 自分のライオンが取られる位置に動く手、取られる状態を放置する手も合法として列挙する(その結果負けるだけ)。
- ヒヨコがゴール段に入る移動は `promote=True`。ニワトリは成らない。ゴール段に **打つ** 手は成らない(`promote=False`)。
- ライオンを取る手も通常の取る手として列挙する。

### 勝敗(2種類)
- **キャッチ:** 相手のライオンを取る。
- **トライ:** 自分のライオンがゴール段に入り、その次の相手の手で取られない。

## 3. 依存するもの

| 依存先 | 使うもの |
|---|---|
| `Board` | `pieces_of(player)`, `get(pos)`, `in_bounds(pos)`, `empty_positions()`, `find_lion(player)` |
| `Position` | `offset(dr, dc)`, `is_valid()`(`Position` 自体が値オブジェクト) |
| `Piece` | `kind`, `owner`, `get_move_offsets()` |
| `Player` | `goal_row`, `hand_kinds()`, `peek_hand(kind)` |
| `Move` | 生成する対象、`key`, `piece.kind`, `captured`, `player` |

## 4. 属性
なし。`__init__` に引数はない(`Rule()` で生成する)。

## 5. メソッド仕様

### `legal_moves(self, board: Board, player: Player) -> list[Move]`
- **引数:**
  - `board`:現在の盤面(変更しない)
  - `player`:手番のプレイヤー
- **戻り値:** 合法手の `Move` のリスト。**順序は決定的**(同じ盤面なら常に同じ順序)。順序は「移動の手(`pieces_of` の順、各駒の `get_move_offsets()` の順)→ 打つ手(`hand_kinds()` の順、各 `empty_positions()` の順)」。
- **内部処理:**

  **(a) 移動の手**
  1. 結果リスト `moves` を空で用意する
  2. `board.pieces_of(player)` の各 `(pos, piece)` について
  3. 　`piece.get_move_offsets()` の各 `(dr, dc)` について
  4. 　　`to = pos.offset(dr, dc)` を求める
  5. 　　`board.in_bounds(to)` でなければ次へ
  6. 　　`target = board.get(to)`
  7. 　　`target` があり `target.owner is player` なら次へ(味方は取れない)
  8. 　　`promote = (piece.kind == "chick" and to.row == player.goal_row)` を求める
  9. 　　`Move(player, piece, pos, to, captured=target, promote=promote)` を `moves` に追加する

  **(b) 打つ手**
  10. `player.hand_kinds()` の各 `kind` について
  11. 　`piece = player.peek_hand(kind)`
  12. 　`board.empty_positions()` の各 `to` について
  13. 　　`Move(player, piece, None, to)` を `moves` に追加する(`captured=None`, `promote=False`)

  14. `moves` を返す
- **例外:** なし
- 同じ種類の持ち駒を複数持っていても、打つ手は **種類ごとに1つ** だけ作る(同じ手の重複を作らない)。

### `is_legal(self, board: Board, player: Player, move: Move) -> bool`
- **引数:** `board`, `player`, `move`:検査したい手
- **戻り値:** `move` が合法手に含まれれば `True`
- **内部処理:**
  1. `legal_moves(board, player)` を求める
  2. その中に `m.key == move.key` となる `m` があれば `True`、なければ `False`
- `move.key` は `(player, piece.kind, from_pos, to_pos)`。`==` で `Move` 全体を比べない(打つ手で使う駒オブジェクトが異なる場合があるため)。

### `is_attacked(self, board: Board, pos: Position, by_player: Player) -> bool`
- **引数:**
  - `board`:現在の盤面
  - `pos`:調べたいマス
  - `by_player`:攻撃する側のプレイヤー
- **戻り値:** `by_player` の盤上の駒のどれかが、1手で `pos` に移動できれば `True`(=`pos` にいる相手の駒を取れる)
- **内部処理:**
  1. `board.pieces_of(by_player)` の各 `(p, piece)` について
  2. 　`piece.get_move_offsets()` の各 `(dr, dc)` について
  3. 　　`p.offset(dr, dc) == pos` となるものがあれば `True`
  4. 最後まで見つからなければ `False`
- **持ち駒を打つ手は考慮しない**(打つ手では駒を取れないため)。
- `pos` に何があるか(空か、味方か)は調べない。「`by_player` の駒の利きが `pos` に届くか」だけを返す。

### `judge_winner(self, board: Board, last_move: Move, next_player: Player) -> Optional[Player]`
- **引数:**
  - `board`:`last_move` を **反映した後** の盤面
  - `last_move`:今指された手
  - `next_player`:次の手番のプレイヤー(`last_move.player` の相手)
- **戻り値:** 勝者の `Player`。まだ決まらなければ `None`。
- **内部処理:** 次の順に調べ、最初に成立したものを返す。

  1. **キャッチ:** `last_move.captured` があり `last_move.captured.kind == "lion"` なら、`last_move.player` の勝ち。
  2. **トライの成立:** `pos = board.find_lion(next_player)` が `None` でなく、`pos.row == next_player.goal_row` なら、`next_player` の勝ち。
     - 意味:`next_player` のライオンが前の手でゴール段に入り、今の `last_move` で取られなかった。
  3. **トライの開始:** `pos = board.find_lion(last_move.player)` が `None` でなく、`pos.row == last_move.player.goal_row` で、かつ `is_attacked(board, pos, next_player)` が `False` なら、`last_move.player` の勝ち。
     - 意味:今ゴール段に入った(または居続けている)ライオンが、次の相手の手で取られない。
  4. どれも成立しなければ `None`

- **補足:**
  - ゴール段に入ったライオンが取られる状態なら 3 は成立せず `None`。次に相手が取れば、その手の判定で 1(キャッチ)が成立する。相手が取らなければ、次の判定の 2 で勝ちになる。
  - 2 を 3 より先に調べるのは、「先に入っていたライオンが生き残った」という事実を優先するため。
  - 初期配置のライオンは自陣の最奥の段(先手 row 3、後手 row 0)にいて、**自分のゴール段ではない** ので、開始直後に誤判定されない。

## 6. 使われ方

| 呼び出し元 | 使うメソッド | 目的 |
|---|---|---|
| `Game.play_turn` | `legal_moves` | 手番のプレイヤーに提示する手を求める |
| `Game.play_turn` | `is_legal` | `Controller` が返した手の検査 |
| `Game.play_turn` | `judge_winner` | 手を反映した後に勝敗を判定する |
| `Controller`(Human/AI) | (`legal_moves` の結果を受け取るのみ) | |

```python
rule = Rule()
moves = rule.legal_moves(board, first)
winner = rule.judge_winner(board, last_move, second)   # last_move は first の手
```

## 7. テスト項目

### 7.1 `legal_moves`
先手 `first`、後手 `second` を用意し、盤面は `Board` に手で駒を置いて作る。

| # | 盤面 | 手番 | 期待結果 |
|---|---|---|---|
| 1 | 初期配置(先手:キリン(3,0) ライオン(3,1) ゾウ(3,2) ヒヨコ(2,1)、後手:ゾウ(0,0) ライオン(0,1) キリン(0,2) ヒヨコ(1,1)) | 先手 | 4手:ライオン(3,1)→(2,0)、ライオン(3,1)→(2,2)、キリン(3,0)→(2,0)、ヒヨコ(2,1)→(1,1)(これは後手のヒヨコを取る手で `captured.kind == "chick"`) |
| 2 | 同上 | 後手 | 4手(先手と点対称) |
| 3 | 先手のライオン (3,1) のみ、後手のライオン (0,1) | 先手 | 5手:(2,0)(2,1)(2,2)(3,0)(3,2) |
| 4 | 先手のライオン (0,0) のみ(他に駒なし) | 先手 | 3手:(0,1)(1,0)(1,1)(盤外を含まない) |
| 5 | 先手のライオン (3,1)、同じ盤の (2,1) に先手のヒヨコ | 先手 | ライオンの手に (2,1) が含まれない。ヒヨコ→(1,1) がある |
| 6 | 先手のヒヨコ (1,1)、(0,1) は空 | 先手 | ヒヨコ→(0,1) の手が `promote=True` |
| 7 | 先手のヒヨコ (2,1) | 先手 | ヒヨコ→(1,1) の手が `promote=False` |
| 8 | 先手のニワトリ (1,1) | 先手 | `row == 0` に進む手があっても `promote=False` |
| 9 | 先手のゾウ (3,2) のみ | 先手 | 1手:(2,1) |
| 10 | 後手のヒヨコ (2,1) | 後手 | 1手:(3,1)、`promote=True` |
| 11 | 2頭のライオンのみ((3,1) と (0,1))、先手の持ち駒:ヒヨコ1枚 | 先手 | 移動5手+打つ手10手=15手。打つ手は `from_pos is None`、`is_drop` が `True` |
| 12 | 上の盤で、持ち駒がヒヨコ2枚 | 先手 | 15手(同種の重複を作らない) |
| 13 | 上の盤で、持ち駒がヒヨコとゾウ | 先手 | 5 + 10 + 10 = 25手 |
| 14 | 持ち駒を打つ手の `to_pos` | | 駒のあるマスを含まない |
| 15 | 呼び出し前後で `board` と `player.hand` | | 変化しない |
| 16 | 同じ盤面で2回呼ぶ | | 同じ順序の同じ結果 |
| 17 | 相手の駒を取る手 | | `captured` に相手の駒がある |
| 18 | 後手のライオンが先手のライオンを取れる位置 | 後手 | 先手のライオンを取る手が含まれる(`captured.kind == "lion"`) |
| 19 | ライオンが取られる位置に動く手 | | 除外されない(王手の概念がない) |

### 7.2 `is_legal`

| # | 操作 | 期待結果 |
|---|---|---|
| 20 | `legal_moves` の各要素 | すべて `True` |
| 21 | 盤外へ出る手を手作りした `Move` | `False` |
| 22 | 味方の駒がいるマスへの手 | `False` |
| 23 | 持っていない駒を打つ手 | `False` |
| 24 | 駒のあるマスに打つ手 | `False` |
| 25 | 打つ手を、持ち駒と同種の別オブジェクトのヒヨコで作る | `True`(`key` で比べるため) |

### 7.3 `is_attacked`

| # | 盤面 | 期待結果 |
|---|---|---|
| 26 | 後手のキリン (1,1) | `(2,1)` `(0,1)` `(1,0)` `(1,2)` が `True`、`(2,0)` `(0,0)` は `False`(`by_player=second`) |
| 27 | 後手のヒヨコ (1,1) | `(2,1)` のみ `True`(後手は下へ進む)、`(0,1)` は `False` |
| 28 | 先手のヒヨコ (2,1) | `(1,1)` のみ `True` |
| 29 | 後手のゾウ (1,1) | `(0,0)` `(0,2)` `(2,0)` `(2,2)` が `True`、`(0,1)` は `False` |
| 30 | 攻撃側に駒がない | すべて `False` |
| 31 | 攻撃対象のマスが空でも | 利きが届けば `True`(空かどうかは見ない) |
| 32 | 持ち駒を持っているだけ | `False`(打つ手は考えない) |

### 7.4 `judge_winner`
各ケースで、手を **反映した後の** 盤面を手で作ってから呼ぶ。

| # | 盤面(反映後) | `last_move` / `next_player` | 期待結果 |
|---|---|---|---|
| 33 | `captured` が後手のライオン | 先手の手 / 後手 | 先手 |
| 34 | `captured` が先手のライオン | 後手の手 / 先手 | 後手 |
| 35 | `captured` がヒヨコ、ライオンは両者とも自陣 | 先手の手 / 後手 | `None` |
| 36 | 先手のライオン (0,0)、後手のライオン (1,2)(利きは (0,0) に届かない) | 先手の手(ライオン (1,0)→(0,0))/ 後手 | 先手(トライ成立) |
| 37 | 先手のライオン (0,0)、後手のライオン (1,1)(利きが (0,0) に届く) | 同上 | `None`(トライ不成立) |
| 38 | 先手のライオン (0,0)、後手のキリン (0,1)(利きが (0,0) に届く)、後手のライオンは別の場所 | 同上 | `None` |
| 39 | 先手のライオン (1,1)、後手のライオン (3,0)(後手のゴール段にいる) | 先手の手(取らない)/ 後手 | 後手(後手のライオンが生き残った) |
| 40 | 先手のライオン (3,1)、後手のライオン (0,1)(両者とも自陣) | 先手の手 / 後手 | `None`(初期のライオンの位置で誤判定しない) |
| 41 | 後手のライオン (3,2)、先手のライオン (2,1)(後手が前の手でゴール段に入った) | 先手の手(取らない)/ 後手 | 後手 |
| 42 | 後手のライオンが (3,2)、先手のゾウ (2,1) が (3,2) を取れる配置、先手が別の手を指した | 先手の手 / 後手 | 後手(取れても取らなければ生き残り) |
| 43 | 呼び出し前後の `board` | | 変化しない |

## 8. 実装上の注意
- **盤面や持ち駒を変更しない。** `legal_moves` の中で `board.set` や `player.hand` の変更をしない。
- 先手・後手の向きの反転は `Piece.get_move_offsets()` に任せる。`Rule` に `if player.direction == ...` を書かない(ゴール段は `player.goal_row` を使う)。
- 盤外判定は `board.in_bounds(to)` を使う。リストを直接触らない。
- 王手判定を実装しない。
- `judge_winner` の順序(キャッチ → 相手のトライ成立 → 自分のトライ開始)を守る。
- 手の同一性の比較は `Move.key` で行う。
- 担当が多い場合は、`legal_moves` と `is_legal` を一人、`is_attacked` と `judge_winner` をもう一人で分担できる(`judge_winner` が `is_attacked` を使う)。
