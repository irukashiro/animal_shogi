# 仕様書 07: Game(対局の進行)と main.py

- 実装ファイル:`dobutsu/game.py`、`main.py`
- テストファイル:`tests/test_game.py`

## 1. 役割
1局の **進行を管理** する。他のクラスを組み合わせて動かす、全体の指揮役。

- 盤面とプレイヤーを生成し、初期配置を行う。
- 手番ごとに「表示 → 合法手の取得 → 手の選択 → 盤面への反映 → 勝敗判定 → 手番交代」を繰り返す。
- 手(`Move`)を盤面と持ち駒に **反映する処理(`apply_move`)** を持つ。

ルール判定は `Rule`、表示は `View`、入力は `Controller` に任せ、`Game` 自身はルールの詳細を持たない。

## 2. 前提知識

### 座標と向き
- 盤面は 4行×3列、`Position(row, col)`。先手は下側(row 3 側)から上へ進み `direction = -1`、後手は上側から下へ進み `direction = +1`。

### 初期配置

| 先手の駒 | 座標 | 後手の駒 | 座標 |
|---|---|---|---|
| `Giraffe` | (3,0) | `Elephant` | (0,0) |
| `Lion` | (3,1) | `Lion` | (0,1) |
| `Elephant` | (3,2) | `Giraffe` | (0,2) |
| `Chick` | (2,1) | `Chick` | (1,1) |

先手が最初に指す。

### 手(Move)の反映規則
`Move` の内容 → 盤面・持ち駒への反映:

| 手の種類 | 条件 | 反映内容 |
|---|---|---|
| 取る | `move.captured` がある | `to_pos` の駒を盤から外し、`captured.with_owner(player)` を `player` の持ち駒に加える |
| 移動 | `move.from_pos` がある | `from_pos` の駒を盤から外し、`to_pos` に置く |
| 成る | `move.promote` が `True` | 置く駒を `piece.promote()`(ニワトリ)に替える |
| 打つ | `move.from_pos is None` | `player` の持ち駒から `piece.kind` の駒を取り出して `to_pos` に置く |

取る処理を移動より先に行う(移動先を空けるため)。

### 使うクラスのインターフェース(他担当の仕様)

| クラス | 使うメソッド |
|---|---|
| `Board` | `Board()`, `set(pos, piece)`, `remove(pos)` |
| `Player` | `Player(name, direction)`, `hand`, `add_to_hand(piece)`, `remove_from_hand(kind)` |
| `Piece` | `with_owner(player)`, `Chick.promote()`, `Lion(player)` など |
| `Rule` | `legal_moves(board, player)`, `is_legal(board, player, move)`, `judge_winner(board, last_move, next_player)` |
| `View` | `show_board(board, players, current)`, `show_message(text)`, `show_result(winner)` |
| `Controller` | `choose_move(board, player, legal_moves)` |
| `Move` | 属性 `player` `piece` `from_pos` `to_pos` `captured` `promote`、`is_drop` |

## 3. 依存するもの
全クラス。import は次の通り。

```python
from dobutsu.position import Position
from dobutsu.pieces import Lion, Elephant, Giraffe, Chick
from dobutsu.player import Player
from dobutsu.board import Board
from dobutsu.rule import Rule
from dobutsu.view import View
from dobutsu.controller import Controller
from dobutsu.move import Move
```

## 4. 属性

| 名前 | 型 | 説明 |
|---|---|---|
| `board` | `Board` | 盤面 |
| `first` | `Player` | 先手(`Player("先手", -1)`) |
| `second` | `Player` | 後手(`Player("後手", 1)`) |
| `players` | `list[Player]` | `[first, second]`(`View.show_board` に渡す) |
| `controllers` | `dict[Player, Controller]` | プレイヤーごとの `Controller`。`Player` は同一性でハッシュされるので辞書のキーにできる |
| `view` | `View` | 表示 |
| `rule` | `Rule` | ルール判定 |
| `current_player` | `Player` | 手番 |
| `winner` | `Optional[Player]` | 勝者。終了前は `None` |
| `history` | `list[Move]` | 指した手の記録(古い順) |
| `is_draw` | `bool` | 引き分けで終了したか(通常は `False`。千日手を実装する場合のみ使う) |

## 5. メソッド仕様

### `__init__(self, first_controller: Controller, second_controller: Controller, view: View, rule: Rule) -> None`
- **引数:**
  - `first_controller`:先手を操作する `Controller`
  - `second_controller`:後手を操作する `Controller`
  - `view`:表示を担当する `View`
  - `rule`:ルール判定の `Rule`
