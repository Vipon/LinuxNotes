# VFS
__Файловая система__ - способ организации данных на диске.

__VFS__ - прослойка между системными вызовами и какой-нибудь конкретной файловой системой. __VFS__ предоставляет общий интерфейс для работы с многими файловыми системами.

## Файловые системы
Сами файловые системы условно можно разбить на 3 категории:
* __Блочные (дисковые)__ - управляют данными на локальном носителе (диск, флэш).
    + ext*
    + vfat
    + btrfs
    + xfs
    + ntfs
    + ufs
* __Сетевые__ - обеспечивают доступ к данным на других компьютерах.
    + nfs
    + smbfs
    + ceph
    + zfs
* __Virtual (специальные)__:
    + proc
    + tmpfs
    + sysfs
    + Filesystem in Userspace (FUSE)
    + ramfs
    + encryptfs

## Структуры данных VFS
### struct file_system_type
Описание любой файловой системы начинается со структуры данных - file_system_type (/include/linux/fs.h). Данная структура описывает драйвер файловой системы, являет singleton'ом и содержит название файловой системы, и ссылки на методы монтирования.
```c
struct file_system_type {
    const char *name;
    int fs_flags;
    struct dentry *(*mount) (struct file_system_type *, int,
               const char *, void *); // get_sb() - старая версия
    void (*kill_sb) (struct super_block *);
    ...
}
```

### struct super_block
Конкретные экземпляры файловых систем описываются struct super_block (/include/linux/fs.h). По сути super_block описывает конкретный кусок дерева, куда фаловая система смонтирвона.
```c
struct super_block {
    const struct super_operations   *s_op;  /* таблица виртуальных методов
                                                            супер блока  */
    unsigned long           s_flags;    /* флаги монтирования */
    int                     s_count;    /* счётчики ссылок */
    atomic_t                s_active;
    void                   *s_fs_info;  /* информация о суперблоке конкретной
                                                            файловой системы */
    struct dentry          *s_root;     /* элемент, описывающий корневую
                                                директорию файловой системы */
    ...
    struct list_lru     s_dentry_lru;   /* Списки LRU для кэшей. */
    struct list_lru     s_inode_lru;
    ...
}
```

### Индексный дескриптор
struct inode (/include/linux/fs.h) описывает конкретный файл в fs, файл может быть переименовам, но inode при этом не изменится. Inode адресуются по номерам. В ядре, чтобы быстро найти inode есть большая хэш-тыблица (icache), ключом в которой является пара - {super_block, ino}.
```c
struct inode {
    umode_t         i_mode;     /* тип файла и права доступа */
    kuid_t          i_uid;      /* индефикатор пользователя */
    kgid_t          i_gid;      /* индефикатор группы */
    unsigned int    i_flags;    /* флаги файловой системы */
    const struct inode_operations   *i_op; /* таблица виртуальных методов */
    unsigned long   i_ino;      /* номер дискриптора */
    unsigned int    i_nlink;    /* количество жёстких ссылок */
    struct timespec i_atime;    /* время последжнего доспута */
    struct timespec i_mtime;    /* время последней записи */
    struct timespec i_ctime;    /* время изменения метаданных(inode) */
    ...
}
```

### struct dentry
Struct dentry (/include/linux/dcache.h) описывает запись в директории. Dentry используются для организации файлов и каталогов в древовидную структуру в ядре.
```c
struct dentry {
    unsigned int    d_flags;
    struct qstr     d_name;
    struct dentry   *d_parent;      /* указатель на родительскую директорию */
    struct inode    *d_inode;       /* указатель на соответсвующую inode */
    struct list_head d_child;       /* child of parent list */
    struct list_head d_subdirs;     /* our children */
    struct dentry_operations *d_op; /* таблица виртуальных методов */
    struct hlist_node        d_hash;/* подвязка к dcache */
    struct list_head         d_lru; /* подвязка к списку по времени
                                                            использования */
    ...
}
```

В отличие от inode dentry создаётся в момент обращения или поиска файла. При чём неудачный поиск файла, которому не соответсвует никакой реальный файл, так же создаёт dentry, для ускорения обработки последующих запросов.

