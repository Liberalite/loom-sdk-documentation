---
id: karma
title: Karma
sidebar_label: Karma
---
Karmaモジュールは、トランザクションの制限方法を提供する。 ユーザーは様々なkarmaパラメーターにより、トランザクションの数とタイミングを制限される。 Oracleと呼ばれるユーザーは、アクセス無制限となる。

## インストール

インストールを行うには、チェーン初回スタート時、karmaコントラクトをgenesis.jsonファイルに含めること。

```json
     {
            "vm": "plugin",
            "format": "plugin",
            "name": "karma",
            "location": "karma:1.0.0",
            "init": {
            }
     }

```

## アクティベーションとloom.yaml

Karmaの機能のアクティベートは、loom.yaml設定ファイルにより行われる。

* `KarmaEnabled bool` Karmaモジュールのon/offを設定するフラグ。 
* `KarmaSessionDuration int64` 期間(秒)。Karmaは、`SessionDuration`(秒)の任意のインターバル間にユーザーが設定可能なトランザクション数を制限する。
* `KarmaMaxCallCount int64` `SessionDuration`当たりのcallトランザクション数を計算する為に使う基本値。 `KarmaMaxCallCount int64` が `0` の時は、callトランザクション数に制限がないことを示している。
* `KarmaMaxDeployCount int64` `SessionDuration`当たりのデプロイトランザクション数を計算する為に使う基本値。 A `KarmaMaxDeployCount int64` が `0` の時は、デプロイトランザクション数に制限がないことを示している。 loom.yaml フラフメントのサンプル。

```yaml
KarmaEnabled: true
KarmaSessionDuration: 60
KarmaMaxCallCount: 10
KarmaMaxDeployCount: 5
```

## Karmaメソッドへのアクセス

PublicなKarmaメソッドは、SDKを使ってコード中で直接実行することもできるし、 `loom` エグゼクタブルを使用してコマンドラインから実行することもできる。

```bash
$ ./loom karma --help

call a method of karma

Usage:
  loom karma [command]

Available Commands:
  append-sources-for-user add new source of karma to a user, requires oracle verification
  delete-sources-for-user delete sources assigned to user, requires oracle verification
  get-sources             list the karma sources
  get-total               calculate total karma for user
  get-user-state          list the karma sources for user
  reset-sources           reset the sources, requires oracle verification
  update-oracle           change the oracle or set initial oracle


Flags:
  -a, --address string   address file
      --chain string     chain ID (default "default")
  -h, --help             help for karma
  -k, --key string       private key file
  -r, --read string      URI for quering app state (default "http://localhost:46658/query")
  -w, --write string     URI for sending txs (default "http://localhost:46658/rpc")

Use "loom karma [command] --help" for more information about a command.
```

## Oracle

* これはgenesisファイルにて定義されており、KarmaメソッドのUpdateOracleによって更新することができる。
* Karmaの制限による影響を受けない。
* 次のKarma設定トランザクションの呼び出しを成功させるには、oracleを参照することが必要となる。 
    * `AppendSourcesForUser`
    * `DeleteSourcesForUser`
    * `ResetSources`
    * `UpdateOracle` oracleがまだ設定されていない場合、誰もが `UpdateOracle`を呼び出すことができる。 oracleが設定されている場合、 `UpdateOracle` でoracleを変更するには古いoracleを知っていなくてはならない。 

genesisファイル入力例。

```json
    {
        "vm": "plugin",
        "format": "plugin",
        "name": "karma",
        "location": "karma:1.0.0",
        "init": {
                "Oracle": {
                    "chainId": "default",
                    "local": "QjWhaN9qvpdI9MjS1YuL1GukwLc="
                }
        }
    }
```

###### loom karma update-oracle

oracleは、`loom update-oracle` コマンドを用いて更新することができる。これはKarmaメソッドUpdateOracleを使用する。

