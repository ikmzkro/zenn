# Zenn CLI

* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

👇  新しい記事を作成する
$ npx zenn new:article

👇  新しい本を作成する
$ npx zenn new:book

👇  投稿をプレビューする
$ npx zenn preview



Next...
- イーサリアムバリデータの理論概要
- イーサリアムバリデータステーキングコントラクトの実装、デプロイ、運用
- サッカーベットコントラクト
- EigenLayerの理論概要
- NativeTokenとERC20Tokenの入出金コントラクトの実装、デプロイ、運用
- :

Todo

# Sola
- `NAT`の枠割を理解してアーキテクチャに書けない
- `Index`を貼ってSQLを最適化する意味を理解できない、書けない

# SBINFT
- `queue`処理のシーケンス処理が書けない
- `nonce`管理を行うデータベース設計ができない
- `auction Contract`が書けない
- `saleOrder, buyOrder` のマーケットプレースコントラクトが書けない
- カストディ事業者とサーバ事業者との連携、要件定義、詳細設計
- `SBT`書けない
- `Polygon`送金コントラクトが書けなかった（アドレス複数に対して1Matic）
- `Ginco`が何やってるかわからん

# Questry
- `DID/VC`をSoliditで書けない、応用できない
- `SBT`書けない
- `nonce`管理を行うデータベース設計ができない
- `gas`の最適化を設計できない、書けない
- `estimateGas`を成功させたことがない
- `ERC2771`のバックエンドが自力で書けない
- `AWS KMS`を用いたキーローテーション設計ができない、実装できない
- `AWS ECS ECR RDS ELB SG`を用いた本格的なアーキテクチャ構築ができない
- `__UUPSUpgradeable_init`なContractを書いたことがない
- `Docker`を理解していない
- `Prisma`とかのORMの役割を理解していない、コマンドで誤魔化してる
- `typechain`をうまく利用できている感覚がない
- `web3auth`がなぜMPCとして不完全なのか？
- valueをsolidityに渡す際に16進数にする理由
- `SlowMist`でのオンチェーン解析
- Fireblock, Sygna, Chainalysysを用いたカストディシステム構築に向けた要件定義、アーキテクチャ設計ができない

# STIR
- `Akira`さんのレビューを全て起票する
- `Naoto Sato`さんのバックエンド実装を参考にして理解度の差分を埋める
- `ERC20`送金コントラクトが書けない、テストが特に書けない
- `ERC20`のテスト用コントラクトアドレスがわからない
- `rawTransactinon`, `maxPriorityGas/Fee`がバックエンドで書けない
- `Docker`何もわからん
- providerがlocalhostに向いているとdockerでfailする理由
- `CICD`を使ったスマコンのテスト自動化が書けない
- `Fireblocks SDK`のStakingデバッグができない

# Turingum
- `Staking Contract`の基盤が書けない
- `Wallet Connect`のシーケンス図を`RPC URL`を含めて記載できない
- `POS`の概要を完全に体系化できていない
- `Fireblocks SDK`を用いて作業を効率化できていない
- `ERC4626, 7540`を用いたVaultのコントラクト設計、実装ができない
- `EigenLayer`のPOCが実装できない
- `EIGEN`トークンや`Eigen DA`の解像度がまるで低い
- `Optimistic Rollup`, `Zk Rollup`がわからない、実装に活かしてガスコストを削減できない
- `Fireblocks`用の`Azure SGX`環境のアーキテクチャ設計、構築処理
- `Gnosis Safe`ウォレットの構築

# Hachathon
- EigenLayer Hachathon
  - 英語でのプレゼン
- Chiliz Sports Hackathon
  - `Pyth`, `Azuro`などに挑戦できない
  - `4337`のPOCが実装できない
  - `docker`を利用したローカルノードのビルドができない
  - `Merkle Root`を利用したコントラクトの実装、検証ができない
