---
## OTP Поведения

---
![Image-Absolute](assets/title.png)

---
## Какво е OTP?
![Image-Absolute](assets/what_is_otp.jpg)

---
* OTP е платформата, с която се разпространява Erlang.
* Версиите на Erlang са версии на OTP.  |
* OTP е стандартната библиотека на Erlang. |
* OTP е Open Telecom Platform, но с времето OTP се е превърнала от платформа за писане на телекомуникационни програми в нещо повече. |

---
![Image-Absolute](assets/erlang.jpg)

---
Платформата OTP идва с:
* Интерпретатор и компилатор на Erlang. |
* Стандартните библиотеки на Erlang. |
* Dialyzer. |
* Mnesia - дистрибутирана база данни. |
* ETS - база данни в паметта. |
* Дебъгер. |
* И много други... |

---
Ние разгледаме три абстракции от OTP:
* GenServer
* Supervisor
* Application

---
## GenServer
![Image-Absolute](assets/generic.jpg)

---
Какви функции?
* Функции за синхронна и/или асинхронна комуникация. |
* Функции, които се извикват при създаване на процес или преди унищожаването му. |
* Функции за нотифициране при грешки и за логване. |

---
* Когато комбинираме GenServer-и със Supervisor процеси лесно можем да създадем fault-tolerant система.

---
* С помощта на мета-програмиране GenServer е специален тип поведение.
* Задава callback функции, които имат имплементации по подразбиране. |
* За да направим от един модул GenServer, използваме "use GenServer". |

---
```elixir
defmodule MyWorker do
  use GenServer
end
```

---
### Поведението GenServer : Callback функции

---
#### init
![Image-Absolute](assets/init.jpg)

---
```elixir
@type args :: term()
@type state :: any()
@type reason :: any()
@type timeout :: non_neg_integer()

@type init_result ::
  {:ok, state()} |
  {:ok, state(), timeout() | :hibernate |  {:continue, term()}} |
  :ignore |
  {:stop, reason()}

@callback init(args) :: init_result()
```

---
#### handle_call
![Image-Absolute](assets/handle_call.jpg)

---
```elixir
@type from :: {pid(), ref()}
@type handle_call_result ::
  {:reply, reply, new_state} |
  {:reply, reply, new_state, timeout | :hibernate | {:continue, term()}} |
  {:noreply, new_state} |
  {:noreply, new_state, timeout | :hibernate | {:continue, term()}} |
  {:stop, reason, reply, new_state} |
  {:stop, reason, new_state}
  when reply: term, new_state: term, reason: term

@spec handle_call(request :: term, from, state :: term) ::
  handle_call_result
```

---
#### handle_cast
![Image-Absolute](assets/handle_cast.png)

---
```elixir
@spec handle_cast(request :: term, state :: term) ::
  {:noreply, new_state} |
  {:noreply, new_state, timeout | :hibernate | {:continue, term()}} |
  {:stop, reason :: term, new_state} when new_state: term
```

---
#### handle_info
![Image-Absolute](assets/handle_all.jpg)

---
```elixir
@spec handle_info(msg :: :timeout | term, state :: term) ::
  {:noreply, new_state} |
  {:noreply, new_state, timeout | :hibernate | {:continue, term()}} |
  {:stop, reason :: term, new_state} when new_state: term
```

---
#### terminate
![Image-Absolute](assets/terminate.png)

---
```elixir
@type reason :: :normal | :shutdown | {:shutdown, term} | term
@spec terminate(reason, state :: term) :: term
```

---
## Supervisor
![Image-Absolute](assets/supervisor.jpg)

---
### Какво е Supervisor?

* Важно е програмата да върви.
* Ако части от нея се сринат, не е проблем - нещо наблюдава тези части. |
* Нещо ще се погрижи те да бъдат възстановени. |
* Това нещо е Supervisor. |

---
* Подобно на GenServer, Supervisor е поведение, за което callback функциите имат имплементации по подразбиране.

---
```elixir
defmodule SomeSupervisor do
  use Supervisor
end
```

---
### Supervisor: Стратегии
![Image-Absolute](assets/strategy.jpg)

---
:one_for_one

* Ако наблюдаван процес 'умре', той се рестартира.
* Другите не се влияят.
* Тази стратегия е добра за процеси които нямат връзки и комуникация помежду си, които могат да загубят състоянието си без това да повлияе на другите процеси-деца на Supervisor-а им.

---
:one_for_all

