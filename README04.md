# Azure Active Directory B2C カスタム ポリシーを使用してユーザー体験に分岐を作成する

同じアプリの異なるユーザーが、カスタム ポリシー内のデータの値に応じて、異なるユーザー体験をたどることができます。 Azure Active Directory B2C (Azure AD B2C) カスタム ポリシーを使用すると、条件に応じて技術プロファイルを有効または無効にして、この機能を実現できます。 たとえば、「Azure AD B2C カスタム ポリシーを使用してユーザー入力を検証する」では、Precondition を使用し、"accountType" クレームの値に基づいて、検証技術プロファイルを実行する必要があるかどうかを判別しました。

技術プロファイルには、技術プロファイルを実行するかどうかを指定できる `EnabledForUserJourneys` 要素も用意されています。 `EnabledForUserJourneys` 要素には、"OnClaimsExistence" を含む 5 つの値のいずれかが含まれています。これを使用して、指定した特定のクレームが技術プロファイルに存在するときにのみ技術プロファイルを実行するように指定します。 技術プロファイルの `EnabledForUserJourneys` 要素の詳細を確認してください。

## シナリオの概要

「Azure AD B2C カスタム ポリシーを使用してユーザー入力を検証する」の記事では、ユーザーは詳細を 1 つの画面で入力します。 この記事では、ユーザーはまず、自分のアカウントの種類である "Contoso Employee Account (Contoso 従業員アカウント)" または "個人用アカウント" を選択する必要があります。 "Contoso Employee Account (Contoso 従業員アカウント)" を選択したユーザーは、詳細の入力に進むことができます。 しかし、"個人用アカウント" を選択したユーザーは、詳細の指定に進む前に有効な招待アクセス コードを指定する必要があります。 そのため、"個人用アカウント" というアカウントの種類を使用するユーザーには、体験を完了するために追加のユーザー インターフェイスが表示されます。

この記事では、技術プロファイル内の `EnabledForUserJourneys` 要素を使用して、クレームの値に基づいて異なるユーザー体験を作成する方法について説明します。 まず、ユーザーが自分のアカウントの種類を選択します。

## 前提条件

- まだ持っていない場合は、お使いの Azure サブスクリプションにリンクされている Azure AD B2C テナントを作成します。

- Web アプリケーションを登録し、ID トークンの暗黙的な許可を有効にします。 リダイレクト URI には https://jwt.ms を使用します。

- お使いのコンピューターに Visual Studio Code (VS Code) がインストールされている必要があります。

- 「Azure AD B2C カスタム ポリシーを使用してユーザー入力を検証する」の手順を完了します。 この記事は、独自のカスタム ポリシーの作成と実行に関するハウツー ガイド シリーズの一部です。

## Step 1 - クレームを宣言する 

"個人用アカウント"を選択したユーザーは、有効なアクセス コードを指定する必要があります。 そのため、この値を保持するためのクレームが必要です

1. VS Code で `ContosoCustomPolicy.XML` ファイルを開きます。 

1. `ClaimsSchema` セクションで、次のコードを使用して accessCode と isValidAccessCode クレームを宣言します。

    ```xml    
        <ClaimType Id="accessCode">
            <DisplayName>Access Code</DisplayName>
            <DataType>string</DataType>
            <UserHelpText>Enter your invitation access code.</UserHelpText>
            <UserInputType>Password</UserInputType>
            <Restriction>
                <Pattern RegularExpression="[0-9][0-9][0-9][0-9][0-9]" HelpText="Please enter your invitation access code. It's a 5-digit number, something like 95765"/>
            </Restriction>
        </ClaimType>
        <ClaimType Id="isValidAccessCode">
            <DataType>boolean</DataType>
        </ClaimType>
    ```


## Step 2 - クレームの変換を定義する

`ClaimsTransformations` 要素を見つけて、次のクレームの変換を追加します。

