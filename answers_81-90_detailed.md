# Ответы на вопросы 81–90 (подробная версия)

Эта версия рассчитана на читателя, который встречает темы впервые. Каждый ответ начинается с мотивации, разбирается механизм по шагам, с примерами и контекстом.

---

## 81. Регистрация символьных устройств. cdev_init(&dev->cdev, &scull_fops);
*Источник: Linux Device Drivers (LDD3); robert_lav.md, Глава 13*

### Что такое scull

**scull** (Simple Character Utility for Loading Localities) — учебный драйвер из книги «Linux Device Drivers, 3rd Edition» (LDD3). Эмулирует символьное устройство, по сути работающее как область памяти. Используется во всех последующих вопросах батча как наглядный пример.

### Зачем нужна регистрация

Драйвер должен «представиться» ядру:
- Какие номера устройств он обслуживает (вопросы 77-78).
- Какие методы он реализует (read, write, open, ...).
- Какое имя у устройства.

Только после регистрации устройство становится доступным через `/dev`.

### Структура cdev

В Linux символьное устройство представлено структурой `struct cdev`:

```c
struct cdev {
    struct kobject kobj;             // для sysfs
    struct module *owner;            // владелец-модуль
    const struct file_operations *ops; // методы (read, write, ...)
    struct list_head list;
    dev_t dev;                       // диапазон номеров
    unsigned int count;              // число minor номеров
};
```

### Этапы регистрации

**Этап 1: Получить номера устройств.**

```c
dev_t dev;
alloc_chrdev_region(&dev, 0, scull_nr_devs, "scull");
```

**Этап 2: Инициализировать cdev.**

```c
cdev_init(&dev->cdev, &scull_fops);
dev->cdev.owner = THIS_MODULE;
```

**Этап 3: Зарегистрировать в ядре.**

```c
int err = cdev_add(&dev->cdev, devno, 1);
```

После этого ядро «знает» о новом устройстве и может вызывать его методы.

### cdev_init() — подробно

```c
void cdev_init(struct cdev *cdev, const struct file_operations *fops);
```

Что делает:
1. Обнуляет структуру cdev.
2. Инициализирует kobject.
3. Связывает с таблицей операций fops.

После cdev_init:
```c
dev->cdev.ops = fops;    // методы
dev->cdev.owner = NULL;  // нужно установить отдельно!
```

Обычно сразу же:
```c
dev->cdev.owner = THIS_MODULE;
```

`THIS_MODULE` — макрос, указывающий на структуру текущего модуля. Это нужно для:
- Подсчёта ссылок на модуль (нельзя выгрузить пока используется).
- Корректной работы при выгрузке.

### Альтернативный способ инициализации

Вместо встроенной cdev можно выделить отдельно:
```c
struct cdev *my_cdev = cdev_alloc();
my_cdev->ops = &my_fops;
my_cdev->owner = THIS_MODULE;
```

Используется реже — обычно cdev — поле в struct устройства.

### cdev_add() — регистрация

```c
int cdev_add(struct cdev *p, dev_t dev, unsigned count);
```

Параметры:
- `p` — указатель на cdev.
- `dev` — первый номер устройства (dev_t).
- `count` — сколько последовательных minor-номеров обслуживает этот cdev.

Что делает:
- Добавляет cdev в глобальную хеш-таблицу cdev'ов.
- Связывает диапазон номеров с этим cdev.

После успешного `cdev_add` ядро может:
- При `open` по соответствующему `/dev/...` найти cdev.
- Вызвать его методы.

### cdev_del() — удаление

```c
void cdev_del(struct cdev *p);
```

Вызывается при выгрузке модуля. Удаляет cdev из системы.

**Важно:** до cdev_del необходимо убедиться, что нет активных пользователей. После cdev_del нельзя обращаться к cdev.

### Полный пример (scull-стиль)

```c
struct scull_dev {
    struct scull_qset *data;     // данные
    int quantum;
    int qset;
    unsigned long size;
    struct semaphore sem;
    struct cdev cdev;            // встроенная cdev
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

В init-функции:
```c
for (i = 0; i < scull_nr_devs; i++) {
    scull_devices[i].quantum = scull_quantum;
    scull_devices[i].qset = scull_qset;
    sema_init(&scull_devices[i].sem, 1);
    scull_setup_cdev(&scull_devices[i], i);
}
```

### Альтернатива: register_chrdev (старый API)

Старый интерфейс:
```c
int register_chrdev(unsigned int major, const char *name,
                    const struct file_operations *fops);
```

- Регистрирует диапазон **256 minor номеров** (весь major).
- Использует внутреннюю cdev.
- Менее гибкий.

Сейчас рекомендуется использовать `alloc_chrdev_region + cdev_init + cdev_add`.

### Резюме

- Регистрация символьного устройства — 3 шага: выделить номера, инициализировать cdev, добавить cdev.
- `cdev_init` связывает cdev с file_operations.
- `cdev_add` делает устройство доступным.
- `cdev_del` удаляет при выгрузке.
- Старый `register_chrdev` заменён современным API.

---

## 82. Структура struct scull_qset {void **data; struct scull_qset *next;};
*Источник: LDD3 — пример scull-драйвера*

### Зачем особая структура для scull

scull — учебный драйвер из LDD3, демонстрирующий работу с устройством-памятью. Нужно эффективно хранить данные **произвольного** размера, выделять/освобождать по мере необходимости.

Простое решение — `kmalloc(N)` — плохо:
- N не известно заранее.
- Большие N не получишь через kmalloc.
- Realloc неудобен.

**Решение:** связанный список **массивов** указателей на блоки.

### Структура хранения

Три уровня:

**Уровень 1: scull_dev** — описание устройства.
**Уровень 2: scull_qset** — узел списка с массивом указателей.
**Уровень 3: quantum** — собственно блоки данных.

```c
struct scull_dev {
    struct scull_qset *data;     // голова списка
    int quantum;                 // размер блока
    int qset;                    // элементов в массиве указателей
    unsigned long size;          // фактический размер данных
    struct semaphore sem;
    struct cdev cdev;
};

struct scull_qset {
    void **data;                 // массив указателей на quantum'ы
    struct scull_qset *next;     // следующий узел
};
```

### Иллюстрация

```
scull_dev:
  data ─────┐
  quantum   │
  qset      │
  size      │
            ↓
  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  │ scull_qset  │ →  │ scull_qset  │ →  │ scull_qset  │ → NULL
  │  data ────┐ │    │  data ────┐ │    │  data ────┐ │
  └───────────│─┘    └───────────│─┘    └───────────│─┘
              ↓                  ↓                  ↓
  [Q0][Q1][Q2][Q3]...  [Q0][Q1][Q2]...  [Q0]...
   ↓
  данные   данные   данные   данные ...
  (quantum байт каждый)
