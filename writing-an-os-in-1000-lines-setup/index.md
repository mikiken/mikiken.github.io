# "Writing an OS in 1000 lines"の環境構築をしたときのメモ


同じ内容をZennのスクラップにも書いたんですが、こっちにも一応書いとく

https://zenn.dev/mikiken/scraps/483401c1bef8a1


本の記述に従い環境構築をしてみた

https://operating-system-in-1000-lines.vercel.app/ja/setting-up-development-environment

しかし、[5. ブート](https://operating-system-in-1000-lines.vercel.app/ja/boot)の説明通りに`run.sh`を記述し実行したところ、以下のようなエラーが出た
```bash
❯ ./run.sh
+ QEMU=qemu-system-riscv32
+ qemu-system-riscv32 -machine virt -bios default -nographic -serial mon:stdio --no-reboot
qemu-system-riscv32: Unable to load the RISC-V firmware "opensbi-riscv32-virt-fw_jump.bin"
```
パッと見た感じ、QEMUが`-bios default`で呼び出すファームウェアが、本の執筆当時とは変わっていそう(?)


そこで、`run.sh`を以下のように変更した
```bash
#!/bin/bash
set -xue

# QEMUの実行バイナリへのパス
QEMU=qemu-system-riscv32

# QEMUを起動
$QEMU -machine virt -bios opensbi-riscv32-generic-fw_dynamic.bin -nographic -serial mon:stdio --no-reboot
```
すると、エラー自体は出なくなったが、QEMUを起動しても何も表示されない
```
❯ ./run.sh
+ QEMU=qemu-system-riscv32
+ qemu-system-riscv32 -machine virt -bios opensbi-riscv32-generic-fw_dynamic.bin -nographic -serial mon:stdio --no-reboot



```


そこで、以下の手順を試した
1. aptで入れたQEMU関係のパッケージを全てアンインストール
2. riscv-gnu-toolchainをソースコードからビルドする
3. QEMUをソースコードからビルドする


### aptで入れたQEMU関係のパッケージを全てアンインストール
```bash
sudo apt-get remove --purge "qemu-*"
```


### riscv-gnu-toolchainをソースコードからビルドする
ビルドに必要なパッケージをインストールしておく。
```bash
sudo apt install -y texinfo bison flex libgmp-dev
```
環境によっては、上記以外にも足りないパッケージがあるかもしれないので、ビルド実行後にエラーが出たら適宜インストールしてから再実行する。

以下のコマンドを実行し、ソースコードからriscv-gnu-toolchainをビルドする。
```bash
cd ~
git clone --depth=1 https://github.com/riscv-collab/riscv-gnu-toolchain.git
cd riscv-gnu-toolchain/
./configure --prefix=/opt/riscv32 --with-arch=rv32i --with-abi=ilp32
sudo make
```


### QEMUをソースコードからビルドする
この本が書かれたのが2023年の8月頃なので、当時の安定リリースであるQEMUのv8.0系をソースコードからビルドし、インストールする。

ビルドに必要なパッケージをインストールしておく。
```bash
sudo apt install -y pkg-config ninja-build libglib2.0 libpixman-1-dev
```
環境によっては、上記以外にも足りないパッケージがあるかもしれないので、ビルド実行後にエラーが出たら適宜インストールしてから再実行する。

以下のコマンドを実行する。
```bash
mkdir ~/qemu && cd ~/qemu
git clone --depth=1 --branch stable-8.0 https://github.com/qemu/qemu.git
sudo mkdir /opt/qemu-system-riscv32
sudo ./configure --target-list=riscv32-softmmu --prefix=/opt/qemu-system-riscv32/
sudo make -j $(nproc)
sudo make install
```
ビルドが完了すると、上の`--prefix`で指定したパスに`qemu-system-riscv32`という実行バイナリが生成されているはず。そのディレクトリに対してPATHを通す。

すなわち`.bashrc`に以下を追加する。
```bash
export PATH="$PATH:/opt/qemu-system-riscv32/bin"
```


### 動作確認
本の記述通りに`run.sh`を記述し、実行した。
```bash
❯ ./run.sh 
+ QEMU=qemu-system-riscv32
+ qemu-system-riscv32 -machine virt -bios default -nographic -serial mon:stdio --no-reboot

OpenSBI v1.2
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name             : riscv-virtio,qemu
Platform Features         : medeleg
Platform HART Count       : 1
Platform IPI Device       : aclint-mswi
Platform Timer Device     : aclint-mtimer @ 10000000Hz
Platform Console Device   : uart8250
Platform HSM Device       : ---
Platform PMU Device       : ---
Platform Reboot Device    : sifive_test
Platform Shutdown Device  : sifive_test
Firmware Base             : 0x80000000
Firmware Size             : 208 KB
Runtime SBI Version       : 1.0

Domain0 Name              : root
Domain0 Boot HART         : 0
Domain0 HARTs             : 0*
Domain0 Region00          : 0x02000000-0x0200ffff (I)
Domain0 Region01          : 0x80000000-0x8003ffff ()
Domain0 Region02          : 0x00000000-0xffffffff (R,W,X)
Domain0 Next Address      : 0x00000000
Domain0 Next Arg1         : 0x87e00000
Domain0 Next Mode         : S-mode
Domain0 SysReset          : yes

Boot HART ID              : 0
Boot HART Domain          : root
Boot HART Priv Version    : v1.12
Boot HART Base ISA        : rv32imafdch
Boot HART ISA Extensions  : time,sstc
Boot HART PMP Count       : 16
Boot HART PMP Granularity : 4
Boot HART PMP Address Bits: 32
Boot HART MHPM Count      : 16
Boot HART MIDELEG         : 0x00001666
Boot HART MEDELEG         : 0x00f0b509
```
ひとまず正しく動いてそう


### ldがriscv32向けのelfを吐けない
本の[5.ブート](https://operating-system-in-1000-lines.vercel.app/ja/boot)の通りに`kernel.c`, `kernel.ld`を作成し、`run.sh`でビルドしようとすると、以下のエラーが出た
```bash
❯ ./run.sh
+ QEMU=qemu-system-riscv32
+ CC=clang
+ CFLAGS='-std=c11 -O2 -g3 -Wall -Wextra --target=riscv32 -ffreestanding -nostdlib'
+ clang -std=c11 -O2 -g3 -Wall -Wextra --target=riscv32 -ffreestanding -nostdlib -Wl,-Tkernel.ld -Wl,-Map=kernel.map -o kernel.elf kernel.c
/usr/bin/ld: unrecognised emulation mode: elf32lriscv
Supported emulations: elf_x86_64 elf32_x86_64 elf_i386 elf_iamcu elf_l1om elf_k1om i386pep i386pe
clang: error: ld command failed with exit code 1 (use -v to see invocation)
```
手元の環境のClangはリンカとしてldを呼んでいるが、ldのターゲットにriscv32がない模様

一旦以下のように`ld.lld`を別途呼び出すようにしたら、ビルドできるようになった
(Clangが呼び出すリンカを変更できないか探ったが、よく分からなかった)
```bash
#!/bin/bash
set -xue

# QEMUの実行バイナリへのパス
QEMU=qemu-system-riscv32

CC=clang
LINKER=ld.lld

# コンパイルオプション
CFLAGS="-c -std=c11 -O2 -g3 -Wall -Wextra --target=riscv32 -ffreestanding -nostdlib -mno-relax"

# カーネルをビルド
$CC $CFLAGS -o kernel.o kernel.c
$LINKER -m elf32lriscv -L/lib -Tkernel.ld -Map=kernel.map kernel.o -o kernel.elf

# QEMUを起動
$QEMU -machine virt -bios default -nographic -serial mon:stdio --no-reboot -kernel kernel.elf
```


### 参考にさせていただいたサイト
https://qiita.com/ms5/items/74efa5d1ada4912b6a53

https://blog.rogiken.org/blog/2023/04/06/32bit-risc-v-linux%E3%82%92%E4%BD%9C%E3%82%8Aqemu%E3%81%A7%E5%AE%9F%E8%A1%8C%E3%81%99%E3%82%8B/
