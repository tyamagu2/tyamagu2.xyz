+++
date = "2016-05-02T23:05:50+09:00"
draft = false
title = "Go で ping"

+++

Go で ping を作った。https://github.com/tyamagu2/go-ping

Raw Socket を使って ICMP の Echo Message を送り、Echo Reply Message を受け取っている。
またその際の RTT やパケロス率を計算して表示してくれる。IPv6 には対応していない。

以下、作成の記録。

### ping の仕組み

まずは ping の仕組みを調べるところから。おもむろに手元の MBP で `man ping` してマニュアルを開く。
以下 BSD System Manager's Manual PING(8) より抜粋。

```
NAME
     ping -- send ICMP ECHO_REQUEST packets to network hosts

...

ICMP PACKET DETAILS
    ....
     If the data space is at least eight bytes large, ping uses the first eight bytes of this space to include a timestamp which it uses in the computation of round trip times.  If
     less than eight bytes of pad are specified, no round trip times are given.
...
```

以下ざっくりと説明。

ICMP というのはそういうプロトコル。ping は ICMP の ECHO_REQUEST を相手に送る。
ICMP の詳細は後述するが、ECHO_REQUEST を送ったら ECHO_REPLY が返ってくる。ECHO なのでデータの中身は同じ。
そのデータの先頭 8 byte にタイムスタンプを含めておいて、RTT を計算する、という仕組みになっている。
データにタイムスタンプが含まれていることは RTT の計算に必須ではないのだけど、
こうしておけばステートレスにできて楽、ということだと思われる。

### ICMP

