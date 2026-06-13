# Ответы на вопросы 81–90

---

## 81. Регистрация символьных устройств. cdev_init(&dev->cdev, &scull_fops);
*Источник: Linux Device Drivers (LDD3); robert_lav.md, Глава 13*

### Структура cdev

Символьное устройство в ядре Linux представлено структурой **`struct cdev`**:

```c
struct cdev {
    struct kobject kobj;
    struct module *owner;
    const struct file_operations *ops;
    struct list_head list;
    dev_t dev;
    unsigned int count;
};
```

### Регистрация драйвера — этапы

**1. Выделить номера устройств** (вопросы 77, 78).

```c
dev_t dev;
alloc_chrdev_region(&dev, 0, scull_nr_devs, "scull");
```

**2. Инициализировать структуру cdev.**

```c
cdev_init(&dev->cdev, &scull_fops);
dev->cdev.owner = THIS_MODULE;
dev->cdev.ops = &scull_fops;
```

**3. Зарегистрировать в ядре.**

```c
int err = cdev_add(&dev->cdev, devno, 1);
```

### cdev_init()

```c
void cdev_init(struct cdev *cdev, const struct file_operations *fops);
```

Инициализирует структуру `cdev`:
- Обнуляет поля.
- Устанавливает kobject.
- Связывает с `file_operations`.

### cdev_add()

```c
int cdev_add(struct cdev *p, dev_t dev, unsigned count);
```

- `p` — структура cdev.
- `dev` — стартовый номер устройства.
- `count` — сколько последовательных minor-номеров.

После успешного вызова устройство **доступно** через `/dev`.

### cdev_del()

```c
void cdev_del(struct cdev *p);
```

Удаляет устройство из системы. Вызывается при выгрузке драйвера.

### Полный пример (на основе scull из LDD)

```c
struct scull_dev {
    struct scull_qset *data;
    int quantum;
    int qset;
    unsigned long size;
    unsigned int access_key;
    struct semaphore sem;
    struct cdev cdev;
};

static struct scull_dev *scull_devices;

static void scull_setup_cdev(struct scull_dev *dev, int index)
{
    int err, devno = MKDEV(scull_major, scull_minor + index);

    cdev_init(&dev->cdev, &scull_fops);
    dev->cdev.owner = THIS_MODULE;
    dev->cdev.ops = &scull_fops;
    err = cdev_add(&dev->cdev, devno, 1);
    if (err)
        printk(KERN_NOTICE "Error %d adding scull%d", err, index);
}
```

### Альтернатива: register_chrdev (устаревшая)

Старый интерфейс:
```c
int register_chrdev(unsigned int major, const char *name,
                    const struct file_operations *fops);
```

- Регистрирует major-номер и **сразу 256 minor-номеров**.
- Использует встроенную структуру cdev.
- Менее гибкий, но проще.

Современная практика — `cdev_*` API.

---

## 82. Структура struct scull_qset {void **data; struct scull_qset *next;};
*Источник: LDD3 — пример scull-драйвера*

### Контекст

**scull** (Simple Character Utility for Loading Localities) — учебный драйвер из книги LDD3. Реализует «область памяти» как символьное устройство.

Структура хранения данных — **списочно-массивная**: связанный список «наборов квантов» (quantum sets).

### Определение

```c
struct scull_qset {
    void **data;
    struct scull_qset *next;
};
```

### Структура хранения

```
struct scull_dev:
  ┌─────────┐
  │ data    │ → struct scull_qset → struct scull_qset → ...
  │ quantum │       │ data[]              │ data[]
  │ qset    │       │   ↓ ↓ ↓             │   ↓ ↓ ↓
  │ size    │       │  [Q][Q][Q]          │  [Q][Q][Q]
  └─────────┘       │  [Q][Q][Q]          │  [Q][Q][Q]
                    └ next                └ next ...
                    каждый Q = quantum памяти
```

**Параметры:**
- **`quantum`** — размер одного блока данных (по умолчанию 4000 байт).
- **`qset`** — число квантов в одном scull_qset (по умолчанию 1000).
- **`data`** в qset — массив из `qset` указателей на кванты.

