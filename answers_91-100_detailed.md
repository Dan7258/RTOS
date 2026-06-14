# Ответы на вопросы 91–100 (подробная версия)

Эта версия рассчитана на читателя, который встречает темы впервые. Каждый ответ начинается с мотивации, разбирается механизм по шагам, с примерами и контекстом.

---

## 91. Выделение памяти kmalloc(). Флаги gfp_mask и модификаторы операций, зон. Модификаторы типов. __GFP_WAIT. GFP_NOIO. GFP_ATOMIC.
*Источник: robert_lav.md, Глава 12 «Управление памятью»*

### Контекст

В вопросе 90 рассмотрены различия `malloc`, `kmalloc`, `vmalloc`. Здесь — подробно о флагах `kmalloc`.

### Зачем нужны разные флаги

Память в ядре приходится выделять в разных контекстах:
- **Обычный код** (системный вызов) — можно подождать.
- **Обработчик прерывания** — ждать нельзя.
- **Спинлок удерживается** — ждать нельзя.
- **Внутри кода swap** — нельзя инициировать I/O.
- **Внутри FS** — нельзя обращаться к FS.
- **Для DMA** — нужна нижняя память.

Разные ситуации → разные ограничения на аллокатор. Флаги — способ их выразить.

### Структура флагов

`gfp_t` (Get Free Page type) — битовая маска. Делится на:

**1. Модификаторы операций** (что аллокатор может делать).
**2. Модификаторы зон** (из какой памяти выделять).
**3. Модификаторы типа** (готовые комбинации).

### Модификаторы операций

Из robert_lav.md (Глава 12):

| Флаг | Назначение |
|---|---|
| `__GFP_WAIT` | Может блокироваться |
| `__GFP_HIGH` | Высокий приоритет |
| `__GFP_IO` | Может инициировать I/O |
| `__GFP_FS` | Может вызывать операции FS |
| `__GFP_COLD` | «Холодную» страницу |
| `__GFP_NOWARN` | Не печатать warning |
| `__GFP_REPEAT` | Повторять при неудаче |
| `__GFP_NOFAIL` | Не возвращать ошибку |
| `__GFP_NORETRY` | Не повторять |
| `__GFP_NO_GROW` | Не растить slab |
| `__GFP_COMP` | Compound page |
| `__GFP_ZERO` | Обнулить |

### __GFP_WAIT — самый важный

Из robert_lav.md:
> «`__GFP_WAIT` — означает, что распределитель памяти может переходить в состояние ожидания».

Если установлен:
- Аллокатор может **спать**.
- Может ждать освобождения памяти через swap.
- Подходит для любого «обычного» контекста.

Если НЕ установлен:
- Аллокатор должен вернуть результат **немедленно**.
- Либо найти память сразу, либо вернуть NULL.
- Используется в IRQ-контексте, под спинлоками.

### __GFP_IO

«Может инициировать I/O.» Например, swap out страниц для освобождения памяти.

Если НЕ установлен (как в `GFP_NOIO`):
- Можно блокироваться, но без инициации I/O.
- Полезно в коде I/O subsystem — иначе деадлок.

### __GFP_FS

«Может вызывать операции файловой системы.»

Если НЕ установлен (как в `GFP_NOFS`):
- Можно I/O, но без FS-операций.
- Полезно в коде самой ФС — иначе рекурсия и потенциальный deadlock.

### Модификаторы зон

В Linux память делится на **зоны**:
- **ZONE_DMA** — нижние 16 МБ (для старых ISA-DMA).
- **ZONE_DMA32** — нижние 4 ГБ (для 32-битных DMA-устройств).
- **ZONE_NORMAL** — обычная (от 16 МБ до 896 МБ на x86).
- **ZONE_HIGHMEM** — высокая (выше 896 МБ на 32-бит x86).

Зачем зоны:
- Не все устройства могут адресовать всю память.
- Не вся память может быть постоянно отображена в АП ядра (на 32-бит).

### Модификаторы типов — готовые комбинации

Из robert_lav.md:

**GFP_ATOMIC = `__GFP_HIGH`**

- Высокий приоритет.
- НЕ блокируется.
- Для IRQ-контекста, под спинлоками.
- Может вернуть NULL чаще.

**GFP_NOWAIT = 0**

- Как ATOMIC, но без особого приоритета.

**GFP_NOIO = `__GFP_WAIT`**

- Может ждать.
- НЕ инициирует I/O.
- Для блочного I/O-кода.

**GFP_NOFS = `__GFP_WAIT | __GFP_IO`**

- Может I/O.
- НЕ вызывает FS.
- Для FS-кода.

**GFP_KERNEL = `__GFP_WAIT | __GFP_IO | __GFP_FS`**

- Всё разрешено.
- Стандартный флаг для обычного кода ядра.

**GFP_USER = аналогично KERNEL** (но с пометкой для пользователя).

**GFP_HIGHUSER = `GFP_USER | __GFP_HIGHMEM`** — для аллокаций пользовательского АП.

**GFP_DMA = `__GFP_DMA`** — DMA-совместимая память.

### Когда что использовать

```c
// Обычный код
ptr = kmalloc(size, GFP_KERNEL);

// В обработчике прерывания
ptr = kmalloc(size, GFP_ATOMIC);

// Под спинлоком
spin_lock(&lock);
ptr = kmalloc(size, GFP_ATOMIC);
spin_unlock(&lock);

// В коде блочного I/O
ptr = kmalloc(size, GFP_NOIO);

// В коде ФС
ptr = kmalloc(size, GFP_NOFS);

// DMA-буфер
ptr = kmalloc(size, GFP_DMA | GFP_KERNEL);
```

### Что произойдёт при ошибке

`GFP_KERNEL` редко возвращает NULL — ядро будет ждать, пока память не освободится.

`GFP_ATOMIC` чаще возвращает NULL — нельзя ждать, и если памяти нет в резервах — ошибка.

Из-за этого: после `kmalloc(size, GFP_ATOMIC)` обязательно проверять NULL.

### Комбинирование

Флаги можно комбинировать `|`:
```c
ptr = kmalloc(size, GFP_KERNEL | __GFP_ZERO);
// эквивалент kzalloc(size, GFP_KERNEL)
```

### Резюме

- gfp-флаги управляют поведением аллокатора.
- Модификаторы операций: что можно делать (`__GFP_WAIT`, `__GFP_IO`, `__GFP_FS`).
- Модификаторы зон: откуда брать (`__GFP_DMA`, `__GFP_HIGHMEM`).
- Модификаторы типов: готовые комбинации (`GFP_KERNEL`, `GFP_ATOMIC`, `GFP_NOIO`, `GFP_NOFS`).
- Использовать `GFP_KERNEL` в обычном коде, `GFP_ATOMIC` в IRQ, `GFP_NOIO/NOFS` в подсистемах I/O/FS.

---

## 92. Заготовленные кэши. Использование кэшей в scull. void *kmem_cache_alloc (kmem_cache_t *cache, int flags); scullc_cache = kmem_cache_create("scullc", scullc_quantum, 0, SLAB_HWCACHE_ALIGN, NULL, NULL);
*Источник: robert_lav.md, Глава 12*

### Зачем заготовленные кэши

Из robert_lav.md (Глава 12):
> «Выделение и освобождение памяти, занимаемой структурами данных, — одна из самых частых операций, которые выполняются в любом ядре».

Часто-используемые структуры:
- `task_struct` (процессы).
- `inode`, `dentry` (файловая система).
- `vm_area_struct` (память).
- Драйверные структуры.

Если каждый раз делать `kmalloc(sizeof(struct task_struct), GFP_KERNEL)`:
- Поиск свободного блока.
- Возможная фрагментация (структуры разных размеров вперемешку).
- Полная переинициализация при каждом выделении.

Решение — **кэширование структур одного размера**.

### Идея slab-аллокатора

Из robert_lav.md:
> «Программисты обычно ведут списки свободных ресурсов (free list). В список свободных ресурсов заносится некоторый набор уже выделенных структур данных, готовых к использованию».

Раньше каждая подсистема ядра вела свой список. Это:
- Дублирование кода.
- Нет централизованного управления (нельзя сократить кэши при нехватке памяти).

Решение — **slab allocator**: централизованный механизм кэширования.

### История

Из robert_lav.md:
> «Концепции блочного распределения памяти впервые были реализованы в операционной системе SunOS 5.4 фирмы Sun Microsystems. Для уровня кеширования структур данных в операционной системе Linux используется такое же название и похожие особенности реализации».

### Цели slab

Из robert_lav.md:
1. **Кэширование часто-используемых структур.**
2. **Уменьшение фрагментации.**
3. **Уменьшение времени инициализации** — освобождённый объект «помнит» свою инициализацию.
4. **Хорошее использование процессорного кэша.**

### Создание кэша

```c
kmem_cache_t *kmem_cache_create(const char *name,
                                 size_t size,
                                 size_t align,
                                 unsigned long flags,
                                 void (*ctor)(void *),
                                 void (*dtor)(void *));
```