```

### Параметры по умолчанию

- `quantum = 4000` (~4 КБ — размер блока).
- `qset = 1000` — указателей в массиве.

Один scull_qset содержит:
- 1000 указателей × 8 байт (на 64-bit) = 8 КБ массив.
- Может указывать на 1000 квантов × 4000 байт = **4 МБ данных**.

### Преимущества структуры

**1. Динамическое расширение.**

При записи за пределы текущего размера:
- Добавляется новый scull_qset.
- В нём — массив указателей.
- Кванты выделяются по мере необходимости.

**2. Память выделяется только по факту.**

Если запись в позицию 1M, то:
- Кванты для первой 1M существуют.
- Дальше — указатели NULL, память не выделена.

**3. Эффективное освобождение.**

При truncate (`O_TRUNC`) или ioctl reset — пройти весь список, освободить все кванты.

### Поиск элемента — scull_follow

Чтобы найти scull_qset под номером n:

```c
struct scull_qset *scull_follow(struct scull_dev *dev, int n)
{
    struct scull_qset *qs = dev->data;

    if (!qs) {
        // первая запись — выделить голову
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

Логика:
1. Если списка нет — создать голову.
2. Идти по списку n раз.
3. Если на пути нет узла — создать.

### Расчёт позиции

Чтобы по байтовой позиции pos найти конкретный байт:

```c
int itemsize = quantum * qset;   // байт на один scull_qset
int item    = pos / itemsize;     // номер scull_qset
int rest    = pos % itemsize;     // байт внутри scull_qset
int s_pos   = rest / quantum;     // номер кванта
int q_pos   = rest % quantum;     // позиция в кванте
```

Пример: pos = 5_000_000, quantum=4000, qset=1000:
- itemsize = 4_000_000.
- item = 1 (второй scull_qset).
- rest = 1_000_000.
- s_pos = 250 (251-й квант).
- q_pos = 0.

### Освобождение — scull_trim

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
    dev->data = NULL;
    return 0;
}
```

Идёт по списку, освобождает:
1. Каждый квант (kfree dptr->data[i]).
2. Массив указателей (kfree dptr->data).
3. Сам scull_qset (kfree dptr).

### Преимущества и недостатки

**+** Динамическое расширение.
**+** Память пропорциональна реально используемой.
**+** Простая логика.

**−** Доступ — O(n) по списку (для очень больших файлов медленно).
**−** Накладные расходы (заголовки kmalloc, указатели).

### Применение в учебных целях

Структура demonstrates:
- Динамическое управление памятью в ядре.
- Применение kmalloc.
- Сложные структуры данных в драйверах.
- Обработка ошибок (NULL проверки).

В реальных драйверах используются разные подходы, но идеи (динамическая структура, освобождение в trim) общие.

---

## 83. Символьное устройство методы open и release.
*Источник: LDD3*

### Где open и release в file_operations

```c
struct file_operations {
    // ...
    int (*open) (struct inode *, struct file *);
    int (*release) (struct inode *, struct file *);
    // ...
};
```

### open — что делает

Вызывается, когда пользователь делает `open(2)` на устройстве. Задачи:

1. **Проверить устройство** (готово ли, не сломано ли).
2. **Инициализировать**, если первый open.
3. **Идентифицировать** конкретное устройство по minor.
4. **Установить `filp->private_data`** для удобства последующих read/write.
5. **Обработать флаги** (O_RDONLY, O_WRONLY, O_TRUNC, ...).

### Пример: scull_open

```c
int scull_open(struct inode *inode, struct file *filp)
{
    struct scull_dev *dev;

    // Найти устройство по cdev
    dev = container_of(inode->i_cdev, struct scull_dev, cdev);
    filp->private_data = dev;

    // Если открыт только на запись и без O_APPEND — очистить
    if ((filp->f_flags & O_ACCMODE) == O_WRONLY) {
        if (down_interruptible(&dev->sem))
            return -ERESTARTSYS;
        scull_trim(dev);
        up(&dev->sem);
    }
    return 0;
}
```

### container_of — магический макрос

```c
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
```

Дано: указатель на поле структуры. Получить: указатель на саму структуру.

Применение:
- `inode->i_cdev` — указывает на `cdev` поле в `scull_dev`.
- `container_of(inode->i_cdev, struct scull_dev, cdev)` — даёт указатель на `scull_dev`.

Это **стандартный приём** в ядре Linux для работы с внутренними структурами.

### private_data — что это

```c
filp->private_data
```

В `struct file` есть поле void *private_data. Драйвер может хранить там **что угодно** — обычно указатель на свои данные устройства.

В дальнейших вызовах (read, write, ioctl, release):
```c
struct scull_dev *dev = filp->private_data;
```

Это удобнее, чем каждый раз искать через cdev.

### Множественный open

Один `inode` может иметь несколько `struct file` (для разных open):

```c
fd1 = open("/dev/scull0", O_RDWR);  // file1, inode42
fd2 = open("/dev/scull0", O_RDWR);  // file2, inode42
```

При каждом open вызывается метод `open`. Создаются разные filp с разными private_data (но обычно указывающими на одну `scull_dev`).

### release — что делает

Вызывается при **последнем** `close(2)`. Задачи:

1. **Освободить ресурсы**, выделенные в open.
2. **Завершить операции** (flush буферов).
3. **Закрыть** устройство, если ни один процесс не использует.

### Пример: scull_release

```c
int scull_release(struct inode *inode, struct file *filp)
{
    return 0;   // scull simple — нечего делать
}
```

scull не выделяет ничего в open, поэтому release пустой.

### Когда вызывается release

- Когда **последний** fd закрыт.
- **НЕ** при каждом close (это flush).

Сценарий с fork:
```c
int fd = open("/dev/foo", O_RDWR);  // open вызвался
pid_t pid = fork();                  // ребёнок наследует fd

if (pid == 0) {
    // ребёнок
    close(fd);    // НЕ release (родитель ещё держит)
} else {
    // родитель
    close(fd);    // release (теперь никто не держит)
}
```

### Подсчёт пользователей

Иногда нужно знать, сколько процессов открыло устройство. Используется счётчик в `scull_dev`:

```c
struct scull_dev {
    // ...
    int num_users;
};

int my_open(...) {
    dev->num_users++;
    // ...
}

int my_release(...) {
    dev->num_users--;
    if (dev->num_users == 0) {
        // последний — освободить ресурсы
    }
    return 0;
}
```

Атомарность счётчика — через `atomic_t` или семафор.

### Простое vs сложное устройство

Простое (как scull) — open/release пустые или почти пустые.

Сложное (например, scull-pipe) — open инициализирует буферы, выделяет память; release освобождает.

### Резюме

- `open` — при каждом `open(2)` пользователя, идентифицирует устройство, инициализирует state.
- `release` — при **последнем** `close(2)`, освобождает ресурсы.
- `container_of` находит structuru драйвера по полю.
- `private_data` хранит указатель для удобного доступа в read/write/etc.

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

Параметры:
- `filp` — `struct file`, через `private_data` доступ к устройству.
- `buf` — буфер пользователя (помечен `__user`).
- `count` — запрошенное количество байт.
- `f_pos` — указатель на текущую позицию.

Возврат: число фактически переданных байт. Может быть < count!

### Аннотация __user

`__user` — макрос для sparse (статический анализатор):
- Предупреждает, если user-указатель используется напрямую без `copy_*_user`.
- В runtime — ничто.

### scull_read — пример

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

    // Захват семафора (защита dev->size, dev->data)
    if (down_interruptible(&dev->sem))
        return -ERESTARTSYS;

    // EOF?
    if (*f_pos >= dev->size)
        goto out;

    // Не запрашивать больше, чем есть
    if (*f_pos + count > dev->size)
        count = dev->size - *f_pos;

    // Найти позицию в структуре
    item = (long)*f_pos / itemsize;
    rest = (long)*f_pos % itemsize;
    s_pos = rest / quantum; q_pos = rest % quantum;

    dptr = scull_follow(dev, item);
    if (dptr == NULL || !dptr->data || !dptr->data[s_pos])
        goto out;

    // Не выходить за пределы одного кванта
    if (count > quantum - q_pos)
        count = quantum - q_pos;

    // Копирование к пользователю
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

### Разбор шагов

**1. Получение указателя на устройство.**
```c
struct scull_dev *dev = filp->private_data;
```

**2. Захват семафора.**

Защищает от параллельной модификации структуры.

```c
if (down_interruptible(&dev->sem))
    return -ERESTARTSYS;
```

Возврат `-ERESTARTSYS` означает: системный вызов прерван сигналом, ядро рестартует его (если возможно).

**3. Проверка EOF.**

```c
if (*f_pos >= dev->size)
    goto out;   // вернёт 0 — EOF
```

**4. Обрезка count.**

Не читать больше, чем есть.

**5. Расчёт позиции** (см. вопрос 82).

**6. Поиск scull_qset.**

```c
dptr = scull_follow(dev, item);
```

**7. Дальше не пересекать границу кванта.**

```c
if (count > quantum - q_pos)
    count = quantum - q_pos;
```

Пользователь повторит read для следующей порции.

**8. Копирование.**

```c
copy_to_user(buf, dptr->data[s_pos] + q_pos, count)
```

При ошибке (плохой указатель пользователя) → `-EFAULT`.

**9. Обновление позиции.**

```c
*f_pos += count;
```

Метод **обязан** обновлять `f_pos`. Иначе следующий read прочтёт то же.

**10. Возврат**.

Число переданных байт или код ошибки.

### scull_write — пример

```c
ssize_t scull_write(struct file *filp, const char __user *buf,
                    size_t count, loff_t *f_pos)
{
    struct scull_dev *dev = filp->private_data;
    struct scull_qset *dptr;
    int quantum = dev->quantum, qset = dev->qset;
    int itemsize = quantum * qset;
    int item, s_pos, q_pos, rest;
    ssize_t retval = -ENOMEM;

    if (down_interruptible(&dev->sem))
        return -ERESTARTSYS;

    // Найти позицию
    item = (long)*f_pos / itemsize;
    rest = (long)*f_pos % itemsize;
    s_pos = rest / quantum; q_pos = rest % quantum;

    // Создать узел и квант, если нужно
    dptr = scull_follow(dev, item);
    if (dptr == NULL) goto out;
    if (!dptr->data) {
        dptr->data = kmalloc(qset * sizeof(char *), GFP_KERNEL);
        if (!dptr->data) goto out;
        memset(dptr->data, 0, qset * sizeof(char *));
    }
    if (!dptr->data[s_pos]) {
        dptr->data[s_pos] = kmalloc(quantum, GFP_KERNEL);
        if (!dptr->data[s_pos]) goto out;
    }

    // Не за пределы кванта
    if (count > quantum - q_pos)
        count = quantum - q_pos;

    // Копирование от пользователя
    if (copy_from_user(dptr->data[s_pos] + q_pos, buf, count)) {
        retval = -EFAULT;
        goto out;
    }
    *f_pos += count;
    retval = count;

    // Обновить размер
    if (dev->size < *f_pos)
        dev->size = *f_pos;

  out:
    up(&dev->sem);
    return retval;
}
```

### Возможность частичной операции

Возврат `count` < запрошенного — нормально. Пользовательская библиотека повторит вызов:

```c
// glibc, упрощённо
while (total < n) {
    ssize_t r = sys_read(fd, buf + total, n - total);
    if (r < 0) {
        if (errno == EINTR) continue;
        return -1;
    }
    if (r == 0) break;  // EOF
    total += r;
}
```

### Типичные коды возврата

- **> 0:** число переданных байт.
- **0:** EOF (для read).
- **-EAGAIN:** неблокирующее устройство не готово.
- **-EFAULT:** ошибка доступа к памяти пользователя.
- **-EINTR / -ERESTARTSYS:** прерывание сигналом.
- **-ENOMEM:** нет памяти.
- **-ENOSPC:** на устройстве нет места.
- **-EIO:** аппаратная ошибка.

### Резюме

- read/write — основные методы передачи данных.
- Принимают user-указатель, count, позицию.
- Используют `copy_to_user` / `copy_from_user`.
- Возвращают число переданных байт (может быть меньше count).
- Должны обновлять `*f_pos`.
- При ошибке возвращают отрицательный errno.

---

## 85. Символьное устройство метод ioctrl. Передача параметров. Возвращаемое значение.
*Источник: LDD3*

### Зачем ioctl

read/write — для **данных**. Но устройства имеют ещё **параметры** и **команды**:
- Установить bitrate UART.
- Сбросить накопитель.
- Получить статус.
- Включить/выключить функцию.

Эти операции не сводятся к чтению/записи байт. Для них — **ioctl**.

### Сигнатура

В современных ядрах:
```c
long (*unlocked_ioctl) (struct file *filp, unsigned int cmd, unsigned long arg);
```

В старых (с BKL):
```c
int (*ioctl) (struct inode *inode, struct file *filp,
              unsigned int cmd, unsigned long arg);
```

`unlocked_ioctl` — без big kernel lock (BKL); рекомендуется к использованию.

### Со стороны пользователя

```c
#include <sys/ioctl.h>
int ioctl(int fd, unsigned long cmd, ...);
```

Вариативный — реально не больше одного аргумента (как `unsigned long`).

```c
int val = 42;
ioctl(fd, MY_SET_CMD, val);            // передать значение
ioctl(fd, MY_GET_CMD, &val);           // передать указатель
```

Драйвер интерпретирует `arg` в зависимости от cmd.

### Кодирование cmd

Чтобы команды разных драйверов не пересекались, в Linux используется **схема кодирования cmd**:

```c
#include <asm/ioctl.h>

#define _IO(type, nr)              // нет передачи данных
#define _IOR(type, nr, datatype)   // read: данные ядро → пользователь
#define _IOW(type, nr, datatype)   // write: пользователь → ядро
#define _IOWR(type, nr, datatype)  // оба направления
```

- `type` — magic number (символ или число), уникальный для драйвера.
- `nr` — номер команды (1, 2, 3...).
- `datatype` — тип передаваемых данных (для расчёта размера).

### Пример определений (scull)

```c
#define SCULL_IOC_MAGIC  'k'
#define SCULL_IOCRESET    _IO(SCULL_IOC_MAGIC, 0)
#define SCULL_IOCSQUANTUM _IOW(SCULL_IOC_MAGIC, 1, int)
#define SCULL_IOCSQSET    _IOW(SCULL_IOC_MAGIC, 2, int)
#define SCULL_IOCGQUANTUM _IOR(SCULL_IOC_MAGIC, 3, int)
#define SCULL_IOCGQSET    _IOR(SCULL_IOC_MAGIC, 4, int)
#define SCULL_IOCXQUANTUM _IOWR(SCULL_IOC_MAGIC, 5, int)
#define SCULL_IOC_MAXNR 14
```

«S» = Set, «G» = Get, «X» = eXchange.

### Структура cmd внутри

`cmd` — 32-битное число с полями:
- Биты 31-30: направление (NONE/READ/WRITE/RW).
- Биты 29-16: размер данных.
- Биты 15-8: type.
- Биты 7-0: nr.

Это позволяет извлекать поля макросами:
```c
_IOC_DIR(cmd)    // _IOC_NONE/_IOC_READ/_IOC_WRITE/(_R|_W)
_IOC_TYPE(cmd)   // type
_IOC_NR(cmd)     // nr
_IOC_SIZE(cmd)   // размер
```

### Реализация ioctl

```c
long scull_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int err = 0, retval = 0;

