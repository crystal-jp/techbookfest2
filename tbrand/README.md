Title: Crystalのマクロを利用しライブラリ作成
Author: Taichiro Suzuki / @tbrand_dev

# Crystalのマクロを利用しライブラリ作成
初めまして、@taichiro_devと申します。
仕事では主にスマホアプリのCI/CD周りを担当しています。
これまで言語自体にはそんなに関心がなかった私でも、Crystalには初めて知ったときからずっと魅了されてきました。
Crystal自体の魅力は一旦置いておいて、今回は主にCrystalのマクロの魅力について私なりに書かせていただきたいと思います。

## マクロでできること
マクロとは簡単に言うとプログラムを書くプログラムです。
それにより記述量を減らしたり、(そうなるとは限りませんが)可読性・抽象度が向上します。

なおマクロはコンパイル時にロジックを展開することで高速化にも使用できます。
今回はこちらについては言及致しません。

## Crystalのマクロを書いてみる
では早速、実際にマクロを書いてみます。
まずはとてもくだらない例を示します。
次のような関数について考えます。
```
def hoge
  puts "HOGE!"
end
```
hogeを呼び出せばHOGE!と表示される関数です。
マクロはプログラムを書くプログラムなのですから、当然このhoge関数を書くこともできます。
次のように書きます。
```
macro define_hoge
  def hoge
    puts "HOGE!"
  end
end

define_hoge

hoge
# => "HOGE!"
```
このdefine_hogeというのがマクロです。
define_hogeマクロを呼び出すことで、中身(ここではhoge関数)をソースコードに埋め込んでいます。

ただこれでは冗長なだけで、何のためにマクロがあるのか全くわかりません。
もう少し実用的なマクロを書いてみましょう。
String型(`"ok"`)の変数をSymbol型(`:ok`)に変換しようと思います。
```
macro string_to_symbol(str)
  :{{str.id}}
end

ok = string_to_symbol("ok")
puts typeof(ok)
# => Symbol
```
string_to_symbolマクロはstrを引数に取ります。
strは任意のStringです。(実はStringではなくても良いです。)
マクロの中では引数を{{}}で囲むことでその引数を使用する(ソースコードに埋め込む)ことができます。
.idというのは簡単に言うと、引数の中身を取り出しています。
つまり:{{str.id}}は、strの中身を取り出して先頭に:(コロン)をつける、ということを行っています。
これで晴れて"ok"は:okとなるわけです。

最後の例として、マクロの中でfor文を使用したいと思います。
次のように使用します。
```
macro def_num_method(*nums)
  {% for n in nums %}
    def puts_{{n.id}}
      puts "{{n.id}}"
    end
  {% end %}
end

def_num_method(1, 2, 3)

# マクロによって以下のようにメソッドが定義される
#
# def puts_1
#	  puts "1"
# end
#
# def puts_2
#	  puts "2"
# end
#
# def puts_3
#	  puts "3"
# end

puts_1 # => 1
puts_2 # => 2
puts_3 # => 3
```
だいたい何をやっているかはわかると思います。
注意として、{% %}で囲まれた公文は{{ }}とは違いソースコードには埋め込まれません。
また、{% %}を駆使すればCrystalのコードをなんでも書けるというわけではありません。
残念ながら、マクロで使用出来る構文は限られています。
興味を持っていただいた方はCrystalのソースコードを読んでみてはいかがでしょうか。
Crystalはセルフホスティングなので、ソースコードが読みやすいというのも魅力の一つです。

Crystalが標準で用意しているマクロの例を一部紹介します。
getterというマクロはRubyで言えばattr_readerで、その名の通りgetterを定義する時に使用します。
例えば、Dogクラスにnameという属性があったとして、getterを定義する時は以下のように使用します。
```
class Dog
  getter name
end
```
このgetterマクロでは以下のように関数が展開されます。
関数を見ればgetterなのは一目瞭然ですよね。
```
def name
  @name
end
```
同様にsetterや、getterとsetterを同時に展開するpropertyというマクロもあります。
さらに、recordというマクロもあります。
こちらは構造体をシンプルに定義する時に便利です。
次のように使用します。
```
record Dog, name : String, age : Int32

dog = Dog.new("Pochi", 8)
puts dog.name # => Pochi
puts dog.age  # => 8
```

