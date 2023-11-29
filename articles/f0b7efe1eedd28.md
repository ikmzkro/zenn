---
title: "Pythonで作るマークルツリー"
emoji: "🌲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "solidity", "ethereum", "markletree", "crypto"]
published: false
---

# マークルツリーとは何か
マークルツリーとは、効率的な包含証明のために設計されたデータ構造の一つです。
完全なブロックチェーンデータを保持できないスマホのような軽量クライアントは、このマークルツリーを用いてブロックヘッダのマークルルートフィールドを確認することで、トランザクションの整合性を確認し、取引の安全性を確認できます。


マークルツリーの構成は[サトシナカモトの論文](https://bitcoin.org/bitcoin.pdf)が分かりやすいです。
![merkletree](/images/merkletree.png)


マークルツリーを構築するうえで必要なものは以下の2点です。
1. アイテムの順序付きリスト
2. 暗号学的ハッシュ関数


アイテムはブロック内のトランザクションで、以下が順序付きリストの例です。暗号学的ハッシュ関数は `hashlib.sha256` を使用します。
```
hex_hashes = [
    "9745f7173ef14ee4155722d1cbf13304339fd00d900b759c6f9d58579b5765fb",
    "5573c8ede34936c29cdfdfe743f7f5fdfbd4f54ba0705259e62f39917065cb9b",
    "82a02ecbb6623b4274dfcab82b336dc017a27136e08521091e443e62582e8f05",
]
```


マークルツリーを構築するために以下のアルゴリズムを利用します。
1. ハッシュ関数を使用して、順序付きリストのアイテム全てをハッシュ化します
2. 1つのハッシュになれば、処理を完了します
3. 2とならない場合、ハッシュの個数が奇数なら末尾のハッシュを複製して偶数にします
4. ハッシュを順番にペアにして、連結したものをハッシュ化して親レベルのハッシュを生成します。※`親レベルのハッシュの個数は半分になります`
5. 2に戻ります


つまり、`マークルツリーはアイテムの順序付きリスト全体をただ1つのハッシュにするというアイデア`です。


マークルツリーに関する用語を整理します。
- リーフ：最下位のハッシュリスト
- 内部ノード：それ以外のハッシュリスト
- マークルルート：最高位のハッシュ
- マークルペアレント: ハッシュを順番にペアにして、連結したものをハッシュ化したもの
- マークルペアレントレベル: 各ペアの親ハッシュを得るための条件 (ex: 順序付きハッシュリストが３つ以上存在すること)
- マークルパス: マークルルートを求める最短経路

# マークルツリーを構築する
:::順序付きハッシュリスト
```main.py
# 順序付きハッシュリストをインプットしてハッシュ化
hex_hashes = [
    'c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5',
    'c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5',
    'f391da6ecfeed1814efae39e7fcb3838ae0b02c02ae7d0a5848a66947c0727b0',
    '3d238a92a94532b946c90e19c49351c763696cff3db400485b813aecb8a13181',
    '10092f2633be5f3ce349bf9ddbde36caa3dd10dfa0ec8106bce23acbff637dae',
]
hashes = [bytes.fromhex(x) for x in hex_hashes]

# ハッシュ数が奇数なら末尾のハッシュ値を複製して連結
if len(hashes) % 2 == 1:
    hashes.append(hashes[-1])

# マークルペアレントレベル
parent_level = []

# ハッシュ値を２つずつ選択してペアを作成
for i in range(0, len(hashes), 2):
    parent = merkle_parent(hashes[i], hashes[i+1])
    parent_level.append(parent)
for item in parent_level:
    print(item.hex())

```
8b30c5ba100f6f2e5ad1e2a742e5020491240f8eb514fe97c713c31718ad7ecd
7f4e6f9e224e20fda0ae4c44114237f97cd35aca38d83081c9bfd41feb907800
3ecf6115380c77e8aae56660f5634982ee897351ba906a6837d15ebc3a225df0
```
:::

# おまけ


# 参考
https://bitcoin.org/bitcoin.pdf
https://books.google.co.jp/books/about/%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0_%E3%83%93%E3%83%83%E3%83%88%E3%82%B3%E3%82%A4%E3%83%B3.html?id=FagHzgEACAAJ&source=kp_book_description&redir_esc=y
https://github.com/learn-co-students/session7-jsong-programming-blockchain-demo/blob/master/index.ipynb
https://alis.to/mozk/articles/3PYokoOWP7XY
https://github.com/sskgik/merkletree_with_python/blob/main/merkletree.py
https://github.com/sskgik/Merkletree/blob/master/Program.cs