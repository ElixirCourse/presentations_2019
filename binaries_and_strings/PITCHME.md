---
## Низове и binaries
![Image-Absolute](assets/title.jpg)

---
## Съдържание

1. Как да да работим с двоичните структури - binaries
2. Как са представени двоичните структури
3. Низове
4. Обхождане на низове

---
## Binaries
![Image-Absolute](assets/binary_code.jpg)

---
### Конструкция
```elixir
<<>>

<< 123, 23, 1 >>

<< 0b01111011, 0b00010111, 0b00000001 >> == << 123, 23, 1 >>
#=> true

<< 280 >>
#=> true <<24>>
```

---
```elixir
<< 280::size(16) >> # Същото като << 280::16 >>
#=> << 1, 24 >>

<< 0b00000001, 0b00011000 >> == << 280::16 >>
#=> true

<< 129::7 >>
#=> <<1::size(7)>> Или 7 бита от 10000001 : 0000001
```

---
* Интересно е съхранението на числа с плаваща запетая като `binaries`.
* Винаги се представят като `binary` от 8 байта (double precision):

```elixir
<< 5.5::float >>
#=> <<64, 22, 0, 0, 0, 0, 0, 0>>
```

---
### Конкатенация
![Image-Absolute](assets/concatenate.jpg)

---
```elixir
<< 83, 79, 83 >> <> << 24, 4 >>
#=> <<83, 79, 83, 24, 4>>

<< 83, 79, 83, << 24, 4 >> >>
#=> <<83, 79, 83, 24, 4>>
```

---
### Размер

```elixir
Kernel.byte_size(<< 83, 79, 83 >>)
#=> 3

byte_size(<< 34::5, 23::2, 12::2 >>)
#=> 2
```

---
### Размер

```elixir
Kernel.bit_size(<< 34::5, 23::2, 12::2 >>)
#=> 9
```

---
### Проверки

1. `Kernel.is_bitstring/1` - Винаги е истина за каквато и да е валидна поредица от данни между `<<` и `>>`. Няма значение дължината върната от `Kernel.bit_size/1`.
2. `Kernel.is_binary/1` - Истина е само ако `Kernel.bit_size/1` връща число кратно на `8` - тоест структурата е поредица от байтове.

---
### Sub-binaries
![Image-Absolute](assets/sub_binary.jpg)

---
С функцията `Kernel.binary_part/3`, можем да боравим с части от дадена `binary` структура:

```elixir
binary_part(<< 83, 79, 83, 23, 21, 12 >>, 1, 3)
#=> <<79, 83, 23>>
```

---
Функцията `Kernel.binary_part/3` вътрешно ползва `:erlang.binary_part/3`, която е *NIF*, позволен в guard-ове:

```elixir
case <<17, 13, 14, 15>> do
  v when binary_part(v, 2, -2) == <<17, 13>> -> :ok
  _ -> :nope
end
#=> :ok
```

---
Още една функция от Erlang : `:erlang.split_binary/2`, отново ползва `:erlang.binary_part/3`, но не може да е част от guard:

```elixir
:erlang.split_binary(<<4, 65, 129>>, 2)
#=> {<<4, 65>>, <<129>>}
```

---
## Pattern matching

Като всичко друго в Elixir и с `binaries` можем да съпоставяме:

```elixir
<< x, y, x >> = << 83, 79, 83 >>
#=> << 83, 79, 83 >>

x
#=> 83

y
#=> 79
```

---
```elixir
<< x, y, z::binary >> = << 83, 79, 83, 43, 156 >>
#=> <<83, 79, 83, 43, 156>>

z
#=> << 83, 43, 156 >>

<< x, y, z >> = << 83, 79, 83, 43, 156 >>
#=> ** (MatchError)

<< x, y, z::bitstring >> = << 83, 79, 83, 43, 156 >>
#=> <<83, 79, 83, 43, 156>>
```

---
```elixir
<<
  sign::size(1),
  exponent::size(11),
  mantissa::size(52)
>> = << 4815.162342::float >>
#=> <<64, 178, 207, 41, 143, 62, 204, 196>>

#=> sign
0
```

---
```elixir
:math.pow(-1, sign) *
(1 + mantissa / :math.pow(2, 52)) *
:math.pow(2, exponent - 1023)

4815.162342
```

---
Модификатори:
* `float`
* `binary` или `bytes`
* `integer` модификатор, който се прилага по подразбиране
* `bits` или `bitstrig`
* `utf8`
* `utf16`
* `utf32`

