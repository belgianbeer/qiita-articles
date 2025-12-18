<!-- https://qiita.com/belgianbeer/items/d3c0f33d372c8c02ccb6 -->
# トイカメラの画像データにExif情報を追加してみた

## 3COINSのトイカメラ

少し前に3COINSのトイカメラを入手しました。使ってみると今時のカメラメーカー製のデジカメでは得られない絵が撮れて面白いです。

カメラの外観

![DSCF4879.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/6ab0431e-bec2-455e-afd1-d8d67cbf03d8.jpeg)

トイカメラで撮った写真

![PHO00263.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/130638/37be3990-c4af-4623-b2e5-34ed6151d9fb.jpeg)

ちなみに2500円(+消費税)とお手頃価格でした(2025年12月現在)。

## Exif情報が無い！

購入前は安いから時計なんて無いだろうなと思っていたわけですが、実際にはちゃんと時計があって感心しました。でも撮ったデータを見たらデジタルカメラやスマートフォンでは当たり前のExif情報が無く、画像データとしてはすっぴんのJPEGのみ。じゃあ時計はどこに使われるのかと思いきや、画像の左下に秒までの撮影時刻が写しこまれている(時刻の写しこみはON/OFF可)のと、画像ファイルのタイムスタンプとして記録されていました。

撮影画像を見る分には時刻データが画像に写っているのは便利ですが、Exif情報が無いと写真データを扱うアプリで正しく時刻情報を扱うことができなくて、エラーになったり撮影時刻順に並ばないという事態になります。

## 無いのなら付ければいいさExif情報