    // 1. Проверка type
    if (_IOC_TYPE(cmd) != SCULL_IOC_MAGIC) return -ENOTTY;

    // 2. Проверка номера
    if (_IOC_NR(cmd) > SCULL_IOC_MAXNR) return -ENOTTY;

    // 3. Проверка доступности памяти пользователя
    if (_IOC_DIR(cmd) & _IOC_READ)
        err = !access_ok(VERIFY_WRITE, (void __user *)arg, _IOC_SIZE(cmd));
    else if (_IOC_DIR(cmd) & _IOC_WRITE)
        err = !access_ok(VERIFY_READ, (void __user *)arg, _IOC_SIZE(cmd));
    if (err) return -EFAULT;

    // 4. Обработка команды
    switch (cmd) {
        case SCULL_IOCRESET:
            scull_quantum = SCULL_QUANTUM;
            scull_qset = SCULL_QSET;
            break;

        case SCULL_IOCSQUANTUM:
            if (!capable(CAP_SYS_ADMIN)) return -EPERM;
            retval = __get_user(scull_quantum, (int __user *)arg);
            break;

        case SCULL_IOCGQUANTUM:
            retval = __put_user(scull_quantum, (int __user *)arg);
            break;

        case SCULL_IOCXQUANTUM:
            // обменять: вернуть старое, установить новое
            {
                int tmp = scull_quantum;
                if (!capable(CAP_SYS_ADMIN)) return -EPERM;
                retval = __get_user(scull_quantum, (int __user *)arg);
                if (retval == 0) retval = __put_user(tmp, (int __user *)arg);
            }
            break;

        default:
            return -ENOTTY;
    }
    return retval;
}
```

### Передача данных

Способы передачи `arg`:

**1. Прямое значение (integer).**

```c
ioctl(fd, MY_SET, value);
```

Драйвер:
```c
case MY_SET:
    my_var = (int)arg;
    break;
