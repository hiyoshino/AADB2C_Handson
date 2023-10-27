
# Azure Active Directory B2C カスタム ポリシーを使用してローカル アカウントのサインアップおよびサインイン フローを設定する

この記事では、ユーザーが Azure AD B2C ローカル アカウントを作成するか、サインインできるようにする Azure Active Directory B2C (Azure AD B2C) カスタム ポリシーを記述する方法について説明します。 ローカル アカウントとは、ユーザーがアプリケーションにサインアップしたときに、Azure AD B2C ディレクトリに作成されるアカウントを指します。

## 概要

Azure AD B2C では、OpenID Connect 認証プロトコルを使用してユーザーの資格情報が検証されます。 Azure AD B2C では、ユーザーの資格情報を他の情報と共にセキュリティで保護されたエンドポイントに送信し、そこで資格情報が有効かどうかが判断されます。 まとめると、Azure AD B2C による OpenID Connect の実装を利用すると、Web アプリケーションでのサインアップ、サインイン、その他の ID 管理を Azure Active Directory (Azure AD) に委託できます。

Azure AD B2C のカスタム ポリシーには、セキュリティで保護された Microsoft エンドポイントへの呼び出しに使用する [OpenID Connect 技術プロファイル](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/openid-connect-technical-profile)が用意されています。 OpenID Connect 技術プロファイルの詳細については、こちらを参照してください。

## Step 1 - OpenID Connect 技術プロファイルを構成する

OpenID Connect 技術プロファイルを構成するには、次の 3 つの手順を実行する必要があります。

- 追加のクレームを宣言します。
- Azure portal にアプリを登録します。
- 最後に、OpenID Connect 技術プロファイル自体を構成します

### Step 1.1 - 追加のクレームを宣言する

`ContosoCustomPolicy.XML` ファイルで *ClaimsSchema* セクションを見つけ、次のコードを使用してクレームをさらに追加します。

```xml
    <!--<ClaimsSchema>-->
        ...
        <ClaimType Id="grant_type">
            <DisplayName>grant_type</DisplayName>
            <DataType>string</DataType>
            <UserHelpText>Special parameter passed for local account authentication to login.microsoftonline.com.</UserHelpText>
        </ClaimType>
        
        <ClaimType Id="scope">
            <DisplayName>scope</DisplayName>
            <DataType>string</DataType>
            <UserHelpText>Special parameter passed for local account authentication to login.microsoftonline.com.</UserHelpText>
        </ClaimType>
        
        <ClaimType Id="nca">
            <DisplayName>nca</DisplayName>
            <DataType>string</DataType>
            <UserHelpText>Special parameter passed for local account authentication to login.microsoftonline.com.</UserHelpText>
        </ClaimType>
        
        <ClaimType Id="client_id">
            <DisplayName>client_id</DisplayName>
            <DataType>string</DataType>
            <AdminHelpText>Special parameter passed to EvoSTS.</AdminHelpText>
            <UserHelpText>Special parameter passed to EvoSTS.</UserHelpText>
        </ClaimType>
        
        <ClaimType Id="resource_id">
            <DisplayName>resource_id</DisplayName>
            <DataType>string</DataType>
            <AdminHelpText>Special parameter passed to EvoSTS.</AdminHelpText>
            <UserHelpText>Special parameter passed to EvoSTS.</UserHelpText>
        </ClaimType>
    <!--</ClaimsSchema>-->
```

### Step 1.2 -  Identity Experience Framework アプリケーションを登録する 

Azure AD B2C では、ローカル アカウントでのユーザーのサインアップとサインインのために使用する 2 つのアプリケーションを登録する必要があります。IdentityExperienceFramework (Web API) と、IdentityExperienceFramework アプリへのアクセス許可が委任された ProxyIdentityExperienceFramework (ネイティブ アプリ) です。

まだ行っていない場合は、次のアプリケーションを登録します。 

