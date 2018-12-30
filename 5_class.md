# 5章 木曜日：クラス定義
rubyにおけるクラス定義はオブジェクトの動作を定義するだけでなく、実際にコードを実行する。
この章では、クラスを修正するクラスマクロとメソッドをコードでラップするアラウンドエイリアスを学ぶ。また左記の魔術を活用するために特異クラス(別名：シングルトンクラス)も学ぶ。

## 5.1.1 クラス定義の中身
クラス内にはメソッドだけではないいろんなコードを置くことができる。

```
class MyClass
  p 'hello'
end

# => 'hello'

result = class MyClass
  self
end

result # => MyClass
```

クラスやモジュール定義の中だと、クラスがカレントオブジェクトselfになる。

## 5.1.2 カレントクラス
カレントオブジェクトは `self` で参照できる、そしてカレントオブジェクトは常に存在する(selfはいつでも使えるということ)。
同様に常にカレントクラス(or カレントモジュール)も存在するがこれは、selfのような便利なキーワードがなく参照できない。しかしコードを見ればカレントクラスがなんなのかはわかる。

- プラグラムのトップレベルではカレントクラスはmainクラスのObjectになる(このためにトップレベルでメソッドを定義するとObjectのインスタンスメソッドになる)
- classキーワード(class Hoge ... end)をオープンすると、そのクラスがカレントクラスになる
  - classキーワードには**必ずクラス名が必要になる**
- メソッドの中ではカレントオブジェクトのクラスがカレントクラスになる。(self.class の結果がカレントクラスになる)

```rb
class C
  def m1
    def m2; end
  end
end

class D < C; end

obj = D.new
obj.m1 # => :m2
```

### class_eval
`Module#class_eval` (別名module_eval)はそこにあるクラスのコンテキストでブロックを評価する。

```rb
# クラス名を受け取ってメソッドを追加する
def add_method_to(a_class)
  a_class.class_eval do
    def m; 'hello'; end
  end
end

add_method_to(String)
# Stringクラスにメソッド m を追加した！
'aaa'.m # => 'hello'
```

class_evalはinstance_evalとは違い、selfとカレントクラスを変更する。上の例ではカレントクラスを引数で受け取ったStringに変更することで、Stringクラスをオープンして内部にメソッドを定義することができた。
class_evalはclassキーワードより柔軟らしい。classキーワードはクラス名の定数が必要だが、class_evalはクラスを参照している変数であればレシーバーとして使用できる。
後、classキーワードはその時点での束縛を捨ててクラスをオープンするのに対して、class_evalはフラットスコープを持っているので、 `class_eval do ~ end` の外にあるローカル変数も do ~ end のブロック内部から参照することができる。

クラスをオープンして内部にメソッドを定義したいなら `class_eval` , クラス以外のオブジェクトをオープンして内部のインスタンス変数などを変更したい場合は `instance_eval` を使うという使い分けで良さそう。

こんなものどこでつかうんですかねｗ

## 5.1.3 クラスインスタンス変数
カレントクラスの話が役に立つことを証明するためにクラスインスタンス変数の話をする。

```
class MyClass
  @var = 1
  
  def self.read
    @var
  end
  
  def write
    @var = 2
  end
  
  def read
    @var
  end
end

obj = MyClass.new
obj.read # => nil
obj.write
obj.read # => 2
MyClass.read # => 1
```

上のコードではクラス内の異なるスコープの場所に同一名のインスタンス変数(@var)が定義されている。

classキーワードのすぐ下で定義されている@varはselfがMyClassとなる場所で定義されているため、MyClassというオブジェクトに属するインスタンス変数として認識される。このインスタンス変数はクラスインスタンス変数と呼ばれ、クラスメソッドからしか参照できない。
`MyClass.instance_variables` とすると@varが出てくる

writeメソッド内で定義されている@varはselfがobjの場所で定義されているので、MyClassインスタンスのオブジェクト(この例だとobj)のインスタンス変数として認識される。このインスタンス変数はインスタンスメソッド or initializeメソッドからしか参照できない。
`MyClass.new.instance_variables` とすると何も出てこない。これは初期化段階ではこのインスタンスオブジェクトは内部に何もインスタンス変数を持たないから。


### clasキーワードを使わずにあるクラスを継承するクラスを作成する
下記のようにすれば、Arrayクラスを継承した新しいクラスを定義することができる。

```
c = Class.new(Array) do
  def my_method
    p 'hoge!'
  end
end
```