---
```elixir
<< x::5, y::bits >> = << 225 >>
#=> <<225>> или 11100001
x
#=> 28 или 11100
y
#=> <<1::size(3)>> или 001
```

---
* `size` задава дължина в битове, когато работим с `integer`.
* Aко работим с `binary` модификатор, `size` е в байтове.

---
```elixir
<< x::binary-size(4), _::binary >> =
  << 83, 222, 0, 345, 143, 87 >>
#=> <<83, 222, 0, 89, 143, 87>>

x # 4 байта
#=> <<83, 222, 0, 89>>
```

---
## Binaries - имплементация
![Image-Absolute](assets/implementation.jpg)

---
* Всеки процес в Elixir има собствен heap.
* За всеки процес различни структури от данни и стойности са съхранени в този heap |
* Когато два процеса си комуникират, съобщенията се копират между heap-овете им. |

---
### Heap Binary
* Ако `binary`-то е `64` **байта** или по-малко, то е съхранявано в `heap`-a на процеса си.
* Такива `binary` структурки наричаме **heap binaries**.

---
### Refc Binary
* Когато структурата ни е по-голяма от `64` **байта** се пази в обща памет за всички process-и на даден `node`.
* В *porcess heap*-a се пази малко обектче - **ProcBin**.
* `binary` структура може да бъде сочена от множество **ProcBin** указатели от множество процеси.

---
* Пази се _reference counter_ за всеки от тези указатели.
* Когато той стане `0`, Garbage Collector-ът ще може да изчисти `binary`-то от общия `heap`.
* Такива `binary` структури наричаме **refc binaries**.

---
### Sub binaries
* Указател към съществуващо `binary` пазещ по-малка от неговата дължина.
* Няма копиране.
* Броячът на референции се увеличава.
* Функцията `binary_part` да речем създава такива.

---
### Matching context
* Подобен на `sub binary`, но оптимизиран за `binary pattern matching`.
* Държи указател към двоичните данни в паметта.
* Когато нещо е `match`-нато, указателят се придвижва напред.

---
### Matching context
* Компилаторът избягва създаването на `sub binary` за всяка `match`-ната променлива.
* Ако е възможно се преизползва един и същ `match context`.

---
```elixir
x = << 83, 222, 0, 89 >>
y = << x, 225, 21 >>
z = << y, 125, 156 >>
a = << y, 15, 16, 19 >>
```

---
## Низове
![Image-Absolute](assets/strings.jpg)

---
Низовете в Elixir използват `unicode` - предтавляват `binary` структури -
поредица от `unicode codepoint`-и в `utf8` енкодинг.

---
* При `UTF-8` даден `codepoint` може да се съхранява в от един до четири байта (може и в повече).
* Първите `128` кодпойнта съвпадат с `ASCII` `codepoint`-ите.
* Te се пазят в един байт, който започва с `0`:

```
0XXXXXXX
```

---
```elixir
<< 0b00110000 >>
#=> "0"
<< 0b00110001 >>
#=> "1"
<< 0b00110010 >>
#=> "2"
<< 0b00110101 >>
#=> "5"
<< 0b00111001 >>
#=> "9"
```

---
* След като `128`-възможности за `1` байт се изчерпат, започват да се ползват по `2` байта.
* В този шаблон за `2`-байтови `codepoint`-и, виждаме че първият байт винаги започва с `110`.
* Двете единици значат - `2` байта, байт започващ с `10`, означава че е част от поредица от байтове.

```
110XXXXX 10XXXXXX
```

---

```elixir
?Ъ
#=> 1066

Integer.to_string(1066, 2)
#=> "10000101010"
```

---
```
Префиксваме с 110 и допълваме до байт с битовете от числото
(110)10000

Префиксваме с 10 и допълваме до байт с битовете, които останаха
(10)101010
```

```elixir
<< 0b11010000, 0b10101010 >>
#=> "Ъ"
```

---
Шаблонът за `3`-байтови `codepoint`-и е:

```
1110XXXX 10XXXXXX 10XXXXXX
```

---
А шаблонът за `4`-байтовите:

```
11110XXX 10XXXXXX 10XXXXXX 10XXXXXX
```

---
Можем да заключим че има три типа байтове:
1. Единичен - започва с `0` и представлява първите `128` ASCII-compatible `codepoint`-a.
2. Байт-продължение - започва с `10`, следва други байтове в `codepoint` от повече от `1` байт.
3. Байт-начало - започва с `110`, `1110`, `11110`, първи байт от серия байтове представляващи `codepoint`.

