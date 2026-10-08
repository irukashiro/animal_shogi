# 仕様書 04: Move(指し手)

- 実装ファイル:`dobutsu/move.py`
- テストファイル:`tests/test_move.py`

## 1. 役割
「誰が、どの駒を、どこからどこへ動かす(または打つ)か」という **1手分のデータ** を表す。
判定や盤面の更新などの処理は持たない。`Rule` が作り、`Controller` が選び、`Game` が盤面に反映する。

## 2. 前提知識
- 手には2種類ある。
  - **移動:** 盤上の駒を動かす。`from_pos` は移動元の座標。
  - **打つ(ドロップ):** 持ち駒を空きマスに置く。`from_pos` は `None`。
- 移動先に相手の駒があれば、それを取る。取られる駒を `captured` に入れる。
- ヒヨコがゴール段に入る移動は、ニワトリに成る手であり `promote = True`。このとき `piece` は **成る前のヒヨコ** のまま。成る処理は `Game.apply_move` が行う。
- 座標は `Position(row, col)`、`row` が 0〜3、`col` が 0〜2。

## 3. 依存するもの
型ヒントのためだけに参照する。

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Optional, TYPE_CHECKING
if TYPE_CHECKING:
    from dobutsu.position import Position
    from dobutsu.pieces import Piece
    from dobutsu.player import Player
```

## 4. 属性
`@dataclass(frozen=True)` で定義する。属性の定義順は次の通り(順序を変えない)。

| 順 | 属性 | 型 | 既定値 | 説明 |
|---|---|---|---|---|
| 1 | `player` | `Player` | なし | 指すプレイヤー |
| 2 | `piece` | `Piece` | なし | 動かす駒(打つ場合は持ち駒の駒)。**成る手でも成る前の駒** |
| 3 | `from_pos` | `Optional[Position]` | なし | 移動元。**打つ手は `None`** |
| 4 | `to_pos` | `Position` | なし | 移動先(打つ場所) |
| 5 | `captured` | `Optional[Piece]` | `None` | 取る相手の駒。取らなければ `None` |
| 6 | `promote` | `bool` | `False` | 成る手なら `True` |

## 5. メソッド・プロパティ仕様

### `is_drop` (プロパティ)
- **戻り値:** 打つ手なら `True`(`from_pos is None`)、移動なら `False`
- **内部処理:** `return self.from_pos is None`

### `key` (プロパティ)
- **戻り値:** 手の同一性を判定するためのタプル `(player, piece.kind, from_pos, to_pos)`
- **内部処理:** `return (self.player, self.piece.kind, self.from_pos, self.to_pos)`
- **目的:** 打つ手では、同じ種類の持ち駒が複数あっても、どれを使っても同じ手とみなしたい。`Move` を `==` で比べると `piece` の同一性まで比べてしまうため、同じ手かどうかの判定には `key` を使う。
- `Rule.is_legal` と `HumanController` が使う。

### (任意)`__post_init__` による整合性検査
実装は任意。実装する場合は次の条件で `ValueError` を送出する。
- `promote` が `True` なのに `piece.kind != "chick"`
- 打つ手(`from_pos is None`)なのに `captured` がある、または `promote` が `True`
- `captured` があるのに `captured.owner is player`(味方の駒を取る手)

## 6. 使われ方

| 呼び出し元 | 使い方 |
|---|---|
| `Rule.legal_moves` | `Move(...)` を生成して返す |
| `Rule.is_legal` | `move.key` で合法手と比較する |
| `Rule.judge_winner` | `move.player`、`move.captured` を読む |
| `Controller.choose_move` | `legal_moves` の中から1つ選んで返す |
| `Game.apply_move` | `from_pos` `to_pos` `captured` `promote` `is_drop` を読んで盤面に反映する |
| `Game.history` | 指した手の記録として保持する |

```python
# 先手のヒヨコ (2,1) が後手のヒヨコを取って (1,1) へ移動
Move(first, chick, Position(2, 1), Position(1, 1), captured=enemy_chick)

# 先手がヒヨコを (2,0) に打つ
Move(first, chick_in_hand, None, Position(2, 0))

# 先手のヒヨコ (1,1) が (0,1) に入り、ニワトリに成る
Move(first, chick, Position(1, 1), Position(0, 1), promote=True)
```

## 7. テスト項目
`Player` `Piece` `Position` を用意する(未実装なら、必要な属性を持つダミーでよい)。

| # | 操作 | 期待結果 |
|---|---|---|
| 1 | `Move(first, chick, Position(2,1), Position(1,1))` | `captured is None`, `promote is False` |
| 2 | 上の `is_drop` | `False` |
| 3 | `Move(first, chick, None, Position(2,0))` の `is_drop` | `True` |
| 4 | `m.player = second` | `FrozenInstanceError`(frozen) |
| 5 | 同じ内容の2つの `Move`(同じ `chick` オブジェクトを使用)を `==` で比較 | `True` |
| 6 | 同じ内容だが `to_pos` が異なる2つを `==` で比較 | `False` |
| 7 | 打つ手を、同種の別のヒヨコオブジェクトで2つ作り、`a.key == b.key` | `True` |
| 8 | 上の `a == b` | `False`(`piece` の同一性が異なるため) |
| 9 | `key` の内容 | `(first, "chick", Position(2,1), Position(1,1))` |
| 10 | (`__post_init__` を実装した場合)ゾウで `promote=True` | `ValueError` |
| 11 | (`__post_init__` を実装した場合)打つ手で `captured` あり | `ValueError` |

## 8. 実装上の注意
- 属性の順序を仕様通りにする。`from_pos` が `None` を取れても、既定値は付けない(`to_pos` の前にあるため。既定値のない属性を既定値のある属性の後に置くとエラーになる)。
- `Move` は **データのみ** を持つ。盤面を見て判定する処理を入れない。
- `promote=True` の手でも `piece` はヒヨコのまま。ニワトリに替える処理を `Move` に入れない。