```

**2. Указатель — копирование через put_user/get_user.**

```c
int value;
ioctl(fd, MY_GET, &value);
```

Драйвер:
```c
case MY_GET:
    retval = __put_user(my_var, (int __user *)arg);
    break;
```

**3. Указатель на структуру — copy_*_user.**

```c
struct mystruct s;
ioctl(fd, MY_GETS, &s);
```

Драйвер:
```c
case MY_GETS:
    retval = copy_to_user((void __user *)arg, &my_struct, sizeof(my_struct));
    break;
```

### put_user / get_user

Для **одного** элемента известного типа:
```c
__put_user(val, user_ptr);   // в пользователя
__get_user(val, user_ptr);   // от пользователя
```

Быстрее, чем `copy_to_user`, потому что компилятор знает тип.

### Возвращаемое значение

- **0:** успех.
- **Положительное:** успех с дополнительной информацией.
- **-ENOTTY:** неизвестная команда (стандарт). Историческое название: «Not a typewriter».
- **-EFAULT:** ошибка доступа к памяти пользователя.
- **-EPERM:** нет прав.
- **-EINVAL:** некорректные аргументы.

### Резюме

- ioctl — общий механизм управления устройством.
- cmd кодируется через `_IO`/`_IOR`/`_IOW`/`_IOWR`.
- Параметры через `arg` (значение или указатель).
- Используются `__put_user`/`__get_user`/`copy_*_user` для безопасности.
- Возврат: 0 при успехе, отрицательный errno при ошибке.

---

## 86. Символьное устройство метод ioctrl. Разрешения и запрещённые операции. FIOASYNC
*Источник: LDD3*

### Проверка разрешений

ioctl может выполнять опасные операции:
- Менять параметры устройства.
- Выполнять привилегированные действия.
- Изменять глобальное состояние.

Поэтому многие команды требуют **проверки прав**.

### capable()

```c
if (!capable(CAP_SYS_ADMIN))
    return -EPERM;
```

Возможные привилегии:
- **CAP_SYS_ADMIN** — общая «root-like» возможность.
- **CAP_NET_ADMIN** — сеть.
- **CAP_DAC_OVERRIDE** — обход прав файлов.
- **CAP_SYS_RAWIO** — прямой ввод-вывод.
- **CAP_SYS_NICE** — приоритеты процессов.
- **CAP_SYS_TIME** — изменение времени.

`capable` возвращает 1 если есть привилегия, 0 — нет.

### Проверка mode

Если команда модифицирует state, может потребоваться проверить, что fd открыт на запись:

```c
if (!(filp->f_mode & FMODE_WRITE))
    return -EPERM;
```

### Проверка диапазона

Чтобы случайно не обработать команду «не для этого драйвера»:

```c
if (_IOC_TYPE(cmd) != MY_IOC_MAGIC)
    return -ENOTTY;
