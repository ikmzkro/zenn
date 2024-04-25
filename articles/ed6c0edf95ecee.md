---
title: "【Ethereum Staking】StakerからDeposit Contractへの入出金プロセス"
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
### Unknown
署名されたDeposit TxがMempool(待機室)に滞留した状態。
指定したガス代が高ければ待機時間は短くなる

### Deposited
一度でもDepositContractにDepositTxが取り込まれるとバリデータのステータスはUnknownからDepositedに遷移しますが、下記理由により最低でも13.6時間かかります。

- validatorは1slot = 12secで1block作成する
- validatorは1epoch(slotよりもlongtermな単位) = 32slot = 32blockを生成するまで384sec=6.4minかかる
- 2048block+64epoch待機した後、beaconChainがDepositを認識する。
  - 2048 x 12秒 = 24,576秒 = 409.6分 = 約6.82時間
  - 64 x 6.4分 = 409.6分 = 約6.82時間



# Withdraw Credential
Withdraw CredentialはStakerが32ETHをDeposit Contractに入金する際に指定する引出認証情報です。
Deposit Contractに送信するWithdrawal Credentialsにはprefixとして0x00と0x01のいずれかが組み込まれます。

0x00は旧式のWithdrawal Credentialsを示し、0x00にBLS公開鍵のハッシュが後続する形式をとります。
0x01はバリデータが正しく引出鍵を設定していることを示し、0x01に11byteのゼロとEthereumAddressが後続する形式をとります。

旧式(0x00)のままでは正しく引出アドレスを設定できていないため、Copella upgrades以降はBLSToExecutionChangeをコールし0x00から0x01に移行するマイグレーションを実行する必要があります。
引出アドレスはEOAもCAも両方指定可能らしいです。

Withdraw Credentialは一度設定しまうと更新不可能となるため、Stakerとなる際に用意がひつようになる。
仮に更新したい場合はValidatorを停止した後、預け入れたETHを全額引き出した後再度デポジットさせる必要があります。

## The Merge
冒頭にも説明したConsensus LayerとExecution layerの統一化をThe Mergeと呼び、下記変更がありました。
- Shanghai upgrades: 主にEthereumのEL(Execution Layer)を対象とした更新(EIP-4895)
- Copella upgrades: 主にEthereumのCL(Consensus Layer)を対象とした更新

これらのShanghai/Capella以降、withdrawalsはpartial, fullの２種類存在します。

partialはポジション（初期投資額32ETH+報酬額）のうち報酬額のみを引き出すことができます。
しかし、引出額や引出タイミングは指定できません。
これは、定期的に残高を監視し報酬額を引出鍵に自動送金する仕組みとなっているためです。

一方でfullはポジション（初期投資額32ETH+報酬額）全額を引き出すことができます。
バリデータノードが自主的にネットワークから退出する、もしくはSlashingというペナルティを犯した場合に引出鍵へのアクセスが可能となります。

# ref
https://kb.beaconcha.in/ethereum-staking/ethereum-2-keys
https://mainnet.earthether.org/en/withdrawals
https://www.youtube.com/watch?v=RwwU3P9n3uo
https://kb.beaconcha.in/ethereum-staking/deposit-process