```bash
$ loom karma update-oracle --help
change the oracle

Usage:
  loom karma update-oracle (new oracle) [old oracle] [flags]

Flags:
  -h, --help   help for update-oracle

Global Flags:
  -a, --address string   address file
      --chain string     chain ID (default "default")
  -k, --key string       private key file
  -r, --read string      URI for quering app state (default "http://localhost:46658/query")
  -w, --write string     URI for sending txs (default "http://localhost:46658/rpc")
```

oracleを `old oracle` から `new oracle`へ変更する。  
例えば、 default:0xDeffe041cC978a193fCf0CE18b43b25B4592FC90 が現在のoracleだと

```bash
$ karma update-oracle   default:0xAfaA41C81A316111fB0A4644B7a276c26bEA2C9F default:0xDeffe041cC978a193fCf0CE18b43b25B4592FC90  -k ./cmd/loom/data/pri
oracle changed
```

oracle が設定されていない場合は、古いoracle は省略できる。

## ソース

Karmaはソースから生成される。ソースは、異なるユーザーのタイプや特権レベル、もしくはステータスの一時的な変更と一致する。

```go
type KarmaSourceReward struct {
    Name   string
    Reward int64
}
```

`Name`はソースを識別する為に使用される分類文字列である。 `Reward`は、ソースと連想されているKarmaの乗算に使われる。 高い値の `Reward` は、低い値の `Reward` よりもソースにより生成されたKarmaの各ポイントが効果的であることを意味している。

ソースはgenesisファイル中で、または後からKarmaメソッドの GetSources 及び ResetSources を呼び出すことで構成できる。

### ソース: Genesis File

ソースは、DAppChain genesisファイルのKarmaセグメント `init`にて設定できる。 こうして初めてソースが有効となり、DAppChainは実行される。例えば:

```json
    {
        "vm": "plugin",
        "format": "plugin",
        "name": "karma",
        "location": "karma:1.0.0",
        "init": {
            "Params": {
                "sources": [
                    {
                        "name": "sms",
                        "reward": "1"
                    },
                    {
                        "name": "oauth",
                        "reward": "3"
                    },
                    {
                        "name": "token",
                        "reward": "4"
                    }
                ]
            }
        }
    }
```

これはリワードレベルを変えながら、"sms"、"oauth" 及び "token"の３つのソースでDAppチェーンを起動する。

### ソース: ResetSources

Karmaメソッド`ResetSources` は、実行中のDAppチェーンのソースを含めたKarmaパラメーターをリセットする為に使用される。 `UpdateConfig` を使用してKarma構成パラメーターを設定する前に、 `GetSources` で既存のパラメーターをダウンロードし、それを修正することができる。

###### loom karma get-sources

Karmaソースをリストするには、 `loom` コマンドのget-sourcesを使用する。

```bash
$ loom karma get-sources --help
list the karma sources

Usage:
  loom karma get-sources [flags]

Flags:
  -h, --help   help for get-sources

Global Flags:
  -a, --address string   address file
      --chain string     chain ID (default "default")
  -k, --key string       private key file
  -r, --read string      URI for quering app state (default "http://localhost:46658/query")
  -w, --write string     URI for sending txs (default "http://localhost:46658/rpc")

Process finished with exit code 0
```

例

```bash
$ loom karma get-sources
{
  "sources": [
    {
      "name": "sms",
      "reward": "1"
    },
    {
      "name": "oauth",
      "reward": "3"
    },
    {
      "name": "token",
      "reward": "4"
    }
  ]
}
```

###### loom karma reset-sources

`loom karma update-sources` コマンドで到達したResetSourcesのKarmaメソッドを使用して、ソースをリセットすることができる。 既存のソースに追加したい場合、 `loom karma get-sources` を使って既存のソースを取得する。

