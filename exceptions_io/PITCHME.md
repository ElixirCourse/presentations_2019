---
## Грешки
## Вход-изход

---
## Грешки
![Image-Absolute](assets/exception.jpg)

---
Част от общата концепция на Elixir и Erlang е грешките да бъдат **фатални** и да **убиват процеса**,
в който са възникнали.

Нека някой друг процес (supervisor) се оправя с проблема.

---
![Image-Absolute](assets/fatality.jpg)

---
Ако работим с файлове, задачата на един процес е просто да го отвори и прочете - какво се случва,
ако файлът липсва (а не трябва), е проблем на някой друг процес.

---
* Грешките в Elixir **не са** препоръчителни за употреба.
* Имат славата на GOTO програмиране и наистина е хубаво да помислим дали има нужда от тях в дадена ситуация.

---
- Прието е функции, при които има проблем да връщат
    ```elixir
      {:error, <проблем>}
    ```
- Ако се изпълняват с успех ще имат резултат|
    ```elixir
      {:ok, <резултат>}
    ```
- Имената на функции, които биха могли да 'вдигат' грешка, обикновено завършват на '!'.|

---
*'Вдигане'* и *'спасяване'*

![Image-Absolute](assets/iamerror.jpg)

---
```elixir
raise "Ужаст!"
#=> (RuntimeError) Ужаст!
```

---
```elixir
raise RuntimeError
#=> (RuntimeError) runtime error
```

---
```elixir
raise ArgumentError, message: "Грешка, брато!"
#=> (ArgumentError) Грешка, брато!
```

---
```elixir
try do
  1 / 0
rescue
  [RuntimeError, ArgumentError] ->
    IO.puts("Няма да стигнем до тук.")
  error in [ArithmeticError] ->
    IO.puts("На нула не се дели, #{error.message}")
  any_other_error ->
    IO.puts("Лошаво... #{any_other_error.message}")
else
  IO.puts("Няма грешка.")
after
  IO.puts("Finally!")
end
```

---
### Създаване на нови типове грешки
![Image-Absolute](assets/new_exception.jpg)

---
```elixir
defmodule VeryBadError do
  defexception message: "Лошо!!!"
end
```

---
```elixir
try do
  raise VeryBadError
rescue
  error in VeryBadError ->
    IO.puts(inspect(error, structs: false))
end
#=> %{
#=>    __exception__: true,
#=>    __struct__: VeryBadError,
#=>    message: "Лошо!!!"
#=> }
```

---
Throw/Catch

![Image-Absolute](assets/spaghetti.jpg)

---
С `throw` *'подхвърляме'* стойност, която може да се *'хване'* по-късно:

```elixir
try do
  b = 5
  throw b
  IO.puts "this won't print"
catch
  :some_atom           -> IO.puts "Caught #{:some_atom}!"
  x when is_integer(x) -> IO.puts(x)
end
#=> 5
```

---
Нещо още по-рядко (г/д колкото трикрак еднорог):
```elixir
try do
  exit("I am exiting") # така можем да излезем от процес
catch
  :exit, _ -> IO.puts "not really"
end # и процесът всъщност остава жив
```

---
### Сега - забравете за тях и не ги ползвайте.
![Image-Absolute](assets/forget.jpg)

---
1. В кода на mix няма прихващане на грешки.
2. В кода на компилатора на Elixir има точно пет прихващания.

---

За целите на този курс се забраняват (освен ако не е необходимо или изрично указано иначе):
* `if, unless, cond` (напомняме)
* `try/catch`
* `try/rescue`
* `raise, throw`


---
## Вход-изход
![Image-Absolute](assets/pipes.jpg)

---
Изход с `IO.puts/2` и `IO.write/2`

```elixir
IO.puts("По подразбиране пишем на стандартния изход.")
IO.puts(:stdio, "Можем да го направим и така.")
IO.puts(:stderr, "Или да пишем в стандартния изход за грешки.")
```

---
```elixir
IO.write(:stderr, "Това е грешка!")
```

---
* Първият аргумент на `puts` и `write` може да е атом или *pid*. Нарича се `device` и представлява друг процес.
* Вторият аргумент се очаква да е нещо от тип *chardata*.

---
![Image-Absolute](assets/wait_what.jpg)

---
Какво е *chardata*?