(`dtor` устарел в новых версиях.)

Параметры:
- **`name`** — имя для отладки (виден в `/proc/slabinfo`).
- **`size`** — размер одного объекта в кэше.
- **`align`** — выравнивание (0 = по умолчанию, обычно sizeof(void*)).
- **`flags`** — флаги создания.
- **`ctor`** — конструктор, вызываемый для нового объекта (или NULL).
- **`dtor`** — деструктор (обычно NULL).

### Флаги создания

| Флаг | Назначение |
|---|---|
| `SLAB_HWCACHE_ALIGN` | Выравнивать по линии кэша CPU (обычно 64 байта) |
| `SLAB_POISON` | Заполнять «мусором» при отладке |
| `SLAB_RED_ZONE` | Защитные зоны для детекта переполнения |
| `SLAB_PANIC` | Падать в panic при ошибке создания |
| `SLAB_CACHE_DMA` | Из DMA-зоны |

### SLAB_HWCACHE_ALIGN — зачем

Современные процессоры имеют линии кэша 64 байта. Если объекты выровнены, доступ эффективнее:
- Один объект полностью помещается в линию.
- Нет false sharing между объектами.

Цена: некоторое расходование памяти на padding.

### Использование в scullc

scullc — модификация scull, использующая kmem_cache для квантов:

```c
static kmem_cache_t *scullc_cache;

static int __init scullc_init(void)
{
    scullc_cache = kmem_cache_create("scullc",
                                      scullc_quantum,
                                      0,                   // align
                                      SLAB_HWCACHE_ALIGN,
                                      NULL, NULL);        // ctor, dtor
    if (!scullc_cache)
        return -ENOMEM;
    // ... остальная инициализация ...
    return 0;
}
```

### Выделение объекта

```c
void *kmem_cache_alloc(kmem_cache_t *cache, gfp_t flags);
```

Возвращает указатель на объект размера, заданного при `create`. `flags` — обычные GFP_*.

```c
quantum = kmem_cache_alloc(scullc_cache, GFP_KERNEL);
if (!quantum)
    return -ENOMEM;
```

Обычно быстрее, чем `kmalloc`:
- В типичном случае — берётся готовый объект из локального CPU-кэша.
- Нет поиска.

### Освобождение

```c
void kmem_cache_free(kmem_cache_t *cache, void *obj);
```

Объект **не освобождается фактически** — возвращается в кэш для повторного использования.

```c
kmem_cache_free(scullc_cache, quantum);
```

### Уничтожение кэша

```c
int kmem_cache_destroy(kmem_cache_t *cache);
```

Возвращает:
- 0 — успех.
- Не 0 — ошибка (есть невозвращённые объекты).

```c
static void __exit scullc_exit(void)
{
    if (scullc_cache)
        kmem_cache_destroy(scullc_cache);
}
```

**Важно:** все объекты должны быть возвращены **до** destroy. Иначе destroy не выполнится → утечка.

### scullc vs scull

scullc отличается от scull заменой:
- `kmalloc(quantum, GFP_KERNEL)` → `kmem_cache_alloc(scullc_cache, GFP_KERNEL)`.
- `kfree(quantum)` → `kmem_cache_free(scullc_cache, quantum)`.

Все остальное идентично.

### Преимущества scullc

- **Быстрее выделение** (особенно после нескольких free).
- **Память выровнена** по линии кэша → лучше производительность.
- **Меньше фрагментации** (все объекты одного размера).

### Связь с kmalloc

`kmalloc` сам внутри использует slab:

```
kmalloc(150, GFP_KERNEL) → kmem_cache_alloc(kmalloc-192-cache, GFP_KERNEL)
```

Существуют готовые kmalloc-кэши на разные размеры (16, 32, 64, ..., 8192, ...).

`kmalloc` округляет size до ближайшего размера кэша. Это **тратит память**, особенно для нестандартных размеров.

Свой kmem_cache экономит память, если размер нестандартный (как scullc_quantum = 4000, который попадает в кэш `kmalloc-8192` через kmalloc).

### /proc/slabinfo

Все кэши видны через /proc:

```
$ cat /proc/slabinfo
# name <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab>
inode_cache    12345  20000   720  22  4
dentry         98765  150000  192  21  1
task_struct    300    500     2944 11  8
kmalloc-256    5000   7000    256  16  1
scullc         100    200     4000  1   1
```

Поля:
- **name** — имя кэша.
- **active_objs** — используется.
- **num_objs** — всего выделено.
- **objsize** — размер объекта.
- **objperslab** — объектов на slab.
- **pagesperslab** — страниц на slab.

### Конструктор и деструктор

```c
void ctor(void *obj);
```

Вызывается, когда **новая страница slab** заполняется объектами. Может проинициализировать поля, которые не меняются между использованиями.

Пример: для inode конструктор может инициализировать списки, мьютексы. После free объект возвращается с этими полями — следующий пользователь не должен их инициализировать заново.

### Применение

Slab-кэши используются массово:
- **`mm_struct_cachep`** — описания АП.
- **`vm_area_struct_cachep`** — регионы памяти.
- **`task_struct_cachep`** — процессы.
- **`inode_cachep`** — индексы файлов.
- **`dentry_cache`** — элементы каталогов.
- **Драйверные структуры.**

### Резюме

- Slab-аллокатор — централизованное кэширование объектов одного размера.
- Преимущества: быстро, меньше фрагментации, лучше cache locality.
- API: `kmem_cache_create`, `kmem_cache_alloc`, `kmem_cache_free`, `kmem_cache_destroy`.
- scullc использует kmem_cache для квантов.
- `kmalloc` внутри сам использует slab.
- Видны в `/proc/slabinfo`.

---

## 93. Основы USB устройства. Структура драйвера. Оконечные точки. BULK. ISOCHRONOUS.
*Источник: LDD3; tanenbaum.md, Глава 1.3.5*

### USB — что это

**USB (Universal Serial Bus)** — стандарт последовательной шины для подключения периферии.

Из tanenbaum.md (1.3.5):
> «Шина USB была разработана для подключения к компьютеру всех медленных устройств ввода-вывода вроде клавиатуры и мыши. Но как-то неестественно было бы называть устройства USB 3.0… медленными».

### Эволюция

- **USB 1.0/1.1:** 12 Мбит/с (full speed), 1.5 Мбит/с (low speed).
- **USB 2.0:** 480 Мбит/с (high speed).
- **USB 3.0:** 5 Гбит/с (SuperSpeed).
- **USB 3.1:** 10 Гбит/с.
- **USB 4:** до 40 Гбит/с.

### Принцип master/slave

Из tanenbaum.md:
> «USB является централизованной шиной, в которой главное (корневое) устройство один раз в миллисекунду опрашивает все устройства ввода-вывода, чтобы понять, есть ли у них данные для передачи».

Хост (компьютер) опрашивает устройства, а не наоборот. Это упрощает архитектуру USB-устройств (они **отвечают**, а не **прерывают**).

### Hotplug

Устройство можно подключать на лету. Из tanenbaum.md:
> «Любое USB-устройство можно подключать к компьютеру, и оно будет функционировать без необходимости его перезагрузки».

### Структура USB-устройства

Иерархия:
```
Device
  └── Configuration (одна активна)
       └── Interface (один или больше — каждый драйверам отдельный)
            └── Endpoint (один или больше — точки передачи)
```

Подробнее — в вопросе 94.

### Оконечные точки (endpoints)

**Endpoint** — логическая точка передачи данных. Каждая характеризуется:
- **Адресом** (0..15).
- **Направлением** (IN — к хосту, OUT — от хоста).
- **Типом** передачи.

Endpoint 0 (EP0) всегда есть, двунаправленный — для control transfers.

### Типы передачи

**1. Control.**

Setup, конфигурирование, специальные команды.
- Двунаправленные.
- Гарантия доставки.
- Через EP0.

**2. Bulk.**

Большие объёмы без жёстких временных требований.
- Гарантия доставки (повторение при ошибке).
- Гарантия отсутствия ошибок (CRC).
- **Без гарантии задержки.**
- Использует свободное время шины.
- Применение: принтеры, флешки, сканеры, сетевые адаптеры.

**3. Interrupt.**

Малые данные с гарантией latency.
- Гарантия частоты опроса (1-255 мс).
- Гарантия доставки.
- Применение: HID-устройства (клавиатуры, мыши, джойстики).

**4. Isochronous.**

Стабильный поток в реальном времени.
- Гарантированная пропускная способность.
- **БЕЗ гарантии доставки** — потерянные пакеты не повторяются.
- Регулярные передачи (каждый USB-кадр).
- Применение: аудио, видео, телефония.

### Bulk подробнее

Bulk-передача:
- Хост запрашивает данные (для IN) или посылает (для OUT).
- Устройство отвечает или ждёт.
- При ошибке — повтор.

Максимальный размер пакета:
- USB 1.0: 64 байт.
- USB 2.0: 512 байт.
- USB 3.0: 1024 байт.

