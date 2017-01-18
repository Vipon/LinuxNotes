##Списки Linux

Для каждого списка должны быть реализованы функции: инициализация, вставка и
удаление элемента, поиск элемента в списке.
В Linux реализована универсальная структура двунаправленного списка, которая
позволяет эффективно решать данные задачи для произвольного содержимого
списков(include/linux/types.h) - struct list_head, для этого её нужно
включать в структуру данных.

```c
struct list_head {
    struct list_head *next, *prev;
};
```

Функции и макросы для работы со списками описаны в (include/linux/list.h).
Список создаётся с помощью макроса LIST_HEAD(name):

```c
#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) \
    struct list_head name = LIST_HEAD_INIT(name)
```

После этого первый элемент списка указывает сам на себя:

```
/——\    /———\
|  |    |    |
| _\/__\/_   |
 \| next |   /
  -———————  /  list_head
  | prev | /
  -———————
```

Получить адрес структуры данных, которая содержит list\_head можно с помощью макроса list\_entry:

```c
/**
 * list_entry -     get the struct for this entry
 * @ptr:            the &struct list_head pointer.
 * @type:           the type of the struct this is embedded in.
 * @member: the name of the list_head within the struct.
 */
#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)  // описание макроса найдём в (include/linux/kernel.h)
/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:    the pointer to the member.
 * @type:   the type of the container struct this is embedded in.
 * @member: the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({          \
    const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

Таким образом когда мы используем макрос list_entry(ptr, my_type,
list_in_my_type), он заменяется макросом container_of и раскрывается в:

```c
const typeof( ((my_type *)0)->list_in_my_type ) *__mptr = (ptr);
(my_type *)( (char *)__mptr - offsetof(my_type,list_in_my_type) );
```

Что окончательно будет выглядеть, как:
const list_head *__mptr = ptr;
(my_type *)( (char *)__mptr  - offsetof(my_type,list_in_my_type) );

offset обрабатывается во время компиляции и заменяется на смещение элемента
list_in_my_type в типе(структуре) my_type.

Кратко опишем другие функции list_head:

```c
list_add(new, head) // вставит элемент new после head.
list_add_tail(new, head) // вставить new перед head.
list_del(struct list_head *entry) // удаляет entry из списка.
list_for_each(pos, head) // после на каждой итерации ops содержит очередной элемент из списка:

#define list_for_each(pos, head) \
    for (pos = (head)->next; pos != (head); pos = pos->next)
```

Так же в Linux есть ещё один тип двунаправленных списков hlist_head
(include/linux/types.h), которые отличаются от предыдущих тем, что их
голова указывает только на начало списка; он не цикличный. Такие списки
применяются в основном в хеш-таблицах, конец списка нас не интересует, а
память для хеш-таблиц является критичным ресурсом.
struct hlist_head {
    struct hlist_node *first;
};

Обратим внимание, что у головы списка всего один указатель, а обрабатывать
список хочется унифицировано, иными слова должен быть всего один набор
функций для всех элементов списка, будь они хоть сразу после головы, хоть в
середине списка. При этом обрабатывать список нужно по возможности быстро,
как и предыдущий list_head. Потому каждый элемент списка представлен
структурой - hlist_node:
struct hlist_node {
    struct hlist_node *next, **pprev;
};

Так как у hlist_head есть указатель на следующий, то и у hlist_node он так
же есть - *next, однако hlist_node хранит указатель  не на предыдущий
элемент, а на то место, где хранится предыдущий элемент -  **pprev, а
хранится он в поле *next предыдущего элемента, а так как адрес поля *next
предыдущего элемента совпадает с адресом самого элемента, то мы без лишних
операций получаем адрес предыдущего элемента.

```
hlist_head
   /———————————————\    /—————————\   /—————\
__|_______       __\/_|___       _\/_|____
| *first | <—    | *next |<—     | *next |
——————————    \  —————————   \   —————————
hlist_head     \—| pprev |    \— | pprev |
                 —————————       —————————
                 hlist_node
```

В остальном философия работы с  hlist_head похожа на предыдущий список.
(include/linux/list.h)
Инициализация:
```c
#define HLIST_HEAD(name) struct hlist_head name = {  .first = NULL }
```
Получение адреса структуры, где находится элемент списка:
```c
#define hlist_entry(ptr, type, member) container_of(ptr,type,member)
```
Добавление после головы списка:
```
static inline void hlist_add_head(struct hlist_node *n, struct hlist_head *h);
```
Добавление перед элементом:
```
static inline void hlist_add_before(struct hlist_node *n, struct hlist_node *next);
```
Добавление после элемента:
```
static inline void hlist_add_behind(struct hlist_node *n, struct hlist_node *prev);
```
Удаление:
```
static inline void hlist_del(struct hlist_node *n);
```
Итерация по списку:
```
#define hlist_for_each(pos, head) \
    for (pos = (head)->first; pos ; pos = pos->next)
```
Вообще все операции со списками выполняются с помощью примитива
синхронизации — RCU, чтобы обеспечить атомарность работы с ними. Про него
можете прочитать в Linux Synchronization.txt.
Чтение списка — поиск в списке. Выполняется без особых замысловатых
движений, просто читаем:
```c
rcu_read_lock();
…
rcu_read_unlock();
```
Удаление из списка элемента так же не сложно. При поиске не используется
поле prev, потому его можно изменить сразу. Поле next предыдущего тоже
можно спокойно изменить, это произойдёт атомарно. А старый элемент можно
оставить без изменения, чтобы читало мог от туда выйти.
```c
next->prev = prev;
prev->next = next;
```
Добавление элемента в список происходят по такому же сценарию, только
сначала создаём новый элемент и инициализируем его поля.
