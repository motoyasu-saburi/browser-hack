# Race condition in firefox!sandbox::CopyPolicyToTarget leading to Information Disclosure of broker heap address

https://bugzilla.mozilla.org/show_bug.cgi?id=1599008

Windows クライアントのみ。
Firefox Sandbox 実装の共有メモリに２つの Race Condition が見つかった。

* CopyPolicyToTarget でのブローカーヒープアドレスの情報開示 (このレポートの内容)
* sandbox::SharedMemIPCServer::ThreadPingEventReady のメモリ破損（サンドボックスエスケープ）（別レポート）

Firefox (70.0.1 でテスト済み) render sandbox が USER_LIMITED と `初期整合性レベル(Initial Integrity Level) == 遅延整合性レベル (Delayed Integrity Level | INTEGRITY_LEVEL_LOW)` で設定されているため、
新しいレンダープロセスが起動している間 (新しいタブプロセスが生成されたとき) であっても、どのレンダープロセスも他のレンダープロセス (読み込み/書き込み/作成/リード/重複処理) と対話することができます。

（つまりrender sandbox が `初期統合性レベル` と `遅延整合性レベル` で同値の場合、
新しいレンダープロセスだろうと、古いレンダープロセスだろうが sandbox が同じと解釈されるということでいいのかな？）

Google Chrome is not affected because the renderer processes runs with a USER_LOCKDOWN token (also the initial integrity level = LOW and delayed IL is UNTRUSTED) which prevent access to other renderer processes.

Google Chromeの場合は、 renderer processes が USER_LOCKDOWN トークン（ `初期統合性レベル = LOW` で `Delayed Integrity Level = UNTRUSTED` )で実行されている。
そのため、他のレンダラープロセスへのアクセスができないため、影響を受けない。

Race Condition の脆弱性は、新しいプロセスのブートストラップ中に発生し、新しいタブプロセスが作成されると、ブローカーは次のように実行する

1. CreateProcess はロックダウントークン(Firefox レンダラの場合は USER_LIMITED)でSUSPENDED、TargetProcess::CreateではILが低いため、TargetProcess::Createを実行します。
2. JOBオブジェクトに対象プロセスを割り当てる
3. ターゲットスレッドのトークンを初期化用の初期トークン(USER_RESTRICTED_SAME_ACCESS)に変更する。
4. TargetProcess::InitでIPCServer共有メモリを作成する
5. TargetProcess::Initでターゲットとの共有メモリをDuplicateHandleします。
6. CopyPolicyToTargetでポリシーをコピーし、ポインタを更新する