## ライブラリの開発・設計にどう使用するか
結論から言うと、私自身も試行錯誤行っている段階で、まだ正解を得られていません。
なので、ボトムアップに説明することは難しいです。
実際に私が開発しているtopaz-crystal/topazというライブラリでどのようにマクロを使用しているかを具体例で説明したいと思います。
なお、topaz自体の機能については最後に簡単に説明したいと思います。
また以下の説明では、簡単のためにライブラリで使用されている実際のコードとは違うコードになっているところもあります。m(__)m

ではまず、ユーザーに継承してもらう親クラスを定義していきましょう。
これは「親クラスにマクロを定義し、それを継承した子クラスがそのマクロを呼び出す」という設計にするためです。
もちろんマクロはグローバルに定義することも可能ですが、名前の衝突などを考慮しなるべく空間を区切った方が良いと思います。
(逆に、どこからもアクセス可能なマクロである必要がある場合もありますので、一概には言えません。)
この後の例では、ライブラリ側のクラスをModel、ユーザー側のクラスをUserとして記述します。
```
class Model
end

class User < Model
end
```
これでModelは子クラスにマクロを提供できるようになり、またUserはModelのマクロを利用できるようになりました。

ではここからは具体例を示していきます。
ModelクラスはUserの属性を受け取り、それをもとに必要な関数をすべて展開する、ということにします。
そこでまず、Modelクラスに属性を受け取るマクロを定義しましょう。
なお簡単のため、属性はすべてString型とします。
```
class Model
  macro attrs(*vals)
  end
end

class User < Model
  attrs(name, gender)
end
```
これでModelクラスはUserクラスから属性を受け取ることができるようになりました。
ここではUserクラスはModelクラスにnameとgenderを渡しています。
ではこれらの情報をもとに関数を展開しましょう。
まずはコンストラクタを展開しようと思います。
せっかく属性をすべて渡しているのですから以下のように書きたいですよね。
```
user = User.new("tbrand", "man")
```
ではそのように書けるようにマクロを書いてみましょう。
```
class Model
  macro attrs(*vals)
    def initialize(
      {% for val in vals %}
        @{{val.id}} : String,
      {% end %}
      )
    end
  end
  
  # 展開後の例
  # def initialize(@name : String, @gender : String)
  # end
end
```
一気に見辛くなりました。
ここではコンストラクタ(initialize)の引数をマクロで展開しています。
{% for ... %}ですべての属性を参照して、initializeの引数に@{{val.id}}を埋め込んできます。
@{{val.id}}はvalの中身を取り出して、先頭に@をつけてクラス変数にしています。
: Stringは@nameと@genderは文字列型だよ、ということをコンパイラに教えてあげています。
Crystalは静的に型を定義する言語なので、このように明示的に型を示す必要がある場合があります。
コンパイラがコードの他の箇所から型が推測できるようであれば、このような指定は必要ありません。

なお、Crystalでは関数の引数にクラス変数を定義することで、その後改めて代入する必要がなくなります。
すなわち、次の2つの関数は等しいです。
```
def set_name(@name)
end

def set_name(name)
  @name = name
end
```
実は先ほど紹介したrecordもマクロでコンストラクタを展開しています。
recordのコンストラクタは上のModelクラスよりももっとフレキシブルなコンストラクタを利用できます。
例えば次のように定義できます。
```
record Dog, name : String, age : Int32
```
recordクラスではマクロでどのようにコンストラクタを定義しているのか、興味のある方はぜひソースコードで確認してみてください。

コンストラクタで属性を定義したので、getterが欲しくなりますよね。
getterはそのまま以下のように展開しましょう。
```
class Model
  macro attrs(*vals)
    {% for val in vals %}
      getter {{val.id}} : String
    {% end %}
  end

  # 展開後の例
  # getter name : String
  # getter gender : String
end
```

これでuser.nameのように属性の値を取得できるようになりました。
さて、モデルをまるごとHashに変換するto_h関数を定義してみましょう。

