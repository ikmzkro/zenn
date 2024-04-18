---
title: "イーサリアムステーキングコントラクト"
emoji: "🥩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---


## 分からないこと

- 32ETHデポジットしてコントラクトからバリデータ鍵有効かしてもらって実際にバリデータ稼働するまでの詳細フロー
- 登場するウォレットの役割分担
- 報酬、ペナルティの理解度あげたい
- WalletConnet→Fireblocksへの連携の詳細
- WalletConnetを利用した実装の詳細

## 目標

- 実際にこのシステムを利用するユーザの立場になれる
- 先方からのしつもんに答えることができる
- 後続チケットを理解しFireblocksとステーキングに関するチケットのでヴバッグができFireblocksの知見が手に入り、手を動かすことができる
- ドリコム案件に入るための整理がつく

- [x]  Ethereum Stakingにおける入出金や鍵の仕組み

---

### EthereumはBeacon ChainとExecution layerから成り立つ。

- [x]  https://zenn.dev/razokulover/scraps/b37ee063f23740
- [ ]  [https://scrapbox.io/bitpickers/Dankshardingとは？](https://scrapbox.io/bitpickers/Danksharding%E3%81%A8%E3%81%AF%EF%BC%9F)
- [ ]  [EIP-4844-OPtimisticRollupについて整理する](https://www.notion.so/EIP-4844-OPtimisticRollup-7c45b1b02fec4e2f9914fca3ce452aab?pvs=21)
- POS Phase0: ビーコンチェーン
    - Consensus Layerともいい、コンセンサスアルゴリズムを行うだけでスマコンの機能は搭載していない。
    - Ethereum UpgradesというPoW→PoSへの移行計画のなかではじめに導入されたチェーン
    - シャードチェーンとステーカー（旧マイナー：32ETHをデポジットしてバリデータとして参戦するユーザ）との調整役
    - 乱数生成に強みがあるので抽出機能に優れている（偏りがなく冪等性がある）
        - ブロックを生成する責任者の抽出
        - シャードチェーンの検証を行うバリデータの抽出
- POS Phase1:  シャードチェーン
    - Ethereum UpgradesというPoW→PoSへの移行計画のなかで２番目に導入されたチェーン
    - 処理性能を向上させる目的で導入されていた、現在は4844にくわれている状況？？
- 既存のEtereumをExecution Layerをいいスマ婚機能を搭載していて、Consensus Layer（ビーコンチェーン）とマージすることでEthereum Upgradesが実現される

---

### ValidatorはBeacon Chainを認証するProof of Stakeの参加者

- ビーコンチェーンはPoS以降計画の初期段階で導入された抽出機能に強みをもつもので、ブロックを生成するステーカーを抽出することができる。ビーコンチェーンによりバリデータとシャードチェーンを調整できる。

### デポジットしてバリデータとなる処理フローをおさらい

- 32ETHを入金用ウォレット（個人、機関）からコントラクトにデポジットする
- バリデータキーが有効化され、ユーザはウォレットで署名してこのバリデータキーをバリデータノード（Staking Service Provider）に保管しに行く
    - 認証やブロック生成で署名する際に利用する
- 入金、出金にも必要になるの？・・・
- バリデータキー、入金ウォレット、引き出しウォレット

引き出しという概念はもともとなかった様子で、現状は必要ないみたい。しかしながら、

新たに32ETHをデポジットする際、Withdraw Credentialを作成する。バリデータを削除、解除する際に必要な出金アドレスをデポジットする際に指定する。

このキーは一度設定した羅変更できない、編集更新が不可能であるためバリデータ停止と新規作成が必要になる。

1. Validator Private KeyをValidator Client側で生成する
2. STakerが32ETHをDepositContractに預金する
3. バリデータ公開鍵、停止時用の引き出しアドレス、32ethをTXoBJ
    1. DepositするETHのAmount, Withdraw Credential、ブロック作成時のtx fee受け取りアドレス
        1. ブロック作成時のtx fee受け取りアドレスが一般的に必要なのか・・・これ認識あってなかったな・・・・
4. バリデータ投票期間、ビーコンチェーンの役割
5. バリデータキュー（今日のMTGで出てきた用語）
    1. Deposit Queueに並ぶ。。。なるほど選ばれたヴぁりデータが待機してる感じね
6. ビーコンチェーン＿

下記を理解せずに画面だけ作るってやっべえぞ。。。

### デポジットのプロセス

バリデータの秘密鍵を用意する、これはホットウォレットでないと署名できるように管理する。

ETH、バリデータ登録解除した際に資産を引き出すためのアドレス（ネットワークにつながってほしくない＝盗まれたら終わりなのでコールドウォレットで管理する）をまとめてデポジットコントラクトに投げると、コントラクトはそのＴｘを検証し始める

４時間待機する

既に複数のバリデータが列をなしている

ここからActiveになった状態になるとビーコンチェーンと呼ばれるPOSphase0で実装された、

乱数生成に優れた投票システムにより、バリデータ選択の候補に入る。

これにより、ビーコンチェーンの乱数生成にヒットする可能性がでてきて、当たった場合はブロックを取り込む責任を負い、

トランザクション手数料のFeeをいただく。

たまってきたら引き出しアドレスから資産を引っこ抜く。

ステータス

1. **Unknown**：いかなるTxもまずはMemopoolという待機室に留まる。
2. **Deposited**：デポジットコントラクトもTxが取り込まれると検証が始まる。
3. **Validator Queue - Status: Pending：**ビーコンチェーン（乱数を生成してバリデータを選択する）に参加する
4. **Active：**正しいブロックの提案とビーコンチェーン承認に参加する＝ステーキングが可能になる
5. **Slashed：**違反があった場合にネットワークから除外される。36日後に引き出し可能にあるらしい
6. **Exited：**１日後には引き出し可能になる

バリデータがブロック提案wできるのは１エポック１回だけみたい

**Attestation rewardは・・・ベット資料にもあるのでそん時また。**

これが一番わかりやすそうやな

https://kb.beaconcha.in/ethereum-staking/ethereum-2-keys

うどんさんの参考資料全部読もうか。

---

- [x]  Ethereum Staking開始機能の設計提案

どうやら既存システムのJSON系背式でのデポジットTx情報のあっぷろー度は手間がかかるよう、複数のバリデータを一括登録できるようにフロントアプリケーション側でループ処理できるようにするみたい。１回のTxで複数のバリデータがデポジットプロセスに進めるようにする。

シーケンスフローは理解した、バリデータの公開鍵生成はニーモニックかな？どうやって作ってるんやろ

ちな、deposit.json, keystore.jsonを自動でバックエンド生成するのでそれも確認だな

- [ ]  デポジットコントラクトで複数デポジットするコード、フロントの渡し方とコントラクトの受け方
- [ ]  バリデータの公開鍵生成コード

- [x]  **BatchDeposit　Staskefidhのコントラクトが参考になるみたい**

https://etherscan.io/address/0x0194512e77d798e4871973d9cb9d7ddfc0ffd801#code#L25

- [x]  Deposit Contract

https://etherscan.io/address/0x00000000219ab540356cbb839cbe05303d7705fa#code

---

- [ ]  Ethereum報酬について

https://kb.beaconcha.in/ を参考にしつつ体系化していく。

1エポック＝ 6.4 分ごとにバリデータはネットワークにブロック認証の提案を行う。

 16,384 のバリデータがネットワークに参加する必要がありここから選択される。

Aggregationというコレクション＝集合体がある。あてステーションを提案するためには

コミッティという委員会が存在する。委員会の人グル＾プにはは１2８件のバリデータが存在する。

→うち１６人がアグリゲータとなる、これはランダムに選択される。

1エポックという最小単位においてバリデータはブロックの提案を行う。

ただし、 16,384 →　128人で構成される委員会に入れられてしまう。

さらに、16人はAgregatorとして選ばれてしまう。

未集計、未承認の証明書をこの16人に私、受け取ったら証明書をマージする

そいつをプロぽーさに転送する

→ビーコンチェーンノードに送る

さらに他の委員会でも同じようなプロ背rスが踏まれる。

マージされた証明書を投げていく

これが繰り返され、 16,384件からaggregatorを除いた全ての投票がビーコンチェーンのノードに集約される。

最後にビーコンチェーンが乱数を生成しバリデータ一つを選択し、ブロック提案者が選抜される。

それによって、ビーコンチェーンのブロックの末尾に追加される。

このような過程でほぼすべてのバリデータがブロックの提案を行うわけだが、受け取ることのできる報酬は下記２点のパラメータに依存する。

1. 基本報酬
2. 包含遅延
    1. 証明書の１個１個はブロック単位で遅延する。・・いや分からん
    2. まあ後にブロック提案するほど遅延する
        1. 半減期レベルで提言していく

報酬得るためには50%以上にいる必要があんのか

https://kb.beaconcha.in/ethereum-staking/glossary#validator

スラッシュって、 同じスロットの 2 つの異なるブロックに署名するとかいうもんなのね

まあ浮気すんなって話

- [ ]  ビーコンチェーン

https://notes.ethereum.org/@djrtwo/Bkn3zpwxB#High-level-overview

1. Attestation報酬

https://kb.beaconcha.in/ethereum-staking/rewards-and-penalties#base-reward

Attestation報酬(CL)(1/2)は成功した際に報酬を貰い、３つに分類される

- Source Casper-FFG 14 -14 Finalize epoch(通常はepoch(N-1))
    - 5slot以内に投票する必要有
- Target Casper-FFG 26 -26 Justify epoch(通常はepoch(N))
    - 32slot以内に投票する必要あり、猶予がない外部んぺなるてぃはおおきくなっている
- Head LMD-GHOST 14 0 Vote block(通常は最新のblock root hash)
    - 1slot以内に投票する必要有

これつまりは報酬の遅延みたいなこと言ってんのかな。

1. Sync Committee報酬

Sync 2 -2 Sync committee参加時のweight

全部で16000件のバリデータがあってそこから委員会単位に分割される。

128バリデータで委員会を組む。

先頭のブロックヘッダをしょうめいし証明を追加するらしい、よお分かってないな

約１日で５１２バリデータがランダムに選ばれる

→64か月に１度選ばれる確率なので、５年に１度の逸材って訳だわ

んで、その委員会に参加した報酬がもらえる。

1. Proposer報酬(CL)

Proposer 8 0 Block Propose時のweight

これが一番で買いはず。

だってブロック提案して報酬もらえるんやで、大きくなかったらやってられへんやろ。

ガス代の優先手数料(Priority fee)を報酬として獲得

→これはEIP16xxとかでやられた八やね。

https://coincheck.com/ja/article/542

- [ ]  MEV

むっずいお

---

- [x]  Staking Validator比較 主にblockdaemon

ビーコンチェーンはコンセンサスレイヤを担うチェーンで、

イーサリアムPOS以降の初期段階で取り込まれたもの。

こいつは乱数生成を行いブロック提案者となるバリデータを抽出する

シャードチェーンは次期段階で取り込まれたチェーンdえ、

処理速度のせい脳を改善するもの。

バリデータステーキングの処理フローについておさらいする。

まずはバリデータが、デポジット、イグジット用のキーペアをそれぞれ用意する。

32ETHをデポジットアドレスに入金する。

メモリプールに待機する

withdrwa credentia; 引き出しアドレス、デポジットアドレスの公開鍵、デポジットする金額

の３つをトランザクションに詰め込んで、バリデータのデポジット用アドレス＝ホットウォレットで

署名を行う

署名はデポジットコントラクトに預けられる。

デポジットコントラクトはTx内容を検証する

ちゃんと理解しましょう。検証するでは不足しすぎている。

トランザクションは待機列を作り、承認を受け取ったバリデータから順に並ぶ

そして承認されたバリデータはACTIVE状態になりステーキングが可能になる。

この状態では32ETHをデポジットしてブロックの提案を行うフェーズに入る。

承認をうけネットワークに参加したブロックの提案者はepoch事に

きめられた提案を行う。

１６０００件のバリデータが参加するには管理しにくいので、、、グループに分けていく。

この単位をコミッティと呼ばれる。

コミッティ委には１２８人のバリデータが参加する。

そのうちの１６名はアグリゲータと呼ばれ、バリデータからのブロック証明書をマージして

ビーコンチェーンノードにブロードキャストする

そして、全てのコミッティからのブロードキャストから１つを

ビーコンチェーンが乱数生成によって割り当て、ブロックを確定させるただ一人のバリデータが誕生する。

バリデータは報酬を受け取ることが可能になり、

証明報酬、コミッティ報酬、提案報酬の３種類がある。スロット遅延により報酬のペナルティが変わる。

取り込まれたらガス代手数料ガウンたらで報酬が入る模様。

とりまバリデータが報酬を受け取る確率はとても低い・・・

違反さえしなければ大丈夫なんだよな。基本マイナスにはならないはず。

ValidatorはBeacon Chainを認証するProof of Stakeの参加者という役割をまだ理解できていないな

嫌理解した、ビーコンノードに提案していくあの流れがBeaconChainを承認するってことかOK

ちな、ここまではあくまでコンセンサスレイヤであって実際のスマ婚機能はExecutionLayerにゆだねられる

デポジットコントラクトについてはこちらを参照。

https://etherscan.io/address/0x00000000219ab540356cbb839cbe05303d7705fa#code

```vbnet
interface IDepositContract {
    /// @notice A processed deposit event.
    event DepositEvent(
        bytes pubkey, バリデータの公開鍵
        bytes withdrawal_credentials,　バリデータ委解除維持用の引き出しアドレス
        bytes amount,　デポジット額
        bytes signature,　デポジット額、ガス代受取アドレス、withdrawal_credentialsのOBJに対してpubkeyのprivkeyで署名したもの
        bytes index バリデータID
    );
```

```vbnet
    /// @notice Submit a Phase 0 DepositData object.
    /// @param pubkey A BLS12-381 public key.
    /// @param withdrawal_credentials Commitment to a public key for withdrawals.
    /// @param signature A BLS12-381 signature.
    /// @param deposit_data_root The SHA-256 hash of the SSZ-encoded DepositData object. //  マークるツリーのルートハッシュカナ。リーフが何か分からん
    /// Used as a protection against malformed input.
    function deposit(
        bytes calldata pubkey,
        bytes calldata withdrawal_credentials,
        bytes calldata signature,
        bytes32 deposit_data_root
    ) external payable;
    
        /// @notice Query the current deposit root hash.
    /// @return The deposit root hash.　ルートハッシュは初回デポジットジにセットするらしい。なんのハッシュだろう。リーフが何か分からんとなあ。。。
    function get_deposit_root() external view returns (bytes32);

    /// @notice Query the current deposit count.
    /// @return The deposit count encoded as a little endian 64-bit number.
    function get_deposit_count() external view returns (bytes memory);
```

まあだいたいは大丈夫

検証の差異にどんなリーフ使ってるのかはまだ未確認

withdrwa credentialについて

ちな、バリデータ完了時における報酬受取の差異に引き出しアドレスによっての署名は不要になる

これは、引き出しプロセスにおいてWithdraw Credentialの提出があるかららしい

ちな、ステークを行う際に３２ETHデポジットするがｍこの際にWithdraw Credentialを提出する

Ｔｘに詰めてブロードキャストする。

このWithdraw Credentialと手数料受取用アドレスとデポジット３２ＥＴＨをＴｘに詰めてブロードキャストするので

提出済みのWithdraw Credentialがあれば引き出し時に署名は必要ない。。。


ここにある参考文献読むだけで全然違うな

鍵の生成過程

https://kb.beaconcha.in/ethereum-staking/ethereum-2-keys#extended-overview-of-ethereum-staking-keys

ここでもあるがコールドウォレットでの署名は行わないのでFbは特に不要だな

なぜSTIRではFBつかってるのか不思議でしかないな


https://zenn.dev/mizuneko4345/scraps/163ec9fbb9dc3a