---
title: "【Ethereum Staking】StakerとDeposit Contractを仲介する入出金鍵の役割"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# 前提知識
※飛ばしていただいて構いません！

Ethereumが起動した当時はPOWというPCのマシンパワーによって数あてゲームの勝者となるためのマイナーが存在していましたが、
電力やネットワーク維持コスト等の課題が多くあり電力供給量が不安定であるためにネットワーク全体が安定しませんでした。
そこでPOSというステーカーによる正しいブロックを投票することで決定する方式に徐々に移行することでこれらの問題を解決しようとしました。

そういった過程で起動したEthereumはConsensu LayerとExecution layerから成り立っており、このConsensu Layerのことを一般にはBeacon Chainといいます。
これはPOSの初期にデプロイされたブロックチェーンのコンセンサスを扱う領域です。一方でExecution layerはスマートコントラクト等を実行する領域を指します。

POSに参加するためには、Stakerが32ETHをDeposit Contractに入金しネットワークに承認される必要があります。
稼動が承認されたStaker=ValidatorはBeacon Chainを認証するPOSの参加者として稼働することができ運がよければ報酬を受けとることができます。

ちなみにConsensu LayerとExecution layerで発生した報酬をそれぞれCL報酬、EL報酬と呼びます。



















# ref
https://kb.beaconcha.in/ethereum-staking/ethereum-2-keys