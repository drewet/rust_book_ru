% Многозадачность

Многозадачность и параллелизм являются невероятно важными проблемами в области
компьютерной науки. А также это актуальная тема для современной индустрии.
Компьютеры приобретают все больше и больше ядер, но многие программисты не
готовы в полной мере использовать это.

Средства Rust для безопасной работы с памятью в полной мере применимы и при
работе в многозадачной среде. Поэтому многозадачные программы на Rust должны
безопасно работать с памятью, и не создавать состояния гонки данных. Система
типов Rust способна справиться с этими задачами еще на этапе компиляции,
благодаря мощным средствам, которые она предоставляет.

Прежде чем мы поговорим об особенностях многозадачности, которые идут с Rust,
важно понять вот что: Rust является достаточно низкоуровневым, поэтому все это
предусмотрено в стандартной библиотеке, а не в самом языке. Это означает, что
если вам не нравится какой-то аспект в способе обработки многозадачности,
который использует Rust, вы всегда можете реализовать альтернативный способ.
[mio](https://github.com/carllerche/mio) представляет реальный пример этого
принципа в действии.

## Справочная информация: `Send` и `Sync`

О многозадачности рассуждать довольно трудно. В Rust, у нас есть система
строгой, статической типизации, чтобы помочь нам делать выводы о нашем коде. В
связи с этим Rust дает нам два трейта, помогающих нам разбираться в коде,
который, по всей вероятности, является многозадачным.

### `Send`

Первый трейт, о котором мы будем говорить, называется
[`Send`](../std/marker/trait.Send.html). Когда тип `T` реализует `Send`, это
указывает компилятору, что право владения переменными этого типа можно безопасно
перемещать между потоками.

Это важно для соблюдения некоторых ограничений. Например, если у нас есть канал,
соединяющий два потока, и мы хотели бы иметь возможность отправлять некоторые
данные по каналу из одного потока в другой. Следовательно, мы должны
гарантировать, что для отправляемого типа данных реализован трейт `Send`.

И наоборот, если бы у нас была библиотека, упакованная с помощью FFI, который не
является потокобезопасным, то нам не следовало бы реализовывать трейт `Send`,
благодаря чему компилятор поможет нам добиться невозможности покинуть текущий
поток.

### `Sync`

Второй из этих трейтов называется [`Sync`](../std/marker/trait.Sync.html). Когда
тип `T` реализует `Sync`, это указывает компилятору, что переменные этого типа
не имеют возможности использовать небезопасную память, когда они используются из
нескольких потоков одновременно.

Например, совместное использование неизменяемых данных с помощью атомарного
счетчика ссылок является потокобезопасным. Rust обеспечивает такой тип,
`Arc<T>`, и он реализует `Sync`, так что при помощи этого типа можно безопасно
обмениваться данными между потоками.

Эти два трейта позволяют использовать систему типов, чтобы обеспечить надежные
гарантии о свойствах вашего кода в условиях многозадачности. Прежде чем мы
продемонстрируем как, сначала мы должны узнать, как создать многозадачную
программу в Rust!

## Потоки

Стандартная библиотека Rust предоставляет библиотеку для потоков, которая
позволяет запускать Rust код параллельно. Вот простой пример использования
`std::thread`:

```
use std::thread;

fn main() {
    thread::spawn(|| {
        println!("Hello from a thread!");
    });
}
```

Метод `thread::spawn()` в качестве единственного аргумента принимает замыкание,
которое выполняется в новом потоке. Он возвращает дескриптор потока, который
может быть использован для ожидания завершения этого потока и извлечения его
результата:

```
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        "Hello from a thread!"
    });

    println!("{}", handle.join().unwrap());
}
```

Многие языки имеют возможность выполнять потоки, но это дико опасно. Есть целые
книги о том, как избежать ошибок, которые происходят от совместного
использования изменяемого состояния. В Rust снова помогает система типов,
которая предотвращает гонки данных на этапе компиляции. Давайте поговорим о том,
как же на самом деле обеспечивается совместное использование чего-либо в
условиях нескольких потоков.

## Безопасное совместное использование изменяемого состояния

Благодаря системе типов Rust, у нас есть понятие, которое звучит как ложь:
"безопасное совместное использование изменяемого состояния." Многие программисты
считают, что совместное использование изменяемого состояния - это очень, очень
плохо.

Кто-то однажды сказал это:

> Совместно используемое изменяемое состояние является корнем всех зол.
> Большинство языков пытаются решить эту проблему через 'изменяемое' часть, но
> Rust решает ее через 'совместно используемое' часть.

Та же самая [система владения](ownership.html), которая помогает предотвратить
неправильное использование указателей, также помогает исключить гонки данных,
один из худших видов ошибок многозадачности.

В качестве примера приведем программу на Rust, которая входила бы в состояние
гонки данных на многих языках. На Rust она не будет компилироваться:

```ignore
use std::thread;

fn main() {
    let mut data = vec![1u32, 2, 3];

    for i in 0..3 {
        thread::spawn(move || {
            data[i] += 1;
        });
    }

    thread::sleep_ms(50);
}
```

Она выдает ошибку:

```text
8:17 error: capture of moved value: `data`
        data[i] += 1;
        ^~~~
```

В данном случае мы знаем, что наш код _должен_ быть безопасным, но Rust в этом
не уверен. И, на самом деле, он не является безопасным: так как у нас есть
ссылка на `data` в каждом потоке, а поток становится владельцем ссылки, то у нас
есть три владельца! Это плохо. Мы можем исправить это с помощью типа `Arc<T>`,
который является атомарным указателем со счетчиком ссылок. 'атомарный' означает,
что им безопасно можно обмениваться между потоками.

`Arc<T>` предполагает наличие еще одного свойства у своего содержимого, чтобы
гарантировать, что его можно безопасно использовать из нескольких потоков: он
предполагает, что его содержимое реализует трейт `Sync`. В нашем случае мы также
хотим, чтобы была возможность изменять значение содержимого. Нам нужен тип,
который может обеспечить возможность изменения своего содержимого лишь одиним
пользователем единовременно. Для этого мы можем использовать тип `Mutex<T>`. Вот
вторая версия нашего кода. Она по-прежнему не работает, но по другой причине:

```ignore
use std::thread;
use std::sync::Mutex;

fn main() {
    let mut data = Mutex::new(vec![1u32, 2, 3]);

    for i in 0..3 {
        let data = data.lock().unwrap();
        thread::spawn(move || {
            data[i] += 1;
        });
    }

    thread::sleep_ms(50);
}
```

Вот ошибка:

```text
<anon>:9:9: 9:22 error: the trait `core::marker::Send` is not implemented for the type `std::sync::mutex::MutexGuard<'_, collections::vec::Vec<u32>>` [E0277]
<anon>:11         thread::spawn(move || {
                  ^~~~~~~~~~~~~
<anon>:9:9: 9:22 note: `std::sync::mutex::MutexGuard<'_, collections::vec::Vec<u32>>` cannot be sent between threads safely
<anon>:11         thread::spawn(move || {
                  ^~~~~~~~~~~~~
```

Вы можете видеть, что [`Mutex`](../std/sync/struct.Mutex.html) содержит метод
[`lock`](../std/sync/struct.Mutex.html#method.lock), который имеет следующую
сигнатуру:

```ignore
fn lock(&self) -> LockResult<MutexGuard<T>>
```

Так как трейт `Send` не был реализован для `MutexGuard<T>`, мы не можем
перемещать гварда через границы потоков, что и сказано в сообщении об ошибке.

Мы можем использовать `Arc<T>`, чтобы исправить это. Вот рабочая версия:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1u32, 2, 3]));

    for i in 0..3 {
        let data = data.clone();
        thread::spawn(move || {
            let mut data = data.lock().unwrap();
            data[i] += 1;
        });
    }

    thread::sleep_ms(50);
}
```

Теперь мы вызываем `clone()` для нашего `Arc`, что увеличивает внутренний
счетчик. Затем эта ручка перемещается в новый поток. Давайте более подробно
рассмотрим тело потока:

```rust
# use std::sync::{Arc, Mutex};
# use std::thread;
# fn main() {
#     let data = Arc::new(Mutex::new(vec![1u32, 2, 3]));
#     for i in 0..3 {
#         let data = data.clone();
thread::spawn(move || {
    let mut data = data.lock().unwrap();
    data[i] += 1;
});
#     }
#     thread::sleep_ms(50);
# }
```

Во-первых, мы вызываем метод `lock()`, который захватывает блокировку мьютекса.
Так как вызов данного метода может потерпеть неудачу, то он возвращает
`Result<T, E>`, но, поскольку это просто пример, мы используем `unwrap()`, чтобы
получить ссылку на данные. Реальный код должен иметь более надежную обработку
ошибок в такой ситуации. После этого мы свободно изменяем данные, так как у нас
есть блокировка.

Под конец мы запускаем короткий таймер, ожидающий некоторое время, отведенное
для выполнения потоков. Но такой вариант не является идеальным: возможно, мы
выбрали разумное время ожидания но, скорее всего, мы будем ждать либо больше чем
нужно, либо меньше чем необходимо, в зависимости от того, сколько на самом деле
времени потребуется потокам, чтобы закончить вычисления.

Более точной альтернативой использованию таймера было бы использование одного из
механизмов, предусмотренных стандартной библиотекой Rust для синхронизации
потоков друг с другом. Давайте поговорим об одном из них: каналах.

## Каналы

Вот версия нашего кода, которая использует каналы для синхронизации, вместо того
чтобы ждать в течение определенного времени:

```
use std::sync::{Arc, Mutex};
use std::thread;
use std::sync::mpsc;

fn main() {
    let data = Arc::new(Mutex::new(0u32));

    let (tx, rx) = mpsc::channel();

    for _ in 0..10 {
        let (data, tx) = (data.clone(), tx.clone());

        thread::spawn(move || {
            let mut data = data.lock().unwrap();
            *data += 1;

            tx.send(());
        });
    }

    for _ in 0..10 {
        rx.recv();
    }
}
```

Мы используем метод `mpsc::channel()`, чтобы построить новый канал. В этом
примере мы в каждом из десяти потоков вызываем метод `send`, который передает по
каналу простое значение `()`, а затем в главном потоке ждем, пока не будут
приняты все десять значений.

В то время как по этому каналу послается просто общий сигнал, в общем случае мы
можем отправить по каналу любые данные, которые реализуют трейт `Send`!

```
use std::thread;
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();

    for _ in 0..10 {
        let tx = tx.clone();

        thread::spawn(move || {
            let answer = 42u32;

            tx.send(answer);
        });
    }

   rx.recv().ok().expect("Could not receive answer");
}
```

`u32` реализует `Send` потому что мы можем сделать копию. Итак, создается поток,
в котором вычисляется ответ, а затем этот ответ с помощью метода `send()`
передается обратно по каналу.


## Паника

`panic!` аварийно завершает выполняемый в данный момент поток. Вы можете
использовать потоки Rust, как простой механизм изоляции:

```
use std::thread;

let result = thread::spawn(move || {
    panic!("oops!");
}).join();

assert!(result.is_err());
```

Представленный в коде выше `Thread` возвращает `Result`, что позволяет нам
проверить, произошло ли завершение потока в результате паники или нет.
