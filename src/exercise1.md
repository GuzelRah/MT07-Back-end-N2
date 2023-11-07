# Алгоритмы и методы балансировки
Существует много различных алгоритмов и методов балансировки нагрузки. Выбирая конкретный алгоритм, нужно исходить, во-первых, из специфики конкретного проекта, а во-вторых — из целей. которые мы планируем достичь.

#### Цели  балансировки:

1. справедливость: нужно гарантировать, чтобы на обработку каждого запроса выделялись системные ресурсы и не допустить возникновения ситуаций, когда один запрос обрабатывается, а все остальные ждут своей очереди;  
2. эффективность: все серверы, которые обрабатывают запросы, должны быть заняты на 100%; желательно не допускать ситуации, когда один из серверов простаивает в ожидании запросов на обработку (сразу же оговоримся, что в реальной практике эта цель достигается далеко не всегда);    
3. сокращение времени выполнения запроса: нужно обеспечить минимальное время между началом обработки запроса (или его постановкой в очередь на обработку) и его завершения;
4. сокращение времени отклика: нужно минимизировать время ответа на запрос пользователя.  

#### Cвойства балансировки:

1. предсказуемость: нужно чётко понимать, в каких ситуациях и при каких нагрузках алгоритм будет эффективным для решения поставленных задач;  
2. равномерная загрузка ресурсов системы;  
3. масштабирумость: алгоритм должен сохранять работоспособность при увеличении нагрузки.

## Round Robin
Round Robin, или алгоритм кругового обслуживания, представляет собой перебор по круговому циклу: первый запрос передаётся одному серверу, затем следующий запрос передаётся другому и так до достижения последнего сервера, а затем всё начинается сначала.

Самой распространёной имплементацией этого алгоритма является, конечно же, метод балансировки __Round Robin DNS__. Как известно, любой DNS-сервер хранит пару «имя хоста — IP-адрес» для каждой машины в определённом домене. Этот список может выглядеть, например, так:

example.com	xxx.xxx.xxx.2  
www.example.com	xxx.xxx.xxx.3  
 
С каждым именем из списка можно ассоциировать несколько IP-адресов:   

example.com	xxx.xxx.xxx.2  
www.example.com	xxx.xxx.xxx.3  
www.example.com	xxx.xxx.xxx.4  
www.example.com	xxx.xxx.xxx.5  
www.example.com	xxx.xxx.xxx.6
  
DNS-сервер проходит по всем записям таблицы и отдаёт на каждый новый запрос следующий IP-адрес: например, на первый запрос — xxx.xxx.xxx.2, на второй — ххх.ххх.ххх.3, и так далее. В результате все серверы в кластере получают одинаковое количество запросов.

### Плюсы 
1.  Независимость от протокола высокого уровня. Для работы по алгоритму Round Robin используется любой протокол, в котором обращение к серверу идёт по имени.
2. Никак не зависит от нагрузки на сервер: кэширующие DNS-серверы помогут справиться с любым наплывом клиентов.
3. Не требует связи между серверами, поэтому он может использоваться как для локальной, так и для глобальной балансировки.
4. Решения на базе алгоритма Round Robin отличаются низкой стоимостью: чтобы они начали работать, достаточно просто добавить несколько записей в DNS.

### Минусы
 1. Чтобы распределение нагрузки по этому алгоритму отвечало упомянутым выше критериями справедливости и эффективности, нужно, чтобы у каждого сервера был в наличии одинаковый набор ресурсов. При выполнении всех операций также должно быть задействовано одинаковое количество ресурсов. В реальной практике эти условия в большинстве случаев оказываются невыполнимыми.
2. Также при балансировке по алгоритму Round Robin совершенно не учитывается загруженность того или иного сервера в составе кластера. Представим себе следующую гипотетическую ситуацию: один из узлов загружен на 100%, в то время как другие — всего на 10 — 15%. Алгоритм Round Robin возможности возникновения такой ситуации не учитывает в принципе, поэтому перегруженный узел все равно будет получать запросы. Ни о какой справедливости, эффективности и предсказуемости в таком случае не может быть и речи.
3. совершенно не учитывается количество активных на данный момент подключений.

В силу описанных выше обстоятельств сфера применения алгоритма Round Robin весьма ограничена.