```xml
    <!---<ClaimsTransformations>-->
        <ClaimsTransformation Id="CheckIfIsValidAccessCode" TransformationMethod="CompareClaimToValue">
            <InputClaims>
                <InputClaim ClaimTypeReferenceId="accessCode" TransformationClaimType="inputClaim1"/>
            </InputClaims>
            <InputParameters>
                <InputParameter Id="compareTo" DataType="string" Value="88888"/>
                <InputParameter Id="operator" DataType="string" Value="equal"/>
                <InputParameter Id="ignoreCase" DataType="string" Value="true"/>
            </InputParameters>
            <OutputClaims>
                <OutputClaim ClaimTypeReferenceId="isValidAccessCode" TransformationClaimType="outputClaim"/>
            </OutputClaims>
        </ClaimsTransformation>
        <ClaimsTransformation Id="ThrowIfIsNotValidAccessCode" TransformationMethod="AssertBooleanClaimIsEqualToValue">
            <InputClaims>
                <InputClaim ClaimTypeReferenceId="isValidAccessCode" TransformationClaimType="inputClaim"/>
            </InputClaims>
            <InputParameters>
                <InputParameter Id="valueToCompareTo" DataType="boolean" Value="true"/>
            </InputParameters>
        </ClaimsTransformation>
    <!---</ClaimsTransformations>-->
``` 

"CheckIfIsValidAccessCode" と "ThrowIfIsNotValidAccessCode" という 2 つのクレームの変換を定義しました。 "CheckIfIsValidAccessCode" は、CompareClaimToValue 変換メソッドを使用して、ユーザーが入力したアクセス コードを静的な値 88888 (この値をテストに使用しましょう) と比較し、"isValidAccessCode" クレームに true または false を割り当てます。 "ThrowIfIsNotValidAccessCode" は、2 つのクレームの 2 つのブール値が等しいかどうかを確認し、等しくない場合は例外をスローします。

## Step 3 - 技術プロファイルを構成または更新する 

ここで、2 つの新しいセルフアサート技術プロファイルが必要です。1 つはアカウントの種類を収集し、もう 1 つはユーザーからアクセス コードを収集します。 手順 2 で定義したクレームの変換を実行してユーザーのアクセス コードを検証するために、新しいクレームの変換の種類の技術プロファイルも必要です。 別のセルフアサート技術プロファイルでアカウントの種類を収集するため、UserInformationCollector セルフアサート技術プロファイルを更新して、アカウントの種類を収集しないようにする必要があります。

1. `ClaimsProviders` 要素を見つけ、次のコードを使用して新しいクレーム プロバイダーを追加します。

    ```xml
        <!--<ClaimsProviders>-->
            <ClaimsProvider>
                <DisplayName>Technical Profiles to collect user's access code</DisplayName>
                <TechnicalProfiles>
                    <TechnicalProfile Id="AccessCodeInputCollector">
                        <DisplayName>Collect Access Code Input from user Technical Profile</DisplayName>
                        <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.SelfAssertedAttributeProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"/>
                        <Metadata>
                            <Item Key="ContentDefinitionReferenceId">SelfAssertedContentDefinition</Item>
                            <Item Key="UserMessageIfClaimsTransformationBooleanValueIsNotEqual">The access code is invalid.</Item>
                            <Item Key="ClaimTypeOnWhichToEnable">accountType</Item>
                            <Item Key="ClaimValueOnWhichToEnable">personal</Item>
                        </Metadata>
                        <DisplayClaims>
                            <DisplayClaim ClaimTypeReferenceId="accessCode" Required="true"/>
                        </DisplayClaims>
                        <OutputClaims>
                            <OutputClaim ClaimTypeReferenceId="accessCode"/>
                        </OutputClaims>
                        <ValidationTechnicalProfiles>
                            <ValidationTechnicalProfile ReferenceId="CheckAccessCodeViaClaimsTransformationChecker"/>
                        </ValidationTechnicalProfiles>
                        <EnabledForUserJourneys>OnClaimsExistence</EnabledForUserJourneys>
                    </TechnicalProfile>
                    <TechnicalProfile Id="CheckAccessCodeViaClaimsTransformationChecker">
                        <DisplayName>A Claims Transformations to check user's access code validity</DisplayName>
                        <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.ClaimsTransformationProtocolProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"/>
                        <OutputClaims>
                            <OutputClaim ClaimTypeReferenceId="isValidAccessCode"/>
                        </OutputClaims>
                        <OutputClaimsTransformations>
                            <OutputClaimsTransformation ReferenceId="CheckIfIsValidAccessCode"/>
                            <OutputClaimsTransformation ReferenceId="ThrowIfIsNotValidAccessCode"/>
                        </OutputClaimsTransformations>
                    </TechnicalProfile>
                </TechnicalProfiles>
            </ClaimsProvider>
        <!--</ClaimsProviders>-->
    ```

