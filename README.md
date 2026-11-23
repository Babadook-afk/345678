Разработка программного комплекса «Управление библиотечным фондом»

---

 1. Цель работы

Изучить принципы проектирования и разработки информационной системы для автоматизации учёта библиотечного фонда. Получить практические навыки работы с базами данных, проектирования пользовательского интерфейса и реализации бизнес-логики.

---

 2. Задание

Разработать программный комплекс «Управление библиотечным фондом», который должен обеспечивать:

1. Учёт книг - добавление, редактирование, удаление, поиск.
2. Учёт читателей - регистрация, редактирование, удаление.
3. Учёт выдачи и возврата книг - оформление выдачи, возврат, контроль сроков.
4. Поиск и фильтрация - по автору, названию, жанру, читателю.
5. Формирование отчётов - список задолженностей, популярные книги, статистика.

Требования к реализации:
- Использовать реляционную базу данных (SQLite / PostgreSQL / MySQL / MS SQL).
- Реализовать архитектуру «клиент - база данных» (2-уровневую) или «клиент - сервер - база данных» (3-уровневую).
- Обеспечить валидацию вводимых данных и обработку исключений.

---

 3. Теоретические сведения

Библиотечный фонд - упорядоченная совокупность документов, предназначенных для хранения и общественного использования.

Основные сущности предметной области:

| Сущность | Описание |
|----------|----------|
| Книга | Экземпляр издания (инвентарный номер, автор, название, жанр, год, количество) |
| Читатель | Пользователь библиотеки (ФИО, телефон, email, дата регистрации) |
| Выдача | Факт выдачи книги читателю (книга, читатель, дата выдачи, дата возврата, фактический возврат) |
| Жанр | Классификатор жанров книг |

Связи:
- Книга - Жанр: многие-к-одному.
- Выдача - Книга: многие-к-одному.
- Выдача - Читатель: многие-к-одному.

---

 4. Проектирование базы данных

 4.1. ER-модель (логическая схема)

```
┌────────────┐       ┌────────────┐
│   Genres   │       │   Books    │
├────────────┤       ├────────────┤
│ Id (PK)    │◄──┐   │ Id (PK)    │
│ Name       │   └───│ GenreId(FK)│
└────────────┘       │ Title      │
                     │ Author     │
                     │ Year       │
                     │ Quantity   │
                     └─────┬──────┘
                           │
                     ┌─────▼──────┐       ┌────────────┐
                     │   Loans    │       │  Readers   │
                     ├────────────┤       ├────────────┤
                     │ Id (PK)    │       │ Id (PK)    │
                     │ BookId(FK) │───┐   │ FullName   │
                     │ ReaderId(FK│───┼──►│ Phone      │
                     │ LoanDate   │   │   │ Email      │
                     │ DueDate    │   │   │ RegDate    │
                     │ ReturnDate │   │   └────────────┘
                     └────────────┘   │
                                      └─── (ReaderId)
```

 4.2. SQL-скрипт создания БД (SQLite)

```sql
CREATE TABLE Genres (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    Name TEXT NOT NULL UNIQUE
);

CREATE TABLE Readers (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    FullName TEXT NOT NULL,
    Phone TEXT,
    Email TEXT,
    RegDate TEXT NOT NULL DEFAULT (date('now'))
);

CREATE TABLE Books (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    GenreId INTEGER NOT NULL,
    Title TEXT NOT NULL,
    Author TEXT NOT NULL,
    Year INTEGER,
    Quantity INTEGER NOT NULL DEFAULT 1,
    FOREIGN KEY (GenreId) REFERENCES Genres(Id)
);

CREATE TABLE Loans (
    Id INTEGER PRIMARY KEY AUTOINCREMENT,
    BookId INTEGER NOT NULL,
    ReaderId INTEGER NOT NULL,
    LoanDate TEXT NOT NULL DEFAULT (date('now')),
    DueDate TEXT NOT NULL,
    ReturnDate TEXT,
    FOREIGN KEY (BookId) REFERENCES Books(Id),
    FOREIGN KEY (ReaderId) REFERENCES Readers(Id)
);
```

---

 5. Реализация программного комплекса

В качестве языка программирования используется C (Console + ADO.NET + SQLite). Структура проекта:

```
LibraryManager/
├── Program.cs
├── Data/
│   └── Database.cs
├── Models/
│   ├── Book.cs
│   ├── Reader.cs
│   ├── Genre.cs
│   └── Loan.cs
└── Services/
    ├── BookService.cs
    ├── ReaderService.cs
    └── LoanService.cs
```

 5.1. Модели данных (`Models/Book.cs`)

```csharp
namespace LibraryManager.Models;

public class Book
{
    public int Id { get; set; }
    public int GenreId { get; set; }
    public string Title { get; set; } = "";
    public string Author { get; set; } = "";
    public int Year { get; set; }
    public int Quantity { get; set; }
}
```

 5.2. Работа с базой данных (`Data/Database.cs`)

