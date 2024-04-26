---
title: "【Ethereum Staking】ERC20で投票するStakingコントラクト"
emoji: "🥩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ethereum", "crypto"]
published: false
---

# TL;DR


# 前提知識
※Ethereum Stakingで扱う鍵の役割に関する知見をつけるにあたり最低限必要な理解を行うためのセクションですので飛ばしていただいて構いません！

Ethereumが起動した当時はPOWというPCのマシンパワーによって数あてゲームの勝者となるためのマイナーが存在していましたが、
電力やネットワーク維持コスト等の課題が多くあり電力供給量が不安定であるためにネットワーク全体が安定しませんでした。
そこでPOSというステーカーによる正しいブロックを投票することで決定する方式に徐々に移行することでこれらの問題を解決しようとしました。

そういった過程で起動したEthereumはConsensu LayerとExecution layerから成り立っており、このConsensu Layerのことを一般にはBeacon Chainといいます。
これはPOSの初期にデプロイされたブロックチェーンのコンセンサスを扱う領域です。一方でExecution layerはスマートコントラクト等を実行する領域を指します。

POSに参加するためには、Stakerが32ETHをDeposit Contractに入金しネットワークに承認される必要があります。
稼動が承認されたStaker=ValidatorはBeacon Chainを認証するPOSの参加者として稼働することができ運がよければ報酬を受けとることができます。

ちなみにConsensu LayerとExecution layerで発生した報酬をそれぞれCL報酬、EL報酬と呼びます。

# StakerとDeposit Contractを仲介する入出金鍵の役割
鍵にはValidator KeyとWithdraw Keyの２種類あり、それぞれにPrivate Key(秘密鍵)とPublic Key(公開鍵)が存在します。

