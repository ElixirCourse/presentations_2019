---
## Конкурентно програмиране I

---
![Image-Absolute](assets/To-be-a-one-man-band.jpg)

---
## Съдържание

0. Неща които ще трябва да знаем
1. Абстракцията `Task`
2. Абстракцията `Agent`

---
## Неща които ще трябва да знаем

---

### Process.sleep/1

* "Приспива" процеса от който викаме функцията |
* Единствения аргумент, който получава е време (в милисекунди) за което процеса да "спи" |
* Докато един процес "спи" той не прави нищо |

---

### Process.sleep/1

Използване

```elixir
iex> Process.sleep(10_000)
# Спим за 10 секунди
```

---

### Kernel.apply/2

Използва се така:

```elixir
func = fn x, y -> x*y end
apply(func, [3, 4]) # => 12
# горното е аналогично на func.(3, 4)
apply(func, [3]) # => Грешка
```

---

### Kernel.apply/3

* Същото като Kernel.apply/2, но за именувани функции |
* Използва се така: </br>apply(module, function, arguments) |
* module е атом |
* function е атом |
* arguments е списък с аргументи, които да бъдат подадени на функцията |
* Често на това ще казваме MFA |

---

### Kernel.apply/3

Използване

```elixir
defmodule A do
  def mult(x, y), do: x * y
end

apply(A, :mult, [3, 4]) # => 12
# горното е аналогично на A.mult(3, 4)
apply(A, :mult, [3]) # => Грешка
```

---

### Четирите измерения на Eликсир

![Image-Absolute](assets/multiple.png)

---

### Функционалният Еликсир

* Това е кодът, който се изпълнява в един процес. |
* Той се изпълнява последователно. |
* Данните са непроменими. |
* Това са нещата, за които предимно говорим до сега. |

---

### Конкурентният Еликсир

* Съвкупност от много процеси вървящи в един BEAM Node (една инстанция на виртуалната машина). |
* Те си комуникират с размяна на съобщения. |
* Всеки от тях има роля или множество роли. |
* Ако ги разглеждаме като конструкции в езика, отвън изглеждат променими. |
* В днешната лекция ще разгледаме две абстракции над процеси. |

---

### Дистрибутираният Еликсир

* Съвкупност от много BEAM Node-ове, които са свързани. |
* Процесите от тези "нодовеа" могат да комуникират помежду си. |
* За това ще говорим в края на курса. |

---

### Мета Еликир

* Еликсир ни дава възможност да пишем макроси. |
* Те са специални функции, който се изпълняват по време на компилация и генерира друг код. |
* С тях можем да разширяваме езика с нови конструкции. |
* На практика Еликсир е имплементиран чрез макроси. |
* И за това ще говорим в края на курса. |

---
## Задачи
![Image-Absolute](assets/taskmaster.webp)

---
### Задача

* Най-лесният начин да изпълним нещо конкуентно на текущата ни код е с абстракцията дадена ни в модула Task.
* Функциите от този модул работят със специална структура описваща задачата.

```elixir
%Task{
  owner: #PID<0.89.0>,
  pid: #PID<0.273.0>,
  ref: #Reference<0.999173714.2842427393.228068>
}
```
---
### Задача

* За сега не е необходимо напълно да разбираме, какво представлява тази структура. |
* Създаваме я с една от функциите: |
  * Task.async/1 - вариант с анонимна функция
  * Task.async/3 - вариант с MFA
* Можем да получим резултата от изпълнената задача с функциите Task.await/2 |

---

```elixir
iex> task = Task.async(fn -> 1 + 1 end)
%Task{...}

# Друга логика може да се изпълни тук...
# Когато сме готови:

iex> Task.await(task)
2
```

---
### async и await

* Функцията Task.async всъщност създава нов Elixir-ски процес. |
* Този процес е специализиран да изпълни едно основно действие и да върне резултата от него. |
* Функцията Task.await ще блокира текущия процес докато задачата завърши и резултата не е наличен. |
* Ако резултатът е наличен, когато я извикаме, направо ще върне стойността му. |

---
### async и await

```elixir
iex> task = Task.async(fn ->
  Process.sleep(10_000) # Симулираме дълга задача
  1 + 2
end)
%Task{...}
iex> Task.await(task)
** (exit) exited in: Task.await(%Task{...}, 5000)
     ** (EXIT) time out
```
---

#### Какво ще стане, ако задачата отнеме твърде много време?

---

### await и timeout

* Вторият параметър на Task.await/2 е timeout (5_000 по подразбиране). |
* Ако изиваме await и това време изтече се случва грешка. |
* Можем да дадем цяло положително число за timeout, което означава колко милисекунди да чакаме. |
* Също така може да дадем и атома :infinity (означава, че няма timeout). |

