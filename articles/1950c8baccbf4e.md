---
title: "【React Native】inverted を true にした FlatList でフリーズする不具合を修正した話"
emoji: "🌟"
type: "tech"
topics:
  - "android"
  - "reactnative"
published: true
published_at: "2023-03-22 12:38"
publication_name: "aldagram_tech"
---

こんにちは！アルダグラムでエンジニアをしている渡邊です！

弊社の KANNA アプリではチャット機能があり、ユーザー同士でチャットのやり取りが行えます。
ただ、そのチャット画面でしばらくするとスクロール動作が遅くなってしまい操作がまともにできない、という不具合の問い合わせがありました。
調査した結果不具合を修正することができたので、その内容について共有いたします。
（ただし今回記載した修正方法を適用する場合、あくまで自己責任でお願いします）

# 原因

結論から言うと、Android 13 の端末で FlatList の inverted が true の場合にフリーズすることがある、ということでした。
ただ必ず発生するわけではなく、どうやら inverted が true の FlatList の要素の中でアニメーションを行っていると発生するようです。
こちらについては React Native のプロジェクトに Issue が上がっていました。
https://github.com/facebook/react-native/issues/34583

ただ私個人の端末が Android 13 の Pixel 6 なのですが、この端末では発生しませんでした。
ユーザーから報告があった端末と同じ端末である Galaxy S21 を会社にお願いして購入いただいたので、こちらで確認したところ不具合を確認することができました。
（端末を購入いただき、ありがとうございました！）

# 解決方法

