# Запуск системи

Запуск PX4 контрольований скриптами оболонки.
На NuttX вони знаходяться у директорії [ROMFS/px4fmu_common/init.d](https://github.com/PX4/PX4-Autopilot/tree/main/ROMFS/px4fmu_common/init.d), деякі з них також використовуються на Posix системах (Linux/MacOS).
Скрипти які використовуються тільки на Posix системах знаходяться у [ROMFS/px4fmu_common/init.d-posix](https://github.com/PX4/PX4-Autopilot/tree/main/ROMFS/px4fmu_common/init.d-posix).

Усі файли, які починаються з числа і підкреслення (наприклад, `10000_airaipl`) є попередньо визначеними конфігураціями планерів.
They are exported at build-time into an `airframes.xml` file which is parsed by [QGroundControl](https://qgroundcontrol.com) for the airframe selection UI.
Як додати нову конфігурацію описано [тут](../dev_airframes/adding_a_new_frame.md).

Файли що залишилися є частиною загальної логіки запуску.
Перший файл що виконується є скрипт [init.d/rcS](https://github.com/PX4/PX4-Autopilot/blob/main/ROMFS/px4fmu_common/init.d/rcS) (або [init.d-posix/rcS](https://github.com/PX4/PX4-Autopilot/blob/main/ROMFS/px4fmu_common/init.d-posix/rcS) на Posix), який викликає інші скрипти.

Наступні секції розділені відповідно до операційної системи, на яких виконується PX4.

## Posix (Linux/MacOS)

На Posix системна оболонка використовується як інтерпретатор скриптів (наприклад, /bin/sh що є символьним посиланням на dash в Ubuntu).
Щоб це працювало потрібно кілька речей:

- Модулі PX4 повинні виглядати для системи як окремі виконувані файли.
  Це робиться за допомогою символьних посилань.
  For each module a symbolic link `px4-<module> -> px4` is created in the `bin` directory of the build folder.
  При виконанні двійкового файлу перевіряється його шлях (`argv[0]`) і якщо це модуль (починається з `px4-`) він відправляє команду на основний екземпляр px4 (див. нижче).

  :::tip
  The `px4-` prefix is used to avoid conflicts with system commands (e.g. `shutdown`), and it also allows for simple tab completion by typing `px4-<TAB>`.

:::

- Оболонка повинна знати, де шукати символьні посилання.
  For that the `bin` directory with the symbolic links is added to the `PATH` variable right before executing the startup scripts.

- Оболонка запускає кожен модуль як новий (клієнтський) процес.
  Кожен клієнтський процес повинен спілкуватися з головним екземпляром px4 (сервером), де справжні модулі працюють як потоки.
  This is done through a [UNIX socket](https://man7.org/linux/man-pages/man7/unix.7.html).
  Сервер прослуховує сокет, до якого клієнти можуть під'єднатися та надіслати команду.
  Сервер відправляє вихідні дані та код повернення назад до клієнта.

- The startup scripts call the module directly, e.g. `commander start`, rather than using the `px4-` prefix.
  This works via aliases: for each module an alias in the form of `alias <module>=px4-<module>` is created in the file `bin/px4-alias.sh`.

- Скрипт `rcS` виконується з основного екземпляра Px4.
  Він не запускає жодних модулів, але спочатку оновлює змінну `PATH`, а потім просто запускає оболонку з файлом `rcS` як аргумент.

- Крім того, декілька екземплярів серверу можуть бути запущені для симуляції кількох засобів.
  A client selects the instance via `--instance`.
  В скрипті екземпляр доступний за допомогою змінної `$px4_instance`.

Модулі можна виконувати з будь-якого терміналу, коли PX4 вже запущено в системі.
Наприклад:

```sh
cd <PX4-Autopilot>/build/px4_sitl_default/bin
./px4-commander takeoff
./px4-listener sensor_accel
```

### Динамічні модулі

Зазвичай всі модулі компілюються в єдиний виконуваний файл PX4.
Однак, на Posix системах, є можливість компіляції модуля в окремий файл, який можна завантажити в PX4 використовуючи команду `dyn`.

```sh
dyn ./test.px4mod
```

## NuttX

NuttX має інтегрований інтерпретатор оболонки ([NuttShell (NSH)](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=139629410)), тому скрипти можуть бути виконані безпосередньо.

### Налагодження завантаження системи

Відмова драйверу програмного компонента не призведе до перерваного завантаження.
Це контролюється директивою `set +e` в скрипті запуску.

Послідовність завантаження можна налагодити під'єднавши [системну консоль](../debug/system_console.md) та перезавантажити плату за живленням.
Отриманий журнал завантаження містить детальну інформацію про послідовність завантажування і має містити підказки, чому завантаження переривалось.

#### Основні причини невдалого завантаження

- Для користувацьких додатків: у системі закінчилася оперативна пам'ять.
  Run the `free` command to see the amount of free RAM.
- Відмова програмного забезпечення або припущення яке призвело до трасування стеку.

### Заміна запуску системи

Весь процес завантаження може бути замінений шляхом створення файлу з новою конфігурацією `/etc/rc.txt` на картці microSD (ніщо в старій конфігурації не буде автоматично запущено, і якщо файл порожній, зовсім нічого не буде запущено).

Налаштування стандартного завантаження майже завжди є кращим підходом.
Це описано нижче.

### Налаштування запуску системи

Найкращий спосіб змінити запуск системи - це ввести [нову конфігурацію планера](../dev_airframes/adding_a_new_frame.md).
Файл конфігурації планеру може бути включений у прошивку або на SD карту.

Якщо вам потрібно "підлаштувати" конфігурацію що існує, наприклад запустити один або більше застосунків або встановити значення кількох параметрів, можна вказати це створивши два файли у директорії `/etc/` на SD картці:

- [/etc/config.txt](#customizing-the-configuration-config-txt): modify parameter values
- [/etc/extras.txt](#starting-additional-applications-extras-txt): start applications

Ці файли описані нижче.

:::warning
Системні файли завантаження - це UNIX файли, які потребують закінчення рядків UNIX.
Якщо редагуєте їх на Windows - використовуйте відповідний редактор.
:::

:::info
Ці файли згадуються в коді PX4 як `/fs/microsd/etc/config.txt` та `/fs/microsd/etc/extras.txt`, де коренева директорія microSD карти визначається шляхом `/fs/microsd`.
:::

#### Налаштування конфігурації (config.txt)

Файл `config.txt` можна використовувати для зміни параметрів.
Він завантажується після того, як головна система була налаштована та _перед тим_ як завантажена.

Наприклад, ви можете створити файл на SD картці, `etc/config.txt` з такими значеннями параметрів як показано:

```sh
param set-default PWM_MAIN_DIS3 1000
param set-default PWM_MAIN_MIN3 1120
```

#### Запуск додаткових застосунків (extras.txt)

`extras.txt` можна використовувати для запуску додаткових застосунків після завантаження основної системи.
Зазвичай це будуть контролери корисного навантаження або подібні необов'язкові користувацькі компоненти.

:::warning
Виклик невідомої команди в файлах завантаження системи може призвести до збою завантаження.
Зазвичай система не транслює повідомлення mavlink після збою при завантаженні, в такій ситуації перевірте повідомлення про помилки, які виведено в системній консолі.
:::

Наступний приклад показує, як запускати користувацькі застосунки:

- Створіть файл на SD картці `etc/extras.txt` із цим вмістом:

  ```sh
  custom_app start
  ```

- Команду можна зробити необов'язковою шляхом оздоблення команди директивами `set +e` та `set -e`:

  ```sh
  set +e
  optional_app start      # Will not result in boot failure if optional_app is unknown or fails
  set -e

  mandatory_app start     # Will abort boot if mandatory_app is unknown or fails
  ```