鍵生成の全体像は下記のようになります。
![](https://storage.googleapis.com/zenn-user-upload/d059495e8c14-20240425.png)

## Validator Key
バリデーター署名キーはバリデーター秘密鍵、バリデータ公開鍵の２つで構成されます。

鍵の役割は下記のようになります。
![](https://storage.googleapis.com/zenn-user-upload/d43224dd9804-20240425.png)

バリデーター秘密鍵はブロック提案や証明などのオンチェーン操作にアクティブに署名する役割を担うため、
常時ネットワークに接続されたホットウォレットに保持するのが通例です。

鍵管理に関しては注意する必要があり、仮に流出した場合はスラッシングリスクを意図的に負うことやバリデータキーの解除をすることで
DepositContractに預けた残高を減額、アクセス不可とされる可能性があります。

バリデーターの公開鍵はDeposit data (tx)に含めることでバリデーターを識別させるような役割を担います

## Withdraw Key
バリデーター出金キーは出金用秘密鍵、出金用公開鍵の２つで構成されます。

鍵の役割は下記のようになります。
![](https://storage.googleapis.com/zenn-user-upload/7fc0b73a84bc-20240425.png)

ただし、Withdraw Keyは旧概念であり現在は`Withdraw Credential`と呼ばれる引出認証情報を元にステークしたETHを出金するプロセルを取ります。つまり出金時に出金用秘密鍵で署名生成を行うフローは不要です。

# 32ETHをDeposit Contractに投げてからValidatorが稼動するまで
Validatorがネットワーク内で稼働するまでの流れは下記のようになります。

![](https://storage.googleapis.com/zenn-user-upload/bd37fef8c9f3-20240425.png)
https://kb.beaconcha.in/ethereum-staking/deposit-process

## Stakingできるようになるまでの処理フロー
1. Validator Keyの秘密鍵をValidator Client側で生成
2. 32ETH以上を保持したEOAでDepositTxを生成しDeposit Contractにbroadcast
    1. txData(預金額、Withdraw Credential、ブロック作成時のTx手数料受取アドレス)を作成する。
    2. △: Validator Keyの公開鍵で署名しValidator Keyの公開鍵を取引データに含める
    3. △: 32ETH以上持っているアドレスでDeposit Txに署名する
3. Deposit txがブロックに取り込まれる
4. △: 最大4時間まつ
5. Deposit Queueに並ぶ。
6. Validatorの稼働状況がACTIVEになる


△: また、2-3までの間に、 Signed Voluntary Exit transaction data (Validatorがずっとオフラインであったり、Validatorが悪意ある行動をした際にネットワークに送信する、署名済みの全額引き出しtx)を取得しておく必要がある。

## Validatorのステータス
| ステータス                  | 説明                                                |
| -------------------------  | --------------------------------------------------- |
| `Unknown`                  | 署名されたDeposit TxがMempool(待機室)に滞留した状態。指定したガス代が高ければ待機時間は短くなる  |
| `Deposited `               | 一度でもDepositContractにDepositTxが取り込まれるとバリデータのステータスはUnknownからDepositedに遷移しますが、下記理由により最低でも`13.6時間`かかります。validatorは1slot=12secで1block作成するので、validatorは1epoch(slotよりもlongtermな単位)=32slot=32blockを生成するまで384sec=6.4minかかります。2048block+64epoch待機した後、beaconChainがDepositを認識するため、(2048 x 12秒 = 24,576秒 = 409.6分 = 約6.82時間) + (64 x 6.4分 = 409.6分 = 約6.82時間) 13.6時間となります。|
| `Deposit Invalid` | トランザクションには無効なBLS署名があった状態です。 |
| `Pending` | ビーコンチェーンからデポジットにアクセスできるようになり、Queueに並びます。Mempoolでガス代に応じて待機時間が短くなったように、Queueでは合計デポジット額に応じて待機時間は短くなります。1epochで8バリデータをアクティブ化することができるため、6.4minで8バリデータなので、1日あたり1800バリデータをアクティブ化できます。 |
| `Active` | ACTIVEになったバリデータはステーキングを実施しブロックの提案や証明書への署名を行い投票によりETHを獲得する可能性を秘めます。|
| `Active Offline` | ACTIVEになったにも関わらずバリデータが2epoch=12.8minの間オフライン状態になりブロックの提案や証明を行っていない状態です。|
| `Exiting Online` | 下記理由によりバリデータはオンラインだがネットワークから退出している最中の状態です。①Deposit Contractに預けた資金が16ETHを下回り強制退場となった場合（Slashing, Penaltyにより減額され続けた等）②バリデータ側は自主的にステーキングをやめる自主退場の場合 |
| `Exiting Offline` | `Exiting Online`のオフラインバージョンです |
| `Slashing Online` | バリデータはオンラインですが、悪意のあるペナルティ行為があったため、ネットワークから強制終了された状態です。これは1epochないで同じブロックに投票するなどの違反行為をさします。 |
| `Slashing Offline` | `Slashing Online`のオフラインバージョンです。また、バリデータは最低限25分間待機Queueに並ぶことになります。しかしSlashingによりペナルティを受け減額された後ですのでStaking額が減り待機時間は以前よりも長くなります。 |
| `Slashed` | 度重なるペナルティを犯した場合、バリデータがネットワークから追い出された状態です。DepositContractに預けた資金は 36 日後に引き出すことができます。|
| `Exited` | バリデータがネットワークから離脱した状態です。DepositContractに預けた資金は 1 日後に引き出し可能になります。 |

# Withdraw Credential
Withdraw CredentialはStakerが32ETHをDeposit Contractに入金する際に指定する引出認証情報です。
Deposit Contractに送信するWithdrawal Credentialsにはprefixとして0x00と0x01のいずれかが組み込まれます。

0x00は旧式のWithdrawal Credentialsを示し、0x00にBLS公開鍵のハッシュが後続する形式をとります。
0x01はバリデータが正しく引出鍵を設定していることを示し、0x01に11byteのゼロとEthereumAddressが後続する形式をとります。

旧式(0x00)のままでは正しく引出アドレスを設定できていないため、Copella upgrades以降はBLSToExecutionChangeをコールし0x00から0x01に移行するマイグレーションを実行する必要があります。
引出アドレスはEOAもCAも両方指定可能らしいです。

Withdraw Credentialは一度設定しまうと更新不可能となるため、Stakerとなる際に用意がひつようになる。
仮に更新したい場合はValidatorを停止した後、預け入れたETHを全額引き出した後再度デポジットさせる必要があります。


The Merge＝Consensus LayerとExecution layerの統一化により下記変更がありました。
- Shanghai upgrades: 主にEthereumのEL(Execution Layer)を対象とした更新(EIP-4895)
- Copella upgrades: 主にEthereumのCL(Consensus Layer)を対象とした更新

これらのShanghai/Capella以降、withdrawalsはpartial, fullの２種類が存在します。

partialはポジション（初期投資額32ETH+報酬額）のうち報酬額のみを全額引き出すことができます。
しかし、引出額や引出タイミングは指定できません。これは、Withdraw addressという引出アドレスを設定すれば、プロトコルレベルで定期的に残高を監視し報酬額を引出鍵に自動送金する仕組みとなっているためです。

△Voluntary Withdraw＝Fullかな
1ブロックあたりPartial WithdrawやVoluntary Withdrawのtxは16件存在し、Tx手数料は一切かからないらしい

一方でfullはポジション（初期投資額32ETH+報酬額）全額を引き出すことができます。
バリデータノードが自主的にネットワークから退出する、もしくはSlashingというペナルティを犯した場合に引出鍵へのアクセスが可能となります。

Active Validatorから抜けだすイグジットメッセージを送る
Exit Queueの最後尾に並び承認を〇
1epochで最大7Validator, １日あたる1,575Validator退出可能、このプロセスでも報酬は発生している
通常自主的な退出であれば１日以降で引出可能になる
Withdraw Queueに入り５日待機するば引き出しが完了する、同じくトランザクション手数料は一切かからない。

# Stakerへの報酬と罰則

https://kb.beaconcha.in/ethereum-staking/rewards-and-penalties

https://zenn.dev/mizuneko4345/scraps/ca1ba05bf94bce


- 報酬とは何が起因で発生するのか
- Consensu LayerとExecution layerで発生した報酬をそれぞれCL報酬、EL報酬と呼びます。

# CL報酬
CL報酬は、Attestation報酬、Sync Committee報酬、Proposer報酬の３種類がある
プロトコルにはGasperを扱う。
https://ethereum.org/ja/developers/docs/consensus-mechanisms/pos/gasper/

## Attestation
https://kb.beaconcha.in/ethereum-staking/attestation

1epoch = 約6.4分毎にバリデータはネットワークに認証、正しいブロックの提案を行う。
このブロックの提案が正しければ責任を負った対価として報酬をいただける。

Consensu Layer(CL)のAttestation報酬はSource、Target、Headに分類される。
△：Source、Targetは報酬と罰則がついになる、Headには報酬=0であるため罰則という概念がそもそもないらしい

Sourceはぶろっくがならんでいるなかでも前のcheckpointに要因する5slotいないに投票する制限がある。
Targetは最新のcheckpointで32slotいないに投票する
HEADは最新のslotで1slotいないに投票する

```
{Source Vote}{}{}{}{}{}{}{}{} ~
~{Target Vote}{}{}{}{}{}{Chain head vote}{Chain head vote}{}{}
```

## Sync Committee
ネットワークに参加したバリデータは委員会を作成し
その委員会に班長が存在する＝Agreegator
んでAgreegatorはバリデータの投票（どのブロックは正しい）といった内容を
マージしてネットワークに伝番する役割を担う
そしてそのなかで１つのバリデータが選ばれブロックを書き込むことができるようになる

---

1epoch6台まで待機中のvalidatorがネットワークに入ってくる(epoch単位でしかvalidatorは入れない)
1epoch6台=6minなので、1min1台入るイメージ
ネットワークにすでに参加している約40万台のvalidatorは
次のepochに入ってくるvalidatorを乱数を基準にランダムに選択する

これからの1epoch=32slot=32block=6分間を承認する
validatorがネットワークから選ばれる

validatorはslotの単位に割り当てられる
各slotにはcommittee(クラス)という単位がある

commitee(クラス)は最低でも128存在している
1slotあたり最大committeeは64
担当する台数が1slotあたり1万近く存在する

commiteeの中にも各validatiorの意見をマージしてネットワークに伝搬するagreegator(班長)がいる
validator Aはブロック正しい、validator Bはブロック間違ってるなどの意見をまとめてる
この伝搬が遅れれば遅れるほど報酬が下がる
そして間違ったブロックに繋ぐ=not heaviestと、鎖が繋がっていかないので報酬が下がる

commiteeは、ネットワーク伝搬委員会に選出された際に支払われる報酬
先頭のブロックヘッダを監視し認証し、そのブロックヘッダに署名を追加する。
256epoch=27.3hに512validatorが選択される。
当選されたバリデータはそれ以降の256epochで全slotに投票義務が課せられる
また、64ヶ月に1度当たる確率らしい

## Proposer
ブロックを提案し承認された際に支払われる報酬
1epoch=6.4minあたり12台のバリデータが選出される
これは1日に3200台選ばれるのと同値で、4ヶ月に一度当選する確率らしい

ブロックの提案には失敗という概念は存在せず承認されたかされなかったか
なので、報酬という概念のみ存在する。違反による罰則は基本ない

1epoch=32soltの各スロットに対してブロックを提案していく

ここまでが、Beacon Chainを認証するPOSの参加者（Validator）の動き
とそれに対する報酬の整理となる。

https://kb.beaconcha.in/ethereum-staking/attestation

# EL報酬
ブロックの提案報酬とMEV報酬の2種類が存在する、
またいずれかの方式を採用するため報酬は片方でしか発生しない。

## ブロックの提案報酬
ブロックを提案し承認された際に発生する報酬
1epoch32台のバリデータが選出される、これは1日で7200台の計算
報酬はtx作成者が指定したガス代の優先手数料から得ることができる

1epochには32slotあり、
mempoolに滞留したtxが抽出され
各スロットのブロックへと内包されていく

この内包されていったブロック内のtxを
proposerが選択していくのだが、
これが中央集権的であるという課題がある。

このtxにはtxdata, basegasfee, priority fee
が詰め込まれておりそのうちpriority feeのがproposerの報酬額となる
txは複数あるため、tx.priority feeの合計額が報酬額全体となる

## MEV報酬
proposerがslotにまびくtxを中央集権的に選択するかだいを解決するために、
選択権力を分散化させ、得られるgasfeeを最大化するための
MEVBOOSTと呼ばれる仕組みを用いることで得られる報酬

# 罰則(ペナルティ)
Attestationの失敗によるペナルティには複数種類がある
バリデータがオフラインになっており正常に稼働していないと判明した時
正しくないブロックに投票していたことでチェーンが伸びない
期間内に制限が設けられる状況下で投票が間に合わなかったとき

CLの報酬、罰則のルールの参考になるものはこれ
- Source
 - 前のcheckpoint
 - 5slot以内に投票
- Target
 - 最新のcheckpoint
 - 32slot以内に投票
- HEAD
 - 最新のslot
 - 1slot以内に投票

バリデータがオフラインになっておりブロックへの投票をサボっていた場合、
正式に稼働していたはずの報酬の7.5割を失う、失いすぎやろ

# Slashing
- attestatin時の二重投票によるもの
- proposer時に同一slot内でブロックを2つ提案する
- attestatin、proposer時に過去の投票内容に矛盾点が生じる場合に発生する
　- これは誤ったブロックの提案、投票などによるものかな
- ペナルティは8,192epoch=36dayの資金ロックと1/32の資金が直ちにバーンされる、max1ethバーンなの？
- 折り返し日には追加ペナルティが発生しちゃう
- slashingによる報酬は基本発生しない
- slashingは発生させてはいけないリスク
- ステーキング額が16eth以下となった場合の強制退場とはまた別だが、slashingが起因ではそうなり得る



# Stakingコントラクト

## Re-Entrancy
動作確認用にETHを入出金するコントラクトで脆弱性を検証してみます。

https://solidity-by-example.org/hacks/re-entrancy/





# EigenLayerへのアプローチ

# ref
https://kb.beaconcha.in/ethereum-staking/glossary
https://kb.beaconcha.in/ethereum-staking/ethereum-2-keys
https://mainnet.earthether.org/en/withdrawals
https://www.youtube.com/watch?v=RwwU3P9n3uo
https://kb.beaconcha.in/ethereum-staking/deposit-process
https://kb.beaconcha.in/ethereum-staking/rewards-and-penalties
https://kb.beaconcha.in/ethereum-staking/attestation
https://www.youtube.com/live/BssxTeTwK58?si=MGuaIZEAj0eX8T_a