* Ако наблюдаван процес 'умре', всички наблюдавани процеси биват 'убити' и след това всички се стартират наново.
* Обикновено тази стратегия се използва за процеси, чиито състояния зависят доста едно от друго и ако само един от тях бъде рестартиран ще е се наруши общата логина на програмата.

---
:rest_for_one

* Ако наблюдаван процес 'умре', всички наблюдавани процеси стартирани СЛЕД недго също 'умират'.
* Всички тези процеси, включително процесът-причина се рестартират по първоначалния си стартов ред.
* Тази стратегия е подходяща за ситуация като : процес 'A' няма зависимости, но процес 'Б' зависи от 'А', а има и процес 'В', който зависи както от 'Б', така и транзитивно от 'А'.

---
### Supervisor: max_restarts и max_seconds

* С опцията :max_restarts задаваме колко пъти може един процес да бъде рестартиран в даден период от време.
* По подразбиране е 3.
* Ако за :max_seconds секунди пробваме да рестартираме процеса :max_restarts пъти, трябва да се откажем.

---
### Какво е Supervisor.child_spec()
![Image-Absolute](assets/spec.png)

---
```elixir
@type child_spec() :: %{
  :id => atom() | term(),
  :start => {module(), atom(), [term()]},
  optional(:restart) => :permanent | :transient | :temporary,
  optional(:shutdown) => timeout() | :brutal_kill,
  optional(:type) => :worker | :supervisor,
  optional(:modules) => [module()] | :dynamic
}

```

---
### Наредена двойка за спецификация
```elixir
children = [
  {MyModule, [<args>]}
]
```

---
Алтернативно, модулът може да има зададена функция:
```elixir
def child_spec(arg) do
  %{
    id: <id>,
    start: {MyModule, :start_link, [arg]},
    # Другите ключове на спецификация могат да се зададат
  }
end
```

---
### Дърво от Supervisor-и

---
![Image-Absolute](assets/tree.png)

---
### DynamicSupervisor
![Image-Absolute](assets/dynamic_kids.jpg)

---
* Специален вид Supervisor
* Обикновено започва без процеси-деца |
* Когато има нужда - такива процеси се стартират със start_child/2 |
* Обикновено работи с деца с една и съща спецификация, но може и други да се задават |

---
### Task.Supervisor

* Понякога искаме задачите да изпращат резултат, но да не се link-ват към текущия процес. |
* Да речем това са задачи, които си говорят с отдалечен компонент/service и биха могли да получат грешка отвън. |

---
### GenServer и Task

* Когато създадем задача в GenServer с Task.Supervisor.async_nolink, ако не използваме Task.await/2, можем да си дефинираме handle_info callback, който ще бъде извикан с резултата от задачата във формата `{ref, result}`.

---
* Важното в подобни случаи е да дефинираме един допълнителен handle_info callback:
```elixir
def handle_info({:DOWN, _ref, :process, _pid, _status}, state)
```

---
## Application
![Image-Absolute](assets/application-controller.png)

---
### Какво е Application?

* Application е компонент в Elixir/Erlang, който може да бъде спиран и стартиран като едно цяло. |
* Може да бъде използван от други Apllication-и. |
* Един Application се грижи за едно supervision дърво и средата в която то върви. |

---
### Поведението Application
![Image-Absolute](assets/hullo.png)

---
#### start

* Извиква се при стартиране на Application-а. |
* Очаква се да стартира процеса-корен на програмата, обикновено това е root Supervisor. |

---
На start/2 се подават два аргумента.
* Първият обикновено е атома :normal, но при дистрибутирана програма би могъл да е {:takeover, node} или {:failover, node}. |
* Вторият са аргументи за програмата, които се задават при конфигурация. |

---
#### stop

* Когато Application-а бъде спрян, тази функция се извиква със състоянието върнато от start/2. |
* Използва се за изчистване на ресурси и има имплементация по подразбиране, която просто връща :ok. |

---
### Създаване и конфигурация на Application
![Image-Absolute](assets/creation-001.jpg)

---
* OTP Application поведението и логиката около него идват от Erlang/OTP.
* Application-ите се конфигурират със специален .app файл, написан на Erlang. |
* Той се слага при .beam файловете, които описва и след това може да се зареди на node, който има в пътя си директорията с него и тези .beam файлове. |

---
![Image-Absolute](assets/file.jpg)

---
```bash
mix new <app_project_name> --sup
```

---
### mix файл и конфигурация на един Application
![Image-Absolute](assets/mix.jpg)

---
## Край
![Image-Absolute](assets/end.jpg)
