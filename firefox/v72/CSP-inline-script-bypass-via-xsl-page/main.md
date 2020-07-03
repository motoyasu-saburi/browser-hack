# Content-Security-Policy inline script execution is bypassed on XSL pages

XSLのページでは、CSPの inline script ルールが無視される問題。

https://bugzilla.mozilla.org/show_bug.cgi?id=1597645

CSPの inline script を XSL page でバイパスする例

---------------------- 以下読んでない部分

https://traintimes.org.uk/firefox-csp/ は、私が見つけた主な問題を示しています。これは、インライン JavaScript がないという CSP ヘッダとともに送信された XML ファイルですが、Firefox で実行されるインライン JavaScript を含むテンプレートを含む XSL スタイルシートを含んでいます (インラインスクリプトとインラインイベントハンドラの両方)。

https://traintimes.org.uk/firefox-csp/html.php は XSL テンプレートと同じ HTML ですが、HTML として提供されており、ここではインライン JavaScript が Firefox で実行されないことがわかります。
Chromeでは、エラーが出ます。HTML 版と XML 版の両方で、"Refused to execute inline script because it violates the following Content Security Policy directive" というエラーが出ます。
