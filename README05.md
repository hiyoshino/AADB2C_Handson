
# Azure Active Directory B2C カスタム ポリシーを使用してユーザー アカウントを作成して読み取る

Azure Active Directory B2C (Azure AD B2C) は Azure Active Directory (Azure AD) 上に構築されているため、Azure AD ストレージを使用してユーザー アカウントを格納します。 Azure AD B2C ディレクトリのユーザー プロファイルには、名、姓、市区町村、郵便番号、電話番号などの組み込みの属性セットが付属していますが、外部データ ストアを必要とすることなく、独自のカスタム属性を使用してユーザー プロファイルを拡張できます。

カスタム ポリシーは、ユーザー情報を格納、更新、または削除する Azure AD 技術プロファイルを使用して、Azure AD ストレージに接続できます。 この記事では、JWT トークンが返される前にユーザー アカウントを格納して読み取る一連の Azure AD 技術プロファイルを構成する方法について説明します。

## シナリオの概要

今まで作成してきたポリシーでは、最後にユーザー アカウントを格納せずに JWT を返しました。 ポリシーの実行が完了した後に情報が失われないように、ユーザー情報を格納する必要があります。 今回は、ユーザー情報を収集して検証したら、JWT トークンを返す前にユーザー情報を Azure AD B2C ストレージに格納してから、読み取る必要があります。 次の図はそのプロセス全体を示したものです。


## Step 1 - 要求を宣言する 

さらに 2 つの要求 (`userPrincipalName` と `passwordPolicies`) を宣言する必要があります。
  
1. `ContosoCustomPolicy.XML`ファイルで *ClaimsSchema* 要素を見つけ、次のコードを使用して `userPrincipalName` と `passwordPolicies` 要求を宣言します。

    ```xml
        <ClaimType Id="userPrincipalName">
            <DisplayName>UserPrincipalName</DisplayName>
            <DataType>string</DataType>
            <UserHelpText>Your user name as stored in the Azure Active Directory.</UserHelpText>
        </ClaimType>
        <ClaimType Id="passwordPolicies">
            <DisplayName>Password Policies</DisplayName>
            <DataType>string</DataType>
            <UserHelpText>Password policies used by Azure AD to determine password strength, expiry etc.</UserHelpText>
        </ClaimType>
    ```
    
    `userPrincipalName` と `passwordPolicies` 要求の使用方法の詳細については、[「ユーザー プロファイルの属性」](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/user-profile-attributes)の記事を参照してください。

## Step 2 -  Azure AD 技術プロファイルを作成する

2 つの [Azure AD 技術プロファイル](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/active-directory-technical-profile)を構成する必要があります。 一方の技術プロファイルでは Azure AD ストレージにユーザーの詳細を書き込み、もう一方では Azure AD ストレージからユーザー アカウントを読み取ります。

1. `ContosoCustomPolicy.XML` ファイルで *ClaimsProviders* 要素を見つけ、次のコードを使用して新しい要求プロバイダーを追加します。 この要求プロバイダーは、Azure AD 技術プロファイルを保持します。

    ```xml
        <ClaimsProvider>
            <DisplayName>Azure AD Technical Profiles</DisplayName>
            <TechnicalProfiles>
                <!--You'll add you Azure AD Technical Profiles here-->
            </TechnicalProfiles>
        </ClaimsProvider>
    ``` 
