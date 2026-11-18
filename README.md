# Raspberry Pi の電話交換局

JJYタイムサーバーと同じ Raspberry Pi 4 上に、Asterisk を使う電話交換局を設けるための設定例です。まず Tailscale 上のソフトフォン2台で内線通話を確認します。東京広域電話網への局間接続は、接続先の管理者との調整後に追加します。

## 現状

- 実機は Debian 13（trixie、arm64）です。Asterisk 22.12.0、FreePBX 17 と Tailscale を導入し、`asterisk-voip`、`apache2`、`chrony`、`jjyd` が稼働しています。FreePBX 17 の公式対象は Debian 12／amd64 であり、この構成は対象外です。
- 内線番号 `1001`、`1002` は局内テスト用です。電話網の局番ではありません。
- 実機の APT には Asterisk 本体の候補がありません。公式の Asterisk 22.12.0 のソースを `~/voip-build` に取得し、公開 SHA-256 と一致することを確認してビルドしました。ソースからの導入のため、Asterisk の更新は別途必要です。
- `chrony`、`jjyd`、GPIO と時刻設定は変更していません。ただし PBX の CPU・I/O 負荷が JJY の送信タイミングに与える影響は実測が必要です。

## FreePBX の管理画面

Tailscale に接続した端末で `http://100.114.71.89:8080/admin/` を開きます。初回は管理者を作成してください。Apache はこの Tailscale アドレスに限定して待ち受け、既存の ntfy の 80・443 番を使いません。通信は Tailscale 上で保護します。Tailscale のアクセス制御も信頼できる端末に限定してください。

内線 1001・1002 は FreePBX に移行済みです。FreePBX が生成する `/etc/asterisk/pjsip.conf` や `extensions.conf` を直接編集しないでください。RTP のアドレス指定は `/etc/asterisk/pjsip.endpoint_custom_post.conf` にあり、ひな形は [pjsip.endpoint_custom_post.conf.example](pjsip.endpoint_custom_post.conf.example) です。SIP のバインド先は FreePBX の SIP Settings で管理します。AMI は `127.0.0.1:5038` に限定しています。

Asterisk の移行前バックアップは Pi の `/root/freepbx-migration/asterisk-before.tar.gz` です。秘密情報を含みます。FreePBX には MariaDB、Apache、Sury の PHP 8.2 と PM2 を追加しています。Asterisk と PHP はそれぞれ更新状況を確認してください。PM2 の依存パッケージには npm audit で高 8 件・中 4 件の警告があり、互換性を確認せずに自動修正していません。

## 電話計算サービス

東京広域電話網から `6832110`、やがみ局の内線から `2110` にかけると、ボタンで入力した式の答えを音声で聞けます。詳しい操作は[電話計算サービスの案内](dials.md)をご覧ください。

## この Pi で内線を使う

各ソフトフォンを Pi と同じ Tailscale ネットワークに接続します。Pi の `~/voip-build/phone-credentials.txt` に SIP サーバーアドレスと内線 1001・1002 のパスワードがあります。Pi に SSH 接続して `cat ~/voip-build/phone-credentials.txt` で確認し、別々の端末に設定してください。ソフトフォンでは UDP、コーデックは PCMU または PCMA を使います。内線間で発着信して双方向の音声を確認します。

内線の SIP 登録と登録解除は FreePBX への移行後にも Pi 上の試験で成功しました。Android の内線 1001 と Windows の内線 1002 の実端末で通話テスト済みです。通話中の JJY 受信は未確認です。Pi の Tailscale アドレスが変わった場合は、FreePBX の SIP バインド先、RTP の `media_address`、Apache の待ち受けアドレスを更新し、各サービスを再起動してください。

代表番号 `1000` は FreePBX の Ring Groups で管理します。音声案内を再生した後、1001 と 1002 を同時に30秒間呼び出し、応答がなければ切断します。録音は System Recordings の「やがみ局 代表案内」、音声ファイルは `/var/lib/asterisk/sounds/custom/yagami-greeting.wav` です。案内音声は Kubernetes 上の VOICEVOX で作成しました（VOICEVOX:四国めたん）。案内文は「お電話ありがとうございます。やがみ局です。担当者をお呼びしますので、しばらくお待ちください。VOICEVOX、四国めたんがご案内しました。」です。FreePBX の `LAUNCH_AGI_AS_FASTAGI` は無効にして、同時呼び出しに必要な AGI をローカルで起動します。

状態確認には `sudo systemctl status asterisk-voip apache2 chrony jjyd ntfy`、`sudo asterisk -rx 'pjsip show endpoints'`、`chronyc tracking` を使います。サービスは [asterisk-voip.service](asterisk-voip.service) と [Apache の追加設定](apache2-tailscale.conf) です。不要な IAX2・UNISTIM・DUNDi は FreePBX の Asterisk Modules 設定で無効にしています。

