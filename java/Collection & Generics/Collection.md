# コレクション
## ラッパークラス
ラッパークラスは参照型のため null を扱える
等価性は.equals()を使う

|`基本データ型`|`ラッパークラス`|
|:---|:---|
|byte|Byte|
|short|Short|
|int|Integer|
|long|Long|
|float|Float|
|double|Double|
|char|Character|
|boolean|Boolean|

### 型変換 
**データ型→ラッパークラス**
イミュータブルなので値の変更はできない
- valueOF()を使う
**ラッパークラス→データ型**
- int Value(),long Value()など

### オートボクシング / オートアンボクシング （自動型変換）
- 基本型→ラッパークラスは自動型変換オートボクシング
- ラッパークラス→基本型はオートアンボクシング

