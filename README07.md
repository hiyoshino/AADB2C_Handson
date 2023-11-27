
# Azure Active Directory B2C カスタム ポリシーで REST API を呼び出す

Azure Active Directory B2C (Azure AD B2C) カスタム ポリシーを使用すると、Azure AD B2C の外部で実装されたアプリケーション ロジックと対話できるようになります。これを行うには、エンドポイントに対して HTTP 呼び出しを行います。Azure AD B2C カスタム ポリシーは、この目的のための RESTful 技術プロファイルを提供します。この機能を使用すると、Azure AD B2C カスタム ポリシー内では利用できない機能を実装できます。

この記事では、次の方法を学びます。

- RESTful 技術プロファイルを使用して、Node.js RESTful サービスへの HTTP 呼び出しを実行します。
- RESTful サービスがカスタム ポリシーに返すエラーを処理または報告します。

## シナリオ概要

「[Azure AD B2C カスタム ポリシーを使用してユーザー体験にブランチを作成する](https://github.com/hiyoshino/AADB2C_Handson/blob/main/README04.md)」では、"個人用アカウント" を選択したユーザーは、有効な招待アクセス コードを指定して続行する必要があります。 ここでは静的アクセス コードを使用していますが、実際のアプリはこの方法では動作しません。 アクセス コードを発行するサービスがカスタム ポリシーの外部にある場合は、そのサービスを呼び出し、ユーザーが入力したアクセス コードを渡して検証する必要があります。 そのアクセス コードが有効な場合は、サービスから HTTP 200 (OK) 応答が返され、Azure AD B2C によって JWT トークンが発行されます。 それ以外の場合は、サービスから HTTP 409 (競合) 応答が返されるため、ユーザーはアクセス コードを再入力する必要があります。


## Step 1 - RESTful 技術プロファイルを定義する

カスタム ポリシーから HTTP 呼び出しを行う必要があります。 Azure AD B2C カスタム ポリシーには、外部サービスの呼び出しに使用する [RESTful 技術プロファイル](https://learn.microsoft.com/ja-jp/azure/active-directory-b2c/restful-technical-profile)が用意されています。

### Step 1.1 - 追加のクレームを宣言する

`ContosoCustomPolicy.XML` ファイルで `ClaimsProviders` セクションを見つけ、次のコードを使用して新しい RESTful 技術プロファイルを定義します。

※`yourapiservice`は作成したAPIサービスに書き換える必要があります。

```xml
    <!--<ClaimsProviders>-->
        <ClaimsProvider>
            <DisplayName>HTTP Request Technical Profiles</DisplayName>
            <TechnicalProfiles>
                <TechnicalProfile Id="ValidateAccessCodeViaHttp">
                    <DisplayName>Check that the user has entered a valid access code by using Claims Transformations</DisplayName>
                    <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.RestfulProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
                    <Metadata>
                        <Item Key="ServiceUrl">https://yourapiservice.azurewebsites.net/validate-accesscode</Item>
                        <Item Key="SendClaimsIn">Body</Item>
                        <Item Key="AuthenticationType">None</Item>
                        <Item Key="AllowInsecureAuthInProduction">true</Item>
                    </Metadata>
                    <InputClaims>
                        <InputClaim ClaimTypeReferenceId="accessCode" PartnerClaimType="accessCode" />
                    </InputClaims>
                </TechnicalProfile>
            </TechnicalProfiles>
        </ClaimsProvider>
    <!--</ClaimsProviders>-->
``` 

プロトコルから、RestfulProvider を使用するように技術プロファイルが構成されていることを確認できます。 また、メタデータ セクションでは、次の情報も確認できます。

- ServiceUrl は API エンドポイントを表しています。 その値は https://yourapiservice.azurewebsites.net/validate-accesscode です。 必ず呼び出す REST API エンドポイントの値を更新してください。

- SendClaimsIn には、RESTful クレーム プロバイダーへの入力要求の送信方法を指定します。 指定できる値: Body (default)、Form、Header、Url、または QueryString。 この記事のように Body を使用する場合は、POST HTTP メソッドを呼び出し、要求の本文でキーと値のペアとして書式設定されている場合は API に送信するデータを呼び出します。 GET HTTP メソッドを呼び出し、クエリ文字列としてデータを渡す方法を確認してください。

- AuthenticationType には、RESTful クレーム プロバイダーにより実行される認証の種類を指定します。 RESTful クレーム プロバイダーは保護されていないエンドポイントを呼び出すため、ここでは AuthenticationType を None に設定します。 認証の種類を Bearer に設定する場合は、アクセス トークンのストレージを指定する CryptographicKeys 要素を追加する必要があります。 RESTful クレーム プロバイダーでサポートされる認証の種類を確認してください。

- InputClaim の PartnerClaimType 属性は、API でデータを受け取る方法を指定します。

次に示すような POST 要求を実施します。(user-code-coeには入力したアクセスコードに置き換えます。)

```http

        POST https://yourapiservice.azurewebsites.net/validate-accesscode  HTTP/1.1
        Host: yourapiservice.azurewebsites.net/validate-accesscode
        Content-Type: application/x-www-form-urlencoded
    
        accessCode=user-code-code

```

入力したアクセスコードが `88888` の場合は HTTP 200 OK を返しますが、コードが間違っている場合は、Status 409 とともに以下の JSON メッセージが返信されます。

```json

        {
            "version": "1.0",
            "status": 409,
            "code": "errorCode",
            "requestId": "requestId",
            "userMessage": "The access code you entered is incorrect. Please try again.",
            "developerMessage": "The provided code 54321 does not match the expected code for user.",
            "moreInfo": "https://docs.microsoft.com/en-us/azure/active-directory-b2c/string-transformations"
        }

```



### Step 1.2 - 検証技術プロファイルを更新する

「[Azure AD B2C カスタム ポリシーを使用してユーザー体験に分岐を作成する](https://github.com/hiyoshino/AADB2C_Handson/blob/main/README04.md)」では、クレームの変換を使用して accessCode を検証しました。 この記事では、外部サービスへの HTTP 呼び出しを行って accessCode を検証します。 そのため、新しいアプローチを反映するようにカスタム ポリシーを更新する必要があります。

AccessCodeInputCollector 技術プロファイルを見つけ、ValidationTechnicalProfile 要素の ReferenceId を ValidateAccessCodeViaHttp に更新します。


更新前: 

```xml
    <ValidationTechnicalProfile ReferenceId="CheckAccessCodeViaClaimsTransformationChecker"/>
```
更新後: 

```xml
    <ValidationTechnicalProfile ReferenceId="ValidateAccessCodeViaHttp"/>
```

この時点で、IdCheckAccessCodeViaClaimsTransformationChecker の技術プロファイルは不要となるため、削除することができます。


### Step 2 - カスタム ポリシー ファイルをアップロードする

REST API アプリが実行されていることを確認し、「[カスタム ポリシー ファイルをアップロードする](https://github.com/hiyoshino/AADB2C_Handson/blob/main/README03.md#step-5---%E3%82%AB%E3%82%B9%E3%82%BF%E3%83%A0-%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC-%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E3%82%A2%E3%83%83%E3%83%97%E3%83%AD%E3%83%BC%E3%83%89%E3%81%99%E3%82%8B)」の手順に従って、ポリシー ファイルをアップロードします。 ポータルに既にあるファイルと同じ名前のファイルをアップロードする場合は、[カスタム ポリシーが既に存在する場合は上書きします] を選択してください。


### Step 3 - カスタム ポリシーをテストする

「[カスタム ポリシーをテストする](https://github.com/hiyoshino/AADB2C_Handson/blob/main/README03.md#step-6---%E3%82%AB%E3%82%B9%E3%82%BF%E3%83%A0-%E3%83%9D%E3%83%AA%E3%82%B7%E3%83%BC%E3%82%92%E3%83%86%E3%82%B9%E3%83%88%E3%81%99%E3%82%8B)」の手順に従って、カスタム ポリシーをテストします。

- **AccountType** で、**Personal Account** を選択します。
- 必要に応じて残りの詳細を入力し、[Continue] を選択します。 新しい画面が表示されます。
- **Access Code** に **88888**と入力し、[Continue] を選択します。 ポリシーの実行が完了すると、https://jwt.ms にリダイレクトされ、デコードされた JWT トークンが表示されます。 手順を繰り返し、88888 以外の別のアクセス コードを入力すると、**The access code you entered is incorrect. Please try again.** というエラーが表示されます。


### Step 3 - デバッグ モードを有効にする

開発時には、API によって送信される詳細なエラー (developerMessage や moreInfo など) を確認する必要がある場合があります。 この場合は、RESTful 技術プロバイダーでデバッグ モードを有効にする必要があります。

1. ValidateAccessCodeViaHttp 技術プロバイダーを見つけて、技術プロバイダーの metadata に次の項目を追加します。

    ```xml
        <Item Key="DebugMode">true</Item>
    ```

1. 変更を保存し、ポリシー ファイルをアップロードします。

1. カスタム ポリシーをテストします。 間違ったアクセス コードを入力するようにします。 下のスクリーンショットに示すようなエラーが表示されます。

 ![ScreenShot](/media/screenshot-error-enable-debug-mode.png)









