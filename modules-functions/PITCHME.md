---
## Функционално програмиране с Elixir

---
![Image-Absolute](assets/bulgaria_elixir.png)

---

## Въпроси

* инсталирахте ли си Elixir?
    - препоръчвам инструмента asdf
* отворихте ли elixir-lang.{org, bg}?
* присъединихте ли се във {Facebook, Discord}?
    - не е задължително, но лесно ще си взимате домашното/информацията

---

## Полезни връзки и материали

* [Нашото блогче](https://elixir-lang.bg/posts/summary)
* [Elixir in Action](https://www.manning.com/books/elixir-in-action)
  - търсете втория едишън
* [The little elixir and otp guidebook](https://www.manning.com/books/the-little-elixir-and-otp-guidebook)


---
## Модули, функции и рекурсия

---

### Дефиниране на модул

```elixir
defmodule Times do
  def double(n) do
    n * 2
  end
end
```

---

### Компилиране

* като използваме `elixirc Times`
* ще генерира файл на име `Elixir.Times.beam`, който съдържа байткода
* `iex` автоматично ще зареди всякакви `*.beam` файлове, ако са в същата директория
* никога няма да го правим по този начин

---

### Забелязвате ли префикса `Elixir.` ?

---

### Компилиране и зареждане на файл от конзолата

```bash
iex(1)> c "times.ex"
[Times]
iex(2)> Times.double(10)
20
```

---

### .ex vs .exs

* .ex е за файлове, които искаме да се компилират.
* .exs е за файлове, които са скриптове и ще изпълняваме.

---

### elixirc vs elixir

* elixirc само компилира код до .beam
* elixir ще компилира в паметта и изпълни дадения файл, няма да произведе .beam файл
* компилират кода по един и същ начин

---

### Именовани функции

---

#### Публични фукнции дефинираме с def/2

* макрос е
* публичните функции могат да се извикват от други модули
* имената могат да са utf8
* [има конвенции за функции завършващи на `?`/`!`](https://hexdocs.pm/elixir/master/naming-conventions.html)

---

#### {Частни, поверителни, За собствено ползуване} фукнции дефинираме с defp/2

* пак е макрос
* могат да се извикват само вътре в даден модул
* компилаторът може да хване само private функции, които не ползваме

---

### Пример

---

```
defmodule FMI do
  def смятай(a, b) do
    смятай_бе(a, b)
  end

  defp смятай_бе(a, b) do
    a + b
  end
end

IO.puts(FMI.смятай(3, 4)) # => 7
```
---

### Функции с различен брой параметри

```bash
iex(1)> String.split("Elixir is awesome. It totally kicks bum.")
["Elixir", "is", "awesome.", "It", "totally", "kicks", "bum."]

iex(2)> String.split(
  "Elixir is awesome. It totally kicks bum.",
  ".",
  parts: 2)
["Elixir is awesome", " It totally kicks bum."]
```

---

### do..end блокове с малко код

```elixir
def double(n), do: n * 2
```

---

### Дефиниране на функции и pattern matching

```elixir
defmodule Factorial do
  def of(0), do: 1
  def of(n), do: n * of(n - 1)
end
```

---

### Реда на дефиниране на функциите има значение

```elixir
defmodule Factorial do
  def of(n), do: n * of(n - 1)
  def of(0), do: 1
end
```

---

### Guard клаузи при дефиниране на функции

```elixir
defmodule Fibonachi do
  def of(1), do: 1
  def of(2), do: 1
  def of(n) when is_number(n), do: of(n - 1) + of(n - 2)
end
```

---

### Параметри със стойности по подразбиране

* дефинират се с `\\`
* преди Борко да каже "Ебаси гадния език!"
* = се използва за патърн мачинг, за това е нужен друг синтаксис

```elixir
defmodule Example do
  def func(p1, p2 \\ 2, p3 \\ 3, p4) do
    IO.inspect [p1, p2, p3, p4]
  end
end

Example.func("a", "b") # => ["a", 2, 3, "b"]
Example.func("a", "b", "c") # => ["a", "b", 3, "c"]
Example.func("a", "b", "c", "d") # => ["a", "b", "c", "d"]
```

---

### Pipe оператора - `|>`

---

#### Нека разгледаме един пример без този оператор

---

```elixir
iex(30)> Enum.join(
  String.split(
    String.upcase(
      "name,sex,location"),
    ","),
  " "
)
"NAME SEX LOCATION"
```

---

### Pipe оператора - `|>`

```elixir
iex(33)> "name,sex,location"
  |> String.upcase
  |> String.split(",")
  |> Enum.join(" ")
"NAME SEX LOCATION"
```

---

### Връзки между модули и влагане

```elixir
defmodule Outer do
  defmodule Inner do
    def inner_func do
      "hello world"
    end
  end

  def outer_func do
    "Greeting from the inner func: #{Outer.Inner.inner_func}"
  end
end

Outer.outer_func # => "Greeting from the inner func: hello world"
Outer.Inner.inner_func # => "hello world"
```

---

### Няма нищо специално за вложените модули

* само синтактична захар
* крайните модули ще са `:"Elixir.Outer"` и `:"Elixir.Outer.Inner"`
* пак трябва да импортираме функциите от бащата модул във вложения модул
* context-ът е същия, което означава, че всякакви импорти и преименувания ще се запазят
* [ето ви документация](https://hexdocs.pm/elixir/Kernel.html#defmodule/2)
* ето ви пример за неща, които **НЕ МОЖЕМ ДА ПРАВИМ**:

---

```elixir
defmodule Deep.Outer do
  def outer, do: :outer

  defmodule Inner do
    # Трябва Deep.Outer.outer/1 да го реферираме директно,
    # няма да се компилира
    def inner_outer, do: Outer.outer()

    # Също няма да се компилира
    def inner_outer2, do: outer()
  end
end
```

---

##### Между другото нищо не ни пречи директно да направим това:

```elixir
defmodule Deep.Outer.Inner do
  def inner, do: :inner
end
```

---

### import

* Позволяват ни да викаме всички функции на чужд модул без да ги префиксваме с името на модула.
* с други думи не е нужно да използваме пълното име на функцията
* only
* except
* `:macros/:functions`
* може да импортираме в тялото на функция

---
```elixir
defmodule CsvUtils do
  import String

  def upcase_space_transform(csv_line) do
    csv_line
    |> upcase
    |> split(",")
    |> Enum.map(&strip/1)
    |> Enum.join(" ")
  end
end
```

---

### alias

* Преименува даден модул в друг модул, за да използваме по-кратко име.
* `__MODULE__`
* ерлангски модули
* {синтаксис}

---

```elixir
defmodule Outer.Inner do
  def inner_func do
    "hello world"
  end
end

defmodule Outer do
  alias Outer.Inner, as: Inner

  def outer_func do
    "Greeting from the inner func: #{Inner.inner_func}"
  end
end
```

---

### require

* използва се когато искаме да използваме макроси на даден модул
* compile time ще се "разшири" макроса
* повече за require/use когато стигнем до мета-програмирането
* примерче:

---

```elixir
iex(1)> Integer.is_odd(3)
** (CompileError) iex:1: you must require Integer before invoking the macro Integer.is_odd/1
    (elixir) src/elixir_dispatch.erl:97: :elixir_dispatch.dispatch_require/6
iex(1)> require Integer
Integer
iex(2)> Integer.is_odd(3)
true
iex(3)>
```

---

### Модулни атрибути

* дефинират се с `@`
* мислете ги за константи
* имат и други приложения, повече за тях по време на мета-програмирането

---

```elixir
defmodule Greeter do
  @standard_greeting "Hello, stranger!"

  def greet(nil) do
    IO.puts @standard_greeting
  end

  def greet(name) do
    IO.puts "Hello, #{name}!"
  end
end
```

---

### Използване на Erlang модули

```elixir
:rand.uniform(100) # => 98
:rand.uniform(100) # => 41
```

---

### Анонимни функции

* всъщност може да ви покажем само анонимни функции и това е достатъчно за целия курс
* един лайк във фб, ако разбрахте шегата
* за разлика от Erlang, анонимните функции в Elixir не са fun.
* дефинират се с `fn -> end`

---

#### Пример

```
iex(7)> fn a, b -> a + b end.(3, 4)
7
```

---

### Забелязахте ли как я извикахме?

---

### С точка!

* Преди Борката да изкрещи "BAD LANGUAGE DESIGN!"
* Нещо си нещо си lisp-2 език(Note to author: може и да го обясниш)

---


#### Могат да имат няколко глави като нормалните функции

```elixir
iex(8)> add = fn
...(8)>   a, b when is_integer(a) and is_integer(b) -> a + b
...(8)>   a, b when is_list(a) and is_list(b) -> a ++ b
...(8)>   a, b when is_binary(a) and is_binary(b) -> <<a::binary, b::binary>>
...(8)> end
iex(9)> res = add.(<<0::size(16)>>, <<1::size(16)>>)
<<0, 0, 0, 1>>
iex(10)> Kernel.bit_size(res)
32
```

---

### Залавяне на функции

* Function capturing
* става с така наречения "capture operator"
* &пълно-име-на-функция/арност
* може да прихващаме импортирани функции
* може да създаваме функции
* не може да създава функции с арност 0
* може да работи с друи три оператора: `[]`, `{}` и `%{}`

---

#### Пример

```elixir
iex(5)> fun = &is_integer/1
&:erlang.is_integer/1
iex(6)> fun.(0)
true
```

---

#### Oще пример

```elixir
iex(19)> succ = & &1 + 1
#Function<6.127694169/1 in :erl_eval.expr/5>
iex(20)> succ.(1)
2
```
---

#### Oще пример с оператор

```elixir
iex(13)> doubler = &{&1, &1}
#Function<6.127694169/1 in :erl_eval.expr/5>
iex(14)> doubler.(1)
{1, 1}
```

---

## Mix

* билд инструмент идващ с Elixir
* позволявани да #{insert} нашия проект
    1. създаваме
    2. изпълняваме
    3. компилираме
    4. тестваме
    5. менажира "зависимостите ни"
    6. всъщност може да го накараме да прави какво си искаме с нашия проект
* абе влиза в кода и си прави каквото си иска

---

### Малък cheat sheet с неща, които може да прави

* mix compile
* mix test
* iex -S mix
* mix format
* mix help
  - потърси помощ

---

#### Note to author: Git Gud

* leGIT presentation
* the git that keeps on giving
* Beware the HEADless horseman of gitty hollow

---

### Да поразцъкаме малко mixture-и :-)

1. Stargazers
2. Undocumented

---

### Въпроси?