- **内部処理:**
  1. `self.first = Player("先手", -1)`、`self.second = Player("後手", 1)` を作る
  2. `self.players = [self.first, self.second]`
  3. `self.controllers = {self.first: first_controller, self.second: second_controller}`
  4. `self.view = view`、`self.rule = rule`
  5. `self.setup()` を呼ぶ

### `setup(self) -> None`
- **引数・戻り値:** なし
- **内部処理(再呼び出しで何度でも初期状態に戻せること):**
  1. `self.board = Board()` を新しく作る
  2. 両プレイヤーの `hand` を空にする(`hand.clear()`)
  3. 2節の初期配置の通りに `board.set` で8枚の駒を置く(先手の駒は `self.first` を、後手の駒は `self.second` を所有者にする)
  4. `self.current_player = self.first`
  5. `self.winner = None`、`self.history = []`、`self.is_draw = False`

### `opponent(self, player: Player) -> Player`
- **引数:** `player`
- **戻り値:** `player` の相手のプレイヤー(`first` なら `second`、`second` なら `first`)
- **内部処理:** `self.second if player is self.first else self.first`

### `apply_move(self, move: Move) -> None`
- **引数:** `move`:合法であることが確認済みの手
- **戻り値:** なし。`board` と `move.player.hand` を更新する。
- **内部処理:**
  1. `player = move.player`
  2. **取る処理:** `move.captured` がある場合
     1. `taken = self.board.remove(move.to_pos)`
     2. `player.add_to_hand(taken.with_owner(player))`
        - `with_owner` で所有者が変わり、ニワトリはヒヨコに戻る
  3. **置く駒の決定と元の位置の処理:**
     - 打つ手(`move.is_drop`)のとき:`piece = player.remove_from_hand(move.piece.kind)`
     - 移動のとき:`piece = self.board.remove(move.from_pos)`
  4. **成る処理:** `move.promote` が `True` なら `piece = piece.promote()`
  5. `self.board.set(move.to_pos, piece)`
- **例外:** 通常は発生しない(`Rule` が合法手のみを渡すため)。`Board` / `Player` が出す `ValueError` は握りつぶさず、そのまま伝える。

### `play_turn(self) -> None`
- **引数・戻り値:** なし
- **内部処理:**
  1. `self.view.show_board(self.board, self.players, self.current_player)`
  2. `moves = self.rule.legal_moves(self.board, self.current_player)`
  3. `moves` が空なら、相手の勝ちとして終了:`self.winner = self.opponent(self.current_player)` を設定して `return`
  4. `controller = self.controllers[self.current_player]`
  5. `move = controller.choose_move(self.board, self.current_player, moves)`
  6. `self.rule.is_legal(self.board, self.current_player, move)` が `False` なら `ValueError`(`Controller` の不具合を検出する)
  7. `self.apply_move(move)`、`self.history.append(move)`
  8. `next_player = self.opponent(self.current_player)`
  9. `self.winner = self.rule.judge_winner(self.board, move, next_player)`
  10. `self.winner` が決まったら `return`(手番は交代しない)
  11. 決まっていなければ `self.switch_player()`

### `switch_player(self) -> None`
- **内部処理:** `self.current_player = self.opponent(self.current_player)`

### `is_finished(self) -> bool`
- **戻り値:** `self.winner is not None or self.is_draw`

### `run(self) -> Optional[Player]`
- **戻り値:** 勝者。引き分けなら `None`。
- **内部処理:**
  1. `self.is_finished()` が `False` の間 `self.play_turn()` を繰り返す
  2. ループ終了後、`self.view.show_board(self.board, self.players, self.current_player)` で最終盤面を表示する
  3. `self.view.show_result(self.winner)`
  4. `self.winner` を返す
- **補足:** 最終盤面を表示するときの `current_player` は、勝者がライオンを取った/トライした時点の手番(勝者)のままでよい。

## 6. main.py の仕様

- **役割:** 部品を組み立てて対局を開始する。ロジックは書かない。
- **内部処理:**
  1. `View()` を生成する
  2. 先手・後手がそれぞれ人間か AI かを決める(最初は両方 `HumanController(view)` でよい。選択式にするなら標準入力で `1:人間 2:AI` を尋ねる)
  3. `Game(first_controller, second_controller, view, Rule())` を生成する
  4. `game.run()` を呼ぶ

```python
from dobutsu.game import Game
from dobutsu.rule import Rule
from dobutsu.view import View
from dobutsu.controller import HumanController

def main() -> None:
    view = View()
    game = Game(HumanController(view), HumanController(view), view, Rule())
    game.run()

if __name__ == "__main__":
    main()
```

## 7. 使われ方

