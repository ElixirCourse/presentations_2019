---
## Конкурентно програмиране II
Връзки между процеси и състояние

---
![Image-Absolute](assets/title.png)

---
## Съдържание

1. Връзки между процеси
2. Наблюдение на процеси
3. Пазене на състояние в процеси

---
## Връзки между процеси
![Image-Absolute](assets/chains.jpg)

---
Какво става, когато процес 'умре'?

* Бива изчистен от паметта и просто спира да съществува. |
* Никой ли не научава за това? |

---
```elixir
defmodule Quitter do
  def run do
    Process.sleep(3000)
    exit(:i_am_tired)
  end
end
```

---
```elixir
pid = spawn(Quitter, :run, [])

# След няколко секунди
Process.alive?(pid)
#=> false
```

---
![Image-Absolute](assets/quitter.jpg)

---
* Aко обаче ползваме `spawn_link`:

```elixir
spawn_link(Quitter, :run, [])
```

---
```elixir
spawn_link(Quitter, :run, [])

# След няколко секунди ще има грешка
```

---
```elixir
defmodule Quitter do
  def run_no_error do
    Process.sleep(3000)
  end
end

pid = spawn_link(Quitter, :run_no_error, [])

# След няколко секунди
Process.alive?(pid)
#=> false
```

---
### spawn_link
* Ако някой от свързаните процеси излезе с грешка - всички свързани 'умират'.
* Ако някой от свързаните процеси излезе без грешка - другите не 'умират'.
* Връзките са двупосочни.


---
Process.unlink
```elixir
pid = spawn_link(Quitter, :run, [])

Process.unlink(pid)

# Няма грешка
```

---
### Process.link
![Image-Absolute](assets/connection.jpg)

---
```elixir
defmodule Simple do
  def run do
    receive do
      {:link, pid} when is_pid(pid) ->
        Process.link(pid)
        run()
      :links ->
        IO.puts(inspect(Process.info(self(), :links)))
        run()
      :die ->
        IO.puts("#{inspect self()} is hit!")
        exit(:ouch)
    end
  end
end
```

---
```elixir
send(pid1, {:link, pid2}) # {:link, #PID<0.272.0>}
send(pid2, {:link, pid3}) # {:link, #PID<0.274.0>}
send(pid3, {:link, pid4}) # {:link, #PID<0.276.0>}
send(pid4, {:link, pid5}) # {:link, #PID<0.278.0>}
```

---
```elixir
send(pid1, :links) # {:links, [#PID<0.272.0>]}
send(pid2, :links) # {:links, [#PID<0.270.0>, #PID<0.274.0>]}
send(pid3, :links) # {:links, [#PID<0.272.0>, #PID<0.276.0>]}
send(pid4, :links) # {:links, [#PID<0.274.0>, #PID<0.278.0>]}
send(pid5, :links) # {:links, [#PID<0.276.0>]}
```

---
### Съобщения при грешка
![Image-Absolute](assets/unexpected-error.jpg)

---
* Тази грешка, която транзитивно убива свързаните процеси, е специално съобщение.
* Тези специални съобщения се наричат сигнали.  |
* 'Exit' сигналите са 'тайни' съобщения, които автоматично убиват процесите, които ги получат. |

---
* За да имаме 'fault-tolerant' система е добре да имаме способ да убиваме процесите ѝ
и да разбираме кога процес е бил ликвидиран.
* Тази fault-tolerant система трябва да има начин да рестартира процеси, когато те 'умрат'.

---
### Системни процеси
![Image-Absolute](assets/A-System-Administrator.jpg)

---
* В Elixir има специален вид процеси - 'системни процеси'.
* Това са нормални процеси, които могат да трансформират 'exit' сигналите, които получават и нормални съобщения. |

---
```elixir
defmodule Quitter do
  def run do
    Process.sleep(3000)
    exit(:i_am_tired)
  end
end

Process.flag(:trap_exit, true)

spawn_link(Quitter, :run, [])
receive do msg -> IO.inspect(msg); end

#=> {:EXIT, #PID<0.95.0>, :i_am_tired}
```

---
### Грешки и поведение
![Image-Absolute](assets/shit.jpg)

---
```elixir
action = <?>
system_process = <?>

Process.flag(:trap_exit, system_process)
pid = spawn_link(action)

receive do
  msg -> IO.inspect(msg)
end
```

---
### Случай 1 :
```elixir
action = fn -> :nothing end
```

1. При `system_process = false` : Нищо. Текущият процес чака.
2. При `system_process = true`  : Ще видим `{:EXIT, pid, :normal}`. Текущият процес продължава.

---
### Случай 2 :
```elixir
action = fn -> exit(:stuff) end
```

1. При `system_process = false` : Текущият процес умира.
2. При `system_process = true`  : Ще видим `{:EXIT, pid, :stuff}`. Текущият процес продължава.

---
### Случай 3 :
```elixir
action = fn -> exit(:normal) end
```

1. При `system_process = false` : Нищо. Текущият процес чака.
2. При `system_process = true`  : Ще видим `{:EXIT, pid, :normal}`. Текущият процес продължава.

---
### Случай 4 :
```elixir
action = fn -> raise("Stuff") end
```

1. При `system_process = false` : Текущият процес умира.
2. При `system_process = true`  : Ще видим `{:EXIT, pid, {%RuntimeError{message: "Stuff"}....}`. Текущият процес продължава.

---
### Случай 5 :
```elixir
action = fn -> throw("Stuff") end
```

1. При `system_process = false` : Текущият процес умира.
2. При `system_process = true`  : Ще видим `{:EXIT, pid, {{:nocatch, "Stuff"}, [...]}}`. Текущият процес продължава.