Dentry, у которых указатель на inode = NULL, называются negetive и обозначают несуществующий файл. Такие dentry могут появиться, если мы ищем несуществующий файл, или удалили существующий.

Двум и более dentry может соответсвовать общий inode - это жёсткие ссылки. Мягкие ссылки организованы по другому: у inode в поле i_mode стоит бит, сигнализирующий о том, что это символическая ссылка и данные нужно интерпретировать, как путь до целевого файла.

## Процедура lookup
__Процедура lookup__ - процедура превращения пути в объект dentry.

__Путь__ - последовательность имён файлов, разделённых '/'.

__Имя файла__ - любая последовательность символов, кроме '/' и '\0'.

Запустить процедуру lookup можно системным вызовом open(). Рассмотрим работу lookup на примере вызова open("/a/b/c", ...).

Dentry с именем "/" создаётся в момент монтирования файловой системы. Потому создавать эту dentry не надо. Среди дочерних dentry ищется dentry с именем "a". Сначала ищется в кэше - dcache (/fs/dcache.c), организованном в виде глобальной хэш таблицы. Ключом в этой хэш таблицы является пара {имя, parent_dentry}.
```c
static struct hlist_bl_head *dentry_hashtable __read_mostly;

void __d_lookup_done(struct dentry *dentry)
{
    struct hlist_bl_head *b = in_lookup_hash(dentry->d_parent,
                         dentry->d_name.hash);
    ...
    INIT_LIST_HEAD(&dentry->d_lru);
}
```

Если среди дочерних dentry искомая не найдена, то ядро создаёт dentry c d_inode = NULL и обращается к файловой системе, отдавая ей пару {inode_parent, dentry}. Если файл существует, то файловая система записывает номер inode в соответсвующее поле dentry.
```
_____________     dentry    inode
|           |     _____     _____
|super_block| --> |"/"| --> | 1 | --> i_op->lookup()
|           |     -----     -----
-------------       \
                    \/
                   _____  Если существует  _____
                   |"a"| --------------->  | 13|
                   -----      |            -----
                              \ не существует
                               ---------------> NULL (negative dentry)
```

Это был пример работы для блочной файловой системы. Однако в случае сетевой всё должно быть немного подругому, так как файл в таком случае сможет создаться без нашего ведома, а кэш по прежнему будет говорить нам, что файла нет.

Каждая файловая система может предоставить метод d_revalidate в списке виртуальных методов dentry. Данный метод всегда позовётся, если есть запись dentry в кэше. Возможная политика предварительной проверки достоверности кэша - таймаут.

## Освобождение памяти
Со временем может понадобиться освободить память, например, она просто может закончится. Об этом первым узнает buddy алокатор и позовёт reclaimer. Reclaimer чистит чистый дисковый кэш, грязный дисковый кэш, анонимную память, ядерные кэши и т.д.

Dcache - ядерный кэш. Его ядро старается очищать по принципу LRU, для этого все dentry подвязаны в список с помощью поля dentry->d_lru. Голова самого списка находится внутри соответсвующего super_block.

При этом дерево dentry чистится снизу вверх, если функции очистки попадается dentry, у которой есть потомки, то она выкидывается из списка LRU, где окажется после того, как счётчик ссылок на неё станет равен нулю.

Ссылки на dentry держут не только её потомки. Объект fs типа fs_struct, на который есть ссылка в структуре task_struct, и объект file так же могут хранить ссылки на dentry. C fs связыны команды chroot & chdir, а file создаётся в момент открытие файла (open), чтобы хранить о нём информацию. Сами file храняться в массиве __struct files_struct *files;__, который хранится в task_struct, а номера в массиве - файловые дискрипторы.

```c
/* /include/linux/fs_struct.h */
struct fs_struct {
    int users;
    spinlock_t lock;
    seqcount_t seq;
    int umask;
    int in_exec;
    struct path root, pwd;
};

/* /include/linux/path.h */
struct path {
    struct vfsmount *mnt;
    struct dentry *dentry;
};

/* /include/linux/fs.h */
struct file {
    struct path     f_path;
    struct inode    *f_inode;   /* cached value */
    const struct file_operations    *f_op;

    fmode_t         f_mode;
    loff_t          f_pos;
    ...
}
```
