---
title: "Expo Imageを使ったら画像表示速度がバク上がりした話"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React Native", "Expo", "Tech"]
published: false
publication_name: "manalink_dev"
---
目次
1. 背景
2. アプリ側の実装と解決策
3. Expo Imageとは？
4. サーバー側の実装
5. どれくらい早くなったのか？
6. おわりに

**※今回の測定に関しては、弊社のネットワーク速度の影響もありますので、必ずしもこれぐらいの速さになるということはありません。そちらをご了承の上お読みいただければと思います。**

## 1. 背景
弊社のアプリには、チャット機能があるのですが、そこで送られた画像はモーダルとして見て、拡大表示して見たりすることが可能です。サービスの特性上、生徒様の送った問題の写真などが多くなるので、ユーザー体験から考えると、この機能はとても重要となります。ですが、ユーザーの方から「モーダルとして画像を表示するのに時間がかかる」といったお問い合わせがありました。

## 2. アプリ側の実装と解決策
弊社では、[react-native-image-zoom-viewer](https://github.com/siimorasmae/react-native-image-zoom-viewer)というライブラリを使用して、画像をモーダルとして見たり、拡大表示する機能を実現させています。
解決策に関しては2つあります。1つ目はこのライブラリは、下記のサンプルコードのように画像コンポーネントを渡すことができるのですが、そこにExpo ImageのImageコンポーネントを渡すことで、画像をモーダルで開く際の表示速度の改善が見られました。。
```tsx:Images.tsx
import { Image as ExpoImage } from 'expo-image';
import ImageViewer from 'react-native-image-zoom-viewer';

<ImageViewer
  // imgixを用いて、画像をリサイズする。
  imageUrls={props.imageUrls.map((m) => ({ url: getResizedImageForImageModal(m) }))}
  renderImage={(props) => {
    return <ExpoImage source={props.source.uri} style={props.style} />;
  }}
 />
```
2つ目は、Expo Imageの提供するprefetchです。**prefetchは**、画像を事前に読み込み、キャッシュする機能のことです。下記のサンプルコードのようなprefetchをチャットの吹き出しを表示する際に実行することで、画像をモーダルで開く際の表示速度の改善が見られました。
```ts:usePrefetchImage.ts
import { Image as ExpoImage } from 'expo-image';

const prefetchImage = useCallback(async (url: string[] | undefined) => {
    for (const imageUrl of url ?? []) {
      const modifiedUrl = replaceLocalhostToDebuggerHost(imageUrl);
      // リサイズなどをしている場合、リサイズ後のURLを渡す必要があるので、注意。
      await ExpoImage.prefetch(getResizedImageForImageModal(modifiedUrl));
    }
  }, []);
```

## 3. Expo Imageとは？
Expo Imageは、Expo SDK 48以降に含まれているコンポーネントで、画像を表示するためのコンポーネントです。今までは、Expo Managed Workflowを使っている場合は、React NativeのImageコンポーネントを使っていました。
ですが、React NativeのImageコンポーネントには画像をモーダルで開く際の表示速度が遅いという問題がありました。その点Expo Imageは、公式ドキュメントに書かれているように、Expo Imageはスピード重視の設計になっているので、その点でReact NativeのImageコンポーネントよりも優れていると言えます。
https://docs.expo.dev/versions/v48.0.0/sdk/image/

## 4. サーバー側の実装
次にサーバー側の実装です。上記の実装のみでも十分だと考えますが、元々のサイズが大きい画像を表示するとなった場合、画像をモーダルで開く際の表示速度が遅くなってしまう可能性があります。そのため、画像をアップロードする際にも圧縮処理をサーバー側で実装することにしました。ですが、サーバー側で画像の圧縮を行うと、将来的なメモリの枯渇などの問題が発生する可能性があります。下記が圧縮処理の部分の実装です。
```php:uploadImage.php
$interventionImage = InterventionImage::make($image)->orientate()->resize(1200, 1200, function ($constraint) {
            $constraint->aspectRatio();
            $constraint->upsize();
        })->save($tempImagePath);

        $uploadedImage = in_array($image->getMimeType(), ['image/jpeg', 'image/png']) && $isCompress ? new UploadedFile(
            $tempImagePath,
            $interventionImage->basename,
            $interventionImage->mime()
        ) : $image;
```
Intervention Imageを使って画像サイズの圧縮を行っています。Intervention Imageに関しては、下記の記事が参考になりました。
https://qiita.com/fakefurcoronet/items/fe2861ca2846b7347418
https://zakkuri.life/laravel-intervention-image/

## 5. どれくらい早くなったのか？
実際に上記の実装を行う前と行った後で、画像をモーダルで開く際の表示速度を計測してみました。計測方法は、人力でストップウォッチを使って朝(10:00ぐらい)、昼(13:00ぐらい)、夜(18:00ぐらい)の計3回計測しました。4Gかつ、20MB、10MB、300KBの画像を、Expo 47でReact NativeのImageコンポーネントを使っているバージョンと、Expo 48でExpo Imageを使っているバージョンでそれぞれ5回ずつ送信し、その表示速度の平均を取り、比較しました。4Gで計測している理由は、Wi-Fiの場合は、画像をモーダルで開く際の表示速度が早すぎて、あまり参考にならなかったためです。
結果に関しては、長くなるため20MBのときの計測結果だけ載せます。

### Expo Imageを使用していないバージョン
朝: (3 + 3.1 + 2.5 + 2.8 + 3) / 5 = 平均**2.88**秒

昼: (3 + 2.5 + 2.5 + 2.6 + 2.3) / 5 = 平均**2.58**秒

夜: (1.5 + 1.9+ 4.3 + 1.7 + 16) / 5 = 平均**5.08**秒

### Expo Imageを使用しているバージョン
朝: ( 0.6 + 0.7 + 0.6 + 0.6 + 0.6 ) / 5 = 平均**0.62**秒

昼: (0.7 + 0.6 + 0.5 + 0.6 + 0.5) / 5 = 平均**0.58**秒

夜: ( 0.8 + 0.7 + 0.6 + 0.6 + 0.6 ) / 5 = 平均**0.66**秒

上記の結果から分かるように、Expo Imageを使うことで、画像の表示速度が大幅に改善されました。一回一回の開く速度も上がっているのはもちろんですが、表示速度にほとんどブレがないのが素晴らしいです。


## 6. おわりに
今回の場合、画像表示コンポーネントに問題があったこともありますが、アップロードされた画像サイズが大きすぎて、画像をモーダルで開く際の表示速度が遅くなっていたことも原因の1つでした。このようなパフォーマンス改善は複合的にやらなければいけないのだと良い学びになりました。

