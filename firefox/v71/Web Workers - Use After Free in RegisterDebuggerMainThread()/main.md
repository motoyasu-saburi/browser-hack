# Web Workers - Use After Free in RegisterDebuggerMainThread()

https://bugzilla.mozilla.org/show_bug.cgi?id=1546331

ネストされたワーカーを使用すると、ワーカーの破棄中に解放後使用が発生する可能性があります。これにより、悪用可能なクラッシュが発生する可能性があります。
