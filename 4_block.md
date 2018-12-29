# 4章 水曜日：ブロック

## 4.1.2 ブロックの基本
メソッド内部では、 `Kernel#block_given?` メソッド使ってブロックの有無を確認できる
block_given? がfalseの時に yieldを使うとエラー( `LocalJumpError (no block given (yield))` )になる

```rb
def method
  return yield if block_given?
  'ブロックがありません'
end

method                    # => 'ブロックがありません'
method { 'ブロックがある' } # => 'ブロックがある'
```

## 4.3 ブロックはクロージャ
ブロックのコードは単体では実行できない。実行するにはローカル変数、インスタンス変数、selfといった環境が必要になる。それらはオブジェクトに紐付けられた名前のことで束縛とも呼ばれる。
ブロックとは。これらをまとめてから、じっこうの準備をするもの。コードと束縛の集まり両方が含まれる。

```rb
# コード
x = a + 1
x += @6

# 束縛
a # => 2
@6 # => 4

# 実行するコード
x = a + 1 # => 3
x += @6   # => 7x
```

ブロックを定義するとその時点でその場所にある束縛を取得し、実行するコードにその束縛を反映するっぽい。
ブロックの中で新しく束縛を定義することもできるが、ブロックから抜けるとその束縛は消える。
このような特性があるからブロックはクロージャと呼ばれる

## 4.3.2 スコープゲート
プログラムがスコープを切り替えて、新しくスコープをオープンする場所は三箇所ある
- クラス定義(class MyClass ... end)
- モジュール定義(module MyModule ... end)
- メソッド(def ... end)

これらはスコープゲートとも呼ばれる

## 4.3.3 スコープのフラット化
下記のようにスコープゲートを通り抜けて束縛を渡したい時はどうするか

```rb
my_var = '成功'

class MyClass
  # my_var を表示したい
  
  def my_method
    # my_var を表示したい
  end
end
```

### classのスコープゲートを超える
classのスコープゲートにmy_varを渡すことはできないが、classをスコープゲートではない何かに置き換えることはできる。ここではメソッドを呼び出しを使ってみる
classキーワードの代わりにメソッドをよびだせばmy_varをクロージャに包んでクロージャをメソッドに渡せる。
ここではclassキーワードと同じ効果のある `Class.new` を使う。

```rb
my_var = '成功'

MyClass = Class.new do
  puts "クラス定義の中は#{my_var}"
  
  def my_method
    # ここで表示するにはどうする
  end
end
```

### methodのスコープゲートを超える
classの時と同じように、defキーワードではない手法、つまりメソッド呼び出しでメソッドを定義すればいい。
ここでは `Module#define_method` を使う

```rb
my_var = '成功'

MyClass = Class.new do
  puts "クラス定義の中は#{my_var}"
  
  define_method :my_method do
    puts "メソッド定義の中も#{my_var}"
  end
end

puts MyClass.new.my_method 
# => クラス定義の中は成功, メソッド定義の中も成功
```


スコープゲートをメソッド呼び出しに置き換えると、他のスコープの変数が見えるようになる。技術的には入れ子構造のレキシカルスコープと呼ばれる。rubistからはフラットスコープと呼ばれる

## 4.3.4 クロージャーのまとめ
下記のように置き換えることでフラットスコープを行うことができる

```rb
class -> Class.new
module -> Module.new
def -> Module.define_method
```


ブロック = クロージャーで、クロージャはある時点での値や状態を束縛して包むことができる

## 4.4 instance_eval
`BasicObject#instalce_eval` では受け取るブロック内でレシーバーをselfにするのでメソッドのレシーバーであるオブジェクト内のインスタンス変数やプライベートメソッドもcallすることが可能になる。また受け取った

```rb
class MyClass
  def initialize(number)
    @n = number
  end
end

obj = MyClass.new(2)

obj.instance_eval do
  self # => #<MyClass .....>
  @n # => 2
end

# フラットスコープ扱いなので、obj.instance_eval 時にローカル変数numberの値を参照することが可能
number = 10
obj.instance_eval { @n = number }
obj.instance_eval { @n } # => 10
```


instance_evalに渡したブロックのことをコンテキスト探査機と呼ぶらしい。(あっそｗ)

## 4.4.1 カプセル化の破壊
コンテキスト探査機を使うとカプセル化を破壊できてしまうので良くない。しかし時にカプセル化が邪魔になるテストの時などには使うこともある。が使わなくていいなら使わない方が良さそう。

## 4.4.2 クリーンルーム
ブロックを評価するためだけに生成されるオブジェクトのことをクリーンルームと呼ぶ。
クリーンルームには不要なメソッドやインスタンス変数、定数がないほうが好ましく、そういう観点でいうとBasicObjectが一番クリーンらしい

