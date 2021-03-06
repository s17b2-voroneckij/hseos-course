# Move-семантика в языке C++ 11

## Постановка задачи

Рассмотрим класс-оболочку для записи файлов:

```cpp
class FileWriter {

  int fd_;
  std::string name_;

  public:
    FileWriter();
    FileWriter(const std::string & name);
    bool isValid() const;
    std::string name() const;
    size_t writeText(const std::string & s);
    ~FileWriter();

};

FileWriter::FileWriter()
  : fd_(-1)
{
}

FileWriter::FileWriter(const std::string & name)
  : name_(name)
{
  fd_ = ::open(name.c_str(), O_WRONLY|O_CREAT, 0666);
}

bool FileWriter::isValid() const
{
  return fd_ >= 0;
}

std::string FileWriter::name() const
{
  return name_;
}

size_t FileWriter::writeText(const std::string & s)
{
  return fd_ >= 0
      ? ::write(fd_, s.c_str(), s.length())
      : size_t(0);
}

FileWriter::~FileWriter()
{
  if (isValid()) {
    ::close(fd_);
  }
}

void writeSomeText(FileWriter writer)
{
  static const std::string Data = "Hello!";
  assert(Data.length() == writer.writeText(Data));
  std::cout << "Successfully written to " << writer.name() << std::endl;
}
```

Файл автоматически закрывается (если, конечно, он ранее был открыт) с помощью деструктора.

Возможна ситуация, когда файл закрывается еще раз, хотя он уже был закрыт ранее.


```cpp
void writeSomeText(FileWriter writer)
{
  static const std::string Data = "Hello!";
  assert(Data.length() == writer.writeText(Data));
  std::cout << "Successfully written to " << writer.name() << std::endl;
}

int main()
{
  FileWriter a("/tmp/out.txt");
  FileWriter b = a;

  assert(a.isValid());
  assert(b.isValid());

  writeSomeText(a);  // OK
  writeSomeText(b);  // Ошибка! Файл, который был связан с 'a',
                     // а следовательно и с 'b', уже закрыт!

  return 0;
}
```

Другой неприятный случай - попытка использовать уже закрытый файл:

```cpp
int main()
{
  FileWriter b;

  /* другая область видимости */ {
    FileWriter a("/tmp/out.txt");
    assert(a.isValid());
    b = a;
    assert(b.isValid());
  }

  static const std::string Data = "Hello!";

  // Ошибка! Переменная b хоть и существует, но
  // данные стали некорректными из-за уничтожения
  // переменной a, поверхностной копией которой является переменная b
  assert(Data.length() == b.writeText(Data));

  return 0;
}
```

Избежать таких ситуаций можно, если гарантировать, что для каждого ресурса `int fd_`
будет существовать его *единственный владелец*, который во-первых, обязан освободить занятый ресурс,
когда к нему будет потерян доступ, а во-вторых, никто кроме владельца не имеет права
освобождать этот ресурс.

Первое условие (гарантия освобождения ресурса) реализуется с помощью семантики *деструктора*.

При уничтожении объекта-оболочки (с помощью оператора `delete` или при выходе за пределы
видимости переменной, выделенной в стеке), будет вызван метод-деструктор, который закроет файл.

Второе условие (гарантия **единственного** освобождения ресурса) можно гарантировать запретом
копирования дескриптора `int fd_` другому объекту класса `FileWriter`.


```cpp
class FileWriter {

  . . .

  public:
    explicit FileWriter();
    explicit FileWriter(const std::string & name);

    . . .

};
```

Ключевое слово `explicit` перед объявлением конструктора означает, что компилятор не будет
создавать конструкторы, которые **явным образом** не объявлены в классе: а) конструктор по
умолчанию (без параметров); б) конструктор копирования.

Помимо использования `explicit`, можно запретить конструктор копирования еще двумя способами:

 1. Объявить конструктор в `private:`-секции класса
 2. Использовать конструкцию `= delete` после объявления (начиная с C++11)


Тем не менее, в некоторых случаях нужно иметь возможность "присвоить" значение некопируемого
объекта какой-либо другой переменной, например, для хранения в контейнере или массиве:

```cpp
int main(int argc, char *argv[])
{
  std::deque<FileWriter> files;
  for (int i=1; i<argv; ++i) {
    FileWriter writer(argv[i]);
    files.push_back(writer); // Ошибка компиляции!!!
  }
  . . .
}
```

Возникает дилемма: как выполнить "присваивание", не нарушая вышеописанных принципов *владения*
ресурсами?


## Перемещение объектов

В стандарте C++ 2011 года (и более поздних стандартов) предусмотрена операция *перемещения*
объектов, которые по каким-либо причинам нельзя копировать:

```cpp
  FileWriter a("/tmp/out.txt");
  assert(a.isValid());          // OK
  FileWriter b = std::move(a);
  assert(b.isValid());          // OK
  assert(a.isValid());          // Ошибка!

```

Функция `std::move` *перемещает* содержимое объекта из одной переменной в другую. Таким образом,
после вызова `b = std::move(a)`, с переменной `b` можно работать точно так же, как с переменной
`a`, но при этом, значение в `a` становится недействительным.

Что именно происходит во время перемещения, и как происходит инвалидация премещаемой переменной,
- это определяется *конструктором перемещения*, который должен быть явным образом
реализован для класса.

```cpp
class FileWriter {

  . . .

  public:
    explicit FileWriter();
    explicit FileWriter(const std::string & name);
    explicit FileWriter(FileWriter && other);

    . . .

};

FileWriter::FileWriter(FileWriter && other)
{
  // Используем существующий дескриптор
  // из перемещаемого объекта
  fd_ = other.fd_;

  // Забираем у объекта other возможность
  // что-либо сделать с открытым дескриптором
  other.fd_ = -1;
}

```

Параметром конструктора перемещения является `rvalue`-ссылка, которая обозначается двумя
символами `&&`. Этот конструктор должен инициализировать поля объекта **уже существующими**
полями из другого объекта, после чего - сделать их недоступными исходному объекту.


## Заимствование объектов

Иногда требуется выполнить какие-либо действия над экземплярами некопируемого объекта без явной
передачи владения, например с помощью сторонней функции. Это можно сделать, если *одолжить* объект,
то есть передать его во временное пользование, без права уничтожения. На языке C++ это можно сделать
с помощью ссылки.

```cpp
// Обратите внимание на & в аргументе
void writeSomeData(FileWriter & reference);

int main()
{
  // Функция main создает объект writer и
  // является его владельцем
  FileWriter writer("/tmp/out.txt");

  // Объект writer передается во временное
  // пользование функции writeSomeData
  // (по ссылке)
  writeSomeData(writer);

  return 0; // В этот момент удаляется writer
}

void writeSomeData(FileWriter & reference)
{
  static const std::string = "Hello";

  // Функция writeSomeData может делать с заимствованным
  // объектом все что угодно, кроме удаления
  reference.writeText(Data);

  // При выходе из функции, удаляется только ссылка reference,
  // а сам исходный объект writer, владельцем которого является
  // функция main, продолжает свое существование
}
```

## Некопируемые объекты в стандартной библиотеке C++

Примерами некопируемых объектов можно считать:

 1. `std::unique_ptr<T>` - уникальный smart-указатель; как можно догадаться из названия,
 объект, на который ссылается этот указатель, должен существовать ровно в единственном
 экземпляре, и освобождаться при уничтожении этого указателя;

 2. `std::ifstream` и `std::ofstream` - классы, объекты которых связаны
 с файлами на диске;

 3. `std::thread` - нить выполнения, с которой связан свой стек и выполняемое состояние;
 понятие копирования для выполняемого состояния не определено.


## Некопируемые объекты и контейнеры

Стандартные контейнеры могут использовать конструкторы перемещения в том случае, когда у
объекта не существует конструктора копирования.

Например, сигнатура `std::vector::push_back` [определена](http://www.cplusplus.com/reference/vector/vector/push_back/)
следующим образом:

```cpp
void push_back (const value_type& val); // (1)
void push_back (value_type&& val);      // (2)
```

Реализация первого варианта использует явное копирование, поэтому на элементы вектора
накладывается ограничение: они должны быть копируемыми, иначе использование `push_back`
приведет к ошибке компиляции.

Второй вариант подразумевает, что для класса элементов вектора определена операция
перемещения.

Аналогичные move-сигнатуры методов добавления и доступа существуют и для других
стандартных контейнеров.
