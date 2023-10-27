#  Azure Active Directory B2C カスタム ポリシーを使用してユーザー入力を検証する 

Azure Active Directory B2C (Azure AD B2C) カスタム ポリシーを使用すると、ユーザー入力を必須にするだけでなく、それらを検証することもできます。 <DisplayClaim ClaimTypeReferenceId="givenName" Required="true"/> のようにユーザー入力を "必須" としてマークできますが、ユーザーが有効なデータを入力するとは限りません。 Azure AD B2C には、ユーザー入力を検証するためのさまざまな方法が用意されています。 この記事では、ユーザー入力を収集し、次の方法を使用してそれらを検証するカスタム ポリシーを作成する方法について説明します。

1. 選択するオプションの一覧を指定して、ユーザーが入力するデータを制限します。 この方法では、要求を宣言するときに "列挙値" を追加して使用します。

1. ユーザー入力が一致する必要があるパターンを定義します。 この方法では、要求を宣言するときに "正規表現" を追加して使用します。

1. ルールのセットを定義し、ユーザー入力が 1 つ以上のルールに従うように要求します。 この方法では、要求を宣言するときに "述語" を追加して使用します。

1. ユーザー入力の収集中に、ユーザーがパスワードを正しく再入力したことを検証するには、特殊な要求タイプ reenterPassword を使用します。

要求宣言レベルでは定義できない複雑なビジネス ルールを定義する "検証技術プロファイル" を構成します。 たとえば、別の要求内の他の値のセットに対して検証する必要があるユーザー入力を収集します。 


## 前提条件

- まだ持っていない場合は、お使いの Azure サブスクリプションにリンクされている Azure AD B2C テナントを作成します。

- Web アプリケーションを登録し、ID トークンの暗黙的な許可を有効にします。 リダイレクト URI には https://jwt.ms を使用します。

- お使いのコンピューターに Visual Studio Code (VS Code) がインストールされている必要があります。

- 「Azure AD B2C カスタム ポリシーを使用してユーザー入力を収集して操作する」の手順を完了します。 この記事は、独自のカスタム ポリシーの作成と実行に関するハウツー ガイド シリーズの一部です。

## Step 1 - ユーザー入力オプションを制限してユーザー入力を検証する

特定の入力に対してユーザーが入力できるすべての可能な値がわかっている場合は、ユーザーが選択する必要がある値の有限セットを指定できます。 この目的のために DropdownSinglSelect、CheckboxMultiSelect、RadioSingleSelect の UserInputType を使用できます。 この記事では、入力の種類として RadioSingleSelect を使用します。

1. VS Code でファイル `ContosoCustomPolicy.XML`を開きます。 

1. `ContosoCustomPolicy.XML` ファイルの `ClaimsSchema` 要素で、次の要求タイプを宣言します。
  
    ```xml
        <ClaimType Id="accountType">
            <DisplayName>Account Type</DisplayName>
            <DataType>string</DataType>
            <UserHelpText>The type of account used by the user</UserHelpText>
            <UserInputType>RadioSingleSelect</UserInputType>
            <Restriction>
                <Enumeration Text="Contoso Employee Account" Value="work" SelectByDefault="true"/>
                <Enumeration Text="Personal Account" Value="personal" SelectByDefault="false"/>
            </Restriction>
        </ClaimType>
    ```

accountType 要求を宣言しました。 要求の値がユーザーから収集されるときに、ユーザーは、値 work に対応する [Contoso Employee Account](Contoso 従業員アカウント) または値 personal に対応する [個人用アカウント] のいずれかを選択する必要があります。

