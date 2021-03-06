## Цель семинара
Научиться работать в командной строке

## Запуск командной строки

Доступ к командной строке осуществляется:
 * Запуском программы-терминала
 * Локальным входом в систему (при отсутствии графического рабочего окружения)
 * Удаленным входом по SSH

## Команды и аргументы

Многие команды принимают *аргументы*. Аргументы отделяются от команды символом
пробела, и между собой разделяются также пробелами.

Пример:

```
mkdir -p ~/new_dir
```

Здесь `mkdir` - это имя команды, а `-p` и `~/new_dir` - это первый и второй
аргументы. Сама команда - это аргумент с индексом `0`. При реализации своих
программ на Си/C++ аргументы могут быть получены из функции `main`:

```
/* my_mkdir.c */
/* Call example:
   my_mkdir -p ~/new_dir
*/
int main(int argc, char * argv[])
{
    const char * const progName = argv[0];  // "my_mkdir"
    const char * const firstArg = argv[1];  // "-p"
    const char * const secndArg = argv[2];  // "/home/username/new_dir"
                    // "~" - это сокращение для пути к домашнему каталогу
                    // текущего пользователя, используемое командным
                    // интерпретатором. При передаче аргумента в программу,
                    // это сокращение преобразуется в полное имя
    ...
    if ( ... ) {
        // Success
        return 0;  // код возврата из main запоминается интерпретатором
                   // после завершения работы программы. Значение 0 является
                   // стандартом де-факто для ситуаций, когда работа программы
                   // завершилась успешно
    }
    else {
        // Something wrong
        assert(0 != errorCode);
        return errorCode;  // не нулевые значения кода возврата обычно
                           // используются для сообщения о том, что программа
                           // не выполнила то, что от нее требовалось
    }
}

```

На языке Python доступ к аргументам осуществляется с помощью модуля `sys`
стандартной библиотеки:

```
# my_mkdir.py -p ~/new_dir

import sys
prog_name = sys.argv[0]  # "my_mkdir.py"
first_arg = sys.argv[1]  # "-p"
secnd_arg = sys.argv[2]  # "/home/username/new_dir"

.....

if success:
    sys.exit(0)
else:
    sys.exit(error_code)

```

## Аргументы, содержащие символ пробела

Пробел является разделителем между аргументами. В некоторых случаях требуется,
чтобы он был значением аргумента. Есть два способа передать пробел в аргумент:
 * экранировать его с помощью символа `\`;
 * заключить весь аргумент в кавычки.

**Пример:**

```
mkdir ~/Program\ Files
mkdir ~/"Program Files (x86)"
```

Обратите внимание, что во втором случае первые два символа аргумента `~/`
**не заключены** в кавычки.

**Вопрос:** Почему не будет работать команда `mkdir "~/Program Files (x86)"`?

<!-- Правильный ответ:
Если ~ находится в кавычках, то этот символ трактуется именно как символ, а не
раскрывается в имя домашнего каталога.
-->

## Перенаправление вывода команд в файл

Команда `echo` выводит в *стандартный поток вывода* некоторый
произвольный текст. Пример ее использования:

```
echo Hello, World!
```

Обычно стандартный поток вывода отображается в терминале, но его содержимое
может быть *перенаправлено* в другой файл или другой команде.

Этот способ может использоваться для создания текстовых файлов, не используя
текстовые редакторы:

```
echo Hello, World! >hello.txt
```

Здесь будет создан текстовый файл `hello.txt`, который содержит текст,
созданный командой `echo`. При этом, на экране текст не отобразится

Оператор `>` очищает содержимое существующего файла, либо создает новый файл.

Если нужно сохранить предыдущее содержимое, то используется оператор `>>`:

```
echo Hello >hello.txt
echo World! >>hello.txt
```

## Поток вывода и поток ошибок

На экран терминала отображается содержимое двух текстовых потоков:
*потока вывода* и *потока ошибок*. В некоторых терминалах поток ошибок
выделяется красным цветом.

Операторы `>` и `>>` по умолчанию подразумевают вывод содержимого
потока вывода.

```
> python print_stdout_stderr.py >out.txt
Text of stderr

# Файл out.txt содержит текст "Text of stdout"
```

Содержимое print_stdout_stderr.py из примера выше:

```
import sys
sys.stdout.write("Text of stdout\n")
sys.stderr.write("Text of stderr\n")
```

Потоку вывода соответствует *файловый дескриптор* `1`, а потоку ошибок - `2`.
Запись `КОМАНДА >ФАЙЛ` эквивалентна записи `КОМАНДА 1>ФАЙЛ`.

Для того, чтобы перенаправить поток ошибок в файл, нужно явно указать его
файловый дескриптор:

```
python print_stdout_stderr.py 1>out.txt 2>err.txt
```

Перенаправлять потоки можно не только в физически существующие файлы, но и
в другие потоки:

```
python print_stdout_stderr.py 2>&1  # вывести оба потока в поток вывода