### Преимущества структуры

1. **Динамическое расширение:** не нужно знать размер заранее, добавляем qset по мере записи.
2. **Эффективное удаление:** при truncate просто освобождаем все кванты.
3. **Балансировка:** компактная иерархия указатель → массив указателей → данные.

### Размер одного «уровня»

Один scull_qset держит:
- `qset` указателей × 8 байт (на 64-bit) = 8 КБ.
- `qset` × `quantum` = 1000 × 4000 = **4 МБ данных**.

При записи 100 МБ нужно ~25 qset-узлов в списке.

### Поиск элемента

```c
struct scull_qset *scull_follow(struct scull_dev *dev, int n)
{
    struct scull_qset *qs = dev->data;

    if (!qs) {
        qs = dev->data = kmalloc(sizeof(struct scull_qset), GFP_KERNEL);
        if (qs == NULL) return NULL;
        memset(qs, 0, sizeof(struct scull_qset));
    }

    while (n--) {
        if (!qs->next) {
            qs->next = kmalloc(sizeof(struct scull_qset), GFP_KERNEL);
            if (qs->next == NULL) return NULL;
            memset(qs->next, 0, sizeof(struct scull_qset));
        }
        qs = qs->next;
    }
    return qs;
}
```

### Освобождение

```c
int scull_trim(struct scull_dev *dev)
{
    struct scull_qset *next, *dptr;
    int qset = dev->qset, i;

    for (dptr = dev->data; dptr; dptr = next) {
        if (dptr->data) {
            for (i = 0; i < qset; i++)
                kfree(dptr->data[i]);
            kfree(dptr->data);
            dptr->data = NULL;
        }
        next = dptr->next;
        kfree(dptr);
    }
    dev->size = 0;
    dev->quantum = scull_quantum;
    dev->qset = scull_qset;
    dev->data = NULL;
    return 0;
}
```

### Расчёт позиции в read/write

При запросе байта по позиции pos:
- `item` = pos / (qset × quantum) — номер scull_qset.
- `rest` = pos % (qset × quantum).
- `s_pos` = rest / quantum — номер кванта в qset.
- `q_pos` = rest % quantum — позиция в кванте.

```c
item = (long)*f_pos / itemsize;
rest = (long)*f_pos % itemsize;
s_pos = rest / quantum;
q_pos = rest % quantum;
```

---

## 83. Символьное устройство методы open и release.
*Источник: LDD3*

### open

```c
int (*open)(struct inode *inode, struct file *filp);
```

Вызывается при `open(2)` от пользователя. Назначение:
- Проверить право доступа.
- Инициализировать устройство, если первый open.
- Установить `filp->private_data` для удобства последующих вызовов.
- Идентифицировать конкретное устройство по minor.

### Пример

```c
int scull_open(struct inode *inode, struct file *filp)
{
    struct scull_dev *dev;

    dev = container_of(inode->i_cdev, struct scull_dev, cdev);
    filp->private_data = dev;

    // Если открыт для записи и не для добавления — очистить
    if ((filp->f_flags & O_ACCMODE) == O_WRONLY) {
        if (down_interruptible(&dev->sem))
            return -ERESTARTSYS;
        scull_trim(dev);
        up(&dev->sem);
    }
    return 0;
}
```

### container_of

Макрос для получения **содержащей структуры** по полю:
```c
container_of(ptr, type, member)
```

Здесь: `inode->i_cdev` указывает на `cdev` внутри `scull_dev`. По нему получаем `scull_dev`.

### release

```c
int (*release)(struct inode *inode, struct file *filp);
```

Вызывается при **последнем** `close(2)`. Назначение:
- Освободить ресурсы, выделенные в open.
- Завершить операции.
- Закрыть устройство, если ни один процесс больше не использует.

### Пример

```c
int scull_release(struct inode *inode, struct file *filp)
{
    return 0;   // scull простой, ничего не делает
}
```

### Особенности

- **release вызывается не на каждый close** — только когда `f_count` дескриптора падает до 0.
- При `fork` дескрипторы делятся → release вызовется только когда оба процесса закроют.
- В **flush** — на каждый close (см. вопрос 79).