| 呼び出し元 | 使うもの |
|---|---|
| `main.py` | `Game(...)`、`run()` |
| テスト | `setup()`、`apply_move()`、`play_turn()` |

## 8. テスト項目
`Game` のテストでは、`View` と `Controller` に **テスト用の偽物(スタブ)** を使う。

- `StubView`:`show_board` `show_message` `show_result` が何もしない(呼ばれた引数を記録してもよい)。
- `ScriptedController`:あらかじめ渡した `Move` のリストを順に返す。または、条件に合う手を `legal_moves` から選んで返す関数を渡せる。

### 8.1 初期状態(`setup`)

| # | 操作 | 期待結果 |
|---|---|---|
| 1 | `Game(...)` 直後の `board.get(Position(3,1))` | 先手の `Lion` |
| 2 | 初期配置の8座標 | 2節の表の通りの種類と所有者(先手の駒の `owner is game.first`) |
| 3 | 上記以外のマス(4マス) | 空 |
| 4 | `current_player is game.first`、`winner is None`、`history == []` | `True` |
| 5 | 両プレイヤーの `hand` | `[]` |
| 6 | 手を何手か指した後 `setup()` を呼ぶ | 初期状態に戻る(盤・持ち駒・履歴・手番) |

### 8.2 `apply_move`

| # | 操作 | 期待結果 |
|---|---|---|
| 7 | 先手のヒヨコ (2,1)→(1,1) を取る手(後手のヒヨコを取る) | `(2,1)` が空、`(1,1)` に先手のヒヨコ、先手の `hand` に **所有者が先手の** ヒヨコ1枚 |
| 8 | 上の後、後手の盤上の駒数 | 3枚(ヒヨコが減る) |
| 9 | 通常の移動(取らない) | 移動元が空、移動先に同じ駒、持ち駒は変化なし |
| 10 | ヒヨコがゴール段に入る手(`promote=True`) | 移動先の駒が `Hen`、所有者は同じ |
| 11 | ニワトリを取る手 | 取った側の `hand` に **`Chick`**(ニワトリではない)が入る |
| 12 | 持ち駒のヒヨコを打つ手 | 盤上の `to_pos` にヒヨコ、`hand` からヒヨコが1枚減る |
| 13 | 同種を2枚持った状態で1枚打つ | `hand` に1枚残る |

### 8.3 `play_turn` / `run`

| # | 操作 | 期待結果 |
|---|---|---|
| 14 | 先手が普通の手を指して `play_turn()` | `current_player is game.second`、`len(history) == 1`、`winner is None` |
| 15 | `play_turn()` 中に `view.show_board` が呼ばれる | 1回 |
| 16 | 先手がライオンを取る手を指す局面 | `winner is game.first`、手番は交代しない、`is_finished()` が `True` |
| 17 | `Controller` が合法でない手を返す(スタブで再現) | `ValueError` |
| 18 | 合法手が0の局面(手作りした盤面) | `winner` が相手になる |
| 19 | `run()`:先手がライオンを取るまでの短い手順を `ScriptedController` で与える | 戻り値が先手、`show_result` が先手で1回呼ばれる |
| 20 | `run()` の終了後 | `is_finished()` が `True` |
| 21 | `run()` 完走時のトライによる終了(トライが成立する手順を与える) | 戻り値がトライした側 |

### 8.4 手順の例(テスト19用)
先手が2手で相手のヒヨコを取って成り…という長い手順を作る代わりに、**手作りの盤面** を `game.board` に直接組む方法を使う。

```python
game = Game(c1, c2, StubView(), Rule())
game.board = Board()                  # 空の盤に置き直す
game.board.set(Position(1, 1), Lion(game.first))
game.board.set(Position(0, 1), Lion(game.second))
# 先手がライオンで (0,1) を取る手を ScriptedController に与える
```

## 9. 実装上の注意
- `Game` は **ルールの詳細を書かない**。トライやキャッチの判定、合法手の生成は `Rule` を呼ぶ。
- `Game` は **表示・入力の詳細を書かない**。`print` や `input` を使わない(`View` / `Controller` の仕事)。
- `apply_move` の中の処理順を守る(取る → 移動元を空ける → 成る → 置く)。取る前に置くと、`Board.set` が相手の駒を上書きして持ち駒に入らない。
- 勝者が決まった手番では `switch_player` を呼ばない。`run()` 終了後の表示で、`current_player` が勝者となる。
- 千日手(引き分け)は任意課題。実装する場合は、盤面と持ち駒と手番の組を記録し、同一局面が3回現れたら `is_draw = True` とする。
- `ScriptedController` は `Controller` を継承したテスト用クラス。テストファイルの中に書いてよい。