- ETH Tokyo 2024 
  - `GraphQL`で`ENS`サブドメイン情報を取得できてかもしれない
  - `EOA`でできること、できないこと、`CA`でできること、できないことが整理できなかった
  - `2771`と`MultiCall`の併用時の脆弱性
  - `We found a bug of cabinet RPC URL.I discovered a bug where meta-transactions are not processed correctly when using the Cabinet RPC URL.Specifically, I was unable to generate TypedSignedData that includes a tuple of objects.*Note: The same code worked correctly when using the Alchemy RPC URL.`
  - `decodeInputData`の使い方がわかってない
  - `tx receipt`で`Event`をキャッチしていく実装が自力で書けない
  - 脳死で`ChatGPT`に聞かずに、`GithubCopilot`でコードを書く訓練
  - txに`{gasLimit; 20000}` とやると reverted txが治る理由、gaLimitとは？んで適正価格は？
  - event発行は`The Graph`でのオンチェーン解析に繋がる
  - `OZDefender`以外のMetaTxのRelayer Serverの構築
  - `INTMAX`がどのようにMPCとして機能しているのか
  - `pinata API`を用いた画像、メタデータの自動保管
  - `Safe`とTobanの連携
  - `Superfluid`とTobanの連携

# OSS
- mint-rally
  - `ZKP`を用いたAPIのPOCを実装できない 
- Toban
  - ETH Tokyo 2024 と同様

---

できないことを以下のカテゴリ別に整理してみました。それぞれのカテゴリに属する内容を分けて記載しています。

### 1. **インフラ・アーキテクチャ**
- NATの役割を理解してアーキテクチャに反映できない
- AWS KMSを用いたキーローテーション設計ができない、実装できない
- AWS ECS, ECR, RDS, ELB, SGを用いた本格的なアーキテクチャ構築ができない
- Dockerを理解していない
- Dockerでのローカルノードのビルドができない
- Azure SGX環境のアーキテクチャ設計、構築ができない
- OSSツール（e.g., SlowMist）を用いたオンチェーン解析ができない

### 2. **スマートコントラクト**
- ERC20, ERC721, ERC2771などのコントラクトの実装ができない
- Auction Contractの実装ができない
- SaleOrder, BuyOrderのマーケットプレースコントラクトが書けない
- Polygon送金コントラクトが書けない（複数アドレスに対して1Matic）
- DID/VCをSolidityで実装・応用できない
- SBT（ソウルバウンドトークン）が書けない
- __UUPSUpgradeable_initなContractを書いたことがない
- Staking Contractの基盤が書けない
- ERC4626, 7540を用いたVaultのデプロイができない
- EigenLayerのPOCが実装できない
- Merkle Rootを利用したコントラクトの実装、検証ができない

### 3. **ブロックチェーン技術とセキュリティ**
- Nonce管理を行うデータベース設計ができない
- Gasの最適化を設計できない
- EstimateGasを成功させたことがない
- Optimistic Rollup, Zk Rollupを理解し、ガスコスト削減に活用できない
- `EOA`と`CA`でのできることの整理ができていない
- MultiCallと2771の併用時の脆弱性について理解していない
- `INTMAX`や`web3auth`がどのようにMPCとして機能しているのか理解できていない

### 4. **バックエンド開発**
- Queue処理のシーケンス処理が書けない
- Prismaやその他ORMの役割を理解していない
- Fireblocks SDKを用いたStaking実装ができない
- `GraphQL`で`ENS`サブドメイン情報を取得できていない
- `decodeInputData`の使い方がわかっていない
- CICDを使ったスマートコントラクトのテスト自動化が書けない
- RawTransactionやmaxPriorityGas/Feeの実装ができない
- Fireblocks SDKを用いて作業を効率化できない

### 5. **プロジェクト管理・要件定義**
- カストディ事業者とサーバ事業者との連携、要件定義、詳細設計ができない
- Wallet Connectのシーケンス図をRPC URLを含めて記載できない
- POSの概要を完全に体系化できていない
- `Ginco`が何をしているか理解できない

### 6. **ハッカソン・OSS活動**
- 英語でのプレゼンができない
- PythやAzuroに挑戦できない
- 4337のPOCが実装できない
- The Graph以外のMetaTxのRelayer Serverの構築ができない
- SafeとTobanの連携実装ができない
- SuperfluidとTobanの連携実装ができない
- ZKPを用いたAPIのPOCが実装できない

この整理をもとに、改善したい分野や重点的に学びたい技術を見つけることができます。
- 