if (_IOC_NR(cmd) > MY_IOC_MAXNR)
    return -ENOTTY;
```

### Проверка указателей

```c
if (_IOC_DIR(cmd) & _IOC_READ)
    err = !access_ok(VERIFY_WRITE, (void __user *)arg, _IOC_SIZE(cmd));
else if (_IOC_DIR(cmd) & _IOC_WRITE)
    err = !access_ok(VERIFY_READ, (void __user *)arg, _IOC_SIZE(cmd));
if (err) return -EFAULT;
```

`access_ok` — проверяет, что user-указатель в допустимом диапазоне. Это **предварительная** проверка; для самого копирования используются `copy_*_user` (они тоже проверяют).

### Запрещённые операции и -ENOTTY

`-ENOTTY` (исторически «inappropriate ioctl for device», от tty: «Not a typewriter») — для:
- **Неизвестных** cmd.
- **Команд другого типа** (не для этого драйвера).
- **Незарегистрированных** ioctl.

```c
default:
    return -ENOTTY;
```

### Стандартные ioctl

Некоторые команды стандартизированы и могут быть применимы к любому fd:

| Команда | Назначение |
|---|---|
| `FIONBIO` | File IO Non-Blocking IO |
| `FIONREAD` | Number of bytes available to read |
| `FIOQSIZE` | Размер файла |
| `FIOASYNC` | Асинхронный режим |
| `FIONCLEX` | Сбросить close-on-exec |
| `FIOCLEX` | Установить close-on-exec |

### FIOASYNC

Включает/выключает асинхронный режим: SIGIO посылается процессу при готовности устройства.

```c
int on = 1;
ioctl(fd, FIOASYNC, &on);    // включить
```

Также нужно сообщить ядру PID для уведомления:
```c
fcntl(fd, F_SETOWN, getpid());
```

После этого при готовности устройства процесс получает `SIGIO`. Установив обработчик, можно реагировать асинхронно.

### Реализация в драйвере — fasync

Драйвер должен реализовать метод `fasync`:

```c
int (*fasync) (int fd, struct file *filp, int mode);
```

И иметь поле для списка процессов-подписчиков:

```c
struct my_dev {
    // ...
    struct fasync_struct *async_queue;
};
```

Реализация:
```c
int my_fasync(int fd, struct file *filp, int mode)
{
    struct my_dev *dev = filp->private_data;
    return fasync_helper(fd, filp, mode, &dev->async_queue);
}
```

`fasync_helper` управляет списком — добавляет или удаляет процесс при включении/выключении.

### Уведомление

Когда событие произошло (в обработчике прерывания или другом контексте):

```c
if (dev->async_queue)
    kill_fasync(&dev->async_queue, SIGIO, POLL_IN);
```

- `POLL_IN` — данные для чтения.
- `POLL_OUT` — есть место для записи.
- `POLL_ERR` — ошибка.

### Освобождение в release

При закрытии fd процесс должен быть удалён из списка:

```c
int my_release(struct inode *inode, struct file *filp)
{
    my_fasync(-1, filp, 0);   // удалить из списка
    // ...
    return 0;
}
```

### Современная замена SIGIO

В современных приложениях SIGIO мало используется — есть лучшие механизмы:

**epoll** — Linux-специфичный, масштабируется на тысячи fd.

```c
int epfd = epoll_create1(0);
struct epoll_event ev = { .events = EPOLLIN, .data.fd = fd };
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

while (1) {
    struct epoll_event events[10];
    int n = epoll_wait(epfd, events, 10, -1);
    for (int i = 0; i < n; i++) {
        // обработать events[i].data.fd
    }
}
```

Преимущества над SIGIO:
- Не использует сигналы (асинхронные, сложные в обработке).
- Поддерживает level- и edge-triggered.
- Эффективен для большого числа fd.

### Резюме

- Разрешения через `capable(CAP_*)`.
- Запрещённые команды возвращают `-ENOTTY` или `-EPERM`.
- Стандартные ioctl: FIONBIO, FIONREAD, FIOASYNC.
- FIOASYNC — асинхронный режим с SIGIO.
- Реализуется через `fasync` метод и `fasync_helper`.
- Современная замена SIGIO — epoll.

---

## 87. Использование семафоров в scull. Семафоры по чтению и записи. void downgrade_write(struct rw_semaphore *sem);
*Источник: LDD3; robert_lav.md, Глава 10*

### Зачем семафор в scull

scull имеет несколько общих ресурсов:
- Связный список scull_qset.
- Поле `size`.
- Параметры `quantum`, `qset`.

Без синхронизации **два** параллельных вызова (например, два write или write + read) могут:
- Перетереть указатели.
- Освободить память, которой пользуется другой поток.
- Возвратить мусор.

### Бинарный семафор

```c
struct scull_dev {
    struct scull_qset *data;
    int quantum;
    int qset;
    unsigned long size;
    struct semaphore sem;   // ← бинарный, для взаимного исключения
    struct cdev cdev;
};
```

Инициализация:
```c
sema_init(&dev->sem, 1);   // count = 1 = бинарный
```

### Использование

```c
ssize_t scull_read(...)
{
    if (down_interruptible(&dev->sem))
        return -ERESTARTSYS;

    // CS — работа со структурой
    // ...

    up(&dev->sem);
    return ret;
}
```

### Почему `down_interruptible`

Из robert_lav.md:
- `down()` — переводит в TASK_UNINTERRUPTIBLE. Нельзя прервать сигналом.
- `down_interruptible()` — TASK_INTERRUPTIBLE. Можно прервать сигналом, возвращает `-EINTR`.

Драйвер должен возвращать `-ERESTARTSYS` (специальный код, говорящий ядру: «перезапусти syscall, если возможно»).

### RW-семафоры

Если устройство **читается часто, пишется редко**, бинарный семафор избыточен — он сериализует и чтение.

Решение — **RW-семафор**:
- N параллельных читателей.
- 1 эксклюзивный писатель.

```c
struct rw_semaphore rwsem;
init_rwsem(&rwsem);
```

Или статически:
```c
DECLARE_RWSEM(rwsem);
```

### API RW-семафоров

```c
down_read(&rwsem);     // захват для чтения
up_read(&rwsem);

