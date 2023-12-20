# 自作サイト用にOGP画像を自動生成するツールを作った


この記事は、[CAMPHOR- Advent Calendar 2023](https://advent.camph.net) の21日目の記事です。

## はじめに
TwitterとかSlackなどで、URLを貼り付けると、以下のように記事のサムネイルが表示されることがあると思います。

![OGPのサムネイルの例](https://res.cloudinary.com/zenn/image/upload/s--OCWkJEl_--/c_fit%2Cg_north_west%2Cl_text:notosansjp-medium.otf_55:Python%25E3%2581%25A7%25E3%2582%25A8%25E3%2582%25A2%25E3%2582%25B3%25E3%2583%25B3%25E3%2581%25AE%25E3%2583%25AA%25E3%2583%25A2%25E3%2582%25B3%25E3%2583%25B3%25E4%25BF%25A1%25E5%258F%25B7%25E3%2582%2592%25E3%2583%2587%25E3%2582%25B3%25E3%2583%25BC%25E3%2583%2589%25E3%2581%2597%25E3%2583%2591%25E3%2583%25BC%25E3%2582%25B9%25E3%2581%2599%25E3%2582%258B%2Cw_1010%2Cx_90%2Cy_100/g_south_west%2Cl_text:notosansjp-medium.otf_37:mikiken%2Cx_203%2Cy_121/g_south_west%2Ch_90%2Cl_fetch:aHR0cHM6Ly9zdG9yYWdlLmdvb2dsZWFwaXMuY29tL3plbm4tdXNlci11cGxvYWQvYXZhdGFyL2MwNWE5NjgyMzcuanBlZw==%2Cr_max%2Cw_90%2Cx_87%2Cy_95/v1627283836/default/og-base-w1200-v2.png)

SNS向けに記事のメタ情報を記述するプロトコルとしては、[Open Graph protocol](https://ogp.me/) というものが広く使われています。

今回は、このサイトにOGPに準拠した記事のサムネイルを自動生成するツール作ってみました。

## 方針
自分しか使わないツールなので、とりあえず以下のような方針で進めることにしました。

- サムネイルには、記事のタイトル, ユーザー名, アイコン, サイト名を表示する
- 記事の.mdファイルのパスを渡すと、Front Matterから記事タイトルを取得して、サムネイルを生成する
- Hugoと連携させる可能性を一応考え、Goで実装する
- SVGでサムネイルのテンプレートを用意しておき、そこに記事タイトルを埋め込み、PNGで出力する

## 実装してみる
### 雛形の用意
まずはサムネイルの雛形を用意します。適当にPowerPointで雛形を作成し、Inkscapeで調整すると以下のような感じになりました。~~適当に作ったらZennのパクリみたいになってしまった~~

![](images/ogp/ogp_img_template.svg)

雛形のSVGファイルをテキストエディタで開き、いろいろ手直しを加えた結果、以下のようになりました。

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<svg width="1200" height="630" viewBox="0 0 317.5 166.6875" version="1.1"
   xmlns:xlink="http://www.w3.org/1999/xlink"
   xmlns="http://www.w3.org/2000/svg"
   xmlns:svg="http://www.w3.org/2000/svg">
   <defs>
      <style>
         <![CDATA[@import url('https://fonts.googleapis.com/css2?family=Noto+Sans+JP:wght@300;700&display=swap');]]>
            .noto-sans{font-family: 'Noto Sans JP', sans-serif;}
      </style>
      <linearGradient x1="802.40997" y1="636.23401" x2="157.59" y2="-132.23399" gradientUnits="userSpaceOnUse" spreadMethod="reflect" id="fill">
         <stop offset="0" stop-color="#265073"/>
         <stop offset="0.06" stop-color="#265073"/>
         <stop offset="1" stop-color="#9AD0C2"/>
      </linearGradient>
      <clipPath clipPathUnits="userSpaceOnUse" id="clipPath">
         <circle cx="-102.56742" cy="102.00123" style="stroke-width:0.246697" r="52.916668" />
      </clipPath>
   </defs>
   <g transform="translate(27.425752,2.2446047)">
      <g style="overflow:hidden" transform="matrix(0.33075834,0,0,0.33075834,-27.425752,-2.2446047)">
         <rect x="0" y="0" width="960" height="504" fill="#ffffff" />
         <rect x="0" y="0" width="960" height="504" fill="url(#fill)" style="fill:url(#fill)" />
         <path d="M 48,62.4088 C 48,49.4805 58.4805,39 71.4088,39 H 887.591 C 900.519,39 911,49.4805 911,62.4088 V 441.591 C 911,454.52 900.519,465 887.591,465 H 71.4088 C 58.4805,465 48,454.52 48,441.591 Z" fill="#ffffff" fill-rule="evenodd" fill-opacity="0.94902" />
         <g clip-path="url(#clip2)">
            <use width="100%" height="100%" xlink:href="#img1" transform="matrix(0.818182,0,0,0.818182,74,384)" x="0" y="0" />
         </g>
         <text class="noto-sans" font-size="27px" font-weight="300" transform="translate(150.168,424)">mikiken (@mikikeen)</text>
         <text fill="#7f7f7f" class="noto-sans" font-size="21px" font-weight="300" transform="translate(772.616,443)">mikiken.net</text>
         <foreignObject x="80" y="74" width="800" height="500">
            <html xmlns="http://www.w3.org/1999/xhtml">
               <div class="noto-sans" style="font-size:48px; font-weight:700;">
               {{.article_title}}
               </div>
            </html>
         </foreignObject>
      </g>
      <image width="106" height="106" preserveAspectRatio="none" xlink:href="<アイコンをbase64エンコーディングした文字列>" id="avatar" x="-155.48409" y="49.08456" clip-path="url(#clipPath)" transform="matrix(0.19915161,0,0,0.19915161,27.96638,114.72905)" />
   </g>
</svg>
```

上記のSVGファイルのうち、
```XML
<foreignObject x="80" y="74" width="800" height="500">
   <html xmlns="http://www.w3.org/1999/xhtml">
      <div class="noto-sans" style="font-size:48px; font-weight:700;">
      {{.article_title}}
      </div>
   </html>
</foreignObject>
```

の部分が、記事のタイトルを埋め込む部分です。`{{.article_title}}`の部分を、あとで記事のタイトルに置き換えます。

一応、ポイントとしては、

- テンプレートにローカルのフォントを用いると、実行時にフォントのバイナリを手元に用意しておく必要があり面倒なので、Webフォントを使うようにした

- 記事のタイトルが長い場合、改行を入れたいが、SVGの仕様では要素幅に合わせて自動的にテキストを折り返す機能がないっぽい。そこで、`foreignObject`要素を使って、HTMLをSVGに埋め込むようにした

という感じです。

### Goでコードを書く
実装のコードを載せようかとも思ったんですが、上で言った内容をただ書いてるだけなので、割愛します。実際のコードは、以下のリポジトリにあります。

{{<cardlink url="https://github.com/mikiken/ogp-img-generator/">}}

少しトリッキーなことをしている点としては、PNG画像を生成するために、ヘッドレスブラウザを起動して、スクリーンショットを撮っているところです。Webフォントを埋め込んでいるため、今回はこの方法を取っているんですが、少し大掛かり感はあります。

```Go
func convertToPng(svgContent []byte) []byte {
	// get svg size
	width, height, err := getSvgSize(svgContent)
	if err != nil {
		fmt.Println(err)
	}
	// launch headless browser
	page, err := rod.New().MustConnect().Page(proto.TargetCreateTarget{})
	if err != nil {
		fmt.Println(err)
	}
	// set svg content to page
	if err = page.SetDocumentContent(string(svgContent)); err != nil {
		fmt.Println(err)
	}

	// take screenshot
	img, err := page.MustWaitStable().Screenshot(true, &proto.PageCaptureScreenshot{
		Format: proto.PageCaptureScreenshotFormatPng,
		Clip: &proto.PageViewport{
			X:      7.5,
			Y:      7.5,
			Width:  float64(width),
			Height: float64(height),
			Scale:  1,
		},
		FromSurface: true,
	})

	if err != nil {
		fmt.Println(err)
	}

	return img
}
```

Goを書くのは初めてだったんですが、ChatGPTに質問しつつ1日くらいで大体書けました。

### Hugo側でOGP画像のパスを設定する
あとは、Hugoが生成するHTMLの`head`要素の中に、OGPに準拠するように`meta`要素を追加します。

Hugo側でも、OGPやTwitter Cardの設定の雛形は用意されており、`layouts/partials/head/meta.html`に以下のように書くと、必要な`meta`要素を生成してくれます。

```HTML
{{- template "_internal/opengraph.html" . -}}
{{- template "_internal/twitter_cards.html" . -}}
```

自分は少しカスタマイズしたかったので、以下のような記述を追加しました。

```HTML
{{- template "_internal/opengraph.html" . -}}

{{ if .Params.autoGenOgpImg }}
<meta property="og:image" content="{{ .Site.BaseURL }}images/ogp/content/{{.File.Dir}}{{.File.BaseFileName}}.png">
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:image" content="{{ .Site.BaseURL }}images/ogp/{{.File.Dir}}{{.File.BaseFileName}}.png">
{{- else -}}
<meta name="twitter:card" content="summary">
{{- end -}}

{{- /* Twitter Card Configuration */}}
<meta name="twitter:title" content="{{ .Title }}">
<meta name="twitter:description" content="{{ with .Description }}{{ . }}{{ else }}{{if .IsPage}}{{ .Summary }}{{ else }}{{ with .Site.Params.description }}{{ . }}{{ end }}{{ end }}{{ end -}}">

{{- $twitterSite := "" }}
{{- with site.Params.social }}
{{- if reflect.IsMap . }}
{{- $twitterSite = .twitter }}
{{- end }}
{{- else }}
{{- with site.Social.twitter }}
{{- $twitterSite = . }}
{{- warnf "The social key in site configuration is deprecated. Use params.social.twitter instead." }}
{{- end }}
{{- end }}

{{- with $twitterSite }}
{{- $content := . }}
{{- if not (strings.HasPrefix . "@") }}
{{- $content = printf "@%v" $twitterSite }}
{{- end }}
<meta name="twitter:site" content="{{ $content }}">
{{- end }}
```

~~しかし書き方をミスってるっぽく、現状うまくTwitter Cardにサムネイルが表示されない~~


## 使い方
```bash
$ ogp-img-generator <雛形のSVGファイルのパス> <記事の.mdファイルのパス>*
```
みたいなコマンドを打つと、`static/images/ogp/content/`以下に、記事のファイルパスに対応したサムネイルのPNG画像が生成されます。

また、Gitコマンドと組み合わせて、以下のようなコマンドを実行すると、変更された記事に対してのみサムネイルを生成できます。

```bash
git add -N . && git diff --name-only | grep \.md$ | xargs ogp-img-generator <雛形のSVGファイルのパス> && git reset HEAD
```

## 今後の課題など
- 上の処理をCIでやるようにしたい
    - Goで実装したので、`go install`するだけで実行バイナリが生成でき、割とやりやすそう
- Twitter Cardが上手く出ないのを直す
