---
## OTP Поведения

---
![Image-Absolute](assets/title.png)

---
## Съдържание

1. Какво е OTP?
2. GenServer
3. Supervisor
4. Application

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
Ние ще разгледаме три абстракции от OTP:
* GenServer
* Supervisor
* Application

---
Ще разгледаме и базата ETS
* База данни в паметта
* Бърз и конкурентен достъп
* Работи се с tuple-и и Elixir/Erlang данни

---
## GenServer
![Image-Absolute](assets/generic.jpg)

---
* Имплементирахме си наша версия, но сега ще разгледаме OTP версията
* GenServer представлява процес.
* Представен е от модул, който предлага функции за различни, често срещани случаи при работа с процеси.

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
По този начин декларираме, че даден модул ще изпълни поведението GenServer.
Това поведение и логиката около него ни дават следните възможности:
* Да стартираме процес. |
* Да поддържаме състояние в този процес. |
* Да получаваме request-и и да връщаме отговори. |
* Да спираме процеса. |

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
* Можем да приемем init/1 за конструктор на GenServer процеса.
* Тя приема аргументи от какъвто и да е тип и връща състоянието на процеса. |
* За множество аргументи можем да използваме списък. |

---
* Когато GenServer.start_link/3 се извика с даден модул, ако той дефинира init/1, то тя ще се извика като конструктор.
* Поведението по-подразбиране, което се изпълнява, ако не дефинираме тази функция е, да се използват аргументите като състояние:

---
Ако върнем:
* {:ok, <състояние>}, процесът стартира с това състояние.
* {:ok, <състояние>, timeout-в-милисекунди}, процесът ще стартира със състоянието, и ако не получи съобщение за timeout време, ще получи автоматично съобщение :timeout. |
* {:ok, <състояние>, :hibernate}, процесът ще хибернира. |
* {:ok, state, {:continue, continue}}, процесът стартира със сътоянието, но веднага се извиква handle_continue/2 |

---
Ако върнем:
* :ignore, процесът ще излезе нормално и start_link ще върне :ignore. |
* {:stop, reason}, процесът ще излезе с върнатата причина и start_link/3 ще върне {:error, reason}. |
* Някой от {:ok, state} вариантите, GenServer.start_link/3 ще върне {:ok, pid}. |

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
* Функциите handle_call се извикват когато GenServer-ът получи съобщение, изпратено от GenServer.call/3.
* Функцията GenServer.call/3 изпраща съобщение до GenServer процес и чака за отговор. |
* Това е синхронна комуникация. |

---
GenServer.call/3
* Първият ѝ аргумент е pid или име на GenServer процес. |
* Вторият - съобщението, което трябва да се изпрати. |
* Tретият е timeout в милисекунди. |

---
* Ако върнем с {:reply, <отговор>, <състояние>}, ще върнем отговорът като резултат на GenServer.call\3 и ще продължим със състоянието, което връщаме.
* Можем да очакваме подобно на init/1 поведение ако timeout, :hibernate или {:continue, <continue>} са част от резултата.

---
* Ако отговорът е noreply, процесът извикал GenServer.call/3 няма да получи отговор и ще чака.
* Всеки процес може да отговори с GenServer.reply(from, reply).
* Нужно е само "from" да е точно тази наредена двойка, която е получена в handle_call.

---
Има три основни причини да върнем noreply от handle_call:
1. Защото сме отговорили с GenServer.reply/2 преди да върнем резултат. |
2. Защото ще отговори след като handle_call е свършила изпълнението си. |
3. Защото някой друг процес трябва да отговори. |

---
* Връщаме :stop резултат, когато искаме да прекратим изпълнението на GenServer процеса.
* Ще се извика terminate(reason, state) ако е дефинирана и процесът ще прекрати изпълнение с причина, дадената причина.

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
* handle_cast функциите се изпълняват при асинхронна комуникация.
* Това става чрез извикването на GenServer.cast(pid|name, request), която винаги връща :ok не и
чака за отговор.

---
* handle_cast най-често се използват за промяна на състоянието.

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
* Използват се за прихващане на всякакви други съобщения, да речем такива, пратени със send.
* Приемат и връщат аналогични параметри/резултати на тези на handle_cast/2.