---
### await и timeout

```elixir
iex> task = Task.async(fn ->
  Process.sleep(10_000) # Симулираме дълга задача
  1 + 2
end)
%Task{...}
iex> Task.await(task, 11_000) # Ще почакаме около 10 секунди
3
```

---

#### Какво ще стане ако извикаме `await` повторно, след като вече имаме резултат?

---

```elixir
iex> task = Task.async(fn -> 2 + 2 end)
%Task{...}
iex> Task.await(task)
4
iex> Task.await(task)
** (exit) exited in: Task.await(%Task{...}, 5000)
    ** (EXIT) time out
```

---
### async и await

* Повторното извикване ще блокира и ще чака резултат. |
* Тъй като задачата вече не съществува, няма да има нов резултат. |
* След пет секунди ще има timeout грешка. |
* Изводът е, че едно извикване на async трябва да съответства на точно едно извикване на await! |

---
### MFA

```elixir
iex> task = Task.async(Kernel, :+, [2, 3])
%Task{...}
iex> Task.await(task)
5
```

---
#### Parallel map - повторение

```elixir
defmodule TaskEnum do
  def map(enumerable, fun) do
    enumerable
    |> Enum.map(& Task.async(fn -> fun.(&1) end))
    |> Enum.map(& Task.await(&1))
  end
end
```

---
### Проверка за резултат на даден интервал
![Image-Absolute](assets/are_we_there_yet.jpeg)


---
### yield

* Абстракцията Task може да се ползва като future стойностите от други езици и системи. |
* За тази цел, бихме могли да ползваме функцията Task.yield/2. |
* Подобно на Task.await/2, yield блокира текущия процес за дадено време: |
  * Ако резултатът не е готов за даденото връща nil.
  * В противен случай връща {:ok, result}
---
```elixir
iex> task = Task.async(fn ->
  Process.sleep(5_000)
  3 + 3
end)
%Task{...}
iex> Task.yield(task, 1_000)
nil
iex> Task.yield(task, 2_000)
nil
iex> Task.yield(task, 3_000)
{:ok, 6}
```

---
* Изглежда, че *poll*-ваме задачата за резултат, но не е така.
* Процесите в Elixir комуникират помежду си чрез размяна на съобщения, което прилича на изпращане на *push нотификации*.
* Когато задачата си свърши работата, резултатът ще бъде **изпратен** до текущия процес.
* Само процесът `onwer` на задачата може да извика `Task.await/2` или `Task.yield/2`.

---
```elixir
task1 = Task.async(fn -> 4 + 3 end)
#=> %Task{owner: #PID<0.10199.0>, ...}

Task.async(fn -> Task.yield(task1) end)
#=> ** (EXIT from #PID<0.10199.0>) process exited with reason:
#=>     ** (ArgumentError) task %Task{owner: #PID<0.10199.0>, ...}
#=>       must be queried from the owner
#=>       but was queried from #PID<0.10205.0>
```

---
* При успех `yield` връща `{:ok, result}`.
* Ако все още няма резултат, връща `nil`.
* Друго нещо, което може да върне е `{:exit, reason}`, ако задачата завърши без резултат:

```elixir
defmodule Calc do
  def sum(a, b) when is_number(a) and is_number(b), do: a + b
  def sum(_, _), do: Kernel.exit(:normal)
end
```

---
```elixir
task = Task.async(Calc, :sum, [4, 4])
#=> %Task{...}

{:ok, 8} = Task.yield(task)
#=> {:ok, 8}

task = Task.async(Calc, :sum, ["5", 4])
#=> %Task{...}

{:exit, :normal} = Task.yield(task)
#=> {:exit, normal}
```

---
### Освобождаване на задача

```elixir
task = Task.async(fn -> Process.sleep(10_000) end)

case Task.yield(task) || Task.shutdown(task) do
  {:ok, result} -> result
  nil -> {:error, "Task took too much time!"}
end
#=> {:error, "Task took too much time!"}

Process.alive?(task.pid)
#=> false
```

---
### Фукнцията Task.async_stream/3
![Image-Absolute](assets/async_stream.gif)

---
* Приема колекция и функция, която да изпълни на всеки елемент на колекцията.
* Подадената функция се изпълнява в различен `Task` за всеки елемент.
* Когато опитаме да консумираме този *stream*, всяка задача ще бъде изчакана дадено време.

---
```elixir
defmodule TaskEnum do
  def map(enumerable, fun) do
    enumerable
    |> Task.async_stream(fun)
    |> Stream.map(fn {:ok, val} -> val end)
    |> Enum.to_list()
  end
end
```

