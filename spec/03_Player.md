# 仕様書 03: Player(プレイヤー)

- 実装ファイル:`dobutsu/player.py`
- テストファイル:`tests/test_player.py`

## 1. 役割
対局者1人分の情報を保持する。

- 名前(表示用)
- 前方向(先手か後手かの区別。駒の移動方向とゴール段の決定に使う)
- 持ち駒

盤面や他のプレイヤーは知らない。ルール判定も行わない。

## 2. 前提知識
- 先手は盤の下側から上へ進み、`direction = -1`。後手は上側から下へ進み、`direction = +1`。
- ゴール段(ヒヨコが成る段、ライオンのトライの段)は、先手が row 0、後手が row 3。
- 持ち駒は取った駒の置き場所。**持ち駒に入る駒は、所有者がこのプレイヤーで、ニワトリではなくヒヨコ**でなければならない。所有者の変更とニワトリの戻しは呼び出し側(`Game.apply_move`)が `Piece.with_owner` で行ってから渡す。
- 同じ種類の持ち駒を複数持つことがある(例:ヒヨコ2枚)が、実際のゲームでは同種は最大2枚。

## 3. 依存するもの
- `Piece`:持ち駒の型。**型ヒントのためだけに参照する**。使う属性は `kind`、`owner`。
- `dobutsu.position.BOARD_ROWS`:ゴール段の計算に使う。

```python
from __future__ import annotations
from typing import TYPE_CHECKING
from dobutsu.position import BOARD_ROWS
if TYPE_CHECKING:
    from dobutsu.pieces import Piece
```

## 4. 属性

| 名前 | 型 | 説明 |
|---|---|---|
| `name` | `str` | 表示名。例:`"先手"` `"後手"` |
| `direction` | `int` | 前方向の `dr`。先手 `-1`、後手 `+1` |
| `hand` | `list[Piece]` | 持ち駒。初期値は空リスト |

プロパティ:

| 名前 | 型 | 説明 |
|---|---|---|
| `goal_row` | `int` | ゴール段の行番号 |

## 5. メソッド仕様

### `__init__(self, name: str, direction: int) -> None`
- **引数:**
  - `name`:表示名
  - `direction`:`-1`(先手)または `+1`(後手)
- **内部処理:**
  1. `direction` が `-1` でも `+1` でもなければ `ValueError`
  2. `self.name`、`self.direction` を設定する
  3. `self.hand = []` とする(**インスタンスごとに新しいリストを作る**)
- **例外:** `direction` が不正なとき `ValueError`

### `goal_row` (プロパティ)
- **戻り値:** 先手(`direction == -1`)なら `0`、後手(`direction == +1`)なら `BOARD_ROWS - 1`
- **内部処理:** `direction` で分岐して返す。

### `add_to_hand(self, piece: Piece) -> None`
- **引数:** `piece`:持ち駒に加える駒
- **戻り値:** なし
- **内部処理:**
  1. `piece.owner is not self` なら `ValueError`
  2. `piece.kind == "hen"` なら `ValueError`
  3. `self.hand.append(piece)`
- **例外:** 上記の2条件で `ValueError`

### `remove_from_hand(self, kind: str) -> Piece`
- **引数:** `kind`:取り出す駒の種類(`"chick"` `"elephant"` `"giraffe"`)
- **戻り値:** 取り出した駒(`hand` から取り除かれる)
- **内部処理:**
  1. `hand` を先頭から調べ、`piece.kind == kind` の最初の駒を探す
  2. 見つかったら `hand` から取り除いて返す
  3. 見つからなければ `ValueError`
- **例外:** 該当する駒がないとき `ValueError`

### `peek_hand(self, kind: str) -> Piece`
- **引数:** `kind`:駒の種類
- **戻り値:** `hand` の中の、その種類の最初の駒。**`hand` からは取り除かない。**
- **内部処理:** `remove_from_hand` と同じ探し方で、取り除かずに返す。
- **例外:** 該当する駒がないとき `ValueError`
- 用途:`Rule.legal_moves` が「打つ手」の `Move.piece` を作るときに使う。

