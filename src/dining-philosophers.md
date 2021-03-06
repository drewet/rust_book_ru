% Обедающие философы

Для нашего второго проекта мы выбрали классическую задачу с параллелизмом. Она
называется "Обедающие философы". Задача была сформулирована в 1965 году
Эдсгером Дейкстрой, но мы будем использовать версию задачи,
[адаптированную][CSP] в 1985 году Ричардом Хоаром.

[CSP]: http://www.usingcsp.com/cspbook.pdf

>В древние времена, богатые филантропы пригласили погостить пятерых выдающихся
философов. Им выделили каждому по комнате, в которой они могли заниматься
своей профессиональной деятельностью - мышлением. Также была общая столовая,
где стоял большой круглый стол, а вокруг него пять стульев. Каждый стул имел
табличку с именем философа, который должен был сидеть на нем. Слева от каждого
философа лежала золотая вилка и в центре стола стояла большая миска со
спагетти, которая постоянно пополняется. Как подобает философам, они большую
часть своего времени проводят в раздумьях. Но однажды они почувствовали голод
и отправились в столовую. Каждый сел на свой стул, взял по вилке и воткнул её
в миску со спагетти. Но из-за запутанной сущности спагетти потребовалась
вторая вилка, которая бы отправляла спагетти в рот. Для этого требовалась бы
вилка справа от каждого философа. Философы положили свои вилки и встали из-за
стола, продолжая думать. Вилка может быть использована только одним философом
одновременно. Если другой философ захочет её взять, то ему придется ждать
когда она освободится.

Эта классическая задача показывает различные элементы параллелизма. Сложность
реализации задачи состоит в том, что простая реализация может зайти в
безвыходное состояние. Давайте рассмотрим простой пример решения этой проблемы:

1. Философ берет вилку в свою левую руку.
2. Затем берет вилку в свою правую руку.
3. Он ест.
4. Кладет вилки на место.

Теперь представим это как последовательность действий философов:

1. Философ 1 начинает выполнять алгоритм, берет вилку в левую руку.
2. Философ 2 начинает выполнять алгоритм, берет вилку в левую руку.
3. Философ 3 начинает выполнять алгоритм, берет вилку в левую руку.
4. Философ 4 начинает выполнять алгоритм, берет вилку в левую руку.
5. Философ 5 начинает выполнять алгоритм, берет вилку в левую руку.
6. ...? Все вилки заняты и никто не может начать есть! Безвыходное состояние.

Есть различные пути решения этой задачи. Мы в этом руководстве покажем свое
решение. Сначала давайте начнем с моделирования задачи. Начнем с философов:

```rust
struct Philosopher {
    name: String,
}

impl Philosopher {
    fn new(name: &str) -> Philosopher {
        Philosopher {
            name: name.to_string(),
        }
    }
}

fn main() {
    let p1 = Philosopher::new("Джудит Батлер");
    let p2 = Philosopher::new("Рая Дунаевская");
    let p3 = Philosopher::new("Зарубина Наталья");
    let p4 = Philosopher::new("Эмма Гольдман");
    let p5 = Philosopher::new("Шмидт Анна");
}
```

Здесь мы создаем [структуру][struct] представляющую философа. На данный
момент, нам нужно всего лишь имя. Мы выбрали тип [`String`][string] для
хранения имени вместо `&str`. Обычно проще работать с типом, владеющим
данными, чем с типом, использующим ссылки.

[struct]: structs.html
[string]: strings.html

Посмотрим на это:

```rust
impl Philosopher {
    fn new(name: &str) -> Philosopher {
        Philosopher {
            name: name.to_string(),
        }
    }
}
```

Этот блок `impl` позволяет объявлять что-нибудь для структуры `Philosopher`. В
нашем случае мы объявляем "статическую функцию" `new`. Первая строка этой
функции выглядит так:

```rust
fn new(name: &str) -> Philosopher {
```

Она принимает один аргумент `name` типа `&str`. Это ссылка на другую строку.
Возвращает новый экземпляр нашей структуры `Philosopher`.

```rust
Philosopher {
    name: name.to_string(),
}
```

Это создаст новый экземпляр `Philosopher` и присвоит полю `name` значение
переданного аргумента `name`. Но не сам аргумент, а результат его вызова
`.to_string()`. Это создаст копию строки нашего указателя `&str` и даст новый
экземпляр `String`, который будет присвоен полю `name` структуры `Philosopher`.