### Множественный open

Если несколько процессов открывают одно устройство:
- `open` вызывается каждый раз.
- Создаются разные `struct file`, но один `inode`.
- `release` — только при последнем close.

---

## 84. Символьное устройство методы read и write.
*Источник: LDD3*

### Сигнатуры

```c
ssize_t (*read) (struct file *filp, char __user *buf,
                 size_t count, loff_t *f_pos);
ssize_t (*write) (struct file *filp, const char __user *buf,
                  size_t count, loff_t *f_pos);
```

- `buf` — буфер пользователя (помечен `__user`).
- `count` — сколько байт запрашивается.
- `f_pos` — текущая позиция в файле (обновляется методом).
- Возврат: число фактически переданных байт (может быть < count).

### read для scull

```c
ssize_t scull_read(struct file *filp, char __user *buf,
                   size_t count, loff_t *f_pos)
{
    struct scull_dev *dev = filp->private_data;
    struct scull_qset *dptr;
    int quantum = dev->quantum, qset = dev->qset;
    int itemsize = quantum * qset;
    int item, s_pos, q_pos, rest;
    ssize_t retval = 0;

    if (down_interruptible(&dev->sem))
        return -ERESTARTSYS;
    if (*f_pos >= dev->size)
        goto out;
    if (*f_pos + count > dev->size)
        count = dev->size - *f_pos;

    item = (long)*f_pos / itemsize;
    rest = (long)*f_pos % itemsize;
    s_pos = rest / quantum;
    q_pos = rest % quantum;

    dptr = scull_follow(dev, item);
    if (!dptr || !dptr->data || !dptr->data[s_pos])
        goto out;

    if (count > quantum - q_pos)
        count = quantum - q_pos;

    if (copy_to_user(buf, dptr->data[s_pos] + q_pos, count)) {
        retval = -EFAULT;
        goto out;
    }
    *f_pos += count;
    retval = count;

  out:
    up(&dev->sem);
    return retval;
}
```

### Ключевые моменты

**1. Захват семафора.** `down_interruptible` блокирует поток до получения семафора, но возвращает `-ERESTARTSYS` при сигнале.

**2. copy_to_user / copy_from_user.** Безопасное копирование между ядром и пользователем (вопрос 33).

**3. Возврат -EFAULT** при ошибке доступа к пользовательской памяти.

**4. Обновление f_pos.** Метод должен обновить позицию.

**5. Возможность частичного чтения.** Если нашли меньше байт — вернуть, что есть. Пользователь повторит.

### write для scull

Симметричен read:
- Растёт размер при необходимости.
- Выделяет новые qset/кванты.
- copy_from_user.

```c
ssize_t scull_write(struct file *filp, const char __user *buf,
                    size_t count, loff_t *f_pos)
{
    struct scull_dev *dev = filp->private_data;
    /* ... аналогично read, но с copy_from_user */
    /* + выделение памяти если нужно */
}
```

### Возвращаемые значения

- **> 0:** число переданных байт.
- **0:** конец файла (для read).
- **-EAGAIN:** неблокирующее чтение, данных нет.
- **-EFAULT:** ошибка доступа к памяти пользователя.
- **-EINTR:** прерывание сигналом.
- **-ENOSPC:** нет места.
- **-EIO:** ошибка устройства.

---

## 85. Символьное устройство метод ioctrl. Передача параметров. Возвращаемое значение.
*Источник: LDD3*

### Назначение ioctl

Системный вызов `ioctl()` — для **управления устройством**, не подходящего под read/write:
- Установка параметров.
- Получение статуса.
- Специфичные для устройства команды.

### Сигнатура

В старых ядрах:
```c
int (*ioctl) (struct inode *inode, struct file *filp,
              unsigned int cmd, unsigned long arg);
```

В современных (с 2.6.36):
```c
long (*unlocked_ioctl) (struct file *filp, unsigned int cmd, unsigned long arg);
long (*compat_ioctl) (struct file *filp, unsigned int cmd, unsigned long arg);
```