```bash
$ loom karma reset-sources --help
reset the sources

Usage:
  loom karma reset-sources (oracle) [ [source] [reward] ]... [flags]

Flags:
  -h, --help   help for update-sources

Global Flags:
  -a, --address string   address file
      --chain string     chain ID (default "default")
  -k, --key string       private key file
  -r, --read string      URI for quering app state (default "http://localhost:46658/query")
  -w, --write string     URI for sending txs (default "http://localhost:46658/rpc")

```

例えば、`default:0xDeffe041cC978a193fCf0CE18b43b25B4592FC90` がすでにoracleとして設定されていたとしよう。

```bash
$ loom karma reset-sources default:0xDeffe041cC978a193fCf0CE18b43b25B4592FC90 "oauth" 3 "token" 5 "test" 7
sources successfully updated
```

##### go-loom update sources

次のGoコードフラグメントは、これをgo-loomを使ってGoで行う方法の例を示している。読みやすくするため、エラーチェックはスキップしている。

```go
import `github.com/loomnetwork/go-loom/builtin/types/karma`

func AddSource(name string, reward int64, signer auth.Signer, oracle loom.Address, karmaContact client.Contract) {

    // 既存の構成パラメーターを取得
    var resp karma.KarmaSources
    _, err := karmaContact.StaticCall("GetSources", oracle.MarshalPB(), oracle, &resp)

    // 新しいソースを追加
    var configVal karma.KarmaSourcesValidator
    configVal.Oracle = oracle.MarshalPB()
    configVal.Sources = append(resp.Sources, &karma.KarmaSourceReward {
            Name: name,
            Reward: reward,
    })

    // DAppChain上のソース情報を更新
    _, err = karmaContact.Call("ResetSources", &configVal, signer, nil)
}
```

## ユーザー

Karmaが有効になっていれば、各ユーザーは彼らのKarma配分でトランザクションを制限される。

###### デプロイトランザクション

どんなトランザクションを行うにも、ちょうど１Karmaが必要となる。

`KarmaMaxDeployCount`がゼロであった場合、デプロイトランザクション数に制限はない。

それ以外の場合、ユーザーは `SessionDuration`(秒) の間に `KarmaMaxDeployCount` だけのデプロイトランザクションを実行することができる。

###### Callトランザクション

どんなトランザクションを行うにも、ちょうど１Karmaが必要となる。

`KarmaMaxCallCount` がゼロであった場合、callトランザクション数に制限はない。

それ以外の場合、ユーザーは `SessionDuration`(秒) の間に `KarmaMaxCallCount + total karma` だけのcallトランザクションを実行することができる。 `total karma1` は下で説明しているように、ユーザーのソースカウントより計算される。

### #ユーザーKarma

各ユーザーは、ゼロまたはそれより多いソースと連想付けられる。 このリストは、Karmaソースリストに現在あるアクティブソース、またはKarmaソースリストに現在ない非アクティブソースのどちらも含むことがある。

```go
type KarmaSource struct {
    Name  string
    Count int64 
}

type KarmaAddressSource struct {
    User    *types.Address 
    Sources []*KarmaSource 
}
```

`Name` はソースを識別し、上の `KarmaSourceReward` 中の `Name` フィールドに一致している。 `Count` は、アドレスと連想付けられた特定ソースの数。 Karmaソースがユーザーに提供するのは:

`KarmaSource.Count*KarmaSourceReward.Reward`

Karmaの総量は、ユーザーと連想づけられたアクティブKarmaソースのKarmaを合計したものである。 ユーザーと連想づけられたソースは、genesisファイル中でも、もしくはKarmaメソッドの `AppendSourcesForUser` 及び `DeleteSourcesForUser` によっても構成することができる。

#### ユーザー: Genesis File

genesisファイル中でユーザーとソースを連想づけることが可能である。これにより、ユーザーは新たなdAppチェーンがスタートするとすぐに、有効なKarmaを持つことができる。例えば:

```json
        {
            "vm": "plugin",
            "format": "plugin",
            "name": "karma",
            "location": "karma:1.0.0",
            "init": {
                "Params": {
                    "sources": [
                        {
                            "name": "sms",
                            "reward": "1"
                        },
                        {
                            "name": "oauth",
                            "reward": "3"
                        },
                        {
                            "name": "token",
                            "reward": "4"
                        }
                    ],
                    "users": [
                        {
                            "user": {
                                "chainId": "default",
                                "local": "QjWhaN9qvpdI9MjS1YuL1GukwLc="
                            },
                            "sources": [
                                {
                                    "name": "oauth",
                                    "count": "10"
                                },
                                {
                                    "name": "token",
                                    "count": "3"
                                }
                            ]
                        }
                    ],
                }
            }
        }
```

このgenesisファイルフラグメントは３つのソースを作成し、ローカルアドレス `QjWhaN9qvpdI9MjS1YuL1GukwLc`のユーザーに対して、 `oauth`から10リワード、 `token` から3リワードを与えることになる。 つまりこのユーザーは `10*3 + 3*4 = 42` Karmaでスタートする。

#### ユーザー: AppendSourcesForUser 及び DeleteSourcesForUser

Karmaメソッド `AppendSourcesForUser` を使って、実行中のDAppChainでユーザーにソースを追加することができる。 新しいソースの名前、さらにリワード数カウントのリストが必要となる。 DeleteSourcesForUser でソースを削除することが可能。

###### loom karma append-sources-for-user

Karmaメソッド AppendSourcesForUser を使って、新しいソースをユーザーと連想づけることができるが、これは `loom karma append-sources-for-user` コマンドでアクセス可能となっている。

```bash
$ ./loom karma append-sources-for-user  --help
add new source of karma to a user, requires oracle verification

Usage:
  loom karma append-sources-for-user (user) (oracle) [ [source] [count] ]... [flags]

Flags:
  -h, --help   help for append-sources-for-user

Global Flags:
  -a, --address string   address file
      --chain string     chain ID (default "default")
  -k, --key string       private key file
  -r, --read string      URI for quering app state (default "http://localhost:46658/query")
  -w, --write string     URI for sending txs (default "http://localhost:46658/rpc")
```

たとえば、`default:0xDeffe041cC978a193fCf0CE18b43b25B4592FC90` がoracleであり、 `default:0xAfaA41C81A316111fB0A4644B7a276c26bEA2C9F` が新しいソースを追加したいユーザーであるとする。

```bash
$ ./loom karma append-sources-for-user default:0xAfaA41C81A316111fB0A4644B7a276c26bEA2C9F  default:0xDeffe041cC978a193fCf0CE18b43b25B4592FC90 "oauth" 4 "token" 3 -k ./cmd/loom/data/pri
sources successfully updated
```

###### loom karma delete-sources-for-user

Karmaメソッド DeleteUsersForUser を用いて、既存のソースをユーザーとの連想から切り離すことができる。これを行うために、 `loom karma delete-sources-for-user` を使おう。

```bash
$ loom karma delete-sources-for-user --help
delete sources assigned to user, requires oracle verification

Usage:
  loom karma delete-sources-for-user (user) (oracle) [name]... [flags]

Flags:
  -h, --help   help for delete-sources-for-user

Global Flags:
  -a, --address string   address file
      --chain string     chain ID (default "default")
  -k, --key string       private key file
  -r, --read string      URI for quering app state (default "http://localhost:46658/query")
  -w, --write string     URI for sending txs (default "http://localhost:46658/rpc")
```

たとえば、`default:0xDeffe041cC978a193fCf0CE18b43b25B4592FC90` がoracleであり、 `default:0xAfaA41C81A316111fB0A4644B7a276c26bEA2C9F` がソースを削除したいユーザーであるとする。

```bash
$ ./loom karma delete-sources-for-user default:0xAfaA41C81A316111fB0A4644B7a276c26bEA2C9F  default:0xDeffe041cC978a193fCf0CE18b43b25B4592FC90 "oauth" "token" -k ./cmd/loom/data/pri
sources successfully deleted
```