down_write(&rwsem);    // захват для записи
up_write(&rwsem);
```

Без `_interruptible` варианта для writer (есть для reader).

### downgrade_write

```c
void downgrade_write(struct rw_semaphore *sem);
```

Атомарно превращает write-lock в read-lock.

### Зачем нужна downgrade_write

Сценарий: захватили writer-lock для модификации. Сделали изменения. Дальше нужно **продолжать читать** изменённые данные, но писать больше не будем.

Без downgrade:
```c
down_write(&sem);
modify();
up_write(&sem);
// ← здесь другой писатель может вклиниться и снова изменить!
down_read(&sem);
read_more();
up_read(&sem);
```

Между up_write и down_read возможна гонка.

С downgrade:
```c
down_write(&sem);
modify();
downgrade_write(&sem);
// атомарно: writer → reader. Никто не может вклиниться.
read_more();
up_read(&sem);
```

### Семантика downgrade

- Текущий writer становится reader.
- Освобождает write-lock (другие read-locks могут пройти).
- Уже ждущие writers продолжают ждать.

### Преимущества downgrade

- **Атомарность** перехода.
- **Параллелизм** для последующего чтения (другие читатели проходят).
- **Не отдаёт writer-lock** другому writer'у.

### Применение

Удобно в случаях:
- Модификация структуры → дальнейшее использование без модификации.
- Подготовка данных → чтение.
- Lazy initialization.

Пример:
```c
struct cache_entry *get_cache(int key)
{
    struct cache_entry *entry;
    down_read(&cache_sem);
    entry = find_in_cache(key);
    up_read(&cache_sem);

    if (!entry) {
        down_write(&cache_sem);
        // двойная проверка — мог появиться пока ждали
        entry = find_in_cache(key);
        if (!entry) {
            entry = create_cache_entry(key);
            insert_in_cache(entry);
        }
        downgrade_write(&cache_sem);
        // теперь reader-lock, продолжаем использовать entry параллельно с другими
        use(entry);
        up_read(&cache_sem);
    }
    return entry;
}
```

### Семафоры в scull — реальная практика

В стандартном scull (LDD3) используется обычный бинарный семафор. RW-семафор был бы избыточен, потому что:
- Структура достаточно компактная.
- Большая часть кода — изменение размера.
- Накладные расходы RW > выгода.

В вариациях (scull-pipe) добавляются дополнительные примитивы для блокирующего I/O.

### Резюме

- scull использует бинарный семафор для защиты структуры.
- `down_interruptible` — позволяет прервать сигналом.
- При ошибке драйвер возвращает `-ERESTARTSYS`.
- RW-семафоры — для read-mostly данных.
- `downgrade_write` атомарно превращает write-lock в read-lock — устраняет гонку при последовательной модификации и чтении.

---

## 88. Блокирующий ввод/вывод. Очереди ожидания. Состояния процесса. wait_queue_head_t my_queue;
*Источник: robert_lav.md, Глава 4; LDD3*

### Зачем блокирующий I/O

Сценарий:
- Пользователь делает `read(fd, ...)` от устройства.
- Устройство **не готово** (нет данных).

Варианты:
1. **Polling:** пользователь крутит цикл `read` — плохо, расход CPU.
2. **Возврат ошибки** (EAGAIN) — пользователю нужно повторять.
3. **Блокирующий I/O** — драйвер засыпает в `read`, пока не будет данных.

Блокирующий I/O — обычная семантика Unix.

### Очередь ожидания

Из robert_lav.md (Глава 4):
> «Замораживание задачи обрабатывается с помощью очередей ожидания (wait queue). Очередь ожидания представляет собой обычный список процессов, которые ожидают наступления определенного события».

В Linux — структура `wait_queue_head_t`:

```c
#include <linux/wait.h>

wait_queue_head_t my_queue;
init_waitqueue_head(&my_queue);
```

Или статически:
```c
DECLARE_WAIT_QUEUE_HEAD(my_queue);
```

### Состояния процесса при блокировке

Из robert_lav.md:
- **TASK_RUNNING** — работает или готов к работе.
- **TASK_INTERRUPTIBLE** — спит, можно разбудить сигналом или wake_up.
- **TASK_UNINTERRUPTIBLE** — спит, можно разбудить только wake_up.

`TASK_UNINTERRUPTIBLE` процессы видны в `ps` как `D` («uninterruptible disk sleep») — их нельзя завершить, даже `SIGKILL`. Используются в критических операциях I/O.

В большинстве случаев — `TASK_INTERRUPTIBLE`.

### Засыпание — функции

**Простое (старое):**
```c
add_wait_queue(&q, &wait);
set_current_state(TASK_INTERRUPTIBLE);
if (!condition) schedule();
set_current_state(TASK_RUNNING);
remove_wait_queue(&q, &wait);
```

**Современные макросы:**

```c
wait_event(queue, condition);                    // UNINTERRUPTIBLE
wait_event_interruptible(queue, condition);       // INTERRUPTIBLE
wait_event_timeout(queue, condition, timeout);
wait_event_interruptible_timeout(queue, condition, timeout);
```

Они инкапсулируют правильный паттерн с циклом и проверкой условия.

### Пример: блокирующий read

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

        // Ждать данные
        if (wait_event_interruptible(dev->inq, dev->size > 0))
            return -ERESTARTSYS;

        if (down_interruptible(&dev->sem))
            return -ERESTARTSYS;
    }

    // Здесь данные есть — копировать
    // ...
    up(&dev->sem);
    return count;
}
```

### Ключевые моменты

**1. Освобождение семафора перед сном.**

Если мы засыпаем с семафором, никто не сможет положить данные — deadlock.

**2. Проверка O_NONBLOCK.**

Если установлен — возвращаем `-EAGAIN`.

**3. Цикл проверки.**

После пробуждения проверяем условие — `wait_event_*` может возвращать spurious wakeup.

**4. Захват семафора заново.**

После сна — снова down (могло измениться состояние).

### Пробуждение

Когда событие произошло (например, обработчик прерывания получил данные):
```c
ssize_t my_write(...) {
    // ... поместить данные в структуру ...
    dev->size += count;
    wake_up_interruptible(&dev->inq);
    return count;
}
```

### Варианты wake_up

```c
wake_up(&q);                 // и INTERRUPTIBLE, и UNINTERRUPTIBLE
wake_up_interruptible(&q);   // только INTERRUPTIBLE
wake_up_nr(&q, n);           // первых n
wake_up_all(&q);             // всех
wake_up_interruptible_all(&q);
wake_up_interruptible_sync(&q);  // не вызывать сразу schedule
```

### Эксклюзивное ожидание

При `wake_up` обычно будятся **все** ожидающие. Если работу могут выполнить только один — это растрата (thundering herd).

Решение — эксклюзивное ожидание:
```c
prepare_to_wait_exclusive(&q, &wait, TASK_INTERRUPTIBLE);
```

Будит только одного помеченного эксклюзивным.

Применяется, например, в очередях задач.

### Гонки и spurious wakeup

См. вопрос 23 для подробностей. Кратко:
- Цикл проверки условия защищает от spurious wakeup.
- `prepare_to_wait` атомарно ставит в очередь и меняет состояние — нет lost wakeup.
- `wait_event_*` инкапсулирует правильный паттерн.

### Резюме