python print_stdout_stderr.py >combined_out.txt 2>&1
                    # вывести все содержимое в файл combined_out.txt
```


## Поток ввода

Некоторые программы ожидают ввод данных с клавиатуры. В UNIX-подобных
системах такие программы читают данные из *потока ввода*. Любой команде
можно указать произвольный файл в качестве потока ввода, - содержимое этого
файла будет использовано в качестве ввода с клавиатуры.

Для перенаправления потока ввода используется оператор `<`.

**Пример:**

```
# Программа enumerate_lines.py читает текст из потока ввода,
# и выводит исходные строки с нумерацией

python enumerate_lines.py <lorem_ipsum.txt

```

## Комбинации клавиш `Ctrl+D` и `Ctrl+C`

Реализация программы `enumerate_lines.py`, из предыдущего примера:

```
import sys
lines = sys.stdin.readlines()
for number, line in enumerate(lines):
    print("%2d: %s" % (number, line[:-1]))
```

Данная реализация читает все строки из входного потока, после чего
выполняет вывод строк с нумерацией.

В какой момент происходит завершение выполнения метода `readlines()`?

Когда программа читает данные из файла, как в примере выше, все строки
считаются прочитанными тогда, когда закончился файл.

Если чтение происходит с клавиатуры, то принудительно *закрыть
входной поток* можно с помощью комбинации клавиш `Ctrl+D`.

Многие программы, например `bash` или `python` считают закрытие
потока ввода признаком того, что пора завершать свою работу (то есть
эта операция эквивалентна команде `exit`).

Для принудидельного завершения работы программы **до того, как она выполнила
свою работу**, используется сочетание клавиш `Ctrl+C`, которое посылает
выполняемой программе сигнал о прерывании работы. Некоторые программы могут
этот сигнал перехватывать и игнорировать, либо реагировать на него каким-то
особенным образом, например `python` создает соответствующую исключительную
ситуацию.

## Команда `cat`
Выводит содержимое одного или нескольких файлов в стандартный поток вывода.

**Пример:**

```
cat file1.txt file2.txt  # на экран будет выведено содержимое сначала
                         # file1.txt, затем file2.txt
```

Эта команда часто используется для слияния нескольких файлов в один (в том
числе не только текстовых, но и бинарных):

```
cat header.txt body.txt footer.txt > document.txt
```

Если вызвать команду `cat` без аргументов, то в качестве входа испоьзуется
*стандартный поток ввода*. Таким способом можно создавать текстовые файлы,
не используя текстовый редактор:

```
cat > new_file.txt  # Будет осуществляться ввод с клавиатуры до нажатия Ctrl+D
```


## Команды `more` и `less` и перенаправление потока
Не все терминалы имеют возможность прокрутки содержимого, поэтому вывод командой
`cat` (или любой другой командой) содержимого большого текстового файла не
влезет на один экран.

Для того, чтобы организовать возможность прокрутки, предусмотрены команды
`more`, или `less`, которые принимают на вход текст из стандартного потока
ввода, и выводят его на экран постранично.

Команда `more` имеет возможность только прокрутки вперед с помощью клавиши
`ПРОБЕЛ`, команда `less` позволяет прокручивать текст как вперед, так и назад.

```
cat lorem_ipsum.txt | less
```

В данном примере программа `cat` выводит содержимое файла `lorem_ipsum.txt`
в стандартный поток вывода, а программа `less` отображает данные на экране из
стандартного потока ввода.

Оператор `|` между командами соединяет стандартный поток
вывода первой команды (`cat lorem_ipsum.txt`) со стандартным потоком
ввода второй команды (`less`), образуя *канал* передачи данных.

### `grep`
Выполняет фильрацию строк входного потока по *регулярному выражению* или
входжению некоторого текста и выводит результат в поток вывода.

Шаблон поиска является аргументом (заключается в кавычки, если содержит
пробельные символы или начинается с символа `-`). Часто используемые опции:
 * `-F` - выполнять поиск простого текста, а не регулярного выражения
 * `-v` - инвертировать условие поиска на противоположное
 * `-q` - ничего не выводить на экран (если требуется только код возврата)

Если найдена хотя бы одна строка, то код вовзрата равен `0`, иначе `1`.
Код возврата, отличный от `0` и `1` означает ошибку.

**Пример использования:**

**Файл some_text.txt:**

```
Hello, World!
# Hello
# This is a comment
This is not a comment
```

**Вывести все строки, содержащие слово 'Hello':**

```
cat some_text.txt | grep -F Hello
```

**Вывести все строки, содержащие фразу 'a comment':**

```
cat some_text.txt | grep -F "a comment"
```

**Вывести все строки, начинающиеся с символа решетки:**

```
cat some_text.txt | grep ^#
  # символ ^ в регулярном выражении обозначает начало строки