Эффективность: высокая, если шина свободна.

### Isochronous подробнее

Изохронная передача:
- Зарезервированное время в каждом USB-кадре (1 мс для full speed, 125 мкс для high speed).
- Передаются всегда — даже если данных нет (нулевые пакеты).
- При ошибке — пропуск, без повтора.
- Размер пакета фиксирован.

Расчёт для аудио CD-качества (44.1 kHz, 16-bit, стерео):
- 44100 × 2 × 2 = 176400 байт/с.
- 176400 / 1000 = 176.4 байт/кадр USB.
- → endpoint выделяет 184 байт/кадр.

### Сравнение типов

| Тип | Гарантия доставки | Latency | Применение |
|---|---|---|---|
| Control | Да | Среднее | Setup |
| Bulk | Да | Низкое (best effort) | Большие данные |
| Interrupt | Да | Гарантированная | HID |
| Isochronous | НЕТ | Гарантированная | Аудио/видео |

### Структура драйвера USB в Linux

```c
#include <linux/usb.h>

// Таблица поддерживаемых устройств
static struct usb_device_id my_table[] = {
    { USB_DEVICE(VENDOR_ID, PRODUCT_ID) },
    { }
};
MODULE_DEVICE_TABLE(usb, my_table);

// Структура драйвера
static struct usb_driver my_driver = {
    .name       = "my_usb_driver",
    .id_table   = my_table,
    .probe      = my_probe,
    .disconnect = my_disconnect,
};

module_usb_driver(my_driver);
```

### probe

```c
static int my_probe(struct usb_interface *intf,
                    const struct usb_device_id *id)
{
    struct usb_host_interface *iface_desc;
    struct usb_endpoint_descriptor *endpoint;
    int i;

    iface_desc = intf->cur_altsetting;

    for (i = 0; i < iface_desc->desc.bNumEndpoints; i++) {
        endpoint = &iface_desc->endpoint[i].desc;

        if (usb_endpoint_is_bulk_in(endpoint)) {
            // bulk-in endpoint
        } else if (usb_endpoint_is_bulk_out(endpoint)) {
            // bulk-out endpoint
        }
    }

    usb_set_intfdata(intf, my_dev);
    return 0;
}
```

### disconnect

```c
static void my_disconnect(struct usb_interface *intf)
{
    struct my_dev *dev = usb_get_intfdata(intf);

    usb_set_intfdata(intf, NULL);

    // cleanup
    kfree(dev);
}
```

### URB — единица передачи

```c
struct urb *usb_alloc_urb(int iso_packets, gfp_t mem_flags);
int usb_submit_urb(struct urb *urb, gfp_t mem_flags);
void usb_free_urb(struct urb *urb);
```

URB — описание одной USB-операции. Заполняется и отправляется хост-контроллеру.

Helper-функции для построения URB:
- `usb_fill_control_urb()`
- `usb_fill_bulk_urb()`
- `usb_fill_int_urb()`

Для изохронных URB заполняются вручную.

### Простой синхронный вариант

Для bulk-передач можно без URB:
```c
int usb_bulk_msg(struct usb_device *usb_dev, unsigned int pipe,
                 void *data, int len, int *actual_length, int timeout);
```

Блокирует до завершения. Удобно для тестов.

### Резюме

- USB — централизованная шина, хост опрашивает устройства.
- Иерархия: device → configuration → interface → endpoint.
- 4 типа передачи: control, bulk, interrupt, isochronous.
- Bulk: большие данные без latency.
- Isochronous: real-time потоки без гарантии доставки.
- В Linux драйвер регистрирует id_table, probe, disconnect.

---

## 94. Основы USB устройства. Структура драйвера. Интерфейсы. Конфигурации. struct usb_host_interface *altsetting
*Источник: LDD3*

### Иерархия — полная картина

```
USB Device
  ├── Configuration 1
  │    ├── Interface 0
  │    │    ├── Alternate Setting 0 (default)
  │    │    │    ├── Endpoint 1 (BULK IN)
  │    │    │    └── Endpoint 2 (BULK OUT)
  │    │    ├── Alternate Setting 1
  │    │    │    ├── Endpoint 1 (BULK IN, разные параметры)
  │    │    │    └── Endpoint 2 (BULK OUT)
  │    │    └── ...
  │    ├── Interface 1
  │    │    └── ...
  │    └── ...
  ├── Configuration 2
  │    └── ...
  └── ...
```

### Конфигурация (Configuration)

**Конфигурация** — глобальный режим устройства, набор интерфейсов и их параметров.

Большинство устройств имеют **одну** конфигурацию. Несколько — редкость, обычно для:
- Bus-powered vs self-powered режимы.
- Различные функциональные наборы.

В каждый момент **активна одна** конфигурация. Хост может переключать через `usb_set_configuration`.

### Интерфейс (Interface)

**Интерфейс** — функциональная группа endpoint'ов, соответствующая **одной функции** устройства.

Примеры:
- **USB-аудио карта:** Audio Control + Audio Streaming + Audio Streaming (микрофон).
- **Многофункциональный принтер:** Printer + Scanner + Card Reader.
- **Composite device:** клавиатура + мышь (две HID-функции).

### Важное: один драйвер на интерфейс

В Linux **каждый интерфейс** обрабатывается **отдельным драйвером**. То есть один USB-устройство может обслуживаться **несколькими драйверами** одновременно.

Это даёт гибкость:
- Стандартные функции (HID, MSC) — используют стандартные драйверы ядра.
- Кастомные функции — используют кастомные драйверы.
- Можно «отсоединить» один интерфейс, оставив остальные.

### probe принимает интерфейс

```c
static int my_probe(struct usb_interface *intf, const struct usb_device_id *id);
```

Не `usb_device`, а `usb_interface`. Это явно говорит: «драйвер обрабатывает интерфейс».

### Доступ к устройству через интерфейс

```c
struct usb_device *udev = interface_to_usbdev(intf);
```

Через usb_device можно получить общую инфу: VID, PID, скорость, и т.д.

### Альтернативные настройки (altsettings)

Один интерфейс может иметь **несколько альтернативных настроек**, отличающихся параметрами endpoint'ов.

Применение — **переключение режимов** без полного сброса.

### Пример: USB-камера

```
Interface 1 (Video Streaming)
  ├── altsetting 0: нет данных
  │    └── 0 endpoints (или 0-байтовые)
  ├── altsetting 1: 320×240@15fps
  │    └── 1 isochronous IN, 192 байт/пакет
  ├── altsetting 2: 640×480@30fps
  │    └── 1 isochronous IN, 768 байт/пакет
  └── altsetting 3: 1280×720@30fps
       └── 1 isochronous IN, 2400 байт/пакет
```

Хост выбирает altsetting:
```c
usb_set_interface(udev, ifnum, altsetting);
```

### Почему altsettings, не конфигурации

Конфигурация — глобальная, переключение тяжёлое (сброс устройства).
Altsetting — локальный для интерфейса, легче и быстрее.

Также при включении видеостриминга можно переключаться между качествами на лету.

### Зачем altsetting 0 без данных

USB резервирует **bandwidth** для isochronous endpoints **при выборе altsetting**. Если устройство не передаёт данные — выбираем altsetting 0 (без isoc endpoints), bandwidth не резервируется → другие устройства могут использовать.

При начале стриминга — выбираем altsetting с нужным качеством. После окончания — обратно в 0.

### struct usb_host_interface

```c
struct usb_host_interface {
    struct usb_interface_descriptor desc;
    struct usb_host_endpoint *endpoint;
    char *string;
    /* ... */
};
```

Хранит:
- **desc** — дескриптор интерфейса (число endpoints, класс, ...).
- **endpoint** — массив endpoint'ов.
- **string** — имя.

### struct usb_interface

```c
struct usb_interface {
    struct usb_host_interface *altsetting;     // массив altsettings
    struct usb_host_interface *cur_altsetting; // текущий
    unsigned num_altsetting;                    // число
    /* ... */
};
```

`altsetting` — **массив** структур `usb_host_interface`. Размер — `num_altsetting`.

`cur_altsetting` — указатель на текущий активный.

### Доступ к endpoint'ам

```c
struct usb_host_interface *iface = intf->cur_altsetting;
struct usb_endpoint_descriptor *ep;

for (int i = 0; i < iface->desc.bNumEndpoints; i++) {
    ep = &iface->endpoint[i].desc;

    if (usb_endpoint_is_bulk_in(ep)) {
        bulk_in_addr = usb_endpoint_num(ep);
        bulk_in_size = le16_to_cpu(ep->wMaxPacketSize);
    } else if (usb_endpoint_is_bulk_out(ep)) {
        // ...
    }
}
```

### Helper-макросы для endpoint'ов

