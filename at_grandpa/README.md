Title: Crystalを計測するためのあれこれ
Author: Yoshiyuki Tsuchida
Twitter: @at_grandpa

こんにちは、@at_grandpaです。今回はCrystalについての本ということで、「Crystalを計測するためのあれこれ」について書かせていただきます。計測に詳しい方々からは「Crystalに限った内容ではない！」とツッコミをいただきそうですが、そうです、Crystalに限ったことではないのです（すみません）。ではなぜCrystalの本にこれを掲載するかというと、Crystalは現状、

* 「Rubyの代替」というイメージが（今なお）強い
* githubに「Oh, and we don't want to write C code to make the code run faster.」とあるように、公式でも処理速度を推している

ということがあり、何かと「Crystalって速いの？どれくらい速いの？」と注目されがちです。だったら計測してみたい！というのが人情ですが、「どうやって計測したらいいんだ。。。」と躓いてしまっては、Crystalの良さを知る前にモチベーションが下がってしまいます。そこで、いろんな計測方法をここで示し、よりCrystalの素晴らしさを体感していただこうというのが主旨です。

と言いますか、「自分が気になって調べたことをここに記載した」というのが本音です。「原稿を書く」という経験がなく、見積もりを誤った結果、ギリギリで執筆しているというのが現状です。心中お察しくださいませ。

# はじめに

はじめに、今回の内容について共有しておくべきことを書きます。

* Crystalのversionは `0.21.1` です
* 出力結果などを省略する部分は `--snip--` と表記します
* 都合上、実行結果を多少改変しています
    * 実際とは異なるpathや改行が入る場合があります
    * 紙面の関係でスペースなどを削除している場合があります
    * その他、意味が崩れない程度の修正をしている場合があります
* 掲載されているサンプルコードは下記githubからダウンロードできます
    * https://github.com/at-grandpa/techbookfest2-crystal-profiling
* 各種実行の際は自己責任でお願いします

それでは、まずはCrystalの環境構築から行います。

# Crystalを実行するための準備

最新版のCrystalの環境を得るための一番簡単な方法は、公式に提供されている docker image をpullしてくる方法だと思います。下記のようなDockerfileとMakefileを用意すれば、Crystalの最新版の実行環境と、今回の内容を一通り実行できる環境を構築できます。

```
●Dockerfile::
FROM crystallang/crystal

RUN apt-get -y update
RUN apt-get -y upgrade
RUN apt-get -y install curl
RUN apt-get -y install apache2-utils
RUN apt-get -y install google-perftools
RUN apt-get -y install libgoogle-perftools-dev
RUN mkdir -p /root/dev (注:開発用のディレクトリを作成)
WORKDIR /root/dev (注:作業ディレクトリを開発用ディレクトリに設定)


●Makefile::
REPOSITORY=dev
TAG=default
HOST_WORKDIR=$(PWD)
CONTAINER_WORKDIR=/root/dev
CONTAINER_NAME=crystal-lang-dev

build:
    docker build --tag=$(REPOSITORY):$(TAG) .

run:
    docker run \
        -v $(HOST_WORKDIR):$(CONTAINER_WORKDIR) \
        --name $(CONTAINER_NAME) \
        -it $(REPOSITORY):$(TAG) \
        /bin/bash

attach:
    docker attach $(CONTAINER_NAME)
```

これらをHostマシンの作業ディレクトリに用意してbuildとrunを実行すれば、最新のCrystalを使用できる環境ができあがります。また、Hostマシンのディレクトリがコンテナ内のディレクトリにマウントされているので、実行はコンテナ内で、コード編集はHostマシンで行うことができます。buildしてコンテナに入ってみましょう。

```
!!! cmd
$ make build
docker build --tag=dev:default .
Sending build context to Docker daemon 2.972 MB
Step 1 : FROM crystallang/crystal
 ---> d960a077fe1e

--snip--

Successfully built 63804e3a81a4
$ make run
docker run \
    -v /path/to/dir:/root/dev \
    --name crystal-lang-dev \
    -it dev:default \
    /bin/bash
root@507648819acf:~/dev# crystal -v
Crystal 0.21.1 [3c6c75e] (2017-03-06) LLVM 3.5.0
```

では早速、計測のあれこれを見ていきましょう。

# timeコマンド

まずはおなじみtimeコマンドです。プログラムの実行時にtimeコマンドを使用するだけです。

```
●time.cr::
sleep 1.0
```

buildして実行してみます。

```
!!! cmd
$ crystal build time.cr
$ time ./time

real   	0m1.018s
user   	0m0.000s
sys    	0m0.000s
```

