# オファー

XRP Ledgerの分散型取引所では、通貨の取引注文は「オファー」と呼ばれます。オファーはXRPと発行済み通貨の取引、または発行済み通貨間の取引（同一通貨コードだがイシュアーが異なる発行済み通貨を含む）を行うことができます。（同一コードでイシュアーが異なる通貨は、[rippling](rippling.html)によって取引することもできます。）

- オファーを作成するには、[OfferCreateトランザクション][]を送信します。
- 即時に履行されないオファーはレジャーデータの[Offerオブジェクト](offer.html)になります。その後のオファーとPaymentにより、レジャーのOfferオブジェクトは消費されます。
- [複数通貨間の支払い](cross-currency-payments.html)はオファーを消費して流動性を提供します。


## オファーのライフサイクル

OfferCreateトランザクションの処理時に、このトランザクションは一致するオファーまたはクロスオファーを可能な限り自動的に消費します。（既存のオファーのレートが要求よりも良い場合には、オファーの作成者は`TakerGets`の全額よりも低い額を支払い、`TakerPays`を全額を受領できます。）それにより`TakerPays`の額を完全に満たせない場合、オファーはレジャーのOfferオブジェクトになります。（[OfferCreateフラグ](offercreate.html#offercreateフラグ)を使用してこの動作を変更できます。）

既存のオファーに対応する追加のOfferCreateトランザクション、またはオファーを使用してペイメントパスを接続する[Paymentトランザクション][]のいずれかにより、レジャー上のオファーは履行されます。オファーの部分的な履行と、部分的な資金化が可能です。1つのトランザクションで、レジャーのオファーを最大850件まで消費できます。（この数を超えるとメタデータが大きくなり過ぎて、[`tecOVERSIZE`](tec-codes.html)となります。）

オファーの`TakerGets`パラメーターで指定された通貨をいくらかでも（ゼロ以外のプラスの額）保有している限り、オファーを作成できます。オファーは、`TakerPays`の額が満たされるまで、`TakerGets`の額を上限として保有している通貨を売却します。オファーによって誰かに負債が発生することはありません。

レジャーにすでに記録されているオファーにクロスするオファーを出した場合、関連する額に関係なく古いオファーは自動的に取り消されます。

次の場合には、オファーは一時的または永久に _資金化されない_ 可能性があります。

* 作成者に`TakerGets`の通貨がない場合。
    * 作成者がその通貨を取得すると、オファーに資金が再び供給されます。
* オファーに資金を供給するために必要な通貨が[凍結されたトラストライン](freezes.html)で保有されている場合。
    * トラストラインの凍結が解除されると、オファーに資金が再び供給されます。
* オファーに必要な新しいトラストラインの準備金として十分な額のXRPを作成者が保有していない場合。（[オファーとトラスト](#オファーとトラスト)を参照してください。）
    * 作成者がXRPをさらに取得するか、または必要準備金が減額されると、オファーに資金が再び供給されます。
* オファーに指定されている有効期限が最新の閉鎖済みレジャーの閉鎖時刻よりも前である場合。（[オファーの有効期限](#オファーの有効期限)を参照してください。）

資金化されないオファーはレジャーに無期限で残ることがありますが、特に影響はありません。次の方法でのみ、オファーはレジャーから*完全に*削除されます。

* Paymentトランザクションまたは対応するOfferCreateトランザクションにより全額が請求される。
* OfferCancelトランザクションまたはOfferCreateトランザクションによりオファーが明示的に取り消される。
* 同じアカウントからのOfferCreateトランザクションが以前のオファーにクロスする。（この場合、古いオファーが自動的に取り消されます。）
* トランザクションの処理中にオファーへの資金が供給されていないことが判明する。一般的に、オファーがオーダーブックの先中で最も有利なレートであったことが原因です。
    * これには、オファーのいずれかの側が、`rippled`の精度でサポートされているよりも0に近いことが検出されるケースも含まれます。

### 資金化されていないオファーの追跡

すべてのオファーの資金化状況の追跡は、コンピューターにとって負荷の高い処理となることがあります。特に積極的に取引しているアドレスでは大量のオファーがオープンです。1つの残高が、さまざまな通貨を購入する多数のオファーの資金化の状況に影響することがあります。このため、`rippled`ではオファーの検出と削除を積極的に行なっていません。

クライアントアプリケーションでオファーの資金化の状況をローカルで追跡できます。このためには、最初に[book_offersメソッド][]を使用してオーダーブックを取得し、次にオファーの`taker_gets_funded`フィールドを調べます。次に`transactions`ストリームを[サブスクライブ](subscribe.html)し、トランザクションメタデータを監視してどのオファーが変更されるかを確認します。


## オファーとトラスト

トラストラインの限度額（[TrustSet](trustset.html)を参照）はオファーに影響しません。つまり、オファーを使用して、イシュアーの信用できる最大精算額を上回る額を取得できます。

ただし、XRP以外の残高を保有するには、それらの残高を発行するアドレスへのトラストラインが必要です。オファーが受け入れられると、必要なトラストラインが自動的に作成され、その限度額が0に設定されます。[アカウントが保有する必要のある準備金はトラストラインによって増加する](reserves.html)ため、新しいトラストラインを必要とするオファーがある場合、そのトラストラインの準備金として十分なXRPをアドレスに保有する必要があります。

トラストラインはあなたが信用するイシュア―を示し、限度額の範囲内でそのイシュア―のイシュアンスを支払いとして受領します。オファーは特定通貨を取得するための明示的な指示であるため、これらの限度額を超えることができます。


## オファーの優先度

既存のオファーは為替レート（「オファークオリティ」とも呼ばれます）によってグループにまとめられます。為替レートは、`TakerGets`と`TakerPays`の比率として計算されます。為替レートが高いオファーが優先的に処理されます。（つまり、オファーを受け入れる人は、支払われる通貨額に対して可能な限り多くを受領します。）同じ為替レートのオファーは、最も古いレジャーバージョンで出されたオファーに基づいて受け入れられます。

同じ為替レートのオファーが同じレジャーバージョンに記録されている場合、オファーの処理順序は[レジャーへトランザクションを適用する](https://github.com/ripple/rippled/blob/5425a90f160711e46b2c1f1c93d68e5941e4bfb6/src/ripple/app/consensus/LedgerConsensus.cpp#L1435-L1538 "Source: Applying transactions")ための[正規の順序](https://github.com/ripple/rippled/blob/release/src/ripple/app/misc/CanonicalTXSet.cpp "Source: Transaction ordering")によって決定します。この動作は確定的かつ効率的であり、操作することが困難であるように設計されています。


## オファーの有効期限

トランザクションの伝搬と確定には時間がかかることがあるため、レジャーのタイムスタンプからオファーの有効性を判断します。オファーが有効期限切れとなるのは、有効期限が最新の検証済みレジャーよりも前の場合だけです。つまり、`Expiration`フィールドが指定されているオファーが「アクティブ」であると見なされるのは、ローカルクロックに関係なく、その有効期限が最新の検証済みレジャーのタイムスタンプよりも後である場合です。

閉鎖時刻が有効期限と同じかそれよりも後である完全な検証済みレジャーが作成されたら、`Expiration`が指定されているオファーの最終的な処理を即時に判断できます。

**注記:** レジャーを変更できるのは新しいトランザクションだけであることから、有効期限切れのオファーは、非アクティブになった後でもレジャーに残ることがあります。このオファーは資金化されていないと見なされ特に影響はありませんが、（[ledger_entry](ledger_entry.html)コマンドなどの）実行結果に、引き続き表示される可能性があります。後に、サーバーが処理中に有効期限切れのオファーを検出すると、このオファーは別のトランザクション（別のOfferCreateなど）の結果として最終的に削除されることがあります。

OfferCreateトランザクションが最初にレジャーに追加された時点で、このトランザクションに指定されている`Expiration`時刻をすでに経過していた場合は、トランザクションはオファーを実行しません。このようなトランザクションの結果コードは、[Checks Amendment][]:not_enabled:が有効であるかどうかによって異なります。Checks Amendmentが有効な場合、トランザクションの結果コードは`tecEXPIRED`です。それ以外の場合、トランザクションの結果コードは`tesSUCCESS`です。いずれの場合でも、このトランザクションには、[トランザクションコスト](transaction-cost.html)として支払われたXRPを消却する以外に影響はありません。


<!--{# common link defs #}-->
{% include '_snippets/rippled-api-links.md' %}
{% include '_snippets/tx-type-links.md' %}
{% include '_snippets/rippled_versions.md' %}