## 5.3.1 特異メソッドの導入
Rubyでは特定のオブジェクトにメソッドを追加することができる。(普段僕が使用しているのはクラスに定義されたクラスメソッドやインスタンスメソッドなのだ)
下記のようにdefキーワードに対して `OBJECT.method_name` とするとレシーバーのOBJECTに対して特異メソッド(シングルトンメソッド)を定義できる。
下記の例の場合、オブジェクトstrには `title?` メソッドが追加されているが、strのclassであるStringクラスには title? メソッドは追加されていない。つまりStringクラスは汚染されていない。

```
str = 'hello world!!'

# strが全部大文字ならtrueを返す
def str.title?
  self.upcase == self
end

str.title? # => false
str.singleton_methods # => [:title?] 
# Stringクラスは汚染されていない
String.methods.grep(/title?/) # => []
```

## 5.3.2 クラスメソッドの真実
1行目は変数で参照したオブジェクトのメソッドを呼び出している。
2行目は定数で参照したオブジェクト(クラス)のメソッドを呼び出している
異なる種類のメソッド呼び出しだが、やっていることは一緒らしい！

```
obj.a_method
String.a_method
```

ちなみに `String.a_method` はStringという定数で参照がついているオブジェクト(クラス)にあるクラスメソッドを呼び出している。他のオブジェクト(クラスやインスタンス変数)からはクラスメソッドは呼び出せない。ということは **クラスメソッドは、クラスの特異メソッド** といえる！

特異メソッドを定義する構文は常にこうなる。
objectの部分は、オブジェクトへの参照・クラス名の定数・selfのいずれかが使われる・

```
def object.method_name
  # do something
end
```

## 5.3.3 クラスマクロ
Rubyのオブジェクトにはアトリビュート(オブジェクト内部の構造(たぶんインスタンス変数)を参照するもののことだと思う)がないので、アトリビュートがほしい時は2つのミミックメソッドを定義する。

下記のようなメソッド(アクセサとも呼ぶ)は書くのがめんどくさいのでRubyには `Module#attr_*` 系のメソッドがある。

```
class MyClass
  # commandメソッド
  def my_attribute=(value)
    @my_attributes = value
  end
  
  # queryメソッド
  def my_attribute
    @my_attribute
  end
end
```

attr_* 系のメソッドはModuleクラスで定義されているのでselfがモジュールでも使える。
attr_* 系のメソッドをクラスマクロと呼ぶ。クラスマクロはキーワードのように見えるが、クラス定義の中で使えるクラスメソッドなのが正体。

## 5.4.1 特異クラス
特異メソッドはどこに定義されるのか。通常メソッドはクラスに定義されるためメソッド探索で右に上に探索していくとどこまで見つかる。しかし特異メソッドはある特定のオブジェクトに対して定義されるため、右に上に探索しても見つからない。これはクラスメソッドも同じだ。

## 5.4.2 特異クラスの出現
オブジェクトは裏に特別なクラスを持っていて、これは特異クラスと呼ばれる(メタクラスやシングルトンクラスとも呼ばれる)
特異メソッドはこの裏にある特異クラスに対して定義される。
普段使っているObject#classでは特異クラスを隠してしまって、その実態はつかめない。
`Object#singleton_class` を使うと特異クラスを参照することができ、かつ `.instance_methods` とすると特異メソッドが特異クラス内に住んでいることが確認できる

```
obj = Object.new

def obj.hello
  p 'hello'
end

# 特異メソッドはオブジェクトobjのクラスには住んでいない
obj.class.instance_methods.grep(/hello/) # => []

# 特異クラスを覗ける
obj.singleton_class # => #<Class:#<Object:0x00007f9c1a23cae0>>
# 特異クラスの中に特異メソッドが住んでいることが確認できた！
obj.singleton_class.instance_methods.grep(/hello/)
```

## 5.4.3 メソッド探索再び
特異クラスのスーパークラスは菜になるのか。答えはイニシャライズしたクラスになる。

```
class D
end

obj = D.new
obj.singleton_class.superclass # => D
```

特異メソッドを持っている場合も通常のメソッド探索と同じようにメソッド探索が行われる。違うのはレシーバーのオブジェクトのクラスを見に行く前に、レシーバーオブジェクトの特異クラスを見に行くこと。

```
      Object
        ↑
        C
        ↑
        D
        ↑
obj → #obj
```

## 5.4.4 特異クラスと継承