何に使うのかよくわからんｗ

## 4.5 呼び出し可能オブジェクト
ブロックはコードを保管してyieldを使って実行するためのもの。似たようなものに、Proc, lambda, メソッドの中身がある

## 4.5.1 Procオブジェクト
Procは元々オブジェクトではないブロックを保管して、あとで実行するためにあるっぽい。
Proc.newでオブジェクトになったブロックは、 `Proc#call` で呼び出せる。

```
proc = Proc.new { |x| x + 1 }
proc.call(1) # => 2
```

lambdaでも同じようなことができる

### &修飾
- メソッドから他のメソッドにブロックを渡したいとき
- ブロックをProcに変換したいとき

ブロックを受け取りたいメソッドの引数には `&` をprefixとしてつけておくと、メソッドに渡されたブロックをProcに変換するという意味になる

```
def math(a, b)
  yield(a, b)
end

def do_math(a, b, &operation)
  math(a, b, &operation)
end

# mathメソッドに対して 2, 3, ブロックを渡して、mathメソッド内のyieldがブロックを評価する
do_math(2, 3) { |x, y| x * y } # => 6
```

`do_math` はブロックなしでcallすると、 `&operation` がnilになり、mathメソッド内のyieldで死ぬ

ブロックにprefix `&` をつけない場合は、そのブロックはProcのままでブロックには戻らない

```
def my_method(&proc)
  # procのまま
  proc
  
  # 仮にこうするとprocが受け取った引数procをブロックに変換する
  # &proc
end

p my_method { |name| "hello, #{name}!!" }
# my_method内で &proc としてないので、classもProcになっている
p.class        # => Proc　
p.call('Bill') # => hello, Bill!
```

## 4.5.2 proc 対 lambda
両者には2つの大きな違いがある。returnキーワードに関すること、そして引数のチェックだ。

### return
lambdaの場合単にlambdaから戻るだけだが、Procの場合はProcが定義されたスコープから戻ろうとする。

~~なんかよくわからない~~
Procの場合はreturn後にメソッド自体から抜ける。lambdaの場合はreturnした後メソッドに戻りメソッドが最後まで実行される。

```
# proc
def proc_method
  proc = Proc.new { return p "proc"}
  # procではreturnが実行されるとそれ以後の処理から抜けてしまう。
  proc.call
  p "proc method"
end

proc_method #=> "proc"

# lambda
def lambda_method
  lambda1 = lambda{ return p "lambda"}
  # lambdaでは return は p "lambda" をするだけ
  lambda1.call
  p "lambda method"
end

lambda_method #=> "lambda"
              #=> "lambda method"
```

## 引数チェック
Procでは指定されている引数の数より多かったり少なかったりするとよしなに調節してエラーにならないようにしてくれる。
lambdaではArgumentErrorが吐かれる

```
p = Proc.new { |a, b| [a, b] }
# 多い分は切り落とされる
p.call(1, 2, 3) # => [1, 2] 
# 少ない分はnilが当てられる。これよくない希ガス
p.call(1) # => [1, nil]
```

### Proc vs lambdaまとめ
lambdaの方がメソッドの挙動に似ているため、Procでないといけない場合以外はlambdaを使うのが良さそう

## 4.5.3 Methodオブジェクト
メソッドも呼び出し可能なオブジェクト。lambdaのように `.call` を使って呼び出しができる

```
class MyClass
  def initialize(value)
    @x = value
  end
  
  def my_method
    @x
  end  
end

object = MyClass.new(1)
m = object.method :my_method
m.class # => Method
m.call #=> 1 呼び出せた！！
```

`Object#method` を呼び出すとメソッドそのものをMethodオブジェクトとして取得でき、 `Method#call` で実行ができる。
Ruby2.1には `Kernel#singleton_method` があって、特異メソッド(singleton_method)の名前からMethodオブジェクトに変換できる。

Methodオブジェクトは `Method#to_proc` でProcに変換できるし、ブロックは `define_method` でメソッドに変換できる。
lambdaとMethodは少し違っていて、lambdaはそのlambdaが定義されたスコープで評価されるけど、Methodは所属するオブジェクトのスコープで評価される。

### 4.5.4 呼び出し可能オブジェクトのまとめ
- ブロック(※オブジェクトではない)
- Proc
- lambda
- メソッド
  - オブジェクトに束縛され、そのオブジェクトのスコープで評価される。他のオブジェクトに束縛を映すこともできる

メソッドとlambdaでは、returnで呼び出し可能オブジェクトから戻る。ブロックとProcでは呼び出し可能オブジェクトの元のコンテキストから戻る。また項数(引数)に対する挙動も異なる。メソッドはかなり厳密で、定義したとおりに渡さないといけない。lambdaもほぼメソッドと同じ。Procとブロックはゆるい。