---
#### handle_continue
![Image-Absolute](assets/proxy.duckduckgo.com.jpeg)

---
```elixir
@type new_state :: term()
@spec handle_continue(continue :: term(), state :: term()) ::
  {:noreply, new_state}
  | {:noreply, new_state, timeout() | :hibernate | {:continue, term()}}
  | {:stop, reason :: term(), new_state}
```

---
* Използва се за допълнителна инициализация или за допълнителна работа след handle*
* Поддържа се от OTP 21+
* Преди такива неща се постигаха със send_after или proc_lib

---
#### terminate
![Image-Absolute](assets/terminate.png)

---
```elixir
@type reason :: :normal | :shutdown | {:shutdown, term} | term
@spec terminate(reason, state :: term) :: term
```

---
* Извиква се преди терминиране на GenServer процес.
* Причината идва от резултат от типа {:stop, ...} върнат от handle_* функциите, |
* Може и да дойде от exit сигнал, ако GenServer процеса е системен процес. |
* Supervisor процеси могат да пращат EXIT съобщения, които да се предадат на тази функция. |

---
* Ще се извика ако има 'raise' в процеса.
* Ще се извика, ако процесът извика Kernel.exit/1 |
* Няма да се извика, ако процесът е убит с ':brutal_kill' |
* Не е гарантирано да се извика, затова чистене е по-добре да се прави от друг, свързан или наблюдаващ процес |

---
За дебъг, support и тестване : :sys
* :sys.get_state/1 и :sys.replace_state/2
* :sys.get_status/1 и :sys.statistics/2
* :sys.trace/2

---
## Supervisor
![Image-Absolute](assets/supervisor.jpg)

---
### Какво е Supervisor?
* Това е процес, чиято роля е да наглежда други процеси и да се грижи за тях.
* С помощта на Supervisor, по лесен начин можем да изградим fault tolerant система. |
* Идеологията около това поведение е лесна за възприемане. |
* Трудното е да направим добър дизайн на такава система. |

---
* Често сме споменавали за идеологията "Let it crash!".
* Тази мантра се базира върху следното:

---
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
В модула Supervisor има код, който може:
* Да инициализира и стартира Supervisor процес. |
* Да осигури, че Supervisor процесът прихваща EXIT сигнали. |
* Да стартира определен списък от процеси-деца, зададени на Supervisor-а и да ги link-не към него. |

---
Поведението Supervisor дава възможност:
* Ако някой от процесите-деца 'умре' непредвидено, Supervisor-ът ще получи сигнал и ще предприеме конкретно действие, зададено при имплементацията му. |
* Ако Supervisor-ът бъде терминиран, всичките му процеси-деца биват 'убити'. |

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
* Децата на Supervisor се задават (обикновено) от двойка : модул и аргументи.
* Този модул обикновено имплементира GenServer. |
* Можем да зададем и речник, следващ child_spec типа.

---
#### id

* Това е стойност която се ползва от Supervisor-ите вътрешно. |
* Може да се ползва за debugging да речем. |
* Ако процесът има име - това е името му. |
* По подразбиране, името е модула. Ако искаме да ползваме един и същ модул за много деца и не даваме име - трябва да я специфицираме |

---
#### start

* Тази стойност е tupple съдържащ MFA.
* Използва се за стартиране на процеса-дете. |
* По подразбиране, функцията за стартиране на процеса е :start_link. |

---
* Задължително тази функция трябва да стартира процес и да го свързва със процеса, който я е извикал. |
* Аргументите ще се подадат на зададената като атом функция при старт. |
* Тези аргументи се специфицират тук |

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
#### restart

* Атом, който указва кога и дали 'терминиран' процес-дете ще се рестартира. Възможните стойности са:

---
:permanent

* Процесът винаги се рестартира от Supervisor-а.
* Това е стойността по подразбиране на restart.
* Този начин на рестартиране е подходящ за дълго-живеещи процеси, които не трябва да 'умират'.

---
:transient

* С тази опция, даденият процес-дете няма да бъде рестартиран ако излезе нормално - с exit(:normal) или просто завърши изпълнение.
* Ако обаче излезе с друга причина (exit(:reason)), ще бъде рестартиран.

---
:temporary