```c
usb_endpoint_dir_in(ep)       // направление
usb_endpoint_dir_out(ep)
usb_endpoint_xfer_bulk(ep)    // тип
usb_endpoint_xfer_int(ep)
usb_endpoint_xfer_isoc(ep)
usb_endpoint_xfer_control(ep)
usb_endpoint_is_bulk_in(ep)   // комбинации
usb_endpoint_is_bulk_out(ep)
usb_endpoint_is_int_in(ep)
usb_endpoint_is_isoc_in(ep)
usb_endpoint_num(ep)          // адрес
usb_endpoint_maxp(ep)         // макс. размер пакета
```

### Изменение altsetting

```c
int usb_set_interface(struct usb_device *dev, int ifnum, int alternate);
```

Например, при начале захвата видео:
```c
usb_set_interface(udev, 1, 2);   // интерфейс 1, altsetting 2 (640×480)
```

После этого хост-контроллер резервирует bandwidth для isoc endpoint'ов нового altsetting.

### Полный пример probe

```c
static int webcam_probe(struct usb_interface *intf,
                        const struct usb_device_id *id)
{
    struct usb_device *udev = interface_to_usbdev(intf);
    struct usb_host_interface *iface;
    struct usb_endpoint_descriptor *ep;
    int i;

    pr_info("Webcam connected: %d altsettings\n", intf->num_altsetting);

    for (i = 0; i < intf->num_altsetting; i++) {
        iface = &intf->altsetting[i];
        pr_info("  altsetting %d: %d endpoints\n",
                iface->desc.bAlternateSetting, iface->desc.bNumEndpoints);

        for (int j = 0; j < iface->desc.bNumEndpoints; j++) {
            ep = &iface->endpoint[j].desc;
            pr_info("    ep%02x type=%d size=%d\n",
                    ep->bEndpointAddress,
                    usb_endpoint_type(ep),
                    usb_endpoint_maxp(ep));
        }
    }

    return 0;
}
```

### Резюме

- USB-устройство: device → configurations → interfaces → endpoints.
- Один интерфейс = один драйвер (composite устройства обслуживаются несколькими драйверами).
- Альтернативные настройки (altsettings) — варианты параметров endpoints.
- `struct usb_interface` имеет массив `altsetting` и текущий `cur_altsetting`.
- Переключение altsetting через `usb_set_interface`.
- Применение: USB-камеры (качество), USB-аудио (sample rate).

---

## 95. Пример изохорной точки в whireshark. Устройство отправляющее изохорные точки.
*Источник: LDD3; USB-протокол*

### Что такое изохронный endpoint

См. вопрос 93. Изохронные передачи — для **стабильного потока** real-time.

### Какие устройства используют isochronous

- **USB-микрофон** (USB Audio Class — UAC).
- **USB-аудио карта.**
- **USB-наушники** с микрофоном.
- **USB-камера** (USB Video Class — UVC).
- **USB-телевизионные тюнеры.**
- **USB-телефония.**

Общее: **поток** в реальном времени, где потеря отдельных кадров терпима, но **задержка** — нет.

### Пример: USB-микрофон

USB-аудио устройство соответствует **USB Audio Class** (UAC). Структура:

```
Configuration 1
  ├── Interface 0 (Audio Control) — control endpoint
  └── Interface 1 (Audio Streaming)
       ├── altsetting 0: нет endpoints
       └── altsetting 1: isochronous IN endpoint
```

При начале записи хост:
1. Выбирает altsetting 1.
2. Подписывается на isochronous endpoint.
3. Получает кадры аудио каждый USB-фрейм (1 мс).

### Wireshark — захват USB

В Linux для захвата USB:
```bash
sudo modprobe usbmon
sudo wireshark
# выбрать "usbmonX" интерфейс
```

В Windows: USBPcap + Wireshark.

### Фильтрация

В Wireshark:
- `usb.transfer_type == 0x00` — isochronous (или 0x01 для interrupt, 0x02 для control, 0x03 для bulk).
- `usb.endpoint_address == 0x81` — конкретный endpoint.

### Структура изохронного пакета

В Wireshark отображается:
```
URB type: COMPLETE (or SUBMIT)
URB transfer type: ISOCHRONOUS (0x00)
URB id: 0x...
Endpoint: 0x81 (IN, EP 1)
Device: 4
URB sec / usec: timestamps
URB packet length: 192
ISO error count: 0
ISO descriptor:
    Offset: 0
    Length: 192
    Status: 0 (success)
Data (192 bytes): 88 92 7B 6D ...
```

### Поля

- **URB packet length:** общий размер.
- **ISO descriptor:** информация о конкретном изохронном пакете в URB.
  - **Length:** реальное число байт.
  - **Status:** 0 = успех, иначе ошибка.
- **Data:** собственно аудио.

### Множественные пакеты в одном URB

Изохронный URB может содержать **несколько** изохронных пакетов:
```
URB:
  ISO packet 0: 192 байт, status 0
  ISO packet 1: 192 байт, status 0
  ISO packet 2: 192 байт, status 0
  ...
```

Это уменьшает накладные расходы — один URB вместо множества.

### Расчёт для аудио CD-качества

Параметры:
- Sample rate: 44100 Hz.
- Bit depth: 16 бит = 2 байта.
- Channels: 2 (стерео).

Bandwidth: 44100 × 2 × 2 = 176400 байт/с.

На USB-фрейм (1 мс): 176400 / 1000 = 176.4 байт.

Но! Размер пакета должен быть целым числом байт. Обычно резервируется 184 байт/фрейм, иногда отправляется 176, иногда 184 — чтобы средняя скорость была 176.4 байт/фрейм.

### Пример последовательности пакетов

В Wireshark подряд:
```
Frame 1: ISO 184 bytes, status 0
Frame 2: ISO 176 bytes, status 0
Frame 3: ISO 184 bytes, status 0
Frame 4: ISO 184 bytes, status 0
Frame 5: ISO 176 bytes, status 0
...
```

Каждый ровно через 1 мс.

### Реальные данные (PCM)

Содержимое пакета — PCM-семплы:
```
Big-endian, signed 16-bit, interleaved L/R:
88 92  // L sample 1: 0x8892
7B 6D  // R sample 1: 0x7B6D
... и т.д.
```

### Устройства, отправляющие изохронные пакеты

**1. USB-микрофон / USB-аудио гарнитура.**

Постоянно шлёт аудио в isochronous endpoint. Хост получает и подаёт на ALSA/PulseAudio.

**2. USB-камера.**

Шлёт видео в isochronous endpoint. Размер зависит от разрешения и fps.

Для 640×480@30fps, MJPEG (~50 КБ/кадр):
- 30 кадров/с × 50 КБ = 1.5 МБ/с = 12 Мбит/с.
- На USB 2.0 (480 Мбит/с) — занимает ~2.5%.

**3. USB-TV тюнер.**

Поток MPEG-2 TS — isochronous.

**4. USB-телефония / IP-телефон.**

G.711 audio, 64 Кбит/с — небольшая нагрузка.

### Когда статус != 0

В Wireshark статусы:
- 0 — успех.
- -EOVERFLOW — пакет больше ожидаемого.
- -EXDEV — bandwidth не доступен.

При нагрузке на USB-шину или конфликтах bandwidth могут быть потери. Для isoc это нормально.

### Сравнение с bulk

Если бы аудио шло через bulk:
- Гарантия доставки.
- Но возможны задержки.
- Результат — заикания при загруженной шине.

Isochronous: предсказуемая latency ценой возможной потери — это лучше для аудио.

### Резюме

- Isochronous — для real-time потоков.
- Отправляющие устройства: USB-аудио, видео, телефония.
- В Wireshark: transfer_type = 0x00, регулярные пакеты, frame numbers.
- Структура: URB → массив ISO packets с descriptor каждого.
- Bandwidth резервируется при выборе altsetting.

---

## 96. Пример данных получаемых от мыши в whireshark.
*Источник: LDD3; HID-протокол*

### USB-мышь

USB-мышь — **HID-устройство** (Human Interface Device). Использует **interrupt** endpoint (не bulk, не isochronous).

### Почему interrupt

- Малые данные (4 байта).
- Нужна гарантированная latency.
- Гарантия доставки (потеря клика недопустима).
- Bulk — не подходит (latency не гарантирована).
- Isochronous — не подходит (нужна гарантия доставки).
- Interrupt — идеально.

### Структура USB-мыши

```
Device
  └── Configuration 1
       └── Interface 0 (HID, boot protocol — mouse)
            ├── EP0 (control, для setup и HID get/set)
            └── EP1 IN (interrupt, для отчётов о движении)
```

### HID-протокол

HID — стандарт для устройств взаимодействия с человеком. Формат отчёта определяется **HID Report Descriptor**.

Стандартный «boot protocol mouse» отчёт — 3 байта:
```
Байт 0: кнопки (бит 0 — левая, бит 1 — правая, бит 2 — средняя)
Байт 1: dX (signed)
Байт 2: dY (signed)
```

Современные мыши обычно 4 байта:
```
Байт 0: кнопки
Байт 1: dX
Байт 2: dY
Байт 3: dWheel (прокрутка)
```

Gaming мыши могут иметь 5+ байт с дополнительными кнопками или 16-битными координатами.

### Wireshark — захват

