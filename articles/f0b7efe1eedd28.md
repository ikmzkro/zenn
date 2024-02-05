---
title: "マークルツリー"
emoji: "🌲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "merkletree", "crypto"]
published: false
---

# 作ったもの
https://github.com/ikmzkRo/WLAQ-mint

# どんな機能？
NFTを発行する際、複数のアドレスがいくつまで発行可能かを明示したホワイトリストを作成します。このホワイトリストに登録されたアドレスに基づいてNFTが発行可能となるよう、マークルツリーの技術を駆使して制御を行います。

今回はウォレットアドレスとトークン発行数を特定のホワイトリストで管理するスマートコントラクトを実装しました。

# マークルツリーとは何か
ブロックチェーンの基本技術であるマークルツリーの理論的な理解をPythonを使用して進めていきます。

## マークルツリーの基本概念
マークルツリーは、[包含証明](https://zenn.dev/link/comments/75c3f508fb9744)を効率的に行うために設計されたデータ構造の一例です。軽量クライアントなど、完全なブロックチェーンデータを保持できない状況では、マークルツリーを使用してブロックヘッダのマークルルートフィールドを確認することで、トランザクションの整合性を検証し、取引の安全性を確保できます。

このメカニズムは、ウォレットを使用する際に非常に有益です。ウォレットがブロックチェーン上の全トランザクションデータを保持すると遅くなる可能性があるため、代わりにデータの要約だけを保持します。ただし、これだけでは自身のトランザクションデータを直接閲覧できません。そのため、自身のアドレスを使用して改ざんの有無を検証し、その後元データから必要なトランザクションデータを取得することで、軽量で効率的に取引履歴を管理できます。

マークルツリーの構成は[サトシナカモトの論文](https://bitcoin.org/bitcoin.pdf)が分かりやすいです。
![merkletree](/images/merkletree.png)

## マークルツリーの構築手順
マークルツリーを構築するうえで必要なものは以下の2点です。
1. アイテムの順序付きリスト: マークルツリーは、ブロック内のトランザクションなどのアイテムが順序付きリストとして提供されることで構築されます。これにより、ツリーの構造が形成されます。
2. 暗号学的ハッシュ関数: マークルツリーでは、各ノードのデータは暗号学的ハッシュ関数によってハッシュ化されます。通常、hashlib.sha256のような安全なハッシュ関数が使用され、それによってツリー内の各ノードが一意のハッシュ値を持つようになります。

これらの要素を組み合わせて、マークルツリーを構築することが可能です。順序付きリストからアイテムを取り、それぞれのアイテムに対してハッシュ関数を適用し、ツリーの上位レベルへと進んでいくことで、最終的に一つのルートハッシュ（マークルルート）が得られます。これによって、マークルツリーを使用したデータの整合性を検証することができます。

マークルツリーを構築するために以下のアルゴリズムを利用します。
1. ハッシュ関数を使用して、順序付きリストのアイテム全てをハッシュ化する。
2. ハッシュの個数が1つになれば、処理を完了する。
3. ハッシュの個数が2つ以上でかつ奇数なら、末尾のハッシュを複製して偶数にする。
4. ハッシュを順番にペアにして、連結したものをハッシュ化して親レベルのハッシュを生成する。親レベルのハッシュの個数は半分になる。
5. 生成されたハッシュが1つでなければ、2に戻る。

このアルゴリズムによって得られるマークルルートは、マークルツリー全体の整合性を確認する際に極めて重要な要素です。マークルツリーは、アイテムの順序付きリスト全体を単一のハッシュにまとめる手法であり、このハッシュが変更されていない限り、元のデータに変更が加えられていないことを示します。

## マークルツリーの用語整理
以下はマークルツリーに関する用語の整理です。
1. リーフ：マークルツリーの最下位に位置するノードで、元のデータ（例: トランザクション）を表します。各リーフはハッシュ関数によってハッシュ化されています。
2. 内部ノード： リーフ以外のノードで、マークルツリーの中間レベルに位置します。内部ノードはリーフや他の内部ノードのハッシュから計算されます。
3. マークルルート：マークルツリーの最上位にあるノードで、全体の整合性を示す単一のハッシュです。これが変更されると、マークルツリー全体の変更が検知されます。
4. マークルペアレント:  二つのハッシュを取り、それらを連結してハッシュ化したものです。この操作により、新しいノード（親ノード）が生成されます。
5. マークルペアレントレベル: マークルツリーにおいて、各ペアの親ハッシュを得るための特定のレベル。通常、順序付きハッシュリストが3つ以上存在する場合にマークルペアレントレベルが形成されます。
6. マークルパス: マークルツリーのルートからリーフまでの最短経路に含まれるハッシュ値の列。これを使用することで、特定のリーフがマークルルートに正しく結びついていることを検証できます。
7. マークルプルーフ: マークルツリーのルートから対象のノードまでのパスを示す情報で、対象がマークルツリー内に存在することを検証するのに使用されます。マークルルートを再度計算しデータの有効性を証明します。

## マークルツリーを構築する
:::details マークルペアレントを求める
```py: main.py
from helper import hash256

leftHash = bytes.fromhex('c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5')
rightHash = bytes.fromhex('c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5')
parent = hash256(leftHash + rightHash)

print(parent.hex())
```
出力:
```
8b30c5ba100f6f2e5ad1e2a742e5020491240f8eb514fe97c713c31718ad7ecd
```
:::
ハッシュをペアにし、新しいハッシュを生成するプロセスでは、通常、左側のハッシュ（L）と右側のハッシュ（R）として、それらを連結した結果を親ハッシュ（P）またはマークルペアレントと呼びます。
```
P = H(L||R)　※||は連結
```

:::details マークルペアレントレベルに対応する新たなハッシュリストを求める
```py: main.py
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
出力:
```
8b30c5ba100f6f2e5ad1e2a742e5020491240f8eb514fe97c713c31718ad7ecd
7f4e6f9e224e20fda0ae4c44114237f97cd35aca38d83081c9bfd41feb907800
3ecf6115380c77e8aae56660f5634982ee897351ba906a6837d15ebc3a225df0
```
:::
ハッシュの個数が奇数の場合、末尾のハッシュを複製して偶数に調整し、それからハッシュをペアにして連結して新しいハッシュを生成する手順は、マークルツリーの構築において一般的な処理です。例として、5つのハッシュから始め、末尾のハッシュを複製してハッシュリストを偶数に整理した後、ペアを作成して連結すると、最終的に要素数3のハッシュリストが得られることが期待されます。


:::details マークルルートを求める
```py: main.py
from helper import merkle_parent_level
hex_hashes = [
    'c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5',
    'c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5',
    'f391da6ecfeed1814efae39e7fcb3838ae0b02c02ae7d0a5848a66947c0727b0',
    '3d238a92a94532b946c90e19c49351c763696cff3db400485b813aecb8a13181',
    '10092f2633be5f3ce349bf9ddbde36caa3dd10dfa0ec8106bce23acbff637dae',
    '7d37b3d54fa6a64869084bfd2e831309118b9e833610e6228adacdbd1b4ba161',
    '8118a77e542892fe15ae3fc771a4abfd2f5d5d5997544c3487ac36b5c85170fc',
    'dff6879848c2c9b62fe652720b8df5272093acfaa45a43cdb3696fe2466a3877',
    'b825c0745f46ac58f7d3759e6dc535a1fec7820377f24d4c2c6ad2cc55c0cb59',
    '95513952a04bd8992721e9b7e2937f1c04ba31e0469fbe615a78197f68f52b7c',
    '2e6d722e5e4dbdf2447ddecc9f7dabb8e299bae921c99ad5b0184cd9eb8e5908',
    'b13a750047bc0bdceb2473e5fe488c2596d7a7124b4e716fdd29b046ef99bbf0',
]
hashes = [bytes.fromhex(x) for x in hex_hashes]
current_hashes = hashes
while len(current_hashes) > 1:
    current_hashes = merkle_parent_level(current_hashes)
print(current_hashes[0].hex())
```
出力:
```
acbcab8bcc1af95d8d563b77d24c3d19b18f1486383d75a5085c4e86c86beed6
```
:::
while ループの繰り返し条件としてハッシュリストの長さが1より大きいかどうかを判定し、長さが1になればループを終了します。

:::details マークルツリーの構造を把握する
```py: main.py
import math

# マークルツリーの構造を把握
from helper import validate_merkle_root

# リーフ数は16（軽量クライアントが最初に必要な情報）
total = 16

# 二分木探索方法に必要な深さを求める（ここではlog2^16=4）
max_depth = math.ceil(math.log(total, 2))
merkle_tree = []
for depth in range(max_depth + 1):
    num_items = math.ceil(total / 2**(max_depth - depth))
    level_hashes = [None] * num_items
    merkle_tree.append(level_hashes)
for level in merkle_tree:
    print(level)
```
出力:
```
[None]
[None, None]
[None, None, None, None]
[None, None, None, None, None, None, None, None]
[None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None, None]
```
:::
軽量クライアントがフルノードから受信する情報の中で、必要なのはリーフ数（トランザクション数）です。これは、マークルツリーの構造を理解するための基本的な情報であり、リーフ数を知ることでマークルツリーの深さを計算できます。具体的には、マークルツリーの深さは log2^(リーフ数) で把握できます。そして、この深さ情報を for ループの繰り返し条件に利用しています。

:::details ハッシュリストを指定してマークルツリーの構造を把握する
```py: main.py
from merkleblock import MerkleTree
from helper import merkle_parent_level
hex_hashes = [
    "9745f7173ef14ee4155722d1cbf13304339fd00d900b759c6f9d58579b5765fb",
    "5573c8ede34936c29cdfdfe743f7f5fdfbd4f54ba0705259e62f39917065cb9b",
    "82a02ecbb6623b4274dfcab82b336dc017a27136e08521091e443e62582e8f05",
    "507ccae5ed9b340363a0e6d765af148be9cb1c8766ccc922f83e4ae681658308",
    "a7a4aec28e7162e1e9ef33dfa30f0bc0526e6cf4b11a576f6c5de58593898330",
    "bb6267664bd833fd9fc82582853ab144fece26b7a8a5bf328f8a059445b59add",
    "ea6d7ac1ee77fbacee58fc717b990c4fcccf1b19af43103c090f601677fd8836",
    "457743861de496c429912558a106b810b0507975a49773228aa788df40730d41",
    "7688029288efc9e9a0011c960a6ed9e5466581abf3e3a6c26ee317461add619a",
    "b1ae7f15836cb2286cdd4e2c37bf9bb7da0a2846d06867a429f654b2e7f383c9",
    "9b74f89fa3f93e71ff2c241f32945d877281a6a50a6bf94adac002980aafe5ab",
    "b3a92b5b255019bdaf754875633c2de9fec2ab03e6b8ce669d07cb5b18804638",
    "b5c0b915312b9bdaedd2b86aa2d0f8feffc73a2d37668fd9010179261e25e263",
    "c9d52c5cb1e557b92c84c52e7c4bfbce859408bedffc8a5560fd6e35e10b8800",
    "c555bc5fc3bc096df0a0c9532f07640bfb76bfe4fc1ace214b8b228a1297a4c2",
    "f9dbfafc3af3400954975da24eb325e326960a25b87fffe23eef3e7ed2fb610e",
]
tree = MerkleTree(len(hex_hashes))
tree.nodes[4] = [bytes.fromhex(h) for h in hex_hashes]
tree.nodes[3] = merkle_parent_level(tree.nodes[4])
tree.nodes[2] = merkle_parent_level(tree.nodes[3])
tree.nodes[1] = merkle_parent_level(tree.nodes[2])
tree.nodes[0] = merkle_parent_level(tree.nodes[1])

print(tree)
```
出力:
```
*597c4baf.*
6382df3f..., 87cf8fa3...
3ba6c080..., 8e894862..., 7ab01bb6..., 3df760ac...
272945ec..., 9a38d037..., 4a64abd9..., ec7c95e1..., 3b67006c..., 850683df..., d40d268b..., 8636b7a3...
9745f717..., 5573c8ed..., 82a02ecb..., 507ccae5..., a7a4aec2..., bb626766..., ea6d7ac1..., 45774386..., 76880292..., b1ae7f15..., 9b74f89f..., b3a92b5b..., b5c0b915..., c9d52c5c..., c555bc5f..., f9dbfafc...
```
:::
ハッシュリストを指定することで、マークルツリーとその最上位にあるマークルルートを確認することができ、これによってデータの整合性を視認しやすくなります。

:::details マークルパスの探索（偶数・奇数）
偶数
```py: main.py
from merkleblock import MerkleTree
from helper import merkle_parent
hex_hashes = [
    "9745f7173ef14ee4155722d1cbf13304339fd00d900b759c6f9d58579b5765fb",
    "5573c8ede34936c29cdfdfe743f7f5fdfbd4f54ba0705259e62f39917065cb9b",
    "82a02ecbb6623b4274dfcab82b336dc017a27136e08521091e443e62582e8f05",
    "507ccae5ed9b340363a0e6d765af148be9cb1c8766ccc922f83e4ae681658308",
]
tree = MerkleTree(len(hex_hashes))
tree.nodes[2] = [bytes.fromhex(h) for h in hex_hashes]
while tree.root() is None:
    if tree.is_leaf():
        tree.up()
    else:
        left_hash = tree.get_left_node()
        if left_hash is None:
            tree.left()
        elif tree.right_exists():
            right_hash = tree.get_right_node()
            if right_hash is None:
                tree.right()
            else:
                tree.set_current_node(merkle_parent(left_hash, right_hash))
                tree.up()
        else:
            tree.set_current_node(merkle_parent(left_hash, left_hash))
            tree.up()
print(tree)
```
出力:
```
3ba6c080...
272945ec..., 9a38d037...
9745f717..., 5573c8ed..., 82a02ecb..., 507ccae5...
```

奇数
```py: main.py
from merkleblock import MerkleTree
from helper import merkle_parent
hex_hashes = [
    "9745f7173ef14ee4155722d1cbf13304339fd00d900b759c6f9d58579b5765fb",
    "5573c8ede34936c29cdfdfe743f7f5fdfbd4f54ba0705259e62f39917065cb9b",
    "82a02ecbb6623b4274dfcab82b336dc017a27136e08521091e443e62582e8f05",
]
tree = MerkleTree(len(hex_hashes))
tree.nodes[2] = [bytes.fromhex(h) for h in hex_hashes]
while tree.root() is None:
    if tree.is_leaf():
        tree.up()
    else:
        left_hash = tree.get_left_node()
        if left_hash is None:
            tree.left()
        elif tree.right_exists():
            right_hash = tree.get_right_node()
            if right_hash is None:
                tree.right()
            else:
                tree.set_current_node(merkle_parent(left_hash, right_hash))
                tree.up()
        else:
            tree.set_current_node(merkle_parent(left_hash, left_hash))
            tree.up()
print(tree)
```
出力:
```
a779fe9f...
272945ec..., bdca1c60...
9745f717..., 5573c8ed..., 82a02ecb...
```
:::
バイナリツリー（二分木）の走査には主に2つのアプローチがあります: 幅優先探索と深さ優先探索。

1. 幅優先探索 (Breadth-First Search, BFS): レベルごとに左から右に進んでいく探索方法です。同じレベルのノードを全て調査してから、次のレベルに進みます。幅優先探索は通常、キューを使用して実装されます。
2. 深さ優先探索 (Depth-First Search, DFS): 各ノードにおける子ハッシュのうち、ある方向（通常左）に進んでから、他の方向に進む探索方法です。深さ優先探索は通常、再帰を使用して実装されます。

マークルツリーの構築においては、通常深さ優先探索が採用されます。これは、リーフに到達すると親ハッシュに戻る必要がなく、またマークルツリーの構造において深さ優先探索が都合が良いからです。特に、リーフが奇数の場合でも、深さ優先探索ではスムーズに処理できます。

実装においては、再帰や明示的なスタックを使用して深さ優先探索を実現することが一般的です。

## マークルルートを検証する
これまでのコードでマークルツリーの構築が確認できました。本項では、これまでの知識を整理し、技術検証を進めます。

:::details マークルルートの導出
```py: verify.py
class Tree:
    def __init__(self, leaves):
        self.leaves = [Node(leaf) for leaf in leaves]
        self.layer  = self.leaves[::]
        self.root   = None
        self.build_tree()
```
入力:
```
block = ["あめ", "あめ", "みかん", "みかん", "みかん", "りんご", "りんご", "どーなっつ", "どーなっつ"]
print(block)

merkletree = Tree(block)

merkle_root = merkletree.root
print(merkle_root)
```
出力:
```
bf03b42cf8e4e8581e08c7595775dc2d9d79c34f9b95885a5d42df100786a218
```
:::
このリストはブロック内のトランザクションに対するマークルパスであり、最終的に求まったマークルルートはウォレットにおいてそのブロック内の全トランザクションの整合性を検証するための重要な情報です。

ウォレットがマークルツリーを用いてマークルルートを計算し、それがブロックヘッダのマークルルートと一致することを確認することで、ブロック内のトランザクションデータが改ざんされていないことを検証できます。これにより、ウォレットは効率的に自身の残高やトランザクションの整合性を確認することができます。

:::details マークルパスの探索
```py: verify.py
def search(self, data):
    target = None
    hash_value = sha256(data.encode()).hexdigest()
    for node in self.leaves:
        if node.hash == hash_value:
            target = node
    return target

def get_pass(self, data):
    target = self.search(data)
    markle_pass = []
    if not(target):
        return
    markle_pass.append(target.hash)
    while target.parent:
        sibling = target.sibling
        markle_pass.append((sibling.hash, sibling.position))
        target = target.parent
    return markle_pass   
```
入力:
```
merkle_pass = merkletree.get_pass("みかん")
print(merkle_pass)
```
出力:
```
['1cbe93b936a4bd4d40c2224462023917cd77c962376feb9128d9e569c9856aaf', ('4261abfc91324dc5319312592125610a16b0b0a996fcdfae1d24766b918afae9', 'right'), ('0a6b2aa3a1fc41a9394c48adbf1cb1c3eb3e3a9db975906fb743308f23e1f15b', 'right'), ('2165f2270bbe8ce292395df14cb4e65e271fe608013c018d6b5864ec9f1865c8', 'left'), ('94f4c2ba6d697dc41109e2194343bf721e2bbc5de48068609578c0f4ba5481ed', 'right')]
```
:::
merkle_pass を求めると"みかん"ノードからマークルツリーのルートまでの経路が示されます。このマークルパスを使うことで、特定のデータがマークルツリー内で正当な位置にあることを検証できます。

0. `1cbe93b936a4bd4d40c2224462023917cd77c962376feb9128d9e569c9856aaf`: "みかん"ノードのハッシュ値
1. `('4261abfc91324dc5319312592125610a16b0b0a996fcdfae1d24766b918afae9', 'right')`: "みかん"ノードの親ノードの右側にある兄弟ノードのハッシュ値とその位置情報。"みかん"ノードとその兄弟ノード（右側のノード）のハッシュを連結して親ノードを作り、そのハッシュがこの要素になります。
2. `('0a6b2aa3a1fc41a9394c48adbf1cb1c3eb3e3a9db975906fb743308f23e1f15b', 'right')`: これは上記のノード（親ノードの右側のノード）の親ノードの右側にある兄弟ノードのハッシュ値とその位置情報です。同様に、ハッシュを連結して親ノードを作り、そのハッシュがこの要素になります。
3. `('2165f2270bbe8ce292395df14cb4e65e271fe608013c018d6b5864ec9f1865c8', 'left')`: 同様の説明で、これは上記ノードの親ノードの左側にある兄弟ノードのハッシュ値とその位置情報です。
4. `('94f4c2ba6d697dc41109e2194343bf721e2bbc5de48068609578c0f4ba5481ed', 'right')`: 同様の説明で、これは上記ノードの親ノードの右側にある兄弟ノードのハッシュ値とその位置情報です。

これらの要素をたどることで、"みかん" ノードからマークルツリーのルートノードまでの経路が示されます。

:::details マークルパスの検証
```py: verify.py
def caluculator(markle_pass):
    value = markle_pass[0]
    for node in markle_pass[1:]:
        sib = node[0]
        position = node[1]
        if position == "right":
            value = sha256(value.encode() + sib.encode()).hexdigest()
        else:
            value = sha256(sib.encode() + value.encode()).hexdigest()
    return value   
```
入力:
```
candidate = caluculator(merkle_pass)
print(candidate)
```
出力:
```
bf03b42cf8e4e8581e08c7595775dc2d9d79c34f9b95885a5d42df100786a218
```
:::
マークルツリーとそのマークルパスを使用してトランザクションの整合性を確認し、最終的にマークルルートが一致することで改ざんが行われていないことを検証できました。
これにより、軽量クライアントやウォレットは全体のブロックチェーンデータを保持せずに、特定のトランザクションの整合性を確認することができるというメリットがあります。


:::details マークルパスの検証
```py: verify.py
def caluculator(markle_pass):
    value = markle_pass[0]
    for node in markle_pass[1:]:
        sib = node[0]
        position = node[1]
        if position == "right":
            value = sha256(value.encode() + sib.encode()).hexdigest()
        else:
            value = sha256(sib.encode() + value.encode()).hexdigest()
    return value   
```
入力:
```
block = ["あめ", "あめ", "みかん", "みかん", "りんご", "りんご", "どーなっつ", "どーなっつ"]
print(block)

merkletree = Tree(block)

merkle_pass = merkletree.get_pass("みかん")
print(merkle_pass)

candidate = caluculator(merkle_pass)
print(candidate)

merkle_root = merkletree.root
print(merkle_root)
```
出力:
```
['あめ', 'あめ', 'みかん', 'みかん', 'りんご', 'りんご', 'どーなっつ', 'どーなっつ']
['1cbe93b936a4bd4d40c2224462023917cd77c962376feb9128d9e569c9856aaf', ('1cbe93b936a4bd4d40c2224462023917cd77c962376feb9128d9e569c9856aaf', 'left'), ('aa280a624da2b88334623a7033f5a6eac1da7012ace335a6366828f59377ed1b', 'left'), ('4d8a97b9474b34c013c92ba1662bb98ae8cab97681ae21b9843db45ee2854730', 'right')]
b8cc63db762632990e49bea7832293ae2d18cff643c78c1e2274c7fcf757354a
b8cc63db762632990e49bea7832293ae2d18cff643c78c1e2274c7fcf757354a
```
:::
最後にブロックの内容を改ざんしてみて、それによってマークルツリーの整合性が損なわれ、マークルルートが異なることを確認してみます。
みかんを1つ食べて要素を1つ少なくします。
マークルツリーからマークルパスを取得し、マークルルートを算出すると、改ざん前のツリーのマークルルートと異なる事が確認できます。

- 改ざん前: `bf03b42cf8e4e8581e08c7595775dc2d9d79c34f9b95885a5d42df100786a218`
- 改ざん後: `b8cc63db762632990e49bea7832293ae2d18cff643c78c1e2274c7fcf757354a`


# スマートコントラクト開発とデプロイ
ここまでの理論的な理解を踏まえ、マークルツリーを用いたホワイトリスト付きのスマートコントラクトを実装していきます。

## コードの責務
コードの責務は下記の通りです。
| ファイル                  | 説明                                                |
| ------------------------- | --------------------------------------------------- |
| `contracts/WLAQ.sol`      | マークルプルーフ認証を行い、トークン発行を行うスマートコントラクト   |
| `test/WLAQ.ts`            | `contracts/WLAQ.sol`のテストコード                    |
| `utils/data.ts`           | ホワイトアドレスに登録するウォレットアドレスや数量の定義        |
| `utils/interfaces.ts`     | 型定義                                              |
| `utils/merkletree.ts`     | マークルツリーを構築する関数                           |
| `scripts/deploy.ts`       | `contracts/WLAQ.sol`をテストネット上にデプロイするスクリプト    |

## スマートコントラクト
コントラクトデプロイ時に、計算結果のマークルルートを設定します。
```sol
constructor(bytes32 merkleRoot_) ERC721('Zutto Mayonakade Iinoni', 'ZTMY') {
    merkleRoot = merkleRoot_;
}
```

コントラクト内でのMerkleProof認証には、`@openzeppelin/contracts/utils/cryptography/MerkleProof.sol`で提供されている関数を活用します。

バリデーションを正常に通過した場合（つまり、msg.senderとquantityがホワイトリストに登録され、承認された場合）、指定された数量のトークンを指定のアドレスに発行することが可能です。

```sol
import '@openzeppelin/contracts/token/ERC721/ERC721.sol';
import '@openzeppelin/contracts/utils/Counters.sol';
import {MerkleProof} from '@openzeppelin/contracts/utils/cryptography/MerkleProof.sol';
...
function mint(uint256 quantity, bytes32[] calldata merkleProof) public {
    bytes32 node = keccak256(abi.encodePacked(msg.sender, quantity));
    require(MerkleProof.verify(merkleProof, merkleRoot, node), 'invalid proof');

    for (uint256 i = 0; i < quantity; i++) {
        uint256 tokenId = _tokenIds.current();
        _mint(msg.sender, tokenId);

        _tokenIds.increment();
    }
}
```

ウォレットアドレスの追加や数量の変更など、運用上の課題に対処するために、新しく計算されたマークルルートを再登録するための関数が導入されています。ただし、この関数は誰でも簡単に更新できてしまうと問題が生じる可能性があります。例えば、悪意を持ったユーザが不正なアドレスをホワイトリストに追加し、それに基づいて計算されたマークルルートを登録するなどの行為が考えられます。これを防ぐために、この関数には`onlyOwner`修飾子が適用され、関数の実行にはオーナーの認証が必要です。
```sol
import "@openzeppelin/contracts/access/Ownable.sol";
...
function setMerkleRoot(bytes32 _merkleRoot) public onlyOwner {
    merkleRoot = _merkleRoot;
}
```

現在登録されているマークルルートを返却するための関数が存在します。この関数はオーナーのみが更新できる仕様となっていますが、運用上の確認やテストコードの実行など、情報の取得が必要な際に活用されます。
```sol
function getMerkleRoot() external view returns (bytes32) {
    return merkleRoot;
}
```

https://github.com/ikmzkRo/WLAQ-mint/blob/main/contracts/WLAQ.sol

## テストコード
チェック項目は下記の通りです。
| 項目                  | 説明                                                |
| ------------------------- | --------------------------------------------------- |
|　コントラクトデプロイ      | コントラクトデプロイ時にERC721の名前とシンボルおよびマークルルートが登録されていること。   |
| マークルルートはオーナーのみ更新可能            | 運用上の問題でホワイトリストに変更があった場合、オーナーのみがマークルルートを更新できること。 |
| ホワイトリストに応じたトークン発行       | ホワイトリストに登録されたアドレスと数量に応じた場合のみトークン発行ができること    |
| 残高確認           | ホワイトリストに登録されたアドレスと数量に応じたトークンが発行されたかどうかを確認する       |

https://github.com/ikmzkRo/WLAQ-mint/blob/main/test/WLAQ.ts

## マークルツリー構築用の関数
テストの際に利用する初回デプロイ用と、実際の運用時にホワイトリストを変更できるように設定されています。
```ts
const inputs = flug === 'default'
  ? await makeInputs(usernames, usersQuantity)
  : await makeInputs(usernamesOperation, usersQuantityOperation);
```

ホワイトリストからリーフを生成し、各々をハッシュ化していくことで、マークルツリーを構築します。`getHexRoot()`を使用して、マークルツリーのルートハッシュを取得できます。また、`getHexProof(leaf)`を利用すると、指定されたノードからマークルルートまでの最短経路であるマークルプルーフを取得できます。

```ts
import keccak256 from 'keccak256';
import { MerkleTree } from 'merkletreejs';
...
const leaves = makeLeaves(inputs);
const leavesValue = Object.values(leaves);
const merkleTree = new MerkleTree(leavesValue, keccak256, { sort: true });

const root = merkleTree.getHexRoot();
const proof = merkleTree.getHexProof(leaf);
```

https://github.com/ikmzkRo/WLAQ-mint/blob/main/utils/merkletree.ts

実行結果
:::details yarn run test
```
yarn run v1.22.21
$ hardhat test
Compiled 13 Solidity files successfully


  WLAQ
    Deployment
inputs [
  {
    address: '0x70997970C51812dc3A010C7d01b50e0d17dc79C8',
    quantity: 1
  },
  {
    address: '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC',
    quantity: 2
  },
  {
    address: '0x90F79bf6EB2c4f870365E785982E1f101E93b906',
    quantity: 1
  }
]
leaves {
  '0x70997970C51812dc3A010C7d01b50e0d17dc79C8': '0x3f68e79174daf15b50e15833babc8eb7743e730bb9606f922c48e95314c3905c',
  '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC': '0xd0583fe73ce94e513e73539afcb4db4c1ed1834a418c3f0ef2d5cff7c8bb1dee',
  '0x90F79bf6EB2c4f870365E785982E1f101E93b906': '0xb783e75c6c50486379cdb997f72be5bb2b6faae5b2251999cae874bc1b040af7'
}
leavesValue [
  '0x3f68e79174daf15b50e15833babc8eb7743e730bb9606f922c48e95314c3905c',
  '0xd0583fe73ce94e513e73539afcb4db4c1ed1834a418c3f0ef2d5cff7c8bb1dee',
  '0xb783e75c6c50486379cdb997f72be5bb2b6faae5b2251999cae874bc1b040af7'
]
proofs {
  '0x70997970C51812dc3A010C7d01b50e0d17dc79C8': [
    '0xb783e75c6c50486379cdb997f72be5bb2b6faae5b2251999cae874bc1b040af7',
    '0xd0583fe73ce94e513e73539afcb4db4c1ed1834a418c3f0ef2d5cff7c8bb1dee'
  ],
  '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC': [
    '0xf92db5e3e1d6bed45d8e50fad47eddeb89c5453802b5cb6d944df2f3679da55c'
  ],
  '0x90F79bf6EB2c4f870365E785982E1f101E93b906': [
    '0x3f68e79174daf15b50e15833babc8eb7743e730bb9606f922c48e95314c3905c',
    '0xd0583fe73ce94e513e73539afcb4db4c1ed1834a418c3f0ef2d5cff7c8bb1dee'
  ]
}
      ✔ Should return correct name and symbol
    setMerkleRoot check
      ✔ [S] Should set the Merkle Root correctly by Contract deployer
The initial Merkle root at the time of contract deployment undefined
inputs [
  {
    address: '0x70997970C51812dc3A010C7d01b50e0d17dc79C8',
    quantity: 1
  },
  {
    address: '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC',
    quantity: 2
  },
  {
    address: '0x90F79bf6EB2c4f870365E785982E1f101E93b906',
    quantity: 3
  },
  {
    address: '0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc',
    quantity: 4
  }
]
leaves {
  '0x70997970C51812dc3A010C7d01b50e0d17dc79C8': '0x3f68e79174daf15b50e15833babc8eb7743e730bb9606f922c48e95314c3905c',
  '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC': '0xd0583fe73ce94e513e73539afcb4db4c1ed1834a418c3f0ef2d5cff7c8bb1dee',
  '0x90F79bf6EB2c4f870365E785982E1f101E93b906': '0x8d1187a2c5d69d9d0f6e6c8baf49c9549b9573585daef9b8634509e0cb8d99ae',
  '0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc': '0x9995999b4cf3fbeb689e3cc965ed87e72ff5cb0c750b5e16639e60ac50a9bdd4'
}
leavesValue [
  '0x3f68e79174daf15b50e15833babc8eb7743e730bb9606f922c48e95314c3905c',
  '0xd0583fe73ce94e513e73539afcb4db4c1ed1834a418c3f0ef2d5cff7c8bb1dee',
  '0x8d1187a2c5d69d9d0f6e6c8baf49c9549b9573585daef9b8634509e0cb8d99ae',
  '0x9995999b4cf3fbeb689e3cc965ed87e72ff5cb0c750b5e16639e60ac50a9bdd4'
]
proofs {
  '0x70997970C51812dc3A010C7d01b50e0d17dc79C8': [
    '0x8d1187a2c5d69d9d0f6e6c8baf49c9549b9573585daef9b8634509e0cb8d99ae',
    '0xec8cfb2be54f182816432181c8e4effe0db09107a1e3459b788f06fd9f3d599e'
  ],
  '0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC': [
    '0x9995999b4cf3fbeb689e3cc965ed87e72ff5cb0c750b5e16639e60ac50a9bdd4',
    '0x3c796d3f1fb5030d1d09ecb20dddc5da2479c52d99d12aef90b27fea8d7e6f45'
  ],
  '0x90F79bf6EB2c4f870365E785982E1f101E93b906': [
    '0x3f68e79174daf15b50e15833babc8eb7743e730bb9606f922c48e95314c3905c',
    '0xec8cfb2be54f182816432181c8e4effe0db09107a1e3459b788f06fd9f3d599e'
  ],
  '0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc': [
    '0xd0583fe73ce94e513e73539afcb4db4c1ed1834a418c3f0ef2d5cff7c8bb1dee',
    '0x3c796d3f1fb5030d1d09ecb20dddc5da2479c52d99d12aef90b27fea8d7e6f45'
  ]
}
      ✔ [R] Should not allow setting Merkle Root by Owner
      ✔ [R] Should not allow setting Merkle Root by notOwner
    mint
      ✔ Should allow whitelisted users to mint
      ✔ Should revert when users try to mint over allowed quantity
      ✔ Should revert when non-whitelisted users try to mint


  7 passing (926ms)

Done in 4.38s.npx hardhat verify --network goerli 0x9B97b7bDEFa32b9a26f1Cf27459bcC18281938Ac
```
:::

## コントラクトの Deploy＆Verify
コマンドパターンと`.env`は下記の通りです。
| コマンド                  | 説明                                                |
| ------------------------- | --------------------------------------------------- |
| `npx hardhat run scripts/deploy.ts --network goerli`      | コントラクトをgoerliにデプロイ   |
| `npx hardhat verify --network goerli CA`            | CAにはデプロイした結果のコントラクトアドレスを指定 |

```js: .env
ALCHEMY_API_KEY = "XXXXX"
PRIVATE_KEY = "XXXXX"
ETHERSCAN_API_KEY = "XXXXX"
```

https://github.com/ikmzkRo/WLAQ-mint/blob/main/scripts/deploy.ts

# 課題
## トークン発行後にアドレスと数量の状態を更新できていない
既に指定のアドレスからどれだけトークンが発行されたかという状態を保持する機能が今後必要になる可能性がありますね。

ホワイトリストにウォレットアドレスのみを登録する場合は、以下のような実装となります。トークンを発行した直後、`whitelistClaimed`に`msg.sender`が登録される形です。
```sol
  function whitelistMint(bytes32[] calldata _merkleProof) public payable returns (uint256) {
      require(!whitelistClaimed[msg.sender], "Address already claimed");
      bytes32 leaf = keccak256(abi.encodePacked(msg.sender));
      require(
          MerkleProof.verify(_merkleProof, merkleRoot, leaf),
          "Invalid proof"
      );
      
      _tokenIds.increment();
      uint256 newTokenId = _tokenIds.current();
      _mint(msg.sender, newTokenId);

      whitelistClaimed[msg.sender] = true;
      
      return newTokenId;
  }
```

## マークルルートに対するrequireが不足している
追加のバリデーションとして以下の項目を考慮すると良いでしょう。

- マークルルートが設定されていない
- 既に利用されているマークルルートである

これらの条件を考慮した実装により、より堅牢でセキュアなシステムが構築できます。
```sol
require(root != NULL, "The root is not registered.");
require(!usedMerkleRoots[root], "This root has already used.");
```

# おまけ
マークルツリーを用いたプロジェクトや実装例を貼っておきます。

https://zenn.dev/ryo_takahashi/scraps/ff70f43eb45856#comment-40095f6a46ae37
https://zenn.dev/serinuntius/articles/35c1b6a042174e847766?redirected=1
https://zenn.dev/serinuntius/articles/f56b3dc2871a03?redirected=1
https://zenn.dev/no_plan/articles/581be4ad731a79#merkle-proof%E3%82%92%E7%94%A8%E6%84%8F%E3%81%99%E3%82%8B
https://zenn.dev/0xywzx/articles/bdb6c991f3fc8b

# 感想
- マークルツリー大好き
- NFTのシームレスな配布を目の当たりにしたい
- [zkRollup](https://zenn.dev/serinuntius/articles/f56b3dc2871a03?redirected=1#zkrollup%E3%82%92%E6%8E%98%E3%82%8A%E4%B8%8B%E3%81%92%E3%82%8B) の理解に苦しんでいる。ZKPはホットな話題なので理解を進めていく。

# 参考
https://dev.to/peterblockman/understand-merkle-tree-by-making-a-nft-minting-whitelist-1148#validate-data-using-merkle-tree
https://github.com/learn-co-students/session7-jsong-programming-blockchain-demo/blob/master/index.ipynb
https://zenn.dev/sakuracase/articles/4f58609f3da6e8
https://alis.to/gaxiiiiiiiiiiii/articles/3dy7vLZn0g89