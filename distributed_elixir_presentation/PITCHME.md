---
## Дистрибутиран Elixir

---
### Да си припомним
![Image-Absolute](assets/chemmy.jpeg)

---
Нужни знания досега:

1. Какво е BEAM Node?
2. Какво е OTP Application?
3. Каквое Supersvision дърво?

---
Как си пускаме BEAM node?

```bash
iex -S mix
```

---
Ако не ни е нужен shell?

```bash
mix run --no-halt
```

```bash
elixir -S mix run --no-halt
```

---
И пак mix, mix, mix!

```bash
elixir --detached -S mix run --no-halt
```

---
![Image-Absolute](assets/mixmixmix.jpg)

---
#### MIX
* Mix ни компилира кода и го слага на правилно място. |
* Mix ни генерира app file и ни оправя зависимостите. |
* Mix знае пътищата, които са ни нужни и как да пуска проекта ни. |

---
#### RELEASE
* Има как да правим release, който да се пуска в production и не ползва Mix. |
* OTP Release е нещо, което или може да се пусне чрез инсталиран Erlang (не трябва Elixir) и даже без Erlang. |
* В Elixir, ние си имаме Distilery за да ни построи тези OTP release-и. |

---
![Image-Absolute](assets/distil.png)

---
![Image-Absolute](assets/offfftopic.jpg)


---
![Image-Absolute](assets/title.jpg)

---
### Съдържание

1. Какво означава да сме дистрибутирани?
2. Node-ове и функции за свързване и комуникация
3. Проблемите на дистрибутираните системи
4. Да си напишем чат!
5. Да помислим за проблеми с чата...
6. Тестване на дистрибутиран Elixir
7. Кога да ползваме дистрибутиран Elixir

---
* Дистрибутираност значи програмата ни да върви на повече от една виртуални машини.
* Elixir ни дава добри инструменти за постигане на това. |

---
* Един node е една виртуална машина.
* Те може да са отворени за комуникация с други ноудове или не. |

---
### Функции за статус на node

* Как да видим името на node-a си? |
* Как да направим node-а си 'жив'? |

---
### Свързване на node-ове

* Списък на свързаните node-ове. |
* Свързване на node-ове. |
* Връзките между node-ове са транзитивни. |

---
### Създаване на процеси на отдалечени node-ове
* Node.spawn и неговите версии. |
* Основата на комуникацията между node-ове са процесите. |

---
```elixir
# @slavi
defmodule DB do
  def tell_us_secret do
    IO.puts("Познавам всички и всеки!")
  end
end
```

```elixir
# @valo
Node.spawn(:slavi@meddland, DB, :tell_us_secret, [])
#output: Познавам всички и всеки!
#=> #PID<9473.144.0>
```

---
### Наблюдаване на Node-ове

```elixir
Node.monitor(some_node, true)
```

---
### Извикване на отдалечени функции
* Използва се Erlang модула :rpc. |
* Отдолу работим с процеси. |

---
:rpc.call/4 (Има и cast версия)

```elixir
# @valo

:rpc.call(:slavi@meddland, DB, :tell_us_secret, [])
#=> Познавам всички и всеки!
#=> :ok
```

---
async_call и yield

```elixir
# @valo
pid = :rpc.async_call(:meddle@meddland,
                      Enum,
                      :sort,
                      [(1..2_000_000) |> Enum.shuffle])
#=> #PID<0.115.0>

# Това няма да забие текущия процес докато резултата е готов.

:rpc.yield(pid)
# Ще чака за резултат, или ако е готов ще го върне
```

---
nb_yield

```elixir
# @valo
pid = :rpc.async_call(:slavi@meddland, Process, :sleep, [50_000])
#=> #PID<0.108.0>

# По подразбиране timeout-a е нула.
# Пробва и ако няма резултат - :timeout
:rpc.nb_yield(pid)
#=> :timeout
:rpc.nb_yield(pid, 1_000)
#=> :timeout

:rpc.nb_yield(pid, 50_000)
#=> {:value, :ok}
```