---
* Тези пет поведения покриват всички възможни случаи.
* Засега няма начин системен процес да бъде терминиран с какъвто и да е сигнал.

---
Функцията `Process.exit/2`
<br />
![Image-Absolute](assets/gun.jpg)

---
```elixir
defmodule HelloPrinter do
  def start_link do
    Process.sleep(5000)
    IO.puts("Hello")
    start_link()
  end
end
```

---
![Image-Absolute](assets/shut_up.jpg)

---
```elixir
Process.flag(:trap_exit, true)

pid = spawn_link(HelloPrinter, :start_link, [])
# Започва да печата "Hello" на всеки 5 секунди

Process.exit(pid, :spri_se_be)
# Спира да печата "Hello"

receive do
  msg -> IO.inspect(msg)
end
#=> {:EXIT, #PID<0.162.0>, :spri_se_be}
```

---
```elixir
defmodule HiPrinter do
  def start_link do
    Process.flag(:trap_exit, true)
    print()
  end

  defp print do
    receive do
      {:EXIT, pid, _} -> send(pid, "Няма да стане, батка!")
    after
      6000 -> IO.puts("Hi!")
    end
    print()
  end
end
```

---
```elixir
Process.exit(pid, :kill)
```

---
* Process.exit/2 с атома `:kill`, може да убие всякакъв процес, даже системен.
* Съобщението има статус `:killed`.

---
* Статусът от `:kill` стана `:killed`.
* Процесът, който изпрати :kill, беше свързан с този, който беше 'убит', и не искаме сигналът да рекошира обратно.
* Сигналът `:kill` не е транзитивен към връзките.

---
### Ликвидиране и поведение
![Image-Absolute](assets/terminator.jpg)

---
```elixir
action = <?>
system_process = <?>
status = <?>

Process.flag(:trap_exit, system_process)
pid = spawn_link(action)

Process.exit(pid, status)

receive do
  msg -> IO.inspect(msg)
end
```

---
### Случай 1 :
```elixir
action = fn -> Process.sleep(20_000) end

status = :normal
```

1. При `system_process = false` : Нищо. Текущият процес чака.
2. При `system_process = true`  : Нищо. Текущият процес чака.

---
* Процес не може да бъде 'убит' с `Process.exit(pid, :normal)`


---
### Случай 2 :
```elixir
action = fn -> Process.sleep(20_000) end

status = :stuff
```

1. При `system_process = false` : Грешка. Текущият процес 'умира'.
2. При `system_process = true`  : Получаваме съобщение `{:EXIT, pid, :stuff}`.

---
### Случай 3 :
```elixir
action = fn -> Process.sleep(20_000) end

status = :kill
```

1. При `system_process = false` : Грешка. Текущият процес 'умира'.
2. При `system_process = true`  : Получаваме съобщение `{:EXIT, pid, :killed}`.

---
### Случай 4 :
```elixir
action = fn -> Process.exit(self(), :kill) end

status = <каквото-и-да-е>
```

1. При `system_process = false` : Грешка. Текущият процес 'умира'.
2. При `system_process = true`  : Получаваме съобщение `{:EXIT, pid, :killed}`.

---
* `exit(:reason)` е различен от `Process.exit(pid, :reason)`.
* `exit(:reason)` е нещо като `throw`, предизвиква 'хвърляне', което, ако не е хванато, 'убива' текущия процес.

---
## Наблюдение на процеси
![Image-Absolute](assets/watcher.gif)

---
* Свързването на процеси е двупосочно.
* Ако единият от тях 'умре', другият ще получи EXIT сигнал.  |
* Този сигнал ще 'убие' всички свързани процеси, ако те не са системни.  |

---
* Често искаме един процес да наблюдава друг без да му праща сигнал, ако случайно 'умре'.
* Също така искаме да наблюдаваме процеси без текущият процес да е системен.  |
* Ако те 'умрат', просто искаме да бъдем нотифицирани с нормална нотификация, за да предприемем нещо.  |

---
```elixir
pid = spawn(fn -> Process.sleep(3000) end)

Process.monitor(pid)

receive do
  msg -> IO.inspect(msg)
end
# След 3 секунди ще получим нещо такова
#=> {:DOWN, #Reference<...>, :process, #PID<...>, :normal}
```

---
* Добавяме монитор към процес с Process.monitor(pid).
* Тази фунцкия връща референция.  |
* Референциите са специален тип в Elixir, всяка от тях е уникална за текущия node.  |

---
* Когато получим DOWN съобщението, тази референция ще е вторият му елемент.
* Четвъртият е PID-а на процеса, който е завършил изпълнението си, а петият - статус.  |
* Това са същите тези статуси, които причиняваха сигналите при свързани процеси.  |

---
```elixir
pid = spawn(fn -> Process.sleep(3000) end)

ref = Process.monitor(pid)
Process.demonitor(ref)

receive do
  msg -> IO.inspect(msg)
after
  4000 -> IO.puts("Няма съобщения...")
end
#=> Ще видим 'Няма съобщения...'
```

---
* Има и версия на `spawn`, която създава нов процес и автоматично му добавя монитор:

```elixir
Process.flag(:trap_exit, true)

{pid, ref} = spawn_monitor(fn -> Process.sleep(3000) end)
Process.exit(pid, :kill)

receive do
  msg -> IO.inspect(msg)
end
#=> {:DOWN, <ref>, :process, pid, :killed}
```

---
## Пазене на състояние в процеси

---
![Image-Absolute](assets/wrapped.jpg)


---
# Край
![Image-Absolute](assets/the-end.jpg)
