# zfs_tuning

Применяет ZFS pool/dataset properties (и создаёт перечисленные датасеты) через `community.general.zfs` (filesystem
level: `compression`, `atime`, `xattr`, `acltype`, `recordsize`) + pool-level
через `zpool set` (`autotrim`). Дополнительно включает штатные systemd-таймеры:

* `zfs-scrub-monthly@<pool>.timer` — раз в месяц scrub
* `zfs-trim-monthly@<pool>.timer` — раз в месяц TRIM (для SSD pools)

Роль **не управляет** kernel module options (`/etc/modprobe.d/`) — это делает
сторонняя роль `linux_helper` через `linux_helper_zfs_{default,cluster,host}_options`.

## Платформы

* ArchLinux / Debian / EL — все с ZFS on Linux + systemd

## Требования

* Ansible 2.10+ + collection `community.general` (для модуля `zfs`)
* ZFS installed + pool'ы созданы (роль НЕ создаёт пулы)
* Systemd с шипуемыми zfs-{scrub,trim}-monthly@.timer (Debian: пакет
  `zfsutils-linux`, Arch: пакет `zfs-utils`)

## Переменные

| Переменная | По умолчанию | Описание |
|---|---|---|
| `zfs_tuning_pools` | `[]` | Список pool-объектов (см. ниже). Если пусто — task'и skip'аются (`meta: end_host`). |

### Структура `zfs_tuning_pools`

```yaml
zfs_tuning_pools:
  - name: <pool_name>           # обязательно
    properties:                  # filesystem-level через `zfs set`
      compression: lz4           # | zstd-3 | off
      atime: 'off'               # 'on' | 'off' | relatime
      xattr: sa                  # sa | on
      acltype: posixacl          # posixacl | off
      recordsize: 1M             # 128K (default) / 1M (для крупных файлов)
    pool_properties:             # pool-level через `zpool set`
      autotrim: 'on'             # для SSD pool'ов
    scrub: monthly               # monthly | none — enable systemd timer
    trim: monthly                # monthly | none — для SSD pool'ов
    datasets:                    # необязательно
      - name: repo               # путь относительно пула → <pool>/repo
        properties:
          canmount: 'off'
          recordsize: 1M
      - name: repo/arch          # без properties — просто создать
```

### Датасеты

* Датасет из списка создаётся, если его нет, и получает `properties`
  (`zfs create` / `zfs set`). Свойства, которых нет в списке, наследуются.
* Роль **ничего не удаляет**: датасеты не из списка и их свойства не трогаются.
* Порядок в списке не важен — роль сортирует по имени, родитель идёт раньше
  потомков (`repo` < `repo/arch`).
* Свойства-размеры (`recordsize`, `volblocksize`, `special_small_blocks`,
  `quota`, `refquota`, `reservation`, `refreservation` —
  `zfs_tuning_size_properties`) можно писать как `1M` / `16K`: перед
  сравнением роль переводит их в байты. `community.general.zfs` читает текущее
  значение через `zfs get -p` (`1048576`), и без перевода `1M` считался бы
  изменением на каждом прогоне. То же действует для `properties` самого пула.
* Синонимы значений (`zfs_tuning_value_aliases`): `acltype: posixacl` роль
  переводит в `posix` — так OpenZFS 2.x хранит и показывает это значение, и без
  замены `posixacl` давал бы changed на каждом прогоне.

## Примеры

### SSD pool с VM-storage

```yaml
zfs_tuning_pools:
  - name: ssd-pool
    properties:
      compression: lz4
      atime: 'off'
      xattr: sa
      acltype: posixacl
    pool_properties:
      autotrim: 'on'
    scrub: monthly
    trim: monthly
```

### HDD backup pool (PBS / vzdump archives)

```yaml
zfs_tuning_pools:
  - name: zfs-backup
    properties:
      compression: zstd-3        # лучше compression для backup, write-once
      atime: 'off'
      xattr: sa
      recordsize: 1M             # большие chunks архивов
    scrub: monthly
```

### Samba file storage с snapshot retention (sanoid снаружи)

```yaml
zfs_tuning_pools:
  - name: bigdata
    properties:
      compression: lz4
      atime: 'off'
      xattr: sa
      acltype: posixacl          # критично для Samba ACL через vfs_acl_xattr
    scrub: monthly
```

## Caveat — applied только к новым записям

`recordsize` / `compression` применяются **только к новым блокам**. Старые
данные на pool остаются в текущем формате. Чтобы пересжать всё —
полный `zfs send | zfs recv` на временный destination → удалить старое
→ recv обратно.

`volblocksize` (для zvol) — **immutable** после create. Для существующих
VM-zvol единственный способ применить новый block size — `qm move-disk`
с пересозданием zvol.

## Пример использования

```yaml
- name: Deploy ZFS pool tuning
  become: true
  hosts: storage_zfs
  roles:
    - zfs_tuning
```

## Зависимости

* `community.general` ansible collection (модуль `community.general.zfs`)
* ZFS on Linux + созданные pool'ы

## Лицензия

MIT
