## Архитектура браузера
Любой браузер состоит из следующих компонентов:
1. user interface
2. browser engine - соединительная часть между ui и механизмом рендеринга. Взаимодействует с движком рендера и управляет им
3. rendering engine (популярные движки webkit (chrome) и gecko (firefox)) - отвечает за обработку кода, написанного на html, css, js.
Строится DOM дерево, объектная модель CSS, определяется расположение элементов (стадии рендера)

Движок JavaScript предоставляет heap (кучу) - область в памяти, где хранятся объекты, массивы, функции и т.д., стек вызовов,
предоставляет работу с памятью (выделение, сборка мусора), и, главное - компиляция JS в машинный код

Последняя часть архитектуры браузера - Data Persistence (local storage, session storage, idb, websql, файловая система) - 
хранилище, которое использует браузер. Также обрабатывает cookie, вкладки браузера и т.п..

![google chrome browser architecture](/images/browser_architecture.png)

**Структура движка рендера webkit**

![webkit render engine architecture](/images/render_egnine_architecture.png)

## Event loop (цикл событий)
Первое и важное - event loop не является частью javascript, он предоставляется средой (например, браузером или node js). 
Отсюда следует, что устройство event loop может быть разное.

**В CHROMIUM ДВИЖОК V8, В NODE JS ДВИЖОК ТАКЖЕ V8, НО УСТРОЙСТВО ЦИКЛА СОБЫТИЙ - РАЗНОЕ!**

Event loop - механизм, который позволяет использовать неблокирующую модель ввода и вывода.

Есть стек вызовов - определённая структура данных, хранилище, которое можно представить в виде стопки бумаг.
Если мы хотим взять бумагу - мы берём её сверху стопки, если мы хотим добавить бумагу в стопку - также мы будем класть бумагу сверху.
Callstack аналогично примеру с бумагой складывает функции в определенном порядке, в том, в котором они должны быть вызваны.

У каждого браузера есть Web API - предоставляет функционал, например слушатели событий (нажатий на кнопки и т.п.), загрузки файлов, отправки
fetch запросов - это браузерный функционал.

**Пример №1**

![event loop example 1](/images/event-loop-example-1.png)

При вызове третьей функции, внутри неё вызывается вторая, внутри второй - первая. Первой будет выполнена первая функция - 
последняя попавшая в стек. Затем вторая и третья (аналогично стопке бумаге)

**Пример №2 - рекурсивный вызов функции**

![event loop example 2](/images/event-loop-example-2.png)

Хоть в этом примере и используется рекурсия, ничего не меняется - на изображении это хорошо видно.

### Распространённая проблема
Переполнение стека - одна из частых ошибок при работе с рекурсивными функциями (но и не только) - стоит понимать что
хранилище ограничено. На собеседованиях могут спрашивать как решать подобные проблемы (например, обход дерева, или решение подобной в примере задачи через цикл)

**Решение проблемы на примере функции факториала:**

![solution for recursion problem](/images/recursion-problem-solution.png)

### Но что не так?
Все примеры выше были синхронными - операции выполнялись друг за другом

### Task Queue (очередь задач)
Используется для хранения задач, которые не выполняются немедленно, но должны быть обработаны
**Задачи из очереди выполняются только после вызова всех функций из стека**

За очередь отвечает event loop, а за callstack отвечает движок javascript (V8 и т.д.) 

Task queue состоит из двух типов очередей:
- Macrotask queue (очередь событий (или макрозадач)) - они создаются:
  - таймерами (setTimeout, setInterval)
  - события (клик, загрузка изображения и т.д.)
  - браузерные нюансы (рендер, I/O и т.д.)
- Microtask queue (очередь задач (или микрозадач)) - они создаются:
  - Промисами (99%)
  - queueMicrotask
  - mutationObserver

У этих типов очередей есть приоритеты. Первыми будут выполнены задачи из microtask очереди. При этом, все задачи
из microtask queue должны быть выполненными перед выполнением macrotask.

Когда microtask были выполнены, начинают выполняться macrotask (одна за другой поочередно, а не одновременно).
То есть берётся одна macrotask, и, например если в ней содержится код на выполнение microtask, то цикл повторится.
Текущая macrotask будет ждать выполнения microtask'и для текущей macrotask'и, и после того как микро выполнится - макро 
продолжит свое выполнение, выполнится и только после этого вызовется следующая макро таска. 


## Движок Javascript
Он решает следующие задачи:
- Куча (heap) - и стек вызовов (call stack)
- Работа с памятью (выделение и сбор мусора)
- Компиляция JS в машинный код - **главная задача**
- Оптимизация (кеши, скрытые классы и прочее)

**Пример №3 - регистрация браузерных методов в web api**

![event loop example 3](/images/event-loop-example-3.png)