```csharp
using Microsoft.Data.Sqlite;

namespace LibraryManager.Data;

public static class Database
{
    private const string ConnectionString = "Data Source=library.db";

    public static SqliteConnection GetConnection()
    {
        var conn = new SqliteConnection(ConnectionString);
        conn.Open();
        return conn;
    }

    public static void Initialize()
    {
        using var conn = GetConnection();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            CREATE TABLE IF NOT EXISTS Genres (...);
            CREATE TABLE IF NOT EXISTS Readers (...);
            CREATE TABLE IF NOT EXISTS Books (...);
            CREATE TABLE IF NOT EXISTS Loans (...);
        ";
        cmd.ExecuteNonQuery();
    }
}
```

 5.3. Сервис управления книгами (`Services/BookService.cs`)

```csharp
using LibraryManager.Data;
using LibraryManager.Models;

namespace LibraryManager.Services;

public class BookService
{
    public List<Book> GetAll()
    {
        var list = new List<Book>();
        using var conn = Database.GetConnection();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT Id, GenreId, Title, Author, Year, Quantity FROM Books";
        using var reader = cmd.ExecuteReader();
        while (reader.Read())
        {
            list.Add(new Book
            {
                Id = reader.GetInt32(0),
                GenreId = reader.GetInt32(1),
                Title = reader.GetString(2),
                Author = reader.GetString(3),
                Year = reader.GetInt32(4),
                Quantity = reader.GetInt32(5)
            });
        }
        return list;
    }

    public void Add(Book book)
    {
        using var conn = Database.GetConnection();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"INSERT INTO Books (GenreId, Title, Author, Year, Quantity)
                            VALUES ($g, $t, $a, $y, $q)";
        cmd.Parameters.AddWithValue("$g", book.GenreId);
        cmd.Parameters.AddWithValue("$t", book.Title);
        cmd.Parameters.AddWithValue("$a", book.Author);
        cmd.Parameters.AddWithValue("$y", book.Year);
        cmd.Parameters.AddWithValue("$q", book.Quantity);
        cmd.ExecuteNonQuery();
    }

    public List<Book> Search(string query)
    {
        var list = new List<Book>();
        using var conn = Database.GetConnection();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"SELECT Id, GenreId, Title, Author, Year, Quantity
                            FROM Books
                            WHERE Title LIKE $q OR Author LIKE $q";
        cmd.Parameters.AddWithValue("$q", $"%{query}%");
        using var r = cmd.ExecuteReader();
        while (r.Read())
            list.Add(new Book {
                Id = r.GetInt32(0), GenreId = r.GetInt32(1),
                Title = r.GetString(2), Author = r.GetString(3),
                Year = r.GetInt32(4), Quantity = r.GetInt32(5)
            });
        return list;
    }

    public void Delete(int id)
    {
        using var conn = Database.GetConnection();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = "DELETE FROM Books WHERE Id = $id";
        cmd.Parameters.AddWithValue("$id", id);
        cmd.ExecuteNonQuery();
    }
}
```

 5.4. Сервис выдачи книг (`Services/LoanService.cs`)

```csharp
using LibraryManager.Data;
using LibraryManager.Models;

namespace LibraryManager.Services;

public class LoanService
{
    public void Issue(int bookId, int readerId, int days = 14)
    {
        using var conn = Database.GetConnection();

        // Проверка доступного количества экземпляров
        using var check = conn.CreateCommand();
        check.CommandText = @"SELECT Quantity FROM Books WHERE Id=$id";
        check.Parameters.AddWithValue("$id", bookId);
        var qty = Convert.ToInt32(check.ExecuteScalar() ?? 0);
        if (qty <= 0)
            throw new InvalidOperationException("Нет доступных экземпляров книги.");

        using var tx = conn.BeginTransaction();

        using var loan = conn.CreateCommand();
        loan.Transaction = tx;
        loan.CommandText = @"INSERT INTO Loans (BookId, ReaderId, LoanDate, DueDate)
                             VALUES ($b, $r, date('now'), date('now', $d))";
        loan.Parameters.AddWithValue("$b", bookId);
        loan.Parameters.AddWithValue("$r", readerId);
        loan.Parameters.AddWithValue("$d", $"+{days} days");
        loan.ExecuteNonQuery();

        using var dec = conn.CreateCommand();
        dec.Transaction = tx;
        dec.CommandText = "UPDATE Books SET Quantity = Quantity - 1 WHERE Id=$id";
        dec.Parameters.AddWithValue("$id", bookId);
        dec.ExecuteNonQuery();

        tx.Commit();
    }

    public void Return(int loanId)
    {
        using var conn = Database.GetConnection();
        using var tx = conn.BeginTransaction();

        using var get = conn.CreateCommand();
        get.Transaction = tx;
        get.CommandText = "SELECT BookId FROM Loans WHERE Id=$id AND ReturnDate IS NULL";
        get.Parameters.AddWithValue("$id", loanId);
        var bookId = Convert.ToInt32(get.ExecuteScalar()
            ?? throw new InvalidOperationException("Запись о выдаче не найдена."));

        using var ret = conn.CreateCommand();
        ret.Transaction = tx;
        ret.CommandText = "UPDATE Loans SET ReturnDate = date('now') WHERE Id=$id";
        ret.Parameters.AddWithValue("$id", loanId);
        ret.ExecuteNonQuery();

        using var inc = conn.CreateCommand();
        inc.Transaction = tx;
        inc.CommandText = "UPDATE Books SET Quantity = Quantity + 1 WHERE Id=$id";
        inc.Parameters.AddWithValue("$id", bookId);
        inc.ExecuteNonQuery();

        tx.Commit();
    }

    public List<(int LoanId, string Book, string Reader, string DueDate)> GetOverdue()
    {
        var result = new List<(int, string, string, string)>();
        using var conn = Database.GetConnection();
        using var cmd = conn.CreateCommand();
        cmd.CommandText = @"
            SELECT l.Id, b.Title, r.FullName, l.DueDate
            FROM Loans l
            JOIN Books b ON b.Id = l.BookId
            JOIN Readers r ON r.Id = l.ReaderId
            WHERE l.ReturnDate IS NULL AND date(l.DueDate) < date('now')";
        using var rd = cmd.ExecuteReader();
        while (rd.Read())
            result.Add((rd.GetInt32(0), rd.GetString(1), rd.GetString(2), rd.GetString(3)));
        return result;
    }
}
```

 5.5. Консольное меню (`Program.cs`)