`unlocked_ioctl` — без BKL. `compat_ioctl` — для 32-битных программ на 64-битном ядре.

### Передача параметров

**Со стороны пользователя:**
```c
ioctl(fd, CMD, arg);
```

- `cmd` — номер команды (произвольное целое).
- `arg` — параметр. Может быть значением или указателем.

### Кодирование cmd

В Linux рекомендуется кодировать cmd по специальному формату для уникальности:

```c
#include <asm/ioctl.h>

_IO(type, nr)              // без данных
_IOR(type, nr, datatype)   // read: ядро → пользователь
_IOW(type, nr, datatype)   // write: пользователь → ядро
_IOWR(type, nr, datatype)  // оба направления
```

- `type` — magic number драйвера (для уникальности).
- `nr` — номер команды.
- `datatype` — тип данных.

Пример:
```c
#define SCULL_IOC_MAGIC 'k'
#define SCULL_IOCRESET    _IO(SCULL_IOC_MAGIC, 0)
#define SCULL_IOCSQUANTUM _IOW(SCULL_IOC_MAGIC, 1, int)
#define SCULL_IOCGQUANTUM _IOR(SCULL_IOC_MAGIC, 2, int)
```

### Декодирование cmd

```c
_IOC_DIR(cmd)    // направление: _IOC_NONE/READ/WRITE
_IOC_TYPE(cmd)   // type (magic)
_IOC_NR(cmd)     // номер команды
_IOC_SIZE(cmd)   // размер данных
```

### Реализация ioctl

```c
long scull_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int err = 0, retval = 0;

    if (_IOC_TYPE(cmd) != SCULL_IOC_MAGIC) return -ENOTTY;
    if (_IOC_NR(cmd) > SCULL_IOC_MAXNR) return -ENOTTY;

    if (_IOC_DIR(cmd) & _IOC_READ)
        err = !access_ok(VERIFY_WRITE, (void __user *)arg, _IOC_SIZE(cmd));
    else if (_IOC_DIR(cmd) & _IOC_WRITE)
        err = !access_ok(VERIFY_READ, (void __user *)arg, _IOC_SIZE(cmd));
    if (err) return -EFAULT;

    switch (cmd) {
        case SCULL_IOCRESET:
            scull_quantum = SCULL_QUANTUM;
            scull_qset = SCULL_QSET;
            break;

        case SCULL_IOCSQUANTUM:   // SET (передача int от пользователя)
            if (!capable(CAP_SYS_ADMIN)) return -EPERM;
            retval = __get_user(scull_quantum, (int __user *)arg);
            break;

        case SCULL_IOCGQUANTUM:   // GET (передача int пользователю)
            retval = __put_user(scull_quantum, (int __user *)arg);
            break;

        default:
            return -ENOTTY;
    }
    return retval;
}
```

### Возвращаемое значение

- **0:** успех.
- **Положительное:** успех с информацией (зависит от команды).
- **-ENOTTY:** неизвестная команда (стандартный код).
- **-EFAULT:** ошибка доступа.
- **-EPERM:** нет прав.
- **-EINVAL:** некорректные параметры.

`-ENOTTY` означает «inappropriate ioctl for device» — используется как для незарегистрированных команд, так и для несовместимых устройств.

---

## 86. Символьное устройство метод ioctrl. Разрешения и запрещённые операции. FIOASYNC
*Источник: LDD3*

### Проверка разрешений

Многие ioctl-команды требуют привилегий. Стандартные проверки:

**capable(CAP_*).**
```c
if (!capable(CAP_SYS_ADMIN))
    return -EPERM;
```

Common capabilities:
- `CAP_SYS_ADMIN` — общие администраторские привилегии.
- `CAP_NET_ADMIN` — управление сетью.
- `CAP_DAC_OVERRIDE` — обход прав доступа.
- `CAP_RAWIO` — прямой ввод-вывод.

**Чтение/запись fd.**

Если ioctl модифицирует состояние, может потребоваться, чтобы fd был открыт на запись:
```c
if (!(filp->f_mode & FMODE_WRITE))
    return -EPERM;
```