- Блокирующий I/O — драйвер засыпает до готовности.
- Очередь ожидания — `wait_queue_head_t`.
- Состояния: TASK_INTERRUPTIBLE (можно сигнал) vs TASK_UNINTERRUPTIBLE (нельзя).
- Засыпание через `wait_event_*` макросы.
- Пробуждение через `wake_up_*`.
- Освобождать блокировки перед сном.

---

## 89. Пример блокирующего ввода/вывода. Подробности «засыпания». wait_event_interruptible_timeout(queue, condition, timeout)
*Источник: robert_lav.md, Глава 4; LDD3*

### Полная сигнатура

```c
long wait_event_interruptible_timeout(wait_queue_head_t wq,
                                       condition,
                                       long timeout);
```

Параметры:
- `wq` — очередь.
- `condition` — выражение C, проверяемое в цикле.
- `timeout` — в jiffies (тиков системного таймера).

Возвращает:
- **> 0:** условие выполнено, оставшееся время в jiffies.
- **0:** таймаут истёк, условие так и не выполнилось.
- **-ERESTARTSYS:** прерывание сигналом.

### Пример использования

```c
long ret = wait_event_interruptible_timeout(dev->wq, dev->ready, HZ * 5);

if (ret == 0) {
    // таймаут
    return -ETIMEDOUT;
}
if (ret < 0) {
    // сигнал
    return ret;   // -ERESTARTSYS
}
// условие выполнено, ret = оставшееся время
process_data();
```

### HZ — единица времени

`HZ` — макрос ядра, число тиков таймера в секунду. Значение зависит от настройки ядра (100, 250, 1000).

```c
HZ * 5      // 5 секунд
HZ / 10     // 0.1 секунды
msecs_to_jiffies(100)  // 100 мс, портативно
```

### Что происходит при засыпании — подробно

**Шаг 1: Добавление в очередь.**

```c
DEFINE_WAIT(wait);
add_wait_queue(&q, &wait);
```

`DEFINE_WAIT` — макрос, создающий `wait_queue_entry_t` со специальной функцией пробуждения.

**Шаг 2: Изменение состояния — атомарно.**

```c
prepare_to_wait(&q, &wait, TASK_INTERRUPTIBLE);
```

Внутри:
- Если ещё не в очереди — добавить.
- Установить `current->state = TASK_INTERRUPTIBLE`.

Атомарность достигается через спинлок очереди.

**Шаг 3: Проверка условия.**

```c
if (condition) break;
```

Это **обязательно после prepare_to_wait, до schedule** — защита от lost wakeup. Если `wake_up` пришло между предыдущей проверкой и prepare_to_wait, мы здесь увидим выполненное условие и не пойдём спать.

**Шаг 4: Проверка сигнала.**

```c
if (signal_pending(current)) {
    ret = -ERESTARTSYS;
    break;
}
```

Если есть pending сигнал — не спим, возвращаем -ERESTARTSYS.

**Шаг 5: Уход на покой.**

```c
timeout = schedule_timeout(timeout);
```

`schedule_timeout` ставит таймер и засыпает. При таймауте, сигнале или wake_up — просыпается.

Возвращает оставшееся время.

**Шаг 6: Возврат в нормальное состояние.**

```c
finish_wait(&q, &wait);
```

Внутри:
- `current->state = TASK_RUNNING`.
- Удаление из очереди (если ещё там).

### Полная макроразвёртка (упрощённо)

```c
#define wait_event_interruptible_timeout(wq, condition, timeout) \
({ \
    long __ret = timeout; \
    if (!(condition)) { \
        DEFINE_WAIT(wait); \
        for (;;) { \
            prepare_to_wait(&wq, &wait, TASK_INTERRUPTIBLE); \
            if (condition) break; \
            if (signal_pending(current)) { \
                __ret = -ERESTARTSYS; \
                break; \
            } \
            __ret = schedule_timeout(__ret); \
            if (!__ret) break; \
        } \
        finish_wait(&wq, &wait); \
    } \
    __ret; \
})
```

### Ручная реализация (без макросов)

```c
ssize_t my_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
{
    struct my_dev *dev = filp->private_data;
    DEFINE_WAIT(wait);
    long timeo = HZ * 5;

    add_wait_queue(&dev->inq, &wait);
    while (1) {
        prepare_to_wait(&dev->inq, &wait, TASK_INTERRUPTIBLE);

        if (dev->size > 0) break;

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

    // Здесь данные есть
    // ...
    return count;
}
```

### Где использовать таймаут

- **Аппаратные операции** с лимитом времени.
- **Сетевые протоколы** с keepalive.
- **Watchdog**-логика.
- **Опросы** с периодом.

### Spurious wakeups

Возможно пробуждение без сигнала и без события — из-за внутренних механизмов ядра. Цикл `while/for(;;)` с проверкой condition защищает от этого.

### wait_event vs wait_event_interruptible

| Макрос | Сигналы | Состояние |
|---|---|---|
| `wait_event` | Игнорирует | UNINTERRUPTIBLE |
| `wait_event_interruptible` | Возвращает -ERESTARTSYS | INTERRUPTIBLE |
| `wait_event_timeout` | Игнорирует, есть таймаут | UNINTERRUPTIBLE |
| `wait_event_interruptible_timeout` | Полный набор | INTERRUPTIBLE |

Использовать `wait_event` — только для **очень коротких** ожиданий, где сигнал не критичен.

### Резюме

- `wait_event_interruptible_timeout` — полный паттерн с таймаутом и сигналами.
- Внутри: prepare_to_wait → проверка условия → проверка сигнала → schedule_timeout → finish_wait.
- Атомарность через prepare_to_wait.
- Возврат: время / 0 / -ERESTARTSYS.
- Защита от spurious wakeup через цикл.

---

## 90. Функция malloc(), vmalloc() и kmalloc(). Основные отличия и назначение.
*Источник: robert_lav.md, Глава 12 «Управление памятью»*

### Общая картина

В Linux есть **три уровня** управления памятью:

1. **User space:** `malloc()` — кучка процесса.
2. **Kernel small allocations:** `kmalloc()` — физически непрерывно.
3. **Kernel large allocations:** `vmalloc()` — виртуально непрерывно.

### malloc — пользовательское пространство

```c
#include <stdlib.h>
void *malloc(size_t size);
void free(void *ptr);
```

- В **user space.**
- Возвращает **виртуально непрерывный** блок.
- **Физически** — необязательно непрерывный (страницы могут быть в разных местах).
- Использует системные вызовы `brk` (для маленьких блоков, расширяет heap) или `mmap` (для больших, отдельные регионы).
- Размер: до десятков ГБ (зависит от АП и лимитов).
- Реализация — в библиотеке C (ptmalloc, jemalloc, tcmalloc...).

### kmalloc — ядро, физически непрерывно

