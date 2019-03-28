---

## Тип спецификации и поведения

---

![Image-Absolute](assets/type-specification-title.jpg)

---
## Съдържание

1. Защо erlang не е статично типизиран?
2. Тип спецификации и dialyzer
3. Поведения

---
## Защо erlang не е статично типизиран?
Вие как мислите?

---

1. Динамичното типизиране помага с hot code reloading

---

Или имаш статично типизиране и интерфейсите на модулите не се променят при hot code reload.
Или функциите не проверяват с какви данни са извикани, понеже
компилаторът го е проверил, което е много по-"unsafe", за разлика от
другите опции.
Дистрибутиран erlang прави това още по-трудно.


---

2. Динамичното типизиране помага с изпращане на всякакви съобщения.

---

Опит за статична система за изпращане на съобщения в езика [Links](http://links-lang.org).
Erlang-ският receive не може да бъде направен в Links, понеже всеки
receive ще трябва пълно да изразходва всички pattern-и за съобщения,
за да минава type checker-a.

---

3. Изпращането на съобщения допълнително ще затрудни типовата система.

---

Всяка функция ще трябва да има допълнително информация за какви
съобщения приема.
Тоест вместо `A -> B`, нещо като `A ~ C -> B`, където `C` са видовете
съобщения, на които функцията "може да приеме".

---

4. Друг проблем би бил "смъртта", или терминирането на процес.

---

Наблюдаването на процес би изисквало да може да приема точно
`{:EXIT, ...}` съобщение.
С други думи, ако всеки процес има типове `(m, d)`, където
`m` са съобщения, които приема и `d` причини за смъртта на
един процес, тогава само процес `(a, _)` би наблюдавал процес
`(_, a)`, което е доста лимитиращо.

---

#### Story time

---

Имало е много опити през годинитe за типизиране на Erlang.
Един от тях е бил през 1997 воден от Simon Marlow(lead GHC dev) и
Philip Wadler(haskell design, заслуги за теорията зад монадите).

---

##### Резултатът бил този [научен труд](http://homepages.inf.ed.ac.uk/wadler/papers/erlang/erlang.pdf)

---

##### Кратък коментар от Joe Armstrong по този труд

---

@quote[
One day Phil phoned me up and announced that
a) Erlang needed a type system,
b) he had written a small prototype of a type system and
c) he had a one year’s sabbatical and was going to write a type system for Erlang and “were we interested?”
Answer —“Yes.”
]

---


@quote[Phil Wadler and Simon Marlow worked on a type system for over a year and the results were published in 20. The results of the project were somewhat disappointing. To start with, only a subset of the language was type-checkable, the major omission being the lack of process types and of type checking inter-process messages.]

---

След като процесите и съобщенията са едни от най-важните неща за Ерланг,
става ясно защо тази система не е добавена.

---

Други опити за типизиране на Erlang също се провалили.

---

### Трябва ли да изоставим всякаква надежда?

---
## Dialyzer


---

С трудът зад *HiPE* се заражда инструментът наречен *dialyzer*, който се
използва и до ден днешен. Той си идва със собствен type inference механизъм,
наречен *success typing*.

---
### Как работи?

---
Построява таблици, които съдържат информация с инферирани
типове на абсолютно всички функции в системата.

---

Още наричани PLT - persistent lookup table

---?image=assets/dialyzer.png&size=auto 70%

---

- Добрата страна на PLT-то е, че се билдва инкрементално.
- PS: Как ще се реши този проблем?

---

### "Success Typing 101"

---

Success typing:
- няма да се опита да инферира точния тип на всеки израз
- ще гарантира, че типовете, които инферира са правилни
- ще гарантира, че грешките, които намира са наистина такива

---

#### Класическият пример с `and`
- ghci vs dialyzer demo
- Hindley-Milner type inference vs success typing

---

Dyalyzer Demo

---

Може би не звучи много смислено, но повече зад имплементацията и
решенията може да намерите ето [тук.](http://www.it.uu.se/research/group/hipe/papers/succ_types.pdf)

---

## Type specifications, or typespecs for short.

---

Вече видяхме различни типови спецификации.

---

- Ето [тук](https://hexdocs.pm/elixir/typespecs.html) има пълен списък.
- Be chill, че няма типови класове с контруктори и други шитове.
- (Поне вече има рекурсивни типове)

---

- Даже видях хора, които ги ползват на домашните!
- :bow:

---

Какви други бонуси ни дават типовите спецификации?
  * да допринасят за документация(инструменти като [ExDoc](https://github.com/elixir-lang/ex_doc) ги ползват)
  * когато пишем библиотеки за други хора
  * текстовите редактори

---

## Behaviours

---?image=assets/behaviour.jpg&size=auto 90%

---

##### Начин да разделим абстракцията от имплементацията.

---

Wait what?


---

###### [oще документация](https://hexdocs.pm/elixir/typespecs.html#behaviours)

---

Поведенията ни дават начин да правим:
  * module dispatch
  * осигурява, че даден модул имплементира разни функции
  * намираме абстракции в кода
  * помага за тестове

---

Ако се налага -  мислите за тях като интерфейсите в **!нормалните** езици.

---

#### ok, нави ме, как го правим?

---

- Да кажем, че бихме искали да напишем поведение за парсване на JSON/MsgPack
- (Това беше проект миналата година(сълза, за хората, които трябваше да го направят))

---

Как би изглеждало това?

---

```elixir
defmodule Parser do
  @callback encode(term) :: {:ok, String.t} | {:error, String.t}
  @callback decode(String.t) :: {:ok, term} | {:error, String.t}
end
```

---

И след това го вмъкваме в нашите имплементации по този начин:

```
defmodule JSON do
  @behaviour Parser

  def encode(term), do: # ... encode
  def decode(str), do: # ... decode
end

defmodule MsgPack do
  @behaviour Parser

  def encode(term), do: # ... encode
  def decode(str), do: # ... decode
end
```

---

### Dynamic dispatch

---

Можем динамично да решаваме кога да извикаме даден модул.

---

Нищо не ни пречи да направим следното:

---

```elixir
defmodule Parser do
  # ... rest of code ...

  def encode!(implementation, term) do
    implementation.encode(term)
  end
end

Parser.encode!(JSON, term)
#              ^ - динамично
```

---

Вече можем runtime да избираме какъв формат ни трябва!

---

... чакай малко ...

---

### Behaviours vs Protocols

---?image=assets/fight.jpg&size=auto 90%

---

Двете не правят ли близки неща?

---

Разликите:
  * протоколите ни дават полиморфизъм над типове/дата
  * поведенията ни дават динамично да се включваме където ни трябва

---

Един вид - протоколите са поведения + логика за dynamic dispatch

---

@quote[Protocol is type/data based polymorphism. When I call `Enum.each(foo, ...)`, the concrete enumeration is determined from the type of foo.

Behaviour is a typeless plug-in mechanism. When I call `GenServer.start(MyModule)`, I explicitly pass `MyModule` as a plug-in, and the generic code from GenServer will call into this module when needed.](Sas̄a Jurić)

---

Още от създателя на езика [по темата](https://groups.google.com/d/msg/elixir-lang-talk/S0NlOoc4ThM/tfqdDUSPvvsJ)

---

`@impl true && defoverridable`

---

More DEMO!

---

Въпроси и обратна връзка: *feedback@armenskiq.pop*

---?image=assets/beers.jpg&size=auto 90%