---
### Пример Github
![Image-Absolute](assets/stargazers.jpg)

---
## Агенти
![Image-Absolute](assets/agent.jpg)


---
* Подобно на `Task`, `Agent` е абстракция над Elixir-ски процес.
* Агентът представлява състояние, което е съхранено в отделен процес.
* То е достъпно от други процеси или от един и същ процес на различни етапи от изпълнението му.

---
### SIDE EFFECTS
![Image-Absolute](assets/side_effects.png)

---
### Агентът като стойност

```elixir
{:ok, agent} = Agent.start(fn -> 1 end)
#=> {:ok, #PID<0.187.0>}

Agent.get(agent, fn v -> v end)
#=> 1

Agent.update(agent, fn v -> v + 1 end)
#=> :ok

Agent.get(agent, fn v -> v end)
#=> 2
```

---
### Агентът като споделена стойност

```elixir
defmodule DependentTask do
  def execute(agent) when is_pid(agent) do
    agent |> Agent.get(&(&1)) |> run(agent)
  end

  defp run(n, agent) when n < 5, do: execute(agent)
  defp run(n, _), do: n * n
end
```

---
Стойността е достъпна от множество процеси!
```elixir
{:ok, agent} = Agent.start(fn -> 1 end)
task = Task.async(DependentTask, :execute, [agent])

nil = Task.yield(task, 100)
nil = Task.yield(task, 1000)

:ok = Agent.update(agent, fn _ -> 10 end)
{:ok, val} = Task.yield(task, 100)
#=> {:ok, 100}
```

---
* Задачата и текущият процес имат достъп до една и съща стойност от трети процес - агента.
* Задачата по дефиниция зависи от стойността в агента.
* На практика можем да направим и пуснем друга задача, имаща достъп до адреса на агента, която може да промени тази стойност.
* По такъв начин получаваме нещо като споделено състояние.

---
* Разликата е, че това състояние е винаги *immutable*.
* Съобщенията които постъпват в процеса-агент се усвояват в реда в който са дошли.
* Възможно е да счупим нещата, но ако използваме правилно интерфейса на агента това няма да стане.

---
![Image-Absolute](assets/race_conditions.jpg)

---
Възможно е да имаме *race conditions*.

```elixir
defmodule Value do
  def init(v) do
    {:ok, value} = Agent.start(fn -> v end)
    value
  end

  def get(value), do: Agent.get(value, &(&1))
  def set(value, v), do: Agent.update(value, fn _ -> v end)
end
```

---
Възможно е да имаме *race conditions*.

```elixir
value = Value.init(5)
action = fn ->
  Process.sleep(50)
  v = Value.get(value)
  :ok = Value.set(value, v * 2)
end

{:ok, _} = Task.start(action)
{:ok, _} = Task.start(action)
{:ok, _} = Task.start(action)
{:ok, _} = Task.start(action)

Process.sleep(200)
Value.get(value)
```

---
### Функциите на Agent
* Агентът има `get`, `update` и атомарната `get_and_update` функции.
* С първите две дадохме примери.
* С третата можем да оправим предния пример!


---
### Функциите на Agent
* Видяхме, че `Agent` има функция за стартиране, на която се подава фунцкия, връщаша състоянието на агента.
* Има и `MFA` версия на `start`, както и `start_link` версии.
* Версиите с *link* биха убили процеса на агента, ако този, който го е пуснал вече не съществува и обратно.

---
Защо всичко се изпълнява във функция?
![Image-Absolute](assets/thinking.jpg)

---
* Тези функции, подавани на функциите на модула `Agent` се изпълняват в процеса-агент.
* Tрябва да бъдат кратки, за да не взимат много от времето на процеса.
* Промяната на състоянието на даден процес би трябвало да става в самия процес.
* Именно затова промяната и инициализирането на състоянието на `Agent`-а става във функции, които се ипълняват в процеса му.

---
* Можем да мислим за `Agent`-а и като за сървър.
* Процесите, които го ползват са клиенти, които комуникират с този сървър.
* Агентът трябва да е много прост сървър, неговата роля е просто да съхранява състояние.
* Това състояние може да бъде инициализирано, четено и променяно.
* Тежки изчисления, свързани със състоянието на агента не са част от неговата роля.

---
### Функциите на Agent
* Повечето са синхронни - чакаме за отговор.
* Състоянието на агент може да се променя и асинхронно - функцията `cast`.

---
### Пример Github
![Image-Absolute](assets/agent_github.png)

---
* В реалния живот за *cache* няма да ползвате агенти, защото има по-добри инструменти за това.
* За тестване и междинна работа агентите са добър избор.

---
## Край
![Image-Absolute](assets/end.jpg)