## 設定例の使い方

以下は FreePBX 移行前の、PJSIP が動作する単独の Asterisk に追加する設定例です。現在の Pi は FreePBX が管理しているため、この手順を適用しないでください。

1. Tailscale に Pi とソフトフォンを接続し、相互通信を許可します。Pi で `tailscale ip -4` を実行し、[pjsip.conf.example](pjsip.conf.example) の `bind` と両方の `media_address` を実際の IPv4 アドレスに変更します。`100.64.0.1` は例示用です。
2. 内線ごとに別々の十分長いランダムパスワードを生成し、設定例の `password` を置き換えます。完成した設定は秘密情報として管理し、公開リポジトリへコミットしないでください。
3. 編集済みの設定を Asterisk の設定ディレクトリに配置し、既存設定から `#include` します。設定の所有者と読み取り権限は使用中の Asterisk に合わせ、無関係なユーザーがパスワードを読めないようにします。`extensions.conf.example` も同様に追加します。
4. Tailscale のアクセス制御とホストのファイアウォールで、信頼する端末に限って SIP の UDP 5060 と、Asterisk の `rtp.conf` で設定された RTP の UDP ポート範囲への通信を許可します。SIP/RTP をインターネットへ公開しないでください。設定例では SIP と RTP の両方を Tailscale アドレスにバインドします。
5. Asterisk を再起動して `sudo asterisk -rx 'pjsip show endpoints'` と `sudo asterisk -rx 'dialplan show local-phones'` を確認します。各ソフトフォンに Pi の Tailscale アドレス、内線番号、対応するパスワードを設定し、`1001` と `1002` の双方から発着信し、双方向の音声と切断を確認します。端末と Pi の間に NAT がある場合は音声経路の追加設定が必要なことがあります。
6. 通話中にも `chronyc tracking`、`systemctl status chrony jjyd`、`journalctl -u jjyd` を確認し、JJY の出力タイミングと時計の受信状態を実機で確認します。支障があれば PBX を停止して原因を調べます。

## 東京広域電話網への参加

[公式サイト](https://tkytel.github.io/)の参加窓口から交流し、接続先の局と Tailscale の共有方法、局番、発着信の番号変換、SIP トランクの接続情報を相談します。[MIKANETEL の接続案内](https://scrapbox.io/KusaReMKN/MIKANETEL%40%E6%9D%B1%E4%BA%AC%E5%BA%83%E5%9F%9F%E9%9B%BB%E8%A9%B1%E7%B6%B2%E3%81%A8%E6%8E%A5%E7%B6%9A%E3%81%99%E3%82%8B)も参照できます。相手と合意するまでは中継を有効にしないでください。

やがみ局の希望局番は `683`、設置位置は大阪府です。着信専用トランク `SIP-TRUNK-YAGAMI` は認証付きで最大 100 件の同時登録を受け、`1000` と `1002` に限り着信します。他局への発信ルートとして zin3、ab2m、SHINANOTEL、MikaNeTEL を設定しています。認証情報は Pi の `~/voip-build/trunk-credentials.txt` に保存しており、所有者のみ読み取れます。2件の同時登録、対象外番号の拒否、`1000` の案内音声送出と `1002` への呼出信号を Pi 上の試験で確認しました。実際の他局との接続と音声は未確認です。

[Mantela](https://shotayagami.github.io/yagami-mantela/mantela.json) は GitHub Pages で公開しており、公式 0.1.x スキーマで検証済みです。代表番号 `1000` は Android と PC の端末へ転送する `main` として記述しています。`providers` には zin3、ab2m、SHINANOTEL、MikaNeTEL を記載しています。公開リポジトリには局情報だけを置き、Tailscale 招待 URL や SIP 認証情報は含めていません。[discord-draft.md](discord-draft.md) は投稿文の下書きです。Pi の RTP ポート範囲は UDP 10000〜20000 です。

参考資料の [VoIP 網の構築記事](https://zenn.dev/kusaremkn/articles/abd760f9f2f450)は MikoPBX の局間接続例、[FBニュースの記事](https://www.fbnews.jp/202603/special/)は Pi での RasPBX 導入例です。[Mantela のコミット](https://github.com/Allianaab2m/mantela/commit/6bcae59ba6fa1251f22a266c4a583915b0b53e91)は札幌局の記述を削除した履歴であり、接続設定や利用可能な局番の証拠にはなりません。指定された [Qiita の記事](https://qiita.com/mofurune/items/34601bcff58e491d8d55)は Meta Quest への APK インストールについての記事でした。

