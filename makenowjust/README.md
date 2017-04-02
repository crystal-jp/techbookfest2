Title: Crystalのコンパイラを改造してみよう
Author: MakeNowJust
Twitter: @make_now_just

# はじめに

こんにちは、MakeNowJustです。`crystal tool expand`などを実装しました、Crystalのコントリビューターです。よろしくお願いします。

さて、あるプログラミング言語のコンパイラがその言語自身で書かれていることを、コンパイラが**セルフホスティング**されているといいます。例えば、バージョン1.5以降のGo言語は完全にGo言語で記述されていて、セルフホスティングされているといって差し支えないでしょう。

同じように、CrystalのコンパイラもCrystal自身で書かれています。そのため、C言語やC++などで書かれている他のコンパイラと比較して、Crystalのコンパイラは改造が容易に行えます。（もちろん、Crystalのコードが読めれば、という話ですが）

そこで、この記事ではCrystalのコンパイラの構造を簡単に説明して、Crystalに新しい構文（虚数リテラル）を実装する方法を紹介したいと思います。

この記事は、抽象構文木やパーサーなどコンパイラの実装に関する基本的な用語は理解できる程度の人を対象にしています。また、バージョン`0.21.1`のCrystalを利用しています。

# コンパイラの構造

## `--stats`オプション

Crystalのコンパイラがどのように動いているかを大雑把に知りたいのであれば、コンパイラのソースコードを読み始めるよりもまず、`crystal run --stats`あるいは`crystal build --stats`を実行するといいでしょう。`--stats`はコンパイルの各フェーズでかかった時間やその他の情報を出力するオプションで、適当なファイルに対して実行してみると、次のような出力が得られるはずです。

```
!!!cmd
$ echo 'p "hello world"' > hello.cr
$ crystal run --stat hello.cr
Parse:                             00:00:00.0047250 (   0.19MB)
Semantic (top level):              00:00:00.2320850 (  26.18MB)
Semantic (new):                    00:00:00.0020790 (  34.18MB)
Semantic (type declarations):      00:00:00.0221140 (  34.18MB)
Semantic (abstract def check):     00:00:00.0017410 (  34.18MB)
Semantic (ivars initializers):     00:00:00.0180530 (  34.18MB)
Semantic (cvars initializers):     00:00:00.0257990 (  34.18MB)
Semantic (main):                   00:00:00.0839630 (  50.18MB)
Semantic (cleanup):                00:00:00.0010550 (  50.18MB)
Semantic (recursive struct check): 00:00:00.0012470 (  50.18MB)
Codegen (crystal):                 00:00:00.2100430 (  50.24MB)
Codegen (bc+obj):                  00:00:00.4529360 (  50.24MB)
Codegen (linking):                 00:00:00.2989280 (  50.24MB)

Codegen (bc+obj):
 - no previous .o files were reused
"hello world"
Execute:                           00:00:00.0130500 (  50.24MB)
```

色々と出力されていますが、まず先頭の文字列に注目すると、`crystal run`は`Parse`・`Semantic`・`Codegen`・`Execute`の4つのフェーズに大別されることが分かります。そして`Semantic`や`Codegen`のフェーズの中ではいくつもの処理が行われています。

各フェーズの役割について簡単に説明していきます。

  - `Parse`フェーズでは、言葉の通り与えられたプログラムの構文解析を行います。
  - `Semantic`フェーズでは型チェックやマクロの展開など、コンパイルのためのプログラムの意味解析を行います。いくつかの細かい処理に分かれていますが、重要なのは`(top level)`と`(main)`の二つです。`(top level)`でクラスやメソッドの定義を集め、その後`(main)`で実行されるプログラムの型チェックなどを行います。
  - `Codegen`フェーズでは、LLVMを使ってコード生成をします。
  - `Execute`フェーズで生成したファイルを実行します。（このフェーズは`crystal run`のときにだけあります。）

## ソースコード

次に、Crystalのコンパイラのソースコードがどこにあるかを説明していきます。

