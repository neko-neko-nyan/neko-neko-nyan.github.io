---
title: Установка Windows 10 из под работающей Windows 10
date: 2023-01-08 16:50:00 +0300
categories: [Администрирование, Windows]
tags: [Windows, Установка]
---

## Зачем?

Например, в компьютере не работают usb порты или он не хочет грузиться с флешки.

В моем случае на компе была Windows 10 1904 на HDD, да еще и с кривыми настройками:
- загрузка BIOS, а не EFI (зачем это нужно -- расскажу как-нибудь потом),
- режим sata стоял в legacy, то есть BIOS эмулировал IDE диск, что окончательно убивало производительность.

Жить так дальше было нельзя, и я поставил в компьютер SSD на 120ГБ. Теперь на него надо поставить систему, но
установщик упорно отказывался запускаться -- загрузка "висла" на логотипе windows. Возможно, кривой BIOS не хотел
запускаться с флешки, а может ему не хватало памяти -- её было всего 2 гига.

После этого было решено ставить систему на соседний SSD диск из работающей на HDD windows.

## Тонкости

> Поставить систему таким образом можно только на другой диск
{: .prompt-warning }

Очевидно, что диск, на который мы будем ставить новую систему, должен старой системой видиться.

Режим загрузки (EFI/BIOS) новой системы не зависит от режима загрузки старой. Для разных режимов **новой** системы
инструкции разные -- различия незначительные, но если вберете не тот режим -- придется начинать все заново, а проверить,
правильно ли был выбран режим можно только после полной установки системы (если она не загружается -- значит неправильный).

Но сначала нужно подготовить файл для установки.


## Подготовка

Чтобы установить window нам, очевидно, нужен образ. Хотя для этого способа подойдет и готовая флешка.

### Если есть образ

Монтируем его штатными средствами windows 10 (двойным кликом), открываем папку `sources` и копируем файл `install.wim`
в любое удобное место. Я скопирую в корень диска `C:`. После этого образ можно размонтировать.

### Если есть флешка

На флешке записан тот же самый образ, так что открываем папку `sources` и копируем файл `install.wim`
в любое удобное место.


Вообще можно ставить прямо с флешки, но если во время установки её задеть, то систему придётся переставлять с начала.
Я считаю, что проще скопировать.


## Для EFI загрузки

Для начала надо разметить диск. Управление дисками здесь на не поможет, ведь она не умеет создавать раздел ESP (EFI
System Partition), который нужен для загрузки. По этом идем в командную строку администратора и запускаем `diskpart`.

### Разметка диска

Вводим `list disk`. Видим список дисков. Находим диск, на который будем ставить систему и вводим `sel disk НОМЕР`.

> **Не перепутайте диски!** Можно случайно стереть все (нужные) данные с другого диска, либо сломать старую систему, не установив новую.
> Данные на выбранном диске будут стерты.
{: .prompt-danger }


```console
C:\Windows\System32>diskpart

Microsoft DiskPart version 10.0.22621.1

Copyright (C) Microsoft Corporation.
On computer: NEKO-PC

DISKPART> list disk

  Disk ###  Status         Size     Free     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  Disk 0    Online         1863 GB      0 B
  Disk 1    Online          119 GB   119 GB
DISKPART> sel disk 1
```

Очищаем диск:

```console
clean
```

И конвертируем его в GPT. Напомню, что в windows есть жесткая привязка типа таблицы разделов к режиму загрузки:
MBR в BIOS, GPT в EFI. По другому работать не будет.

```console
convert gpt
```

Создаем разделы. Можно попробовать поменять размеры, но не факт, что тогда всё заработает. Я показываю как делает
штатный установщик.

Сначала раздел ESP (EFI):

```console
create part EFI size=100
format fs=fat32 quick
assign letter=S
```

MSR (он же MicroSoft Reserved):

```console
create part msr size=16
```

Затем основной (системный) раздел:

```console
create part prim
shrink minimum=500
format fs=ntfs quick
assign letter=W
```

И, наконец, раздел восстановления aka recovery:

```console
create part prim
format format quick fs=ntfs
assign letter=R
set id="de94bba4-06d1-4d40-a16a-bfd50179d6ac"
gpt attributes=0x8000000000000001
```

Последние две команды пометят этот раздел как recovery и запретят его изменение.

Проверяем, должно быть так:

```console
DISKPART> list part

  Partition ###  Type              Size     Offset
  -------------  ----------------  -------  -------
  Partition 1    System             100 MB  1024 KB
  Partition 2    Reserved            16 MB   101 MB
  Partition 3    Primary            118 GB   117 MB
  Partition 4    Recovery           606 MB   118 GB
```

Закрываем diskpart командой `exit`.


### Установка системы

Вспоминаем, что у нас есть файл `install.wim`. Для начала посмотрим какие в нем есть редакции:

```console
C:\Windows\System32>dism /get-wiminfo /wimfile:C:\install.wim

Deployment Image Servicing and Management tool
Version: 10.0.22621.1

Index: 1
Name: Windows 10 Home
Description: Windows 10 Home
Size (byte): 14 742 001 112

...
```

Выбираем нужную и ее **индекс** пишем в следующей команде:

```console
C:\Windows\System32>dism /apply-image /wimfile:C:\install.wim /index:ИНДЕКС /applydir:w:\

Deployment Image Servicing and Management tool
Version: 10.0.22621.1

Applying image
...
```

Установка началась, ждём завершения.

Осталось установить загрузчик.

### Установка загрузчика

Тут всё совсем просто. Вводим команду:

```console
C:\Windows\System32>bcdboot w:\Windows /s s: /f UEFI
```

Можно перезагружаться в новую систему. При первом запуске, как и всегда, будет установка драйверов. Отличие от обычной
установки будет только в том, что при первом запуске будет дополнительная страница с лицензионным соглашением.





## Для BIOS загрузки

Да, я скопировал инструкцию для EFI и изменил те пункты, где есть различия.

Для начала надо разметить диск. Управление дисками здесь на не поможет, ведь она не умеет создавать раздел
ESP (EFI System Partition), который нужен для загрузки- `set id` она тоже не умеет. По этом идем в командную строку администратора и запускаем `diskpart`.

### Разметка диска

Вводим `list disk`. Видим список дисков. Находим диск, на который будем ставить систему и вводим `sel disk НОМЕР`.

> **Не перепутайте диски!** Можно случайно стереть все (нужные) данные с другого диска, либо сломать старую систему, не установив новую.
> Данные на выбранном диске будут стерты.
{: .prompt-danger }


```console
C:\Windows\System32>diskpart

Microsoft DiskPart version 10.0.22621.1

Copyright (C) Microsoft Corporation.
On computer: NEKO-PC

DISKPART> list disk

  Disk ###  Status         Size     Free     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  Disk 0    Online         1863 GB      0 B
  Disk 1    Online          119 GB   119 GB
DISKPART> sel disk 1
```

Очищаем диск:

```console
clean
```

И конвертируем его в MBR. Напомню, что в windows есть жесткая привязка типа таблицы разделов к режиму загрузки:
MBR в BIOS, GPT в EFI. По другому работать не будет.

```console
convert mbr
```

Создаем разделы. Можно попробовать поменять размеры, но не факт, что тогда всё заработает. Я показываю как делает
штатный установщик.

Сначала загрузочный раздел:

```console
create part prim size=100
format fs=ntfs quick
assign letter=S
active
```

MSR (он же MicroSoft Reserved):

```console
create part msr size=16
```

Затем основной (системный) раздел:

```console
create part prim
shrink minimum=650
format fs=ntfs quick
assign letter=W
```

И, наконец, раздел восстановления aka recovery:

```console
create part prim
format format quick fs=ntfs
assign letter=R
set id=27
```

Последние две команды пометят этот раздел как recovery и запретят его изменение.

Проверяем, должно быть так:

```console
DISKPART> list part

  Partition ###  Type              Size     Offset
  -------------  ----------------  -------  -------
  Partition 1    System             100 MB  1024 KB
  Partition 2    Reserved            16 MB   101 MB
  Partition 3    Primary            118 GB   117 MB
  Partition 4    Recovery           606 MB   118 GB
```

Закрываем diskpart командой `exit`.


### Установка загрузочного сектора

Записываем загрузочный сектор на наш диск:

```console
C:\Windows\System32>bootsect /nt60 S:
```


### Установка системы

Вспоминаем, что у нас есть файл `install.wim`. Для начала посмотрим какие в нем есть редакции:

```console
C:\Windows\System32>dism /get-wiminfo /wimfile:C:\install.wim

Deployment Image Servicing and Management tool
Version: 10.0.22621.1

Index: 1
Name: Windows 10 Home
Description: Windows 10 Home
Size (byte): 14 742 001 112

...
```

Выбираем нужную и ее **индекс** пишем в следующей команде:

```console
C:\Windows\System32>dism /apply-image /wimfile:C:\install.wim /index:ИНДЕКС /applydir:w:\

Deployment Image Servicing and Management tool
Version: 10.0.22621.1

Applying image
...
```

Установка началась, ждём завершения.

Осталось установить загрузчик.

### Установка загрузчика

Тут всё совсем просто. Вводим команду:

```console
C:\Windows\System32>bcdboot w:\Windows
```

Можно перезагружаться в новую систему. При первом запуске, как и всегда, будет установка драйверов. Отличие от обычной
установки будет только в том, что при первом запуске будет дополнительная страница с лицензионным соглашением.



