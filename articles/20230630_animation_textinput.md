---
title: "React NativeのTextInputの改行時の縦幅の広がりを滑らかにした話"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React Native","Tech"]
published: true
publication_name: "manalink_dev"
---
目次
1. 概要
1. 対象読者
1. 実際に縦幅の広がりを滑らかにしたコード
1. まとめ

## 概要
React Native + ExpoでTextInputを実装したい場合、React Nativeの公式から提供されているTextInputを使用することが多いと思います。以下の動画のBefore/Afterを見てください。いかがでしょうか。Afterを見ていただけると、改行したときのTextInputの縦幅の広がり方がぬるっと滑らかになったことが分かると思います。そこで今回は、React NativeのTextInputで改行するときに滑らかに縦幅が広がるようにした方法とサンプルコードを示していこうと思います。

### Before
![](https://storage.googleapis.com/zenn-user-upload/124686d0fab1-20230630.gif)


### After
※ GIFのフレームレートの影響で改善されていないように見えますが、実際はより滑らかになっています。
![](https://storage.googleapis.com/zenn-user-upload/d3c8ed83e21e-20230630.gif)


## 対象読者
- React Native初級者〜中級者の方
- React Native + Expoのアプリで、チャットを実装したい方
- React NativeのTextInputの縦幅の広がりが気になっていて、解消したい方


## 実際に縦幅の広がりを滑らかにしたコード
- 今回のコードは、[こちら](https://github.com/FaridSafi/react-native-gifted-chat/issues/1727#issuecomment-635623414)のイシューコメントを参考に実装しています。
- 上記のコードを参考に、私達の環境で実際に滑らかに改行させるようにした動きを実装しているのが、下記のコードになります。(このフックの呼び出し側は、TextInputのonContentSizeChangeに代入しているだけなので、省略します)

```ts:useCalculateInputField.ts
type ContentSizeType = {
  width: number;
  height: number;
};

type Props = {
  setInputHeight: React.Dispatch<React.SetStateAction<number>>;
};

export const useCalculateInputField = ({
  // TextInputの高さを動的に変更するために、setterを渡している。
  setInputHeight,
}: Props) => {
  // 改行時のアニメーションを実行し、TextInputの高さを変更する関数
  const calculateInputHeight = (contentSize: ContentSizeType) => {
    if (contentSize?.height) {
      // アニメーションを実行する
      LayoutAnimation.configureNext(CustomLayoutSpring);
      // contentSize.heightをそのままsetしてもいいが、それだと最初に入力した文字が見えなくなってしまったので、改行するごとに受け取ったheightに10足していく。
      setInputHeight(contentSize.height + 10);
    }
  };

  // contentSizeは入力値が変わるたびに変更されるため、それを元にTextInputの高さを動的に変更する。
  const onContentSizeChange = ({ nativeEvent: { contentSize } }) => {
    if (!contentSize) {
      return;
    }
    calculateInputHeight(contentSize);
  }

  return { onContentSizeChange };
};

const CustomLayoutSpring = {
  duration: 200,
  // 要素が作成された場合
  create: {
    type: LayoutAnimation.Types.easeOut,
    property: LayoutAnimation.Properties.opacity,
    springDamping: 0.7,
  },
  // 要素が変更された場合(今回のパターンは改行時など)
  update: {
    type: LayoutAnimation.Types.easeOut,
    springDamping: 1.0,
  },
  // 要素が削除された場合(今回のパターンは改行を消すとき)
  delete: {
    type: LayoutAnimation.Types.easeOut,
    property: LayoutAnimation.Properties.opacity,
    springDamping: 1.0,
  },
};
```
結論から言うと、改行時にアニメーションを適用することでデフォルトのTextInputの改行時の動きを無くすようにしています。今回はReact NativeのLayoutAnimationを用いて、アニメーションを実装しています。
LayoutAnimationは、簡単にアニメーションを実装するためのReact Nativeの公式から提供されているものです。詳しくは公式ドキュメントとこちらの記事が参考になると思います。

https://reactnative.dev/docs/layoutanimation

https://qiita.com/kaba/items/d2fb7b22822f7add19a3

今回のコードでアニメーションを設定している部分は、下記の部分です。durationやspringDampingの設定値は、私達の環境でうまく動作した値なので、適宜変更してください。

```:ts
const CustomLayoutSpring = {
  duration: 200,
  // 要素が作成された場合
  create: {
    type: LayoutAnimation.Types.easeOut,
    property: LayoutAnimation.Properties.opacity,
    springDamping: 0.7,
  },
  // 要素が変更された場合(今回のパターンは改行時など)
  update: {
    type: LayoutAnimation.Types.easeOut,
    springDamping: 1.0,
  },
  // 要素が削除された場合(今回のパターンは改行を消すとき)
  delete: {
    type: LayoutAnimation.Types.easeOut,
    property: LayoutAnimation.Properties.opacity,
    springDamping: 1.0,
  },
};
```
このように設定することで最初に添付した動画のように、デフォルトのTextInputの改行時の動きを滑らかにすることが可能です。


## まとめ
いかがでしたでしょうか。改行時の縦幅の広がりを滑らかにした結果、入力が滑らかになって使いやすくなった印象を受けると思います。この実装はチャットやフォームの入力欄を実装するときに役立つと思うので、ぜひ参考にしてみてください。