1. [「IdentityExperienceFramework](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/tutorial-create-user-flows?pivots=b2c-custom-policy#register-the-identityexperienceframework-application) アプリケーションを登録する」の手順を使用して、Identity Experience Framework アプリケーションを登録します。 次の手順で使用するために、Identity Experience Framework アプリケーション登録用のアプリケーション (クライアント) ID (appID) をコピーします。

1. [「ProxyIdentityExperienceFramework アプリケーションを登録する」](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/tutorial-create-user-flows?pivots=b2c-custom-policy#register-the-proxyidentityexperienceframework-application)の手順を使用して、プロキシ Identity Experience Framework アプリケーションを登録します。 次の手順で使用するために、プロキシ Identity Experience Framework アプリケーション登録用のアプリケーション (クライアント) ID (proxyAppID) をコピーします。

### Step 1.3 - OpenID Connect 技術プロファイルを構成する

`ContosoCustomPolicy.XML` ファイルで ClaimsProviders セクションを見つけ、次のコードを使用して、OpenID Connect 技術プロファイルを保持するクレーム プロバイダー要素を追加します。

```xml
    <!--<ClaimsProviders>-->
        ...
        <ClaimsProvider>
            <DisplayName>OpenID Connect Technical Profiles</DisplayName>
            <TechnicalProfiles>
                <TechnicalProfile Id="SignInUser">
                    <DisplayName>Sign in with Local Account</DisplayName>
                    <Protocol Name="OpenIdConnect" />
                    <Metadata>
                        <Item Key="UserMessageIfClaimsPrincipalDoesNotExist">We didn't find this account</Item>
                        <Item Key="UserMessageIfInvalidPassword">Your password or username is incorrect</Item>
                        <Item Key="UserMessageIfOldPasswordUsed">You've used an old password.</Item>
                        <Item Key="ProviderName">https://sts.windows.net/</Item>
                        <Item Key="METADATA">https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration</Item>
                        <Item Key="authorization_endpoint">https://login.microsoftonline.com/{tenant}/oauth2/token</Item>
                        <Item Key="response_types">id_token</Item>
                        <Item Key="response_mode">query</Item>
                        <Item Key="scope">email openid</Item>
                        <!-- Policy Engine Clients -->
                        <Item Key="UsePolicyInRedirectUri">false</Item>
                        <Item Key="HttpBinding">POST</Item>
                        <Item Key="client_id">proxyAppID</Item>
                        <Item Key="IdTokenAudience">appID</Item>
                    </Metadata>
                    <InputClaims>
                        <InputClaim ClaimTypeReferenceId="email" PartnerClaimType="username" Required="true" />
                        <InputClaim ClaimTypeReferenceId="password" PartnerClaimType="password" Required="true" />
                        <InputClaim ClaimTypeReferenceId="grant_type" DefaultValue="password" />
                        <InputClaim ClaimTypeReferenceId="scope" DefaultValue="openid" />
                        <InputClaim ClaimTypeReferenceId="nca" PartnerClaimType="nca" DefaultValue="1" />
                        <InputClaim ClaimTypeReferenceId="client_id" DefaultValue="proxyAppID" />
                        <InputClaim ClaimTypeReferenceId="resource_id" PartnerClaimType="resource" DefaultValue="appID" />
                    </InputClaims>
                    <OutputClaims>
                        <OutputClaim ClaimTypeReferenceId="objectId" PartnerClaimType="oid" />
                    </OutputClaims>
                </TechnicalProfile>
            </TechnicalProfiles>
        </ClaimsProvider>
    <!--</ClaimsProviders>-->
``` 

次の両方のインスタンスを置き換えます。

- appID を Step 1.2 でコピーした Identity Experience Framework アプリケーションのアプリケーション (クライアント) ID に。

- proxyAppID を Step 1.2 でコピーしたプロキシ Identity Experience Framework アプリケーションのアプリケーション (クライアント) ID に。

## Step 2 - サインイン ユーザー インターフェイスを構成する

ポリシーが実行されるときに、ユーザーがサインインするためのユーザー インターフェイスを表示する必要があります。 このユーザー インターフェイスには、まだアカウントを持っていないユーザーがサインアップするオプションもあります。 このために、以下の 2 つの手順を実行する必要があります。

- ユーザーにサインイン フォームを表示する、セルフアサート技術プロファイルを構成します。
- サインイン ユーザー インターフェイスのコンテンツ定義を構成します

### Step 2.1 - サインイン ユーザー インターフェイス技術プロファイルを構成する

`ContosoCustomPolicy.XML` ファイルで `SignInUser` 技術プロファイルを見つけ、次のコードを使用してその後に SelfAsserted 技術プロファイルを追加します。


```xml
    <TechnicalProfile Id="UserSignInCollector">
        <DisplayName>Local Account Signin</DisplayName>
        <Protocol Name="Proprietary"
            Handler="Web.TPEngine.Providers.SelfAssertedAttributeProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
        <Metadata>
            <Item Key="setting.operatingMode">Email</Item>
            <Item Key="SignUpTarget">AccountTypeInputCollectorClaimsExchange</Item>
        </Metadata>
        <DisplayClaims>
            <DisplayClaim ClaimTypeReferenceId="email" Required="true" />
            <DisplayClaim ClaimTypeReferenceId="password" Required="true" />
        </DisplayClaims>
        <OutputClaims>
            <OutputClaim ClaimTypeReferenceId="email" />
            <OutputClaim ClaimTypeReferenceId="password"  />
            <OutputClaim ClaimTypeReferenceId="objectId" />
        </OutputClaims>
        <ValidationTechnicalProfiles>
            <ValidationTechnicalProfile ReferenceId="SignInUser" />
        </ValidationTechnicalProfiles>
    </TechnicalProfile>
```

ユーザーにサインイン フォームを表示する SelfAsserted 技術プロファイル *UserSignInCollector* が追加されました。 `setting.operatingMode` メタデータに示されているように、ユーザーのメール アドレスをサインイン名として収集するように技術プロファイルを構成しました。 サインイン フォームにはサインアップ リンクが含まれており、ユーザーは `SignUpTarget` メタデータによって示されるサインアップ フォームに誘導されます。 `SignUpWithLogonEmailExchangeClaimsExchange` を設定する方法については、オーケストレーションの手順で説明します。

また、*SignInUser* OpenID Connect 技術プロファイルを `ValidationTechnicalProfile` として追加しました。 そのため、`SignInUser` 技術プロファイルは、ユーザーが [サインイン] ボタンを選択したときに実行されます (Step 5 のスクリーンショットを参照)。

次に会社のアカウントを選択した際のメールアドレスが正しく入力されているかをチェックする `preUserInformationCollector` 技術プロファイルを追加します。`CheckCompanyDomain` 技術プロファイルの直前に挿入します。

```xml
            <TechnicalProfile Id="preUserInformationCollector">
                <DisplayName>Collect User Input Technical Profile</DisplayName>
                <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.SelfAssertedAttributeProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"/>
                <Metadata>
                    <Item Key="ContentDefinitionReferenceId">SelfAssertedContentDefinition</Item>
                    <Item Key="LookupNotFound">The provided email address isn't a valid Contoso Employee email.</Item>
                </Metadata>
                <DisplayClaims>
                <!-- <DisplayClaim ClaimTypeReferenceId="accountType" Required="true"/> --> 
                    <DisplayClaim ClaimTypeReferenceId="email" Required="true"/>
                </DisplayClaims>
                <OutputClaims>
                <!-- <OutputClaim ClaimTypeReferenceId="accountType"/> --> 
                    <OutputClaim ClaimTypeReferenceId="email"/>
                </OutputClaims>
                
                <!--追加-->
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
            </TechnicalProfile>
```

次の手順 (Step 2.2) では、この SelfAsserted 技術プロファイルで使用するコンテンツ定義を構成します。 

### Step 2.2 - サインイン インターフェイスのコンテンツ定義を構成する

`ContosoCustomPolicy.XML`ァイルで *ContentDefinitions* セクションを見つけ、次のコードを使用してコンテンツ定義をサインインします。

```xml
    <!--<ContentDefinitions>-->

            <ContentDefinition Id="SignupOrSigninContentDefinition">
                <LoadUri>~/tenant/templates/AzureBlue/unified.cshtml</LoadUri>
                <RecoveryUri>~/common/default_page_error.html</RecoveryUri>
                <DataUri>urn:com:microsoft:aad:b2c:elements:contract:unifiedssp:2.1.7</DataUri>
                <Metadata>
                    <Item Key="DisplayName">Signin and Signup</Item>
                </Metadata>
            </ContentDefinition>
    <!--</ContentDefinitions>-->
``` 
セルフアサート技術プロファイル `SignupOrSigninContentDefinition` のコンテンツ定義を構成しました。 これは、メタデータ要素を使用して技術プロファイルで指定することも、オーケストレーション手順で技術プロファイルを参照するときに指定することもできます。 以前は、セルフアサート技術プロファイルでコンテンツ定義を直接指定する方法を学習しました。そのため、この記事では、オーケストレーション手順 (Step 3) で技術プロファイルを参照するときに指定する方法について説明します。


## Step 3 - ユーザー体験のオーケストレーション手順を更新する

`ContosoCustomPolicy.XML` ファイルで *HelloWorldJourney* ユーザー体験を見つけ、そのオーケストレーション ステップ コレクションをすべて次のコードに置き換えます。 

```xml
    <!--<OrchestrationSteps>-->
                <!-- 
                    オーケストレーション手順 1 - サインイン ページが表示されるため、ユーザーはサインインするか、[今すぐサインアップ] リンクを選択することができます。
                     UserSignInCollector セルフアサート技術プロファイルがサインイン フォームの表示に使用するコンテンツ定義を指定していることに注意してください。
                -->
                <OrchestrationStep Order="1" Type="CombinedSignInAndSignUp" ContentDefinitionReferenceId="SignupOrSigninContentDefinition">
                    <ClaimsProviderSelections>
                        <ClaimsProviderSelection ValidationClaimsExchangeId="LocalAccountSigninEmailExchange" />
                    </ClaimsProviderSelections>
                    <ClaimsExchanges>
                        <ClaimsExchange Id="LocalAccountSigninEmailExchange" TechnicalProfileReferenceId="UserSignInCollector" />
                    </ClaimsExchanges>
                </OrchestrationStep>
                <!-- 
                    オーケストレーション 手順 2 - この手順は、ユーザーがサインアップする (objectId が存在しない) 場合に実行されるため、
                    AccountTypeInputCollector セルフアサート技術プロファイルを呼び出して、アカウントの種類の選択フォームを表示します。
                --> 
                <OrchestrationStep Order="2" Type="ClaimsExchange">
                    <Preconditions>
                        <Precondition Type="ClaimsExist" ExecuteActionsIf="true">
                            <Value>objectId</Value>
                            <Action>SkipThisOrchestrationStep</Action>
                        </Precondition>
                    </Preconditions>
                    <ClaimsExchanges>
                        <ClaimsExchange Id="AccountTypeInputCollectorClaimsExchange" TechnicalProfileReferenceId="AccountTypeInputCollector"/>
                    </ClaimsExchanges>
                </OrchestrationStep>
                <!-- 
                    オーケストレーション 手順 3 - この手順は、ユーザーがサインアップし (objectId が存在しない)、ユーザーが会社の accountType を選択しない場合に
                    実行されます。 そのため、AccessCodeInputCollector セルフアサート技術プロファイルを呼び出して、accessCode を入力するようにユーザーに求めます。
                -->
                <OrchestrationStep Order="3" Type="ClaimsExchange">
                    <Preconditions>
                        <Precondition Type="ClaimsExist" ExecuteActionsIf="true">
                            <Value>objectId</Value>
                            <Action>SkipThisOrchestrationStep</Action>
                        </Precondition>
                        <Precondition Type="ClaimEquals" ExecuteActionsIf="true">
                        <Value>accountType</Value>
                        <Value>company</Value>
                        <Action>SkipThisOrchestrationStep</Action>
                        </Precondition>
                    </Preconditions>
                    <ClaimsExchanges>
                        <ClaimsExchange Id="GetAccessCodeClaimsExchange" TechnicalProfileReferenceId="AccessCodeInputCollector" />
                    </ClaimsExchanges>
                </OrchestrationStep>
                <!-- 企業を選択した際のEメールアドレス入力が会社のアドレスかをチェックする -->
                <OrchestrationStep Order="4" Type="ClaimsExchange">
                    <Preconditions>
                        <Precondition Type="ClaimsExist" ExecuteActionsIf="true">
                            <Value>objectId</Value>
                            <Action>SkipThisOrchestrationStep</Action>
                        </Precondition>
                        <Precondition Type="ClaimEquals" ExecuteActionsIf="true">
                            <Value>accountType</Value>
                            <Value>personal</Value>
                            <Action>SkipThisOrchestrationStep</Action>
                        </Precondition>
                    </Preconditions>
                    <ClaimsExchanges>
                        <ClaimsExchange Id="preSignUpWithLogonEmailExchange" TechnicalProfileReferenceId="preUserInformationCollector" />
                    </ClaimsExchanges>
                </OrchestrationStep>                 
                <!-- 
                    オーケストレーション 手順 4 - この手順は、ユーザーがサインアップする (objectId が存在しない) 場合に実行されるため、
                    UserInformationCollector セルフアサート技術プロファイルを呼び出して、サインアップ フォームを表示します。 
                    この手順は、ユーザーがサインアップする場合でもサインインする場合でも実行されます。
               -->
                <OrchestrationStep Order="5" Type="ClaimsExchange">
                    <Preconditions>
                        <Precondition Type="ClaimsExist" ExecuteActionsIf="true">
                            <Value>objectId</Value>
                            <Action>SkipThisOrchestrationStep</Action>
                        </Precondition>
                    </Preconditions>
                    <ClaimsExchanges>
                        <ClaimsExchange Id="SignUpWithLogonEmailExchange" TechnicalProfileReferenceId="UserInformationCollector" />
                    </ClaimsExchanges>
                </OrchestrationStep>   
                <!-- 
                   オーケストレーション 手順 5 - この手順では、Azure AD からアカウント情報を読み取ります (AAD-UserRead Azure AD 技術プロファイルを呼び出します)。
                   そのため、ユーザーがサインアップする場合でもサインインする場合でも実行されます。
                -->
                <OrchestrationStep Order="6" Type="ClaimsExchange">
                    <ClaimsExchanges>
                    <ClaimsExchange Id="AADUserReaderExchange" TechnicalProfileReferenceId="AAD-UserRead"/>
                    </ClaimsExchanges>
                </OrchestrationStep>                               
                <!-- 
                    オーケストレーション手順 6 - この手順では UserInputMessageClaimGenerator 技術プロファイルを呼び出して、
                    ユーザーへのあいさつメッセージを組み立てます。
                -->
                <OrchestrationStep Order="7" Type="ClaimsExchange">
                    <ClaimsExchanges>
                    <ClaimsExchange Id="GetMessageClaimsExchange" TechnicalProfileReferenceId="UserInputMessageClaimGenerator"/>
                    </ClaimsExchanges>          
                </OrchestrationStep>                
                <!-- 
                    ポリシーの実行の最後に JWT トークンをアセンブルして返します。
                -->
                <OrchestrationStep Order="8" Type="SendClaims" CpimIssuerTechnicalProfileReferenceId="JwtIssuer" />
    <!--</OrchestrationSteps>-->
```

オーケストレーションStep 2 から 6 では、前提条件を使用してオーケストレーション ステップを実行する必要があるかどうかを判断しました。 ユーザーがサインインしているのかまたはサインアップしているのかを判断する必要があります。  

カスタム ポリシーが実行されると、次のようになります

- **オーケストレーション手順 1** - サインイン ページが表示されるため、ユーザーはサインインするか、[今すぐサインアップ] リンクを選択することができます。 UserSignInCollector セルフアサート技術プロファイルがサインイン フォームの表示に使用するコンテンツ定義を指定していることに注意してください。

- **オーケストレーション 手順 2** - この手順は、ユーザーがサインアップする (objectId が存在しない) 場合に実行されるため、AccountTypeInputCollector セルフアサート技術プロファイルを呼び出して、アカウントの種類の選択フォームを表示します。

- **オーケストレーション 手順 3** - この手順は、ユーザーがサインアップし (objectId が存在しない)、ユーザーが会社の accountType を選択しない場合に実行されます。 そのため、AccessCodeInputCollector セルフアサート技術プロファイルを呼び出して、accessCode を入力するようにユーザーに求めます。

- **オーケストレーション 手順 4** - この手順は企業を選択した際のEメールアドレス入力が会社のアドレスかをチェックします。

- **オーケストレーション 手順 5** - この手順は、ユーザーがサインアップする (objectId が存在しない) 場合に実行されるため、UserInformationCollector セルフアサート技術プロファイルを呼び出して、サインアップ フォームを表示します。 この手順は、ユーザーがサインアップする場合でもサインインする場合でも実行されます。

- **オーケストレーション 手順 6** - この手順では、Azure AD からアカウント情報を読み取ります (AAD-UserRead Azure AD 技術プロファイルを呼び出します)。そのため、ユーザーがサインアップする場合でもサインインする場合でも実行されます。

- **オーケストレーション手順 7** - この手順では UserInputMessageClaimGenerator 技術プロファイルを呼び出して、ユーザーへのあいさつメッセージを組み立てます。

- **オーケストレーション手順 8** - 最後に、手順 8 では、ポリシーの実行の最後に JWT トークンをアセンブルして返します。



## Step 4 - ポリシーをアップロードする

「カスタム ポリシー ファイルをアップロードする」の手順に従って、ポリシー ファイルをアップロードします。 ポータルに既にあるファイルと同じ名前のファイルをアップロードする場合は、**[カスタム ポリシーが既に存在する場合は上書きします]** を選択してください。

## Step 5 - ポリシーをテストする

カスタム ポリシーをテストします。 ポリシーが実行されると、次のスクリーンショットのようなインターフェイスが表示されます。

 ![ScreenShot](/media/screenshot-sign-up-or-sign-in-interface.png)

サインインするには、既存のアカウントのメール アドレスとパスワードを入力します。 アカウントをまだ持っていない場合は、**[Sign up now]** リンクを選択して新しいユーザー アカウントを作成する必要があります。