timeコマンドは一番お気軽な計測方法でしょう。ぜひCrystalのスピードを体感してみてください。

# Benchmarkクラス

CrystalにはBenchmarkクラスがあります。`Benchmark#bm`は、Benchmarkモジュールの基本的なインターフェースです。`Benchmark#ips`は`instruction per second`を表し、相対的に計測できるインターフェースです。実際に使用してみましょう。サンプルは[APIの公式サイト](https://crystal-lang.org/api/0.21.1/Benchmark.html)を参考に多少改変したものを用いています。

まずは`Benchmark#bm`です。

## Benchmark#bm

```
●bm.cr::
require "benchmark"

n = 5000000
Benchmark.bm do |x|
  x.report("times:") { n.times do
    a = "1"
  end }
  x.report("upto:") { 1.upto(n) do
    a = "1"
  end }
end
```

buildして実行してみましょう。

```
!!! cmd
$ crystal build bm.cr
$ ./bm
Warning: benchmarking without the `--release` flag
won't yield useful results
             user     system      total        real
times:   0.010000   0.000000   0.010000 (  0.012450)
upto:    0.010000   0.010000   0.020000 (  0.011968)
```

計測結果が表示されました。そして、「--releaseフラグが付いていません」と丁寧に教えてくれますね。`--release`を付けて再度buildしてみましょう。

```
!!! cmd
$ crystal build --release bm.cr
$ ./bm
             user     system      total        real
times:   0.000000   0.000000   0.000000 (  0.000049)
upto:    0.000000   0.000000   0.000000 (  0.001577)
```

圧倒的に速くなりました。`--release`の効果は絶大です。続いて`Benchmark#ips`です。

## Benchmark#ips

```
●ips.cr::
require "benchmark"

Benchmark.ips do |x|
  x.report("short") { sleep 0.01 }
  x.report("shorter") { sleep 0.001 }
end
```

上記コードをbuildして実行してみましょう。

```
!!! cmd
$ crystal build --release ips.cr
$ ./ips
  short  92.28  ( 10.84ms) (± 3.90%)  3.33× slower
shorter 307.36  (  3.25ms) (±10.97%)       fastest
```

結果は左から、

* ラベル
* 平均ips
* 1イテレーション当たりの平均処理時間
* 平均ipsに対する標準偏差
* 比較結果

となっています。

Crystal標準でこれらのBenchmarkメソッドが用意されているのは便利ですね。

# ApacheBench

Crystalは簡単にWebサーバーを構築できます。その性能評価を`ApacheBench`で行いましょう。簡単のために、コンテナ内から`ApacheBench`を叩くことにします。まずはWebサーバーをコンテナ内で立ち上げます。下記のコードは公式HPのTOPに書かれているものです。

```
●web_server.cr::
# A very basic HTTP server
require "http/server"

server = HTTP::Server.new(8080) do |context|
  context.response.content_type = "text/plain"
  context.response.print "Hello world, got #{context.request.path}!"
end

puts "Listening on http://127.0.0.1:8080"
server.listen
```

buildして、バックグラウンドで実行しましょう。

```
!!! cmd
$ crystal build web_server.cr
$ ./web_server &
Listening on http://127.0.0.1:8080
```

動作確認のため、試しにcurlで叩いてみます。

```
!!! cmd
$ curl http://127.0.0.1:8080/
Hello world, got /!
$ curl http://127.0.0.1:8080/hoge
Hello world, got /hoge!
```

アクセスできています。では、`ApacheBench`で計測してみましょう。

```
!!! cmd
$ ab -n 10000 http://127.0.0.1:8080/
This is ApacheBench,
Version 2.3 <$Revision: 1528965 $>
Copyright 1996 Adam Twiss,
Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation,
http://www.apache.org/

Benchmarking 127.0.0.1 (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
Completed 4000 requests
Completed 5000 requests
Completed 6000 requests
Completed 7000 requests
Completed 8000 requests
Completed 9000 requests
Completed 10000 requests
Finished 10000 requests


Server Software:
Server Hostname:        127.0.0.1
Server Port:            8080

Document Path:          /
Document Length:        19 bytes

Concurrency Level:      1
Time taken for tests:   2.815 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      840000 bytes
HTML transferred:       190000 bytes
Requests per second:    3552.64 [#/sec] (mean)
Time per request:       0.281 [ms] (mean)
Time per request:       0.281 [ms]
(mean, across all concurrent requests)
Transfer rate:          291.43 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.9      0      92
Processing:     0    0   0.1      0       5
Waiting:        0    0   0.1      0       4
Total:          0    0   0.9      0      92

Percentage of the requests served
within a certain time (ms)
  50%      0
  66%      0
  75%      0
  80%      0
  90%      0
  95%      0
  98%      1
  99%      1
 100%     92 (longest request)
```

計測結果がずらりと出力されました。注目する点が`Requests per second`です。

```
!!! cmd
--snip--

Requests per second:    3552.64 [#/sec] (mean)

--snip--
```

性能としてはこのくらいだとわかりました。では`--release`を付けてbuildしてみるとどうなるでしょうか。

```
!!! cmd
$ crystal build --release web_server.cr
$ ./web_server &
Listening on http://127.0.0.1:8080
$ ab -n 10000 http://127.0.0.1:8080/

--snip--

Requests per second:    4264.04 [#/sec] (mean)

--snip--
```

約1.2倍くらいの性能向上が見られます（いろいろと環境依存があるので厳密な比較ではありません）。このような形で`ApacheBench`を用いれば、Webサーバーの性能もサクッと評価できます。

# gperftools

[gperftools](https://github.com/gperftools/gperftools)はgoogleが開発しているプロファイリングツールです。C/C++向けのプロファイリングツールですが、Cバインディングを使えばCrystalでも使用できます。gperftoolsは`CPU PROFILER`と`HEAP PROFILER`があります。メソッド単位のプロファイリングが可能なので、ボトルネックを探すのに重宝します。

では、簡単なプログラムでプロファイリングを行ってみましょう。以下のようなプログラムを計測します。

```
●計測対象::
class ExampleClass
  @instance_var1 : Array(Time)
  @instance_var2 : Array(Time)

  def initialize
    @instance_var1 = method1
    @instance_var2 = method2
  end

  def method1
    (1..1000000).map { |i| Time.new }
  end

  def method2
    (1..500000).map { |i| Time.new }
  end
end

example_class = ExampleClass.new (注:ここを計測する)
```

`ExampleClass.new`のプロファイリングを行います。まず、gperftoolsの本家サイトの説明を読むと、`CPU PROFILER`を使用するには、コンパイル時に`-lprofiler`をリンクするように書かれています。これをCrystalのCバインディングを用いてリンクします。Cではコンパイル時のオプションとして`-lprofiler`を指定しますが、Crystalの場合はコード中にCバインディングを記述すれば、build時に自動でリンクされるようになります。 リンクすれば`ProfilerStart()`と`ProfilerStop()`というCの関数が使えるようになるので、それらをCrystalのメソッド名として定義します。最終的に以下のようなコードを追加します。

```
●profilerのリンク::
@[Link("profiler")]
lib LibPROFILER
  fun cpu_start = ProfilerStart(file : UInt8*)
  fun cpu_stop = ProfilerStop
end
```

続いて、`HEAP PROFILER`のリンクも行います。プロファイリングするためには`-ltcmalloc`をリンクします。こちらも`CPU PROFILER`の時と同じようにCバインディングを記述します。

```
●tcmallocのリンク::
@[Link("tcmalloc")]
lib LibTCMALLOC
  fun heap_start = HeapProfilerStart(file : UInt8*)
  fun heap_dump = HeapProfilerDump
  fun heap_stop = HeapProfilerStop
end
```

`HEAP PROFILER`の場合、`HeapProfilerDump()`を実行することでプロファイリング結果がファイルに出力されます。では、上記のC関数群を便利に使えるようなProfilerモジュールを定義しましょう。

```
●Profilerモジュール::
module Profiler
  PROFILE_DIR = "profile"
  CPU_PROFILE_DUMP_PATH  = "#{PROFILE_DIR}/cpu.prof"
  HEAP_PROFILE_DUMP_PATH = "#{PROFILE_DIR}/heap.prof"

  def self.start
    Dir.mkdir_p(PROFILE_DIR)
    LibPROFILER.cpu_start(CPU_PROFILE_DUMP_PATH)
    LibTCMALLOC.heap_start(HEAP_PROFILE_DUMP_PATH)
  end

  def self.stop
    LibPROFILER.cpu_stop
    LibTCMALLOC.heap_dump
    LibTCMALLOC.heap_stop
  end
end
```

これで、プロファイリングしたい箇所を`Profiler.start`と`Profiler.stop`で囲めば、CPUとHEAPの計測ができます。プロファイリング結果は`profile`ディレクトリに格納されるようにしました。最終的なコードは以下になります。

```
●example.cr::
@[Link("profiler")]
lib LibPROFILER
  fun cpu_start = ProfilerStart(filename : UInt8*)
  fun cpu_stop = ProfilerStop
end

@[Link("tcmalloc")]
lib LibTCMALLOC
  fun heap_start = HeapProfilerStart(filename : UInt8*)
  fun heap_dump = HeapProfilerDump
  fun heap_stop = HeapProfilerStop
end

module Profiler
  PROFILE_DIR = "profile"
  CPU_PROFILE_DUMP_PATH  = "#{PROFILE_DIR}/cpu.prof"
  HEAP_PROFILE_DUMP_PATH = "#{PROFILE_DIR}/heap.prof"

  def self.start
    Dir.mkdir_p(PROFILE_DIR)
    LibPROFILER.cpu_start(CPU_PROFILE_DUMP_PATH)
    LibTCMALLOC.heap_start(HEAP_PROFILE_DUMP_PATH)
  end

  def self.stop
    LibPROFILER.cpu_stop
    LibTCMALLOC.heap_dump
    LibTCMALLOC.heap_stop
  end
end

class ExampleClass
  @instance_var1 : Array(Time)
  @instance_var2 : Array(Time)

  def initialize
    @instance_var1 = method1
    @instance_var2 = method2
  end

  def method1
    (1..1000000).map { |i| Time.new }
  end

  def method2
    (1..500000).map { |i| Time.new }
  end
end

Profiler.start
example_class = ExampleClass.new
Profiler.stop
```

実際にコンパイルして動かしてみましょう。

```
!!! cmd
$ crystal build example.cr
$ ./example
Starting tracking the heap
PROFILE: interrupts/evictions/bytes = 2566/4/49472
Dumping heap profile to profile/heap.prof.0001.heap
$ ls profile/
cpu.prof  heap.prof.0001.heap
```

プロファイリング結果がファイルに出力されています。中身はバイナリなので直接は見れません。詳細な結果は`google-pprof`コマンドを使って見ることができます。

```
!!! cmd
$ google-pprof ./example profile/cpu.prof
Using local file ./example.
Using local file profile/cpu.prof.
Removing _L_unlock_13 from all stack traces.
Welcome to pprof!  For help, type 'help'.
(pprof)
```

`google-pprof`コマンドの引数には、実行バイナリと出力されたプロファイリング結果のファイルを指定します。すると`google-pprof`のインタラクティブシェルに入ることができ、対話的に結果を見ることができます。`top`と打ってみます。

```
!!! cmd
(pprof) top
Total: 2566 samples
  1309  51.0%  51.0%  1309  51.0% sigprocmask
   532  20.7%  71.7%   532  20.7% 0x00007ffd593f0b22
   292  11.4%  83.1%   292  11.4% __xstat64
   182   7.1%  90.2%  1525  59.4% _Ux86_64_getcontext
    46   1.8%  92.0%    46   1.8% __pthread_cond_wait
    28   1.1%  93.1%    28   1.1% _ULx86_64_get_save_loc
    17   0.7%  93.8%    24   0.9% tc_free
    13   0.5%  94.3%  1535  59.8% _ULx86_64_step
    12   0.5%  94.7%  1593  62.1% malloc
    12   0.5%  95.2%  1943  75.7% tzset
```

これはCPUのサンプリング数が多い順にメソッドを表示してくれます。サンプリング数が多いほど、その関数の処理時間が長いことになります。内容は左から、

* function内でサンプルされた数
* function内でサンプルされた数 / 全サンプル数
* (function内でサンプルされた数 / 全サンプル数)の累積
* function内でサンプルされた数 + functionの呼び出し元でサンプルされた数
* (function内でサンプルされた数 + functionの呼び出し元でサンプルされた数) / 全サンプル数
* function名

となっています。これで大まかな傾向はつかむことができます。しかし今回の場合、Crystalのメソッドが見られないですね。特定のメソッドの詳細を追いたい場合は、`list`コマンドを使用します。`initialize`メソッドの詳細を表示してみましょう。

```
!!! cmd
(pprof) list initialize
Total: 2566 samples
ROUTINE ==== *ExampleClass#initialize:Array in /root/dev/example.cr
  0   2518 Total samples (flat / cumulative)
  .      .   32: class ExampleClass
  .      .   33:   @instance_var1 : Array(Time)
  .      .   34:   @instance_var2 : Array(Time)
  .      .   35:
  .      .   36:   def initialize
---
  .   1670   37:     @instance_var1 = method1
  .    848   38:     @instance_var2 = method2
  .      .   39:   end
---
  .      .   40:
  .      .   41:   def method1
  .      .   42:     (1..1000000).map { |i| Time.new }
  .      .   43:   end
  .      .   44:

--snip--
```

Crystalのコードが表示されました。かつ、各行でのサンプル数も表示されています。こちらは左から、

* その行自体のサンプル数（サブメソッドのサンプル数は省く）
* その行の中で呼ばれているサブメソッド全ての累積サンプル数

となっています。method1とmethod2は繰り返し回数が２倍違うので、累積サンプル数も約２倍違うことがわかります。これは行レベルでのボトルネック検出に大いに役立ちそうです。次に`HEAP PROFILER`も見てみましょう。


```
!!! cmd
$ google-pprof ./example profile/heap.prof.*
Using local file ./example.
Using local file profile/heap.prof.0001.heap.
Welcome to pprof!  For help, type 'help'.
(pprof)
```

`HEAP PROFILER`の場合、プロファイリング結果のファイルが連番で出力されるのでワイルドカードで指定します。使い方は`CPU PROFILER`の時と同じです。

```
!!! cmd
(pprof) top
Total: 0.0 MB
  0.0  45.3%  45.3%  0.0 100.0% tzset
  0.0  31.2%  76.6%  0.0 100.0% adjtime
  0.0  23.4% 100.0%  0.0  23.4% __strdup
  0.0   0.0% 100.0%  0.0 100.0% __crystal_main
  0.0   0.0% 100.0%  0.0 100.0% __libc_start_main
  0.0   0.0% 100.0%  0.0 100.0% _start
  0.0   0.0% 100.0%  0.0 100.0% compute_offset
  0.0   0.0% 100.0%  0.0 100.0% compute_ticks_and_offset
  0.0   0.0% 100.0%  0.0 100.0% initialize
```

左から、

* function内で使用されたメモリ(MB)
* function内で使用されたメモリ(MB) / 全使用メモリ(MB)
* (function内で使用されたメモリ(MB) / 全使用メモリ(MB))の累積
* function内で使用されたメモリ(MB) + functionの呼び出し元で使用されたメモリ(MB)
* (function内で使用されたメモリ(MB) + functionの呼び出し元で使用されたメモリ(MB)) / 全使用メモリ(MB)
* function名

となっています。続いてlistコマンド。

```
!!! cmd
(pprof) list initialize
Total: 0.0 MB
ROUTINE ==== *ExampleClass#initialize:Array in /root/dev/example.cr
 0.0    0.0 Total MB (flat / cumulative)
   .      .   33:   @instance_var1 : Array(Time)
   .      .   34:   @instance_var2 : Array(Time)
   .      .   35:   @instance_var3 : Array(String)
   .      .   36:
   .      .   37:   def initialize
---
   .    0.0   38:     @instance_var1 = method1
   .    0.0   39:     @instance_var2 = method2
   .      .   40:     @instance_var3 = method3
   .      .   41:   end
---
   .      .   42:
   .      .   43:   def method1
   .      .   44:     (1..1000000).map { |i| Time.new }
   .      .   45:   end
   .      .   46:

--snip--
```

左から、

* その行自体の使用メモリ(MB)（サブメソッドの使用メモリは省く）
* その行の中で呼ばれているサブメソッド全ての累積使用メモリ(MB)

となっています。今回の例ではHEAPの使用はほぼ見られませんでしたが、巨大なプログラムになるとボトルネックの検出には便利そうです。

このように、gperftoolsを用いれば行レベルでのプロファイリングが可能です。他にも、CPUサンプルのFrequencyを変更できたり、グラフの出力ができたりと、プロファイリングには重宝するツールだと思うのでぜひgperftools公式ページを追ってみてください。ともあれ、これらを簡単に使用できるのもCrystalのCバインディングのおかげだと思うので、この機能を実装してくれた開発者の方に感謝です。

# まとめ

いかがでしたでしょうか。駆け足で説明しましたが、Crystalでもさまざまな計測ができることがわかりました。ぜひ、比較の際に参考にしてみてください。

2017年はCrystalにとって特別な年になりそうです。version 1.0 の公開が宣言されました。また、Parallelismの開発にも期待が寄せられています。Windowsのサポートも前進される予定ですし、Incremental compilation の実装も進んでいくでしょう。海外ではCode Campなどのイベントが積極的に開催されています。動画も公開されており、徐々に日本にも情報が入ってきています。興味のある方は、今後のCrystalの動向をウォッチしてみてください。

Happy crystalling!