```

**Вывести все строки, НЕ начинающиеся с символа решетки:**

```
cat some_text.txt | grep -v ^#
```

**Определить, есть ли в тексте строка, начинающаяся с символа решетки:**

```
cat some_text.txt | grep -q ^#
echo $?  # результат 0, если найдена хотя бы одна строка
```

## Основные команды, о которых нужно помнить всегда

 * `man` - **самая нужная команда**; отображает справку по какой-либо команде
 * `cd` - перейти в другой каталог
 * `ls` - вывести содержимое каталога
 * `mkdir` - создать каталог
 * `cp` - скопировать файл или каталог
 * `mv` - переименовать файл или каталог
 * `rm` - удалить файл или каталог.

Для большинства команд предусмотрен аргумент `--help`, который отображает
короткую инструкцию по использованию команды.

### Команда `man`

`man РАЗДЕЛ ИМЯКОМАНДЫ` выводит справочную информацию по стандартным командам
UNIX, системным вызовам или функциям стандартной библиотеки Си.

Документация разбита на несколько разделов:
 * `1` - команды оболочки
 * `1p` - команды оболочки; выводится документация только на ту
 функциональность, которая соответствует стандарту POSIX
 * `2` - системные вызовы ядра операционной системы
 * `3` - функции стандартной библиотеки Си
 * `3p` - Си-функции POSIX API
 * `4` - описания модулей ядра и системных устройств
 * `5` - описания форматов конфигурационных файлов
 * `6` - описания программ для X11
 * `7` - различные руководства, не попадающие под какую-либо классификацию
 * `8` - руководства по администрированию
 * `n` - руководства Tcl.

Указывать раздел не обязательно. В этом случае будет отображен первый
найденный результат (как правило, руководство по команде). Некоторые
дистрибутивы, например openSUSE, сконфигурированы таким образом, что
команда `man` явно спрашивает номер раздела, если по запросу есть
несколько документов.

**Примеры использования**:

```
man 1p read  # отобразить справку по команде read
man 2 read   # отобразить справку по системному вызову read
man readline # отобразить справку по readline из <stdio.h>
man sudo     # отобразить справку по команде sudo
```

В последних двух случаях нужная страница присутствует только в одном
разделе справки, поэтому номер раздела не указан.


### Команда `ls`

Выводит в *стандартный поток вывода* (обычно это экран или окно терминала)
содержимое указанного каталога. Если каталог не указан в качестве аргумента,
то выводит содержимое текущего каталога.

```
ls ОПЦИИ КАТАЛОГ
```

Опции:
 * `-A` - отображать *скрытые файлы*, то есть те файлы, имена которых
 начинаются с точки
 * `-R` - рекурсивно пройти по всем подкаталогам и вывести их содержимое.


### Команда `mkdir`

Создает новый каталог. Опция `-p` указывает, что нужно создавать все
подкаталоги, которые не существуют в пути.

```
mkdir ~/new_dir  # новый каталог new_dir в домашнем каталоге
mkdir -p ~/new_dir/new_subdir1/new_subdir2  
                 # новый подкаталог new_subdir2 внутри
                 # ~/new_dir/new_subdir1, которые также создаются
```

### Команды `cp` и `mv`

Команда `cp` копирует файл, а команда `mv` перемещает (равнозначно
-- переименовывает) файл.

Первый аргумент - исходный файл или каталог; второй аргумент - новое имя.
Если копируется или перемещается обычный файл, а в качестве второго аргумента
указано имя каталога, то файл будет скопирован/перемещен в указанный каталог
с сохранением своего имени.

При указании опции `-R`, можно рекурсивно копировать каталоги.

### Команда `rm`

**Внимание!** Это деструктивная команда, используйте ее с осторожностью!

Команда `rm` удаляет файл или каталог. Для удаления каталога необходима
опция `-r`.


## Подстановка вывода команд

Вывод команд может быть направлен не только в файл, но и использован как
фрагмент другой команды. Для этого команда с необходимыми аргументами
заключается в *обратные одинарные кавычки* (этот символ находится на
PC-клавиатуре на одной клавише с буквой `Ё`).

**Пример.** Команда `date` выводит текущую дату и время. В качестве аргумента
ей можно передать формат вывода, например `date +%Y%m%d-%H%M` выведет дату
и время в формате `ГГГГММДД-ЧЧММ`. Допустим, нам требуется создать временный
каталог, в имени которого присутствует дата и время. Тогда можно использовать
вывод команды `date` для формирования аргумента команды `mkdir`:

```
mkdir temp-`date +%Y%m%d-%H%M`
```
