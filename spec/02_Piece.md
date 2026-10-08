# 仕様書 02: Piece と各駒(Lion / Elephant / Giraffe / Chick / Hen)

- 実装ファイル:`dobutsu/pieces.py`
- テストファイル:`tests/test_pieces.py`

## 1. 役割
駒1枚を表す。駒は次の3つだけを知っている。

- 自分の **種類**(`kind`)と **表示用の1文字**(`symbol`)
- 自分の **所有者**(`owner`:どちらのプレイヤーの駒か)
- 自分の **移動方向**(1手でどの方向に動けるか)

**盤面の状態は知らない。** 盤外に出るか、味方の駒にぶつかるか、といった判定は `Rule` が行う。

## 2. 前提知識

### 移動方向の表し方
移動方向は `(dr, dc)` の組で表す。`dr` が行方向、`dc` が列方向の変化量。

- 先手は上(row が減る方向)に進むので、先手にとっての「前」は `dr = -1`。
- 後手は下(row が増える方向)に進むので、後手にとっての「前」は `dr = +1`。
- `Player.direction` が先手 `-1`、後手 `+1` である(前方向の `dr`)。

### 各駒の移動方向(先手基準)

| 駒 | クラス | `kind` | `symbol` | 動ける方向 `(dr, dc)` | 個数 |
|---|---|---|---|---|---|
| ライオン | `Lion` | `"lion"` | 獅 | (-1,-1) (-1,0) (-1,1) (0,-1) (0,1) (1,-1) (1,0) (1,1) | 8 |
| ゾウ | `Elephant` | `"elephant"` | 象 | (-1,-1) (-1,1) (1,-1) (1,1) | 4 |
| キリン | `Giraffe` | `"giraffe"` | 麒 | (-1,0) (1,0) (0,-1) (0,1) | 4 |
| ヒヨコ | `Chick` | `"chick"` | 雛 | (-1,0) | 1 |
| ニワトリ | `Hen` | `"hen"` | 鶏 | (-1,-1) (-1,0) (-1,1) (0,-1) (0,1) (1,0) | 6 |

### 後手のときの変換
後手の駒は、先手基準の方向の **`dr` の符号だけを反転** して使う(`dc` はそのまま)。
どの駒も左右対称なので `dc` を反転しても結果は同じであり、反転しない。

例:後手のニワトリは `(1,-1) (1,0) (1,1) (0,-1) (0,1) (-1,0)`。

### 所有者の変更と成り
- 駒を取ったとき、その駒は取った側の持ち駒になる。**駒の属性は書き換えず、所有者を変えた新しい駒を作る**(`with_owner`)。
- ニワトリは、取られると **ヒヨコに戻って** 持ち駒になる。
- ヒヨコがゴール段に入ったとき、ニワトリに成る(`Chick.promote`)。

## 3. 依存するもの
- `Player`:所有者の型。**型ヒントのためだけに参照する**。`Piece` が必要とする `Player` の属性は `direction`(`int`)のみ。
- 循環参照を避けるため、`player.py` の import は次のように書く。

```python
from __future__ import annotations
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from dobutsu.player import Player
```

`Player` が未完成でも、`direction` 属性を持つ簡単なダミークラスでテストできる。

## 4. クラス構成

```
Piece (抽象基底クラス, abc.ABC)
├── Lion
├── Elephant
├── Giraffe
├── Chick    … promote() を追加
└── Hen      … demote() を追加
```

## 5. Piece(抽象基底クラス)

### 属性

| 名前 | 種類 | 型 | 説明 |
|---|---|---|---|
| `kind` | **クラス属性** | `str` | 駒の種類。サブクラスで定義(`"lion"` など) |
| `symbol` | **クラス属性** | `str` | 表示用1文字。サブクラスで定義(`"獅"` など) |
| `owner` | インスタンス属性 | `Player` | この駒の所有者 |

### `__init__(self, owner: Player) -> None`
- **引数:** `owner`:この駒の所有者
- **内部処理:** `self.owner = owner` を設定する。
- **例外:** なし

### `base_offsets(self) -> list[tuple[int, int]]` (抽象メソッド)
- **引数:** なし
- **戻り値:** **先手基準** の移動方向のリスト(2節の表の通り)
- **内部処理:** サブクラスが表の通りのリストを返す。**呼ぶたびに新しいリストを返す**(戻り値を呼び出し側が変更しても影響が出ないようにするため)。
- `@abstractmethod` を付ける。`Piece` を直接生成しようとすると `TypeError` になる。

### `get_move_offsets(self) -> list[tuple[int, int]]`
- **引数:** なし
- **戻り値:** 所有者の向きに合わせた移動方向のリスト
- **内部処理:**
  1. `self.base_offsets()` を取得する
  2. 所有者が先手(`self.owner.direction == -1`)ならそのまま返す
  3. 所有者が後手(`self.owner.direction == +1`)なら、各要素 `(dr, dc)` を `(-dr, dc)` に変換して返す
  - 一般式:`(dr * -self.owner.direction, dc)`(先手は ×1、後手は ×(-1))
- **例外:** なし
- **この反転処理はプログラム全体でここだけに書く。** `Rule` などでは反転しない。

