# Shrine ローカル起動手順

Shrine をビルドし、エミュレータで動かす手順です。接続先は、公開デモサーバーか、ローカルで動かす project-sesame から選べます。2026-11-09 と 2026-11-11 に動作を確認しました。

サーバー側の手順は `~/Workspace/project-sesame/LOCAL_SETUP.md` を参照してください。

## 必要なもの

- Android Studio（Android SDK）
- JDK 17（`brew install openjdk@17`）
  - Gradle 8.13 は、Android Studio に同梱されている JBR 25 では動きません。コマンドラインでビルドするときは `JAVA_HOME` に JDK 17 を指定します。Android Studio でビルドするときは、Settings > Build, Execution, Deployment > Build Tools > Gradle の Gradle JDK を 17 にします。
- `local.properties` に `sdk.dir=/Users/<ユーザー名>/Library/Android/sdk` を書く（git の管理対象外）

## 1. ビルドする

### 公開デモサーバー（project-sesame-426206.appspot.com）につなぐ場合

```sh
cd ~/Workspace/android/identity-samples/Shrine
JAVA_HOME=/opt/homebrew/opt/openjdk@17 ./gradlew assembleDebug
```

注意: 2026-11 時点では、公開サーバーでの Sign in with Google は失敗します。サーバー側で `/federation/options` が nonce を返さなくなったためです。

### ローカルの project-sesame につなぐ場合

先にサーバー側の手順で Sesame とトンネルを起動し、トンネルのホスト名を `-PshrineServerDomain` に指定します。

```sh
cd ~/Workspace/android/identity-samples/Shrine
JAVA_HOME=/opt/homebrew/opt/openjdk@17 ./gradlew assembleDebug \
  -PshrineServerDomain=xxxx.trycloudflare.com
```

- `shrineServerDomain` は `app/build.gradle.kts` に追加したプロパティです。省略すると公開サーバー向けになります。
- トンネルの URL は起動のたびに変わります。そのたびにビルドし直してください。
- 接続先は、生成された `app/build/generated/source/buildConfig/debug/com/authentication/shrine/BuildConfig.java` の `API_BASE_URL` で確認できます。

## 2. エミュレータにインストールする

使うのはエミュレータ `Medium_Phone_API_37.0` です。Android Studio の Device Manager から起動します。CLI から起動すると、Running Devices に取り込めません。

```sh
ADB=~/Library/Android/sdk/platform-tools/adb
$ADB install --user 0 -r app/build/outputs/apk/debug/app-debug.apk
```

- このエミュレータには仕事用プロファイル（user 10）があります。`--user 0` を付けずに入れると両方のプロファイルに入り、仕事用のほうが起動してしまうことがあります。Android Studio の Run（`installDebug`）でも同じことが起きます。
- パスキーを使うには、**個人プロファイル**に Google アカウントを追加し、画面ロック（PIN など）を設定しておく必要があります。

## 3. Restore Credentials を確認する

実機（moto g66j）では、Android Studio の Backup App Data が OS 側のエラー（READ_CONTENT_FAILED）で失敗しました。エミュレータを使ってください。

1. Shrine で新規登録する（このときリストアキーが作られる）。
2. Android Studio で Backup App Data（Device to Device）を実行する。`Shrine/application.backup` ができる。
3. アンインストールして入れ直す。
   ```sh
   $ADB uninstall com.authentication.shrine
   $ADB install --user 0 app/build/outputs/apk/debug/app-debug.apk
   ```
4. Android Studio で Restore App Data を実行し、手順 2 のバックアップを選ぶ。**アプリはまだ起動しない。**
5. Shrine を起動する。

| バックアップしたときの状態 | 起動後 |
|---|---|
| サインインしたまま | リストアキーで自動ログインし、ホーム画面になる |
| サインアウトした後 | サインイン画面になる（サインアウトするとリストアキーが消えるため） |

補足:
- リストアキーが作られるのは、新規登録したときと、パスワードでサインインしたときです。パスキーでのサインインでは作られません。
- 何も設定しないと、Auto Backup でセッション（`files/datastore/auth.preferences_pb`）まで復元されます。そうなると、リストアキーを使わずにログインした状態になります。これを防ぐため、作業ツリーで次のファイルを変更しています。
  - `app/src/main/res/xml/data_extraction_rules.xml`
  - `app/src/main/res/xml/backup_rules.xml`
  - `AndroidManifest.xml`
- `application.backup` にはセッション情報が含まれることがあるので、コミットしないでください。確認が終わったら削除します。
- ログで確かめたいときは `$ADB logcat | grep -iE "Blockstore|restore cred"` を実行します。