### `has_in_hand(self, kind: str) -> bool`
- **引数:** `kind`:駒の種類
- **戻り値:** その種類の持ち駒が1枚以上あれば `True`
- **内部処理:** `hand` のどれかが `piece.kind == kind` かを調べる。

### `hand_kinds(self) -> list[str]`
- **引数:** なし
- **戻り値:** 持ち駒の種類の一覧。**重複を除き、`hand` 内で最初に現れた順**。
- **内部処理:** `hand` を先頭から見て、まだ出ていない `kind` を順に集める。
- 例:`hand` が `[chick, elephant, chick]` なら `["chick", "elephant"]`。

### `__repr__(self) -> str`
- デバッグ用。例:`Player(先手)`。テストの失敗表示が読みやすくなる。

## 6. 使われ方

| 呼び出し元 | 使うもの | 目的 |
|---|---|---|
| `Game.__init__` / `setup` | `Player("先手", -1)`, `Player("後手", 1)`, `hand.clear()` | プレイヤー生成、再初期化 |
| `Game.apply_move` | `add_to_hand`, `remove_from_hand` | 駒を取る、打つ |
| `Rule.legal_moves` | `hand_kinds`, `peek_hand`, `goal_row` | 打つ手の生成、成りの判定 |
| `Rule.judge_winner` | `goal_row` | トライの判定 |
| `Piece.get_move_offsets` | `direction` | 後手の移動方向の反転 |
| `View.show_board` | `name`, `hand`, `direction` | 表示 |

```python
first = Player("先手", -1)
first.goal_row            # 0
taken = Chick(second).with_owner(first)
first.add_to_hand(taken)
first.hand_kinds()        # ["chick"]
first.remove_from_hand("chick")   # taken が返り、hand は空になる
```

## 7. テスト項目
`Piece` が未実装の場合は、`kind` と `owner` 属性だけを持つダミークラスを使ってよい。

| # | 操作 | 期待結果 |
|---|---|---|
| 1 | `p = Player("先手", -1)` | `p.name == "先手"`, `p.direction == -1`, `p.hand == []` |
| 2 | `Player("x", 0)` | `ValueError` |
| 3 | `Player("x", 2)` | `ValueError` |
| 4 | `Player("先手", -1).goal_row` | `0` |
| 5 | `Player("後手", 1).goal_row` | `3` |
| 6 | 2つのプレイヤーの `hand` | 別々のリスト(片方に追加しても他方に影響しない) |
| 7 | 自分が所有者のヒヨコを `add_to_hand` | `hand` に1枚入る |
| 8 | 他人が所有者の駒を `add_to_hand` | `ValueError` |
| 9 | ニワトリを `add_to_hand` | `ValueError` |
| 10 | ヒヨコ・ゾウ・ヒヨコを順に追加後 `hand_kinds()` | `["chick", "elephant"]` |
| 11 | 上の状態で `has_in_hand("chick")` | `True` |
| 12 | 上の状態で `has_in_hand("giraffe")` | `False` |
| 13 | 上の状態で `peek_hand("chick")` | 最初に追加したヒヨコ、`hand` は3枚のまま |
| 14 | 上の状態で `remove_from_hand("chick")` | 最初に追加したヒヨコが返り、`hand` は2枚(`[ゾウ, ヒヨコ]`) |
| 15 | 持っていない種類を `remove_from_hand` | `ValueError` |
| 16 | 持っていない種類を `peek_hand` | `ValueError` |
| 17 | 空の `hand` で `hand_kinds()` | `[]` |

## 8. 実装上の注意
- `hand = []` を **クラス変数やデフォルト引数にしない**(2人のプレイヤーで共有されてしまう)。`__init__` の中で作る。
- `__eq__` を定義しない(`Player` は同一性で比較される。`Game` が辞書のキーにも使う)。
- `add_to_hand` は、渡された駒が正しい状態かを検査するだけで、所有者の変更やニワトリの戻しは行わない(`Game` の責務)。