幸いなことに撮影時刻はファイルのタイムスタンプとして残っているので、その情報を元にExifの各情報を画像データに付加できれば良いわけです。そのためのツールが[ExifTool](https://exiftool.org/)で、画像データのExif情報の表示や変更、追加、削除等を自由に行えます。


## ExifToolのインストール

ExifToolを使うにはインストールが必要ですが、FreeBSDではportsに「p5-Image-ExifTool」というパッケージ名で用意されているので、pkgでインストールできます。

```console
# pkg install p5-Image-ExifTool
```

ExfiToolそのものはPerlで記述されているため、必要に応じてperl5等もインストールされます。

DebianやUbuntuではlibimage-exiftool-perlというパッケージ名です。

```console
# apt install libimage-exiftool-perl
```

### ExifToolの基本的な使い方

ExifToolでファイルを指定すると画像データのExif情報が一式表示され、タグを指定することで、そのタグの情報だけを表示します。デフォルトではタグはわかりやすいワードで表示しますが、`-s`でタグでの表示となります。

次の例はexiftoolを使って3COINSのトイカメラの画像データを表示したところです。

```console
$ exiftool PHO00313.JPG
ExifTool Version Number         : 13.30
File Name                       : PHO00313.JPG
Directory                       : .
File Size                       : 325 kB
File Modification Date/Time     : 2025:10:25 15:22:08+09:00
File Access Date/Time           : 2025:12:17 22:53:24+09:00
File Inode Change Date/Time     : 2025:12:18 07:18:12+09:00
File Permissions                : -rw-rw-rw-
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.02
Resolution Unit                 : inches
X Resolution                    : 72
Y Resolution                    : 72
Image Width                     : 2560
Image Height                    : 1920
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 2560x1920
Megapixels                      : 4.9
$
```

-s オプションを指定してタグで表示させると次のようになります。

```console
$ exiftool -s PHO00313.JPG
ExifToolVersion                 : 13.30
FileSize                        : 325 kB
FileModifyDate                  : 2025:10:25 15:22:08+09:00
----- <中略> -----
ImageWidth                      : 2560
ImageHeight                     : 1920
EncodingProcess                 : Baseline DCT, Huffman coding
BitsPerSample                   : 8
ColorComponents                 : 3
YCbCrSubSampling                : YCbCr4:2:0 (2 2)
ImageSize                       : 2560x1920
Megapixels                      : 4.9
$ 
```

タグに関しては`-TAG`でタグ名を指定します。通常のデジカメの画像データには多数のExif情報があり、単純にexiftoolで表示すると数十行の出力があるのですが、タグを指定することで必要な情報だけを表示できます。またファイルの更新時間等Exifとは直接関係無い情報も`FileModifyDate`でTAGとして指定できます。タグの値を書き換えるには`-TAG=値`で指定し、`-TAG1<TAG2`ではTAG1にTAG2の値をコピーできます。

### Exifの時刻情報

Exifには時刻情報が3種類定義されていて、Exifバージョン2.31からはUTCと時差を記録するためのオフセットも時刻情報に合わせて3種類用意されています。

| 時刻TAG           | 時差のTAG            | 意味                      |
|------------------|---------------------|---------------------------|
| DateTimeOriginal | OffsetTimeOriginal  | 画像データの生成時刻         |
| CreateDate       | OffsetTimeDigitized | デジタルデータの作成時刻      |
| ModifyDate       | OffsetTime          | デジタルデータが加工された時刻 |

通常デジタルカメラの画像データは3種類とも同じ時刻ですが、例えば撮影後にカメラの機能で画像を回転させるとModifyDateが更新されます。またexiftoolでは時刻に関しては**AllData**というエイリアスが用意されていて、3つの時刻をまとめて扱えます。

## 書き込むタグと値

Exifでは撮影データとしてシャッター速度、絞り、レンズの焦点距離、ISO感度等様々な情報を記録できますが、このトイカメラではそれらの撮影データの詳細は不明です。レンズに関してはパッケージや本体正面にF2.8、焦点距離2.2mmと記載されているので、それらも記録します。またメーカー名やモデルなども記録できます。

他にExifには画像の縦横を示すタグがあります。最近のデジタルカメラやスマートフォンで縦や横に構えて撮影すると画像データも撮影時の方向を反映しますが、実際の画像データは一定方向のままで、上下左右の方向を示すOrientationというTAGに合わせてブラウザ等が縦横を合わせて表示します。このトイカメラには撮影時の縦横を検出するセンサーが無く、縦に構えて撮影しても横向きの画像となってしまいます。そこでOrientationを書き込むことで縦横を修正します。

以上を指定すると、次のようになります。なお`-quiet`, `-preserve`, `-overwrite_original`を指定して、元ファイルへの上書きして(バックアップを作らない)、ファイルのタイムスタンプを変更せず、余計なメッセージは出さないようにしています。

```shell
exiftool -quiet -preserve -overwrite_original \
	-AllDates"<FileModifyDate" \
	-OffsetTime="+09:00" \
	-OffsetTimeOriginal="+09:00" \
	-OffsetTimeDigitized="+09:00" \
	-Make="3COINS" \
	-Model="MINI TOY CAMERA 2513/KRTC" \
	-FocalLength="2.2mm" \
	-FNumber="2.8" \
	-Orientation="Horizontal" \
	ファイル名
```

これを最初に例で使った PHO00313.JPG に対して実行した後、exiftoolで情報を表示すると次のようになります。

```console
$ exiftool PHO00313.JPG
ExifTool Version Number         : 13.30
File Name                       : PHO00313.JPG
Directory                       : .
File Size                       : 325 kB
File Modification Date/Time     : 2025:10:25 15:22:08+09:00
File Access Date/Time           : 2025:12:18 07:58:27+09:00
File Inode Change Date/Time     : 2025:12:18 07:58:20+09:00
File Permissions                : -rw-rw-rw-
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.02
Exif Byte Order                 : Big-endian (Motorola, MM)
Make                            : 3COINS
Camera Model Name               : MINI TOY CAMERA 2513/KRTC
Orientation                     : Horizontal (normal)
X Resolution                    : 72
Y Resolution                    : 72
Resolution Unit                 : inches
Modify Date                     : 2025:10:25 15:22:08
Y Cb Cr Positioning             : Centered
F Number                        : 2.8
Exif Version                    : 0232
Date/Time Original              : 2025:10:25 15:22:08
Create Date                     : 2025:10:25 15:22:08
Offset Time                     : +09:00
Offset Time Original            : +09:00
Offset Time Digitized           : +09:00
Components Configuration        : Y, Cb, Cr, -
Focal Length                    : 2.2 mm
Color Space                     : Uncalibrated
Image Width                     : 2560
Image Height                    : 1920
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Aperture                        : 2.8
Image Size                      : 2560x1920
Megapixels                      : 4.9
Create Date                     : 2025:10:25 15:22:08+09:00
Date/Time Original              : 2025:10:25 15:22:08+09:00
Modify Date                     : 2025:10:25 15:22:08+09:00
Focal Length                    : 2.2 mm
$
```

## exif4toycamera

exiftoolをそのまま使うには多数のTAGと値を指定する必要があるので、シェルスクリプトにまとめました。コマンド名は**exif4toycamera**で、次の3つのオプションをサポートしています。

- -r: 時計回りに縦向き (Orientationを"Rotate 270 CW")
- -l: 反時計回りに縦向き (Orientationを"Rotate 90 CW")
- -z offset: 時差の文字列を指定。デフォルトは"+09:00" 

完成したシェルスクリプトはGitHubの次のリポジトリで公開しています。

https://github.com/belgianbeer/exif4toycamera

特にOSに依存するような記述は無いので、exiftoolを用意できればFreeBSDに限らずBSD系のオペレーティングやmacOS、多くのLinuxで動作します。

### Windows版 exif4toycamera.ps1

最近はWindowsでPowerShellを使うことも多いので、PowerShellで記述したWindows版も同じリポジトリに追加してあります。Windows版の場合コマンド名は**exif4toycamera.ps1**となります。なおWindowsでexiftoolを使うには、ExifToolのサイトにあるWindows用のZIPファイルをダウンロードし、ZIPに含まれているREADME.txtの手続きに従ってインストールする必要があります。
