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
    unsigned long           s_flags;    // флаги монтирования
    int                     s_count;    // счётчики ссылок
    atomic_t                s_active;
    void                   *s_fs_info;  /* информация о суперблоке конкретной
                                                            файловой системы */
    struct dentry          *s_root;     /* элемент, описывающий корневую
                                                директорию файловой системы */
    ...
}
```

### Индексный дескриптор
struct inode описывает конкретный файл в fs, файл может быть переименовам, но inode при этом не изменится. Inode адресуются по номерам. В ядре, чтобы быстро найти inode есть большая хэш-тыблица (icache), ключом в которой является пара - {super_block, ino}.
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
