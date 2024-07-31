# Red Hat Developer Sandbox デプロイ手順書

## 1. Red Hat Developer Sandbox の払い出し

1. [Red Hat Developer Sandbox](https://developers.redhat.com/developer-sandbox)にアクセスします。
2. 「Start your sandbox for Free」をクリックします。
3. Red Hat アカウントを作成またはログインします。
4. [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/sandbox)が開かれます。
5. Available services の下に、Red Hat OpenShift があり、Launch ボタンをクリックします。
6. しばらく待つと、Sandbox が作成されて、OpenShift の Web Console 画面が表示されます。

## 2. GitHub リポジトリのフォーク

1. [アプリケーションのリポジトリ](https://github.com/coolboy0961/red-hat-developer-sandbox-apps)と[マニフェストのリポジトリ](https://github.com/coolboy0961/red-hat-developer-sandbox-manifest)にアクセスします。
2. 各リポジトリの右上にある「Fork」ボタンをクリックし、自分の GitHub アカウントにフォークします。
3. フォークした後リポジトリを自身の開発 PC にクローンします。

## 3. Fine-grained Personal Access Token の発行

1. GitHub の右上にあるプロフィールアイコンをクリックし、「Settings」を選択します。
2. 左側のメニューの一番下にある「Developer settings」を選択し、「Personal access tokens」 -> 「Fine-grained tokens」をクリックします。
3. 「Generate new token」をクリックします。
4. 「Repository access」で マニフェストのリポジトリのみを選択します。
5. 「Repository permissions」をクリックして、「Contents」の権限を「Read and write」にします。
6. 「Generate token」をクリックしてトークンを生成します。
7. トークンをコピーし、安全な場所に保存します。(二度と表示されません)
8. このトークンを GitHub 　のリポジトリに保存しないでください。保存すると、すぐ失効になってしまいます。

## 4. ローカル環境での `oc` コマンドのインストール

1. [getting-started-cli](https://docs.okd.io/latest/cli_reference/openshift_cli/getting-started-cli.html)の手順に従って、oc cli をインストールします。
2. MacOS で brew がインストールされていれば、`brew install openshift-cli`だけでインストールできます。

## 5. oc コマンドが使えるように Sandbox 環境へのログイン

1. OpenShift の Web Console 画面の右上にあるプロフィールアイコンをクリックし、「Copy login command」をクリックします。
2. 新しいタブが開かれて、「Display token」をクリックします。
3. 「Log in with this token」の下に `oc login --token=<取得したトークン> --server=<APIサーバーURL>`の文字列をコピーします。
4. ターミナルにそれをコピーして実行します。
5. `oc project`で現在の Namespace が<自身のプロフィール名>-dev になっていることを確認します。

## 6. `secret/git-credentials` の作成

1. 以下のコマンドを実行して、GitHub のトークンを含むシークレットを作成します。
2. username は GitHub のユーザー名です。

```bash
oc create secret generic git-credentials \
  --from-literal=username=<username> \
  --from-literal=token=<token>
```

## 7. Namespace の置換

1. apps と manifest のリポジトリにおいて、`jiadchen-dev` となっているすべての部分を、ご自身の Namespace に置換します。
2. 変更を Git コミットします。

## 8. GitOps Pipeline マニフェストの適用

1. manifest のリポジトリをクローンしたディレクトリーに移動します。
2. gitops-pipeline フォルダに移動します。
3. 以下のコマンドを実行して、GitOps Pipeline のマニフェストを適用します。

```bash
oc apply -k overlays/dev
```

## 9. Event Listener の URL 設定 (gitops-pipeline)

1. 以下のコマンドを実行して、Event Listener の URL を取得します。

```bash
oc get route el-gitops-event-listener-dev
```

2. HOST/PORT の列に` el-gitops-event-listener-dev-jiadchen-dev.apps.sandbox-m2.ll9k.p1.openshiftapps.com`のような URL が表示されます。
3. 取得した URL を Github のマニフェストのリポジトリの Webhook に設定します。

## 10. Webhook の設定 (red-hat-developer-sandbox-manifest)

1. リポジトリの画面を開いて、「Settings」をクリックします。
2. 左のメニューで「Webhooks」を選択します。
3. 「Add webhook」をクリックします。
4. 「Payload URL」に http://`<取得した URL>`を入力します。
5. 「Content type」を 「application/json」に設定します。
6. 「Add webhook」をクリックします。
7. 「Webhook」の「Recent Deliveries」で ping の履歴が表示されます。
8. GitOps Pipeline が動作しますが、namespace の変更はまだ Push されていないため、失敗します。気にしないでください。

## 11. マニフェストリポジトリへのプッシュ

1. 先ほど Namespace を変更するコミットをマニフェストのリポジトリにプッシュします。
2. Webhook が起動し、GitOps パイプラインが動作してすべてのマニフェストが適用されます。
3. パイプラインの実行状況は OpenShift Web Console で確認できます。「Pipelines」をクリックします。
4. 今度は GitOps pipeline の成功を確認します。

## 12. SonarQube の設定

1. gitopsのパイプラインが正常終了したら、SonarQube の URL を確認します。
```bash
oc get route sonarqube
```
2. https://<上記で取得した URL> をブラウザで開きます。「SonarQube is starting」画表示されているので、しばらく待ちます。
3. username:admin, password:admin でログインします。初期パスワードを任意のパスワードに変更します。
4. 右上に「Administration」をクリックし、「My Account」をクリックします。
5. 「Security」をクリックし、「Generate Tokens」の下に「Name」を入力して、「Type」を Global Analysis Token に設定して、「Generate」をクリックします。
6. 発行したトークンをアプリケーションリポジトリの下記ファイルに設定します。
   1. `red-hat-developer-sandbox-apps/apps/sample-app/build.gradle`の`sonar.login`の値を設定します。
   2. `red-hat-developer-sandbox-apps/apps/sample-view/sonar-project.properties`の`sonar.login`の値を設定します。

## 13. 各種サービスの URL を置き換える
1. `red-hat-developer-sandbox-apps/e2e/.env.ci`
   1. TARGET_PAGE_URL に`oc get route it-sample-view`で取得した URL を設定します。
   2. API_BASE_URL に`oc get route it-sample-app`で取得した URL を設定します。
2. `red-hat-developer-sandbox-apps/integration-test/sample-app/src/test/resources/application-it.properties`
   1. application.api.url に`oc get route it-sample-app`で取得した URL を設定します。
3. `red-hat-developer-sandbox-manifest/apps/overlays/it/patch/sample-view/deployment.yaml`
   1. API_BASE_URL の value に`oc get route it-sample-app`で取得した URL を設定します。
4. `red-hat-developer-sandbox-manifest/apps/overlays/dev/patch/sample-view/deployment.yaml`
   1. API_BASE_URL の value に`oc get route dev-sample-app`で取得した URL を設定します。

## 14. Event Listener の URL 設定 (apps-pipeline)

1. 以下のコマンドを実行して、Event Listener の URL を取得します。

```bash
oc get route el-sample-app-event-listener-dev
```

2. HOST/PORT の列に` el-sample-app-event-listener-dev-jiadchen-dev.apps.sandbox-m2.ll9k.p1.openshiftapps.com`のような URL が表示されます。
3. 取得した URL を Github の apps のリポジトリの Webhook に設定します。

## 15. WebHook の設定 (red-hat-developer-sandbox-apps)

1. リポジトリの画面を開いて、「Settings」をクリックします。
2. 左のメニューで「Webhooks」を選択します。
3. 「Add webhook」をクリックします。
4. 「Payload URL」に http://`<取得した URL>`を入力します。
5. 「Content type」を 「application/json」に設定します。
6. 「Add webhook」をクリックします。
7. Apps Pipeline が動作しますが、更新が足りないため、失敗します。気にしないでください。

## 16. デモアプリのデプロイ

1. 7と13での変更を Git にコミットします。
2. red-hat-developer-sandbox-manifest を Git Push します。
3. GitOps パイプラインが実行され、成功を確認します。
4. red-hat-developer-sandbox-apps を Git Push します。
5. Apps パイプラインが実行され、成功を確認します。

## 14. テストレポートの確認

1. テストレポートの URL を確認し、ブラウザでアクセスします。
```bash
oc get route test-reports
```
2. sample-app の単体テストレポート: <テストレポートの URL>/sample-app/unit-test
3. sample-app の結合テストレポート: <テストレポートの URL>/sample-app/integration-test
4. sample-view の単体テストレポート: <テストレポートの URL>/sample-view/unit-test
5. sample-view の結合テストレポート: <テストレポートの URL>/sample-view/storybook
6. e2e テストレポート: <テストレポートの URL>/e2e

## 15. デモアプリの動作確認
1. <`oc get route dev-sample-view`で取得した URL>/hello にアクセスし、デモ画面が表示されます。
2. 「API Call」をクリックし、API からのレスポンスが下に表示されます。`Hello, openshift! version:1.0.46`

## トラブルシューティング
1. sandbox の制約として pod を 50 個に達していると、pod の作成ができなくなるので、pod の削除が必要です。  
Completed の状態の Pod を全部削除するコマンド  
`oc get pods -n jiadchen-dev --field-selector=status.phase=Succeeded -o jsonpath='{.items[*].metadata.name}' | xargs -r oc delete pod -n jiadchen-dev`

2. pipelineRun がたくさん増えると、pod数の上限に達してしまったり、pvのサイズが40GBを超えたりするため、pipelinerun を削除する必要があります。  
すべての pipelinerun を削除するコマンド  
`oc delete pipelinerun --all -n jiadchen-dev`