### Запрещённые операции

**ENOTTY** — стандартный возврат для:
- Неизвестной команды (cmd).
- Команды, не предназначенной для этого устройства.

Старое сообщение от tty: «Not a typewriter» — отсюда название.

### Стандартные ioctl-команды

Некоторые ioctl стандартизированы и могут быть применимы к любому устройству:

**FIONBIO** (File IO Non-Blocking IO) — включить/выключить неблокирующий режим.
```c
ioctl(fd, FIONBIO, &on);
```

**FIONREAD** — узнать, сколько байт доступно для чтения без блокировки.

**FIOQSIZE** — размер файла.

**FIOASYNC** — управление асинхронным режимом.

### FIOASYNC

Включает/выключает асинхронный режим, в котором сигнал `SIGIO` приходит процессу при готовности устройства.

```c
ioctl(fd, FIOASYNC, &on);
```

Если включено, при изменении состояния устройства (доступны данные для чтения, есть место для записи) ядро посылает `SIGIO` процессу-владельцу.

### Реализация FIOASYNC

Драйвер должен реализовать метод **fasync**:
```c
int (*fasync) (int fd, struct file *filp, int mode);
```

В fasync — управление списком процессов для уведомления:
```c
int scull_p_fasync(int fd, struct file *filp, int mode)
{
    struct scull_pipe *dev = filp->private_data;
    return fasync_helper(fd, filp, mode, &dev->async_queue);
}
```

При наступлении события — `kill_fasync()`:
```c
if (dev->async_queue)
    kill_fasync(&dev->async_queue, SIGIO, POLL_IN);
```

### Альтернатива FIOASYNC

Современный подход — **epoll/poll/select** — позволяет ждать несколько fd сразу. Заменяет SIGIO в большинстве случаев.

### Запрещённые модификации в ioctl

В реализации ioctl нужно:
- **Не использовать глобальное состояние без блокировок.**
- **Не блокироваться надолго без необходимости.**
- **Не выполнять опасные операции без проверки прав.**
- **Использовать `access_ok` / `copy_*_user`** для безопасности.

### Резюме

- ioctl-команды кодируются через `_IO`, `_IOR`, `_IOW`, `_IOWR`.
- Возврат: 0 / положительное при успехе; `-ENOTTY` для неизвестных; `-EPERM`, `-EFAULT` при ошибках.
- Стандартные команды: FIONBIO, FIONREAD, FIOASYNC.
- FIOASYNC включает асинхронный режим — сигнал SIGIO при изменении состояния.
- Современная замена SIGIO — epoll.

---

## 87. Использование семафоров в scull. Семафоры по чтению и записи. void downgrade_write(struct rw_semaphore *sem);
*Источник: LDD3; robert_lav.md, Глава 10*

### Зачем семафор в scull

В scull есть **общее состояние**:
- Связный список qset.
- Размер.
- Параметры (quantum, qset).

Если два процесса одновременно вызовут `write` или один — `write`, другой — `read`, без синхронизации возможна порча данных.

### Семафор в scull_dev

```c
struct scull_dev {
    struct scull_qset *data;
    int quantum;
    int qset;
    unsigned long size;
    struct semaphore sem;    // ← защита
    struct cdev cdev;
};
```

Инициализация:
```c
sema_init(&dev->sem, 1);    // count = 1 — бинарный
```

### Использование в open/read/write

```c
if (down_interruptible(&dev->sem))
    return -ERESTARTSYS;
// CS — работа со структурой
up(&dev->sem);
```

### down_interruptible

Возвращает `-EINTR` при сигнале → драйвер должен вернуть `-ERESTARTSYS`, чтобы системный вызов был переначат (если возможно).

### Семафоры по чтению-записи

Если устройство read-mostly — можно использовать RW-семафор для параллельного чтения нескольких потоков.

```c
struct rw_semaphore rwsem;
init_rwsem(&rwsem);

// Читатель
down_read(&rwsem);
// чтение
up_read(&rwsem);

// Писатель
down_write(&rwsem);
// запись
up_write(&rwsem);
```