###### loom karma get-user-state

ユーザーと連想づけられたソースリストを取得するには、loomコマンド `loom karma get-user-state` を使うことができる。

```bash
$ loom karma get-user-state --help
list the karma sources for user

Usage:
  loom karma get-user-state (user address) [flags]

Flags:
  -h, --help   help for get-user-state

Global Flags:
  -a, --address string   address file
      --chain string     chain ID (default "default")
  -k, --key string       private key file
  -r, --read string      URI for quering app state (default "http://localhost:46658/query")
  -w, --write string     URI for sending txs (default "http://localhost:46658/rpc")
```

例えば、ユーザー `default:0xDeffe041cC978a193fCf0CE18b43b25B4592FC90` について情報を取得するには:

```bash
./loom karma get-user-state default:0xDeffe041cC978a193fCf0CE18b43b25B4592FC90
```

## Karmaメソッド

次のメソッドは誰でも呼び出すことができる。

* `GetSources(ctx contract.StaticContext, ko *types.Address) (*ktypes.KarmaSources, error)` これは現在のKarma設定詳細を返却する。現在のソースも含んでいる。
* `GetTotal(ctx contract.StaticContext, params *types.Address) (*karma.KarmaTotal, error)` これはシステム内のKarma総量を返却する。 ソースと連想づけられた各ユーザーのKarma合計。
* `GetUserState(ctx contract.StaticContext, user *types.Address) (*karma.KarmaState, error)` ユーザーと連想づけられたソース及びカウントを返却する。

oracleがまだ存在しなければ、oracleを参照することによってのみ呼び出すことができる。

* `UpdateOracle(ctx contract.Context, params *ktypes.KarmaNewOracleValidator) error`

次のメソッドはoracleを参照することによってのみ呼び出すことができる。

* `ResetSources(ctx contract.Context, kpo *ktypes.KarmaSourcesValidator) error` 有効なソースリストをリセットする。
* `AppendSourcesForUser(ctx contract.Context, ksu *karma.KarmaStateUser) error` ソースの集合をカウント及びユーザーと連想づける。詳細は上記を参照。 
* `DeleteSourcesForUser(ctx contract.Context, ksu *karma.KarmaStateKeyUser) error` ソースをユーザーとの連想から切り離す。詳細は上記を参照。 

他のメソッド

* `Meta() (plugin.Meta, error)`
* `Init(ctx contract.Context, req *InitRequest) error`

## Genesisファイル入力

genesisファイルの入力例を以下に示した。 イニシャルブロックは空で良い。これはDAppChain上にKarmaコントラクトをインストールするだけのものである。 またOracleをgenesisファイル中で定義できる。 定義しない場合も、Karmaメソッド `UpdateOracle` を呼び出してイニシャルoracleを作成することができる。 またKarmaソースのリストでKarmaコントラクトを初期化することが可能である。 こうすると、これらのソースの量を割り当てたユーザーのリストを分配することもできる。

```json
        {
            "vm": "plugin",
            "format": "plugin",
            "name": "karma",
            "location": "karma:1.0.0",
            "init": {
                "Oracle": {
                    "chainId": "default",
                    "local": "QjWhaN9qvpdI9MjS1YuL1GukwLc="
                 },
                "sources": [
                    {
                        "name": "sms",
                        "reward": "1"
                    },
                    {
                        "name": "oauth",
                        "reward": "3"
                    },
                    {
                        "name": "token",
                        "reward": "4"
                    }
                ],
                "users": [
                    {
                        "user": {
                            "chainId": "default",
                            "local": "QjWhaN9qvpdI9MjS1YuL1GukwLc="
                        },
                        "sources": [
                            {
                                "name": "oauth",
                                "count": "10"
                            },
                            {
                                "name": "token",
                                "count": "3"
                            }
                        ]
                    }
                ]

            }
        }
```