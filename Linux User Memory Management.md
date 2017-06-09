# User memory management
Запросов ядра на выделение памяти: alloc_pages() и kmalloc(), приводят к немедленному выделению памяти, если могут быть удовлетворены. Это оправдано, потому что:
* **Ядро** - самый приоритетный компонент системы, его запросы критические.
* Ядро себе доверяет, предполагается, что в ядре нет ошибок.
Для процессов, работающих в режиме пользователя, всё иначе:
* Запросы процесса на память можно отложить.
* В коде пользователя могут быть ошибки, потому нужно быть готовым к обработке ошибок.
Когда процесс запрашивает память, он получает не новые страничные кадры, а право обращаться к новым линейным адресам.

## Адресное пространство процесса
**Адресное пространство процесса** - линейные адреса, к которым процесс может обращаться. Ядро может динамически изменять адресное пространство процесса с помощью добавления или удаления *областей памяти(vm_area_struct)*.
Процесс может получить новые области памяти, например, с помощью вызывов: malloc(), calloc(), mmap(), brk(), shmget() + shmat(), posix_memalign(), mmap() и т.д. В основе всех этих вызовов лежит _void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);_:
>> *addr* - адрес, где выделять память.
>> *flags*:
>>> **NULL** - нет никакой разницы, где выделять память. Параметр *addr* используется, как рекомендация.
>>> **MAP_FIXED** - именно там, где указано в *addr*.
>>> **MAP_ANON(MAP_ANONYMOUS)** - изменения не будут видны ни в каком файле.
>>> **MAP_FILE** - мапим из файла или устройства.
>> *prot*:
>>> **PROT_EXEC**
>>> **PROT_READ**
>>> **PROT_WRITE**
>>> **PROT_NONE**

## Дескриптор памяти
Вся информация относительно адресного пространства процесса хранится в *mm_struct* (дескриптор памяти), на который указывает поле mm в task_struct.

```
task_struct
_________
|   …   |     mm_struct
---------    _________
|   mm  | -> |   …   |
---------    ---------
|   …   |    |  mmap |  ->   vm_area_struct * (VMA) – список двунап.
---------    ---------
             |  pgd  |  - указатель на глобальный каталог страниц
             ---------
```

Описание структур mm_struct и vm_area_struct монжо найти в /include/linux/mm_types.h.

## Область памяти
```
struct vm_area_struct {
    /* The first cache line has the info for VMA tree walking. */

    unsigned long vm_start;     /* Our start address within vm_mm. */
    unsigned long vm_end;       /* The first byte after our end address
                       within vm_mm. */

    /* linked list of VM areas per task, sorted by address */
    struct vm_area_struct *vm_next, *vm_prev;
             ...................
             struct rb_node               vm_rb;
             ………..
    /* Function pointers to deal with this struct. */
    const struct vm_operations_struct *vm_ops;
}
```
У области памяти есть два поля vm_start и vm_end, обозначающие соответственно адрес начала и первого бита после конца выделенной области. Если применить mmap() с одинаковыми аргументами, то ядро не будет создавать новый VMA, а просто изменит vm_end уже существующего.
Все области памяти объеденены в двунаправленный список, где они упорядочены по возрастанию адресов. Для того чтобы не приходилось пробегаться по всему списку при выделении, возвращении памяти или поиску VMA, которому принадлежит адрес, все VMA так же объединены в красно-чёрное дерево(/include/linux/rbtree.h).
```
struct rb_node {
    unsigned long  __rb_parent_color;
    struct rb_node *rb_right;
    struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
    /* The alignment might seem pointless, but allegedly CRIS needs it */

struct rb_root {
    struct rb_node *rb_node;
};
```
**mm_struct -> pgd** - указатель на глобальный каталог страниц каждого процесса. На x86 при переключении процесса **mm_struct -> pgd** помещается в cr3. Изменение cr3 в свою очередь приводит к сбросу TLB. Однако, у двух task_struct может быть один и тот же mm, например, у двух потоков, тогда изменения cr3 не будет, что существенно ускоряет работу с памятью.
Помимо потоков переключение cr3 так же не происходит для kernel_thread. Для них просто нет необходимости в *областях памяти*, так как они всегда обращаются к фиксированным линейным адресам выше TASK_SIZE = PAGE_OFFSET = 0xffff880000000000 (x86_64). Потому собственный mm kernel_thread в task_struct просто не нужен, он равен NULL. Зато в task_struct есть active_mm, равный active_mm вытесненного процесса.
Ещё одним интересным полем в VMA является vm_ops, оно определяет операции, которые можно выполнять для конкретной области памяти.

## Работа с областями памяти
### Описание функций:
* do_mmap() (/mm/mmap.c) – выделение новой области памяти
* do_munmap() (/mm/mmap.c) – возвращение области памяти
* find_vma()(/mm/mmap.c) – поиск области ближайшей к данному адресу
* find_vma_intersection() (/include/linux/mm.h) – поиск области, содержащей адрес.
* get_unmapped_area() (/mm/mmap.c)  - поиск свободного интервала
* insert_vm_struct() (/mm/mmap.c) – внесение области в список дескрипторов

## Выделение интервала линейных адресов
Линейные адреса, которые выделяются, могут быть связаны с файлом (FILE) или нет (ANON). При этом, процесс, который запрашивает память, может владеть ими совместно с кем-то (MAP_SHARED) или уникально (MAP_PRIVATE).
```flow
st=>start:do_mmap()
op1=>operation:get_unmapped_area()
op2=>operation:mmap_region()
cond1=>condition: FILE or ANON?
cond2=>condition:MAP_SHARED or MAP_PRIVATE?
cond3=>condition:MAP_SHARED or MAP_PRIVATE?
e=>end:return addr

st->op->cond1
cond1(FILE)-> cond2
cond1(ANON)->cond3
cond2->op2
cond3->op2
op2->e
```
|                       | FILE                | ANON                 |
|:---------------------:|:-------------------:|:--------------------:|
| MAP_SHARED            | vma->file(get_page) | файл на tmpfs(shmat) |
| MAP_PRIVATE           | library(COW)        | HIGHMEM, ZERO(buddy) |

## Отложенное выделение
Как было сказано выше, запросы пользовательского процесса на память можно отложить до момента, когда память действительно понадобиться. Для этого используется механизм обработки исключения *page fault*, сигнализирующего об отсутствие страницы.
В x86 каждая запись в таблице страниц выровнена по 4096(2^12), потому первые 12 бит несут служебную информацию относительно страницы, например:
* __0 бит__ - P (Present) Flag
* __1 бит__ - R/W (Read/Write) Flag
* __2 бит__ - U/S (User/Supervisor) Flag
Таким образом, если выставить __P бит__ в ноль, то при обращении к данной области памяти будет генерироваться исключение. При генерировании исключения адрес, который его вызвал, сохранится в регистре __cr2__. Итоговый алгоритм можно представить в виде диаграммы.
```flow
st=>start: page fault
op1=>operation:Выделить новый страничный кадр
op2=>operation:Послать SIGSEGV
op3=>operation:Ошибка ядра: уничтожить процесс
cond1=>condition:Принадлежит ли адрес пространству процесса?
cond2=>condition:Соответствуют ли права доступа?
cond3=>condition:Исключение возникло в режиме пользователя?

st->cond1
cond1(Да)->cond2
cond1(Нет)->cond3
cond2(Да)->op1
cond2(Нет)->op2
cond3(Да)->op2
cond3(Нет)->op3
```
Если обращение происходит рядом со stack VMA – область созданная с флагом MAP_GROWDOWN, то происходит расширение области.