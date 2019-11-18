[![FIWARE Banner](https://fiware.github.io/tutorials.Administrating-XACML/img/fiware.png)](https://www.fiware.org/developers)

[![FIWARE Security](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/security.svg)](https://github.com/FIWARE/catalogue/blob/master/security/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Administrating-XACML.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://nexus.lab.fiware.org/repository/raw/public/badges/stackoverflow/fiware.svg)](https://stackoverflow.com/questions/tagged/fiware)
[![FIWARE Security](https://img.shields.io/badge/XACML-3.0-ff7059.svg)](https://docs.oasis-open.org/xacml/3.0/xacml-3.0-core-spec-os-en.html)
<br/>
[![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

<!-- prettier-ignore -->

このチュートリアルでは、直接または **Keyrock** GUI を使用した、**Authzforce**
での、レベル3の高度な認可ルールの管理について説明します。
単純な動詞リソース・ベースのパーミッション
(simple verb-resource based permissions) は、既存のロールに追加された XACML
および新しい XACML パーミッションを使用するように修正されています。
更新されたルールセットは自動的に **Authzforce PDP** にアップロード
されるため、**PEP proxy** などのポリシー実行ポイント (PEP) は最新のルールセット
を適用できます。

チュートリアルは、**Keyrock** GUI を使用してインタラクションの例を示し、
同様に、**Keyrock** と **Authzforce** の REST API にアクセスするために使用する、
[cUrl](https://ec.haxx.se/) コマンドも説明します。
[Postman documentation](https://fiware.github.io/tutorials.Administrating-XACML)
も利用できます。

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/23b7045a5b52a54a2666)

## コンテンツ

<details>
<summary><strong>詳細</strong></summary>

-   [XACML ルールを管理](#administrating-xacml-rules)
    -   [XACML とは](#what-is-xacml)
    -   [PAP - Policy Administration Point (ポリシー管理ポイント)](#pap---policy-administration-point)
        -   [Authzforce PAP](#authzforce-pap)
        -   [Keyrock PAP](#keyrock-pap)
    -   [PEP - Policy Execution Point (ポリシー実行ポイント)](#pep---policy-execution-point)
-   [前提条件](#prerequisites)
    -   [Docker](#docker)
    -   [Cygwin](#cygwin)
-   [アーキテクチャ](#architecture)
-   [起動](#start-up)
    -   [登場人物 (Dramatis Personae)](#dramatis-personae)
-   [XACML 管理](#xacml-administration)
    -   [Authzforce PAP](#authzforce-pap-1)
        -   [新しいドメインを作成](#creating-a-new-domain)
        -   [初期ポリシーセットを作成](#creating-an-initial-policyset)
        -   [初期ポリシーセットをアクティブ化](#activating-the-initial-policyset)
        -   [ポリシーセットを更新](#updating-a-policyset)
        -   [更新されたポリシーセットをアクティブ化](#activating-an-updated-policyset)
    -   [Keyrock PAP](#keyrock-pap-1)
        -   [定義済みのロールと権限](#predefined-roles-and-permissions)
        -   [パスワードでトークンを作成](#create-token-with-password)
        -   [動詞リソースのパーミッションを読み込む](#read-a-verb-resource-permission)
        -   [XACML ルール権限を読み込む](#read-a-xacml-rule-permission)
        -   [リソースへのアクセスを拒否](#deny-access-to-a-resource)
        -   [XACML 権限を更新](#update-an-xacml-permission)
        -   [更新したポリシーセットを Authzforce に渡す](#passing-the-updated-policy-set-to-authzforce)
        -   [リソースへのアクセスを許可](#permit-access-to-a-resource)
    -   [PEP のチュートリアル  - 高度な認可の拡張](#tutorial-pep---extending-advanced-authorization)
        -   [高度な認可の拡張 - サンプル・コード](#extending-advanced-authorization---sample-code)
        -   [高度な認可の拡張 - 例の実行](#extending-advanced-authorization---running-the-example)
-   [次のステップ](#next-steps)

</details>

<a name="administrating-xacml-rules"></a>

# XACML ルールを管理

> **12.3 Central Terminal Area**
>
> -   Red or Yellow Zone
>     -   No private vehicle shall stop, wait, or park in the red or yellow
>         zone.
> -   White Zone
>     -   No vehicle shall stop, wait, or park in the white zone unless actively
>         engaged in the immediate loading or unloading of passengers and/or
>         baggage.
>
> — Los Angeles International Airport Rules and Regulations, Section 12 -
> Landside Motor Vehicle Operations

ビジネス・ルールは時間とともに変化し、それに応じてアクセス制御を
修正できるようにする必要があります。
[以前のチュートリアル](https://github.com/FIWARE/tutorials.XACML-Access-Rules)
には、**Authzforce** にロードされた、静的 XACML `<PolicySet>` が含まれて
いました。このコンポーネントは、すべてのポリシー決定がその場で計算され、
新しいルールが新しい状況下で適用されることができる高度な認可 (レベル3)
アクセス制御を提供します。
[Authzforce](https://authzforce-ce-fiware.readthedocs.io/) Policy Decision Point
(PDP:ポリシー決定ポイント) の詳細は
[以前のチュートリアル](https://github.com/FIWARE/tutorials.XACML-Access-Rules)
で説明しましたが、**Authzforce** PDP は XACML 標準に従ってルールを解釈し、
十分な情報を提供できる場合はアクセスを判断する手段を提供します。

最大限の柔軟性を得るために、必要に応じて新しいアクセス制御 XACML`<PolicySet>` を
ロード、更新、およびアクティブ化することが可能でなければなりません。
そうするために、この **Authzforce** は単純な REST Policy Adminstration Point
(PAP:ポリシー管理ポイント) を提供します、代わりのロール・ベースの
PAP は **Keyrock** 内で利用可能です。

<a name="what-is-xacml"></a>

## XACML とは

eXtensible Access Control Markup Language (XACML) は、ベンダーに依存しない
宣言型アクセス制御ポリシー言語です。これは、一般的なアクセス制御の用語と
相互運用性を促進するために作成されました。ポリシー実行ポイント
(PEP : Policy Execution Point) やポリシー決定ポイント (PDP : Policy
Decision Point) などの要素のアーキテクチャの命名ルールは、XACML
仕様に基づいています。

XACML ポリシーは、`<PolicySet>`, `<Policy>` と `<Rule>` の3つのレベルの
階層に分けられます。`<PolicySet>` は、それぞれが一つ以上の `<Rule>` 要素を
含む `<Policy>` 要素の集合です。

`<Policy>` 内の各 `<Rule>` は、それがリソースへのアクセスを許可すべきか
どうかに関して評価されます。総合的な `<Policy>` 結果は、順番に処理された
すべての `<Rule>` 要素の総合的な結果によって定義されます。そして、別々の
`<Policy>` 結果は、衝突の場合にどちらの `<Policy>` が勝つかを定義する
組み合わせアルゴリズムを使用してお互いに対して評価されます。

さらなる情報は、
[XACML 標準](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=xacml)
内にあり、
[追加のリソース](https://www.webfarmr.eu/xacml-tutorial-axiomatics/)
は Web 上にあります。

<a name="pap---policy-administration-point"></a>

## PAP - Policy Administration Point (ポリシー管理ポイント)

チュートリアルの前半では、**Authzforce** PAP を使用して単純な2つのルール
`<PolicySet>` を管理します。その後、**Keyrock** GUI を使用して既存の
チュートリアルのアプリケーション内の XACML ルールを個々の XACML `<Rule>` レベルで
管理します。**PEP-Proxy** 内のポリシー決定のリクエスト・コードも、複雑な XACML
ルールの実施を可能にするためにカスタマイズする必要があるかもしれません。
<a name="authzforce-pap"></a>

### Authzforce PAP

**Authzforce** PAP 内では、すべての CRUD アクションは `<PolicySet>` レベルで
発生します。そのため、サービスにアップロードする前に、完全で有効な XACML ファイル
を作成する必要があります。XACML をアップロードする前に `<PolicySet>` の有効性を
保証するために利用できる GUI はありません。

<a name="keyrock-pap"></a>

### Keyrock PAP

**Keyrock**は、利用可能なロールとパーミッションに基づいて有効な XACML ファイルを
作成し、これを **Authzforce** に渡すことができます。実際、**Keyrock** は、
**Authzforce** と組み合わせるたびに、これを実行しています。これは、**Authzforce**
によって判断される前に、すべての独自の基本認可 (レベル2) パーミッションを高度な
認可 (レベル3) パーミッションに変換する必要があるためです。

**Keyrock** 内では、各ロールは XACML `<Policy>` に対応し、そのロール内の
各パーミッションは XACML `<Rule>` に対応します。`<Rule>` ごとに XACML を
アップロードおよび修正するための GUI があり、すべての CRUD アクションは
`<Rule>` レベルで発生します。

`<Rule>` を作成する際には注意が必要ですが、XACML の管理を単純化し、
**Authzforce** に対して有効な `<PolicySet>` を作成するために
**Keyrock** を使用できます。

<a name="pep---policy-execution-point"></a>

## PEP - Policy Execution Point (ポリシー実行ポイント)

高度な認可 (レベル3) を使用する場合、ポリシー実行ポイント (PEP) は認可リクエストを
**Authzforce** 内の関連ドメイン・エンドポイントに送信し、**Authzforce** が判断を
下すために必要なすべての情報を提供します。インタラクションの詳細は
[以前のチュートリアル](https://github.com/FIWARE/tutorials.XACML-Access-Rules)にあります。

各リクエストを **Authzforce** に提供するための完全なコードは、チュートリアルの
[Gitリポジトリ](https://github.com/FIWARE/tutorials.Step-by-Step/blob/master/context-provider/lib/azf.js)
にあります。

明らかに "必要なすべての情報" の定義は時間とともに変化するかもしれません。
したがって、アプリケーションは十分な情報が渡されることを確実にするために
送られるリクエストを修正することができるほど十分に柔軟でなければなりません。

<a name="prerequisites"></a>

# 前提条件

<a name="docker"></a>

## Docker

物事を単純にするために、両方のコンポーネントが [Docker](https://www.docker.com)
を使用して実行されます。**Docker** は、さまざまコンポーネントをそれぞれの環境に
分離することを可能にするコンテナ・テクノロジです。

-   Docker Windows にインストールするには
    、[こちら](https://docs.docker.com/docker-for-windows/)の手順に従ってくださ
    い
-   Docker Mac にインストールするには
    、[こちら](https://docs.docker.com/docker-for-mac/)の手順に従ってください
-   Docker Linux にインストールするには
    、[こちら](https://docs.docker.com/install/)の手順に従ってください

**Docker Compose** は、マルチコンテナ Docker アプリケーションを定義して実行する
ためのツールです。
[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Identity-Management/master/docker-compose.yml)
ファイルは、アプリケーションのために必要なサービスを構成するために使用します。つ
まり、すべてのコンテナ・サービスは 1 つのコマンドで呼び出すことができます
。Docker Compose は、デフォルトで Docker for Windows と Docker for Mac の一部と
してインストールされますが、Linux ユーザは
[ここ](https://docs.docker.com/compose/install/)に記載されている手順に従う必要
があります。

<a name="cygwin"></a>

## Cygwin

シンプルな bash スクリプトを使用してサービスを開始します。Windows ユーザは
[cygwin](http://www.cygwin.com/) をダウンロードして、Windows 上の Linux
ディストリビューションと同様のコマンドライン機能を提供する必要があります。

<a name="architecture"></a>

# アーキテクチャ

このアプリケーションは、
[以前のチュートリアル](https://github.com/FIWARE/tutorials.Securing-Access/)
で作成した既存の在庫管理 およびセンサ・ベースのアプリケーションにレベル3の
高度な認可のセキュリティを追加し、
[PEP Proxy](https://github.com/FIWARE/tutorials.PEP-Proxy/) の背後にある
Context Broker へのアクセスを保護します。
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/),
[IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/),
[Keyrock](https://fiware-idm.readthedocs.io/en/latest/) Identity Manager,
[Wilma](https://fiware-pep-proxy.readthedocs.io/en/latest/) PEP Proxy,
[Authzforce](https://authzforce-ce-fiware.readthedocs.io) XACML Server
の5つの FIWARE コンポーネントを利用します。すべてのアクセス制御の決定は、
以前にアップロードされたポリシー・ドメインからルールセットを読み取る
**Authzforce** に委任されます。

Orion Context Brokerと IoT Agent はどちらも、オープンソースの
[MongoDB](https://www.mongodb.com/) テクノロジを使用して、保持している情報を
永続化します。また、
[以前のチュートリアル](https://github.com/FIWARE/tutorials.IoT-Sensors/)
で作成したダミー IoT デバイスも使用します。**Keyrock** は、独自に
[MySQL](https://www.mysql.com/) データベースを使用しています。

したがって、アーキテクチャ全体は次の要素から構成されます :

-   FIWARE
    [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) は
    、[NGSI](https://fiware.github.io/specifications/ngsiv2/latest/) を使用
    してリクエストを受信します
-   FIWARE
    [IoT Agent for Ultralight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/)
    は、
    [Ultralight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    フォーマットのダミー IoT デバイスからノース・バウンドの測定値を受信し、
    Context Broker がコンテキスト・エンティティの状態を変更するための
    [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
    リクエストに変換します
-   FIWARE [Keyrock](https://fiware-idm.readthedocs.io/en/latest/) は、以下を含
    んだ、補完的な ID 管理システムを提供します :
    -   アプリケーションとユーザのための OAuth2 認証システム
    -   ID 管理のための Web サイトのグラフィカル・フロントエンド
    -   HTTP リクエストによる ID 管理用の同等の REST API
-   FIWARE
    [Authzforce](https://authzforce-ce-fiware.readthedocs.io/)
    **Orion** やチュートリアル・アプリケーションなどのリソースへのアクセスを
    保護する解釈可能な Policy Decision Point (PDP) を提供する XACML Server です
-   FIWARE
    [Wilma](https://fiware-pep-proxy.rtfd.io/)
    **Orion** マイクロサービスへのアクセスを保護する PEP Proxy プロキシです。
    認可決定の受渡しを **Authzforce** PDP に委任します
-   [MongoDB](https://www.mongodb.com/) データベース :
    -   **Orion Context Broker** が、データ・エンティティ、サブスクリプション、
        レジストレーションなどのコンテキスト・データ情報を保持するために使用しま
        す
    -   デバイスの URLs や Keys などのデバイス情報を保持するために **IoT Agent**
        によって使用されます
-   [MySQL](https://www.mysql.com/) データベース :
    -   ユーザ ID、アプリケーション、ロール、および権限を保持するために使用され
        ます
-   **在庫管理フロントエンド**には、次のことを行います :
    -   店舗情報を表示します
    -   各店舗でどの商品を購入できるかを示します
    -   ユーザが製品を"購入"して在庫数を減らすことができます
    -   許可されたユーザを制限されたエリアに入れることができます。認可の決定を
        **Authzforce** PDP に委任します
-   HTTP を介して実行されている
    [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    プロトコルを使用す
    る[ダミー IoT デバイス](https://github.com/FIWARE/tutorials.IoT-Sensors)のセ
    ットとして機能する Web サーバ。特定のリソースへのアクセスが制限されています

要素間のやり取りはすべて HTTP リクエストによって開始されるため、
エンティティをコンテナ化して公開ポートから実行することができます。

![](https://fiware.github.io/tutorials.Administrating-XACML/img/architecture.png)

YAMLファイルに記述されている全てのコンテナの設定値は
以前のチュートリアルで説明されています。

<a name="start-up"></a>

# 起動

インストールを開始するには、次の手順に従います :

```console
git clone https://github.com/FIWARE/tutorials.Administrating-XACML.git
cd tutorials.Administrating-XACML

./services create
```

> **注:** Docker イメージの最初の作成には最大 3 分かかります

[services](https://github.com/FIWARE/tutorials.XACML-Access-Rules/blob/master/services)
Bash スクリプトを実行することによって、コマンドラインからすべてのサービスを初期
化することができます :

```console
./services start
```

> :information_source: **注:** クリーンアップをやり直したい場合は、次のコマンド
> を使用して再起動することができます :
>
> ```console
> ./services stop
> ```

<a name="dramatis-personae"></a>

### 登場人物 (Dramatis Personae)

次の `test.com` のメンバは、合法的にアプリケーション内にアカウントを持っています

-   Alice, **Keyrock** アプリケーションの管理者です
-   Bob, スーパー・マーケット・チェーンの地域マネージャで、数人のマネージャがい
    ます :
    -   Manager1 (マネージャ 1)
    -   Manager2 (マネージャ 2)
-   Charlie, スーパー・マーケット・チェーンのセキュリティ責任者。彼の下に数人の
    警備員がいます :
    -   Detective1 (警備員 1)
    -   Detective2 (警備員 2)

次の`example.com` のメンバはアカウントにサインアップしましたが、アクセスを許可す
る理由はありません

-   Eve - 盗聴者のイブ
-   Mallory - 悪意のある攻撃者のマロリー
-   Rob - 強盗のロブ

<details>
  <summary>
   詳細<b>(クリックして拡大)</b>
  </summary>

| 名前       | E メール                  | パスワード |
| ---------- | ------------------------- | ---------- |
| alice      | alice-the-admin@test.com  | `test`     |
| bob        | bob-the-manager@test.com  | `test`     |
| charlie    | charlie-security@test.com | `test`     |
| manager1   | manager1@test.com         | `test`     |
| manager2   | manager2@test.com         | `test`     |
| detective1 | detective1@test.com       | `test`     |
| detective2 | detective2@test.com       | `test`     |

| 名前    | E メール            | パスワード |
| ------- | ------------------- | ---------- |
| eve     | eve@example.com     | `test`     |
| mallory | mallory@example.com | `test`     |
| rob     | rob@example.com     | `test`     |

</details>

<a name="xacml-administration"></a>

# XACML 管理

アクセス制御ポリシーを適用するには、次のことができるようにする必要があります :

1. 一貫した `<PolicySet>` を作成します
2. 必要なデータを提供する ポリシー実行ポイント (PEP) を供給します

お分かりのように、**Keyrock** は最初のポイントに役立ち、**PEP Proxy** 内の
カスタム・コードは2番目のポイントに役立ちます。**Authzforce** 自体は UI を
提供せず、XACML ポリシーの生成と管理には関係ありません。受け取った各
`<PolicySet>`
はすでに別のコンポーネントによって生成されていると想定しています。

本格的な XACML エディタが利用可能ですが、**Keyrock** 内の限られたエディタで
通常、ほとんどのアクセス制御シナリオおいて十分です。

<a name="authzforce-pap-1"></a>

## Authzforce PAP

**Authzforce** はポリシー管理ポイント (PAP) として機能できます。
つまり、**Authzforce** への直接の API 呼び出しを使用して `PolicySet`
を作成および修正できます。

ただし、`<PolicySet>` を作成または修正するための GUI や生成ツールはありません。
すべての CRUD アクションは `<PolicySet>` レベルで発生します。

それほど重要ではないアプリケーション用の完全な XACML `<PolicySet>`
は非常に冗長です。話を簡単にするために、チュートリアルのこの部分では、
空港駐車場の施行用に設計されたまったく新しいアクセス・ルールのセットを
作成します。**Authzforce** は、暗黙的にマルチテナントです。単一の XACML
サーバを使用して複数のアプリケーションに対するアクセス制御ポリシーを
管理できます。各アプリケーションのセキュリティ・ポリシーは、独自の
`<PolicySets>`にアクセスできる個別のドメインに保持されます。
したがって、空港アプリケーションを管理しても、チュートリアルの
スーパーマーケット・アプリケーションの既存のルールが
妨げられることはありません。

<a name="creating-a-new-domain"></a>

### 新しいドメインを作成

**Authzforce** で新しいドメインを作成するには、`<domainProperties>` 要素内に
一意の  `external-id` を含む `/authzforce-ce/domains` エンドポイントに
POST リクエストを送信します。

#### :one: リクエスト

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<domainProperties xmlns="http://authzforce.github.io/rest-api-model/xmlns/authz/5" externalId="airplane"/>'
```

#### レスポンス

レスポンスには、**Authzforce**で内部的に使われている `domain-id` を保持する
`<n2:link>` 要素 に `href`  が含まれています。

空の `PolicySet` が新しいドメイン用に作成されます。デフォルトではすべての
アクセスが許可されます。

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns4:link xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0" rel="item" href="Sv-RRw9vEem6UQJCrBIBDA" title="Sv-RRw9vEem6UQJCrBIBDA"/>
```

新しい`domain-id` (この場合、`Sv-RRw9vEem6UQJCrBIBDA` ) は、以降のすべての
リクエストで使用されます。

#### :two: リクエスト

**Authzforce** に決定をリクエストするには、`domains/{domain-id}/pdp`
エンドポイントに POST リクエストを出します。この場合、ユーザは `white`
ゾーン内の `loading` へのアクセスをリクエストします。

あなた自身の `{domain-id}` を使うために下記のリクエストを修正することを
忘れないでください :

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
      </Attribute>
      <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">white</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

#### レスポンス

リクエストに対するレスポンスには、リソースへのアクセスを `Permit`
(許可) または `Deny` (拒否) するための `Deny` 要素が含まれます。
この時点ですべてのリクエストは `Permit` になります。

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Response xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0">
    <Result>
        <Decision>Permit</Decision>
    </Result>
</Response>
```

<a name="creating-an-initial-policyset"></a>

### 初期ポリシーセットを作成

**Authzforce**で特定のドメイン情報のための `PolicySet` 作成するには、
アップロードする XACML ルールの全セットを含む
`/authzforce-ce/domains/{{domain-id}}/pap/policies` エンドポイントに
POST リクエストを行います。

この初期ポリシーでは、以下のルールが強制されます。

-   **white** ゾーンは、乗客の即時積み降ろしのためです
-   **red** ゾーンで止まることはありません

#### :three: リクエスト

XACML `<PolicySet>` の全データは非常に冗長であり、
以下のリクエストから省略されています :

<details>
  <summary>

あなた自身の `{domain-id}` を使うために下記のリクエストを
修正することを忘れないでください :

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/policies \
  -H 'Content-Type: application/xml' \
  -d '<PolicySet>...etc</PolicySet>'
```

**(クリックすると拡大します)**

  </summary>

あなた自身の `{domain-id}` を使うために下記のリクエストを
修正することを忘れないでください :

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/policies \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<PolicySet xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" PolicySetId="f8194af5-8a07-486a-9581-c1f05d05483c" Version="1" PolicyCombiningAlgId="urn:oasis:names:tc:xacml:3.0:policy-combining-algorithm:deny-unless-permit">
   <Description>Policy Set for Airplane!</Description>
   <Target />
   <Policy PolicyId="airplane" Version="1.0" RuleCombiningAlgId="urn:oasis:names:tc:xacml:3.0:rule-combining-algorithm:deny-unless-permit">
      <Description>Vehicle Roles from the Male announcer in the movie Airplane!</Description>
      <Target>
         <AnyOf>
            <AllOf>
               <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                  <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
                  <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
               </Match>
            </AllOf>
         </AnyOf>
      </Target>
      <Rule RuleId="white-zone" Effect="Permit">
         <Description>The white zone is for immediate loading and unloading of passengers only</Description>
         <Target>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">white</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action" AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
         </Target>
      </Rule>
      <Rule RuleId="red-zone" Effect="Deny">
         <Description>There is no stopping in the red zone</Description>
         <Target>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">red</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">stopping</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action" AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
         </Target>
      </Rule>
   </Policy>
</PolicySet>
'
```

</details>

#### レスポンス

レスポンスには、**Authzforce** 内に保持されているポリシーの内部
id と利用可能な  PolicySet バージョンに関するバージョン情報が
含まれています。新しい `PolicySet` のルールは、`PolicySet`
が有効になるまで適用されません。

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns4:link xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0" rel="item" href="f8194af5-8a07-486a-9581-c1f05d05483c/1" title="Policy 'f8194af5-8a07-486a-9581-c1f05d05483c' v1"/>
```

<a name="activating-the-initial-policyset"></a>

### 初期ポリシーセットをアクティブ化

`PolicySet` をアクティブにするには、`<rootPolicyRefExpresion>`
属性内で更新する `policy-id` を含む
`/authzforce-ce/domains/{domain-id}/pap/pdp.properties`
エンドポイントに PUT リクエストを行います。

#### :four: リクエスト

あなた自身の `{domain-id}` を使うために下記のリクエストを
修正することを忘れないでください :

```console
curl -X PUT \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/pdp.properties \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
  <pdpPropertiesUpdate xmlns="http://authzforce.github.io/rest-api-model/xmlns/authz/5">
    <rootPolicyRefExpression>f8194af5-8a07-486a-9581-c1f05d05483c</rootPolicyRefExpression>
  </pdpPropertiesUpdate>'
```

#### レスポンス

レスポンスは、適用された `PolicySet` についての情報を返します。

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns3:pdpProperties xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0" lastModifiedTime="2019-01-03T15:54:45.341Z">
    <ns3:feature type="urn:ow2:authzforce:feature-type:pdp:core" enabled="false">urn:ow2:authzforce:feature:pdp:core:strict-attribute-issuer-match</ns3:feature>
    <ns3:feature type="urn:ow2:authzforce:feature-type:pdp:core" enabled="false">urn:ow2:authzforce:feature:pdp:core:xpath-eval</ns3:feature>
... ETC
    <ns3:rootPolicyRefExpression>f8194af5-8a07-486a-9581-c1f05d05483c</ns3:rootPolicyRefExpression>
    <ns3:applicablePolicies>
        <ns3:rootPolicyRef Version="1">f8194af5-8a07-486a-9581-c1f05d05483c</ns3:rootPolicyRef>
    </ns3:applicablePolicies>
</ns3:pdpProperties>
```

#### :five: リクエスト

この時点で、`white` ゾーン内で `loading` へのアクセスを
リクエストすると `Permit` を戻します。

あなた自身の `{domain-id}` を使うために下記のリクエストを
修正することを忘れないでください :

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
      </Attribute>
      <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">white</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

#### レスポンス

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Response xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0">
    <Result>
        <Decision>Permit</Decision>
    </Result>
</Response>
```

#### :six: リクエスト

この時点で、`red` ゾーン内で `loading` へのアクセスを
リクエストする `Deny` を戻します。

あなた自身の `{domain-id}` を使うために下記のリクエストを
修正することを忘れないでください :

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
      </Attribute>
      <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">red</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

#### レスポンス

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Response xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0">
    <Result>
        <Decision>Deny</Decision>
    </Result>
</Response>
```

<a name="updating-a-policyset"></a>

### ポリシーセットを更新

**Authzforce** で特定のドメイン情報の `PolicySet` を更新するには、
アップロードする XACML ルールのフルセットを含む
`/authzforce-ce/domains/{{domain-id}}/pap/policies` エンドポイントに
POST リクエストを行います。`Version` は一意である必要があります。

更新されたポリシーについては、以前のルールが逆になります。

-   **red** ゾーンは、乗客の即時積み降ろしのためです
-   **white** ゾーンで止まることはありません

#### :seven: リクエスト

XACML `<PolicySet>` の全データは非常に冗長であり、
以下のリクエストから省略されています。

<details>
  <summary>

あなた自身の `{domain-id}` を使うために下記のリクエストを
修正することを忘れないでください :

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/policies \
  -H 'Content-Type: application/xml' \
  -d '<PolicySet>...etc</PolicySet>'
```

**(クリックすると拡大します)**

  </summary>

あなた自身の `{domain-id}` を使うために下記のリクエストを
修正することを忘れないでください :

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/policies \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<PolicySet xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" PolicySetId="f8194af5-8a07-486a-9581-c1f05d05483c" Version="2" PolicyCombiningAlgId="urn:oasis:names:tc:xacml:3.0:policy-combining-algorithm:deny-unless-permit">
   <Description>Policy Set for Airplane!</Description>
   <Target />
   <Policy PolicyId="airplane" Version="1.0" RuleCombiningAlgId="urn:oasis:names:tc:xacml:3.0:rule-combining-algorithm:deny-unless-permit">
      <Description>Vehicle Roles from the Female announcer in the movie Airplane!</Description>
      <Target>
         <AnyOf>
            <AllOf>
               <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                  <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
                  <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
               </Match>
            </AllOf>
         </AnyOf>
      </Target>
      <Rule RuleId="red-zone" Effect="Permit">
         <Description>The red zone is for immediate loading and unloading of passengers only</Description>
         <Target>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">red</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action" AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
         </Target>
      </Rule>
      <Rule RuleId="white-zone" Effect="Deny">
         <Description>There is no stopping in the white zone</Description>
         <Target>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">white</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
            <AnyOf>
               <AllOf>
                  <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                     <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">stopping</AttributeValue>
                     <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action" AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                  </Match>
               </AllOf>
            </AnyOf>
         </Target>
      </Rule>
   </Policy>
</PolicySet>
'
```

</details>

#### レスポンス

レスポンスには、利用可能な `PolicySet` バージョンに関するバージョン
情報が含まれています。新しい `PolicySet` のルールは `PolicySet`
が有効になるまで適用されません。

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns4:link xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0" rel="item" href="f8194af5-8a07-486a-9581-c1f05d05483c/2" title="Policy 'f8194af5-8a07-486a-9581-c1f05d05483c' v2"/>
```

<a name="activating-an-updated-policyset"></a>

### 更新されたポリシーセットをアクティブ化

アクティブな `PolicySet` を更新するには、`<rootPolicyRefExpresion>`
属性内で更新する `policy-id` を含む
`/authzforce-ce/domains/{domain-id}/pap/pdp.properties`
エンドポイントへ別の PUTリクエストを行います。最新のアップロード・
バージョンを適用するようにルールセットが更新されます。

#### :eight: リクエスト

あなた自身の `{domain-id}` を使うために下記のリクエストを
修正することを忘れないでください :

```console
curl -X PUT \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pap/pdp.properties \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
  <pdpPropertiesUpdate xmlns="http://authzforce.github.io/rest-api-model/xmlns/authz/5">
    <rootPolicyRefExpression>f8194af5-8a07-486a-9581-c1f05d05483c</rootPolicyRefExpression>
  </pdpPropertiesUpdate>'
```

#### レスポンス

レスポンスは適用された `PolicySet` についての情報を返します。

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns3:pdpProperties xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0" lastModifiedTime="2019-01-03T15:58:29.351Z">
    <ns3:feature type="urn:ow2:authzforce:feature-type:pdp:core" enabled="false">urn:ow2:authzforce:feature:pdp:core:strict-attribute-issuer-match</ns3:feature>
    <ns3:feature type="urn:ow2:authzforce:feature-type:pdp:core" enabled="false">urn:ow2:authzforce:feature:pdp:core:xpath-eval</ns3:feature>
... ETC
    <ns3:rootPolicyRefExpression>f8194af5-8a07-486a-9581-c1f05d05483c</ns3:rootPolicyRefExpression>
    <ns3:applicablePolicies>
        <ns3:rootPolicyRef Version="2">f8194af5-8a07-486a-9581-c1f05d05483c</ns3:rootPolicyRef>
    </ns3:applicablePolicies>
</ns3:pdpProperties>
```

#### :nine: リクエスト

新しいポリシーが有効になっているので、この時点で、`white` ゾーンで
`loading` へのアクセスをリクエストすると `Deny` が返されます。

あなた自身の`{domain-id}` を使うために下記のリクエストを
修正することを忘れないでください :

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
      </Attribute>
      <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">white</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

#### レスポンス

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Response xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0">
    <Result>
        <Decision>Deny</Decision>
    </Result>
</Response>
```

#### :one::zero: リクエスト

現在のポリシーの下にある `red` ゾーンで `loading` にアクセスを
リクエストすると、`Permit` を戻します。

あなた自身の `{domain-id}` を使うために下記のリクエストを
修正することを忘れないでください :

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/{domain-id}/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">airplane!</AttributeValue>
      </Attribute>
      <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">red</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
      <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
         <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">loading</AttributeValue>
      </Attribute>
   </Attributes>
   <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

#### レスポンス

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Response xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns2="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6" xmlns:ns3="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns4="http://www.w3.org/2005/Atom" xmlns:ns5="http://authzforce.github.io/core/xmlns/pdp/6.0">
    <Result>
        <Decision>Permit</Decision>
    </Result>
</Response>
```

<a name="keyrock-pap-1"></a>

## Keyrock PAP

**Keyrock** は、ロール・ベースのアクセス制御 ID 管理システムを提供しています。
通常、すべてのパーミッションは、与えられたロールの範囲内でのみユーザに
アクセス可能です。**Keyrock** 内にある基本認可 (レベル2) のアクセス制御
メカニズムを使用して、動詞リソース・ルール (Verb-Resource rules) を
[設定](https://github.com/FIWARE/tutorials.Roles-Permissions/) および
[強制](https://github.com/FIWARE/tutorials.Securing-Access/) する方法を
すでに説明しました。高度なパーミッションを定義するためのデータは、
**Keyrock** GUI または **Keyrock** REST リクエストを使用して管理する
こともできます。

**Keyrock** パーミッションは完全な `<PolicySet>` ではなく個々の XACML
の `<Rule>` 要素に対して機能します。`<PolicySet>` はすべてのロールと
パーミッションを組み合わせることによって生成されます。

<a name="predefined-roles-and-permissions"></a>

### 定義済みのロールと権限

以前のチュートリアルと同様の方法で、2つのロールが作成されました。
1つはストアの警備員 (store detectives)、もう1つは管理ユーザ用
(management users) です。スーパーマーケットのアプリケーションに
一連のパーミッションが設定されており、次の XACML ルールが適用されます。

#### セキュリティ・スタッフ (Security Staff)

-   いつでもドアのロックを解除可能
-   午前9時または午後5時以降にアラーム・ベルを鳴らすことが可能
-   いつでも **context broker** のデータにアクセス可能

#### 管理 (Mangement)

-   価格変更エリアへのアクセス
-   在庫数エリアへのアクセス
-   午前9時から午後5時の間にアラーム・ベルを鳴らすことが可能
-   午前9時から午後5時まで **context broker** のデータにアクセス可能

ご覧のとおり、いくつかの新しいルールには時間要素があり、
もはや単純な動詞リソース・ルール (simple Verb-Resource rules)
ではありません。

ほとんどのルールでは、ポリシー実行ポイント (つまり、**Authzforce** への
リクエストと結果の分析) はチュートリアルの
[コード](https://github.com/FIWARE/tutorials.Step-by-Step/blob/master/context-provider/lib/azf.js)
内にあります。

**ストア** および **IoT デバイス**の **context broker** のデータは、
PEP Proxy の背後で保護されています。これは、セキュリティ・スタッフだけが
コア・タイム外にシステムにアクセスできることを意味します。

> **注** **Keyrock** 内の、高度な認可ルール (レベル3) を使用して
> 保護されているリソースは4つだけです
>
> -   ベルを鳴らすコマンドの送信
> -   価格変更エリアへのアクセス
> -   注文在庫エリアへのアクセス
> -   **context broker** への PEP Proxy アクセス
>
> -   対照的に、1つのリソースは単純な動詞リソースのパーミッション
>     (レベル2) として残されています
>
> -   ドアのロック解除コマンドを送信

<a name="create-token-with-password"></a>

### パスワードでトークンを作成

#### :one::one: リクエスト

ユーザ名とパスワードを入力してアプリケーションに入ります。デフォルトの
スーパーユーザは、`alice-the-admin@test.com` と `test`の値を持ちます。

```console
curl -iX POST \
  http://localhost:3005/v1/auth/tokens \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "name": "alice-the-admin@test.com",
  "password": "test"
}'
```

#### レスポンス

**Keyrock** からのレスポンス・ヘッダは、誰がアプリケーションにログオン
したかを識別する `X-Subject-token` を返します。このトークンは、
後続のすべてのリクエストでアクセスするために必要です。

```
HTTP/1.1 201 Created
X-Subject-Token: d848eb12-889f-433b-9811-6a4fbf0b86ca
Content-Type: application/json; charset=utf-8
Content-Length: 138
ETag: W/"8a-TVwlWNKBsa7cskJw55uE/wZl6L8"
Date: Mon, 30 Jul 2018 12:07:54 GMT
Connection: keep-alive
```

**Keyrock** からのレスポンスのボディは、**Authzforce** が使用中であり、
XACML ルールを使用してアクセス・ポリシーを定義できることを示しています。

```json
{
    "token": {
        "methods": ["password"],
        "expires_at": "2019-01-03T17:04:43.358Z"
    },
    "idm_authorization_config": {
        "level": "advanced",
        "authzforce": true
    }
}
```

<a name="read-a-verb-resource-permission"></a>

### 動詞リソースのパーミッションを読み込む

ちなみに、単純な動詞リソースのパーミッションでは、
`/applications/{{app-id}}/permissions/{permission-id}}` エンドポイントは
その id の下にリストされたパーミッションを返します。`X-Auth-token` は、
ヘッダで提供されなければなりません。

#### :one::two: リクエスト

```console
curl -X GET \
  http://localhost:3005/v1/applications/tutorial-dckr-site-0000-xpresswebapp/permissions/entrance-open-0000-0000-000000000000 \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa'
```

#### レスポンス

レスポンスはリクエストされたパーミッションの詳細を返します。
動詞リソースのパーミッションの場合、`xml` 要素は `null` です。
このパーミッションは、メイン・エントランスのロックを解除するために、
ユーザーが `/door/unlock` エンドポイントに POST リクエストを行うことを
許可されていることを示します。

```json
{
    "permission": {
        "id": "entrance-open-0000-0000-000000000000",
        "name": "Unlock",
        "description": "Unlock main entrance",
        "is_internal": false,
        "action": "POST",
        "resource": "/door/unlock",
        "xml": null,
        "oauth_client_id": "tutorial-dckr-site-0000-xpresswebapp"
    }
}
```

<a name="read-a-xacml-rule-permission"></a>

### XACML ルール権限を読み込む

XACML ルールのパーミッションにも同じ方法でアクセスでき、
`/applications/{{app-id}}/permissions/{permission-id}}` エンドポイントは
その id 下にリストされたパーミッションを返します。`X-Auth-token` は、
ヘッダで提供されなければなりません。

#### :one::three: リクエスト

```console
curl -X GET \
  http://localhost:3005/v1/applications/tutorial-dckr-site-0000-xpresswebapp/permissions/alrmbell-ring-24hr-xaml-000000000000 \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa'
```

#### レスポンス

レスポンスはリクエストされたパーミッションの詳細を返します。XACML
ルールのパーミッションの場合、`xml` 要素は関連付けられた XACML `<Rule>`
の詳細を保持し、`action` および `resource` フィールドは `null` です。

`xml`属性内には既に `<Rule>` が1つあります。
次のように設定されています。

> *警備員* は、午前9時**前**、または午後5時**以降**に、
> アラーム・ベルを鳴らすことができます

<details>
  <summary>
   完全な <code>&lt;Rule&gt;</code> を見るには <b>(クリックして拡大)</b>
  </summary>

```xml
<Rule RuleId="alrmbell-ring-24hr-hours-000000000000" Effect="Permit">
    <Description>Ring Alarm Bell (Outside Core Hours)</Description>
    <Target>
        <AnyOf>
            <AllOf>
                <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                    <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">/bell/ring</AttributeValue>
                    <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                </Match>
            </AllOf>
        </AnyOf>
        <AnyOf>
            <AllOf>
                <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                    <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">POST</AttributeValue>
                    <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action" AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                </Match>
            </AllOf>
        </AnyOf>
        <AnyOf>
            <AllOf>
                <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                    <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">security-role-0000-0000-000000000000</AttributeValue>
                    <AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject" AttributeId="urn:oasis:names:tc:xacml:2.0:subject:role" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                </Match>
            </AllOf>
        </AnyOf>
    </Target>
    <Condition>
        <Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:not">
            <Apply FunctionId="urn:oasis:names:tc:xacml:2.0:function:time-in-range">
                <Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:time-one-and-only">
                    <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" AttributeId="urn:oasis:names:tc:xacml:1.0:environment:current-time" DataType="http://www.w3.org/2001/XMLSchema#time" MustBePresent="false" />
                </Apply>
                <Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:time-one-and-only">
                    <Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:time-bag">
                        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#time">08:00:00</AttributeValue>
                    </Apply>
                </Apply>
                <Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:time-one-and-only">
                    <Apply FunctionId="urn:oasis:names:tc:xacml:1.0:function:time-bag">
                        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#time">17:00:00</AttributeValue>
                    </Apply>
                </Apply>
            </Apply>
        </Apply>
    </Condition>
</Rule>
```

</details>

`<Rule>` の `<Target>` 要素は、以前のチュートリアルで見たのと同じように
動詞リソースのレベルでアクセスを定義します。`<Condition>` 要素は、
ルールの時間部分を保持し、現在のサーバ時間に基づいて **Authzforce**
によって評価されます。

```json
{
    "permission": {
        "id": "alrmbell-ring-24hr-xaml-000000000000",
        "name": "Ring Alarm Bell (Outside Core hours)",
        "description": "Ring Alarm Bell for Security",
        "is_internal": false,
        "action": null,
        "resource": null,
        "xml": "<Rule RuleId=\"alrmbell-ring-24hr-hours-000000000000\" Effect=\"Permit\">\n<Description>Ring Alarm Bell (Outside Core Hours)</Description>\n<Target>\n<AnyOf>\n<AllOf>\n<Match MatchId=\"urn:oasis:names:tc:xacml:1.0:function:string-equal\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#string\">/bell/ring</AttributeValue>\n<AttributeDesignator Category=\"urn:oasis:names:tc:xacml:3.0:attribute-category:resource\" AttributeId=\"urn:thales:xacml:2.0:resource:sub-resource-id\" DataType=\"http://www.w3.org/2001/XMLSchema#string\" MustBePresent=\"true\" />\n</Match>\n</AllOf>\n</AnyOf>\n<AnyOf>\n<AllOf>\n<Match MatchId=\"urn:oasis:names:tc:xacml:1.0:function:string-equal\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#string\">POST</AttributeValue>\n<AttributeDesignator Category=\"urn:oasis:names:tc:xacml:3.0:attribute-category:action\" AttributeId=\"urn:oasis:names:tc:xacml:1.0:action:action-id\" DataType=\"http://www.w3.org/2001/XMLSchema#string\" MustBePresent=\"true\" />\n</Match>\n</AllOf>\n</AnyOf>\n<AnyOf>\n<AllOf>\n<Match MatchId=\"urn:oasis:names:tc:xacml:1.0:function:string-equal\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#string\">security-role-0000-0000-000000000000</AttributeValue>\n<AttributeDesignator Category=\"urn:oasis:names:tc:xacml:1.0:subject-category:access-subject\" AttributeId=\"urn:oasis:names:tc:xacml:2.0:subject:role\" DataType=\"http://www.w3.org/2001/XMLSchema#string\" MustBePresent=\"true\" />\n</Match>\n</AllOf>\n</AnyOf>\n</Target>\n<Condition>\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:not\">\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:2.0:function:time-in-range\">\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:time-one-and-only\">\n<AttributeDesignator AttributeId=\"urn:oasis:names:tc:xacml:1.0:environment:current-time\" DataType=\"http://www.w3.org/2001/XMLSchema#time\" Category=\"urn:oasis:names:tc:xacml:3.0:attribute-category:environment\" MustBePresent=\"false\"></AttributeDesignator>\n</Apply>\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:time-one-and-only\">\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:time-bag\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#time\">08:00:00</AttributeValue>\n</Apply>\n</Apply>\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:time-one-and-only\">\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:time-bag\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#time\">17:00:00</AttributeValue>\n</Apply>\n</Apply>\n</Apply>\n</Apply>\n</Condition>\n</Rule>",
        "oauth_client_id": "tutorial-dckr-site-0000-xpresswebapp"
    }
}
```

**ヒント** 完全な  `<PolicySet>` の注釈付きバージョンは、
[チュートリアル自体](https://github.com/FIWARE/tutorials.Administrating-XACML/blob/master/authzforce/domains/gQqnLOnIEeiBFQJCrBIBDA/policies/ZjgxOTRhZjUtOGEwNy00ODZhLTk1ODEtYzFmMDVkMDU0ODNj/2.xml)
の中にあります。

<a name="deny-access-to-a-resource"></a>

### リソースへのアクセスを拒否

**Authzforce** に決定をリクエストするには、`domains/{domain-id}/pdp`
エンドポイントに POST リクエストを出します。この場合、`/ring/bell`
エンドポイントへの `POST` へのアクセスをリクエストしています。

#### :one::four: リクエスト

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/gQqnLOnIEeiBFQJCrBIBDA/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
  <Attributes Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject">
     <Attribute AttributeId="urn:oasis:names:tc:xacml:2.0:subject:role" IncludeInResult="false">
        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">security-role-0000-0000-000000000000</AttributeValue>
     </Attribute>
  </Attributes>
  <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
     <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">tutorial-dckr-site-0000-xpresswebapp</AttributeValue>
     </Attribute>
     <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">/bell/ring</AttributeValue>
     </Attribute>
  </Attributes>
  <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
     <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">POST</AttributeValue>
     </Attribute>
  </Attributes>
  <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

#### レスポンス

リクエストに対するレスポンスには、リソースへのアクセスを `Permit` または `Deny`
するための `<Decision>` 要素が含まれています。

サーバの時間が午前9時から午後5時の間の場合、アクセス・リクエストは拒否されます。

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns2:Response xmlns="http://www.w3.org/2005/Atom" xmlns:ns2="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns3="http://authzforce.github.io/core/xmlns/pdp/6.0" xmlns:ns4="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns5="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6">
    <ns2:Result>
        <ns2:Decision>Deny</ns2:Decision>
    </ns2:Result>
</ns2:Response>
```

<a name="update-an-xacml-permission"></a>

### XACML 権限を更新

ポリシーは次のように更新されます :

> **警備員**はいつでもベルを鳴らすことができる **Charlie** を除いて、
> 午前9時または午後5時以降にのみアラーム・ベルを鳴らすことができます。

これは、`alrmbell-ring-24hr-xaml-000000000000` パーミッションを、
2つのルールを適用するために修正する必要があることを意味します :

<details>
  <summary>
   新しい <code>&lt;Rule&gt;</code> を見るには <b>(クリックして拡大)</b>
  </summary>

```xml
<Rule RuleId="alrmbell-ring-only-000000000000" Effect="Permit">
    <Description>Allow Full Access to Charlie the Security Manager</Description>
    <Target>
        <AnyOf>
            <AllOf>
                <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                    <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">/bell/ring</AttributeValue>
                    <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource" AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                </Match>
            </AllOf>
        </AnyOf>
        <AnyOf>
            <AllOf>
                <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                    <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">POST</AttributeValue>
                    <AttributeDesignator Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action" AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                </Match>
            </AllOf>
        </AnyOf>
        <AnyOf>
            <AllOf>
                <Match MatchId="urn:oasis:names:tc:xacml:1.0:function:string-equal">
                    <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">charlie</AttributeValue>
                    <AttributeDesignator Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject" AttributeId="urn:oasis:names:tc:xacml:1.0:subject:subject-id" DataType="http://www.w3.org/2001/XMLSchema#string" MustBePresent="true" />
                </Match>
            </AllOf>
        </AnyOf>
    </Target>
</Rule>
```

</details>

これは、ルールを適切なテキスト・ボックスに貼り付けることによって
GUI で最も簡単に実行できますが、プログラムで実行することもできます。

**Keyrock** GUI にログインするには、ログイン・ページ
 `http://localhost:3005/` にユーザ名とパスワードを入力してください。

![](https://fiware.github.io/tutorials.Administrating-XACML/img/login.png)

正しいアプリケーションに移動して、manage roles (ロールの管理)
タブをクリックします。

![](https://fiware.github.io/tutorials.Administrating-XACML/img/manage-roles.png)

編集する permission (パーミッション) を選択してください。

![](https://fiware.github.io/tutorials.Administrating-XACML/img/edit-permission.png)

HTTP 動詞とリソースのルールは空白のままにする必要がありますが、適用可能な
XACML の `<Rule>` 要素は、次に示すように **Advanced XACML Rule**
テキスト・ボックス内にペーストする必要があります。

![](https://fiware.github.io/tutorials.Administrating-XACML/img/permission.png)

あるいは、パーミッションを管理する XACML ルールをプログラム的に修正するには、
`/applications/{{app-id}}/permissions/{permission-id}}` エンドポイントに
PATCH リクエストを出します。リクエストのボディは、3つの属性を含まなければ
なりません。`action` と `resource` は、両方とも `""` に設定しなければならず、
`xml` 属性は XACML テキストを保持するべきです。

#### :one::five: リクエスト

リクエストの完全なデータは非常に詳細であり、以下で省略されています :

<details>
  <summary>

```console
curl -X PATCH \
  http://localhost:3005/v1/applications/tutorial-dckr-site-0000-xpresswebapp/permissions/alrmbell-ring-24hr-xaml-000000000000 \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa' \
  -d '{
    "permission": {
        "action": "",
        "resource": "",
        "xml": "<Rule RuleId=\"alrmbell-ring-only-000000000000\" Effect=\"Permit\"> etc..."
    }
}
```

**(クリックすると拡大します)**

  </summary>

```console
curl -X PATCH \
  http://localhost:3005/v1/applications/tutorial-dckr-site-0000-xpresswebapp/permissions/alrmbell-ring-24hr-xaml-000000000000 \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa' \
  -d '{
    "permission": {
        "action": "",
        "resource": "",
        "xml": "<Rule RuleId=\"alrmbell-ring-only-000000000000\" Effect=\"Permit\">\n<Description>Allow Full Access to Charlie the Security Manager</Description>\n<Target>\n<AnyOf>\n<AllOf>\n<Match MatchId=\"urn:oasis:names:tc:xacml:1.0:function:string-equal\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#string\">/bell/ring</AttributeValue>\n<AttributeDesignator Category=\"urn:oasis:names:tc:xacml:3.0:attribute-category:resource\" AttributeId=\"urn:thales:xacml:2.0:resource:sub-resource-id\" DataType=\"http://www.w3.org/2001/XMLSchema#string\" MustBePresent=\"true\" />\n</Match>\n</AllOf>\n</AnyOf>\n<AnyOf>\n<AllOf>\n<Match MatchId=\"urn:oasis:names:tc:xacml:1.0:function:string-equal\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#string\">POST</AttributeValue>\n<AttributeDesignator Category=\"urn:oasis:names:tc:xacml:3.0:attribute-category:action\" AttributeId=\"urn:oasis:names:tc:xacml:1.0:action:action-id\" DataType=\"http://www.w3.org/2001/XMLSchema#string\" MustBePresent=\"true\" />\n</Match>\n</AllOf>\n</AnyOf>\n<AnyOf>\n<AllOf>\n<Match MatchId=\"urn:oasis:names:tc:xacml:1.0:function:string-equal\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#string\">charlie</AttributeValue>\n<AttributeDesignator Category=\"urn:oasis:names:tc:xacml:1.0:subject-category:access-subject\" AttributeId=\"urn:oasis:names:tc:xacml:1.0:subject:subject-id\" DataType=\"http://www.w3.org/2001/XMLSchema#string\" MustBePresent=\"true\" />\n</Match>\n</AllOf>\n</AnyOf>\n</Target>\n</Rule>\n<Rule RuleId=\"alrmbell-ring-24hr-hours-000000000000\" Effect=\"Permit\">\n<Description>Ring Alarm Bell (Outside Core Hours)</Description>\n<Target>\n<AnyOf>\n<AllOf>\n<Match MatchId=\"urn:oasis:names:tc:xacml:1.0:function:string-equal\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#string\">/bell/ring</AttributeValue>\n<AttributeDesignator Category=\"urn:oasis:names:tc:xacml:3.0:attribute-category:resource\" AttributeId=\"urn:thales:xacml:2.0:resource:sub-resource-id\" DataType=\"http://www.w3.org/2001/XMLSchema#string\" MustBePresent=\"true\" />\n</Match>\n</AllOf>\n</AnyOf>\n<AnyOf>\n<AllOf>\n<Match MatchId=\"urn:oasis:names:tc:xacml:1.0:function:string-equal\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#string\">POST</AttributeValue>\n<AttributeDesignator Category=\"urn:oasis:names:tc:xacml:3.0:attribute-category:action\" AttributeId=\"urn:oasis:names:tc:xacml:1.0:action:action-id\" DataType=\"http://www.w3.org/2001/XMLSchema#string\" MustBePresent=\"true\" />\n</Match>\n</AllOf>\n</AnyOf>\n<AnyOf>\n<AllOf>\n<Match MatchId=\"urn:oasis:names:tc:xacml:1.0:function:string-equal\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#string\">security-role-0000-0000-000000000000</AttributeValue>\n<AttributeDesignator Category=\"urn:oasis:names:tc:xacml:1.0:subject-category:access-subject\" AttributeId=\"urn:oasis:names:tc:xacml:2.0:subject:role\" DataType=\"http://www.w3.org/2001/XMLSchema#string\" MustBePresent=\"true\" />\n</Match>\n</AllOf>\n</AnyOf>\n</Target>\n<Condition>\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:not\">\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:2.0:function:time-in-range\">\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:time-one-and-only\">\n<AttributeDesignator AttributeId=\"urn:oasis:names:tc:xacml:1.0:environment:current-time\" DataType=\"http://www.w3.org/2001/XMLSchema#time\" Category=\"urn:oasis:names:tc:xacml:3.0:attribute-category:environment\" MustBePresent=\"false\"></AttributeDesignator>\n</Apply>\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:time-one-and-only\">\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:time-bag\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#time\">08:00:00</AttributeValue>\n</Apply>\n</Apply>\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:time-one-and-only\">\n<Apply FunctionId=\"urn:oasis:names:tc:xacml:1.0:function:time-bag\">\n<AttributeValue DataType=\"http://www.w3.org/2001/XMLSchema#time\">17:00:00</AttributeValue>\n</Apply>\n</Apply>\n</Apply>\n</Apply>\n</Condition>\n</Rule>"
    }
}'
```

</details>

#### レスポンス

レスポンスには、更新された属性の詳細が表示されます。

```json
{
    "values_updated": {
        "action": null,
        "resource": null,
        "xml": "<Rule RuleId=\"alrmbell-ring-only-000000000000\" Effect=\"Permit\"> etc..."
    }
}
```

<a name="passing-the-updated-policy-set-to-authzforce"></a>

### 更新したポリシーセットを Authzforce に渡す

**Keyrock** が **Authzforce** に接続されている場合、ロールとパーミッションの
関連付けが修正されるたびに、`<PolicySet>` が自動的に更新されます。
これを行う最も簡単な方法は単一の関連付けを削除して再作成することです。

まず DELETE リクエストを使って関連を削除します :

#### :one::six: リクエスト

```console
curl -X DELETE \
  http://localhost:3005/v1/applications/tutorial-dckr-site-0000-xpresswebapp/roles/security-role-0000-0000-000000000000/permissions/alrmbell-ring-24hr-xaml-000000000000 \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa'
```

次に示すように POST リクエストを使用してそれを再作成します :

#### :one::seven: リクエスト

```console
curl -X POST \
  http://localhost:3005/v1/applications/tutorial-dckr-site-0000-xpresswebapp/roles/security-role-0000-0000-000000000000/permissions/alrmbell-ring-24hr-xaml-000000000000 \
  -H 'Content-Type: application/json' \
  -H 'X-Auth-token: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa'
```

#### レスポンス

**Authzforce** が更新されている場合、追加の `authzforce` 属性が
レスポンス内に存在し、新しい `<PolicySet>` が配置されます。

```json
{
    "role_permission_assignments": {
        "role_id": "security-role-0000-0000-000000000000",
        "permission_id": "alrmbell-ring-24hr-xaml-000000000000"
    },
    "authzforce": {
        "create_policy": {
            "message": " Success creating policy.",
            "status": 200
        },
        "activate_policy": {
            "message": "Success",
            "status": 200
        }
    }
}
```

<a name="permit-access-to-a-resource"></a>

### リソースへのアクセスを許可

追加のフィールドをポリシー決定リクエストに追加して、`Permit` レスポンスの
可能性を高めることができます。

以下の例では`emailaddress` と `subject:subject-id` は、
`subject-category:access-subject`
カテゴリ内のリクエストのボディに追加されました。

新しいルールが設定されていれば、ユーザ `charlie`は、一日中いつでも
`/bell/ring` エンドポイントにアクセスできます 。

#### :one::seven: リクエスト

リクエストの完全なデータは非常に詳細であり、以下で省略されています :

<details>
  <summary>

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/gQqnLOnIEeiBFQJCrBIBDA/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
...etc
</Request>'
```

**(クリックすると拡大します)**

  </summary>

```console
curl -X POST \
  http://localhost:8080/authzforce-ce/domains/gQqnLOnIEeiBFQJCrBIBDA/pdp \
  -H 'Content-Type: application/xml' \
  -d '<?xml version="1.0" encoding="UTF-8"?>
<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">
  <Attributes Category="urn:oasis:names:tc:xacml:1.0:subject-category:access-subject">
     <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:subject:subject-id" IncludeInResult="false">
        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">charlie</AttributeValue>
     </Attribute>
     <Attribute AttributeId="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress" IncludeInResult="false">
        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">charlie-security@test.com</AttributeValue>
     </Attribute>
     <Attribute AttributeId="urn:oasis:names:tc:xacml:2.0:subject:role" IncludeInResult="false">
        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">security-role-0000-0000-000000000000</AttributeValue>
     </Attribute>
  </Attributes>
  <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:resource">
     <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:resource:resource-id" IncludeInResult="false">
        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">tutorial-dckr-site-0000-xpresswebapp</AttributeValue>
     </Attribute>
     <Attribute AttributeId="urn:thales:xacml:2.0:resource:sub-resource-id" IncludeInResult="false">
        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">/bell/ring</AttributeValue>
     </Attribute>
  </Attributes>
  <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:action">
     <Attribute AttributeId="urn:oasis:names:tc:xacml:1.0:action:action-id" IncludeInResult="false">
        <AttributeValue DataType="http://www.w3.org/2001/XMLSchema#string">POST</AttributeValue>
     </Attribute>
  </Attributes>
  <Attributes Category="urn:oasis:names:tc:xacml:3.0:attribute-category:environment" />
</Request>'
```

</details>

#### レスポンス

新しい `<Rule>` が適用されたときのレスポンスは `Permit` です。

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns2:Response xmlns="http://www.w3.org/2005/Atom" xmlns:ns2="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" xmlns:ns3="http://authzforce.github.io/core/xmlns/pdp/6.0" xmlns:ns4="http://authzforce.github.io/rest-api-model/xmlns/authz/5" xmlns:ns5="http://authzforce.github.io/pap-dao-flat-file/xmlns/properties/3.6">
    <ns2:Result>
        <ns2:Decision>Permit</ns2:Decision>
    </ns2:Result>
</ns2:Response>
```

<a name="tutorial-pep---extending-advanced-authorization"></a>

## PEP のチュートリアル  - 高度な認可の拡張

セキュリティ・マネージャである Charlie の新しいポリシーでは、
**Authzforce** に渡す追加情報が必要です。

<a name="extending-advanced-authorization---sample-code"></a>

### 高度な認可の拡張 - サンプル・コード

プログラム的には、Policy Execution Point は2つの部分から構成されます。
**Keyrock** に対する OAuth リクエストは、ユーザに関する情報
(割り当てられたロールなど) および、クエリされるポリシー・ドメインを
取得します。

2番目のリクエストが **Authzforce** 内の関連ドメイン・ンドポイントに
送信され、**Authzforce** が判断を下すために必要なすべての情報が
提供されます。**Authzforce** は **permit** または **deny**
のレスポンスで応答し、続行するかどうかの決定はその後行うことができます。

**Authzforce** に送信されるフィールドのリストに `user.username` と
`user.email` が追加されました。

```javascript
function authorizeAdvancedXACML(req, res, next, resource = req.url) {
    const keyrockUserUrl =
        "http://keyrock/user?access_token=" +
        req.session.access_token +
        "&app_id=" +
        clientId +
        "&authzforce=true";

    return oa
        .get(keyrockUserUrl)
        .then(response => {
            const user = JSON.parse(response);
            return azf.policyDomainRequest(
                user.app_azf_domain,
                user.roles,
                user.username,
                user.email,
                resource,
                req.method
            );
        })
        .then(authzforceResponse => {
            res.locals.authorized = authzforceResponse === "Permit";
            return next();
        })
        .catch(error => {
            debug(error);
            res.locals.authorized = false;
            return next();
        });
}
```

各リクエストを **Authzforce** に提供するための完全なコードは、チュートリアルの
[Git リポジトリ](https://github.com/FIWARE/tutorials.Step-by-Step/blob/master/context-provider/lib/azf.js)
にあります。提供された情報は、生成された XACML リクエスト内に`username` と `email`
を含むように拡張されました。

```javascript
const xml2js = require("xml2js");
const request = require("request");

function policyDomainRequest(domain, roles, resource, action, username, email) {
    let body =
        '<?xml version="1.0" encoding="UTF-8"?>\n' +
        '<Request xmlns="urn:oasis:names:tc:xacml:3.0:core:schema:wd-17" CombinedDecision="false" ReturnPolicyIdList="false">\n';
    // Code to create the XML body for the request is omitted
    body = body + "</Request>";

    const options = {
        method: "POST",
        url: "http://authzforceUrl/authzforce-ce/domains/" + domain + "/pdp",
        headers: { "Content-Type": "application/xml" },
        body
    };

    return new Promise((resolve, reject) => {
        request(options, function(error, response, body) {
            let decision;
            xml2js.parseString(
                body,
                { tagNameProcessors: [xml2js.processors.stripPrefix] },
                function(err, jsonRes) {
                    // The decision is found within the /Response/Result[0]/Decision[0] XPath
                    decision = jsonRes.Response.Result[0].Decision[0];
                }
            );
            decision = String(decision);
            return error ? reject(error) : resolve(decision);
        });
    });
}
```

<a name="extending-advanced-authorization---running-the-example"></a>

### 高度な認可の拡張 - 例の実行

**Authzforce** `<PolicySet>` を正常に更新して、Charlie の特別なルールを含めると、
その権限は他のセキュリティ・ロールのユーザとは異なります。

#### Detective1 (警備員 1)
Detective1 は Charlie に対応し、**セキュリティ**・ロールを果たします。

-   `http://localhost:3000` から、ユーザ `detective1@test.com`
    とパスワード `test` でログインします

##### レベル3 : 高度な認可アクセス

-   `http://localhost:3000` で、制限されたアクセス・リンクをクリック
    してください - アクセスが**拒否**されました - これは管理のみの権限です
-   `http://localhost:3000/device/monitor` でデバイス・モニタを開きます
    -   ドアのロックを解除 - アクセスは**許可**されています - これは
        セキュリティ上の唯一の許可です
    -   ベルを鳴らす - アクセスは**拒否**されます - これは、午前9時から
         午後5時の間にセキュリティ・ユーザには許可されていません

#### Charlie セキュリティ・マネージャ

Charlie は、**セキュリティ**・ロールを担っています。

-   `http://localhost:3000` から、ユーザ `charlie-security@test.com`
    とパスワード `test` でログインします

#### レベル3 : 高度な認可アクセス

-   `http://localhost:3000` で、制限されたアクセス・リンクをクリック
    してください - アクセスが**拒否**されました - これは管理のみの権限です
-   `http://localhost:3000/device/monitor` でデバイス・モニタを開きます
    -   ドアのロックを解除 - アクセスは**許可**されています - これは
        セキュリティ上の唯一の許可です
    -   ベルを鳴らす - アクセスは**許可**されます - これは、`charlie` という
        ユーザにのみ許可される例外です

<a name="next-steps"></a>

# 次のステップ

高度な機能を追加することで、アプリケーションに複雑さを加える方法を知りたいですか
？このシリーズの
[他のチュートリアル](https://www.letsfiware.jp/fiware-tutorials)を
読むことで見つけることができます。

---

## License

[MIT](LICENSE) © 2019 FIWARE Foundation e.V.
