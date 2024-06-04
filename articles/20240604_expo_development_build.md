---
title: "Expoのdevelopment buildを導入したので、詰まったところを共有します"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React Native", "expo", "Tech"]
published: true
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

そこでdevelopment buildです。development buildは名前の通り「開発用にビルドされたアプリ」です。そのため、Expo Goと同じ感じで変更したら、その変更が反映されますし、Push通知なども来ます。これがメインのメリットではなく他にもメリットはありますので、詳しくは公式ドキュメントを読んでいただければと思います。

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

**やったこと**: このエラーの解決に一番時間がかかりました。そもそも、`expo-dev-menu`や`expo-dev-launcher`自体が、`expo-dev-client`用のライブラリなので、`expo-dev-client`のバージョンを操作して検証する必要がありました。やったこととしては、`expo-dev-client`を最新版に上げたり、`expo-dev-menu`や`expo-dev-launcher`のChangelogを見て、エラーが発生しているファイルに変更が入っている前後のバージョンに`expo-dev-client`をしたりなどです。

**解決方法**: 上記のようなことをやっても全く解決しませんでした(出る場所は違うが、エラーの内容はほぼ一緒)。そこで`expo-updates`が怪しいと感じました。理由としては他にいじった部分がないこと、エラーのはかれているファイルでExUpdatesというInterfaceが使われていたためです。最初に述べたように`expo-updates`のversionを特定のバージョン以上にすると、ビルドが失敗します。そのため一旦`npx expo install --check`で依存関係を整理してから、`expo-updates`のバージョンを上げると、ビルドは失敗しなくなりました。
**また、上記の`cannot find interface declaration`のエラーも出さずに、ビルドが成功するようになりました。**

## おわりに
今回expoのdevelopment buildを導入した際の詰まったとこを紹介しました。弊社の環境のみ起こっている可能性もありますが、今後development buildを導入する際の助けとなればと思います。
