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
    struct list_head    s_mounts;   /* list of mounts; _not_ for fs use */
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
    unsigned long   i_ino;      /* номер дескриптора */
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

Ссылки на dentry держут не только её потомки. Объект fs типа fs_struct, на который есть ссылка в структуре task_struct, и объект file так же могут хранить ссылки на dentry. C fs связыны команды chroot & chdir, а file создаётся в момент открытие файла (open), чтобы хранить о нём информацию. Сами file храняться в массиве __struct files_struct *files;__, который хранится в task_struct, а номера в массиве - файловые дескрипторы. По файловому дескриптору можно получить inode с помощью фанкции stat().

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

## Удаление файла
Для удаления файла надо сделать системный вызов ulink(). Unlink удалит ссылку из dentry на inode и уменьшит колличество ссылок на inode. При этом у inode описывающей файл есть несколько счётчиков: ссылок и жёстких ссылок. Файл будет удалён только тогда, когда колличество жёстких ссылок станет равно нулю. Если жёстких ссылок больше нуля, а обычных ноль, то объект inode удаляется отолько из оперативной памяти.

При этом может быть ситуация, что был вызван ulink(), а с файлом работает другой процесс. Тогда из dentry не будет удалёна ссылка на inode, чтобы процесс мог и дальше работать с этим файлом, но dentry убирается из dcache, чтобы другие процессы не смогли её найти.

## Монтирование
Монтирование осуществляется с помощью команды:
```
mount -t [FS_TYPE] [PATH_TO_DEV] [MOUNT_POINT]
```

В данном случае выполняется последовательность вызовов: mount -> mount() -> sys_mount(). Системный вызов sys_mount() делает lookup() в поиске __MOUNT_POINT__, чтобы получить соответствующий dentry. После этого sys_mount() ищет структуру file_system_type, соответствующую __FS_TYPE__, чтобы позвать метод mount данной структуры.

Ниже представлен пример функций mount для файловой системы ext4. В итоге данный вызов будет обрабатывать вызов mount_bdev(), который создаст super_block и вернёт указатель на его корневой dentry - s->s_root;
```c
/* /include/linux/fs.h */
struct dentry *(*mount) (struct file_system_type *, int,
                                        const char *, void *);

/* /fs/ext4/super.c */
static struct file_system_type ext4_fs_type = {
    .owner      = THIS_MODULE,
    .name       = "ext4",
    .mount      = ext4_mount,
    .kill_sb    = kill_block_super,
    .fs_flags   = FS_REQUIRES_DEV,
};

static struct dentry *ext4_mount(struct file_system_type *fs_type, int flags,
               const char *dev_name, void *data)
{
    return mount_bdev(fs_type, flags, dev_name, data, ext4_fill_super);
}

/* /include/linux/fs.h */
extern struct dentry *mount_bdev(struct file_system_type *fs_type,
    int flags, const char *dev_name, void *data,
    int (*fill_super)(struct super_block *, void *, int));
```

__PATH_TO_DEV__ - путь к тому, что монтируем. В зависимости от типа файловой системы, это может быть путь к устройству, i, или вообще none, как в случае proc_fs. __PATH_TO_DEV__ так же передаётся в метод mount() конкретной файловой системы, которая знает, как это обработать.

После этого нужно настроить связь между корневой dentry монтируемого устройства и dentry того места, куда монтируем.

Начало примонтированной файловой системы описывается структурой mount(/fs/mount.h). Все mount представляют из себя дерего, у них есть указатель на родителя и содержат интересную структуру - struct vfsmount(/include/linux/mount.h), которая описывает dentry корневого каталого и указатель на super_block примонтированной системы.

Moun'ы хранятся в хэш таблице - mount_hashtable. Ключом в которой является пара: dentry точки монтирования и vfsmount родительского mount.

У dentry, к которой монтрируют другую файловую систему, во флагах выставляется бит DCACHE_MOUNTED(/include/linux/dcache.h), тем самым система помечается, что данная dentry - точка монтирования. Таким образом, во время процедуры lookup система видит, что текущая dentry - точка монтирования и ищет соответсвующую mount в mount_hashtable.
```c
/* /fs/mount.h */
struct mount {
    struct hlist_node mnt_hash;
    struct mount *mnt_parent;
    struct dentry *mnt_mountpoint;
    struct vfsmount mnt;
    ...
}

/* /include/linux/dcache.h */
#define DCACHE_MOUNTED          0x00010000 /* is a mountpoint */

/* /fs/namespace.c */
static struct hlist_head *mount_hashtable __read_mostly;

static inline struct hlist_head *m_hash(struct vfsmount *mnt, struct dentry *dentry)
{
    unsigned long tmp = ((unsigned long)mnt / L1_CACHE_BYTES);
    tmp += ((unsigned long)dentry / L1_CACHE_BYTES);
    tmp = tmp + (tmp >> m_hash_shift);
    return &mount_hashtable[tmp & m_hash_mask];
}

/* /include/linux/mount.h */
struct vfsmount {
    struct dentry *mnt_root;    /* root of the mounted tree */
    struct super_block *mnt_sb; /* pointer to superblock */
    int mnt_flags;
};
```

У одной dentry точки монтирования может быть больше одного mount, для этого есть объект mountpoint(/fs/mount.h), где все mount'ы подвязаны в список. Все mountpoint'ы тоже находятся в глобальной хэш таблице - mountpoint_hashtable, где ключём является dentry места монтирования.

```c
/* /fs/mount.h */
struct mountpoint {
    struct hlist_node m_hash;
    struct dentry *m_dentry;
    struct hlist_head m_list;
    int m_count;
};

/* /fs/namespace.c */
static struct hlist_head *mountpoint_hashtable __read_mostly;

static inline struct hlist_head *mp_hash(struct dentry *dentry)
{
    unsigned long tmp = ((unsigned long)dentry / L1_CACHE_BYTES);
    tmp = tmp + (tmp >> mp_hash_shift);
    return &mountpoint_hashtable[tmp & mp_hash_mask];
}
```