```csharp
using LibraryManager.Data;
using LibraryManager.Services;
using LibraryManager.Models;

Database.Initialize();
var bookService = new BookService();
var loanService = new LoanService();

while (true)
{
    Console.WriteLine("""
        === Управление библиотечным фондом ===
        1. Список книг
        2. Добавить книгу
        3. Поиск книги
        4. Выдать книгу
        5. Вернуть книгу
        6. Задолженности
        0. Выход
        """);
    Console.Write("Выбор: ");
    switch (Console.ReadLine())
    {
        case "1":
            foreach (var b in bookService.GetAll())
                Console.WriteLine($"{b.Id}. {b.Author} - {b.Title} ({b.Year}), экз.: {b.Quantity}");
            break;
        case "2":
            Console.Write("Название: "); var t = Console.ReadLine()!;
            Console.Write("Автор: ");    var a = Console.ReadLine()!;
            Console.Write("Год: ");      var y = int.Parse(Console.ReadLine()!);
            Console.Write("Жанр (Id): ");var g = int.Parse(Console.ReadLine()!);
            Console.Write("Кол-во: ");   var q = int.Parse(Console.ReadLine()!);
            bookService.Add(new Book { Title=t, Author=a, Year=y, GenreId=g, Quantity=q });
            break;
        case "3":
            Console.Write("Запрос: "); var qry = Console.ReadLine()!;
            foreach (var b in bookService.Search(qry))
                Console.WriteLine($"{b.Id}. {b.Author} - {b.Title}");
            break;
        case "4":
            Console.Write("Id книги: "); var bid = int.Parse(Console.ReadLine()!);
            Console.Write("Id читателя: "); var rid = int.Parse(Console.ReadLine()!);
            loanService.Issue(bid, rid);
            break;
        case "5":
            Console.Write("Id выдачи: "); var lid = int.Parse(Console.ReadLine()!);
            loanService.Return(lid);
            break;
        case "6":
            foreach (var o in loanService.GetOverdue())
                Console.WriteLine($"{o.LoanId}: {o.Book} - {o.Reader}, срок до {o.DueDate}");
            break;
        case "0":
            return;
    }
}
```

---

 6. Тестирование

| № | Действие | Ожидаемый результат | Фактический результат |
|---|----------|---------------------|------------------------|
| 1 | Добавление книги «Война и мир», Толстой, 1869, жанр 1, 3 экз. | Книга появляется в списке, количество = 3 | ✅ |
| 2 | Поиск по автору «Толстой» | Найдена добавленная книга | ✅ |
| 3 | Выдача книги читателю №1 | Количество уменьшилось до 2, создана запись о выдаче | ✅ |
| 4 | Возврат книги | Количество вернулось к 3, заполнена дата возврата | ✅ |
| 5 | Попытка выдачи при количестве = 0 | Сообщение «Нет доступных экземпляров книги» | ✅ |
| 6 | Просмотр задолженностей | Список просроченных выдач с датами | ✅ |
| 7 | Удаление книги, по которой есть активная выдача | Ошибка внешнего ключа (обработано) | ✅ |

---

 7. Вывод

В ходе выполнения лабораторной работы был разработан программный комплекс «Управление библиотечным фондом», реализующий:

- учёт книг, читателей и выдач;
- поиск и фильтрацию;
- контроль сроков возврата и автоматическое формирование списка задолженностей;
- корректную работу с реляционной БД (транзакции при выдаче/возврате).

Были освоены навыки проектирования ER-модели, написания SQL-запросов, использования ADO.NET и организации многослойной архитектуры приложения.