### `with_owner(self, player: Player) -> Piece`
- **引数:** `player`:新しい所有者
- **戻り値:** 所有者が `player` の **新しい駒**。元の駒は変更しない。
- **内部処理:**
  1. `type(self)(player)` で、同じクラスの新しい駒を作って返す
  - `Hen` は例外で、`Chick(player)` を返す(下記)
- **例外:** なし

## 6. 各サブクラス

### Lion / Elephant / Giraffe
- `kind`・`symbol`を定義し、`base_offsets()` で2節の表の方向を返す。それ以外の処理はない。

### Chick
- `kind = "chick"`、`symbol = "雛"`、`base_offsets()` は `[(-1, 0)]`。
- 追加メソッド:

#### `promote(self) -> Hen`
- **引数:** なし
- **戻り値:** 同じ所有者の新しい `Hen`
- **内部処理:** `Hen(self.owner)` を返す。元のヒヨコは変更しない。

### Hen
- `kind = "hen"`、`symbol = "鶏"`、`base_offsets()` は `[(-1,-1), (-1,0), (-1,1), (0,-1), (0,1), (1,0)]`。
- 追加メソッド:

#### `demote(self) -> Chick`
- **引数:** なし
- **戻り値:** 同じ所有者の新しい `Chick`
- **内部処理:** `Chick(self.owner)` を返す。

#### `with_owner(self, player: Player) -> Chick` (上書き)
- 所有者を変えるとき **ヒヨコに戻す**(ニワトリを取るとヒヨコになるルール)。
- **内部処理:** `Chick(player)` を返す。

## 7. 使われ方

| 呼び出し元 | 使うメソッド | 目的 |
|---|---|---|
| `Rule.legal_moves` | `get_move_offsets()`, `kind`, `owner` | 移動先の候補を求める |
| `Rule.is_attacked` | `get_move_offsets()` | 駒が指定マスを取れるか調べる |
| `Game.apply_move` | `with_owner()`, `promote()` | 取った駒を持ち駒にする、成る |
| `Game.setup` | `Lion(player)` など | 初期配置の駒を作る |
| `View.show_board` | `symbol`, `owner` | 表示 |

```python
chick = Chick(first)          # first は先手 Player
chick.get_move_offsets()      # [(-1, 0)]
hen = chick.promote()         # 先手の Hen
taken = hen.with_owner(second)  # 後手の Chick（ヒヨコに戻る）
```

## 8. テスト項目
先手 `first`(`direction=-1`)、後手 `second`(`direction=+1`)を用意する。

| # | 操作 | 期待結果 |
|---|---|---|
| 1 | `Piece(first)` | `TypeError`(抽象クラスは生成できない) |
| 2 | `Lion.kind`, `Lion.symbol` | `"lion"`, `"獅"`(インスタンス不要) |
| 3 | 各駒の `kind` / `symbol` が2節の表と一致する | 5種すべて |
| 4 | `len(Lion(first).get_move_offsets())` | 8 |
| 5 | `len(Elephant(first).get_move_offsets())` | 4 |
| 6 | `len(Giraffe(first).get_move_offsets())` | 4 |
| 7 | `len(Chick(first).get_move_offsets())` | 1 |
| 8 | `len(Hen(first).get_move_offsets())` | 6 |
| 9 | `Chick(first).get_move_offsets()` | `[(-1, 0)]` |
| 10 | `Chick(second).get_move_offsets()` | `[(1, 0)]` |
| 11 | `set(Hen(first).get_move_offsets())` | `{(-1,-1),(-1,0),(-1,1),(0,-1),(0,1),(1,0)}` |
| 12 | `set(Hen(second).get_move_offsets())` | `{(1,-1),(1,0),(1,1),(0,-1),(0,1),(-1,0)}` |
| 13 | `set(Elephant(second).get_move_offsets())` | 先手と同じ4方向の集合 |
| 14 | `Chick(first).promote()` | `Hen` のインスタンス、`owner is first` |
| 15 | `chick = Chick(first); chick.promote()` の後の `chick` | `Chick` のまま(元の駒は不変) |
| 16 | `Hen(first).demote()` | `Chick` のインスタンス、`owner is first` |
| 17 | `Elephant(first).with_owner(second)` | `Elephant`、`owner is second` |
| 18 | `e = Elephant(first); e.with_owner(second)` の後の `e.owner` | `first`(元の駒は不変) |
| 19 | `Hen(first).with_owner(second)` | `Chick`、`owner is second` |
| 20 | `offsets = Lion(first).base_offsets(); offsets.clear()` の後 `Lion(first).base_offsets()` | 8要素(呼び出しごとに新しいリスト) |

## 9. 実装上の注意
- `kind` と `symbol` は **クラス属性** にする(インスタンスを作らなくても参照でき、表示や判定で便利)。
- `owner` は作成後に書き換えない。所有者が変わるときは `with_owner` で新しい駒を作る。
- 先手・後手の判定に `owner.name` を使わない。`owner.direction` を使う。
- `Hen` の `with_owner` のみ戻り値の型が異なる。この挙動を忘れると、持ち駒にニワトリが入ってしまう。