ICMP は Internet Control Mesage Protocol のこと。[RFC 792](https://tools.ietf.org/html/rfc792) で定義されている。
ホスト間のやり取り、例えばデータグラムの処理に関するエラーの報告などの用途に使うものとのこと。
あたかも IP の上位層のように IP の機能を使うけど、ICMP も IP の一部で、
IP を実装するモジュールは必ず ICMP も実装しないといけない。

ICMP には複数の Message が定義されている。主にエラー報告に利用されるだけあって、
`Destination Unreachable Message` とか `Time Exceeded Message` のようにエラーの内容を示す Message が多い。
ping で使うのはこのうちの `Echo Message` および `Echo Reply Message` ということになる。

ICMP Message は IP パケットのペイロードに乗っけることになる。
IP パケットのヘッダーにどのような値をセットすべきかについても RFC に記載がある。
特にポイントになるのは、Protocol フィールドに ICMP を表す 1 を指定することぐらい。

### Echo & Echo Reply Message のフォーマット

こうなっている。

```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |     Type      |     Code      |          Checksum             |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |           Identifier          |        Sequence Number        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |     Data ...
    +-+-+-+-+-
```

Type はどの ICMP Message かを表すフィールド。Echo だと 8、Echo Reply だと 0 になる。Code は 0 固定。
Checksum は Type から始まる ICMP Message 全体の Internet Checksum。計算方法は [RFC 1071](https://tools.ietf.org/html/rfc1071) に書いてある。

Identifier と Sequence Number は、Echo と Reply をマッチングするために使う。
例えば複数のプロセスが同じホストに Echo Message を送っていると、自分宛ての Reply かわからなくなる。
そのため、Identifier にプロセス固有の値を入れておくようにする。
また、同じプロセスから複数の Echo Message を送る場合、IP は到達保証も到達順序保証もしないので、
やはり Echo と Reply の対応がわからなくなる。そこで、Echo ごとに Sequence Number をインクリメントしておく。
というような使い方が一般的な模様。

### tcpdump で Echo & Echo Reply Message を見る

実際に見てみる。以下は上から順番に、1 つ目の Echo Message、1 つ目の Echo Reply Message、2 つ目の Echo Message、2 つ目の Echo Reply Message。

```
01:29:33.027715 IP 10.0.1.3 > pages.github.com: ICMP echo request, id 64652, seq 0, length 64
0x0000:  4500 0054 6cd7 0000 4001 4616 0a00 0103
0x0010:  c01e fc9a 0800 1e5a fc8c 0000 5726 2eed
0x0020:  0000 6c02 0809 0a0b 0c0d 0e0f 1011 1213
0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223
0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233
0x0050:  3435 3637
01:29:33.206876 IP pages.github.com > 10.0.1.3: ICMP echo reply, id 64652, seq 0, length 64
0x0000:  45c0 0054 a1fe 0000 3401 1c2f c01e fc9a
0x0010:  0a00 0103 0000 265a fc8c 0000 5726 2eed
0x0020:  0000 6c02 0809 0a0b 0c0d 0e0f 1011 1213
0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223
0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233
0x0050:  3435 3637
01:29:34.028842 IP 10.0.1.3 > pages.github.com: ICMP echo request, id 64652, seq 1, length 64
0x0000:  4500 0054 7ba2 0000 4001 374b 0a00 0103
0x0010:  c01e fc9a 0800 19f0 fc8c 0001 5726 2eee
0x0020:  0000 706a 0809 0a0b 0c0d 0e0f 1011 1213
0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223
0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233
0x0050:  3435 3637
01:29:34.207647 IP pages.github.com > 10.0.1.3: ICMP echo reply, id 64652, seq 1, length 64
0x0000:  45c0 0054 1ac1 0000 3401 a36c c01e fc9a
0x0010:  0a00 0103 0000 21f0 fc8c 0001 5726 2eee
0x0020:  0000 706a 0809 0a0b 0c0d 0e0f 1011 1213
0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223
0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233
0x0050:  3435 3637
```

0 〜 19 オクテットまでが IP ヘッダーで、20 オクテット目からが ICMP Message。

1 個目の Echo と Reply の ICMP ヘッダー以降を抜き出してみる。

```
0x0010:            0800 1e5a fc8c 0000 5726 2eed
0x0020:  0000 6c02 0809 0a0b 0c0d 0e0f 1011 1213
0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223
0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233
0x0050:  3435 3637

0x0010:            0000 265a fc8c 0000 5726 2eed
0x0020:  0000 6c02 0809 0a0b 0c0d 0e0f 1011 1213
0x0030:  1415 1617 1819 1a1b 1c1d 1e1f 2021 2223
0x0040:  2425 2627 2829 2a2b 2c2d 2e2f 3031 3233
0x0050:  3435 3637
```

0 オクテット目、つまり Type が Echo は 08、Reply は 00 になっている。
4, 5 オクテット目は ID で、上に載せた 2 個目の Echo/Reply とも一致している。
6, 7 オクテット目は Sequence Number で 0000。上の 2 個目の Echo/Reply では 0001 で確かにインクリメントされている。

8 オクテット目以降の Data は Echo/Reply で一致している事がわかる。
9 オクテット目からの 8 オクテットはタイムスタンプ担っていて、それ以降はどうやら単にインクリメントしたダミーデータらしい。

### Raw Socket

ping を作るには、ICMP の Echo Message を送れば良いことは分かった。次はどうやってアプリケーションから ICMP、というか IP パケットを送ればよいのかが疑問となる。

調べた結果、これには [Raw Socket](http://linux.die.net/man/7/raw) を使うことになる。
リンク先に書いてあるとおり、Raw Socket とは IP を使ったプロトコルをユーザースペースで実装するためのソケット。
IP を使ったプロトコルというのはつまり、ICMP とか TCP、UDP といったプロトコル。

通常のソケットでは Transport Layer のプロトコルを TCP とか UDP とか指定して使うけど、
Raw Socket を使えば直接 IP パケットを送受信できる。なので TCP とか UDP みたいなプロトコルを自分でつくる事ができる。
ping の場合は ICMP のパケットを自分で Raw Socket を介して送受信することになる（つまり ICMP の一部を自分で実装する）。

ちなみに、Raw Socket を使えるのは所有者の UID が 0（スーパーユーザー）または CAP_NET_RAW というケイパビリティがセットされたプロセスのみとなる。
じゃあ ping はなんで誰でも使えるのかというと、ping は所有者が root で setuid しているから。

それから、ちゃんとしたソースをまだ見つけられてないんだけど、Raw Socket じゃなくて UDP Socket（SOCK_RAW じゃなくて SOCK_DGRAM）を使えば、
スーパーユーザーじゃなくても Echo などの一部の ICMP メッセージを利用することが可能らしい。以下にそれらしきことが書いてある。
まあでも、今回は Raw Socket を使うことにする。

http://www.freebsd.org/cgi/man.cgi?query=icmp&apropos=0&sektion=0&manpath=Darwin+8.0.1%2Fppc&format=html
https://github.com/golang/net/blob/master/icmp/listen_posix.go#L25-L28

### Go で実装

いよいよ Go で実装する。

実は [github.com/golang/net](https://github.com/golang/net) には [icmp のパッケージ](https://github.com/golang/net/tree/master/icmp)もあるし、
[それを使って ping するテスト](https://github.com/golang/net/blob/master/icmp/ping_test.go)もある。
でもこれを使うと楽しみ半減なので、困ったときに参考にするだけに留める。

まず、Raw Socket を使うには [net.Dial](https://golang.org/pkg/net/#Dial) を使う。

```
conn, err := net.Dial("ip4:1", ip.String())
...
wn, err = conn.Write(mb)
...
rn, err := conn.Read(rb)
```

こんな感じ。`"ip4:1"` の `ip4` は IPv4、`1` はプロトコル番号で ICMP を指す。Write でデータを送信、Read で受け取る。

次に ICMP パケットの作成について。ネットワークバイトオーダー（= Big Endian）に変換するには、
[encoding/binary の ByteOrder インターフェース](https://golang.org/pkg/encoding/binary/#ByteOrder) を使う。
Checksum の計算には、上でも書いた [RFC 1071](https://tools.ietf.org/html/rfc1071) に C での実装方法が載っていたので参考にした。
下では省略しているけど、ID にはプロセス ID の下位 16 ビットを、Seq には 0 から順次インクリメントした番号を、
Data には [Time](https://golang.org/pkg/time/#Time) オブジェクトを Marshal したものをセットしている。

```
type Type uint8

const (
    ECHO_REPLY Type = 0
    ECHO       Type = 8
)

type EchoMessage struct {
    Type     Type
    Code     uint8
    Checksum uint16
    ID       uint16
    Seq      uint16
    Data     []byte
}

func (m *EchoMessage) Marshal() []byte {
    b := make([]byte, 8+len(m.Data))
    b[0] = byte(m.Type)
    b[1] = byte(m.Code)
    b[2] = 0
    b[3] = 0
    binary.BigEndian.PutUint16(b[4:6], m.ID)
    binary.BigEndian.PutUint16(b[6:8], m.Seq)
    copy(b[8:], m.Data)
    cs := checksum(b)
    b[2] = byte(cs >> 8)
    b[3] = byte(cs)
    return b
}

func checksum(b []byte) uint16 {
    count := len(b)
    sum := uint32(0)
    for i := 0; i < count-1; i += 2 {
        sum += uint32(b[i])<<8 | uint32(b[i+1])
    }
    if count&1 != 0 {
        sum += uint32(b[count-1]) << 8
    }
    for (sum >> 16) > 0 {
        sum = (sum & 0xffff) + (sum >> 16)
    }
    return ^(uint16(sum))
}
```

受信時にも `binary.BigEndian` を使って、今度はネットワークバイトオーダーからネイティブのバイトオーダーに戻す。
また、Raw Socket から読み込んだパケットには IP ヘッダーも含まれているので、読み飛ばす必要がある。
そのため、まず IP ヘッダーに含まれる IP ヘッダー長フィールドを読む必要がある。
ICMP メッセージを読み取ったら、今度は ID が一致していること、Type が 0、つまり Echo Reply Message であることを確認する。
あとは Data にセットされた送信時刻を取り出して、受信時刻から引いて RTT を計算する。

```
func ParseEchoMessageWithIPv4Header(b []byte) (*EchoMessage, error) {
	// IHL * 4 bytes
	hlen := int(b[0]&0x0f) << 2
	b = b[hlen:]
	m := &EchoMessage{
		Type:     Type(b[0]),
		Code:     uint8(b[1]),
		Checksum: uint16(binary.BigEndian.Uint16(b[2:4])),
		ID:       uint16(binary.BigEndian.Uint16(b[4:6])),
		Seq:      uint16(binary.BigEndian.Uint16(b[6:8])),
	}
	m.Data = make([]byte, len(b)-8)
	copy(m.Data, b[8:])
	return m, nil
}

...
if rm.Type == ECHO_REPLY && rm.ID == id {
	err = sent_at.UnmarshalBinary(rm.Data)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Time.UnmarshalBinary:", err)
	}
	rtt := received_at.Sub(sent_at)
```

そんな感じで ping ができた。あとは goroutine を作って、Timer で 1 秒おきに Echo Message を送るようにしたり、
SIGINT のハンドラを設定して、exit 前に ping っぽく RTT やパケロス率などの統計情報を出すようにした。
ping の出力結果と見比べてみると、RTT もそれっぽいし、良さそう。やったね。

```
>  sudo go run main.go tyamagu2.xyz
PING tyamagu2.xyz ( 192.30.252.154 )
seq = 0 time = 179.915245ms
seq = 1 time = 178.756511ms
seq = 2 time = 179.783102ms
seq = 3 time = 179.712126ms
^C
---- tyamagu2.xyz ping statistics  -----
4 packets transmitted, 4 packets received, 0 % packet loss
min/max/avg =  178.756511ms / 179.915245ms / 179.541746ms

> ping tyamagu2.xyz
PING tyamagu2.xyz (192.30.252.154): 56 data bytes
64 bytes from 192.30.252.154: icmp_seq=0 ttl=52 time=179.688 ms
64 bytes from 192.30.252.154: icmp_seq=1 ttl=52 time=180.406 ms
64 bytes from 192.30.252.154: icmp_seq=2 ttl=52 time=180.126 ms
64 bytes from 192.30.252.154: icmp_seq=3 ttl=52 time=180.013 ms
64 bytes from 192.30.252.154: icmp_seq=4 ttl=52 time=179.822 ms
^C
--- tyamagu2.xyz ping statistics ---
5 packets transmitted, 5 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 179.688/180.011/180.406/0.249 ms
```

----

なんとなく作ってみたけど、結構楽しかった。これまで A Tour of Go やっただけだったので、実際に channel 使ってみたりとか、
net パッケージのコード読んだりとか良い勉強になった。C の ping のコードもいくつかざっと読んだし、
ICMP にも詳しくなったし、tcpdump と ping の man ページもじっくり読めたし。
モチベーションが続けば、もうちょっとコードを精査して整理したりしたい。
