# 「CPUの創りかた」のTD4をFPGAで実装してみた


この記事は [CAMPHOR- Advent Calendar 2023](https://advent.camph.net) の13日目の記事です。

## はじめに
「CPUの創りかた」という本をご存知でしょうか？2003年に出版された本で、汎用ロジックIC(74シリーズ)を使って、簡単な4bitのCPUを作ります。本のタイトルから、デジタル回路の話が多そうだという印象を持つかもしれませんが、LED点灯回路の抵抗値の計算といった、基本的なアナログ回路の内容から丁寧に解説されている名著です。今年で発売から20年を迎えますが、未だに根強い人気を誇っており、現在では30刷以上重版されているようです。

{{<cardlink url="https://book.mynavi.jp/ec/products/detail/id=22065">}}

僕の通う大学では、3年生後期に、FPGAで何らかのデジタル回路を実装するという実験があります。そこで、この本の内容をFPGAで実装してみました。

{{<tweet 1721556673981219065>}}

## 作ったもの
「CPUの創りかた」では、TD4という4bitのCPUを段階を分けて作っていきます。

TD4の仕様をざっくり説明すると、

- 命令長8bit (オペコード 4bit, オペランド 4bit)
- 演算用のレジスタ(4bit)が2つ

という感じです。

実行できる命令としては、以下の12種類があります。

- `ADD A, Im`
- `ADD B, Im`
- `MOV A, Im`
- `MOV B, Im`
- `MOV A, B`
- `MOV B, A`
- `JMP Im`
- `JNC Im`
    - キャリーフラグが0のとき、即値Imで指定されたアドレスにジャンプする
- `IN A`
    - 入力端子からデータを入力し、Aレジスタに代入する
- `IN B`
    - 入力端子からデータを入力し、Bレジスタに代入する
- `OUT B` 
    - Bレジスタの値を出力端子に出力する なお、`OUT A`命令は存在しない。
- `OUT Im`
    - 即値Imを出力端子に出力する

<br>

なお、リポジトリはこれです。

{{<cardlink url="https://github.com/mikiken/td4">}}

あと、学校の授業で成果発表したときのスライドを一応載せときます

<iframe class="speakerdeck-iframe" frameborder="0" src="https://speakerdeck.com/player/0a03e727a29747439945f771ad10c3af" title="簡単な4bitCPUの作成" allowfullscreen="true" style="border: 0px; background: padding-box padding-box rgba(0, 0, 0, 0.1); margin: 0px; padding: 0px; border-radius: 6px; box-shadow: rgba(0, 0, 0, 0.2) 0px 5px 40px; width: 100%; height: auto; aspect-ratio: 560 / 315;" data-ratio="1.7777777777777777"></iframe>

## 実装方法
基本的には本に書いてある通りに実装しました。具体的には、

1. レジスタ
2. ALU
3. プログラムカウンタ
4. 命令デコーダ
5. ROM

の順に実装しました。

なお、実装に使った(実験で渡された)FPGAボードは、PowerMedusa MU500-RXです。
三菱電機マイコン機器ソフトウエアが出しているFPGAボードで、AlteraのEP4CE30F23I7NっていうFPGAが載ってるらしい。

ほぼ本に書いてある順で実装したんですが、ROMだけ実装方針に悩んだので最後に回しました。
本だと8bitをDIPスイッチを16個並べてROMを表現しているんですが、今回使ったFPGAボードには8bitのDIPスイッチが2個しか搭載されていませんでした。

そこで、Verilogを使って、ROMの各アドレスに対して決め打ちした命令を返す素子を書き、ROMを表現することにしました。

```verilog
module rom(
	input  [3:0] address,
	output [7:0] data
);

	function [7:0] select;
		input [3:0]	address;
		case (address)
			//Ramen timer!
			4'b0000: select = 8'b10110111; // OUT 0111
			4'b0001: select = 8'b00000001; // ADD A,0001
			4'b0010: select = 8'b11100001; // JNC 0001
			4'b0011: select = 8'b00000001; // ADD A,0001
			4'b0100: select = 8'b11100011; // JNC 0011
			4'b0101: select = 8'b10110110; // OUT 0110
			4'b0110: select = 8'b00000001; // ADD A,0001
			4'b0111: select = 8'b11100110; // JNC 0110
			4'b1000: select = 8'b00000001; // ADD A,0001
			4'b1001: select = 8'b11101000; // JNC 1000
			4'b1010: select = 8'b10110000; // OUT 0000
			4'b1011: select = 8'b10110100; // OUT 0100
			4'b1100: select = 8'b00000001; // ADD 0001
			4'b1101: select = 8'b11101010; // JNC 1010
			4'b1110: select = 8'b10111000; // OUT 1000
			4'b1111: select = 8'b11111111; // JMP 1111
			default: select = 8'bxxxxxxxx;
			
			/*
			//LED chikachika!
			4'b0000: select = 8'b10110011; // OUT 0011
			4'b0001: select = 8'b10110110; // OUT 0110
			4'b0010: select = 8'b10111100; // OUT 1100
			4'b0011: select = 8'b10111000; // OUT 1000
			4'b0100: select = 8'b10111000; // OUT 1000
			4'b0101: select = 8'b10111100; // OUT 1100
			4'b0110: select = 8'b10110110; // OUT 0110
			4'b0111: select = 8'b10110011; // OUT 0011
			4'b1000: select = 8'b10110001; // OUT 0001
			4'b1001: select = 8'b11110000; // JMP 0000
			4'b1010: select = 8'b00000000;
			4'b1011: select = 8'b00000000;
			4'b1100: select = 8'b00000000;
			4'b1101: select = 8'b00000000;
			4'b1110: select = 8'b00000000;
			4'b1111: select = 8'b00000000;
			default: select = 8'bxxxxxxxx;
			*/
		endcase
	endfunction
	
	assign data = select(address);

endmodule
```

今回使ったFPGAボードはQuartusが使えるので、このVerilogのコードを回路図上のコンポーネントとしてexportし、ROMを実装しました。

## 感想など
ブラックボックスだと思っていたCPUについて、少しは理解が深まったと思います。今度はもう少し大規模なCPUをHDLで書いてみたいですね。

どうでもいいことなんですが、気軽に最初のツイートをしたところ、思ったより反響があって驚きました。
~~やはり自作OS, 自作CPU, 自作言語は人類の三大欲求~~
