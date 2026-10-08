# 仕様書 01: Position(座標)

- 実装ファイル:`dobutsu/position.py`
- テストファイル:`tests/test_position.py`

## 1. 役割
盤面上の1マスの位置(行・列)を表す **不変の値オブジェクト**。盤の大きさ(4行×3列)の定義もここに置く。
他のほぼすべてのクラスが使う、最も基礎的なクラス。

## 2. 前提知識
- 盤面は 4行×3列。`row` は 0〜3、`col` は 0〜2。左上が `(0, 0)`。
- 先手は row が減る方向へ、後手は row が増える方向へ進む。このクラス自体は先手・後手を知らない。
- 盤の外の座標も **オブジェクトとしては作れる**(例:`Position(-1, 0)`)。盤内かどうかは `is_valid()` で判定する。これは、駒の移動先を計算してから盤外を除外する、という使い方のため。

## 3. 依存するもの
なし(標準ライブラリの `dataclasses` のみ)。

## 4. モジュール定数と属性

### モジュール定数

| 名前 | 値 | 説明 |
|---|---|---|
| `BOARD_ROWS` | `4` | 盤の行数 |
| `BOARD_COLS` | `3` | 盤の列数 |

他のクラスは盤の大きさを直接 `4` `3` と書かず、この定数を使う。

### クラス定義
`@dataclass(frozen=True)` で定義する。これにより `==` による比較、`hash`(辞書のキーや `set` の要素にできる)、属性の書き換え禁止が自動で得られる。

| 属性 | 型 | 説明 |
|---|---|---|
| `row` | `int` | 行(0〜3 が盤内) |
| `col` | `int` | 列(0〜2 が盤内) |

## 5. メソッド仕様

### `is_valid(self) -> bool`
- **引数:** なし
- **戻り値:** 盤内なら `True`、盤外なら `False`
- **内部処理:**
  1. `0 <= row < BOARD_ROWS` かつ `0 <= col < BOARD_COLS` ならば `True`
  2. それ以外は `False`
- **例外:** なし

### `offset(self, dr: int, dc: int) -> "Position"`
- **引数:**
  - `dr`:行方向のずらし量(整数、負も可)
  - `dc`:列方向のずらし量(整数、負も可)
- **戻り値:** `Position(row + dr, col + dc)` を新しく作って返す。**盤外になってもエラーにしない。**
- **内部処理:**
  1. `Position(self.row + dr, self.col + dc)` を返す
- **例外:** なし
- 元のオブジェクトは変更しない(不変なので変更できない)。

## 6. 使われ方
- `Rule.legal_moves` が `pos.offset(dr, dc)` で移動先を求め、`is_valid()` で盤外を除く。
- `Board.get(pos)` などが引数の座標として受け取る。
- `Move.from_pos` / `Move.to_pos` に使われる。

```python
p = Position(2, 1)
q = p.offset(-1, 0)      # Position(1, 1)
r = Position(0, 0).offset(-1, 0)   # Position(-1, 0)
r.is_valid()             # False
```

## 7. テスト項目

| # | 操作 | 期待結果 |
|---|---|---|
| 1 | `Position(1, 1) == Position(1, 1)` | `True` |
| 2 | `Position(1, 1) == Position(1, 2)` | `False` |
| 3 | `{Position(1, 1): "a"}[Position(1, 1)]` | `"a"`(辞書のキーに使える) |
| 4 | `Position(0, 0).is_valid()` | `True` |
| 5 | `Position(3, 2).is_valid()` | `True` |
| 6 | `Position(4, 0).is_valid()` | `False` |
| 7 | `Position(0, 3).is_valid()` | `False` |
| 8 | `Position(-1, 0).is_valid()` | `False` |
| 9 | `Position(0, -1).is_valid()` | `False` |
| 10 | `Position(2, 1).offset(-1, 0)` | `Position(1, 1)` |
| 11 | `Position(2, 1).offset(1, -1)` | `Position(3, 0)` |
| 12 | `Position(0, 0).offset(-1, 0).is_valid()` | `False`(例外にならない) |
| 13 | `p = Position(1, 1)` の後 `p.row = 2` | `dataclasses.FrozenInstanceError` |
| 14 | `Position(1, 1).offset(0, 0)` | `Position(1, 1)`(同じ値の別オブジェクト) |

## 8. 実装上の注意
- `is_valid` や `offset` の中で盤外を例外にしないこと(他のクラスが盤外の座標を生成して判定するため)。
- 盤の大きさを変える必要が出たら、`BOARD_ROWS` `BOARD_COLS` のみを変更すればよいように、他のクラスで `4` `3` を直書きしない。