Linux:
```bash
sudo modprobe usbmon
sudo wireshark
# выбрать usbmonX (где X — номер хост-контроллера, к которому подключена мышь)
# фильтр: usb.transfer_type == 0x01 (interrupt)
```

Можно дополнительно: `usb.endpoint_address == 0x81 && usb.src == "host"` для конкретного направления.

### Опрос мыши

Хост опрашивает мышь каждые **N** мс (определяется `bInterval` в endpoint descriptor):
- Обычная мышь: 8-10 мс.
- Gaming мышь: 1 мс (1000 Hz polling).

При каждом опросе хост шлёт **IN-запрос** на endpoint 1. Мышь отвечает отчётом.

### Пример: мышь неподвижна

При отсутствии активности мышь шлёт «нулевой» отчёт:
```
URB type: COMPLETE
URB transfer type: INTERRUPT (0x01)
Endpoint: 0x81 (IN, EP 1)
Device: 5
URB packet length: 4
Data: 00 00 00 00
```

Или, на некоторых устройствах — **NAK** (не отвечает, нет данных). Тогда нет URB COMPLETE с данными, есть только URB SUBMIT.

### Пример: нажатие левой кнопки

```
Data: 01 00 00 00
       ↑
       └ бит 0 = 1 (левая кнопка)
```

После отпускания:
```
Data: 00 00 00 00
```

### Пример: нажатие правой кнопки

```
Data: 02 00 00 00
       ↑
       └ бит 1 = 1 (правая)
```

### Пример: нажатие обеих кнопок одновременно

```
Data: 03 00 00 00
       ↑
       └ биты 0 и 1 = 1
```

### Пример: движение

dX и dY — signed 8-bit (или signed 16-bit на gaming мышах).

Движение вправо-вверх:
```
Data: 00 0A FB 00
          │  │
          │  └ dY = 0xFB = -5 (вверх в экранных координатах)
          └ dX = 0x0A = +10 (вправо)
```

Координаты экрана: Y растёт **вниз**. Поэтому движение «вверх» — отрицательное dY.

### Пример: прокрутка колеса вниз

```
Data: 00 00 00 FF
                ↑
                └ dWheel = -1
```

«Один щелчок вниз». Прокрутка вверх — +1.

### Пример: drag (нажата кнопка + движение)

```
Data: 01 05 03 00
       │  │  │
       │  │  └ dY = +3
       │  └ dX = +5
       └ левая кнопка нажата
```

### Парсинг в Wireshark

Wireshark часто парсит HID-данные:
```
HID Data:
  Buttons:
    Button 1 (Left): Pressed
    Button 2 (Right): Not pressed
    Button 3 (Middle): Not pressed
  X displacement: 10
  Y displacement: -5
  Wheel: 0
```

### Частота отчётов

При обычном опросе 8 мс:
- 125 отчётов в секунду.
- Если мышь не двигается — 125 нулевых отчётов в секунду.

При gaming-poll 1 мс:
- 1000 отчётов в секунду.

В Wireshark видна **временная диаграмма** — каждый отчёт с timestamp.

### Boot Protocol vs Report Protocol

При запуске мышь работает в **Boot protocol** — стандартные 3 байта, поддерживается BIOS.

После загрузки ОС обычно переключает в **Report protocol** — формат определяется HID Report Descriptor устройства.

В Wireshark при захвате с самого начала видно:
1. Setup-команды (control transfers).
2. Чтение HID descriptor (control transfers).
3. Set protocol → Report.
4. Регулярные отчёты.

### Резюме

- USB-мышь использует interrupt endpoint.
- HID-отчёт обычно 4 байта: кнопки, dX, dY, dWheel.
- В Wireshark: `usb.transfer_type == 0x01`.
- Регулярные отчёты (каждые 1-10 мс), даже без активности.
- При нажатии/движении меняются соответствующие байты.

---

## 97. Интерфейс PCI. Адресация. Регистры конфигурации. DeviceID и VendorID. PCI_DEVICE
*Источник: LDD3; tanenbaum.md, Глава 1.3.5*

### PCI — обзор

**PCI (Peripheral Component Interconnect)** — стандарт параллельной шины для подключения периферии.

Из tanenbaum.md (1.3.5):
> «Наиболее старой из них является PCI (Peripheral Component Interconnect — интерфейс периферийных компонентов)».

### PCI vs PCIe

**PCI:**
- Параллельная (32 или 64 бита).
- Общая шина с арбитром.
- 33 МГц (~133 МБ/с).

**PCIe (PCI Express):**
- Последовательная (lanes).
- Точка-точка (через свитч).
- Сейчас доминирует.

С точки зрения программного обеспечения — почти одинаковы.

### Адресация PCI

Каждое устройство (точнее, **функция** устройства) адресуется тройкой:
- **Bus** (0..255) — номер шины.
- **Device** (0..31) — номер устройства на шине.
- **Function** (0..7) — номер функции.

Итого: 256 × 32 × 8 = 65 536 функций на одну PCI-иерархию.

Обозначение в Linux: `BB:DD.F` (например, `00:1f.2`).

Часто это записывают как **BDF** (Bus, Device, Function).

### Просмотр PCI-устройств

```bash
$ lspci
00:00.0 Host bridge: Intel Corporation Xeon E3 Host Bridge
00:02.0 VGA compatible controller: Intel HD Graphics
00:14.0 USB controller: Intel USB 3.0 xHCI
00:1d.0 USB controller: Intel USB 2.0 EHCI
00:1f.2 SATA controller: Intel 8 Series Chipset SATA AHCI Controller
03:00.0 Network controller: Intel Wireless 7260
```

### Шины и мосты

Шина 0 — корневая. Хост-мост соединяет CPU с PCI-устройствами.

Если есть PCI-PCI мосты — появляются дополнительные шины (1, 2, ...).

### Многофункциональные устройства

Одно физическое устройство может иметь несколько функций. Например:
- Function 0: USB 2.0 controller.
- Function 1: USB 3.0 controller.
- Function 2: Audio controller.

Каждая функция воспринимается независимо, имеет свой VID/DID, BAR, и т.д.

### Конфигурационное пространство

Каждая PCI-функция имеет **256 байт** (PCI) или **4096 байт** (PCIe Extended) **конфигурационных регистров**.

Эти регистры доступны через специальный механизм (PCI Configuration Access). ОС может читать/писать их.

### Стандартный header (первые 64 байта)

```
Offset  Field                       Size
0x00    Vendor ID                   2
0x02    Device ID                   2
0x04    Command                     2
0x06    Status                      2
0x08    Revision ID                 1
0x09    Class code                  3
0x0C    Cache Line Size             1
0x0D    Latency Timer               1
0x0E    Header Type                 1
0x0F    BIST                        1
0x10    BAR0                        4
0x14    BAR1                        4
0x18    BAR2                        4
0x1C    BAR3                        4
0x20    BAR4                        4
0x24    BAR5                        4
0x28    Cardbus CIS Pointer         4
0x2C    Subsystem Vendor ID         2
0x2E    Subsystem Device ID         2
0x30    Expansion ROM Base Address  4
0x34    Capabilities Pointer        1
0x3C    Interrupt Line              1
0x3D    Interrupt Pin               1
0x3E    Min Grant                   1
0x3F    Max Latency                 1
```

### Vendor ID и Device ID