```
class Model
  macro attrs(*vals)
    def to_h
      {
        {% for val in vals %}
          "{{val.id}}" => @{{val.id}},
        {% end %}
      }
    end
  end

  # 展開後の例
  # def to_h
  #   {
  #     "name" => @name,
  #     "gender" => @gender,
  #   }
  # end
end
```

ここまで理解できれば、なんでもマクロで書ける気がしてきませんか。
最後に、マクロを表現力を体現するような例を示したいと思います。
getterとは別に属性名(String)からクラス変数の値を取得できるようにしてみましょう。
具体的には、次のようなものです。
```
puts user.get_value_of("name") # => "man"
```
そもそもこの関数を何に使うかは置いといて、これはマクロでないと実現が難しそうですよね。
具体的にはこうなります。
```
class Model
  macro attrs(*vals)
    def get_value_of(attr : String)
      case attr
          {% for val in vals %}
          when "{{val.id}}"
            return @{{val.id}}
          {% end %}
      else
        return nil
      end
    end  
  end
  
  # 展開後の例
  # def get_value_of(attr : String)
  #   case attr
  #   when "name"
  #     return @name
  #   when "gender"
  #     return @gender
  #   else
  #     return nil
  #   end
  # end
end
```

いかがでしたでしょうか。
このようにマクロが利用できるとプログラムの表現力が大幅に増大し、ユーザーフレンドリーな設計をすることも可能になります。
マクロはRubyにはないCrystalの大きな強みですので、ぜひとも完全に習得したいものです。
しかし自責の念を含めて書きますが、マクロを多用すると可読性が落ちることは明らかです。
必要最低限のマクロにとどめることをお勧めします。m(__)m
最後に実際に動くコードを添付いたします。
```
# ライブラリとして使用するクラス
class Model
  macro attrs(*vals)

    # getterを展開
    {% for val in vals %}
      getter {{val.id}} : String
    {% end %}

    # コンスタクタを展開
    def initialize(
          {% for val in vals %}
            @{{val.id}} : String,
          {% end %}
        )
    end

    # ハッシュ化する関数
    def to_h
      {
        {% for val in vals %}
          "{{val.id}}" => @{{val.id}},
        {% end %}
      }
    end

    # 属性"名"から属性値を取得する
    def get_value_of(attr : String)
      case attr
          {% for val in vals %}
          when "{{val.id}}"
            return @{{val.id}}
          {% end %}
      else
        return nil
      end
    end
  end
end

# 以下はユーザーが利用するケース
class User < Model
  # マクロを使用し、クラス内に関数を展開
  attrs(name, gender)
end

user = User.new("tbrand", "man")

puts user.name
# => tbrand
puts user.gender
# => man
puts user.to_h
# => {"name" => "tbrand", "gender" => "man"}
puts user.get_value_of("name")
# => tbrand
```

## Topazでマクロを使用した理由
TopazはActiveRecordのデザインパターン(ぽい)のO/Rマッパー(ぽい)ものです。
まだ完全に発展途上で、ようやく一通り動くようになった程度です。
こちらのライブラリでは設計にマクロを使用し、なるべくユーザーフレンドリーになるよう心がけています。
Crystalは以下のようにDBからデータを引っ張ります。
```
db.query "select * from users" do |rs|
  rs.each do
    rs.read(String)
    rs.read(Int32)
    rs.read(String)
    rs.read(String)
  end
end
```

このように、モデルの構造(カラム)を完全に掌握しないとデータの取得を抽象化することは難しいです。
Railsはコマンドからモデルを作成することでこの問題を解決しています。
しかし、マクロを使えばコンパイル直前までモデルの構造を把握していなくても適切な関数を展開できるので抽象化を実現できます。
```
db.query some_query do |rs|
  rs.each do
	{% for column in columns %}
      rs.read {{column[:type]}}
	{% end %}
  end
end
```
私からのマクロの解説はここで終わりにさせいていただきます。
ここまで読んでいただいてありがとうございました。
ぜひとも未熟なTopazのソースコードを読んでいただいて、PRいただけたら幸いです。m(__)m