### downgrade_write

```c
void downgrade_write(struct rw_semaphore *sem);
```

Атомарно превращает блокировку записи в блокировку чтения. Полезно, когда:
1. Захватили writer-lock для модификации.
2. Сделали модификацию.
3. Дальше нужно только читать — пусть и другие читатели работают параллельно.

```c
down_write(&rwsem);
modify_data();
downgrade_write(&rwsem);
// теперь другие читатели могут пройти
read_some_more();
up_read(&rwsem);
```

Без downgrade пришлось бы:
```c
up_write(&rwsem);
down_read(&rwsem);   // ← между этим возможна гонка!
```

### Атомарность downgrade

`downgrade_write` гарантирует, что **между** write-lock и read-lock никто не сможет «вклиниться»:
- Не пройдёт writer.
- Read-lock сразу становится валидным.

### Применение в scull

В scull обычно используется обычный семафор (бинарный) — структура достаточно компактная, RW-семафор был бы избыточен.

В вариациях (scull pipe, scullp) используются дополнительные примитивы.

---

## 88. Блокирующий ввод/вывод. Очереди ожидания. Состояния процесса. wait_queue_head_t my_queue;
*Источник: robert_lav.md, Глава 4 «Очереди ожидания»; LDD3*

### Блокирующий ввод/вывод

Когда устройство не готово (нет данных для read, нет места для write), драйвер должен **заблокировать** процесс до готовности.

### Альтернатива — поллинг

Плохо: процесс крутит CPU. Решение — **очередь ожидания**.

### Очередь ожидания

```c
#include <linux/wait.h>

wait_queue_head_t my_queue;
init_waitqueue_head(&my_queue);
```

Или статически:
```c
DECLARE_WAIT_QUEUE_HEAD(my_queue);
```

### Состояния процесса

Из robert_lav.md (Глава 4):
- **`TASK_RUNNING`** — работает или готов.
- **`TASK_INTERRUPTIBLE`** — спит, разбудит сигнал или wake_up.
- **`TASK_UNINTERRUPTIBLE`** — спит, только wake_up разбудит. Нельзя прервать сигналом.

### Засыпание

Основной способ — через `wait_event_*`:

```c
wait_event(queue, condition);                    // TASK_UNINTERRUPTIBLE
wait_event_interruptible(queue, condition);       // можно сигналом
wait_event_timeout(queue, condition, timeout);    // с таймаутом
wait_event_interruptible_timeout(queue, condition, timeout);
```

### Пробуждение

```c
wake_up(&queue);              // будит TASK_UNINTERRUPTIBLE и INTERRUPTIBLE
wake_up_interruptible(&queue); // только INTERRUPTIBLE
wake_up_all(&queue);          // всех
```

### Пример из LDD: блокирующий read

```c
ssize_t my_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
{
    struct my_dev *dev = filp->private_data;

    if (down_interruptible(&dev->sem))
        return -ERESTARTSYS;

    while (dev->size == 0) {   // нет данных
        up(&dev->sem);

        if (filp->f_flags & O_NONBLOCK)
            return -EAGAIN;

        if (wait_event_interruptible(dev->inq, dev->size > 0))
            return -ERESTARTSYS;

        if (down_interruptible(&dev->sem))
            return -ERESTARTSYS;
    }

    // данные есть — копировать
    // ...
    up(&dev->sem);
    return count;
}
```

### Пример: разбудить из write или interrupt

```c
ssize_t my_write(...)
{
    // ... записать данные ...
    wake_up_interruptible(&dev->inq);   // разбудить ожидающих read
    return count;
}
```

### Правильный паттерн

Из robert_lav.md, Глава 4 (вопрос 23):
- Цикл с проверкой условия (от spurious wakeup).
- Проверка сигнала.
- Освобождение блокировок перед schedule.

`wait_event_interruptible` — это готовый паттерн:

```c
#define wait_event_interruptible(wq, condition) \
({ \
    int __ret = 0; \
    if (!(condition)) \
        __wait_event_interruptible(wq, condition, __ret); \
    __ret; \
})
```

---