**Vendor ID** — идентификатор производителя.
- Выдаётся PCI-SIG (Special Interest Group, https://pcisig.com).
- 16-битное число.

Примеры:
- 0x8086 — Intel.
- 0x10DE — NVIDIA.
- 0x1022 — AMD.
- 0x14E4 — Broadcom.
- 0x1969 — Atheros.

(0x8086 у Intel — отсылка к процессору i8086.)

**Device ID** — конкретная модель.
- Назначает сам производитель.
- 16-битное.

Пара (VID, DID) однозначно идентифицирует тип устройства.

### Subsystem VID/DID

Дополнительная пара (Subsystem Vendor ID, Subsystem Device ID) — идентифицирует **карту/модуль**, использующий конкретный чип.

Пример: Intel разработала чип Wi-Fi (VID/DID), но карты на его базе делают разные OEM (Subsystem). У них могут быть разные антенны, прошивки.

### Просмотр в lspci

```bash
$ lspci -nn
03:00.0 Network controller [0280]: Intel Corporation Wireless 7260 [8086:08b1] (rev 73)
                                                                  ^^^^^^^^^^^
                                                                  VID:DID
```

```bash
$ lspci -nnv -s 03:00.0
03:00.0 Network controller [0280]: Intel Corporation Wireless 7260 [8086:08b1] (rev 73)
        Subsystem: Intel Corporation Dual Band Wireless-AC 7260 [8086:4070]
                                                                ^^^^^^^^^^^
                                                                Subsystem VID:DID
```

### Class code

3 байта, классификация устройства:
- **Class** (старший байт):
  - 0x01: Mass Storage Controller.
  - 0x02: Network Controller.
  - 0x03: Display Controller.
  - 0x04: Multimedia Controller.
  - 0x06: Bridge.
  - 0x0C: Serial Bus Controller.
- **Subclass:** уточнение.
- **Programming interface:** ещё уточнение.

Например: `0x010601` = Mass Storage / SATA / AHCI.

### BAR — Base Address Register

6 регистров BAR0..BAR5, по 4 байта.

Содержат адреса, по которым устройство **отображено** в адресное пространство:
- **I/O port BAR:** младший бит = 1, остальное — I/O порт.
- **Memory BAR:** младший бит = 0, остальное — адрес памяти.

ОС при загрузке:
1. Читает BAR (узнаёт требуемый размер региона).
2. Резервирует регион в адресном пространстве.
3. Записывает базовый адрес обратно в BAR.

После этого устройство «появляется» по этому адресу.

### Регистры в Linux

```c
int pci_read_config_byte(struct pci_dev *dev, int where, u8 *val);
int pci_read_config_word(struct pci_dev *dev, int where, u16 *val);
int pci_read_config_dword(struct pci_dev *dev, int where, u32 *val);
int pci_write_config_byte(...);
int pci_write_config_word(...);
int pci_write_config_dword(...);
```

### Доступ к BAR

```c
unsigned long start = pci_resource_start(pdev, 0);   // адрес BAR0
unsigned long len = pci_resource_len(pdev, 0);
unsigned long flags = pci_resource_flags(pdev, 0);

if (flags & IORESOURCE_MEM) {
    void __iomem *regs = ioremap(start, len);
    // использовать regs
    iounmap(regs);
}
```

Или удобнее:
```c
pci_iomap(pdev, 0, len);   // BAR0
```

### Идентификация устройств — PCI_DEVICE

Макрос для построения id-структуры:

```c
#define PCI_DEVICE(vend, dev) \
    .vendor = (vend), .device = (dev), \
    .subvendor = PCI_ANY_ID, .subdevice = PCI_ANY_ID
```

Использование:
```c
static struct pci_device_id my_pci_ids[] = {
    { PCI_DEVICE(0x8086, 0x1234) },
    { PCI_DEVICE(0x10DE, 0x5678) },
    { 0, }     // обязательный терминатор
};
MODULE_DEVICE_TABLE(pci, my_pci_ids);
```

### MODULE_DEVICE_TABLE

Создаёт метаинформацию для **hotplug**. Утилита `depmod` создаёт файл с этой инфой; при подключении нового устройства udev/systemd ищет драйвер по VID/DID и загружает соответствующий модуль автоматически.

### Дополнительные макросы

```c
PCI_DEVICE_CLASS(class, mask)         // по классу
PCI_DEVICE_SUB(vend, dev, subvend, subdev)   // с подсистемой
```

### Резюме

- PCI-функция адресуется (Bus, Device, Function) = BDF.
- 256-байтное конфигурационное пространство.
- Vendor ID и Device ID однозначно идентифицируют тип устройства.
- BAR содержат адреса отображения регистров устройства.
- В Linux: `PCI_DEVICE(vid, did)` для таблицы поддерживаемых устройств.
- `MODULE_DEVICE_TABLE(pci, ...)` для автозагрузки драйвера.

---

## 98. Интерфейс PCI. Регистрация PCI драйвера. *probe, *remove, int (*suspend) (struct pci_dev *dev, u32 state);
*Источник: LDD3*

### struct pci_driver

```c
struct pci_driver {
    const char *name;
    const struct pci_device_id *id_table;

    int  (*probe)    (struct pci_dev *dev, const struct pci_device_id *id);
    void (*remove)   (struct pci_dev *dev);
    int  (*suspend)  (struct pci_dev *dev, pm_message_t state);
    int  (*resume)   (struct pci_dev *dev);
    void (*shutdown) (struct pci_dev *dev);

    struct device_driver driver;
    /* ... */
};
```

### Полная регистрация

```c
static struct pci_device_id my_pci_ids[] = {
    { PCI_DEVICE(0x8086, 0x1234) },
    { 0, }
};
MODULE_DEVICE_TABLE(pci, my_pci_ids);

static struct pci_driver my_pci_driver = {
    .name     = "my_pci_driver",
    .id_table = my_pci_ids,
    .probe    = my_probe,
    .remove   = my_remove,
    .suspend  = my_suspend,
    .resume   = my_resume,
};

static int __init my_init(void) {
    return pci_register_driver(&my_pci_driver);
}

static void __exit my_exit(void) {
    pci_unregister_driver(&my_pci_driver);
}

module_init(my_init);
module_exit(my_exit);
```

Или коротко:
```c
module_pci_driver(my_pci_driver);
```

### probe — детально

```c
int (*probe)(struct pci_dev *dev, const struct pci_device_id *id);
```

Вызывается, когда найдено устройство, поддерживаемое драйвером.

**Параметры:**
- `dev` — указатель на pci_dev.
- `id` — запись из id_table, которая совпала.

**Возврат:** 0 при успехе, отрицательный errno при ошибке.

### Стандартные действия в probe

```c
static int my_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    struct my_dev *dev;
    int err;

    // 1. Выделить структуру драйвера
    dev = kzalloc(sizeof(*dev), GFP_KERNEL);
    if (!dev) return -ENOMEM;

    // 2. Включить устройство (Bus Master, MMIO)
    err = pci_enable_device(pdev);
    if (err) goto err_free;

    // 3. Запросить регионы памяти
    err = pci_request_regions(pdev, "my_driver");
    if (err) goto err_disable;

    // 4. Отобразить BAR0 в память ядра
    dev->regs = pci_iomap(pdev, 0, 0);
    if (!dev->regs) {
        err = -ENOMEM;
        goto err_release;
    }

    // 5. Включить DMA
    err = pci_set_dma_mask(pdev, DMA_BIT_MASK(64));
    if (err) goto err_iounmap;
    pci_set_master(pdev);

    // 6. Зарегистрировать обработчик прерывания
    err = request_irq(pdev->irq, my_irq_handler,
                      IRQF_SHARED, "my_driver", dev);
    if (err) goto err_iounmap;

    // 7. Сохранить указатель для других callback'ов
    pci_set_drvdata(pdev, dev);
    dev->pdev = pdev;

    // 8. Зарегистрировать интерфейсы (символьное устройство, сетевой, ...)
    err = my_register_devices(dev);
    if (err) goto err_irq;

    return 0;

err_irq:
    free_irq(pdev->irq, dev);
err_iounmap:
    pci_iounmap(pdev, dev->regs);
err_release:
    pci_release_regions(pdev);
err_disable:
    pci_disable_device(pdev);
err_free:
    kfree(dev);
    return err;
}
```

### pci_enable_device

Активирует устройство:
- Включает Memory Space Enable (бит Command register).
- Включает I/O Space Enable, если нужно.
- Может включить устройство из low-power.

### pci_request_regions

Резервирует регионы BAR в реестре ресурсов ядра. Препятствует конфликтам, если другой драйвер попытается захватить эти же регионы.

### pci_iomap

Отображает BAR в виртуальное АП ядра:
```c
void __iomem *regs = pci_iomap(pdev, 0, 0);   // BAR0, весь размер
```

После этого можно работать с регистрами устройства через `readl/writel`:
```c
u32 val = readl(regs + 0x10);   // прочитать регистр по offset 0x10
writel(0x1234, regs + 0x10);    // записать
```

### pci_set_master

Включает Bus Master bit — устройство может инициировать DMA.

### remove — детально

```c
void (*remove)(struct pci_dev *dev);
```

Вызывается при:
- Отключении устройства (hotplug).
- Выгрузке модуля драйвера.

Зеркальная к probe — освобождает всё:

```c
static void my_remove(struct pci_dev *pdev)
{
    struct my_dev *dev = pci_get_drvdata(pdev);

    my_unregister_devices(dev);
    free_irq(pdev->irq, dev);
    pci_iounmap(pdev, dev->regs);
    pci_release_regions(pdev);
    pci_disable_device(pdev);
    kfree(dev);
}
```

### suspend — power management

```c
int (*suspend)(struct pci_dev *dev, pm_message_t state);
```

Вызывается при переходе системы в спящий режим. Задачи:

1. **Завершить активные операции.**
2. **Сохранить состояние устройства.**
3. **Перевести в low-power.**

```c
static int my_suspend(struct pci_dev *pdev, pm_message_t state)
{
    struct my_dev *dev = pci_get_drvdata(pdev);

    // 1. Прекратить ввод-вывод
    my_stop_io(dev);

    // 2. Сохранить состояние
    pci_save_state(pdev);

    // 3. Отключить устройство
    pci_disable_device(pdev);

    // 4. Перевести в D3 (low-power state)
    pci_set_power_state(pdev, pci_choose_state(pdev, state));

    return 0;
}
```

### resume

```c
int (*resume)(struct pci_dev *dev);
```

Зеркальная suspend:

```c
static int my_resume(struct pci_dev *pdev)
{
    struct my_dev *dev = pci_get_drvdata(pdev);
    int err;

    // 1. В D0
    pci_set_power_state(pdev, PCI_D0);

    // 2. Восстановить state
    pci_restore_state(pdev);

    // 3. Включить устройство
    err = pci_enable_device(pdev);
    if (err) return err;

    pci_set_master(pdev);

    // 4. Возобновить операции
    my_restart_io(dev);

    return 0;
}
```

### pm_message_t

```c
typedef struct pm_message {
    int event;
} pm_message_t;
```

События:
- `PM_EVENT_SUSPEND` — обычный suspend (RAM).
- `PM_EVENT_HIBERNATE` — гибернация (диск).
- `PM_EVENT_FREEZE` — заморозка для подготовки к hibernate.
- `PM_EVENT_PRETHAW` — перед размораживанием.

### Состояния питания

| State | Описание |
|---|---|
| D0 | Полностью включено |
| D1 | Лёгкий сон, можно почти мгновенно вернуться в D0 |
| D2 | Глубже, чем D1 |
| D3hot | Питание есть, но устройство «выключено» |
| D3cold | Без питания |

Переходы:
- D0 → D3 — при suspend.
- D3 → D0 — при resume.

### shutdown

```c
void (*shutdown)(struct pci_dev *dev);
```

Вызывается при выключении/перезагрузке системы. Гарантирует, что устройство не повредит данные при отключении питания.

Обычно:
- Завершить активные операции.
- Сбросить устройство.
- Выключить.

### Hotplug

Через `MODULE_DEVICE_TABLE(pci, my_pci_ids)` ядро знает, какие устройства драйвер поддерживает.

При подключении нового PCI-устройства (например, hot-plug PCIe slot или USB-PCI bridge):
1. Ядро обнаруживает устройство, читает VID/DID.
2. Ищет в `modules.alias` подходящий модуль.
3. udev/systemd загружает модуль.
4. probe вызывается.

### Резюме

- `struct pci_driver` — описание драйвера PCI.
- `probe` вызывается при обнаружении устройства, делает всю инициализацию.
- `remove` — обратное, освобождает ресурсы.
- `suspend`/`resume` — для power management.
- `shutdown` — при выключении системы.
- Стандартный цикл: enable_device → request_regions → iomap → request_irq.
- MODULE_DEVICE_TABLE для автозагрузки.

---

## 99. Файловые системы. Непрерывное размещение файлов. Достоинства и недостатки. Пример. Применение. Атрибуты файла при непрерывном размещении файлов.
*Источник: tanenbaum.md, Глава 4 «Файловые системы»*

### Контекст: задача файловой системы

Файловая система должна **отслеживать**, какие блоки диска принадлежат каким файлам.

Способы организации:
1. **Непрерывное размещение** (этот вопрос).
2. **Связанный список** блоков (вопрос 100).
3. **FAT** — связанный список с таблицей в RAM.
4. **i-узлы** — индекс-узлы с массивом адресов блоков.

### Непрерывное размещение

Самый простой подход. Из tanenbaum.md (4.3.2):
> «Простейшая схема размещения заключается в хранении каждого файла на диске в виде непрерывной последовательности блоков. Таким образом, на диске с блоками, имеющими размер 1 Кбайт, файл размером 50 Кбайт займет 50 последовательных блоков. При блоках, имеющих размер 2 Кбайт, под него будет выделено 25 последовательных блоков».

**Каждый файл — один смежный блок памяти на диске.**

### Атрибуты файла

Минимально нужно хранить:
- **Адрес первого блока.**
- **Количество блоков** (или размер в байтах).

+ обычные атрибуты:
- Имя.
- Тип (файл/каталог/устройство).
- Время создания/изменения/доступа.
- Владелец и группа.
- Права доступа.

### Зная два числа — найдём всё

Адрес n-го блока:
```
block_n = start + n
```

O(1) вычисление. Не нужно никаких служебных структур на диске или в памяти кроме этих двух чисел.

### Пример

Диск с блоками по 1 КБ:

```
Block:   0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16
        [== A ==][======= B =========][== C ==][=== D ===]
```

- Файл A: start=0, length=4. Блоки: 0, 1, 2, 3.
- Файл B: start=4, length=7. Блоки: 4, 5, 6, 7, 8, 9, 10.
- Файл C: start=11, length=3. Блоки: 11, 12, 13.
- Файл D: start=14, length=4. Блоки: 14, 15, 16, 17.

### Внутренняя фрагментация

> «Следует заметить, что каждый файл начинается от границы нового блока, поэтому, если файл A фактически имел длину 3,5 блока, то в конце последнего блока часть пространства будет потеряна впустую».

Если файл = 3500 байт и блок = 1024 байт:
- Занимает 4 блока = 4096 байт.
- 596 байт **внутренней фрагментации**.

Это общая для всех схем проблема.

### Достоинство 1: Простота

Из tanenbaum.md:
> «Его просто реализовать, поскольку отслеживание местонахождения принадлежащих файлу блоков сводится всего лишь к запоминанию двух чисел».

Минимум кода в ФС.

### Достоинство 2: Превосходная производительность чтения

> «Весь файл может быть считан с диска за одну операцию. Для нее потребуется только одна операция позиционирования (на первый блок). После этого никаких позиционирований или ожиданий подхода нужного сектора диска уже не потребуется, поэтому данные поступают на скорости, равной максимальной пропускной способности диска».

Для HDD это **критически важно**. Seek time может составлять миллисекунды (намного больше, чем чтение блока). Если файл фрагментирован — много seek'ов. Непрерывный файл — один seek + последовательное чтение.

### Достоинство 3: O(1) произвольный доступ

```
read(file, byte n):
    block = start + n / blocksize
    offset = n % blocksize
    return disk_read(block) at offset
```

Без обращения к каким-то таблицам.

### Недостаток 1: Фрагментация диска

Из tanenbaum.md:
> «У непрерывного размещения есть также очень серьезный недостаток: со временем диск становится фрагментированным».

При удалении файлов остаются «дыры». Новый файл должен поместиться в одну из них. Если файл больше любой дыры — даже при свободном месте в сумме файл не создаётся.

### Иллюстрация

После удаления:
```
Block:   0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16
        [== A ==]          ────────  [== C ==]
                    дыра 6 блоков                дыра 4 блока (B и D удалены)
```

Файл 8 блоков создать нельзя — нет смежной области.

Решения:
- **Уплотнение (defragmentation)** — переместить все файлы для устранения дыр. Очень медленно.
- **Список свободных областей** + смотреть, помещается ли.

### Недостаток 2: Размер заранее

Из tanenbaum.md:
> «При создании нового файла необходимо знать его окончательный размер, чтобы выбрать подходящий для размещения участок».

Сценарий:
- Создать файл.
- Записать 1 КБ.
- Записать ещё 1 КБ.
- ...

Если изначально выделили 10 КБ и хотим больше — нужно либо:
- Резервировать с запасом (теряем место).
- Переместить файл в большую область (медленно).

Это **смертельный** недостаток для типичного использования.

Из tanenbaum.md, сценарий:
> «Пользователь запускает текстовый процессор, чтобы создать документ. В первую очередь программа спрашивает, сколько байтов в конечном итоге будет занимать документ. Без ответа на этот вопрос она не сможет продолжить работу… Такая система вряд ли осчастливит пользователей».

### Где применяется

Из tanenbaum.md:
> «Есть одна сфера применения, в которой непрерывное размещение вполне приемлемо и все еще используется на практике, — это компакт-диски. Здесь все размеры файлов известны заранее и никогда не изменяются».

**Подходит для:**
- **CD-ROM, DVD-ROM, Blu-ray.** Файлы записываются один раз.
- **Архивы** (TAR, CPIO, ZIP — внутри есть концепция «непрерывный поток»).
- **Магнитные ленты** — последовательная запись.
- **WORM-носители** (Write Once Read Many).

### Что с DVD-фильмами

Из tanenbaum.md:
> «90-минутный фильм может быть закодирован в виде одного-единственного файла длиной около 4,5 Гбайт, но в используемой файловой системе UDF (Universal Disk Format) для представления длины файла применяется 30-разрядное число, которое ограничивает длину файлов одним гигабайтом. Вследствие этого DVD-фильмы, как правило, хранятся в виде трех-четырех файлов размером 1 Гбайт, каждый из которых является непрерывным. Такие физические части одного логического файла (фильма) называются экстентами».

**Экстент** — непрерывный участок данных. Один файл может состоять из нескольких экстентов.

### Возвращение идеи в современных ФС

Из tanenbaum.md:
> «Непрерывное размещение благодаря своей простоте и высокой производительности использовалось в файловых системах магнитных дисков много лет назад… Затем из-за необходимости задания конечного размера файла при его создании эта идея была отброшена. Но неожиданно с появлением компакт-дисков, DVD, Blu-ray-дисков и других однократно записываемых оптических носителей непрерывные файлы снова оказались весьма кстати».

### Экстенты в современных ФС

Современные ФС (ext4, NTFS, XFS, Btrfs) используют **экстенты** — описание непрерывных участков:

```
Файл = массив экстентов
Каждый экстент = (start_block, length)
```

Преимущества:
- В пределах экстента — преимущества непрерывности.
- Между экстентами — гибкость.
- Метаданных меньше, чем у списка блоков.

Это **компромисс** между непрерывным размещением и i-узлами.

### Резюме

- Непрерывное размещение: блоки файла подряд.
- Атрибуты: начальный блок + длина (+ обычные).
- O(1) расчёт адреса любого блока.
- Простота и максимальная производительность чтения.
- Недостатки: фрагментация диска, нужно знать размер заранее.
- Применение: CD/DVD/Blu-ray, ленты, архивы.
- Идея жива в виде **экстентов** в современных ФС.

---

## 100. Файловые системы. Размещение файлов с использованием связанного списка. Достоинства и недостатки. Пример. FAT. Достоинства и недостатки. Пример.
*Источник: tanenbaum.md, Глава 4 «Файловые системы»*

### Связанный список

Из tanenbaum.md (4.3.2):
> «Второй метод хранения файлов заключается в представлении каждого файла в виде связанного списка дисковых блоков. Первое слово каждого блока используется в качестве указателя на следующий блок, а вся остальная часть блока предназначается для хранения данных».

### Структура блока

```
+----------+--------------------------------+
| next ptr | данные                          |
+----------+--------------------------------+
   4 байта    blocksize - 4 байт
```

### Иллюстрация

Файл A — блоки 4, 7, 2, 10, 12:

```
Каталог: A → 4

Блок 4:  [→7][data]
Блок 7:  [→2][data]
Блок 2:  [→10][data]
Блок 10: [→12][data]
Блок 12: [EOF][data]
```

В каталоге хранится только начальный блок. Остальное — переходя по цепочке.

### Достоинство 1: Нет фрагментации

Из tanenbaum.md:
> «В отличие от непрерывного размещения, в этом методе может быть использован каждый дисковый блок. При этом потери дискового пространства на фрагментацию отсутствуют (за исключением внутренней фрагментации в последнем блоке)».

Любой свободный блок можно использовать — не нужна непрерывная область.

### Достоинство 2: Простой каталог

В каталоге — только начальный блок. Размер файла можно подсчитать по цепочке.

### Достоинство 3: Динамический рост

Добавить блок в конец — связать новый блок:
```c
last_block->next = new_block_addr;
new_block->next = EOF;
```

### Недостаток 1: Медленный произвольный доступ

Из tanenbaum.md:
> «Чтобы добраться до блока n, операционной системе нужно начать со стартовой позиции и прочитать поочередно n − 1 предшествующих блоков. Понятно, что осуществление стольких операций чтения окажется мучительно медленным».

Для доступа к байту в середине большого файла — **O(n) операций чтения** диска. Катастрофически медленно.

### Недостаток 2: Размер блока не кратен степени 2

Из tanenbaum.md:
> «Объем хранилища данных в блоках уже не кратен степени числа 2, поскольку несколько байтов отнимает указатель… Когда первые несколько байтов каждого блока заняты указателем на следующий блок, чтение полноценного блока требует получения и соединения информации из двух дисковых блоков».

Если блок 1024, указатель 4 байта → данных 1020 байт. Программы обычно работают блоками степени 2 — нужна склейка из двух физических блоков.

### Недостаток 3: Уязвимость

Потеря одного указателя → потеря всего хвоста файла. На диске с повреждёнными секторами это серьёзно.

### FAT — улучшение

Из tanenbaum.md:
> «Оба недостатка размещения с помощью связанных списков могут быть устранены за счёт изъятия слова указателя из каждого дискового блока и помещения его в таблицу в памяти».

Идея: **все указатели в отдельной таблице** в памяти. Блоки целиком — данные.

### Структура FAT

**FAT (File Allocation Table)** — массив записей, по одной на каждый блок диска:

```
FAT[0] = указатель на следующий блок (или EOF, или FREE, или BAD)
FAT[1] = ...
FAT[2] = ...
...
FAT[N] = ...
```

### Значения в FAT

- **Конкретное число (≥ 2):** номер следующего блока.
- **0:** свободен.
- **-1 (или 0xFFFF, 0xFFFFFFF):** EOF.
- **-2:** плохой блок.

### Пример

Файл A: блоки 4, 7, 2, 10, 12.

В каталоге:
```
A → 4
```

В FAT:
```
FAT[4]  = 7
FAT[7]  = 2
FAT[2]  = 10
FAT[10] = 12
FAT[12] = EOF
```

Чтобы прочитать файл — идём по цепочке в FAT (в памяти, быстро), читаем блоки.

### Преимущество 1: Произвольный доступ без I/O

Цепочка вся в памяти. Чтобы найти n-й блок:
- Пройти по FAT n раз.
- Без обращения к диску для поиска цепочки.

O(n) проход в памяти — быстро (миллионы записей в секунду).

### Преимущество 2: Полный размер блока — данные

Указатели не в блоках → блок 1024 байт = 1024 байт данных. Кратно степени 2.

### Преимущество 3: Простота

FAT — компактная и понятная структура. Легко реализовать.

### Недостаток FAT 1: Таблица в памяти

Из tanenbaum.md:
> «Основным недостатком этого метода является то, что для его работы вся таблица должна постоянно находиться в памяти. Для 1-терабайтного диска, имеющего блоки размером 1 Кбайт, потребовалась бы таблица из 1 млрд записей, по одной для каждого из 1 млрд дисковых блоков… таблица будет постоянно занимать 3 Гбайт или 2,4 Гбайт оперативной памяти».

Не масштабируется на большие диски.

### Недостаток FAT 2: Производительность всё ещё O(n)

Без I/O, но проход по цепочке — линейный.

Для больших файлов с произвольным доступом — медленно.

### Версии FAT

**FAT12:**
- 12 бит на запись.
- Максимум 4096 блоков.
- Для дискет 1.44 МБ.

**FAT16:**
- 16 бит на запись.
- Максимум 65 536 блоков.
- MS-DOS, Windows 3.x/95.
- Максимум диск ~4 ГБ (с блоком 64 КБ).

**FAT32:**
- 32 бита на запись (фактически 28).
- Максимум 2²⁸ блоков.
- Windows 95 OSR2+ и далее.
- Размер диска до 8 ТБ (теоретически), на практике ограничивается Microsoft 32 ГБ.
- Максимум файла — 4 ГБ.

**exFAT:**
- Расширение FAT-32.
- Для флеш-накопителей и больших дисков.
- Без многих ограничений FAT-32 (например, размер файла).
- Лицензионная (но Microsoft открыла спецификацию в 2019).

### Применение FAT

- **Дискеты** (FAT12).
- **Жёсткие диски** старых ПК (FAT16/32).
- **USB-флешки** (FAT32, exFAT).
- **SD-карты** (FAT32, exFAT).
- **Карты памяти камер.**
- **EFI System Partition** в UEFI BIOS — обязательно FAT32.
- **PlayStation, Xbox** игровые приставки старых поколений.

### Почему FAT всё ещё популярна

- **Простота:** легко реализовать.
- **Переносимость:** поддерживается Windows, Linux, macOS, всеми ОС.
- **Открытая спецификация** (с недавних пор).
- **Не требует журналирования** (важно для флешек, где запись изнашивает ячейки).

### Сравнение трёх методов

| Параметр | Непрерывный | Связ. список | FAT |
|---|---|---|---|
| Фрагментация | Внешняя | Нет | Нет |
| Произвольный доступ | O(1), без I/O | O(n) с I/O | O(n) без I/O |
| Размер блока кратен 2 | Да | Нет | Да |
| Размер каталога | Малый (start+len) | Очень малый (start) | Очень малый (start) |
| Память для метаданных | 0 | 0 | Пропорц. диску |
| Подходит для CD/DVD | Идеально | Плохо | Плохо |
| Подходит для общих дисков | Плохо | Плохо | Только малых |
| Подходит для больших дисков | Плохо | Плохо | Плохо (память) |

### Эволюция

```
Непрерывное → Связ. список → FAT → i-узлы → Экстенты
```

Каждая итерация решает проблемы предыдущей. i-узлы (вопрос 80) — современный подход для Linux/Unix. Экстенты — современная оптимизация (ext4, XFS, NTFS).

### Резюме

- Связанный список: указатель в каждом блоке. Простота, нет фрагментации, но медленный random access и нестандартный размер блока.
- FAT: указатели в отдельной таблице в RAM. Решает оба недостатка, но требует много RAM для больших дисков.
- FAT популярен на флеш-носителях и в EFI благодаря простоте и переносимости.
- Для серьёзных файловых систем используются i-узлы и экстенты.


<!-- NAV-FOOTER -->

---

**Навигация:** [← Ответы 81-90 (подробно)](answers_81-90_detailed.md) | [🏠 README](README.md) | [📄 Кратко](answers_91-100.md) | [Ответы 101-112 (подробно) →](answers_101-112_detailed.md)