Почему бы сразу не передавать строку типа `String` напрямую? Так легче ее
вызывать. Если мы принимали бы тип `String`, а тот кто вызывает функцию имеет
ссылку на строку `&str`, то должен был бы приводить ее к типу `String` перед
каждым вызовом. Это уменьшит гибкость кода и придется _каждый раз_ делать
копию строки. Для этой небольшой программы это не очень важно, т.к. мы знаем
что будем использовать только короткие строки.

И последнее на что вы обратите внимание: мы просто объявляем структуру
`Philosopher` и кажется, что ничего больше не делаем. Rust это expression
ориентированный язык программирования, означающее, что каждое выражение
возвращает значение. Это применяется для функций у которых автоматически
возвращается последнее выражение. В нашем случае мы создаем структуру
`Philosopher` в последнем выражении функции, которое возвращается функцией.

Имя функции `new()` не является особенным в Rust, но негласным соглашением
принято так называть функции, которые возвращают новые экземпляры структур.
Давайте посмотрим на функцию `main()`:

```rust
fn main() {
    let p1 = Philosopher::new("Джудит Батлер");
    let p2 = Philosopher::new("Рая Дунаевская");
    let p3 = Philosopher::new("Зарубина Наталья");
    let p4 = Philosopher::new("Эмма Гольдман");
    let p5 = Philosopher::new("Шмидт Анна");
}
```

Здесь, мы связываем пять переменных с пятью новыми философами. Здесь указаны
имена некоторых известных философов, но вы можете указать любые другие. Если
бы мы _не объявили_ свою реализацию функции `new()`, то наш код выглядел бы
так:

```rust
fn main() {
    let p1 = Philosopher { name: "Джудит Батлер".to_string() };
    let p2 = Philosopher { name: "Рая Дунаевская".to_string() };
    let p3 = Philosopher { name: "Зарубина Наталья".to_string() };
    let p4 = Philosopher { name: "Эмма Гольдман".to_string() };
    let p5 = Philosopher { name: "Шмидт Анна".to_string() };
}
```

Это было бы очень не очень изящно и трудно читаемо. Использование
статической функции `new` имеет много плюсов, в нашем простом случае даст
более элегантный код.

Сейчас у нас уже есть каркас программы и теперь можно заняться решением задачи
с обедающими философами. Начнем с конца: сделаем так, чтобы философ сообщал
нам когда он закончит есть. Для этого потребуется метод, сообщающий нам об
окончании обеда, и цикл, запускающий этот метод для каждого философа.

```rust
struct Philosopher {
    name: String,
}   

impl Philosopher { 
    fn new(name: &str) -> Philosopher {
        Philosopher {
            name: name.to_string(),
        }
    }
    
    fn eat(&self) {
        println!("{} закончила обедать.", self.name);
    }
}

fn main() {
    let philosophers = vec![
        Philosopher::new("Джудит Батлер"),
        Philosopher::new("Рая Дунаевская"),
        Philosopher::new("Зарубина Наталья"),
        Philosopher::new("Эмма Гольдман"),
        Philosopher::new("Шмидт Анна"),
    ];

    for p in &philosophers {
        p.eat();
    }
}
```

Давайте сначала рассмотрим на `main()`. Вместо того чтобы каждого философа
привязывать к отдельной переменной, мы создаем `Vec<T>` c ними. `Vec<T>`
называют "вектор" и он является расширенной версией массива. Затем в цикле
[`for`][for] мы перебираем вектор, получая ссылку на каждого философа за
каждую итерацию.

[for]: for-loops.html

В теле цикла мы вызываем `p.eat()`, который объявлен выше:

```rust
fn eat(&self) {
    println!("{} закончила есть.", self.name);
}
```

В Rust методы явно получают параметр self. Вот почему `eat()` является
методом, а `new` статической функцией: `new()` не получает параметр `self`.
Для нашей первой версии `eat()`, мы только выводим имя философа и сообщение о
том, что он закончил есть. Запустив эту программу вы получите:

```text
Джудит Батлер закончила есть.
Рая Дунаевская закончила есть.
Зарубина Наталья закончила есть.
Эмма Гольдман закончила есть.
Шмидт Анна закончила есть.
```

Это было не сложно! Осталось чуть-чуть и приступим к самой задаче.

Дальше нам надо сделать так, чтобы философы не только заканчивали есть, но
также и начинали. Это новая версия программы:

```rust
use std::thread;

struct Philosopher {
    name: String,
}   

impl Philosopher { 
    fn new(name: &str) -> Philosopher {
        Philosopher {
            name: name.to_string(),
        }
    }
    
    fn eat(&self) {
        println!("{} начала есть.", self.name);

        thread::sleep_ms(1000);

        println!("{} закончила есть.", self.name);
    }
}

fn main() {
    let philosophers = vec![
        Philosopher::new("Джудит Батлер"),
        Philosopher::new("Рая Дунаевская"),
        Philosopher::new("Зарубина Наталья"),
        Philosopher::new("Эмма Гольдман"),
        Philosopher::new("Шмидт Анна"),
    ];

    for p in &philosophers {
        p.eat();
    }
}
```

Появилось немного изменений. Давайте посмотрим что изменилось:

```rust
use std::thread;
```

Конструкция `use` предоставляет доступ к области видимости модуля `thread` из
стандартной библиотеки, т.к. мы хотим использовать этот модуль далее в коде.

```rust
    fn eat(&self) {
        println!("{} начала есть.", self.name);

        thread::sleep_ms(1000);

        println!("{} закончила есть.", self.name);
    }
```

Здесь мы выводим на экран два сообщения и используем функцию `sleep_ms` между
выводом. Эта функция останавливает рабочий поток на 1000 миллисекунд и мы её
используем для симуляции процесса обеда философа.

Если вы запустите теперь программу, то увидите что теперь каждый философ по
очереди начинает есть и затем заканчивает:

```text
Джудит Батлер начала есть.
Джудит Батлер закончила есть.
Рая Дунаевская начала есть.
Рая Дунаевская закончила есть.
Зарубина Наталья начала есть.
Зарубина Наталья закончила есть.
Эмма Гольдман начала есть.
Эмма Гольдман закончила есть.
Шмидт Анна начала есть.
Шмидт Анна закончила есть.
```

Превосходно! Теперь у нас осталась проблема: наши философы едят по очереди, а
не одновременно, т.е. мы пока не решили задачу параллелизма.

Для того, чтобы наши философы начали есть одновременно, нам надо внести
немного изменений в код:

```rust
use std::thread;

struct Philosopher {
    name: String,
}   

impl Philosopher { 
    fn new(name: &str) -> Philosopher {
        Philosopher {
            name: name.to_string(),
        }
    }

    fn eat(&self) {
        println!("{} начала есть.", self.name);

        thread::sleep_ms(1000);

        println!("{} закончила есть.", self.name);
    }
}

fn main() {
    let philosophers = vec![
        Philosopher::new("Джудит Батлер"),
        Philosopher::new("Рая Дунаевская"),
        Philosopher::new("Зарубина Наталья"),
        Philosopher::new("Эмма Гольдман"),
        Philosopher::new("Шмидт Анна"),
    ];

    let handles: Vec<_> = philosophers.into_iter().map(|p| {
        thread::spawn(move || {
            p.eat();
        })
    }).collect();

    for h in handles {
        h.join().unwrap();
    }
}
```

Мы добавили еще один цикл в функцию `main()`. Теперь она выглядит так:

```rust
let handles: Vec<_> = philosophers.into_iter().map(|p| {
    thread::spawn(move || {
        p.eat();
    })
}).collect();
```

Тут добавились трудные к пониманию пять строк кода. Давайте разбираться.

```rust
let handles: Vec<_> = 
```

Объявляем новое связывание с именем `handles`. Мы дали ему такое имя, т.к.
собираемся создать несколько потоков и для контролирования их работы нужно
получить дескрипторы каждого из них. Нам надо явно указать здесь тип, а для
чего это надо мы скажем чуть позже. `_` это заполнитель типа. Мы говорим
компилятору "`handles` это вектор тип которого ты можешь самостоятельно
вычислить".

```rust
philosophers.into_iter().map(|p| {
```

Мы берем наш список философов и вызываем `into_iter()`. Это создаст итератор,
который заберёт право владения (ownership) для каждого философа. Нам надо
сделать это для передачи в поток. Мы берем этот итератор и вызываем `map`,
который принимает замыкание как аргумент и вызывает это замыкание для каждого
элемента итерации.

```rust
    thread::spawn(move || {
        p.eat();
    })
```

Вот здесь происходит сам параллелизм. Функция `thread::spawn` берет замыкание
в качестве аргумента и исполняет это замыкание в новом потоке. Это замыкание
нуждается дополнительном указании ключевого слова `move`, которое сообщает что
это замыкание получает право владения (ownership) значениями которое оно
захватывает. В данном случае переменную `p` функции `map`.