## 89. Пример блокирующего ввода/вывода. Подробности «засыпания». wait_event_interruptible_timeout(queue, condition, timeout)
*Источник: robert_lav.md, Глава 4; LDD3*

### wait_event_interruptible_timeout

```c
long wait_event_interruptible_timeout(wait_queue_head_t wq,
                                       condition,
                                       long timeout);
```

Параметры:
- `wq` — очередь ожидания.
- `condition` — условие, выражение C.
- `timeout` — в jiffies.

Возвращает:
- **> 0:** условие выполнено, оставшееся время в jiffies.
- **0:** таймаут.
- **-ERESTARTSYS:** прерывание сигналом.

### Пример

```c
long ret = wait_event_interruptible_timeout(dev->wq, dev->ready, HZ * 5);
if (ret == 0)
    return -ETIMEDOUT;
if (ret < 0)
    return ret;   // сигнал
// условие выполнено
```

`HZ * 5` — 5 секунд (HZ = тиков в секунду).

### Что происходит при засыпании

Из robert_lav.md (Глава 4):

1. **Добавление в очередь:**
```c
DEFINE_WAIT(wait);
add_wait_queue(&q, &wait);
```

2. **Изменение состояния:**
```c
prepare_to_wait(&q, &wait, TASK_INTERRUPTIBLE);
```

Атомарно: задача в очереди + состояние INTERRUPTIBLE.

3. **Проверка условия:**
```c
if (condition) break;
```

Если уже выполнено — выйти без сна.

4. **Проверка сигнала:**
```c
if (signal_pending(current)) {
    ret = -ERESTARTSYS;
    break;
}
```

5. **Уход на покой:**
```c
schedule();        // или schedule_timeout(timeout);
```

`schedule_timeout` ставит таймер и засыпает; при таймауте просыпается.

6. **Возврат:**
```c
finish_wait(&q, &wait);
```

### Альтернатива — ручная реализация

```c
ssize_t my_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
{
    struct my_dev *dev = filp->private_data;
    DEFINE_WAIT(wait);
    long timeo = HZ * 5;

    add_wait_queue(&dev->inq, &wait);
    while (1) {
        prepare_to_wait(&dev->inq, &wait, TASK_INTERRUPTIBLE);
        if (dev->size > 0)
            break;
        if (signal_pending(current)) {
            finish_wait(&dev->inq, &wait);
            return -ERESTARTSYS;
        }
        timeo = schedule_timeout(timeo);
        if (timeo == 0) {
            finish_wait(&dev->inq, &wait);
            return -ETIMEDOUT;
        }
    }
    finish_wait(&dev->inq, &wait);

    // данные есть — обработать
    // ...
}
```

### Эксклюзивное ожидание

При `wake_up` обычно будятся **все** ожидающие. Это часто избыточно (thundering herd). Решение — **эксклюзивное** ожидание:

```c
prepare_to_wait_exclusive(&q, &wait, TASK_INTERRUPTIBLE);
```

Будит только **одного** ожидающего (помеченного эксклюзивным). Используется в очередях задач.

### Гонки и spurious wakeup

Из robert_lav.md:
- Цикл с проверкой условия защищает от spurious wakeup.
- `prepare_to_wait` атомарно ставит в очередь и меняет состояние — нет lost wakeup.

### Часть драйвера пробуждает

Когда событие произошло (например, данные пришли в обработчике прерывания):
```c
dev->size += received;
wake_up_interruptible(&dev->inq);
```

---

## 90. Функция malloc(), vmalloc() и kmalloc(). Основные отличия и назначение.
*Источник: robert_lav.md, Глава 12 «Управление памятью»*

### malloc — пользовательское пространство

```c
void *malloc(size_t size);
void free(void *ptr);
```

Стандарт C. Управляет **кучей** процесса:
- В user space.
- Виртуально непрерывная память.
- Физически — необязательно непрерывная.
- Может вызывать `brk` / `mmap` системные вызовы.
- Размер: до десятков ГБ (зависит от АП).

### kmalloc — ядерное пространство, физически непрерывно

