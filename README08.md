# Azure Active Directory B2C カスタム ポリシーで ドロップダウンリストを表示

Azure Active Directory B2C カスタム ポリシーを使用し、都市を指定するドロップダウンリストをユーザーサインアップ時に選択できるようにする。

## シナリオ概要

サインアップフローでアカウントタイプで Personal を選択した際に、都市を指定するドロップダウンリストを表示し、ユーザーに都市を選択するように促します。
ユーザーが選択した都市は、トークンに city 属性の情報としてアプリケーションに連携します。

## Step 1 - クレームを宣言する

 *city* を追加のクレームとして宣言します。UserInputType に DropdownSingleSelect を指定することで、ユーザーに都市を一覧から選択する画面を表示することが可能です。 SelectByDefault="true" を指定するとデフォルト値として設定します。

1. VS Code で `ContosoCustomPolicy.XML` ファイルを開きます。 

1. `ClaimsSchema` セクションに、次の 'ClaimType' の宣言を追加します。: 

    ```xml
        <ClaimType Id="city">
        <DisplayName>City where you work</DisplayName>
        <DataType>string</DataType>
        <UserInputType>DropdownSingleSelect</UserInputType>
        <Restriction>
            <Enumeration Text="Sapporo" Value="sapporo" SelectByDefault="false" />
            <Enumeration Text="Tokyo" Value="tokyo" SelectByDefault="true" />
            <Enumeration Text="Nagoya" Value="nagoya" SelectByDefault="false" />
            <Enumeration Text="Osaka" Value="osaka" SelectByDefault="false" />
            <Enumeration Text="Fukoka" Value="fukuoka" SelectByDefault="false" />
        </Restriction>
        </ClaimType>
    ```  


## Step 2 - 都市情報を収集する技術プロファイルを構成する

都市を選択するための画面を表示する技術プロファイルは、ClaimsProvider 要素内で以下のように宣言します。:

    ```xml
        <ClaimsProvider>
        <DisplayName>Technical Profiles to collect user's city </DisplayName>
        <TechnicalProfiles>
            <TechnicalProfile Id="UsersCityCollector">
                <DisplayName>Collect user's city from user Technical Profile</DisplayName>
                <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.SelfAssertedAttributeProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"/>
                <Metadata>
                    <Item Key="ContentDefinitionReferenceId">SelfAssertedContentDefinition</Item>
                </Metadata>
                <DisplayClaims>
                    <DisplayClaim ClaimTypeReferenceId="city" Required="true"/>
                </DisplayClaims>
                <OutputClaims>
                    <OutputClaim ClaimTypeReferenceId="city"/>
                </OutputClaims>
            </TechnicalProfile>
        </TechnicalProfiles>
        </ClaimsProvider>
    ```

## Step 3 - UserJourney に都市選択を追加する

都市選択の入力処理を UserJourney に追加します。
HelloWorldJourney の UserJourney を見つけ出し、OrchestrationStep Order="3" の後に、 Order="4" として以下の xml を追加します。追加後、既存の Order=4 以降 Orchestration Step の Order の数を一つ増やします。最終的に Order は 9 になります。

    ```xml
        <OrchestrationStep Order="4" Type="ClaimsExchange">
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
                <ClaimsExchange Id="GetCityClaimsExchange" TechnicalProfileReferenceId="UsersCityCollector" />
            </ClaimsExchanges>
        </OrchestrationStep>
    ```

## Step 4 - RelyingParty に OutputClaim に city を追加する

選択した都市をアプリケーションに Claim を JWTトークンに含めてアプリケーションに連携します。
RelayingParty ポリシーの OutputoClaims 内に以下のクレームを追加します。

    ```xml
        <OutputClaim ClaimTypeReferenceId="city" />

    ```

### Step 5 - カスタム ポリシー ファイルをアップロードする

「[カスタム ポリシー ファイルをアップロードする](https://github.com/hiyoshino/AADB2C_Handson/blob/main/README03.md#step-5---%E3%82%AB%E3%82%B9%E3%82%BF%E3%83%A0-%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC-%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E3%82%A2%E3%83%83%E3%83%97%E3%83%AD%E3%83%BC%E3%83%89%E3%81%99%E3%82%8B)」の手順に従って、ポリシー ファイルをアップロードします。 ポータルに既にあるファイルと同じ名前のファイルをアップロードする場合は、[カスタム ポリシーが既に存在する場合は上書きします] を選択してください。

