# Page cache
__Page cache__ - кэш данных, с которыми работают файловые системы. Эти данные могут быть прочитаны не только с дисков, но и из сети.

## Режимы работы кэша
* __Некэшируемый__ - каждый раз данные читают и записывают сразу на диск. В Linux можно так работать, если открыть файл с флагом *O_DIRECT*.
* __Write through__ - чтение производится из кэша, а запись в кэш приводит к немедленной записи на носитель.
* __Write back__ - чтение и запись производятся только в кэш. Запись на носитель отложена до наступления определённых моментов.

Page cache в Linux по умолчанию работает в режиме __Write back__.

## Структуры данных
__Page cache__ подвязан к struct inode(/include/linux/fs.h) через поле i_mapping типа struct address_space, который содержит дерево закэшированных страниц:
```c
/* /include/linux/fs.h */
struct address_space {
    struct inode        *host;      /* owner: inode, block_device */
    struct radix_tree_root  page_tree;  /* radix tree of all pages */
    ...
}

struct inode {
    ...
    struct address_space    *i_mapping;
    ...
}
```

Каждая struct page(/include/linux/mm_types.h) имеет поля __mapping__ и __index__. __Mapping__ указывает на address_space struct inode, которой принадлежит страница. __Index__ - смещение в файле, выраженное в страницах.
```c
struct page {
    /* First double word block */
    unsigned long flags;        /* Atomic flags, some possibly
                     * updated asynchronously */
    union {
        struct address_space *mapping;  /* If low bit clear, points to
                         * inode address_space, or NULL.
                         * If page mapped as anonymous
                         * memory, low bit is set, and
                         * it points to anon_vma object:
                         * see PAGE_MAPPING_ANON below.
                         */
        void *s_mem;            /* slab first object */
        atomic_t compound_mapcount; /* first tail page */
        /* page_deferred_list().next     -- second tail page */
    };

    /* Second double word */
    union {
        pgoff_t index;      /* Our offset within mapping. */
        void *freelist;     /* sl[aou]b first free object */
        /* page_deferred_list().prev    -- second tail page */
    };
    ...
    atomic_t _mapcount;
    ...
}
```

## Флаги состояния страницы
Поле __flags__ struct page имеет несколько флагов (/include/linux/page-flags.h), описывающих состояние страницы:
* __PG_uptodate__ - выставляется после полного чтения страницы с диска (носителя), если не произвошло ошибки.
* __PG_dirty__ - данные на страничке новее, чем на диске.
* __PG_writeback__ - выставляется при начале записи страницы на носитель.

## Сброс данных на носитель
Когда __Page cache__ работает в режиме __write back__, то может быть несколько ситуаций, когда ядро начнёт сбрасывать данные из кэша на диск:
* Закончилась память в системе, позвали reclaim:
    + Чистый page cache
    + Грязный page cache
    + Анонимную память
    + и т.д.
* Достигнут предел разрешённого размера грязного __page cache__.
* _umount()_ - размонтирование файловой системе.
* _fsync()_ или _sync()_ - принудительная синхронизация. _fsync()_ - сбрасывает кэш файла. _sync()_ - сбрасывает весь грязный __page cache__.
* pdflush - kernel_thread, который отвечает за сброс грязного кэша на носитель.

Ядро проверяет колличество грязного кэша при выполнении операции _write()_, если файл не замаплен (_mmap()_). Если же файл замаплен, то для учёта грязного кэша, первоначально зануляется бит __w__, разрешающий писать, для замапленной страницы в таблице страниц, на которую указывает регистр __cr3__. Таким образом, страничка мапится только для чтение, и при попытке записи в эту страницу произойдёт исключение, которое уже можно обработать, пометив страницу грязной и разрешив в неё запись.

При начале записи на носитель выставляется флаг __PG_writeback__ и очищаается бит __w__ соответствующей записи в таблице страниц.

Если есть несколько процессов, работающих с одинаковым файлом, но при этом один работает с кэшированием, а другой без, то тогда опирация записи второго будут всегда приводить к сбросу данных на носитель, а чтения к их подкачке.

## Сброс анонимной памяти
С анонимной памятью всё несколько сложнее, чем с файлами. В случае с файлами все страничные кадры подвязаны к __inode__, анонимная память связана с процессами, которые ей владеют, причём одним и тем же анонимным страничным кадром могут владеть несколько процессов, например, при шаренной памяти.

Для решения этой задачи в Linux есть функциональность - _обратное отображение(rmap)_.

Во-первых, анонимная память, от файлов отличается, младшим битом в поле __mapping__ дескриптора страниц __struct page__. Если младший бит равен нулю, то это файл и поле указывает на __inode->address_space__, если равен единице, то это анонимная память и указывает на __anon_vma__.

Во-вторых, поле _mapcount в структуре __page__ содержит количество тех, кто имеет ссылку на данных страничный кадр.

!TODO: разобраться получше и переписать.
В итоге, когда создаётся анонимная область памяти, ядро создаёт структуру __anon_vma__, к которой сцепляются __anon_vma_chain__, содержащие VMA, ссылающиеся на одинаковые __page__.

```c
struct anon_vma {
    struct anon_vma *root;      /* Root of this anon_vma tree */
    struct rw_semaphore rwsem;  /* W: modification, R: walking the list */
    atomic_t refcount;
    unsigned degree;
    struct anon_vma *parent;    /* Parent of this anon_vma */
    struct rb_root rb_root; /* Interval tree of private "related" vmas */
};

struct anon_vma_chain {
    struct vm_area_struct *vma;
    struct anon_vma *anon_vma;
    struct list_head same_vma;   /* locked by mmap_sem & page_table_lock */
    struct rb_node rb;          /* locked by anon_vma->rwsem */
    unsigned long rb_subtree_last;
#ifdef CONFIG_DEBUG_VM_RB
    unsigned long cached_vma_start, cached_vma_last;
#endif
};
```