---
multicall и eval_everywhere

```elixir
# @slavi
Node.list()
#=> [:meddle@meddland, :valo@meddland]

:rpc.multicall([Node.self() | Node.list()], Node, :self, [])
#=> {[:slavi@meddland, :meddle@meddland, :valo@meddland], []}
```

---
## EPMD
![Image-Absolute](assets/epmd.jpg)

---
* Когато стартираме node с име или пък му дадем име в последствие, той ще се свърже към програма, наречена EPMD или Erlang Port Mapper Daemon.
* Такава програма върви на всеки компютър на който има поне един 'жив' Erlang или Elixir node.

---
* EPMD е нещо като сървър за имена, който позволява регистриране, комуникация и връзка между node-ове.
* EPMD map-ва имена на node-ове към машинни адреси. |
* Програмата пази само name частта от name@host, защото си знае хоста. |

---
### Кога се стартира EPMD?

* Ако няма вървящ EPMD и стартираме node с име, автоматично се стартира. |
* Портът му по подразбиране е 4369, но може да се конфигурира друг. |
* Не е добра идея, защото Ericsson са го регистрирати официално за EPMD и би трябвало да е свободен. |

---
```bash
iex --sname andi --erl \
  "-kernel inet_dist_listen_min 54300 inet_dist_listen_max 54400"
```

---
```elixir
:net_adm.names()
```

---
## PID
![Image-Absolute](assets/pid.jpg)

---
```elixir
# От valo@meddland

Node.spawn(
  :meddle@meddland,
  fn -> send(pid(0, 86, 0), "Hello from valo!") end
)
# #PID<9107.110.0>
```

---
* Това, което Node.spawn/2 връща е pid, но изглежда малко странно.
* Свикнали сме първото число да е 0, а тук не е. |
* Това е така защото първото число на pid-а е свързано с node-а, на който процеса му се изпълнява. |

---
* Ако процесът върви на текущия node, то винаги е 0.
* Ако обаче процесът върви на друг node, числото уникално ще идентифицира този друг node. |
* Тази стойност за един и същи node ще е различна на различни node-ове, свързани с него. |

---
### Типа данни PID

* Първото число на pid показва на кой node върви процеса.
* Второто е брояч, а третото допълнение към този брояч.
* Когато минем максималния брой процеси за дадения node, третото число се увеличава с едно.

---
### Демонстрация

```elixir
binary_pid = :erlang.term_to_binary(pid)
```

---
### Свързване с вървящ node

```bash
elixir --detached --name bot@127.0.0.1 -S mix run --no-halt
iex --name botor@127.0.0.1 --remsh bot@127.0.0.1 --hidden
```

---
### Отклонение от темата #2
![Image-Absolute](assets/Screenshot_20190528_000840.png)

---
## Дистрибутирани програми - проблемите

---
![Image-Absolute](assets/fallacies-fallacies-one-for-you-and-two-for-me.jpg)

---
* На мрежата не винаги може да се разчита. Node-ове могат да изчезнат.
* Мрежата може да бъде бавна от време на време. Резултатите от извиквания могат да се забавят. |
* Bandwidth. Малки и прости съобщения! |
* Security. Това трябва да си го постигнем сами! |

---
* Топологията на мрежата не е константа - имена и локации - трябва да внимаваме.
* Рядко ние имаме пълен контрол над физическите машини в мрежата. |
* Транспортът е скъп. Малки, прости съобщения! |
* Мрежата няма определен формат. Трябва ние да определим формат и протокол. |

---
### CAP
![Image-Absolute](assets/scalability-cap-theorem1.png)

---
![Image-Absolute](assets/chap1_cap_theorem.png)

---
![Image-Absolute](assets/25518fed78bdf02cec9041d90c00bd5f--jesse-pinkman-aaron-paul.jpg)

---
### Coding Time!
![Image-Absolute](assets/ouroboros.jpg)