Из robert_lav.md (Глава 12):
> «Функция `kmalloc()` аналогична функции `malloc()`, с которой разработчики пользовательских приложений уже знакомы, за исключением того, что в неё добавлен ещё один параметр `flags`».

```c
#include <linux/slab.h>
void *kmalloc(size_t size, gfp_t flags);
void kfree(const void *ptr);
```

Возвращает **физически и виртуально непрерывный** блок памяти.

### Зачем физически непрерывно

- **DMA-операции:** многие устройства работают с физическими адресами, не понимают виртуальную память. Им нужны непрерывные физические блоки.
- **Производительность:** одна запись TLB покрывает большой регион.

### Флаги kmalloc

Из robert_lav.md:
- **`GFP_KERNEL`** — обычное, может блокировать (для подкачки и т.д.).
- **`GFP_ATOMIC`** — нельзя блокировать (для IRQ-контекста, удержанных спинлоков). Может вернуть NULL чаще.
- **`GFP_USER`** — для аллокаций «от имени» пользователя.
- **`GFP_DMA`** — DMA-совместимая память (нижние 16 МБ на x86).
- **`GFP_HIGHUSER`** — высокая память.

### Ограничения kmalloc

- **Максимальный размер:** обычно 128 КБ (`KMALLOC_MAX_SIZE`).
- **Округление вверх:** до следующей степени двойки или специального размера slab-кеша.
  - 16, 32, 64, 96, 128, 192, ..., 8192, 16384, 32768, 65536, 131072.
- Не годится для очень больших или специфичных размеров.

### Пример использования

```c
struct dog {
    char name[64];
    int age;
};

struct dog *p = kmalloc(sizeof(*p), GFP_KERNEL);
if (!p)
    return -ENOMEM;
strcpy(p->name, "Rex");
p->age = 3;

// ... использовать ...

kfree(p);
```

### Когда блокирует

`GFP_KERNEL`:
- Может вызывать **запись на диск** для освобождения страниц.
- Может вызывать **swap-out** других страниц.
- Может занять миллисекунды.
- **Нельзя в:** обработчиках прерываний, под спинлоком, в RCU-критических секциях.

`GFP_ATOMIC`:
- Никогда не блокирует.
- Может **failed** при нехватке памяти.
- Используется в IRQ, под спинлоком.

### vmalloc — ядро, виртуально непрерывно

Из robert_lav.md:
> «Функция `vmalloc()` аналогична функции `kmalloc()`, за исключением того, что она выделяет смежные страницы виртуальной памяти, которые необязательно являются смежными физически».

```c
#include <linux/vmalloc.h>
void *vmalloc(unsigned long size);
void vfree(const void *addr);
```

Возвращает блок:
- **Виртуально непрерывный** (программа видит его как один блок).
- **Физически разрозненный** (страницы могут быть где угодно).

### Как это работает

Из robert_lav.md:
> «Это реализуется путём выделения потенциально несмежных участков физической памяти и изменения информации в таблицах страниц, отображающих эту физическую память в непрерывный участок логического адресного пространства».

Шаги:
1. Выделить N отдельных физических страниц.
2. Найти свободный регион в виртуальном пространстве ядра.
3. Настроить таблицы страниц так, чтобы N страниц шли подряд.
4. Вернуть начальный виртуальный адрес.

### Сравнение kmalloc vs vmalloc

| Параметр | kmalloc | vmalloc |
|---|---|---|
| Физически непрерывная | Да | Нет |
| Виртуально непрерывная | Да | Да |
| Максимум | ~128 КБ | ~ГБ |
| Скорость | Быстро | Медленно |
| TLB | Эффективно | Менее эффективно |
| Подходит для DMA | Да | Нет |
| Может блокировать | Зависит от флага | Всегда блокирует |
| В обработчике IRQ | С GFP_ATOMIC | НЕТ |

### Почему vmalloc медленнее

Из robert_lav.md:
> «Для того чтобы физически несмежные страницы памяти сделать смежными в виртуальном адресном пространстве, функция `vmalloc()` должна соответствующим образом сформировать элементы таблицы страниц. Хуже того, страницы виртуальной памяти, которые выделяются с помощью функции `vmalloc()`, должны отображаться на физические страницы памяти через элементы таблицы страниц по одной, поскольку они физически несмежные. Это приводит к значительно менее эффективному использованию буфера быстрой переадресации, или ББП (Translation Lookaside Buffer, TLB)».

- Каждая страница занимает свою запись TLB.
- Большой блок → много TLB-промахов.
- Сама аллокация дороже (нужно обновлять таблицы страниц).

### Когда использовать vmalloc

Из robert_lav.md:
> «Функция `vmalloc()` используется только тогда, когда это абсолютно необходимо, как правило, для выделения очень больших областей памяти. Например, при динамической загрузке модулей ядра они загружаются в память, которая выделяется с помощью функции `vmalloc()`».

Случаи:
- **Загрузка модулей ядра** — большой блок кода.
- **Большие буферы**, не используемые с DMA.
- **Когда непрерывная физическая память недоступна** из-за фрагментации.

### Пример vmalloc

```c
char *buf = vmalloc(16 * PAGE_SIZE);   // 64 КБ или 128 КБ
if (!buf)
    return -ENOMEM;

// ... использовать ...

vfree(buf);
```

### Иерархия аллокаторов в ядре

**Низкоуровневые (страницы):**
```c
struct page *alloc_pages(gfp_t mask, unsigned int order);
void __free_pages(struct page *page, unsigned int order);
```

**Slab allocator (объекты):**
```c
void *kmem_cache_alloc(kmem_cache_t *cache, gfp_t flags);
void kmem_cache_free(kmem_cache_t *cache, void *obj);
```

**Универсальный kmalloc:**
```c
void *kmalloc(size_t size, gfp_t flags);
```

`kmalloc` использует slab allocator под капотом.

**vmalloc:**
```c
void *vmalloc(unsigned long size);
```

### Сравнительная таблица

| Параметр | malloc | kmalloc | vmalloc |
|---|---|---|---|
| Пространство | User | Kernel | Kernel |
| Физически непрерывно | Нет | Да | Нет |
| Виртуально непрерывно | Да | Да | Да |
| Максимум | Гигабайты | ~128 КБ | Гигабайты |
| Может блокировать | Может (page fault) | Зависит от GFP | Да |
| В контексте IRQ | — | С GFP_ATOMIC | НЕТ |
| Применение | Программы | Большинство в ядре | Большие буферы, модули |
| Освобождение | free | kfree | vfree |

### Резюме

- malloc — пользовательский, виртуально непрерывный.
- kmalloc — в ядре, физически и виртуально непрерывный, до 128 КБ, для DMA.
- vmalloc — в ядре, только виртуально непрерывный, для больших аллокаций, без DMA.
- В 95% случаев в ядре используется kmalloc.
- vmalloc применяется редко: модули, очень большие буферы.