Crystalのソースコード（標準ライブラリ含む）は[GitHubで公開](https://github.com/crystal-lang/crystal)されています。そして、コンパイラのソースコードは`/src/compiler/crystal/`以下に、コンパイラのテスト（スペック）は`/spec/compiler/`以下にあります。

このリポジトリをクローンしてきて`/src/compiler/crystal/`を`ls`すると、次のようなファイルがあることが分かると思います。
（Crystalのリポジトリはそこそこ大きいので素早くクローンするために`--depth=1`オプションを付けています。また、今回は2017/03/20現在の最新版である`2065548`をクローンしてきています。以降もこのリビジョンをベースに作業していきます。）

```
!!!cmd
$ git clone --depth=1 \
    https://github.com/crystal-lang/crystal
$ cd crystal
$ ls -F --group-directories-first \
    src/compiler/crystal/
codegen/  semantic/  command.cr   crystal_path.cr  macros.cr   syntax.cr
command/  syntax/    compiler.cr  exception.cr     program.cr  types.cr
macros/   tools/     config.cr    formatter.cr     semantic.cr util.cr
```

これらのファイルやディレクトリのうち、重要なものをいくつか紹介します。

- `command.cr`： `Crystal::Command`クラスが定義されています。このクラスは`crystal`コマンドの実質的なエントリポイントです。（本当のエントリポイントは`/src/compiler/crystal.cr`ですが、このファイルは実質的に`Crystal::Command.run`を呼んでいるだけです）
- `command/`: `crystal`コマンドの一部のサブコマンド（`crystal tool format`や`crystal spec`など）のエントリポイントがこのディレクトリ以下にあります。
- `tools/`: `crystal tool format`や`crystal init`、`crystal play`などのサブコマンドの実装がこのディレクトリ以下にあります。
- `compiler.cr`: `Crystal::Compiler`クラスが定義されています。このクラスはソースコードをパースして抽象構文木を作り、それに各種処理を適用して最終的に実行ファイルを作るまでの一連の処理をまとめています。
- `program.cr`: `Crystal::Program`クラスが定義されています。このクラスはプログラムの環境を表わしていて、トップレベルのクラスやメソッドの定義を保持しています。このクラスのインスタンスは様々なところで使い回されることになります。
- `types.cr`: 型を表すクラスが定義されています。
- `syntax/`: パーサーの実装や抽象構文木の定義がこのディレクトリ以下にあります。
- `semantic.cr`, `semantic/`: `Semantic`フェーズの各処理の実装がここにあります。
- `codegen.cr`, `codegen/`: `Codegen`フェーズの各処理の実装がここにあります。

ここにあるファイルを弄っていくことで、コンパイラを改造していきます。

## コンパイラのビルド

コンパイラの改造に入る前に、コンパイラのビルド方法も確認しておきたいと思います。

コンパイラのビルドにはCrystal以外に、[こちら](https://github.com/crystal-lang/crystal/wiki/All-required-libraries)に書いてあるライブラリが必要になります。インストール方法もそのページに書かれているので参考にしてください。

依存ライブラリをインストールしたら、`make`コマンドを叩いてビルドを実行するだけです。また、ビルドしてできたファイルを削除したい場合は`make clean`を実行します。また、コンパイラのテストをするには`make compiler_spec`、標準ライブラリのテストをするには`make std_spec`を実行します。（`make spec`も存在しますが、膨大なメモリと実行時間を必要とするので注意してください。）

```
!!!cmd
●図1::Crystalのmake
(注:コンパイラをビルドする)
$ make
(注:ビルドしてできたファイルを削除する)
$ make clean
(注:コンパイラのテストを実行する)
$ make compiler_spec
(注:標準ライブラリのテストを実行する)
$ make std_spec
```

その他の`Makefile`のターゲットは`make help`で確認してください。

# コンパイラを改造

ここで説明しているコードは全て、GitHubのMakeNowJust/crystalの`feature/syntax/imaginary-number-j`ブランチに置いてあります。場合に応じて参考にしてください。

## 虚数リテラル

それではコンパイラを改造していきたいと思います。まず、虚数リテラルとはどのようなものかについて説明したいと思います。

虚数リテラルはRubyやPythonなどにあるもので、`1i`（Ruby）や`1j`（Python）とすることで複素数の値を作ることができるものです。科学計算の分野ではあると重宝するのかもしれません。

今回は、Pythonと同じように`1j`と数値リテラルのあとに`j`を付ける構文を実装します。どうしてRubyと同じにしないのかというと、`i`をサフィックスにすると`1i32`などの数値リテラルのサイズを表す構文と衝突して実装がややこしくなるからです。

というわけで、次のようなプログラムが動くようことを目標とします。

```crystal
p (1 + 2j) == Complex.new(1, 2) # => true
```

## 抽象構文木の追加

新しい構文を追加する際に最初にすることは、その構文に対応する抽象構文木の定義を追加することです。

抽象構文木の定義は仮想的なものを除いて全て`syntax/ast.cr`にあります。また、`Crystal::ASTNode`抽象クラスが全ての抽象構文木の基底クラスとなります。そこで、`syntax/ast.cr`の`NumberLiteral`の定義の後に`ImaginaryNumberLiteral`というクラスを定義します。（リスト1）
（リスト1やこれ以降に出てくるコードは`diff`コマンドの出力になっています。）

`ImaginaryNumberLiteral`は`number`というプロパティを持つクラスです。`number`プロパティには抽象構文木が格納されますが、これは虚数の値の部分を表わしています。

```diff
●リスト1::syntax/ast.cr
--- a/src/compiler/crystal/syntax/ast.cr
+++ b/src/compiler/crystal/syntax/ast.cr
@@ -211,6 +211,23 @@ module Crystal
     def_hash value, kind
   end

+  # An imaginary number literal.
+  #
+  #     1j
+  #
+  class ImaginaryNumberLiteral < ASTNode
+    property number
+
+    def initialize(@number : ASTNode)
+    end
+
+    def clone_without_location
+      ImaginaryNumberLiteral.new @number.clone
+    end
+
+    def_equals_and_hash number
+  end
+
   # A char literal.
   #
   #     "'" \w "'"
```

また、この`ImaginaryNumberLiteral`の追加に合わせて`syntax/to_s.cr`と`syntax/transformer.cr`にも変更を加えます。（リスト2）

`syntax/to_s.cr`は抽象構文木を元のソースコードに戻す処理が定義されています。`syntax/transformer.cr`には抽象構文木を走査して変更を加えるための`Crystal::Transformer`というクラスが定義されています。これらの変更点は周りに合わせてやるだけです。

```diff
●リスト2::syntax/to_s.cr, syntax/transformer.cr
--- a/src/compiler/crystal/syntax/to_s.cr
+++ b/src/compiler/crystal/syntax/to_s.cr
@@ -79,6 +79,11 @@ module Crystal
       true
     end

+    def visit(node : ImaginaryNumberLiteral)
+      node.number.accept self
+      @str << "j"
+    end
+
     def visit(node : CharLiteral)
       node.value.inspect(@str)
     end
--- a/src/compiler/crystal/syntax/transformer.cr
+++ b/src/compiler/crystal/syntax/transformer.cr
@@ -405,6 +405,11 @@ module Crystal
       node
     end

+    def transform(node : ImaginaryNumberLiteral)
+      node.number = node.number.transform self
+      node
+    end
+
     def transform(node : CharLiteral)
       node
     end
```

これらを定義することで、一応エラーを出さずにコンパイラをビルドすることができるようになります。しかし、まだこのクラスを使っていないので特に何も変化がありません。

## パーサー

パーサーを改造して、虚数リテラルの構文を解析して`ImaginaryNumberLiteral`を構築するようにします。

パーサーは`syntax/parser.cr`に実装されています。基本的には単純な再帰下降パーサーなのですが、なにせ5000行ほどあるので全てを理解するのは骨が折れます。そこで、一部にしぼって見ていきます。

虚数リテラルは数値リテラルを拡張したものなので、数値リテラルを解析しているところでそれに続いて`j`があるかどうかをチェックすればいいでしょう。なので数値リテラルを表す抽象構文木のクラスである`NumberLiteral`で検索をかけてみます。すると最初に`parse_atomic_without_location`というメソッドの中で`NumberLiteral.new`をしている場所（853行目）が見つかります。他にもいくつか`NumberLiteral`を使っている部分はありますが、それは型の一部であったり特殊な使われ方をしているため今回は関係がありません。この`parse_atomic_without_location`の`NumberLiteral.new`の後に、虚数リテラルでないかチェックする処理を記述します。（リスト3）

`@token`には現在のトークンが格納されています。それが`j`であるかどうかを確認して、そうだった場合は`ImaginaryNumberLiteral`を作ります。これだと数値リテラルと`j`の間に空白があった場合も虚数リテラルになってしまいそうで薄気味悪いですが、Crystalは空白がプログラムの意味を変えることがあるために、字句解析器が空白をトークンとして返すことがあるので問題は起こりません。

```diff
●リスト3::syntax/parser.cr
--- a/src/compiler/crystal/syntax/parser.cr
+++ b/src/compiler/crystal/syntax/parser.cr
@@ -850,7 +850,11 @@ module Crystal
         parse_attribute
       when :NUMBER
         @wants_regex = false
-        node_and_next_token NumberLiteral.new(@token.value.to_s, @token.number_kind)
+        node = node_and_next_token NumberLiteral.new(@token.value.to_s, @token.number_kind)
+        if @token.type == :IDENT && @token.value == "j"
+          node = node_and_next_token ImaginaryNumberLiteral.new node
+        end
+        node
       when :CHAR
         node_and_next_token CharLiteral.new(@token.value.as(Char))
       when :STRING, :DELIMITER_START
```

これでコンパイラは虚数リテラルを、構文としては解釈できるようになります。しかし、この段階でビルドしたもので虚数リテラルを含むものを実行しようとしても、虚数リテラルが処理されないので上手く動作しません。

```
!!!cmd
$ make
$ echo "p (1 + 2j) == Complex.new(1, 2) # => true" > imlit.cr
$ ./bin/crystal run imlit.cr
Using compiled compiler at .build/crystal
can't execute `p(1j)` at ./imlit.cr:1:1: `1j` has no type (Exception)
0x100657675: *CallStack::unwind:Array(Pointer(Void)) at ??
0x100657611: *CallStack#initialize:Array(Pointer(Void)) at ??
0x1006575e8: *CallStack::new:CallStack at ??
0x1006518e1: *raise<Exception>:NoReturn at ??
0x1006518c1: *raise<String>:NoReturn at ??
0x1006513ff: __crystal_main at ??
0x1006568a8: main at ??
```

## Semantic処理

次は`ImaginaryNumberLiteral`がコンパイルできるようにするために`Semantic`フェーズの処理を実装します。

ところで、Crystalの配列リテラルやハッシュリテラルは、内部的に対応するクラスを`new`して値を追加する、というコードに展開されています。具体例を上げると、リスト4のようなコードは、内部的にはリスト5のようなコードに展開されます。

```crystal
●リスト4::配列リテラル、ハッシュリテラルの元のコード
array = [1, 2, 3]

hash = {"hello" => "world"}
```

```crystal
●リスト5::配列リテラル、ハッシュリテラルの展開後のコード
array = Array(typeof(1, 2, 3)).new(3) do |buffer|
  buffer[0] = 1
  buffer[1] = 2
  buffer[2] = 3
end

hash = Hash(typeof("hello"), typeof("world")).new
hash["hello"] = "world"
hash
```

なので、`ImaginaryNumberLiteral`でも同様に`1j`を`Complex.new(0, 1)`のように展開する方針で進めていきたいと思います。

この変換は`semantic/literal_expander.cr`で実装されているのですが、その前に`semantic/ast.cr`を弄って`ImaginaryNumberLiteral`が展開可能なことを意味するモジュールである`ExpandableNode`を`include`するようにします。（リスト6）

```diff
●リスト6::semantic/ast.cr
--- a/src/compiler/crystal/semantic/ast.cr
+++ b/src/compiler/crystal/semantic/ast.cr
@@ -564,14 +564,15 @@ module Crystal

   module ExpandableNode
     property expanded : ASTNode?
   end

   {% for name in %w(And Or
                    ArrayLiteral HashLiteral RegexLiteral RangeLiteral
+                   ImaginaryNumberLiteral
                    Case StringInterpolation
                    MacroExpression MacroIf MacroFor MultiAssign
                    SizeOf InstanceSizeOf Global Require Select) %}
     class {{name.id}}
       include ExpandableNode
     end
   {% end %}
```

そして`semantic/literal_expander.cr`に`ImaginaryNumberLiteral`に対する展開の実装を追加して、`semantic/main_visitor.cr`をそれを呼び出すように修正します。（リスト7）

`ImaginaryNumberLiteral`の`expand`メソッドで行っていることは単純で、`1j`ならば`Complex.new(0, 1)`に対応するような構文木を組み立てているだけです。また、`semantic/main_visitor.cr`の`expand`メソッドはこれを直接呼び出すわけではなく、これの返り値を`expanded`プロパティに代入するようなものになっています。

```diff
●リスト7::semantic/literal_expander.cr, semantic/main_visitor.cr
--- a/src/compiler/crystal/semantic/literal_expander.cr
+++ b/src/compiler/crystal/semantic/literal_expander.cr
@@ -659,6 +659,22 @@ module Crystal
       Call.new(Path.global(["Regex", "Options"]).at(node), "new", NumberLiteral.new(node.options.value).at(node)).at(node)
     end

+    # Convert a imaginary number literal to a call:
+    #
+    # From:
+    #
+    #     1j
+    #
+    # To:
+    #
+    #     Complex.new(0, 1)
+    #
+    def expand(node : ImaginaryNumberLiteral)
+      path = Path.global("Complex").at(node)
+      zero = NumberLiteral.new("0.0", :f64)
+      Call.new(path, "new", zero, node.number).at(node)
+    end
+
     def expand(node)
       raise "#{node} (#{node.class}) can't be expanded"
     end
--- a/src/compiler/crystal/semantic/main_visitor.cr
+++ b/src/compiler/crystal/semantic/main_visitor.cr
@@ -2717,6 +2717,10 @@ module Crystal
       node.type = program.type_from_literal_kind node.kind
     end

+    def visit(node : ImaginaryNumberLiteral)
+      expand(node)
+    end
+
     def visit(node : CharLiteral)
       node.type = program.char
     end
```

`ExpandableNode`を実際に展開する処理やそこからのコード生成は元から実装されているので、これでコンパイラ側の実装は全て完了しました。しかし、現在のCrystalは`complex`を標準で`require`していないので、このままだと`Complex`クラスが見つからないと言われます。なので、`/src/prelude.cr`に`require "complex"`を追加します。（リスト8）

```diff
●リスト8::/src/prelude.cr
--- a/src/prelude.cr
+++ b/src/prelude.cr
@@ -26,6 +26,7 @@ require "box"
 require "char"
 require "char/reader"
 require "class"
+require "complex"
 require "concurrent"
 require "deque"
 require "dir"
```

これでようやく先程のコードが実行できるようになりました。

```
!!!cmd
$ make
$ ./bin/crystal run imlit.cr
Using compiled compiler at .build/crystal
true
```

## フォーマッタ

と、これでコンパイラの実装は完了したかのように見えますが、他にもまだやることが残っています。`crystal tool format`は現状`ImaginaryNumberLiteral`には対応していないので、虚数リテラルを含むコードに対して実行するとエラーが起こります。

```
!!!cmd
$ ./bin/crystal tool format imlit.cr
Using compiled compiler at .build/crystal
Error:, couldn't format 'imlit.cr', please report a bug including the contents of it: https://github.com/crystal-lang/crystal/issues
```

なので、フォーマッタを修正します。（リスト9）フォーマッタは`tools/formatter.cr`にあります。

基本的には`ToSVisitor`に追加したものと同じような処理になっています。しかしフォーマッタは`ToSVisitor`とは違い、抽象構文木を辿りながら字句解析も同時に行っていくことで、元のコードのコメントなどを保ったまま抽象構文木を文字列にしていきます。そこで、最初に数値部分を処理したあと、新たに定義した`check_ident`というメソッドで`j`が正しく字句解析されていることを確認したあとに、それを出力して字句解析を次に進めています。

```diff
●リスト9::tools/formatter.cr
--- a/src/compiler/crystal/tools/formatter.cr
+++ b/src/compiler/crystal/tools/formatter.cr
@@ -393,6 +393,15 @@ module Crystal
       false
     end

+    def visit(node : ImaginaryNumberLiteral)
+      node.number.accept self
+      check_ident "j"
+      write "j"
+      next_token
+
+      false
+    end
+
     def visit(node : StringLiteral)
       @last_is_heredoc = false

@@ -4488,6 +4497,11 @@ module Crystal
       raise "expecting #{token_type}, not `#{@token.type}, #{@token.value}`, at #{@token.location}" unless @token.type == token_type
     end

+    def check_ident(token_value)
+      check :IDENT
+      raise "expecting `#{token_value}`, not `#{@token.value}`, at #{@token.location}" unless @token.value == token_value
+    end
+
     def check_end
       if @token.type == :";"
         next_token_skip_space_or_newline
```

これでフォーマッタは虚数リテラルに対しても正常に動作するようになります。

## テスト

最後に虚数リテラルに対してもろもろのテストを書きます。（リスト10）本当は最初に書いた方がいいのかもしれませんが、面倒なので後回しにしてしまいました。

各スペックの説明はしませんので、名前の雰囲気で察してください。何となく分かると思います。（`assert_format`はフォーマッタのスペックなんだろうな、とか`it_parses`はパーサーのスペックなんだろう、とか）

```diff
●リスト10::spec/
--- a/spec/compiler/formatter/formatter_spec.cr
+++ b/spec/compiler/formatter/formatter_spec.cr
@@ -41,6 +41,7 @@ describe Crystal::Formatter do
   assert_format "0_u64", "0_u64"
   assert_format "0u64", "0u64"
   assert_format "0i64", "0i64"
+  assert_format "1j"

   assert_format "   1", "1"
   assert_format "\n\n1", "1"
--- a/spec/compiler/parser/parser_spec.cr
+++ b/spec/compiler/parser/parser_spec.cr
@@ -47,6 +47,10 @@ describe "Parser" do

   it_parses "2.3_f32", 2.3.float32

+  it_parses "1j", ImaginaryNumberLiteral.new(1.int32)
+  it_parses "3.14j", ImaginaryNumberLiteral.new(3.14.float64)
+  assert_syntax_error "1 j"
+
   it_parses "'a'", CharLiteral.new('a')

   it_parses %("foo"), "foo".string
--- a/spec/compiler/parser/to_s_spec.cr
+++ b/spec/compiler/parser/to_s_spec.cr
@@ -104,4 +104,5 @@ describe "ASTNode#to_s" do
   expect_to_s %(enum Foo\n  A = 0\n  B\nend)
   expect_to_s %(alias Foo = Void)
   expect_to_s %(type(Foo = Void))
+  expect_to_s %(1j)
 end
--- a/spec/compiler/semantic/primitives_spec.cr
+++ b/spec/compiler/semantic/primitives_spec.cr
@@ -21,6 +21,14 @@ describe "Semantic: primitives" do
     assert_type("2.3_f64") { float64 }
   end

+  it "types a Complex" do
+    assert_type(%(
+      require "prelude"
+
+      1j
+      )) { types["Complex"] }
+  end
+
   it "types a char" do
     assert_type("'a'") { char }
   end
```

`make compiler_spec`としてコンパイラのテストが正常に実行されれば、虚数リテラルの実装は完璧ということになります。（ただし、このスペックの実行はそこそこ時間がかかるので注意してください）

# 結びに

全体的にコード自体の解説の少ない文章で申し訳ありません。ただ、詳細を説明しなければいけないほど複雑なことをやっている部分はあまり無い気がするので問題はないと思います。

ちなみに、今回の虚数リテラルの変更点は追加が82行、削除が1行となっています（図2）。100行にも満たない変更で新しい構文を追加できる、というのは中々面白い話なのではないでしょうか？

```
!!!cmd
●図2::今回のdiff
$ git diff --stat master...feature/syntax/imaginary-number-j
 spec/compiler/formatter/formatter_spec.cr         |  1 +
 spec/compiler/parser/parser_spec.cr               |  4 ++++
 spec/compiler/parser/to_s_spec.cr                 |  1 +
 spec/compiler/semantic/primitives_spec.cr         |  8 ++++++++
 src/compiler/crystal/semantic/ast.cr              |  1 +
 src/compiler/crystal/semantic/literal_expander.cr | 16 ++++++++++++++++
 src/compiler/crystal/semantic/main_visitor.cr     |  4 ++++
 src/compiler/crystal/syntax/ast.cr                | 17 +++++++++++++++++
 src/compiler/crystal/syntax/parser.cr             |  6 +++++-
 src/compiler/crystal/syntax/to_s.cr               |  5 +++++
 src/compiler/crystal/syntax/transformer.cr        |  5 +++++
 src/compiler/crystal/tools/formatter.cr           | 14 ++++++++++++++
 src/prelude.cr                                    |  1 +
 13 files changed, 82 insertions(+), 1 deletion(-)
```

もしこの記事を読んでCrystalに何か新しい構文を実装しようと思っていただけたのなら、それは嬉しい限りです。

それでは、最後までお付き合いいただきありがとうございました。
