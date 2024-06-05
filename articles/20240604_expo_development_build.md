---
title: "Expoのdevelopment buildを導入したので、詰まったところを共有します"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React Native", "expo", "Tech"]
published: false
publication_name: "manalink_dev"
---

## 対象読者
- Expoのdevelopment buildを導入しようとしている方
- Expoのdevelopment buildで詰まっている方

## 環境
- Expo SDK: 50.0.19
- React Native: 0.73.6
- Expo Workflow: Managed Workflow

## Expoのdevelopment buildとは？
React Native + Expoで構成されているアプリの場合、開発する際の動作確認やスタイリングの確認にはSimulatorかExpo Goを使用すると思います。
もし実機で確認したいとなるとExpo Goに限られるのですが、Expo SDK 50以降はSingle Version Supportになったため、最新版にアプデしなければExpo Goでは動作確認ができなくなりました。

そこで候補として上がるのが、development buildです。development buildは名前の通り「開発用にビルドされたアプリ」です。そのため、Expo Goと同じ感じで変更したら、その変更が反映されますし、Push通知なども来ます。これがメインのメリットではなく他にもメリットはありますので、詳しくは公式ドキュメントを読んでいただければと思います。

https://docs.expo.dev/develop/development-builds/create-a-build/

https://docs.expo.dev/develop/development-builds/introduction/

## development build導入の際に詰まったとこ
### 1. そもそもbuildコマンドが実行されない！
**起こったこと**:　`eas build --profile development --platform ios`を実行すると、`Cannot destructure property 'expoUsername' of 'undefined' as it is undefined.`というエラーが出て、ビルドが失敗する

**解決方法**: この[issue](https://github.com/expo/expo/issues/25894)と起きている状況と同じだったため、コメントの内容を一つ一つ試していきました。弊社の環境でうまくいったのは、`expo-updates`のバージョンを下げるというものでした(0.18.4に下げた)。正直このエラーと`expo-updates`がどう関係しているのかはわかっていないため、また何か分かり次第追記させていただきます。

### 2. React Native Firebaseでエラーが発生する
**起こったこと**:　`eas build --profile development --platform ios`を実行後、fastlane部分で画像のようなエラーが出る。
![](https://storage.googleapis.com/zenn-user-upload/639814d993f4-20240604.png)

**解決方法**: build画面に`Possible React Native Firebase configuration issue`と書かれていた。弊社のアプリでは、React Native Firebaseを使用していなかったので削除し、再度ビルドを走らせたところエラーは解消した。参考までにExpoでReact Native Firebaseを使うためのドキュメントを貼っておきます。

https://rnfirebase.io/#expo

### 3. expo-dev-clientの依存関係ライブラリでの`cannnot find interface declaration`エラーの発生
**起こったこと**:　`eas build --profile development --platform ios`を実行すると、`expo-dev-client`の依存関係先である`expo-dev-menu`や`expo-dev-launcher`で`cannot find interface declaration for 'RCTRootViewFactory'`のようなエラーが起こり、ビルドが失敗する

**解決方法**: 上記のようなことをやっても全く解決しませんでした(出る場所は違うが、エラーの内容はほぼ一緒)。そこで`expo-updates`が怪しいと感じました。理由としては他にいじった部分がないこと、エラーのはかれているファイルでExUpdatesというInterfaceが使われていたためです。最初に述べたように`expo-updates`のversionを特定のバージョン以上にすると、ビルドが失敗します。そのため一旦`npx expo install --check`で依存関係を整理してから、`expo-updates`のバージョンを上げると、ビルドは失敗しなくなりました。

その後再度ビルドを走らせたところ、上記の`cannot find interface declaration`のエラーも発生することなく、ビルドが成功するようになりました。

**注意**: この方法で必ずうまくいくわけではなく、「どのライブラリのせいで、expo-updatesのエラーが出ていたのか？」「不要だったライブラリのアップデートはどれか？」など分かっていない部分もありますので、候補の一つとして考えてもらえればと思います。

### 4. 実機で読み取っても、アプリを開くことができない
**起こったこと**:　ビルド成功後、QRコードを実機で読み込み、アプリをインストールしても開くことができない。
![](https://storage.googleapis.com/zenn-user-upload/bdc8dfaa45e3-20240605.png)

**解決方法**: `eas.json`に環境ごとの設定を追加できると思うのですが、そこで`simulator: true`にしていることが原因でした。設定値の通りこれを`true`にすると、simulator用のdevelopment buildになってしまうので、これを`false`にするか、そもそも書かないようにすることで、実機でも開くことができるようになります。

以下のように`eas.json`に新しいprofileを設定する方が良いかなと思います。

```json:eas.json
 "development-device": {
      "distribution": "internal",
      "developmentClient": true
    }
```

## おわりに
今回expoのdevelopment buildを導入した際の詰まったとこを紹介しました。弊社の環境のみ起こっている可能性もありますが、今後development buildを導入する際の助けとなればと思います。