Разберём этот пример: таймаут попадает в коллстек, после этого он регистрируется в web api (и в этот момент запускается таймер),
после того как таймер иссяк - callback из этого setTimeout отправляется в task queue (очередь задач), и когда callstack очистился - 
выполняется console.log внутри (из коллбека)

**Пример №4 - слушатели событий**

![event loop example 4](/images/event-loop-example-4.png)

Разберём: функция addEventListener попадает в callstack, выполняется и происходит регистрация обработчика событий в web api, 
то же самое происходит со второй кнопкой. Теперь, при нажатии на кнопки (слушатели на клики которых установлены в web api),
коллбек будет попадать в task queue, ждать выполнения всех задач в callstack и затем передаваться туда (происходить обработка клика).

Благодаря этому в приложении одновременно можно иметь тысячи слушателей событий на кнопки, поля ввода, селекты и т.д. и в таком
асинхронном режиме они выполняются.

**Слушатели событий будут зарегистрированы до тех пор, пока явно не будут удалены!** 

Стоит также помнить! Что это не спецификация javascript, а браузерный функционал (web api)

### Render (отрисовка)
Стадии рендера:
- Построение DOM
- CSSOM (CSS Object Model)
- Render tree
- Style calculation - применение селекторов к элементам (чем больше вложены селекторы - тем больше ресурсов будет затрачено)
- Layout - по размерам и позиции расставляет элементы (грубо говоря чертеж, макет)
- Paint - рисует из чертежа пиксели (на основании дерева которое получено на предыдущем этапе, строится paint records)
- Compositing - работа со слоями

При изменении каких-то свойств/стилей все эти стадии отрабатывают заново, поэтому **рендер - дорогостоящая операция**

## Что вызывает рендер?
Следующие события вызывают рендер:
- изменение размера окна
- изменение шрифта
- изменение контента
- добавление/удаление классов/стилей
- манипуляция с DOM
- изменение ориентации (альбом/книга)
- вычисление размеров/позиции (в т.ч. например transform)


# Node JS
Это программная платформа. С помощью специальных инструментов позволяет превращать javascript в машинный код. Изначально 
javascript язык разработки браузерных приложений и некоторая функциональность там ограничена (например взаимодействовать с 
операционной системой мы не может, работа с файловой системой крайне ограничена). Node JS позволяет взаимодействовать с устройствами
ввода/вывода через свой специальный API (написанный на C++).

Node JS включает в себя две основополагающие: 
- Движок V8 (трансляция JS в машинный код)
- Libuv, который предоставляет:
  - Кроссплатформенный I/O (ввод/вывод)
  - Цикл событий (event loop), шаблон reactor который будет рассмотрен ниже

![thread scheduler](/images/thread_scheduler.png)

Например, мы захотели выполнить 5 криптографических операций. На изображение видно как они обрабатываются. 
4 потока забирают последовательно каждую из операций, а 5 операция начинает выполняться в первом освободившемся потоке.

С версии node.js 11.7.0 можно управлять потоками из кода с помощью модуля worker_threads.


## Демультиплексор событий
Представляет собой некоторый интерфейс уведомления о событиях. Его задача заключается в сборке и постановке в очередь
событий ввода и вывода, которые поступают из набора наблюдаемых ресурсов, а также блокировка появления новых доступных для обработки событий.

## Шаблон Reactor
По этому шаблону организована работа Node JS.
Состоит из:
- Демультиплексора событий
- Event loop
- Очередь событий (задача в очереди состоит из события и обработчика (функция обратного вызова))

### Рассмотрим как происходит обработка в шаблоне Reactor

![node-reactor-template](/images/node-reactor-template.png)


Последовательность:
1. Приложение создаёт новую операцию ввода или вывода, передав запрос демультиплексору событий. Также приложение должно 
определить обработчик (функцию обратного вызова). Обработчик будет вызван тогда, когда операция будет завершена. Самый важный момент,
что отправка нового запроса для демультиплексора событий не приводит к блокировке приложения - управление немедленно
возвращается к приложению
2. После завершения обработки набора операций ввода и вывода демультиплексор событий добавляет эти события в очередь
3. Цикл событий выполняет обход элементов в очереди событий, после чего для каждого события вызывается соответствующий обработчик - 
функция обратного вызова, которая была указана для того или иного события.
4. Обработка события
5. Может быть две ситуации:<br>
   **5a.** Обработчик, который является частью кода приложения возвращает управление циклу событий. То есть какое-то событие мы обработали и опять вернули управление циклу событий<br>
Во время выполнения этого обработчика могут запрашиваться какие-то новые асинхронные операции (например чтение информации из БД и теперь хотим записать её в файл), это приводит к:<br>
   **5b.** добавлению новых операций в демультиплексор событий. И вся схема повторяется по новой

После того как event loop обработал все элементы из очереди, цикл вновь заблокируется демультиплексором событий и вся процедура начнётся по новой, 
когда появится новый запрос на операцию ввода/вывода.