Из robert_lav.md (Глава 12):
> «Функция `kmalloc()` аналогична функции `malloc()`, с которой разработчики пользовательских приложений уже знакомы, за исключением того, что в неё добавлен ещё один параметр `flags`».

```c
#include <linux/slab.h>
void *kmalloc(size_t size, gfp_t flags);
void kfree(const void *ptr);
```

Возвращает **физически и виртуально непрерывный** блок.

### Флаги kmalloc

- **`GFP_KERNEL`** — обычное, может блокировать.
- **`GFP_ATOMIC`** — нельзя блокировать (для IRQ, спинлоков).
- **`GFP_USER`** — для аллокаций от имени пользователя.
- **`GFP_DMA`** — DMA-совместимая память.

### Когда блокирует

`GFP_KERNEL` может вызывать:
- Освобождение страниц через writeout.
- Очистку кэшей.
- Может занять миллисекунды.

`GFP_ATOMIC` — никогда не блокирует, но может failed.

### Ограничения kmalloc

- **Максимальный размер:** обычно 128 КБ (зависит от ядра).
- **Округление вверх:** до следующей степени 2 (16, 32, 64, ..., 65536).
- Не годится для очень больших аллокаций.

### Использование

```c
struct dog *p = kmalloc(sizeof(struct dog), GFP_KERNEL);
if (!p)
    return -ENOMEM;
// ... использовать p ...
kfree(p);
```

### vmalloc — ядро, виртуально непрерывно

Из robert_lav.md (Глава 12):
> «Функция `vmalloc()` аналогична функции `kmalloc()`, за исключением того, что она выделяет смежные страницы виртуальной памяти, которые необязательно являются смежными физически».

```c
#include <linux/vmalloc.h>
void *vmalloc(unsigned long size);
void vfree(const void *addr);
```

### Отличия kmalloc vs vmalloc

| Параметр | kmalloc | vmalloc |
|---|---|---|
| Физически непрерывная | Да | Нет |
| Виртуально непрерывная | Да | Да |
| Максимум | ~128 КБ | Сотни МБ |
| Скорость | Быстро | Медленно (изменение таблиц страниц) |
| Производительность TLB | Хорошая | Хуже |
| Для DMA | Можно | Нельзя |
| Блокирующая | Зависит от флага | Всегда блокирует |

### Когда использовать vmalloc

Из robert_lav.md:
> «Функция `vmalloc()` используется только тогда, когда это абсолютно необходимо, как правило, для выделения очень больших областей памяти. Например, при динамической загрузке модулей ядра они загружаются в память, которая выделяется с помощью функции `vmalloc()`».

Применение:
- Загрузка модулей ядра.
- Большие буферы, не для DMA.
- Когда непрерывная физическая память недоступна (фрагментация).

### Почему kmalloc обычно лучше

Из robert_lav.md:
> «Несмотря на то что физически смежные страницы памяти необходимы только в определённых случаях, в большей части кода ядра используется для выделения памяти функция `kmalloc()`, а не `vmalloc()`. Это делается в основном из соображений производительности. Для того чтобы физически несмежные страницы памяти сделать смежными в виртуальном адресном пространстве, функция `vmalloc()` должна соответствующим образом сформировать элементы таблицы страниц».

Vmalloc:
- Сложнее реализация.
- Хуже TLB performance (каждая страница в TLB отдельно).
- Медленнее.

### Сравнительная таблица

| Параметр | malloc | kmalloc | vmalloc |
|---|---|---|---|
| Пространство | User | Kernel | Kernel |
| Физически непрерывно | Нет | Да | Нет |
| Виртуально непрерывно | Да | Да | Да |
| Максимум | Гигабайты | ~128 КБ | Большой |
| Может блокировать | Может (page fault) | Зависит от GFP | Да |
| Применение | Программы | Большинство в ядре | Большие буферы, модули |

### Применение

- **kmalloc** — 95% случаев в ядре.
- **vmalloc** — для очень больших аллокаций или модулей.
- **alloc_pages** — для целых страниц (низкий уровень).
- **kmem_cache_alloc** — для частых аллокаций фиксированного размера (вопрос 92).