Внутри потока мы всего лишь вызываем функцию `eat()` у `p`.

```rust
}).collect();
```

И в завершении мы получаем результат этих вызовов `map` и собираем полученный
результат в в коллекцию с помощью функции `collect()`. Для того чтобы Rust
понял какой тип коллекции хотим получить, мы указали для `handle` тип
принимаемого значения `Vec<T>`. Элементы коллекции будут возвращаемые значения
типа `thread::spawn`, которые являются ручками их потоков. Вот так!

```rust
for h in handles {
    h.join().unwrap();
}
```

В конце функции `main()` мы в цикле перебираем каждую ручку и вызываем функцию
`join()` для каждой ручки, которая блокирует дальнейшее исполнение основного
потока, пока не завершится дочерний поток. Это позволяет нам быть уверенными,
что потоки завершат работу до того как произойдет выход их программы.

Если вы запустите эту программу, то вы увидите, что философы едят не дожидаясь
своей очереди! У нас многопоточность!

```text
Рая Дунаевская начала есть.
Рая Дунаевская закончила есть.
Эмма Гольдман начала есть.
Эмма Гольдман закончила есть.
Шмидт Анна начала есть.
Джудит Батлер начала есть.
Джудит Батлер закончила есть.
Зарубина Наталья начала есть.
Зарубина Наталья закончила есть.
Шмидт Анна закончила есть.
```

Но где же вилки? Они пока ещё не смоделированы у нас.

Давайте же начнем. Сначала сделаем новую структуру:

```rust
use std::sync::Mutex;

struct Table {
    forks: Vec<Mutex<()>>,
}
```

Структура `Table` имеет вектор мьютексов (`Mutex`). Мьютекс позволяет
управлять одновременно выполняющимися потоками, давая доступ к содержимому
только одному потоку. Это свойство нужно для реализации наших вилок. Мы в коде
используем пустой кортеж `()` внутри мьютекса, т.к. мы не собираемся
использовать значение, а мьютекс использовать только для приостановки потока.

Давайте изменим программу используя структуру `Table`:

```rust
use std::thread;
use std::sync::{Mutex, Arc};

struct Philosopher {
    name: String,
    left: usize,
    right: usize,
}

impl Philosopher {
    fn new(name: &str, left: usize, right: usize) -> Philosopher {
        Philosopher {
            name: name.to_string(),
            left: left,
            right: right,
        }
    }

    fn eat(&self, table: &Table) {
        let _left = table.forks[self.left].lock().unwrap();
        let _right = table.forks[self.right].lock().unwrap();

        println!("{} начала есть.", self.name);

        thread::sleep_ms(1000);

        println!("{} закончила есть.", self.name);
    }
}

struct Table {
    forks: Vec<Mutex<()>>,
}

fn main() {
    let table = Arc::new(Table { forks: vec![
        Mutex::new(()),
        Mutex::new(()),
        Mutex::new(()),
        Mutex::new(()),
        Mutex::new(()),
    ]});

    let philosophers = vec![
        Philosopher::new("Джудит Батлер", 0, 1),
        Philosopher::new("Рая Дунаевская", 1, 2),
        Philosopher::new("Зарубина Наталья", 2, 3),
        Philosopher::new("Эмма Гольдман", 3, 4),
        Philosopher::new("Шмидт Анна", 0, 4),
    ];

    let handles: Vec<_> = philosophers.into_iter().map(|p| {
        let table = table.clone();

        thread::spawn(move || {
            p.eat(&table);
        })
    }).collect();

    for h in handles {
        h.join().unwrap();
    }
}
```

Много изменений! Однако эта версия является корректно работающей версией.
Приступим к рассмотрению:

```rust
use std::sync::{Mutex, Arc};
```

Нам далее понадобится структура `Arc<T>` из модуля стандартной библиотеки
`std::sync`. Мы поговорим о ней чуть позже.

```rust
struct Philosopher {
    name: String,
    left: usize,
    right: usize,
}
```

Нам понадобилось добавить еще два поля в нашу структуру `Philosopher`. Каждый
философ должен иметь две вилки: одну для левой руки, другую для правой руки.
Мы используем тип `usize` для идентификации каждой вилки. Мы используем его
при при создании философа, передавая идентификаторы двух вилок. Эти два
значения будут использоваться полем `forks` структуры `Table`.

