
# Azure Active Directory B2C カスタム ポリシーを使用してユーザー入力を収集して操作する

Azure Active Directory B2C (Azure AD B2C) カスタム ポリシーを使用すると、ユーザー入力を収集できます。 その後、組み込みのメソッドを使用して、ユーザー入力を操作できます。
この記事では、グラフィカル ユーザー インターフェイスを使用してユーザー入力を収集するカスタム ポリシーを記述する方法について説明します。 
その後、入力にアクセスし、処理してから、最後に JWT トークンでクレームとして返します。 このタスクを完了するには、次を実行します。

- クレームを宣言する。 要求は、Azure AD B2C ポリシーの実行時に、データの一時的なストレージとなります。 ユーザーに関する情報 (名、姓など) や、ユーザーまたは他のシステムから取得したその他のクレームを格納できます。 クレームの詳細については、[「Azure AD B2C カスタム ポリシーの概要」](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/custom-policy-overview#claims)を参照してください。

- 技術プロファイルを定義する。 技術プロファイルには、さまざまな種類の利用者と通信するためのインターフェイスが用意されています。 たとえば、ユーザーと対話してデータを収集できます。

- クレーム変換を構成します。これは、宣言するクレームを操作するために使用します。

- コンテンツ定義を構成する。 コンテンツ定義には、読み込むユーザー インターフェイスが定義されます。 後で、カスタマイズした独自の HTML コンテンツを提供することで、ユーザー インターフェイスをカスタマイズできます。

- Self-Asserted 技術プロファイルと DisplayClaims を使用して、ユーザー インターフェイスを構成してユーザーに表示します。

- オーケストレーション ステップを使用して、特定のシーケンスで技術プロファイルを呼び出します。


## Step 1 - クレームを宣言する

*objectId* と *message* と共に追加のクレームを宣言します。: 

1. VS Code で `ContosoCustomPolicy.XML` ファイルを開きます。 

1. 'ClaimsSchema' セクションに、次の 'ClaimType' の宣言を追加します。: 

    ```xml
        <ClaimType Id="givenName">
            <DisplayName>Given Name</DisplayName>
            <DataType>string</DataType>
            <UserHelpText>Your given name (also known as first name).</UserHelpText>
            <UserInputType>TextBox</UserInputType>
        </ClaimType>
        
        <ClaimType Id="surname">
            <DisplayName>Surname</DisplayName>
            <DataType>string</DataType>
            <UserHelpText>Your surname (also known as family name or last name).</UserHelpText>
            <UserInputType>TextBox</UserInputType>
        </ClaimType>
        <ClaimType Id="displayName">
            <DisplayName>Display Name</DisplayName>
            <DataType>string</DataType>
            <UserHelpText>Your display name.</UserHelpText>
            <UserInputType>TextBox</UserInputType>
        </ClaimType>
    ```  

 givenName、surname、displayName の 3 つのクレームの型を宣言しました。 これらの宣言には、DataType、UserInputType、DisplayName の要素が含まれています。

- **DataType** は、クレームで保持する値のデータ型を指定しています。 DataType 要素がサポートするデータ型の詳細について確認してください。 
- **UserInputType** は、ユーザーからクレームの値を収集する場合にユーザー インターフェイスに表示される UI コントロールを指定します。 Azure AD B2C でサポートされているユーザー入力の種類の詳細について確認してください。   
- **DisplayName** は、ユーザーからクレームの値を収集する場合にユーザー インターフェイスに表示される UI コントロールのラベルを指定します。


## Step 2 - クレームの変換を定義する

A ClaimsTransformation には、特定のクレームを別のものに変換するために使用する関数が含まれます。 たとえば、文字列のクレームを小文字から大文字に変更できます。 Azure AD B2C でサポートされるクレーム変換の詳細について確認してください

1. ContosoCustomPolicy.XML ファイルで、BuildingBlocks セクションの子として <ClaimsTransformations> 要素を追加します。
    ```xml
        <ClaimsTransformations>
        
        </ClaimsTransformations>
    ```

1. ClaimsTransformations 要素内に次のコードを追加します。

    ```xml
        <ClaimsTransformation Id="GenerateRandomObjectIdTransformation" TransformationMethod="CreateRandomString">
            <InputParameters>
            <InputParameter Id="randomGeneratorType" DataType="string" Value="GUID"/>
            </InputParameters>
            <OutputClaims>
            <OutputClaim ClaimTypeReferenceId="objectId" TransformationClaimType="outputClaim"/>
            </OutputClaims>
        </ClaimsTransformation>
    
        <ClaimsTransformation Id="CreateDisplayNameTransformation" TransformationMethod="FormatStringMultipleClaims">
            <InputClaims>
            <InputClaim ClaimTypeReferenceId="givenName" TransformationClaimType="inputClaim1"/>
            <InputClaim ClaimTypeReferenceId="surname" TransformationClaimType="inputClaim2"/>
            </InputClaims>
            <InputParameters>
            <InputParameter Id="stringFormat" DataType="string" Value="{0} {1}"/>
            </InputParameters>
            <OutputClaims>
            <OutputClaim ClaimTypeReferenceId="displayName" TransformationClaimType="outputClaim"/>
            </OutputClaims>
        </ClaimsTransformation>
    
        <ClaimsTransformation Id="CreateMessageTransformation" TransformationMethod="FormatStringClaim">
            <InputClaims>
            <InputClaim ClaimTypeReferenceId="displayName" TransformationClaimType="inputClaim"/>
            </InputClaims>
            <InputParameters>
            <InputParameter Id="stringFormat" DataType="string" Value="Hello {0}"/>
            </InputParameters>
            <OutputClaims>
            <OutputClaim ClaimTypeReferenceId="message" TransformationClaimType="outputClaim"/>
            </OutputClaims>
        </ClaimsTransformation> 
    ```

   次の 3 つのクレーム変換を構成しました。

    - *GenerateRandomObjectIdTransformation* は、CreateRandomString メソッドで指定されたランダムな文字列を生成します。 objectId クレームは、OutputClaim 要素で指定されているように生成された文字列で更新されます。
        
    - *CreateDisplayNameTransformation* は givenName と surname を連結して displayName を作成します。
    
    -  *CreateMessageTransformation* は Hello と displayName を連結して message を作成します。

## Step 3 - コンテンツ定義を構成する

ContentDefinitions を使用すると、ユーザーに表示する Web ページのレイアウトを制御する HTML テンプレートへの URL を指定できます。 サインインまたはサインアップ、パスワードのリセット、エラー ページなど、各ステップに対して特定のユーザー インターフェイスを指定できます。

コンテンツ定義を追加するには、ContosoCustomPolicy.XML ファイルの BuildingBlocks セクションに次のコードを追加します。

```xml
    <ContentDefinitions>
        <ContentDefinition Id="SelfAssertedContentDefinition">
            <LoadUri>~/tenant/templates/AzureBlue/selfAsserted.cshtml</LoadUri>
            <RecoveryUri>~/common/default_page_error.html</RecoveryUri>
            <DataUri>urn:com:microsoft:aad:b2c:elements:contract:selfasserted:2.1.7</DataUri>
        </ContentDefinition>
    </ContentDefinitions>
```   

## Step 4 - 技術プロファイルを構成する

カスタム ポリシーでは、TechnicalProfile が機能を実装する要素です。 クレームとクレーム変換を定義したので、定義を実行するための技術プロファイルが必要です。 技術プロファイルは、ClaimsProvider 要素内で宣言されます。
Azure AD B2C には、一連の技術プロファイルが用意されています。 各技術プロファイルは、特定の役割を実行します。 たとえば、REST 技術プロファイルを使用して、サービス エンドポイントへの HTTP 呼び出しを行います。 クレーム変換技術プロファイルを使用して、要求変換で定義した操作を実行できます。 Azure AD B2C カスタム ポリシーによって提供される技術プロファイルの種類について確認してください。

### クレームの値を設定する
 
objectId、displayName、message の各クレームの値を設定するには、GenerateRandomObjectIdTransformation、CreateDisplayNameTransformation、および CreateMessageTransformation クレーム変換を実行する技術プロファイルを構成します。 クレーム変換は、OutputClaimsTransformations 要素で定義されている順序で実行されます。 たとえば、最初に表示名を作成し、次がメッセージです。
1. 次の ClaimsProvider を ClaimsProviders セクションの子として追加します。 
    
    ```xml
        <ClaimsProvider>
        
            <DisplayName>Technical Profiles to generate claims</DisplayName>
        </ClaimsProvider>
    
    ``` 
1.objectId、displayName、message の各クレームの値を設定するために、先ほど作成した ClaimsProvider 要素内に次のコードを追加します。
   
    ```xml
        <!--<ClaimsProvider>-->
            <TechnicalProfiles>
                <TechnicalProfile Id="ClaimGenerator">
                    <DisplayName>Generate Object ID, displayName and message Claims Technical Profile.</DisplayName>
                    <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.ClaimsTransformationProtocolProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"/>
                    <OutputClaims>
                        <OutputClaim ClaimTypeReferenceId="objectId"/>
                        <OutputClaim ClaimTypeReferenceId="displayName"/>
                        <OutputClaim ClaimTypeReferenceId="message"/>
                    </OutputClaims>
                    <OutputClaimsTransformations>
                        <OutputClaimsTransformation ReferenceId="GenerateRandomObjectIdTransformation"/>
                        <OutputClaimsTransformation ReferenceId="CreateDisplayNameTransformation"/>
                        <OutputClaimsTransformation ReferenceId="CreateMessageTransformation"/>
                    </OutputClaimsTransformations>
                </TechnicalProfile>
            </TechnicalProfiles>
        <!--</ClaimsProvider>-->
    ``` 

### ユーザー入力を収集する

displayName クレームは givenName と surname から生成するため、ユーザー入力として収集する必要があります。 ユーザー入力を収集するには、セルフアサートと呼ばれる種類の技術プロファイルを使用します。 セルフアサート技術プロファイルを構成する場合は、セルフアサート技術プロファイルによりユーザー インターフェイスが表示されるため、コンテンツ定義を参照する必要があります。

1. 次の ClaimsProvider を ClaimsProviders セクションの子として追加します。
    
    ```xml
        <ClaimsProvider>
        
            <DisplayName>Technical Profiles to collect user's details </DisplayName>
        </ClaimsProvider>
    ```

1. 作成した ClaimsProvider 要素内に次のコードを追加します。

    ```xml
        <TechnicalProfiles>
            <TechnicalProfile Id="UserInformationCollector">
                <DisplayName>Collect User Input Technical Profile</DisplayName>
                <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.SelfAssertedAttributeProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"/>
                <Metadata>
                    <Item Key="ContentDefinitionReferenceId">SelfAssertedContentDefinition</Item>
                </Metadata>
                <DisplayClaims>
                    <DisplayClaim ClaimTypeReferenceId="givenName" Required="true"/>
                    <DisplayClaim ClaimTypeReferenceId="surname" Required="true"/>
                </DisplayClaims>
                <OutputClaims>
                    <OutputClaim ClaimTypeReferenceId="givenName"/>
                    <OutputClaim ClaimTypeReferenceId="surname"/>
                </OutputClaims>
            </TechnicalProfile>
        </TechnicalProfiles>
    ```

    givenName と surname の各クレーム用の 2 つの表示クレームに注目してください。 どちらのクレームも必須としてマークされているため、ユーザーは表示されたフォームを送信する前に値を入力する必要があります。 クレームは、DisplayClaims 要素で定義されている順序で画面に表示されます (例: 名の次に姓)。

## Step 5 - User Journeys を定義する

`User Journeys`を使用して、技術プロファイルを呼び出す順序を定義します。 `OrchestrationSteps` 要素を使用して、ユーザー体験の手順を指定します。

`HelloWorldJourney` ユーザー体験の既存の内容を次のコードに置き換えます。 

```xml
    <OrchestrationSteps>
        <OrchestrationStep Order="1" Type="ClaimsExchange">
            <ClaimsExchanges>
                <ClaimsExchange Id="GetUserInformationClaimsExchange" TechnicalProfileReferenceId="UserInformationCollector"/>
            </ClaimsExchanges>
        </OrchestrationStep>
        <OrchestrationStep Order="2" Type="ClaimsExchange">
            <ClaimsExchanges>
                <ClaimsExchange Id="GetMessageClaimsExchange" TechnicalProfileReferenceId="ClaimGenerator"/>
            </ClaimsExchanges>
        </OrchestrationStep>
        <OrchestrationStep Order="3" Type="SendClaims" CpimIssuerTechnicalProfileReferenceId="JwtIssuer"/>
    </OrchestrationSteps>
```

オーケストレーションの手順に従って、ユーザー入力を収集し、objectId、displayName、message の各クレームの値を設定し、最後に Jwt トークンを送信します。

## Step 6 - relying party を更新する 

`RelyingParty` セクションの `OutputClaims` 要素の内容を次のコードに置き換えます。

```xml
    <OutputClaim ClaimTypeReferenceId="objectId" PartnerClaimType="sub"/>
    <OutputClaim ClaimTypeReferenceId="displayName"/>
    <OutputClaim ClaimTypeReferenceId="message"/>
```

Step 6 を完了すると、`ContosoCustomPolicy.XML` ファイルは次のコードのようになります。

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<TrustFrameworkPolicy xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
    xmlns:xsd="http://www.w3.org/2001/XMLSchema" 
    xmlns="http://schemas.microsoft.com/online/cpim/schemas/2013/06" 
    PolicySchemaVersion="0.3.0.0" TenantId="yourtenant.onmicrosoft.com" 
    PolicyId="B2C_1A_ContosoCustomPolicy" 
    PublicPolicyUri="http://yourtenant.onmicrosoft.com/B2C_1A_ContosoCustomPolicy">
    
    <BuildingBlocks>
        <ClaimsSchema>
            <ClaimType Id="objectId">
                <DisplayName>unique object Id for subject of the claims being returned</DisplayName>
                <DataType>string</DataType>
            </ClaimType>
            <ClaimType Id="message">
                <DisplayName>Will hold Hello World message</DisplayName>
                <DataType>string</DataType>
            </ClaimType>

            <ClaimType Id="givenName">
                <DisplayName>Given Name</DisplayName>
                <DataType>string</DataType>
                <UserHelpText>Your given name (also known as first name).</UserHelpText>
                <UserInputType>TextBox</UserInputType>
            </ClaimType>
            <ClaimType Id="surname">
                <DisplayName>Surname</DisplayName>
                <DataType>string</DataType>
                <UserHelpText>Your surname (also known as family name or last name).</UserHelpText>
                <UserInputType>TextBox</UserInputType>
            </ClaimType>
            <ClaimType Id="displayName">
                <DisplayName>Display Name</DisplayName>
                <DataType>string</DataType>
                <UserHelpText>Your display name.</UserHelpText>
                <UserInputType>TextBox</UserInputType>
            </ClaimType>
        </ClaimsSchema>
        <ClaimsTransformations>
            <ClaimsTransformation Id="GenerateRandomObjectIdTransformation" TransformationMethod="CreateRandomString">
                <InputParameters>
                    <InputParameter Id="randomGeneratorType" DataType="string" Value="GUID"/>
                </InputParameters>
                <OutputClaims>
                    <OutputClaim ClaimTypeReferenceId="objectId" TransformationClaimType="outputClaim"/>
                </OutputClaims>
            </ClaimsTransformation>

            <ClaimsTransformation Id="CreateDisplayNameTransformation" TransformationMethod="FormatStringMultipleClaims">
                <InputClaims>
                    <InputClaim ClaimTypeReferenceId="givenName" TransformationClaimType="inputClaim1"/>
                    <InputClaim ClaimTypeReferenceId="surname" TransformationClaimType="inputClaim2"/>
                </InputClaims>
                <InputParameters>
                    <InputParameter Id="stringFormat" DataType="string" Value="{0} {1}"/>
                </InputParameters>
                <OutputClaims>
                    <OutputClaim ClaimTypeReferenceId="displayName" TransformationClaimType="outputClaim"/>
                </OutputClaims>
            </ClaimsTransformation>

            <ClaimsTransformation Id="CreateMessageTransformation" TransformationMethod="FormatStringClaim">
                <InputClaims>
                    <InputClaim ClaimTypeReferenceId="displayName" TransformationClaimType="inputClaim"/>
                </InputClaims>
                <InputParameters>
                    <InputParameter Id="stringFormat" DataType="string" Value="Hello {0}"/>
                </InputParameters>
                <OutputClaims>
                    <OutputClaim ClaimTypeReferenceId="message" TransformationClaimType="outputClaim"/>
                </OutputClaims>
            </ClaimsTransformation> 
        </ClaimsTransformations>
        <ContentDefinitions>
            <ContentDefinition Id="SelfAssertedContentDefinition">
                <LoadUri>~/tenant/templates/AzureBlue/selfAsserted.cshtml</LoadUri>
                <RecoveryUri>~/common/default_page_error.html</RecoveryUri>
                <DataUri>urn:com:microsoft:aad:b2c:elements:contract:selfasserted:2.1.7</DataUri>
            </ContentDefinition>
        </ContentDefinitions>
    </BuildingBlocks>
    <!--Claims Providers Here-->
    <ClaimsProviders>
        <ClaimsProvider>
            <DisplayName>Token Issuer</DisplayName>
            <TechnicalProfiles>
                <TechnicalProfile Id="JwtIssuer">
                    <DisplayName>JWT Issuer</DisplayName>
                    <Protocol Name="None"/>
                    <OutputTokenFormat>JWT</OutputTokenFormat>
                    <Metadata>
                        <Item Key="client_id">{service:te}</Item>
                        <Item Key="issuer_refresh_token_user_identity_claim_type">objectId</Item>
                        <Item Key="SendTokenResponseBodyWithJsonNumbers">true</Item>
                    </Metadata>
                    <CryptographicKeys>
                        <Key Id="issuer_secret" StorageReferenceId="B2C_1A_TokenSigningKeyContainer"/>
                        <Key Id="issuer_refresh_token_key" StorageReferenceId="B2C_1A_TokenEncryptionKeyContainer"/>
                    </CryptographicKeys>
                </TechnicalProfile>
            </TechnicalProfiles>
        </ClaimsProvider>

        <ClaimsProvider>
            <DisplayName>Trustframework Policy Engine TechnicalProfiles</DisplayName>
            <TechnicalProfiles>
                <TechnicalProfile Id="TpEngine_c3bd4fe2-1775-4013-b91d-35f16d377d13">
                    <DisplayName>Trustframework Policy Engine Default Technical Profile</DisplayName>
                    <Protocol Name="None"/>
                    <Metadata>
                        <Item Key="url">{service:te}</Item>
                    </Metadata>
                </TechnicalProfile>
            </TechnicalProfiles>
        </ClaimsProvider>

        <ClaimsProvider>
            <DisplayName>Claim Generator Technical Profiles</DisplayName>
            <TechnicalProfiles>
                <TechnicalProfile Id="ClaimGenerator">
                    <DisplayName>Generate Object ID, displayName and  message Claims Technical Profile.</DisplayName>
                    <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.ClaimsTransformationProtocolProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"/>
                    <OutputClaims>
                        <OutputClaim ClaimTypeReferenceId="objectId"/>
                        <OutputClaim ClaimTypeReferenceId="displayName"/>
                        <OutputClaim ClaimTypeReferenceId="message"/>
                    </OutputClaims>
                    <OutputClaimsTransformations>
                        <OutputClaimsTransformation ReferenceId="GenerateRandomObjectIdTransformation"/>
                        <OutputClaimsTransformation ReferenceId="CreateDisplayNameTransformation"/>
                        <OutputClaimsTransformation ReferenceId="CreateMessageTransformation"/>
                    </OutputClaimsTransformations>
                </TechnicalProfile>
            </TechnicalProfiles>            
        </ClaimsProvider>

        <ClaimsProvider>
            <DisplayName>Technical Profiles to collect user's details</DisplayName>
            <TechnicalProfiles>
                <TechnicalProfile Id="UserInformationCollector">
                    <DisplayName>Collect User Input Technical Profile</DisplayName>
                    <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.SelfAssertedAttributeProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"/>
                    <Metadata>
                        <Item Key="ContentDefinitionReferenceId">SelfAssertedContentDefinition</Item>
                    </Metadata>
                    <DisplayClaims>
                        <DisplayClaim ClaimTypeReferenceId="givenName" Required="true"/>
                        <DisplayClaim ClaimTypeReferenceId="surname" Required="true"/>
                    </DisplayClaims>
                    <OutputClaims>
                        <OutputClaim ClaimTypeReferenceId="givenName"/>
                        <OutputClaim ClaimTypeReferenceId="surname"/>
                    </OutputClaims>
                </TechnicalProfile>
            </TechnicalProfiles>
        </ClaimsProvider>
    </ClaimsProviders>

    <UserJourneys>
        <UserJourney Id="HelloWorldJourney">
            <OrchestrationSteps>
                <OrchestrationStep Order="1" Type="ClaimsExchange">
                    <ClaimsExchanges>
                        <ClaimsExchange Id="GetUserInformationClaimsExchange" TechnicalProfileReferenceId="UserInformationCollector"/>
                    </ClaimsExchanges>
                </OrchestrationStep>
                <OrchestrationStep Order="2" Type="ClaimsExchange">
                    <ClaimsExchanges>
                        <ClaimsExchange Id="GetMessageClaimsExchange" TechnicalProfileReferenceId="ClaimGenerator"/>
                    </ClaimsExchanges>
                </OrchestrationStep>
                <OrchestrationStep Order="3" Type="SendClaims" CpimIssuerTechnicalProfileReferenceId="JwtIssuer"/>
            </OrchestrationSteps>
        </UserJourney>
    </UserJourneys>

    <RelyingParty><!-- 
            Relying Party Here that's your policy’s entry point
            Specify the User Journey to execute 
            Specify the claims to include in the token that is returned when the policy runs
        -->
        <DefaultUserJourney ReferenceId="HelloWorldJourney"/>
        <TechnicalProfile Id="HelloWorldPolicyProfile">
            <DisplayName>Hello World Policy Profile</DisplayName>
            <Protocol Name="OpenIdConnect"/>
            <OutputClaims>
                <OutputClaim ClaimTypeReferenceId="objectId" PartnerClaimType="sub"/>
                <OutputClaim ClaimTypeReferenceId="displayName"/>
                <OutputClaim ClaimTypeReferenceId="message"/>
            </OutputClaims>
            <SubjectNamingInfo ClaimType="sub"/>
        </TechnicalProfile>
    </RelyingParty>
</TrustFrameworkPolicy>
```

まだ行っていない場合は、yourtenant をテナント名のサブドメイン部分 (contoso など) に置き換えます。 テナント名を取得する方法について説明します。

## Step 3 - カスタム ポリシー ファイルをアップロードする

[「カスタム ポリシー ファイルをアップロードする」](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/custom-policies-series-hello-world#step-3---upload-custom-policy-file)の手順に従います。 ポータルに既にあるファイルと同じ名前のファイルをアップロードする場合は、[カスタム ポリシーが既に存在する場合は上書きします] を選択してください。


## Step 4 - カスタム ポリシーをテストする

1. [カスタム ポリシー] で、[B2C_1A_CONTOSOCUSTOMPOLICY] を選択します。

1. カスタム ポリシーの概要ページの [アプリケーションの選択] で、以前に登録した webapp1 などの Web アプリケーションを選択します。 [応答 URL の選択] の値が https://jwt.ms に設定されていることを確認します。

1. [今すぐ実行] ボタンを選択します。

1. 名と姓を入力し、[Continue] (続行) を選びます。

![ScreenShot](/media/screenshot-of-accepting-user-inputs-in-custom-policy.png)" 

ポリシーの実行が完了すると、https://jwt.ms にリダイレクトされ、デコードされた JWT トークンが表示されます。 次の JWT トークン スニペットのようになります。

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
      ...
      "name": "Maurice Paulet",
      "message": "Hello Maurice Paulet"
    }.[Signature]
``` 

完全なコードは [policy/ContosoCustomPolicy02.xml](/policy/ContosoCustomPolicy02.xml)