---
### Графеми и codepoint-и
![Image-Absolute](assets/graphemes.jpg)

---
* Символите или графемите не винаги са точно един `codepoint`.
* Има графеми, които могат да се представят и като един `codepoint` и като няколко.

---
```elixir
name = "Николай"
#=> "Николай"
String.codepoints(name)
#=> ["Н", "и", "к", "о", "л", "а", "й"]
String.graphemes(name)
#=> ["Н", "и", "к", "о", "л", "а", "й"]
String.graphemes(name) == String.codepoints(name)
#=> true
```

---
```elixir
name = "Николаи\u0306"
#=> "Николай"
String.codepoints(name)
#=> ["Н", "и", "к", "о", "л", "а", "и", ̆"<не може да се представи>"]
String.graphemes(name)
#=> ["Н", "и", "к", "о", "л", "а", "й"]
```

---
* Функции като `String.reverse/1`, работят правилно, защото ползват `String.graphemes/1`.

```elixir
String.reverse("Николаи\u0306")
#=> "йалокиН"

# Аналогично:
String.graphemes(name2) |> Enum.reverse |> Enum.join("")
#=> "йалокиН"
```

---
### Дължина на низ
```elixir
Kernel.bit_size "Николаи\u0306"
#=> 128
Kernel.byte_size "Николаи\u0306"
#=> 16
String.length "Николаи\u0306"
#=> 7
```

---
### Под-низове
```elixir
String.at("Искам бира!", 6)
#=> "б"
String.at("Искам бира!", -1)
#=> "!"
String.at("Искам бира!", 12)
#=> nil
```

---
```elixir
<< _::binary-size(11), x::utf8, _::binary >> = "Искам бира!"
#=> "Искам бира!"

x
#=> 1073
<< x::utf8 >>
#=> "б"
```

---
```elixir
<< "Искам ", x::utf8, _::binary >> = "Искам бира!"
#=> "Искам бира!"
```

---
```elixir
String.next_codepoint("Николаи\u0306")
#=> {"Н", "иколай"}
String.next_codepoint("и\u0306")
#=> {"и", "<un-printable>"}
String.next_grapheme("и\u0306")
#=> {"й", ""}
String.next_grapheme_size("и\u0306")
#=> {4, ""}
String.next_grapheme_size("\u0306")
#=> {2, ""}
```

---
```elixir
poem = """
  Ще строим завод,
  огромен завод,
  със яки
        бетонни стени!
  Мъже и жени,
  народ,
  ще строим завод
  за живота!
"""

large_factory = String.slice(poem, 18..30)
#=> "огромен завод"
String.slice(poem, 18, 13)
#=> "огромен завод"
```

---
## Обхождане на низове
![Image-Absolute](assets/iterate_path.jpg)

---
```elixir
defmodule ACounter do
  def count_it_with_slice(str) do
    count_with_slice(str, 0)
  end

  defp count_with_slice("", n), do: n
  defp count_with_slice(str, n) do
    the_next_n = next_n(String.starts_with?(str, "a"), n)
    count_with_slice(String.slice(str, 1..-1), the_next_n)
  end

  defp next_n(true, n), do: n + 1
  defp next_n(false, n), do: n
end
```

---
* Това е около `3.18` секунди за низ с дължина над 2_000_000

```elixir
:timer.tc(ACounter, :count_it_with_slice, [str])
#=> {3176592, 589251}
```

---
```elixir
defmodule ACounter do
  def count_it_with_next_grapheme(str) do
    count_with_next_grapheme(str, 0)
  end

  defp count_with_next_grapheme("", n), do: n
  defp count_with_next_grapheme(str, n) do
    {next_grapheme, rest} = String.next_grapheme(str)
    count_with_next_grapheme(
      rest,
      next_n(next_grapheme == "a", n)
    )
  end
end
```

---
* Добре същият отговор, но сега за `0.8` секунди

```elixir
:timer.tc(ACounter, :count_it_with_next_grapheme, [str])
#=> {792885, 589251}
```

---
```elixir
defmodule ACounter do
  def count_it_with_match(str) do
    count_with_match(str, 0)
  end

  defp count_with_match("", n), do: n

  defp count_with_match(<< c::utf8, rest::binary >>, n)
  when c == ?a do
    count_with_match(rest, n + 1)
  end

  defp count_with_match(<< _::utf8, rest::binary >>, n) do
    count_with_match(rest, n)
  end
end
```

---
* Тази имплементация със същия низ е около *0.1* секунди.

---
## Край
![Image-Absolute](assets/end.jpg)