* Процесът-дете няма да бъде рестартиран ако 'умре'.
* Няма значение дали е излязъл с грешка или не.
* Подходяща е за кратко-живеещи процеси за които е очаквано, че могат да 'умрат' с грешка и няма много код зависещ от тях.

---
#### shutdown

* Когато Supervisor трябва да убие някои или всички свои процеси-деца, той извиква Process.exit(child_pid, :shutdown) за всеки от тях.
* Стойността зададена като shutdown се използва за timeout след като това се случи.
* По подразбиране е 5000 или пет секунди.

---
* Когато процес получи :shutdown, ако е Genserver, ще му се извика terminate/2 функцията.
* Изчистването на някакви ресурси може да отнеме време. |
* Ако това време е повече от зададеното в shutdown, Supervisor-ът ще изпрати нов EXIT сигнал с Process.exit(child_pid, :kill). |

---
* Ако зададем стойност :brutal_kill за shutdown, Supervisor-ът винаги ще терминира даденият процес направо с Process.exit(child_pid, :kill).
* Можем да зададем и :infinity за да оставим процеса да си излезе спокойно.

---
#### type

* Това свойство определя дали процесът дете е worker процес или Supervisor.

---
#### modules

* Трябва да е списък от един елемент - модул.
* Това е модулът съдържащ callback функциите на GenServer имплементация или на Supervisor имплементация.

---
### Функцията child_spec/2
* По лесен начин за задаване на спецификация
* За да не задаваме map, можем да ползваме нея

---
### Наредена двойка за спецификация
```elixir
children = [
  {MyModule, [<args>]}
]
```

---
* Ще се ползват дефаултните стойности за спецификация
* Модулът ще е MyModule и трябва да дефинира start_link/1, на която ще се подадат аргументите

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
### Supervisor - друг начин за направа

---
```elixir
def start_supervising! do
  children =
  [
    {SomeModule, []}
    {SomeOtherModule, [arg1, arg2]}
    {SomeModuleUsingSupervisor, []}
  ]

  opts = [strategy: :one_for_one, name: :my_supervisor]
  Supervisor.start_link(children, opts)

end
```

---
### Функциите на Supervisor

![Image-Absolute](assets/function.jpg)

---
#### Supervisor.start_child

* Динамично добавя нова спецификация към Supervisor и стартира процес за нея.
* Първият аргумент е pid на Supervisor процес а вторият - валиден child_spec.

---
#### Supervisor.count_children

```elixir
SomeSupervisor |> Supervisor.count_children
# %{active: 3, specs: 3, supervisors: 0, workers: 3}
```

---
* active - това е броят на всички активни процеси-деца, които се управляват от подадения Supervisor. |
* specs - това е броят на всички процеси-деца, няма значение дали са 'живи' или 'мъртви'. |

---
* supervisors - броят на всички активни процеси-деца, които се управляват от подадения Supervisor и са Supervisor-и на свой ред. Няма значение дали са активни или не. |
* workers - това е броят на всички активни процеси-деца, които се управляват от подадения Supervisor и не са Supervisor-и. Няма значение дали са активни или не. |

---
#### Supervisor.which_children

* Връща списък с информация за всичките процеси-деца на Supervisor.

---
#### Supervisor.terminate_child

* Може да 'убие' процес-дете на Supervisor, подаден като първи аргумент.

---
#### Supervisor.restart_child

* Рестартира процес-дете, чиято спецификация се пази в подаденият като първи аргумент Supervisor.

---
#### Supervisor.delete_child

* Изтрива спецификация за дадено child_id.

---
#### Supervisor.stop

* Спира подаденият като първи аргумент Supervisor с подадена като втори аргумент причина и изчаква с timeout - трети аргумент.

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
* Добър е когато можем да имаме множество подобни процеси, които се създават динамично.
* Да речем за изграждане на pool от процеси |
* Или за процес на нов request към нашата логика |
* Пример е Task.Supervisor (ще покажем пример с него) |

---
### Task.Supervisor

* Понякога искаме задачите да изпращат резултат, но да не се link-ват към текущия процес. |
* Да речем това са задачи, които си говорят с отдалечен компонент/service и биха могли да получат грешка отвън. |