KANNA アプリではチャットの表示に [react-native-gifted-chat](https://github.com/FaridSafi/react-native-gifted-chat) というライブラリを使っており、このライブラリが inverted が true の FlatList を使っています。
この react-native-gifted-chat にも Issue が上がっており、まずは [こちら](https://github.com/FaridSafi/react-native-gifted-chat/issues/2335#issuecomment-1418912420) に記載されている解決方法を見つけました。

この解決方法は React Native の VirtualizedList という FlatList の内部で使われているコンポーネントに [patch-package](https://github.com/ds300/patch-package) でパッチを当てるというものです。
修正方法を見る限り、もともと VirtualizedList では inverted が true の場合スタイルの transform で scaleY を -1 に設定しているようですが、これを rotate で180度回転させるように変更していました。

```diff
 const styles = StyleSheet.create({
-  verticallyInverted: {
-    transform: [{scaleY: -1}],
-  },
+  verticallyInverted:      
+    Platform.OS === "android"
+    ? {transform: [{ rotate: '180deg' }]}
+    : {transform: [{ scaleY: -1 }]},
   horizontallyInverted: {
     transform: [{scaleX: -1}],
   },
```

この方法を試してみたところ、確かにフリーズすることはなくなりましたが、**スクロールバーが右側ではなく左側に表示されてしまう**ようになりました。
最悪スクロールバーを非表示にすれば解決はするのですが、別の解決方法が先ほどのコメントの下にあったのでこちらも見てみました。
https://github.com/FaridSafi/react-native-gifted-chat/issues/2335#issuecomment-1451589454

```diff
diff --git a/node_modules/react-native-gifted-chat/.DS_Store b/node_modules/react-native-gifted-chat/.DS_Store
new file mode 100644
index 0000000..e69de29
diff --git a/node_modules/react-native-gifted-chat/lib/MessageContainer.js b/node_modules/react-native-gifted-chat/lib/MessageContainer.js
index 6bdf6da..2aed446 100644
--- a/node_modules/react-native-gifted-chat/lib/MessageContainer.js
+++ b/node_modules/react-native-gifted-chat/lib/MessageContainer.js
@@ -168,6 +168,9 @@ export default class MessageContainer extends React.PureComponent {
             }
         };
         this.keyExtractor = (item) => `${item._id}`;
+        const version = parseFloat(Platform.Version);
+        this.renderMessageContainer = Platform.OS === 'android' && version >= 33 ? this.renderMessageContainerWithAndroidFix.bind(this) : this.renderRegularMessageContainer.bind(this);
+        this.invert = (orignialRenderFunction) => (props) => <View style={{scaleY: -1}}>{orignialRenderFunction(props)}</View>;
     }
     scrollTo(options) {
         if (this.props.forwardRef && this.props.forwardRef.current && options) {
@@ -189,10 +192,17 @@ export default class MessageContainer extends React.PureComponent {
         </TouchableOpacity>
       </View>);
     }
-    render() {
+    renderMessageContainerWithAndroidFix() {
+        const { inverted } = this.props;
+        return (<FlatList ref={this.props.forwardRef} extraData={[this.props.extraData, this.props.isTyping]} keyExtractor={this.keyExtractor} enableEmptySections automaticallyAdjustContentInsets={false} data={this.props.messages} style={[styles.listStyle, inverted ? {scaleY: -1} : null]} contentContainerStyle={styles.contentContainerStyle} renderItem={inverted ? this.invert(this.renderRow) : this.renderRow} {...this.props.invertibleScrollViewProps} ListEmptyComponent={this.renderChatEmpty} ListFooterComponent={inverted ? this.invert(this.renderHeaderWrapper) : this.renderFooter} ListHeaderComponent={inverted ? this.invert(this.renderFooter) : this.renderHeaderWrapper} onScroll={this.handleOnScroll} scrollEventThrottle={100} onLayout={this.onLayoutList} onEndReached={this.onEndReached} onEndReachedThreshold={0.1} {...this.props.listViewProps} inverted={false} />);
+    }
+    renderRegularMessageContainer() {
         const { inverted } = this.props;
+        return (<FlatList ref={this.props.forwardRef} extraData={[this.props.extraData, this.props.isTyping]} keyExtractor={this.keyExtractor} enableEmptySections automaticallyAdjustContentInsets={false} inverted={inverted} data={this.props.messages} style={styles.listStyle} contentContainerStyle={styles.contentContainerStyle} renderItem={this.renderRow} {...this.props.invertibleScrollViewProps} ListEmptyComponent={this.renderChatEmpty} ListFooterComponent={inverted ? this.renderHeaderWrapper : this.renderFooter} ListHeaderComponent={inverted ? this.renderFooter : this.renderHeaderWrapper} onScroll={this.handleOnScroll} scrollEventThrottle={100} onLayout={this.onLayoutList} onEndReached={this.onEndReached} onEndReachedThreshold={0.1} {...this.props.listViewProps}/>);
+    }
+    render() {
         return (<View style={this.props.alignTop ? styles.containerAlignTop : styles.container}>
-        <FlatList ref={this.props.forwardRef} extraData={[this.props.extraData, this.props.isTyping]} keyExtractor={this.keyExtractor} enableEmptySections automaticallyAdjustContentInsets={false} inverted={inverted} data={this.props.messages} style={styles.listStyle} contentContainerStyle={styles.contentContainerStyle} renderItem={this.renderRow} {...this.props.invertibleScrollViewProps} ListEmptyComponent={this.renderChatEmpty} ListFooterComponent={inverted ? this.renderHeaderWrapper : this.renderFooter} ListHeaderComponent={inverted ? this.renderFooter : this.renderHeaderWrapper} onScroll={this.handleOnScroll} scrollEventThrottle={100} onLayout={this.onLayoutList} onEndReached={this.onEndReached} onEndReachedThreshold={0.1} {...this.props.listViewProps}/>
+        {this.renderMessageContainer()}
         {this.state.showScrollBottom && this.props.scrollToBottom
                 ? this.renderScrollToBottomWrapper()
                 : null}
```

これは react-native-gifted-chat にパッチを当てる方法で、コードを見ると Android 13 以上の場合だけ inverted が true の時に FlatList のスタイルの scaleY を -1 に設定するものでした。
ただこの設定を行うとリストの要素は下から表示されるのですが、リストの要素の表示が上下反転してしまうので、さらにリストの要素のコンポーネントに対してもスタイルで scaleY に -1 を設定しているようです。

こちらの方法を試してみたところ、フリーズすることもなく、動作確認してみた限りでは特に問題がありませんでした。
先ほどの方法は VirtualizedList に対してパッチを適用するので影響範囲が広くなってしまうのですが、こちらの方法は react-native-gifted-chat のコンポーネントにだけパッチを適用しているのと、Android 13 未満や iOS では今まで通りの方法なので、万が一何か問題があっても影響範囲が小さくなると思いこちらの方法を使って修正することにしました。

一つ注意点として、React Native のバージョンが 0.70.0 以上だとスタイルの scaleY はサポートされていないようです。
その場合ワークアラウンドとして以下のような記述を `index.js` に記述する旨が、先ほどの Issue のコメントに記載されていました。
（弊社のアプリはまだ React Native のバージョンが 0.70.0 未満なので、この点に関しては問題ありませんでした）

```javascript:index.js
import ViewReactNativeStyleAttributes from 'react-native/Libraries/Components/View/ReactNativeStyleAttributes'
ViewReactNativeStyleAttributes.scaleY = true
```

# 最後に

特定の OS のバージョンによって発生する不具合は原因の特定が難しいですね。
今回 GitHub の Issue のコメントを見つけなければ、解決することが難しかったかと思います。

この記事がどなたかの参考になれば幸いです！