"AccessCodeInputCollector" と "CheckAccessCodeViaClaimsTransformationChecker" という 2 つの技術プロファイルを構成しました。 "CheckAccessCodeViaClaimsTransformationChecker" 技術プロファイルは、"AccessCodeInputCollector" 技術プロファイル内から検証技術プロファイルとして呼び出します。 "CheckAccessCodeViaClaimsTransformationChecker" は、それ自体の種類がクレームの変換技術プロファイルであり、手順 2 で定義したクレームの変換を実行します。

 "AccessCodeInputCollector" は、ユーザーからアクセス コードを収集するために使用されるセルフアサート技術プロファイルです。 これには、"OnClaimsExistence" に設定された EnabledForUserJourneys 要素が含まれます。 その Metadata 要素には、この技術プロファイルをアクティブにするクレーム ("accountType") とその値 ("personal") が含まれます。

1. `ClaimsProviders` 要素を見つけ、次のコードを使用して新しいクレーム プロバイダーを追加します。  

    ```xml
        <!--<ClaimsProviders>-->
            <ClaimsProvider>
                <DisplayName>Technical Profile to collect user's accountType</DisplayName>
                <TechnicalProfiles>
                    <TechnicalProfile Id="AccountTypeInputCollector">
                        <DisplayName>Collect User Input Technical Profile</DisplayName>
                        <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.SelfAssertedAttributeProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"/>
                        <Metadata>
                            <Item Key="ContentDefinitionReferenceId">SelfAssertedContentDefinition</Item>
                        </Metadata>
                        <DisplayClaims>
                            <DisplayClaim ClaimTypeReferenceId="accountType" Required="true"/>
                        </DisplayClaims>
                        <OutputClaims>
                            <OutputClaim ClaimTypeReferenceId="accountType"/>
                        </OutputClaims>
                    </TechnicalProfile>
                </TechnicalProfiles>
            </ClaimsProvider>
        <!--</ClaimsProviders>-->
    ```   
    
    ユーザーのアカウントの種類を収集するセルフアサート技術プロファイル AccountTypeInputCollector を構成しました。 これは、AccessCodeInputCollector セルフアサート技術プロファイルをアクティブ化する必要があるかどうかを決定するアカウントの種類の値です。

1. UserInformationCollector セルフアサート技術プロファイルでアカウントの種類が収集されないようにするには、UserInformationCollector セルフアサート技術プロファイルを見つけて、次のようにします。
    
    1. DisplayClaims コレクションから accountType 表示クレームである <DisplayClaim ClaimTypeReferenceId="accountType" Required="true"/> を削除します。
    
    1. OutputClaims コレクションから accountType 出力クレームである <OutputClaim ClaimTypeReferenceId="accountType"/> を削除します。


## Step 4 - ユーザー体験のオーケストレーション手順を更新する

技術プロファイルを設定したため、ユーザー体験のオーケストレーション手順を更新する必要があります。

1. `HelloWorldJourney` ユーザー体験を見つけて、すべてのオーケストレーション手順を次のコードに置き換えます。

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
                    <ClaimsExchange Id="GetMessageClaimsExchange" TechnicalProfileReferenceId="ClaimGenerator"/>
                </ClaimsExchanges>
            </OrchestrationStep>
            <OrchestrationStep Order="5" Type="SendClaims" CpimIssuerTechnicalProfileReferenceId="JwtIssuer"/>
        <!--</OrchestrationSteps>-->
    ``` 
     オーケストレーション手順は、オーケストレーション手順の Order 属性で示されている順序で技術プロファイルを呼び出すことを示しています。 ただし、ユーザーが "個人用アカウント" というアカウントの種類を選択した場合は、AccessCodeInputCollector 技術プロファイルがアクティブになります。 

## Step 5 - カスタム ポリシー ファイルをアップロードする

[「カスタム ポリシー ファイルをアップロードする」](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/custom-policies-series-hello-world#step-3---upload-custom-policy-file)の手順に従います。 ポータルに既にあるファイルと同じ名前のファイルをアップロードする場合は、[カスタム ポリシーが既に存在する場合は上書きします] を選択してください。

## Step 6 - カスタム ポリシーをテストする

カスタム ポリシーをテストします。

1. 最初の画面で、[アカウントの種類] に [個人用アカウント] を選択します。
1. [アクセス コード] に「88888」と入力し、[続行] を選択します。
1. 必要に応じて残りの詳細を入力し、[続行] を選択します。 ポリシーの実行が完了すると、https://jwt.ms にリダイレクトされ、デコードされた JWT トークンが表示されます。
1. 手順 5 を繰り返しますが、今度は [アカウントの種類] を選択し、[Contoso Employee Account] (Contoso 従業員アカウント) を選択して、プロンプトに従います。
 