---
* Бихме могли да стартираме специален DynamicSupervisor като част от нашия Application, който да отговаря за тези задачи.
* Този Supervisor ще създава задачи, които могат и да излязат с грешка, но няма да убият процеса, който ги използва. |
* Даже ще връщат резултат ако са създадени с правилната функция. |

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
* Винаги, когато виртуалната машина стартира, специален процес, наречен 'application_controller' се стартира с нея.
* Този процес стои над всички Application-и, които вървят в тази виртуална машина. |
* Mожем да го наречем Supervisor на Application-ите. |

---
* Можем да приемем application_controller процеса за коренът на дървото от процеси в един BEAM node.
* Под този 'корен' стоят различните Application-и, които са абстракция, обвиваща supervision дърво, която може да се стартира и спира като едно цяло. |
* Те са като мега-процеси, управлявани от application_controller-a. |
* Отвън те изглеждат като един процес за който имаме функции start и stop. |

---
* Когато се стартира един Application се създават два специални процеса, които заедно са наречени 'application master'.
* Тези два процеса създават Application-а и стоят между application_controller процеса и Supervisor-а служещ за корен на supervision дървото. |

---
### Пример : Blogit

---
![Image-Absolute](assets/blogit.png)

---
### Поведението Application
![Image-Absolute](assets/hullo.png)

---
#### start

* Извиква се при стартиране на Application-а. |
* Очаква се да стартира процеса-корен на програмата, обикновено това е root Supervisor. |

---
* Очаква се да върне {:ok, pid}, {:ok, pid, state} или {:error, reason}.
* Този state може да е каквото и да е, може да бъде пропуснат и ще е []. |
* Той се подава на stop/1 callback функцията при спиране. |

---
На start/2 се подават два аргумента.
* Първият обикновено е атома :normal, но при дистрибутирана програма би могъл да е {:takeover, node} или {:failover, node}. |
* Вторият са аргументи за програмата, които се задават при конфигурация. |

---
#### stop

* Когато Application-а бъде спрян, тази функция се извиква със състоянието върнато от start/2. |
* Използва се за изчистване на ресурси и има имплементация по подразбиране, която просто връща :ok. |

---
### Функциите на модула Application

---
#### Application.load

* Зарежда Application в паметта. |
* Зарежда environment данните му и други свързани Application-и. |
* Не го стартира. |

---
#### Application.start

* Стартира Application. |
* За първи аргумент взима атом, идентифициращ Application. |
* За втори типа на Application-а. |

---
Типът на програмата може да бъде: :permanent
* Ако Application-ът умре, всички други Application-и на node-а също умират. |
* Няма значение дали Application-а е завършил нормално или не. |

---
Типът на програмата може да бъде: :transient
* Ако Application-ът умре с :normal причина, ще видим report за това, но другите Application-и на node-а няма да бъдат терминирани. |
* Ако причината обаче е друга, всички други Application-и и целия node ще бъдат спрени. |

---
Типът на програмата може да бъде: :temporary
* Това е типът по подразбиране. |
* С каквато и причина да спре един Application, другите ще продължат изпълнение. |

---
* Ако спрем ръчно Application с функцията Application.stop/1, тези стратегии няма да се задействат.


---
#### Application.ensure_all_started

* Прави същото като Application.start/2 и взима същия тип аргументи, но също стартира всички други Application-и, конфигурирани като зависимости.

---
#### Application.get_application

* Взима модул, който представлява Application, тоест има "use Application". |
* Връща атом представляващ Application. |
* Няма значение дали този Application е активен или не. |
* Важно е да е специфициран. |

---
#### Функции за четене и писане на Application environment

* Application environment е keyword list, който се конфигурира при дефиниране на Application. |
* Има няколко функции за четене от него. |

---
#### Application.spec

Тази функция има две версии. |
* Първата взима само Application и връща цялата му спецификация. |
* Втората взима Application и ключ в спецификацията, за да върне част от нея. |

---
#### Application.stop

* Спира Application. Без да задейства стратегията му. Application-ът остава зареден в паметта.

---
#### Application.unload

* Премахва от паметта спрян Application и неговите зависимости, зададени като included_applications.

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
### Как стартираме един Application?

```bash
elixir -S mix run
```


---
## Край
![Image-Absolute](assets/end.jpg)
