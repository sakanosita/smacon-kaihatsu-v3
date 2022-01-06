---
title: "DFINITY入門: Accept cycles from a wallet"
date: 2022-01-03 20:38
permalink: /cycles-hello
tags:
  - tutorial
  - dfinity
  - motoko
  - icp
  - jp
description: |-
  DFINITY(Internet Computer)でキャニスターを動かす際に必要になるCycle Cost
  ICPをCYCLEに変える、ウォレットにCYCLEをチャージ

---

このページは、DFINITYのチュートリアルを日本語で解説しています。

[「Accept cycles from a wallet」](https://smartcontracts.org/docs/developers-guide/tutorials/simple-cycles.html)

実際に使ったソースコードは[GitHub](https://github.com/smacon-dev/motoko-tutorial/tree/main/cycles_hello)からダウンロードできます。

### 実行環境
* dfx: 0.8.4
* macOS: 11.5.2
* npm version: 8.1.3
* 任意のターミナル
* 任意のテキストエディタ

dfxについて知りたい方はこちらをどうぞ

[5ステップではじめるMotokoプログラミング入門](/hello-motoko)

ターミナルは、なんでもよいのでMac標準のターミナルで大丈夫です。
テキストエディタはVisual Studio Codeを筆者は使っています。

## 手順
### プロジェクトの作成

新しいプロジェクトを作ります。

```
dfx new cycles_hello
cd cycles_hello
```

### コーディング

#### src/cycles_hello/main.mo
```ts
import Nat64 "mo:base/Nat64";
import Cycles "mo:base/ExperimentalCycles";

shared(msg) actor class HelloCycles (
   capacity: Nat
  ) {

  var balance = 0;

  // Return the current cycle balance
  public shared(msg) func wallet_balance() : async Nat {
    return balance;
  };

  // Return the cycles received up to the capacity allowed
  public func wallet_receive() : async { accepted: Nat64 } {
    let amount = Cycles.available();
    let limit : Nat = capacity - balance;
    let accepted =
      if (amount <= limit) amount
      else limit;
    let deposit = Cycles.accept(accepted);
    assert (deposit == accepted);
    balance += accepted;
    { accepted = Nat64.fromNat(accepted) };
  };

  // Return the greeting
  public func greet(name : Text) : async Text {
    return "Hello, " # name # "!";
  };

  // Return the principal of the caller/user identity
  public shared(msg) func owner() : async Principal {
    let currentOwner = msg.caller;
    return currentOwner;
  };

};
```

### 起動＆デプロイ

ローカル実行環境を起動
```
dfx start --clean --background
```

キャニスターのビルド＆デプロイ
```
dfx deploy --argument '(360000000000)'
```
```
出力
cycles_hello % dfx deploy --argument '(360000000000)'
Creating a wallet canister on the local network.
The wallet canister on the "local" network for user "default" is "rwlgt-iiaaa-aaaaa-aaaaa-cai"
Deploying all canisters.
Creating canisters...
Creating canister "cycles_hello"...
"cycles_hello" canister created with canister id: "rrkah-fqaaa-aaaaa-aaaaq-cai"
Creating canister "cycles_hello_assets"...
"cycles_hello_assets" canister created with canister id: "ryjl3-tyaaa-aaaaa-aaaba-cai"
Building canisters...
（中略）
Committing batch.
Deployed canisters.
```

キャニスターのownerのPrincipal IDを表示させます
```
dfx canister call cycles_hello owner
```
```
出力
(principal "zr2yi-7hrww-jgne7-j4gbs-2xu5a-ms3wg-ixp3t-4azyp-ifmeb-yxym6-sqe")
```
このPrincipal IDはデプロイしたIdentityのPrincipal IDで以下のコマンドの結果と一致します。
```
dfx identity get-principal
```

## 動作確認
### CYCLESの確認
CYCLEはWalletキャニスターが管理しています。
このプロジェクトに登場するウォレットは2つあります。

* dfxの実行ユーザー用のウォレット
* cycles_helloキャニスター用のウォレット

cycles_helloキャニスターのウォレットの初期残高を表示
```
dfx canister call cycles_hello wallet_balance
(0 : nat)
```

dfxの実行ユーザーのウォレットからcycles_helloキャニスターのウォレットに256,000,000,000CYCLEを移動
```
dfx canister call rwlgt-iiaaa-aaaaa-aaaaa-cai wallet_send '(record { canister = principal "rrkah-fqaaa-aaaaa-aaaaq-cai"; amount = (256000000000:nat64); } )'
```
```
出力
(variant { 17_724 })
```

cycles_helloキャニスターのウォレットの残高を表示
```
dfx canister call cycles_hello wallet_balance
(256_000_000_000 : nat)
```

dfx実行ユーザーのwallet残高を表示
```
dfx canister call rwlgt-iiaaa-aaaaa-aaaaa-cai wallet_balance
```
```
出力
(record { 3_573_748_184 = 91_744_000_000_000 : nat64 })
```

### キャニスターを実行したらどのウォレットからCYCLEが消費されるか
```
dfx canister call cycles_hello greet '("from DFINITY")'
("Hello, from DFINITY!")
```
```
dfx canister call rwlgt-iiaaa-aaaaa-aaaaa-cai wallet_balance
```
```
出力
(record { 3_573_748_184 = 91_744_000_000_000 : nat64 })
```

DFINITYのDocumentでは、このときにdfx実行ユーザーのwalletのCYCLE残高が減少してますが
筆者の環境では、どちらのウォレットのCYCLEも減りませんでした。

詳しく調べて、また後日、更新したいと思います。


### 停止
dfx.jsonがあるディレクトリで以下のコマンドを実行して、実行環境を停止します。
```
dfx stop
```
