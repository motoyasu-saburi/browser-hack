# Access heap-allocated array with index out of bounds in `nsPlainTextSerializer`

https://bugzilla.mozilla.org/show_bug.cgi?id=1584170

プレーンテキストシリアライザーは、固定サイズの配列を使用して
処理できる要素。ただし、静的サイズの配列をオーバーフローさせて、メモリの破損や潜在的に悪用可能なクラッシュを引き起こす可能性がありました。