* Низ, да речем `"Далия"`.
* Списък от codepoint-и, да речем `[83, 79, 0x53]` или `[?S, ?O, ?S]` или `'SOS'`.
* Списък от codepoint-и и низове - `[83, 79, 83, "mayday!"]`.
* Списък от chardata, тоест списък от нещата в горните три точки : `[[83], [79, ["dir", 78]]]`.

---
```elixir
IO.chardata_to_string([1049, [1086, 1091], "!"])
#=> "Йоу!"
```

---
Имаме и `IO.inspect/2`.
* Връща каквото му е подадено. Може да се `chain`-ва.
* Приема *pretty print* опции.
* Приема етикети.
* Чудесно за debugging.

---

```elixir
defmodule TaskEnum do
  def map(enumerable, fun) do
    enumerable
    |> IO.inspect(label: "Input", structs: false)
    |> Enum.map(& Task.async(fn -> fun.(&1) end))
    |> IO.inspect(label: "Tasks", width: 120, limit: :infinity)
    |> Enum.map(& Task.await(&1))
  end
end
```

---
Вход с IO.read/2, IO.gets/2, IO.getn/2 и IO.getn/3

```elixir
IO.read(:line)
Хей, Хей<enter>
#=> "Хей, Хей\n"
```

```elixir
IO.gets("Кажи нещо!\n")
Кажи нещо!
Нещо!<enter>
#=> "Нещо!\n"
```

---
<p>
IO Server Time
</p>
![Image-Absolute](assets/its-showtime.jpg)

---
* Функции като `write` и `read` имат версии наречени `binwrite` и `binread`.
* Разликата е, че приемат `iodata`, вместо `chardata`.
* По бързи са. Добри за четене на *binary*/не-unicode файлове.


---
Какво е *iodata*?

* Подобно на *chardata*, *iodata* може да се дефинира като списък.
* За разлика от *chardata*, *iodata* списъкът е от цели числа които представляват байтове (0 - 255),
* *binary* с елементи със *size*, кратен на **8** (могат да превъртат) и такива списъци.

---
```elixir
IO.iodata_length([1, 2 | <<3, 4>>])
#=> 4
```

```elixir
IO.iodata_to_binary([1, << 2 >>, [[3], 4]])
#=> <<1, 2, 3, 4>>
```

---
### Файлове
![Image-Absolute](assets/folders.jpg)

---
```elixir
{:ok, file} = File.open("test.txt", [:write])
#=> {:ok, #PID<0.855.0>}

IO.binwrite(file, "some text!")
#=> :ok

File.close(file)
#=> :ok
```

---
Потоци и файлове

```elixir
{:ok, file} = File.open("program.txt", [:read])
#=> {:ok, #PID<0.82.0>}

IO.stream(file, :line)
|> Stream.map(fn line -> line <> "!" end)
|> Stream.each(fn line -> IO.puts line end)
|> Stream.run()
```

---
```elixir
File.stream!(input_name, read_ahead: <buffer_size>)
|> Stream.<transform-or-filter>
|> Stream.into(File.stream!(output_name, [:delayed_write]))
|> Stream.run
```

---
Модула `IO.ANSI`

```elixir
IO.puts [IO.ANSI.blue(), "text", IO.ANSI.reset()]
```

---
Модула `StringIO` и файлове в паметта

```elixir
{:ok, pid} = StringIO.open("data") #PID<0.136.0>}
StringIO.contents(pid) # {"data", ""}

IO.write(pid, "doom!") #:ok
StringIO.contents(pid) # {"data", "doom!"}
IO.read(pid, :line) # "data"
StringIO.contents(pid) # {"", "doom!"}

StringIO.close(pid) # {:ok, {"", "doom!"}}
```

---
```elixir
File.open("data", [:ram])
#=> {:ok, {:file_descriptor, :ram_file, #Port<0.1578>}}
IO.binread(file, :all)
#=> "data"
```

```elixir
IO.binread(file, :all)
#=> ""
```

---
```elixir
:file.position(file, :bof)
IO.binread(file, :all)
#=> "data"
```

---
Модула `Path`

```elixir
Path.join("some", "path")
#=> "some/path"
Path.expand("~/development")
#=> "/home/meddle/development"
```

---
Портове

![Image-Absolute](assets/process_file.jpg)


---
## The End
![Image-Absolute](assets/nap.jpg)