#### Weighted Round Robin
Это — усовершенствованная версия алгоритма Round Robin.  
 Суть усовершенствований заключается в следующем: каждому серверу присваивается весовой коэффициент в соответствии с его производительностью и мощностью. Это помогает распределять нагрузку более гибко: серверы с большим весом обрабатывают больше запросов. Однако всех проблем с отказоустойчивостью это отнюдь не решает. Более эффективную балансировку обеспечивают другие методы, в которых при планировании и распределении нагрузки учитывается большее количество параметров.

# Least Connections (сокращённо — leastconn)
Суть этого алгоритма проста: каждый последующий запрос направляется на сервер с наименьшим количеством поддерживаемых подключений. Least Connections — это изящное и эффективное решение, которое позволяет адекватно распределять нагрузку по серверам с приблизительно одинаковыми параметрами.

![Least Connections](https://upload.wikimedia.org/wikipedia/commons/5/5b/Least_Connection.png?20150924180608)  


 Он учитывает количество подключений, поддерживаемых серверами в текущий момент времени. Каждый следующий вопрос передаётся серверу с наименьшим количеством активных подключений.

#### Weighted Least Connections
Существует усовершенствованный вариант алгоритма Least Connections, предназначенный в первую очередь для использования в кластерах, состоящих из серверов с разными техническими характеристиками и разной производительностью. Он учитывает при распределении нагрузки не только количество активных подключений, но и весовой коэффициент серверов.


#### Locality-Based Least Connection Scheduling  
Первый метод был создан специально для кэширующих прокси-серверов. Его суть заключается в следующем: наибольшее количество запросов передаётся серверам с наименьшим количеством активных подключений. За каждым из клиентских серверов закрепляется группа клиентских IP. Запросы с этих IP направляются на «родной» сервер, если он не загружен полностью. В противном случае запрос будет перенаправлен на другой сервер (он должен быть загружен менее чем наполовину).

#### Locality-Based Least Connection Scheduling with Replication Scheduling 
В алгоритме каждый IP-адрес или группа IP-адресов закрепляется не за отдельным сервером, а за целой группой серверов. Запрос передаётся наименее загруженному серверу из группы. Если же все серверы из «родной» группы перегружены, то будет зарезервирован новый сервер. Этот новый сервер будет добавлен к группе, обслуживающей IP, с которого был отправлен запрос. В свою очередь наиболее загруженный сервер из этой группы будет удалён — это позволяет избежать избыточной репликации.

#### Destination Hash Scheduling
Алгоритм   был создан для работы с кластером кэширующих прокси-серверов, но он часто используется и в других случаях. В этом алгоритме сервер, обрабатывающий запрос, выбирается из статической таблицы по IP-адресу получателя.
### Source Hash Scheduling
Алгоритм  основывается на тех же самых принципах, что и предыдущий, только сервер, который будет обрабатывать запрос, выбирается из таблицы по IP-адресу отправителя.

# Sticky Sessions
__Sticky Sessions__ — это метод, который позволяет балансировщику нагрузки идентифицировать запросы, поступающие от одного и того же клиента, и всегда отправлять эти запросы на один и тот же сервер. В липких сессиях вся пользовательская информация хранится на стороне сервера, и этот метод обычно используется в службах с отслеживанием состояния. Эта функция в основном доступна для балансировщиков нагрузки HTTP. 
![Sticky Sessions](https://traefik.io/static/0cb5dd4290ca488cff5b3d10e389978b/Diagram--1-.jpg)



__Недостатки__:

- Серверы будут загружены неравномерно. Некоторые пользователи проведут на   сайте больше времени, чем другие.
- Балансировщик нагрузки не станет читать и обрабатывать запрос, это не его работа. Определить, откуда поступает запрос, он может только по IP-адресу.


## Хеш (Hash)

Такой способ использует в своей основе механизм хеширования. Он позволяет распределить запросы на основе хеша, для которого обычно используется IP адрес или URL. В таком случае запросы от одного и того же IP будут отправлены на один и тот же сервер. Тоже самое касается URL. Такой алгоритм обычно используют, когда сервер хранит какие-то локальные данные, которые нужны для ответа.  
Этот алгоритм гарантирует, что один и тот же клиент будет всегда направлен на один и тот же сервер.

**Преимущества:**

- Привязывает клиента к определенному IP-адресу (URL)
- Обеспечивает сохранение состояния сеанса клиента на одном сервере.

**Недостатки:**

- Не учитывает текущую загрузку приложений.
- Не смотрит на производительность серверов.
- Может привести к неравномерному распределению нагрузки при большом количестве клиентов с одинаковым IP-адресом (URL)