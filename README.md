

# 初めての Azure Active Directory B2C カスタム ポリシーを作成!- Hello World!

アプリケーションでは、ユーザーがサインアップ、サインイン、またはプロファイルを管理できるようにするユーザー フローを使用できます。ユーザー フローがビジネス固有のニーズをすべてカバーできない場合は、カスタム ポリシーを使用します。

事前に作成されたカスタム ポリシー[スターターパック](https://github.com/Azure-Samples/active-directory-b2c-custom-policy-starterpack)を使用してカスタム ポリシーを作成できますが、カスタム ポリシーの構築方法を理解することが重要です。この記事では、最初のカスタム ポリシーを最初から作成する方法を学習します。

## Step 1 - 署名キーと暗号化キーを構成する

まだ行っていない場合は、次の暗号化キーを作成します。 

  1. [「Identity Experience Framework アプリケーションの署名および暗号化キーを追加する」](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/tutorial-create-user-flows?pivots=b2c-custom-policy#add-signing-and-encryption-keys-for-identity-experience-framework-applications)の手順を使用して、署名キーを作成します。
  
  2. [「ProxyIdentity Experience Framework アプリケーションの署名および暗号化キーを追加する」](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/tutorial-create-user-flows?pivots=b2c-custom-policy#register-the-proxyidentityexperienceframework-application)の手順を使用して、暗号化キーを作成します。
  

## Step 2 - カスタム ポリシー ファイルをビルドする

1. VS Code で、`ContosoCustomPolicy.XML` ファイルを作成して開きます。

2. `ContosoCustomPolicy.XML` ファイルに次のコードを追記します。:
    
    ```xml
        <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
        <TrustFrameworkPolicy
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns:xsd="http://www.w3.org/2001/XMLSchema"
          xmlns="http://schemas.microsoft.com/online/cpim/schemas/2013/06"
          PolicySchemaVersion="0.3.0.0"
          TenantId="yourtenant.onmicrosoft.com"
          PolicyId="B2C_1A_ContosoCustomPolicy"
          PublicPolicyUri="http://yourtenant.onmicrosoft.com/B2C_1A_ContosoCustomPolicy">
        
            <BuildingBlocks>
                <!-- Building Blocks Here-->
            </BuildingBlocks>
    
            <ClaimsProviders>
                <!-- Claims Providers Here-->
            </ClaimsProviders>
            
            <UserJourneys>
                <!-- User Journeys Here-->
            </UserJourneys>
            
            <RelyingParty>
                <!-- 
                    Relying Party Here that's your policy’s entry point
                    Specify the User Journey to execute 
                    Specify the claims to include in the token that is returned when the policy runs
                -->
            </RelyingParty>
        </TrustFrameworkPolicy>

    ```
    `yourtenant` を、テナント名のサブドメイン部分( `contoso`など) に置き換えます。

    XML 要素は、ポリシー ID とテナント名を持つポリシー ファイルの最上位 `TrustFrameworkPolicy` 要素を定義します。 `TrustFrameworkPolicy` 要素には、このシリーズで使用する他の XML 要素が含まれています。

3. 要求を宣言するには、ContosoCustomPolicy.XML ファイルの BuildingBlocks セクションに次のコードを追加します。: 

    ```xml
      <ClaimsSchema>
        <ClaimType Id="objectId">
            <DisplayName>unique object Id for subject of the claims being returned</DisplayName>
            <DataType>string</DataType>
        </ClaimType>        
        <ClaimType Id="message">
            <DisplayName>Will hold Hello World message</DisplayName>
            <DataType>string</DataType>
        </ClaimType>
      </ClaimsSchema>
    ``` 
    要求は変数に似ています。 要求の宣言には、要求のデータ型も表示されます。

4. ContosoCustomPolicy.XML ファイルの ClaimsProviders セクションで、次のコードを追加します。:

    ```xml
        <ClaimsProvider>
          <DisplayName>Token Issuer</DisplayName>
          <TechnicalProfiles>
            <TechnicalProfile Id="JwtIssuer">
              <DisplayName>JWT Issuer</DisplayName>
              <Protocol Name="None" />
              <OutputTokenFormat>JWT</OutputTokenFormat>
              <Metadata>
                <Item Key="client_id">{service:te}</Item>
                <Item Key="issuer_refresh_token_user_identity_claim_type">objectId</Item>
                <Item Key="SendTokenResponseBodyWithJsonNumbers">true</Item>
              </Metadata>
              <CryptographicKeys>
                <Key Id="issuer_secret" StorageReferenceId="B2C_1A_TokenSigningKeyContainer" />
                <Key Id="issuer_refresh_token_key" StorageReferenceId="B2C_1A_TokenEncryptionKeyContainer" />
              </CryptographicKeys>
            </TechnicalProfile>
          </TechnicalProfiles>
        </ClaimsProvider>

        <ClaimsProvider>
          <!-- The technical profile(s) defined in this section is required by the framework to be included in all custom policies. -->
          <DisplayName>Trustframework Policy Engine TechnicalProfiles</DisplayName>
          <TechnicalProfiles>
            <TechnicalProfile Id="TpEngine_c3bd4fe2-1775-4013-b91d-35f16d377d13">
              <DisplayName>Trustframework Policy Engine Default Technical Profile</DisplayName>
              <Protocol Name="None" />
              <Metadata>
                <Item Key="url">{service:te}</Item>
              </Metadata>
            </TechnicalProfile>
          </TechnicalProfiles>
        </ClaimsProvider>
    ```
    
    JWT トークン発行者を宣言しました。 Step 1 で別の名前を使用して署名キーと暗号化キーを構成した場合は、CryptographicKeys セクションで StorageReferenceId に正しい値を使用していることを確認します。

5. ContosoCustomPolicy.XML ファイルの UserJourneys セクションで、次のコードを追加します。:

    ```xml
      <UserJourney Id="HelloWorldJourney">
        <OrchestrationSteps>
          <OrchestrationStep Order="1" Type="SendClaims" CpimIssuerTechnicalProfileReferenceId="JwtIssuer" />
        </OrchestrationSteps>
      </UserJourney>
    ```
    
    UserJourney を追加しました。 ユーザー体験では、Azure AD B2C で要求が処理される際にエンド ユーザーが通過するビジネス ロジックを指定します。 このユーザー体験には、次の手順で定義する要求で JTW トークンを発行する手順が 1 つだけ含まれています。

6.  ContosoCustomPolicy.XML ファイルの RelyingParty セクションで、次のコードを追加します。

    ```xml
      <DefaultUserJourney ReferenceId="HelloWorldJourney"/>
      <TechnicalProfile Id="HelloWorldPolicyProfile">
        <DisplayName>Hello World Policy Profile</DisplayName>
        <Protocol Name="OpenIdConnect" />
        <OutputClaims>
          <OutputClaim ClaimTypeReferenceId="objectId" PartnerClaimType="sub" DefaultValue="abcd-1234-efgh-5678-ijkl-etc."/>
          <OutputClaim ClaimTypeReferenceId="message" DefaultValue="Hello World!"/>
        </OutputClaims>
        <SubjectNamingInfo ClaimType="sub" />
      </TechnicalProfile>
    ```

    RelyingParty セクションは、ポリシーのエントリ ポイントです。 実行する UserJourney と、ポリシーの実行時に返されるトークンに含める要求を指定します。 

完了すると、ContosoCustomPolicy.XML ファイルは次のコードのようになります。

```xml
    <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
    <TrustFrameworkPolicy xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns="http://schemas.microsoft.com/online/cpim/schemas/2013/06" PolicySchemaVersion="0.3.0.0" TenantId="Contosob2c2233.onmicrosoft.com" PolicyId="B2C_1A_ContosoCustomPolicy" PublicPolicyUri="http://Contosob2c2233.onmicrosoft.com/B2C_1A_ContosoCustomPolicy">
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
            </ClaimsSchema>
        </BuildingBlocks>
        <ClaimsProviders><!--Claims Providers Here-->
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
                <Protocol Name="None" />
                <Metadata>
                    <Item Key="url">{service:te}</Item>
                </Metadata>
                </TechnicalProfile>
            </TechnicalProfiles>
            </ClaimsProvider>
        </ClaimsProviders>
      <UserJourneys>
        <UserJourney Id="HelloWorldJourney">
          <OrchestrationSteps>
            <OrchestrationStep Order="1" Type="SendClaims" CpimIssuerTechnicalProfileReferenceId="JwtIssuer" />
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
                    <OutputClaim ClaimTypeReferenceId="objectId" PartnerClaimType="sub" DefaultValue="abcd-1234-efgh-5678-ijkl-etc."/>
                    <OutputClaim ClaimTypeReferenceId="message" DefaultValue="Hello World!"/>
                </OutputClaims>
                <SubjectNamingInfo ClaimType="sub"/>
            </TechnicalProfile>
        </RelyingParty>
    </TrustFrameworkPolicy>
```
    
## Step 3 - カスタム ポリシー ファイルをアップロードする

1. Azure portal にサインインします。
1. ご利用の Azure AD B2C テナントが含まれるディレクトリを必ず使用してください。
    1. ポータル ツールバーの [Directories + subscriptions](ディレクトリ + サブスクリプション) アイコンを選択します。
    1. [ポータルの設定] | [Directories + subscriptions](ディレクトリ + サブスクリプション) ページの [ディレクトリ名] の一覧で自分の Azure AD B2C ディレクトリを見つけて、 [切り替え] を選択します。
1. Azure portal で、 [Azure AD B2C] を検索して選択します。
1. 左側のメニューの [ポリシー] で、[Identity Experience Framework] を選択します。
1. [カスタム ポリシーのアップロード] を選択し、参照して選択し、ContosoCustomPolicy.XML ファイルをアップロードします。


ファイルをアップロードすると、Azure AD B2C によって プレフィックス B2C_1A_ が追加されるため、名前は B2C_1A_CONTOSOCUSTOMPOLICY のようになります


## Step 4 - カスタム ポリシーをテストする

1. [カスタム ポリシー] で、[B2C_1A_CONTOSOCUSTOMPOLICY] を選択します。
2. カスタム ポリシーの概要ページの [アプリケーションの選択] で、以前に登録した webapp1 などの Web アプリケーションを選択します。 [応答 URL の選択] の値が https://jwt.ms に設定されていることを確認します。
3. [今すぐ実行] ボタンを選択します。

ポリシーの実行が完了すると、https://jwt.ms にリダイレクトされ、デコードされた JWT トークンが表示されます。 次の JWT トークン スニペットのようになります。

```json
    {
      "typ": "JWT",
      "alg": "RS256",
      "kid": "pxLOMWFg...."
    }.{
      ...
      "sub": "abcd-1234-efgh-5678-ijkl-etc.",
      ...
      "acr": "b2c_1a_contosocustompolicy",
      ...
      "message": "Hello World!"
    }.[Signature]
``` 

RelyingParty セクションで出力要求として設定した message と sub 要求に注目してください。

完全なポリシーは [Policy/ContosoCustomPolicy01.xml](policy/ContosoCustomPolicy01.xml) になります。
