## Trouves un negative cache !

### Qu'est-ce qu'il y a ...?

普段、私はいくつかの業務用システムの構築や、運用を行なっています  
同僚からの報告をきっかけに、私の触っている業務用システムで、 slab cache が積載メモリの大半を占有する状況が発生していることに気付きました  
実際に確認してみると、ユーザ空間から要求された合計メモリサイズが僅かであるにもかかわらず、メモリ使用量はほぼ 100% の値を指しています  
より詳しく調べる前は、ページキャッシュで占有されているのだろう、と考えましたがそういった状況でもなさそうです

### C'est quoi slab cache ?
Linux kernel には、slab cache と呼ばれる kernel 内部キャッシュが存在します  
この slab cache が、肥大化したものの、何が原因となっているのかすぐにはわからないことが多いと思います
この記事では、その肥大化の原因を具体的に調査する方法について共有します

### Pourquoi tu l'observes ?
slab cache は、linux kernel 内部で確保されるメモリ領域で、自動的に解放される実装となっているようです  
しかしながら、肥大化した状況では、メモリ確保のパフォーマンスに影響があるはずですし、場合によって、必要なメモリの確保ができないケースが出てくるかもしれません  
実のところ、そういった具体的なケースにはまだ実際に出会えていませんが、万が一 slab cache に不要なデータが大量に存在するとしたら、解消するに越したことはないでしょう

### Qui prendais beaucoup de memory..?

このケースでは slab cache dentry が多量のメモリを確保していました  
メモリ使用の内訳は、例えば以下のような PATH から得ることができます  
* `/proc/meminfo` 
* `/proc/slabinfo`
* `/proc/sys/fs/dentry-state`
* `sar command`

これらの PATH からどういった情報を得られるかは、インターネット上から簡単に見つけることができますので、説明は割愛します  
Slab cache の内訳をこれらで確認して得られたのは、

* 再利用可能(SReclaimable)な領域が大半である
* negative dentry が多量に確保されている

といった内容です  

### L'arrangement provisoire

とにかく、slab cache をすぐに解放したいという状況はそう多くないと思います  
先に書いた通り、slab cache は自動的に解放されますし、実際に私の運用する環境でも何か問題が起きていたわけではありません  
しかしながら、Linux では、これらのキャッシュを手動でリリースする仕組みが備わっています  
例えば、以下のコマンドを実行すると、inode や、dentry cache がリリースされます  
私は遭遇したことがありませんが、即時リリースが必要な状況では利用すると良いでしょう
```
echo 2 > /proc/sys/vm/drop_caches
```

### La cause profonde

幸いなことに、運用中のシステムでは同じ状況のサーバがたくさんありましたし、開発環境でも再現することができました    
そこで、影響のない環境で eBPF が使える状況を作り、negative dentry にどういった情報が格納されているのかを調査して見ました  
具体的には、以下の eBPF プログラムを実行した上で、slab cache を手動リリースすることにより、簡単に negative dentry のエントリーを確認できます  

```
from __future__ import print_function
from bcc import BPF
​
bpf_text = """
#include <linux/dcache.h>
int kprobe____dentry_kill(struct pt_regs *ctx, struct dentry *dentry)
{
    if (dentry->d_inode == NULL) {
        bpf_trace_printk("N:%s\\n",dentry->d_name.name);
    }else{
        bpf_trace_printk("P:%s\\n",dentry->d_name.name);
    }
    return 0;
}
​
"""
​
b = BPF(text=bpf_text).trace_print()
```
このプログラムを実行すると、negative dentry にリンクされていたファイル名と、そうでない(有効な)dentry にリンクされていたディレクトリ・ファイル名が出力されます  
前者のファイル名にはプレフィックスとして、`N:`が付与されます

### Quelle est la negative dentry ?

説明が前後しますが、negative dentry は、すでに存在しないファイルPATH を管理するためのキャッシュです  
これは、DNS のネガティブキャッシュと似ていて、特定 PATH のファイルが存在しないことを示すことで、物理デバイスへの I/O を減らしてパフォーマンスを上げるための仕組みです  
正しく機能している状況では、このキャッシュは非常に有用です  

今回は、negative dentry として確保されていた多量のファイル名から、書き出し元のプロセスを容易に特定することができました  
各サーバ上で動作する監視デーモンが、/tmp 以下に高い頻度で一時ファイルを作成・削除することにより、negative dentry が肥大化する原因となっていることがわかりました　　

ちなみに、/tmp が、tmpfs でマウントされているシステムではこの状況は発生しません  
tmpfs は十分高速であるためだと思うのですが、negative dentry の登録処理がスキップされる実装になっています  
私の運用しているシステムでは、/tmp は、EXT4 でマウントされており、/tmp から削除されたファイルは、negative dentry として登録されることになります

### Le remède permanent

今回のケースでは、/tmp を TMPFS としてマウントすることで恒久対応としました  
実際に、slab cache は、非常に小さなサイズとなり、内包される negative dentry も非常に小さなサイズとなっています
eBPF を導入するためには、比較的新しい kernel (4.4 以降だったと思います) が必要になりますから、古いカーネルではこの対応はできません  
また、環境の制約により bcc を利用した eBPF の導入が容易ではないケースも多いかと思います  
そういったケースでは、libbcc に依存しない [ply](https://github.com/iovisor/ply) の利用を検討するのも一手になるでしょう  