```rust
fn new(name: &str, left: usize, right: usize) -> Philosopher {
    Philosopher {
        name: name.to_string(),
        left: left,
        right: right,
    }
}
```

Используем функцию `new()` для задания значений `left` и `right`.

```rust
fn eat(&self, table: &Table) {
    let _left = table.forks[self.left].lock().unwrap();
    let _right = table.forks[self.right].lock().unwrap();

    println!("{} начала есть.", self.name);

    thread::sleep_ms(1000);

    println!("{} закончила есть.", self.name);
}
```

Здесь появились две новые строки, а также добавили один аргумент `table`. У
нас есть доступ к вилкам через структуру `Table`, передавая в нее
идентификаторы вилок `self.left` и `self.right`. Она хранит в себе `Mutex` для
каждой вилки и вызываем `lock()`, блокируя к ней доступ.

Вызов `lock()` может быть неудачным и если это случится, то нужно будет
аварийно завершить работу программы. Это может произойти, если вдруг поток
аварийно завершит работу, а мьютекс останется заблокирован. Такой мьютекс
называется ["отравленным"][poison]. Но в нашем случае это не может произойти,
то мы просто используем `unwarp()`.

[poison]: https://doc.rust-lang.org/stable/std/sync/struct.Mutex.html#poisoning

В этих двух строках мы результат сохраняем в `_left` и `_right`. Зачем мы
используем знаки подчеркивания в начале идентификаторов имен? Это для того
чтобы сказать компилятору, что хотим получить значения, которые далее
_не планируем использовать_. Таким образом Rust не будет выводить
предупреждение о неиспользуемых идентификаторов имен.

Когда же освободится мьютекс? Это произойдет когда `_left` и `_right` выйдут
из области видимости, т.е. по окончанию работы функции.

```rust
    let table = Arc::new(Table { forks: vec![
        Mutex::new(()),
        Mutex::new(()),
        Mutex::new(()),
        Mutex::new(()),
        Mutex::new(()),
    ]});
```

Далее в `main()` мы создаем `Table` и оборачиваем его в `Arc<T>`. Это
"атомарный счетчик ссылок" и нам он нужен для использования нашей структуры
`Table` получения доступа в нескольких потоках. Когда мы его будем передавать
в поток, то будет счетчик увеличиваться, а если поток завершит работу, то
счетчик уменьшится.

```rust
let philosophers = vec![
    Philosopher::new("Джудит Батлер", 0, 1),
    Philosopher::new("Рая Дунаевская", 1, 2),
    Philosopher::new("Зарубина Наталья", 2, 3),
    Philosopher::new("Эмма Гольдман", 3, 4),
    Philosopher::new("Шмидт Анна", 0, 4),
];
```

Здесь мы добавили наши значения `left` и `right` при создании структуры
`Philosopher`. Здесь есть _очень важная_ деталь на которую следует обратить
внимание. Посмотрите на последнюю строку создания `Philosopher`. У Анны Шмидт
должны бы иметь значения `4` и `0` в качестве аргументов, но у нас
имеют `0` и `4`. Это помешает нашей программе попасть в безвыходное состояние,
если все одновременно возьмут вилки одновременно. Так что давайте представим,
что один из философов у нас левша!

```rust
let handles: Vec<_> = philosophers.into_iter().map(|p| {
    let table = table.clone();

    thread::spawn(move || {
        p.eat(&table);
    })
}).collect();
```

Внутри нашего цикла  `map()`/`collect()` мы вызываем `table.clone()`. Метод
`clone()` структуры `Arc<T>` инкрементирует счетчик и когда покинет область
видимости декрементирует. Вы можете заметить, что мы делаем новое связывание с
`table` здесь, затеняя оригинальную `table`. Это позволяет нам не вводить
новый уникальный идентификатор.

Теперь наша программа работает! Только два философа могут обедать одновременно
и после запуска программы вы можете получить такой результат.

```text
Рая Дунаевская начала есть.
Эмма Гольдман начала есть.
Эмма Гольдман закончила есть.
Рая Дунаевская закончила есть.
Джудит Батлер начала есть.
Зарубина Наталья начала есть.
Джудит Батлер закончила есть.
Шмидт Анна начала есть.
Зарубина Наталья закончила есть.
Шмидт Анна закончила есть.
```

Поздравляем! Вы реализовали классическую задачу параллелизма на языке Rust.