1. 作成したばかりの要求プロバイダーで、次のコードを使用して Azure AD 技術プロファイルを追加します。

    ```xml
        <TechnicalProfile Id="AAD-UserWrite">
            <DisplayName>Write user information to AAD</DisplayName>
            <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.AzureActiveDirectoryProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
            <Metadata>
                <Item Key="Operation">Write</Item>
                <Item Key="RaiseErrorIfClaimsPrincipalAlreadyExists">true</Item>
                <Item Key="UserMessageIfClaimsPrincipalAlreadyExists">The account already exists. Try to create another account</Item>
            </Metadata>
            <InputClaims>
                <InputClaim ClaimTypeReferenceId="email" PartnerClaimType="signInNames.emailAddress" Required="true" />
            </InputClaims>
            <PersistedClaims>
                <PersistedClaim ClaimTypeReferenceId="email" PartnerClaimType="signInNames.emailAddress" />        
                <PersistedClaim ClaimTypeReferenceId="displayName" />
                <PersistedClaim ClaimTypeReferenceId="givenName" />
                <PersistedClaim ClaimTypeReferenceId="surname" />
                <PersistedClaim ClaimTypeReferenceId="password"/>
                <PersistedClaim ClaimTypeReferenceId="passwordPolicies" DefaultValue="DisablePasswordExpiration,DisableStrongPassword" />
            </PersistedClaims>
            <OutputClaims>
                <OutputClaim ClaimTypeReferenceId="objectId" />
                <OutputClaim ClaimTypeReferenceId="userPrincipalName" />
                <OutputClaim ClaimTypeReferenceId="email" PartnerClaimType="signInNames.emailAddress" />
            </OutputClaims>
        </TechnicalProfile>
    ```

    新しい Azure AD 技術プロファイル *AAD-UserWrite* が追加されました。 この技術プロファイルの次の重要な部分に注目する必要があります。
    
    - *Operation*: 操作は、実行するアクション (この場合は Write) を指定します。 [Azure AD 技術プロバイダーのその他の操作](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/active-directory-technical-profile#azure-ad-technical-profile-operations)の詳細を確認してください。

    - *Persisted claims*: *PersistedClaims* 要素には、Azure AD ストレージに格納する必要があるすべての値が含まれています。
    
    - *InputClaims*: The *InputClaims* 要素には、ディレクトリ内のアカウントを検索したり、新しいものを作成したりするために使用される要求が含まれています。 すべての Azure AD 技術プロファイルには、入力要求要素が入力要求コレクションに 1 つだけ存在する必要があります。 この技術プロファイルでは、ユーザー アカウントのキー識別子として email 要求が使用されます。 [ユーザー アカウントを一意に識別するために使用できるその他のキー識別子](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/active-directory-technical-profile#inputclaims)の詳細について確認してください。
    

1. `ContosoCustomPolicy.XML`ファイルで `AAD-UserWrite` 技術プロファイルを見つけ、次のコードを使用してその後に新しい技術プロファイルを追加します。

    ```xml
        <TechnicalProfile Id="AAD-UserRead">
            <DisplayName>Read user from Azure AD storage</DisplayName>
            <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.AzureActiveDirectoryProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
            <Metadata>
                <Item Key="Operation">Read</Item>
                <Item Key="RaiseErrorIfClaimsPrincipalAlreadyExists">false</Item>
                <Item Key="RaiseErrorIfClaimsPrincipalDoesNotExist">false</Item>
            </Metadata>
            <InputClaims>
                <InputClaim ClaimTypeReferenceId="email" PartnerClaimType="signInNames.emailAddress" Required="true" />
            </InputClaims>
            <OutputClaims>
                <OutputClaim ClaimTypeReferenceId="objectId" />
                <OutputClaim ClaimTypeReferenceId="userPrincipalName" />
                <OutputClaim ClaimTypeReferenceId="givenName"/>
                <OutputClaim ClaimTypeReferenceId="surname"/>
                <OutputClaim ClaimTypeReferenceId="displayName"/>
            </OutputClaims>
        </TechnicalProfile>
    ```

   新しい Azure AD 技術プロファイル `AAD-UserRead` が追加されました。 この技術プロファイルは、読み取り操作を実行し、`InputClaim` セクションに `email` を持つユーザー アカウントが見つかった場合に `objectId`、`userPrincipalName`、`givenName`、`surname` および `displayName` 要求を返すように構成されました。

## Step 3 - Azure AD 技術プロファイルを使用する

`UserInformationCollector` セルフアサート技術プロファイルを使用してユーザーの詳細を収集した後、`AAD-UserWrite` 技術プロファイルを使用して Azure AD ストレージにユーザー アカウントを書き込む必要があります。 これを行うには、`UserInformationCollector` セルフアサート技術プロファイルの検証技術プロファイルとして `AAD-UserWrite` 技術プロファイルを使用します。

`ContosoCustomPolicy.XML` ファイルで `UserInformationCollector` 技術プロファイルを見つけ、`AAD-UserWrite` 技術プロファイルを検証技術プロファイルとして `ValidationTechnicalProfiles` コレクションに追加します。 これは、`CheckCompanyDomain` 検証技術プロファイルの後に追加する必要があります。 

ユーザー体験のオーケストレーション手順で `AAD-UserRead` 技術プロファイルを使用して、JWT トークンを発行する前にユーザーの詳細を読み取ります。

## Step 4 - ClaimGenerator 技術プロファイルを更新する 

`ClaimGenerator` 技術プロファイルを使用して、`GenerateRandomObjectIdTransformation`、`CreateDisplayNameTransformation`、`CreateMessageTransformation` の 3 つの要求変換を実行します。

1. `ContosoCustomPolicy.XML` ファイルで `ClaimGenerator` 技術プロファイルを見つけ、次のコードに置き換えます。

    ```xml
        <TechnicalProfile Id="UserInputMessageClaimGenerator">
            <DisplayName>User Message Claim Generator Technical Profile</DisplayName>
            <Protocol Name="Proprietary"
                Handler="Web.TPEngine.Providers.ClaimsTransformationProtocolProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
            <OutputClaims>
                <OutputClaim ClaimTypeReferenceId="message" />
            </OutputClaims>
            <OutputClaimsTransformations>
                <OutputClaimsTransformation ReferenceId="CreateMessageTransformation" />
            </OutputClaimsTransformations>
        </TechnicalProfile>
    
        <TechnicalProfile Id="UserInputDisplayNameGenerator">
            <DisplayName>Display Name Claim Generator Technical Profile</DisplayName>
            <Protocol Name="Proprietary"
                Handler="Web.TPEngine.Providers.ClaimsTransformationProtocolProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
            <OutputClaims>
                <OutputClaim ClaimTypeReferenceId="displayName" />
            </OutputClaims>
            <OutputClaimsTransformations>
                <OutputClaimsTransformation ReferenceId="CreateDisplayNameTransformation" />
            </OutputClaimsTransformations>
        </TechnicalProfile>
    ```
    技術プロファイルが 2 つの別々の技術プロファイルに分割されました。 `UserInputMessageClaimGenerator` 技術プロファイルは、JWT トークンで要求として送信されるメッセージを生成します。 `UserInputDisplayNameGenerator` 技術プロファイルは、`displayName` 要求を生成します。 `AAD-UserWrite` 技術プロファイルがユーザー レコードを Azure AD ストレージに書き込む前に、`displayName` 要求値が使用可能でなければなりません。 新しいコードで、アカウントの作成後に `objectId` が作成され、Azure AD によって返されるので `GenerateRandomObjectIdTransformation` を削除します。そのため、ポリシー内で自分自身で生成する必要はありません。


1. `ContosoCustomPolicy.XML` ファイルで `UserInformationCollector` セルフアサート技術プロファイルを見つけ、`UserInputDisplayNameGenerator` 技術プロファイルを検証技術プロファイルとして追加します。 その後、`UserInformationCollector` 技術プロファイルの `ValidationTechnicalProfiles` コレクションは次のコードのようになります。

    ```xml
        <!--<TechnicalProfile Id="UserInformationCollector">-->
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
                <ValidationTechnicalProfile ReferenceId="UserInputDisplayNameGenerator"/>
                <ValidationTechnicalProfile ReferenceId="AAD-UserWrite"/>
            </ValidationTechnicalProfiles>
        <!--</TechnicalProfile>-->
    ```   

    `AAD-UserWrite` 技術プロファイルがユーザー レコードを Azure AD ストレージに書き込む前に、`displayName` 要求値が使用可能でなければならないので、`AAD-UserWrite` の前に検証技術プロファイルを追加する必要があります。

## Step 5 - ユーザー体験のオーケストレーション手順を更新する 

`HelloWorldJourney` ユーザー体験を見つけ、すべてのオーケストレーション手順を次のコードに置き換えます。

```xml
    <!--<OrchestrationSteps>-->
        <OrchestrationStep Order="1" Type="ClaimsExchange">
            <ClaimsExchanges>
                <ClaimsExchange Id="AccountTypeInputCollectorClaimsExchange" TechnicalProfileReferenceId="AccountTypeInputCollector"/>
            </ClaimsExchanges>
        </OrchestrationStep>
        <OrchestrationStep Order="2" Type="ClaimsExchange">
            <ClaimsExchanges>
                <ClaimsExchange Id="GetAccessCodeClaimsExchange" TechnicalProfileReferenceId="AccessCodeInputCollector" />
            </ClaimsExchanges>
            </OrchestrationStep>
        <OrchestrationStep Order="3" Type="ClaimsExchange">
            <ClaimsExchanges>
                <ClaimsExchange Id="GetUserInformationClaimsExchange" TechnicalProfileReferenceId="UserInformationCollector"/>
            </ClaimsExchanges>
        </OrchestrationStep>
        <OrchestrationStep Order="4" Type="ClaimsExchange">
            <ClaimsExchanges>
                <ClaimsExchange Id="AADUserReaderExchange" TechnicalProfileReferenceId="AAD-UserRead"/>
            </ClaimsExchanges>
        </OrchestrationStep>
        <OrchestrationStep Order="5" Type="ClaimsExchange">
            <ClaimsExchanges>
                <ClaimsExchange Id="GetMessageClaimsExchange" TechnicalProfileReferenceId="UserInputMessageClaimGenerator"/>
            </ClaimsExchanges>
        </OrchestrationStep>
        <OrchestrationStep Order="6" Type="SendClaims" CpimIssuerTechnicalProfileReferenceId="JwtIssuer"/>
    <!--</OrchestrationSteps>-->
```

オーケストレーション手順 4 で、`AAD-UserRead` 技術プロファイルを実行して、作成されたユーザー アカウントからユーザーの詳細 (JWT トークンに含まれる) を読み取ります。

`message` 要求を格納しないため、オーケストレーション手順 5 で `UserInputMessageClaimGenerator` を実行して、JWT トークンに含める `message` 要求を生成します。
 
## Step 6 - ポリシーをアップロードする 

[「カスタム ポリシー ファイルをアップロードする」](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/custom-policies-series-hello-world#step-3---upload-custom-policy-file)の手順に従います。 ポータルに既にあるファイルと同じ名前のファイルをアップロードする場合は、[カスタム ポリシーが既に存在する場合は上書きします] を選択してください。

## Step 7 - ポリシーをテストする

[「カスタム ポリシーをテストする」](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/custom-policies-series-validate-user-input#step-6---test-the-custom-policy)の手順に従って、カスタム ポリシーをテストします。

ポリシーの実行が完了し、ID トークンを受け取ったら、ユーザー レコードが作成されていることを確認します。

1. グローバル管理者または特権ロール管理者のアクセス許可を使用して Azure portal にサインインします。

1. ご利用の Azure AD B2C テナントが含まれるディレクトリを必ず使用してください。
    
    1. ポータル ツールバーの [Directories + subscriptions](ディレクトリ + サブスクリプション) アイコンを選択します。

    1. [ポータルの設定] | [Directories + subscriptions] (ディレクトリ + サブスクリプション) ページの [ディレクトリ名] の一覧で自分の Azure AD B2C ディレクトリを見つけて、[切り替え] を選択します。

1. [Azure サービス] で、 [Azure AD B2C] を選択します。 または、検索ボックスを使用して検索し、 [Azure AD B2C] を選択します。

1. [管理] にある [ユーザー] を選択します。

1. 作成したばかりのユーザー アカウントを見つけて選択します。 アカウント プロファイルは、次のスクリーンショットのようになります。

     ![ScreenShot](/media/screenshot-of-create-users-custom-policy.png)


AAD-UserWrite Azure AD 技術プロファイルで、ユーザーが既に存在する場合は、エラー メッセージが表示されることを指定します。

同じ*電子メール* *アドレス*を使用して、カスタム ポリシーをもう一度テストします。 ポリシーが最後まで完了して ID トークンを発行するのではなく、次のスクリーンショットのようなエラー メッセージが表示されます
    ![ScreenShot](/media/screenshot-of-error-account-already-exists.png)

> [!NOTE]
> "パスワード" 要求の値は非常に重要な情報であるため、カスタム ポリシーでの取り扱い方法には十分注意してください。 同様の理由から、Azure AD B2C ではパスワード要求の値は特別な値として扱われます。 セルフアサート技術プロファイルでパスワード要求の値を収集すると、その値は同じ技術プロファイル内、または同じセルフアサート技術プロファイルによって参照される検証技術プロファイル内でのみ使用できます。 そのセルフアサート技術プロファイルの実行が完了し、別の技術プロファイルに移動すると、値は失われます。 

## ユーザーの電子メール アドレスを検証する

ユーザー アカウントの作成に使用する前に、ユーザーの電子メールを検証することをお勧めします。 電子メール アドレスを検証するときは、アカウントが実際のユーザーによって作成されていることを確認します。 また、正しい電子メール アドレスを使用してアカウントを作成していることを確認することもユーザーに役立ちます。

Azure AD B2C のカスタム ポリシーでは、[検証表示コントロール](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/display-control-verification)を使用して電子メール アドレスを検証できます。 確認コードを電子メールに送信します。 コードが送信されたら、ユーザーはメッセージを読んで、表示コントロールによって提供されたコントロールに確認コードを入力し、[コードの確認] ボタンを選択します。

表示コントロールは、特別な機能を備え、Azure Active Directory B2C (Azure AD B2C) バックエンド サービスと作用するユーザー インターフェイス要素です。 これにより、バックエンドで検証技術プロファイルを呼び出すページでユーザーが操作を実行できます。 ページに表示コントロールが表示され、セルフアサート技術プロファイルによって参照されます。

表示コントロールを使用して電子メールの検証を追加するには、次の手順に従います。

### 要求を宣言する

確認コードの保持に使用する要求を宣言する必要があります。

要求を宣言するには、`ContosoCustomPolicy.XML` ファイルで `ClaimsSchema` 要素を見つけ、次のコードを使用して `verificationCode` 要求を宣言します。  

```xml
    <!--<ClaimsSchema>-->
    
        <ClaimType Id="verificationCode">
            <DisplayName>Verification Code</DisplayName>
            <DataType>string</DataType>
            <UserHelpText>Enter your verification code</UserHelpText>
            <UserInputType>TextBox</UserInputType>
        </ClaimType>
    <!--</ClaimsSchema>-->
```

### コードの送信と検証技術プロファイルを構成する

Azure AD B2C では、[Azure AD SSPR 技術プロファイル](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/aad-sspr-technical-profile)を使用して電子メール アドレスを検証します。 この技術プロファイルでは、構成内容に応じて、コードを生成して電子メール アドレスに送信するか、コードを検証できます。

`ContosoCustomPolicy.XML` ファイルで `ClaimsProviders` 要素を見つけ、次のコードを使用して要求プロバイダーを追加します。

```xml
    <ClaimsProvider>
        <DisplayName>Azure AD self-service password reset (SSPR)</DisplayName>
        <TechnicalProfiles>
            <TechnicalProfile Id="AadSspr-SendCode">
            <DisplayName>Send Code</DisplayName>
            <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.AadSsprProtocolProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
            <Metadata>
                <Item Key="Operation">SendCode</Item>
            </Metadata>
            <InputClaims>
                <InputClaim ClaimTypeReferenceId="email" PartnerClaimType="emailAddress" />
            </InputClaims>
            </TechnicalProfile>
            <TechnicalProfile Id="AadSspr-VerifyCode">
            <DisplayName>Verify Code</DisplayName>
            <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.AadSsprProtocolProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
            <Metadata>
                <Item Key="Operation">VerifyCode</Item>
            </Metadata>
            <InputClaims>
                <InputClaim ClaimTypeReferenceId="verificationCode" />
                <InputClaim ClaimTypeReferenceId="email" PartnerClaimType="emailAddress" />
            </InputClaims>
            </TechnicalProfile>
        </TechnicalProfiles>
    </ClaimsProvider>
```

2 つの技術プロファイル `AadSspr-SendCode` と `AadSspr-VerifyCode` が構成されました。 `AadSspr-SendCode` は、コードを生成して、`InputClaims` セクションで指定された電子メール アドレスに送信します。一方、`AadSspr-VerifyCode` はコードを検証します。 技術プロファイルのメタデータで実行するアクションを指定します。

### 表示コントロールを構成する

ユーザーの電子メールを検証できるように、電子メールの検証表示コントロールを構成する必要があります。 構成した電子メールの検証表示コントロールは、ユーザーから電子メールを収集するために使用する電子メール表示要求に置き換わります。

表示コントロールを構成するには、次の手順に従います。

1. `ContosoCustomPolicy.XML` ファイルで `BuildingBlocks` セクションを見つけ、次のコードを使用して表示コントロールを子要素として追加します。

    ```xml
        <!--<BuildingBlocks>-->
        
            <DisplayControls>
                <DisplayControl Id="emailVerificationControl" UserInterfaceControlType="VerificationControl">
                  <DisplayClaims>
                    <DisplayClaim ClaimTypeReferenceId="email" Required="true" />
                    <DisplayClaim ClaimTypeReferenceId="verificationCode" ControlClaimType="VerificationCode" Required="true" />
                  </DisplayClaims>
                  <OutputClaims></OutputClaims>
                  <Actions>
                    <Action Id="SendCode">
                      <ValidationClaimsExchange>
                        <ValidationClaimsExchangeTechnicalProfile TechnicalProfileReferenceId="AadSspr-SendCode" />
                      </ValidationClaimsExchange>
                    </Action>
                    <Action Id="VerifyCode">
                      <ValidationClaimsExchange>
                        <ValidationClaimsExchangeTechnicalProfile TechnicalProfileReferenceId="AadSspr-VerifyCode" />
                      </ValidationClaimsExchange>
                    </Action>
                  </Actions>
                </DisplayControl>
            </DisplayControls> 
        <!--</BuildingBlocks>-->
    ``` 

  表示コントロール `emailVerificationControl` が宣言されました。 次の重要な部分に注目してください。

    - *DisplayClaims* - セルフアサート技術プロファイルと同様に、このセクションでは、表示コントロール内でユーザーから収集される要求のコレクションを指定します。
    
    - *Actions* - 表示コントロールによって実行されるアクションの順序を指定します。 各アクションは、アクションの実行を担当する技術プロファイルを参照します。 たとえば、*SendCode* は、コードを生成して電子メール アドレスに送信する `AadSspr-SendCode` 技術プロファイルを参照します。

1. `ContosoCustomPolicy.XML` ファイルで、`UserInformationCollector` セルフアサート技術プロファイルを見つけ、`email` 表示要求を `emailVerificationControl` 表示コントロールに置き換えます。 
    
    From: 
    
    ```xml
        <DisplayClaim ClaimTypeReferenceId="email" Required="true"/>
    ```

    To: 

    ```xml
        <DisplayClaim DisplayControlReferenceId="emailVerificationControl" />
    ```

1. Step 6 と Step 7 の手順を使用して、ポリシー ファイルをアップロードしてテストします。 ここで、ユーザー アカウントを作成する前に、お使いの電子メール アドレスを検証する必要があります。  

## Azure AD 技術プロファイルを使用してユーザー アカウントを更新する

新しいユーザー アカウントを作成するのではなく、ユーザー アカウントを更新するように Azure AD 技術プロファイルを構成できます。 これを行うには、次のコードを使用して、指定したユーザー アカウントがまだ `Metadata` コレクションに存在しない場合にエラーをスローするように、Azure AD 技術プロファイルを設定します。 *Operation* を *Write* に設定する必要があります。

```xml
    <!--<Item Key="Operation">Write</Item>-->
    <Item Key="RaiseErrorIfClaimsPrincipalDoesNotExist">true</Item>
``` 
 