Azure AD B2C では、ポリシーをさまざまな言語に対応させ、複数の言語でアカウントの種類の制限を指定することもできます。 詳細については、[ユーザー属性の追加に関する記事](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/configure-user-input?pivots=b2c-custom-policy)の [UI のローカライズ](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/configure-user-input?pivots=b2c-custom-policy#optional-localize-the-ui)に関するページを確認してください。

1. `Id="UserInformationCollector"`がある技術プロファイルを見つけ、次のコードを使用して *accountType* 要求を表示要求として追加します。

    ```xml
        <DisplayClaim ClaimTypeReferenceId="accountType" Required="true"/>
    ```  
1. `Id="UserInformationCollector"`がある技術プロファイルで、次のコードを使用して *accountType* 要求を出力要求として追加します。 

    ```xml
        <OutputClaim ClaimTypeReferenceId="accountType"/>
    ```

1. アクセス トークンにアカウントの種類の要求を含めるには、`RelyingParty` 要素を見つけ、次のコードを使用して *accountType* 要求をトークン要求として追加します。 

    ```xml
        <OutputClaim ClaimTypeReferenceId="accountType" />
    ```

## Step 2 - 正規表現を使用してユーザー入力を検証する

すべての可能なユーザー入力値が事前にわかっていない場合は、ユーザーが自分でデータを入力することを許可します。 この場合は、"正規表現" または "パターン" を使用して、ユーザー入力に必要な形式を指定できます。 たとえば、メール アドレスのテキストでは、どこかに "アットマーク (@)" 記号と "ピリオド (.)" が含まれている必要があります。

要求を宣言するときには、カスタム ポリシーを使用して、ユーザー入力が一致する必要がある正規表現を定義できます。 必要に応じて、入力が式と一致しない場合にユーザーに表示されるメッセージを指定できます。



1.  `ClaimsSchema` 要素を見つけ、次のコードを使用して *email* 要求を宣言します。

    ```xml
        <ClaimType Id="email">
            <DisplayName>Email Address</DisplayName>
            <DataType>string</DataType>
            <DefaultPartnerClaimTypes>
                <Protocol Name="OpenIdConnect" PartnerClaimType="email"/>
            </DefaultPartnerClaimTypes>
            <UserHelpText>Your email address. </UserHelpText>
            <UserInputType>TextBox</UserInputType>
            <Restriction>
                <Pattern RegularExpression="^[a-zA-Z0-9.!#$%&amp;&apos;^_`{}~-]+@[a-zA-Z0-9-]+(?:\.[a-zA-Z0-9-]+)*$" HelpText="Please enter a valid email address something like maurice@contoso.com"/>
            </Restriction>
        </ClaimType>
    ``` 

1. `Id="UserInformationCollector"`がある技術プロファイルを見つけ、次のコードを使用して email 要求を表示要求として追加します。

    ```xml
        <DisplayClaim ClaimTypeReferenceId="email" Required="true"/>
    ```

1. `Id="UserInformationCollector"`がある技術プロファイルで、次のコードを使用して email 要求を出力要求として追加します。

    ```xml
        <OutputClaim ClaimTypeReferenceId="email"/>
    ```

1. `RelyingParty` 要素を見つけ、次のコードを使用して email をトークン要求として追加します。

    ```xml
        <OutputClaim ClaimTypeReferenceId="email" />
    ```

## Step 3 - *predicates* 使用してユーザー入力を検証する

正規表現を使用してユーザー入力を検証しました。 ただし、正規表現には弱点が 1 つあります。それは、入力を修正するまでエラー メッセージが表示されるが、入力がどの要件を満たしていないかが具体的に示されないことです。

*predicates*の検証を使用すると、ルール (*predicates*) のセットと、ルールごとの独立したエラー メッセージを定義することで、この問題に対処できます。 カスタム ポリシーでは、述語に組み込みのメソッドがあります。これを使用して、作成するチェックを定義します。 たとえば、IsLengthRange の*predicates*メソッドを使用して、ユーザーの password が、指定された最小パラメーター (値) と最大パラメーター (値) の範囲内にあるかどうかのチェックを行うことができます。

Predicates によって、要求タイプに対してチェックを行う検証が定義されますが、PredicateValidations は要求タイプに適用できるユーザー入力検証を形式設定する*predicates*セットをグループ化します。 たとえば、パスワードに使用できるさまざまな種類の文字を検証する検証の*predicates*グループを作成します。 Predicates 要素と PredicateValidations 要素はどちらも、ポリシー ファイルの BuildingBlocks セクションの子要素です。


1. `ClaimsSchema` 要素を見つけ、次のコードを使用して *password* 要求を宣言します。

    ```xml
        <ClaimType Id="password">
          <DisplayName>Password</DisplayName>
          <DataType>string</DataType>
          <AdminHelpText>Enter password</AdminHelpText>
          <UserHelpText>Enter password</UserHelpText>
          <UserInputType>Password</UserInputType>
        </ClaimType>
    ``` 

1. 次のコードを使用して`Predicates` 要素を`BuildingBlocks` セクションの子として追加します。

    ```xml
        <Predicates>
        
        </Predicates>
    ```

1. `Predicates` 要素内で、次のコードを使用して述語を定義します。

    ```xml
      <Predicate Id="IsLengthBetween8And64" Method="IsLengthRange" HelpText="The password must be between 8 and 64 characters.">
        <Parameters>
          <Parameter Id="Minimum">8</Parameter>
          <Parameter Id="Maximum">64</Parameter>
        </Parameters>
      </Predicate>
    
      <Predicate Id="Lowercase" Method="IncludesCharacters" HelpText="a lowercase letter">
        <Parameters>
          <Parameter Id="CharacterSet">a-z</Parameter>
        </Parameters>
      </Predicate>
    
      <Predicate Id="Uppercase" Method="IncludesCharacters" HelpText="an uppercase letter">
        <Parameters>
          <Parameter Id="CharacterSet">A-Z</Parameter>
        </Parameters>
      </Predicate>
    
      <Predicate Id="Number" Method="IncludesCharacters" HelpText="a digit">
        <Parameters>
          <Parameter Id="CharacterSet">0-9</Parameter>
        </Parameters>
      </Predicate>
    
      <Predicate Id="Symbol" Method="IncludesCharacters" HelpText="a symbol">
        <Parameters>
          <Parameter Id="CharacterSet">@#$%^&amp;*\-_+=[]{}|\\:',.?/`~"();!</Parameter>
        </Parameters>
      </Predicate>
    
      <Predicate Id="PIN" Method="MatchesRegex" HelpText="The password must be numbers only.">
        <Parameters>
          <Parameter Id="RegularExpression">^[0-9]+$</Parameter>
        </Parameters>
      </Predicate>
    
      <Predicate Id="AllowedCharacters" Method="MatchesRegex" HelpText="An invalid character was provided.">
        <Parameters>
          <Parameter Id="RegularExpression">(^([0-9A-Za-z\d@#$%^&amp;*\-_+=[\]{}|\\:',?/`~"();! ]|(\.(?!@)))+$)|(^$)</Parameter>
        </Parameters>
      </Predicate>
    
      <Predicate Id="DisallowedWhitespace" Method="MatchesRegex" HelpText="The password must not begin or end with a whitespace character.">
        <Parameters>
          <Parameter Id="RegularExpression">(^\S.*\S$)|(^\S+$)|(^$)</Parameter>
        </Parameters>
      </Predicate>
    ```

    いくつかのルールを定義しました。これらを組み合わせることで、許容されるパスワードが記述されます。 次に、*Predicate*をグループ化して、ポリシーで使用できるパスワード ポリシーのセットを形成できます。 

1. 次のコードを使用して `PredicateValidations` 要素を `BuildingBlocks` セクションの子として追加します。

    ```xml
        <PredicateValidations>
        
        </PredicateValidations>
    ``` 
1. `PredicateValidations` 要素内で、次のコードを使用して PredicateValidations を定義します。

    ```xml
        <PredicateValidation Id="SimplePassword">
            <PredicateGroups>
                <PredicateGroup Id="DisallowedWhitespaceGroup">
                    <PredicateReferences>
                        <PredicateReference Id="DisallowedWhitespace"/>
                    </PredicateReferences>
                </PredicateGroup>
                <PredicateGroup Id="AllowedCharactersGroup">
                    <PredicateReferences>
                        <PredicateReference Id="AllowedCharacters"/>
                    </PredicateReferences>
                </PredicateGroup>
                <PredicateGroup Id="LengthGroup">
                    <PredicateReferences>
                        <PredicateReference Id="IsLengthBetween8And64"/>
                    </PredicateReferences>
                </PredicateGroup>
            </PredicateGroups>
        </PredicateValidation>
        <PredicateValidation Id="StrongPassword">
            <PredicateGroups>
                <PredicateGroup Id="DisallowedWhitespaceGroup">
                    <PredicateReferences>
                        <PredicateReference Id="DisallowedWhitespace"/>
                    </PredicateReferences>
                </PredicateGroup>
                <PredicateGroup Id="AllowedCharactersGroup">
                    <PredicateReferences>
                        <PredicateReference Id="AllowedCharacters"/>
                    </PredicateReferences>
                </PredicateGroup>
                <PredicateGroup Id="LengthGroup">
                    <PredicateReferences>
                        <PredicateReference Id="IsLengthBetween8And64"/>
                    </PredicateReferences>
                </PredicateGroup>
                <PredicateGroup Id="CharacterClasses">
                    <UserHelpText>The password must have at least 3 of the following:</UserHelpText>
                    <PredicateReferences MatchAtLeast="3">
                        <PredicateReference Id="Lowercase"/>
                        <PredicateReference Id="Uppercase"/>
                        <PredicateReference Id="Number"/>
                        <PredicateReference Id="Symbol"/>
                    </PredicateReferences>
                </PredicateGroup>
            </PredicateGroups>
        </PredicateValidation>
        <PredicateValidation Id="CustomPassword">
            <PredicateGroups>
                <PredicateGroup Id="DisallowedWhitespaceGroup">
                    <PredicateReferences>
                        <PredicateReference Id="DisallowedWhitespace"/>
                    </PredicateReferences>
                </PredicateGroup>
                <PredicateGroup Id="AllowedCharactersGroup">
                    <PredicateReferences>
                        <PredicateReference Id="AllowedCharacters"/>
                    </PredicateReferences>
                </PredicateGroup>
            </PredicateGroups>
        </PredicateValidation>
    ```
    3 つの述語の検証 *StrongPassword*、*CustomPassword*、*SimplePassword* を定義しました。 ユーザーに入力させたいパスワードの特性に応じて、述語の検証でいずれかを使用できます。 この記事では、強力なパスワードを使用します。

1. *password*  要求タイプ宣言を見つけ、次のコードを使用して、含まれている UserInputType 要素宣言の直後に述語の検証 *StrongPassword* を追加します。

    ```xml
        <PredicateValidationReference Id="StrongPassword" />
    ``` 
1. `Id="UserInformationCollector"`がある技術プロファイルを見つけ、次のコードを使用して password 要求を表示要求として追加します。

    ```xml
        <DisplayClaim ClaimTypeReferenceId="password" Required="true"/>
    ```

1. `Id="UserInformationCollector"`がある技術プロファイルを見つけ、次のコードを使用して password 要求を表示要求として追加します。

    ```xml
        <OutputClaim ClaimTypeReferenceId="password"/>
    ``` 

> [!NOTE]    
> セキュリティ上の理由から、ポリシーによって生成されたトークンにユーザーのパスワードを要求として追加することはしません。 そのため、Relying Party 要素には password 要求を追加しません。

## Step 4 - パスワードを検証し、パスワードを確認する

ユーザーが入力したパスワードを記憶していることを確認する手段として、ユーザーにパスワードを 2 回入力するように要求できます。 この場合は、2 つのエントリの値が一致することを確認する必要があります。 カスタム ポリシーを使用すると、この要件を簡単に実現できます。 要求タイプ password と reenterPassword は特殊であると見なされるため、これらがユーザー入力の収集に使用されると、UI は、ユーザーがパスワードを正しく再入力したことを検証します。

カスタム ポリシーでパスワードの再入力を検証するには、次の手順に従います。

1. `ContosoCustomPolicy.XML`フィルの`ClaimsSchema` セクションで、次のコードを使用して、password 要求の直後に reenterPassword 要求を宣言します。

    ```xml
        <ClaimType Id="reenterPassword">
            <DisplayName>Confirm new password</DisplayName>
            <DataType>string</DataType>
            <AdminHelpText>Confirm new password</AdminHelpText>
            <UserHelpText>Reenter password</UserHelpText>
            <UserInputType>Password</UserInputType>
        </ClaimType>    
    ```
1. パスワード確認入力をユーザーから収集するには、セルフアサートの技術プロファイル `UserInformationCollector` を見つけ、次のコードを使用して *reenterPassword* 要求を表示要求として追加します。

    ```xml
        <DisplayClaim ClaimTypeReferenceId="reenterPassword" Required="true"/>
    ```
1. `ContosoCustomPolicy.XML` ファイルで、セルフアサートの技術プロファイル `UserInformationCollector` を見つけ、次のコードを使用して *reenterPassword* 要求を出力要求と

    ```xml
        <OutputClaim ClaimTypeReferenceId="reenterPassword"/>
    ```

## Step 5 - カスタム ポリシー ファイルをアップロードする

[「カスタム ポリシー ファイルをアップロードする」](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/custom-policies-series-hello-world#step-3---upload-custom-policy-file)の手順に従います。 ポータルに既にあるファイルと同じ名前のファイルをアップロードする場合は、[カスタム ポリシーが既に存在する場合は上書きします] を選択してください。

## Step 6 - カスタム ポリシーをテストする

1. カスタム ポリシー で、[B2C_1A_CONTOSOCUSTOMPOLICY] を選択します。

1. カスタム ポリシーの概要ページの [アプリケーションの選択] で、以前に登録した webapp1 などの Web アプリケーションを選択します。 [応答 URL の選択] の値が https://jwt.ms に設定されていることを確認します。

1. [今すぐ実行] ボタンを選択します。

1. [名] と [姓] を入力します。

1. [アカウントの種類] を選択します。

1. [メール アドレス] に、maurice@contoso などの不適切な形式のメール アドレス値を入力します。

1. [パスワード] に、設定されている強力なパスワードのすべての特性には従わないパスワード値を入力します。

1. [続行] ボタンを選択します。 下に示すような画面が表示されます。

    ![ScreenShot](/media/screenshot-of-user-input-validation.png)

   続行する前に、入力を修正する必要があります。

1. エラー メッセージで推奨される正しい値を入力し、もう一度 [Continue] ボタンを選択します。 ポリシーの実行が完了すると、https://jwt.ms にリダイレクトされ、デコードされた JWT トークンが表示されます。 トークンは、次の JWT トークン スニペットのようになります。

```json
    {
      "typ": "JWT",
      "alg": "RS256",
      "kid": "pxLOMWFg...."
    }.{
      ...
      "sub": "c7ae4515-f7a7....",
      ...
      "acr": "b2c_1a_contosocustompolicy",
      "accountType": "work",
      ...
      "email": "maurice@contoso.com",
      "name": "Maurice Paulet",
      "message": "Hello Maurice Paulet"
    }.[Signature]
``` 

## Step 7 - 検証技術プロファイルを使用してユーザー入力を検証する

Step 1、Step 2、Step 3 で使用した検証手法は、すべてのシナリオに適用できるわけではありません。 ビジネス ルールが複雑であり、要求宣言レベルでは定義できない場合は、検証技術を構成し、それをセルフアサートの技術プロファイルから呼び出すことができます。

> [!NOTE] 
> セルフアサート技術プロファイルでのみ、検証技術プロファイルを使用できます。 [検証技術プロファイル](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/validation-technical-profile)の詳細情報
 
### シナリオの概要

ユーザーの [アカウントの種類] が [Contoso Employee Account](Contoso 従業員アカウント) である場合は、メール ドメインが定義済みのドメインのセットに基づいていることを確認する必要があります。 これらのドメインは、contoso.com、fabrikam.com、woodgrove.com です。 それ以外の場合は、有効な [Contoso Employee Account](Contoso 従業員アカウント) を使用するか、[個人用アカウント] に切り替えるまで、ユーザーにエラーが表示されます。

検証技術プロファイルを使用してユーザー入力を検証する方法については、次の手順に従います。 要求変換の種類の検証技術プロファイルを使用しますが、REST API サービスを呼び出してデータを検証することもできます。これについては、このシリーズの後の方で説明します。



1.  `ContosoCustomPolicy.XML` ファイルの `ClaimsSchema` セクションで、次のコードを使用して *domain* 要求と *domainStatus* 要求を宣言します。

    ```xml
        <ClaimType Id="domain">
          <DataType>string</DataType>
        </ClaimType>
        
        <ClaimType Id="domainStatus">
          <DataType>string</DataType>
        </ClaimType>
    ```        
1. `ClaimsTransformations`セクションを見つけ、次のコードを使用して要求変換を構成します。

    ```xml
        <ClaimsTransformation Id="GetDomainFromEmail" TransformationMethod="ParseDomain">
            <InputClaims>
                <InputClaim ClaimTypeReferenceId="email" TransformationClaimType="emailAddress"/>
            </InputClaims>
            <OutputClaims>
                <OutputClaim ClaimTypeReferenceId="domain" TransformationClaimType="domain"/>
            </OutputClaims>
        </ClaimsTransformation>
        <ClaimsTransformation Id="LookupDomain" TransformationMethod="LookupValue">
            <InputClaims>
                <InputClaim ClaimTypeReferenceId="domain" TransformationClaimType="inputParameterId"/>
            </InputClaims>
            <InputParameters>
                <InputParameter Id="contoso.com" DataType="string" Value="valid"/>
                <InputParameter Id="fabrikam.com" DataType="string" Value="valid"/>
                <InputParameter Id="woodgrove.com" DataType="string" Value="valid"/>
                <InputParameter Id="errorOnFailedLookup" DataType="boolean" Value="true"/>
            </InputParameters>
            <OutputClaims>
                <OutputClaim ClaimTypeReferenceId="domainStatus" TransformationClaimType="outputClaim"/>
            </OutputClaims>
        </ClaimsTransformation>
    ``` 
    
    GetDomainFromEmail 要求変換は、[ParseDomain](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/string-transformations#parsedomain) メソッドを使用してメールからドメインを抽出し、それを domain 要求に格納します。 LookupDomain 要求変換は、抽出されたドメインを使用して、それを定義済みのドメインのセットで検索し、domainStatus 要求に valid を割り当てることによって、それが有効であるかどうかを確認します。

1. `Id=UserInformationCollector`がある技術プロファイルと同じ要求プロバイダーに技術プロファイルを追加するには、次のコードを使用します。

    ```xml
        <TechnicalProfile Id="CheckCompanyDomain">
            <DisplayName>Check Company validity </DisplayName>
            <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.ClaimsTransformationProtocolProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"/>
            <InputClaimsTransformations>
                <InputClaimsTransformation ReferenceId="GetDomainFromEmail"/>
            </InputClaimsTransformations>
            <OutputClaims>
                <OutputClaim ClaimTypeReferenceId="domain"/>
            </OutputClaims>
            <OutputClaimsTransformations>
                <OutputClaimsTransformation ReferenceId="LookupDomain"/>
            </OutputClaimsTransformations>
        </TechnicalProfile>
    ```   
    GetDomainFromEmail 要求変換と LookupDomain 要求変換を実行する要求変換の技術プロファイルを宣言しました。

1. `Id=UserInformationCollector`がある技術プロファイルを見つけ、次のコードを使用して `OutputClaims` 要素の直後に `ValidationTechnicalProfile` を追加します。

    ```xml
        <ValidationTechnicalProfiles>
            <ValidationTechnicalProfile ReferenceId="CheckCompanyDomain">
                <Preconditions>
                    <Precondition Type="ClaimEquals" ExecuteActionsIf="false">
                        <Value>accountType</Value>
                        <Value>work</Value>
                        <Action>SkipThisValidationTechnicalProfile</Action>
                    </Precondition>
                </Preconditions>
            </ValidationTechnicalProfile>
        </ValidationTechnicalProfiles>
    ```
    
セルフアサートの技術プロファイル UserInformationCollector に検証技術プロファイルを追加しました。 技術プロファイルは、accountType 値が work に等しくない場合にのみスキップされます。 技術プロファイルが実行され、メール ドメインが有効でない場合は、エラーがスローされます。

1. `Id=UserInformationCollector`, がある技術プロファイルを見つけ、`metadata` タグ内に次のコードを追加します。
 
    ```xml
        <Item Key="LookupNotFound">The provided email address isn't a valid Contoso Employee email.</Item>
     ```
    
    ユーザーが有効なメール アドレスを使用しない場合に備えて、カスタム エラーを設定しました。

1. [「カスタム ポリシー ファイルをアップロードする」](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/custom-policies-series-hello-world#step-3---upload-custom-policy-file)の手順に従って、ポリシー ファイルをアップロードします。

1. Step 6 の手順に従って、カスタム ポリシーをテストします。
    1. [アカウントの種類] で、[Contoso Employee Account](Contoso 従業員アカウント) を選択します。
    1. [メール アドレス] に、maurice@fourthcoffee.com などの無効なメール アドレスを入力します。
    1. 必要に応じて残りの詳細を入力し、[Continue] を選択します。
    
    maurice@fourthcoffee.com は有効なメール アドレスではないため、下のスクリーンショットに示すようなエラーが表示されます。 有効なメール アドレスを使用して、カスタム ポリシーを正常に実行し、JWT トークンを受け取る必要があります。

    ![ScreenShot](/media/screenshot-of-error-due-to-invalid-email-address.png)" 

