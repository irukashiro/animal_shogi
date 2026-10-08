# 担当割り振り(5人)

## 1. 負担の見積り
各クラスの実装量とテスト量から、相対的な作業量を見積もった(単位:pt。数値は目安)。
`Rule` は他と比べて大きいため、2つに分けて見積もっている。

| 対象 | 見積り(pt) | テスト項目数 | 主な作業 |
|---|---|---|---|
| `Position` | 1 | 14 | データクラス、2メソッド |
| `Piece` と5種の駒 | 2.5 | 20 | 抽象クラス、方向表、`with_owner` `promote` `demote` |
| `Player` | 2 | 17 | 持ち駒の出し入れ、検査 |
| `Move` | 1 | 11 | データクラス、`key` |
| `Board` | 4 | 26 | 10個前後のメソッド、盤外の扱い |
| `Rule` の前半(`legal_moves`, `is_legal`) | 3.5 | 25 | 移動・打つ手の生成、成りの判定 |
| `Rule` の後半(`is_attacked`, `judge_winner`) | 4.5 | 18 | 攻撃判定、勝敗の3段階判定 |
| `View` | 3 | 14 | 表示の整形、出力のテスト |
| `Controller` 3種 | 4.5 | 27 | 入力の解析、再入力、AI |
| `Game` + `main.py` + テスト用スタブ | 6.5 | 21 | `apply_move`、進行管理、スタブ(`StubView`, `ScriptedController`) |
| **合計** | **32.5** | | 1人あたり約 6.5 pt |

## 2. 割り振り

| 担当 | 受け持つもの | 見積り | まとまりの理由 |
|---|---|---|---|
| **担当1** | `Piece` と5種の駒、`Rule.legal_moves`、`Rule.is_legal` | 6.0 pt | 合法手の生成は、駒の移動方向(`get_move_offsets`)をそのまま使う。駒と移動の手の生成を同じ人が見る |
| **担当2** | `Board`、`View` | 7.0 pt | どちらも盤面(`Board`)を扱う。`View` は `Board.get` で盤を読み、表示するだけ |
| **担当3** | `Player`、`Rule.is_attacked`、`Rule.judge_winner` | 6.5 pt | `judge_winner` が `goal_row`(`Player`)と `is_attacked` を使う。勝敗判定と持ち駒・ゴール段の知識が集中する |
| **担当4** | `Position`、`Move`、`Controller`(`HumanController`, `AIController`) | 6.5 pt | `HumanController` が入力を `Position` と `Move` に変換する。入力→手の変換が一人にまとまる |
| **担当5** | `Game`、`main.py`、テスト用スタブ | 6.5 pt | 全体の組み立て役。`apply_move` で全クラスの使い方を把握する。結合の窓口 |

### 担当ごとの成果物とテストファイル

| 担当 | 実装ファイル | テストファイル |
|---|---|---|
| 担当1 | `pieces.py`、`rule.py` の `legal_moves` / `is_legal` | `test_pieces.py`、`test_rule_moves.py`(`06_Rule.md` の 7.1 と 7.2) |
| 担当2 | `board.py`、`view.py` | `test_board.py`、`test_view.py` |
| 担当3 | `player.py`、`rule.py` の `is_attacked` / `judge_winner` | `test_player.py`、`test_rule_judge.py`(`06_Rule.md` の 7.3 と 7.4) |
| 担当4 | `position.py`、`move.py`、`controller.py` | `test_position.py`、`test_move.py`、`test_controller.py` |
| 担当5 | `game.py`、`main.py`、`tests/helpers.py`(スタブ) | `test_game.py` |

読む仕様書:自分の担当クラスの仕様書(`01`〜`09`)と `00_README.md`。

## 3. `Rule` を2人で分ける際の取り決め
`rule.py` を担当1と担当3が同時に編集するため、次のように進める。

1. 担当1が、4メソッドすべてを `raise NotImplementedError` にした `rule.py` の骨組みを最初に作る。
2. 担当1は `legal_moves` と `is_legal` だけ、担当3は `is_attacked` と `judge_winner` だけを実装する。互いのメソッドを編集しない。
3. テストファイルは `test_rule_moves.py`(担当1)と `test_rule_judge.py`(担当3)に分ける。
4. 同じファイルを同時に編集できる環境(Git など)でない場合は、担当3が先に別ファイルで実装し、結合時に担当1が `rule.py` へ貼り付ける。
5. `judge_winner` が `is_attacked` を使うが、どちらも担当3なので担当1との調整は不要。`legal_moves` と `is_attacked` の間に依存はない。

## 4. 担当間で受け渡すもの
他人のクラスを使う側は、相手の実装が終わる前に **骨組み(シグネチャのみ)** を受け取って作業を始める。

| 渡す人 | 渡すもの | 受け取る人 | 用途 |
|---|---|---|---|
| 担当4 | `Position`、`Move` | 全員 | ほぼ全クラスが使う。**最初に完成させる**(どちらも短時間で終わる) |
| 担当3 | `Player` | 担当1、担当2、担当5 | `direction`、`hand`、`goal_row` |
| 担当1 | `Piece` と5種の駒 | 担当2、担当3、担当5 | `symbol`、`kind`、`get_move_offsets` |
| 担当2 | `Board` | 担当1、担当3、担当5 | 合法手・攻撃判定・初期配置 |
| 担当2 | `View` | 担当4、担当5 | `HumanController` のメッセージ表示、`Game` の表示 |
| 担当1・3 | `Rule` | 担当5 | `Game.play_turn` |
| 担当4 | `Controller` | 担当5 | `Game` の入力 |

### 進行の目安
1. **最初の1回(全員):** 仕様書を確認し、疑問点とルール上の決定事項(ヒヨコを打てる場所など)を合意する。
2. **骨組み作成:** 全員が自分のクラスの骨組みを作って共有する。担当4の `Position` と `Move` は、骨組みではなく中身まで先に完成させる。
3. **実装と単体テスト:** 各自、仕様書のテスト項目に沿って実装する。他人のクラスが未完成の間は、ダミーやスタブで代用する。
4. **結合:** 依存の少ない順に結合する。`Position`/`Move` → `Piece`/`Player` → `Board` → `Rule` → `View`/`Controller` → `Game`。
5. **通し確認:** 担当5が `main.py` を実行し、全員で1局を通して動かす。

## 5. 相互レビューの組み合わせ
実装後に、次の組で互いのコードを読み合う。依存関係の強い相手と組むことで、仕様の食い違いを見つけやすくする。

| レビューする人 | レビューされる人 | 観点 |
|---|---|---|
| 担当1 ⇔ 担当3 | `Rule` の前半と後半、`Piece` と `Player` | 向きの扱い(`direction`、`get_move_offsets`)が一致しているか |
| 担当2 ⇔ 担当5 | `Board` と `Game` | `Game.apply_move` の呼び出し順で `Board` が正しく動くか |
| 担当4 ⇔ 担当5 | `Controller` と `Game` | `choose_move` の戻り値が `Game` の検査を通るか |

## 6. 負担の偏りについて
- 見積りは目安であり、実際の負担は経験や進み具合で変わる。早く終わった人は、未完成の部分のテストを手伝う。
- 担当2(7.0 pt)が最も重い。`View` の表示整形は後回しにできるので、`Board` を先に完成させる。
- 担当4は項目数(52)が多いが、`Position` と `Move` が単純なため、見積りは平均と同程度にしている。
- 余裕が出たら、`AIController` の拡張や千日手の判定(任意課題)を、早く終わった人が引き受ける。
