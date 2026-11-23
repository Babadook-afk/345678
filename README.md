 Программный комплекс «Управление библиотечным фондом»

Ниже - полностью рабочий проект на Python + Flask + SQLite (легко разворачивается одной командой, БД создаётся автоматически). Комплекс закрывает базовые бизнес‑процессы библиотеки: учёт фонда, читателей, выдачи/возврата, отчётность, контроль просрочек.

 Что умеет система

| Модуль    |   Возможности |
|---|---|
| Авторизация | Вход по логину/паролю (роли: `admin`, `librarian`) |
| Фонд книг | CRUD, поиск по названию/автору/ISBN/инв. номеру, учёт экземпляров |
| Читатели | CRUD, номер читательского билета, контакты |
| Выдача/возврат | Выдача с контролем остатка, автоматический расчёт срока возврата, возврат |
| Просрочки | Автоопределение, фильтры: активные / просроченные / возвращённые |
| Дашборд | Сводная статистика + последние выдачи |

 Структура проекта

```
library_system/
├── app.py
├── requirements.txt
├── templates/
│   ├── base.html
│   ├── login.html
│   ├── index.html
│   ├── books.html
│   ├── book_form.html
│   ├── readers.html
│   ├── reader_form.html
│   ├── loans.html
│   └── loan_form.html
└── static/
    └── style.css
```

---

 📄 `requirements.txt`

```txt
Flask==3.0.3
Flask-SQLAlchemy==3.1.1
Flask-Login==0.6.3
Werkzeug==3.0.3
```

---

 📄 `app.py`

```python
import os
from datetime import datetime, timedelta

from flask import (Flask, render_template, request, redirect,
                   url_for, flash)
from flask_sqlalchemy import SQLAlchemy
from flask_login import (LoginManager, UserMixin, login_user,
                         login_required, logout_user, current_user)
from werkzeug.security import generate_password_hash, check_password_hash
from sqlalchemy import or_

BASE_DIR = os.path.abspath(os.path.dirname(__file__))

app = Flask(__name__)
app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY', 'change-me-in-prod')
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + os.path.join(BASE_DIR, 'library.db')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['LOAN_DAYS'] = 14   срок выдачи по умолчанию

db = SQLAlchemy(app)
login_manager = LoginManager(app)
login_manager.login_view = 'login'
login_manager.login_message = 'Необходимо войти в систему'


 ============ МОДЕЛИ ============
class User(UserMixin, db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, nullable=False)
    password_hash = db.Column(db.String(256), nullable=False)
    role = db.Column(db.String(20), default='librarian')

    def set_password(self, pwd):
        self.password_hash = generate_password_hash(pwd)

    def check_password(self, pwd):
        return check_password_hash(self.password_hash, pwd)


class Book(db.Model):
    __tablename__ = 'books'
    id = db.Column(db.Integer, primary_key=True)
    inventory_number = db.Column(db.String(32), unique=True, nullable=False)
    isbn = db.Column(db.String(20))
    title = db.Column(db.String(256), nullable=False)
    author = db.Column(db.String(256), nullable=False)
    year = db.Column(db.Integer)
    genre = db.Column(db.String(64))
    publisher = db.Column(db.String(128))
    total_copies = db.Column(db.Integer, default=1)
    available_copies = db.Column(db.Integer, default=1)
    shelf = db.Column(db.String(32))

    loans = db.relationship('Loan', backref='book', lazy=True,
                            cascade='all, delete-orphan')

    @property
    def issued(self):
        return self.total_copies - self.available_copies


class Reader(db.Model):
    __tablename__ = 'readers'
    id = db.Column(db.Integer, primary_key=True)
    card_number = db.Column(db.String(32), unique=True, nullable=False)
    full_name = db.Column(db.String(256), nullable=False)
    phone = db.Column(db.String(32))
    email = db.Column(db.String(128))
    address = db.Column(db.String(256))
    registered_at = db.Column(db.DateTime, default=datetime.utcnow)

    loans = db.relationship('Loan', backref='reader', lazy=True,
                            cascade='all, delete-orphan')


class Loan(db.Model):
    __tablename__ = 'loans'
    id = db.Column(db.Integer, primary_key=True)
    book_id = db.Column(db.Integer, db.ForeignKey('books.id'), nullable=False)
    reader_id = db.Column(db.Integer, db.ForeignKey('readers.id'), nullable=False)
    issued_at = db.Column(db.DateTime, default=datetime.utcnow)
    due_at = db.Column(db.DateTime, nullable=False)
    returned_at = db.Column(db.DateTime)
    issued_by = db.Column(db.String(64))

    @property
    def status(self):
        if self.returned_at:
            return 'returned'
        if self.due_at < datetime.utcnow():
            return 'overdue'
        return 'active'


@login_manager.user_loader
def load_user(uid):
    return db.session.get(User, int(uid))


 ============ АВТОРИЗАЦИЯ ============
@app.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('index'))
    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        password = request.form.get('password', '')
        user = User.query.filter_by(username=username).first()
        if user and user.check_password(password):
            login_user(user)
            return redirect(url_for('index'))
        flash('Неверный логин или пароль', 'danger')
    return render_template('login.html')


@app.route('/logout')
@login_required
def logout():
    logout_user()
    return redirect(url_for('login'))


 ============ ДАШБОРД ============
@app.route('/')
@login_required
def index():
    stats = {
        'books': Book.query.count(),
        'copies': db.session.query(
            db.func.coalesce(db.func.sum(Book.total_copies), 0)).scalar(),
        'readers': Reader.query.count(),
        'active_loans': Loan.query.filter(Loan.returned_at.is_(None)).count(),
        'overdue': Loan.query.filter(
            Loan.returned_at.is_(None),
            Loan.due_at < datetime.utcnow()).count(),
    }
    recent = (Loan.query.filter(Loan.returned_at.is_(None))
              .order_by(Loan.issued_at.desc()).limit(5).all())
    return render_template('index.html', stats=stats, recent=recent)


 ============ ФОНД КНИГ ============
@app.route('/books')
@login_required
def books():
    q = request.args.get('q', '').strip()
    query = Book.query
    if q:
        like = f'%{q}%'
        query = query.filter(or_(
            Book.title.ilike(like),
            Book.author.ilike(like),
            Book.isbn.ilike(like),
            Book.inventory_number.ilike(like),
        ))
    items = query.order_by(Book.title).all()
    return render_template('books.html', books=items, q=q)


@app.route('/books/add', methods=['GET', 'POST'])
@login_required
def book_add():
    if request.method == 'POST':
        try:
            book = Book(
                inventory_number=request.form['inventory_number'].strip(),
                isbn=request.form.get('isbn', '').strip(),
                title=request.form['title'].strip(),
                author=request.form['author'].strip(),
                year=int(request.form['year']) if request.form.get('year') else None,
                genre=request.form.get('genre', '').strip(),
                publisher=request.form.get('publisher', '').strip(),
                shelf=request.form.get('shelf', '').strip(),
                total_copies=int(request.form.get('total_copies') or 1),
            )
            book.available_copies = book.total_copies
            db.session.add(book)
            db.session.commit()
            flash('Книга добавлена', 'success')
            return redirect(url_for('books'))
        except Exception as e:
            db.session.rollback()
            flash(f'Ошибка: {e}', 'danger')
    return render_template('book_form.html', book=None)


@app.route('/books/<int:book_id>/edit', methods=['GET', 'POST'])
@login_required
def book_edit(book_id):
    book = db.session.get(Book, book_id)
    if not book:
        flash('Книга не найдена', 'danger')
        return redirect(url_for('books'))
    if request.method == 'POST':
        try:
            issued = book.issued
            new_total = int(request.form.get('total_copies') or 1)
            if new_total < issued:
                flash(f'Нельзя установить количество меньше выданных ({issued})', 'danger')
                return redirect(url_for('book_edit', book_id=book.id))
            book.inventory_number = request.form['inventory_number'].strip()
            book.isbn = request.form.get('isbn', '').strip()
            book.title = request.form['title'].strip()
            book.author = request.form['author'].strip()
            book.year = int(request.form['year']) if request.form.get('year') else None
            book.genre = request.form.get('genre', '').strip()
            book.publisher = request.form.get('publisher', '').strip()
            book.shelf = request.form.get('shelf', '').strip()
            book.total_copies = new_total
            book.available_copies = new_total - issued
            db.session.commit()
            flash('Изменения сохранены', 'success')
            return redirect(url_for('books'))
        except Exception as e:
            db.session.rollback()
            flash(f'Ошибка: {e}', 'danger')
    return render_template('book_form.html', book=book)


@app.route('/books/<int:book_id>/delete', methods=['POST'])
@login_required
def book_delete(book_id):
    book = db.session.get(Book, book_id)
    if book:
        if book.loans:
            flash('Нельзя удалить книгу, по которой есть выдачи', 'danger')
        else:
            db.session.delete(book)
            db.session.commit()
            flash('Книга удалена', 'success')
    return redirect(url_for('books'))


 ============ ЧИТАТЕЛИ ============
@app.route('/readers')
@login_required
def readers():
    q = request.args.get('q', '').strip()
    query = Reader.query
    if q:
        like = f'%{q}%'
        query = query.filter(or_(Reader.full_name.ilike(like),
                                 Reader.card_number.ilike(like),
                                 Reader.phone.ilike(like)))
    items = query.order_by(Reader.full_name).all()
    return render_template('readers.html', readers=items, q=q)


@app.route('/readers/add', methods=['GET', 'POST'])
@login_required
def reader_add():
    if request.method == 'POST':
        try:
            r = Reader(
                card_number=request.form['card_number'].strip(),
                full_name=request.form['full_name'].strip(),
                phone=request.form.get('phone', '').strip(),
                email=request.form.get('email', '').strip(),
                address=request.form.get('address', '').strip(),
            )
            db.session.add(r)
            db.session.commit()
            flash('Читатель добавлен', 'success')
            return redirect(url_for('readers'))
        except Exception as e:
            db.session.rollback()
            flash(f'Ошибка: {e}', 'danger')
    return render_template('reader_form.html', reader=None)


@app.route('/readers/<int:rid>/edit', methods=['GET', 'POST'])
@login_required
def reader_edit(rid):
    r = db.session.get(Reader, rid)
    if not r:
        flash('Читатель не найден', 'danger')
        return redirect(url_for('readers'))
    if request.method == 'POST':
        try:
            r.card_number = request.form['card_number'].strip()
            r.full_name = request.form['full_name'].strip()
            r.phone = request.form.get('phone', '').strip()
            r.email = request.form.get('email', '').strip()
            r.address = request.form.get('address', '').strip()
            db.session.commit()
            flash('Изменения сохранены', 'success')
            return redirect(url_for('readers'))
        except Exception as e:
            db.session.rollback()
            flash(f'Ошибка: {e}', 'danger')
    return render_template('reader_form.html', reader=r)


@app.route('/readers/<int:rid>/delete', methods=['POST'])
@login_required
def reader_delete(rid):
    r = db.session.get(Reader, rid)
    if r:
        if any(l.returned_at is None for l in r.loans):
            flash('Нельзя удалить читателя с активными выдачами', 'danger')
        else:
            db.session.delete(r)
            db.session.commit()
            flash('Читатель удалён', 'success')
    return redirect(url_for('readers'))


 ============ ВЫДАЧА / ВОЗВРАТ ============
@app.route('/loans')
@login_required
def loans():
    status = request.args.get('status', 'active')
    query = Loan.query
    if status == 'active':
        query = query.filter(Loan.returned_at.is_(None))
    elif status == 'overdue':
        query = query.filter(Loan.returned_at.is_(None),
                             Loan.due_at < datetime.utcnow())
    elif status == 'returned':
        query = query.filter(Loan.returned_at.isnot(None))
    items = query.order_by(Loan.issued_at.desc()).all()
    return render_template('loans.html', loans=items, status=status)


@app.route('/loans/issue', methods=['GET', 'POST'])
@login_required
def loan_issue():
    if request.method == 'POST':
        book = db.session.get(Book, int(request.form['book_id']))
        reader = db.session.get(Reader, int(request.form['reader_id']))
        days = int(request.form.get('days') or app.config['LOAN_DAYS'])
        if not book or not reader:
            flash('Выберите книгу и читателя', 'danger')
        elif book.available_copies < 1:
            flash('Нет доступных экземпляров', 'danger')
        else:
            loan = Loan(book_id=book.id, reader_id=reader.id,
                        due_at=datetime.utcnow() + timedelta(days=days),
                        issued_by=current_user.username)
            book.available_copies -= 1
            db.session.add(loan)
            db.session.commit()
            flash('Книга выдана', 'success')
            return redirect(url_for('loans'))
    available_books = Book.query.filter(Book.available_copies > 0)\
                                 .order_by(Book.title).all()
    readers = Reader.query.order_by(Reader.full_name).all()
    return render_template('loan_form.html', books=available_books,
                           readers=readers, default_days=app.config['LOAN_DAYS'])


@app.route('/loans/<int:loan_id>/return', methods=['POST'])
@login_required
def loan_return(loan_id):
    loan = db.session.get(Loan, loan_id)
    if loan and loan.returned_at is None:
        loan.returned_at = datetime.utcnow()
        loan.book.available_copies += 1
        db.session.commit()
        flash('Книга возвращена', 'success')
    return redirect(request.referrer or url_for('loans'))


 ============ ИНИЦИАЛИЗАЦИЯ ============
def init_db():
    with app.app_context():
        db.create_all()
        if not User.query.filter_by(username='admin').first():
            u = User(username='admin', role='admin')
            u.set_password('admin')
            db.session.add(u)
            db.session.commit()
            print('✔ Создан администратор: admin / admin')


if __name__ == '__main__':
    init_db()
    app.run(debug=True)
```

---

 📄 `templates/base.html`

```html
<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}Библиотека{% endblock %}</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
  <link href="{{ url_for('static', filename='style.css') }}" rel="stylesheet">
</head>
<body class="bg-light">
{% if current_user.is_authenticated %}
<nav class="navbar navbar-expand-lg navbar-dark bg-dark mb-4">
  <div class="container">
    <a class="navbar-brand" href="{{ url_for('index') }}">📚 Библиотека</a>
    <div class="navbar-nav">
      <a class="nav-link" href="{{ url_for('index') }}">Дашборд</a>
      <a class="nav-link" href="{{ url_for('books') }}">Фонд</a>
      <a class="nav-link" href="{{ url_for('readers') }}">Читатели</a>
      <a class="nav-link" href="{{ url_for('loans') }}">Выдачи</a>
    </div>
    <div class="ms-auto text-light">
      <span class="me-3">{{ current_user.username }}</span>
      <a class="btn btn-sm btn-outline-light" href="{{ url_for('logout') }}">Выйти</a>
    </div>
  </div>
</nav>
{% endif %}

<div class="container">
  {% with msgs = get_flashed_messages(with_categories=true) %}
    {% for cat, m in msgs %}
      <div class="alert alert-{{ cat }} alert-dismissible fade show">{{ m }}
        <button class="btn-close" data-bs-dismiss="alert"></button>
      </div>
    {% endfor %}
  {% endwith %}
  {% block content %}{% endblock %}
</div>

<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

 📄 `templates/login.html`

```html
{% extends 'base.html' %}
{% block title %}Вход{% endblock %}
{% block content %}
<div class="row justify-content-center">
  <div class="col-md-4">
    <div class="card mt-5 shadow-sm">
      <div class="card-body">
        <h4 class="mb-3">Вход в систему</h4>
        <form method="post">
          <div class="mb-3">
            <label class="form-label">Логин</label>
            <input name="username" class="form-control" required>
          </div>
          <div class="mb-3">
            <label class="form-label">Пароль</label>
            <input type="password" name="password" class="form-control" required>
          </div>
          <button class="btn btn-primary w-100">Войти</button>
        </form>
        <p class="text-muted small mt-3 mb-0">По умолчанию: admin / admin</p>
      </div>
    </div>
  </div>
</div>
{% endblock %}
```

 📄 `templates/index.html`

```html
{% extends 'base.html' %}
{% block title %}Дашборд{% endblock %}
{% block content %}
<h1 class="mb-4">Панель управления</h1>
<div class="row g-3 mb-4">
  <div class="col-md-3"><div class="card text-bg-primary"><div class="card-body">
    <small>Книг в каталоге</small><h2>{{ stats.books }}</h2></div></div></div>
  <div class="col-md-3"><div class="card text-bg-info"><div class="card-body">
    <small>Экземпляров</small><h2>{{ stats.copies }}</h2></div></div></div>
  <div class="col-md-3"><div class="card text-bg-success"><div class="card-body">
    <small>Читателей</small><h2>{{ stats.readers }}</h2></div></div></div>
  <div class="col-md-3"><div class="card text-bg-warning"><div class="card-body">
    <small>Активных выдач</small><h2>{{ stats.active_loans }}</h2></div></div></div>
</div>
{% if stats.overdue %}
<div class="alert alert-danger">
  ⚠ Просроченных выдач: <strong>{{ stats.overdue }}</strong> -
  <a href="{{ url_for('loans', status='overdue') }}">посмотреть</a>
</div>
{% endif %}

<h4>Последние выдачи</h4>
<table class="table table-striped bg-white">
  <thead><tr><th>Книга</th><th>Читатель</th><th>Дата</th><th>Срок</th></tr></thead>
  <tbody>
  {% for l in recent %}
    <tr>
      <td>{{ l.book.title }}</td>
      <td>{{ l.reader.full_name }}</td>
      <td>{{ l.issued_at.strftime('%d.%m.%Y') }}</td>
      <td>{{ l.due_at.strftime('%d.%m.%Y') }}</td>
    </tr>
  {% else %}
    <tr><td colspan="4" class="text-muted">Нет активных выдач</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

 📄 `templates/books.html`

```html
{% extends 'base.html' %}
{% block title %}Фонд{% endblock %}
{% block content %}
<div class="d-flex justify-content-between align-items-center mb-3">
  <h2>Книжный фонд</h2>
  <a class="btn btn-success" href="{{ url_for('book_add') }}">+ Добавить книгу</a>
</div>
<form class="row g-2 mb-3">
  <div class="col-auto"><input name="q" value="{{ q }}" class="form-control" placeholder="Поиск: название, автор, ISBN..."></div>
  <div class="col-auto"><button class="btn btn-outline-secondary">Найти</button></div>
</form>
<table class="table table-hover bg-white">
  <thead><tr>
    <th>Инв.№</th><th>Название</th><th>Автор</th><th>Год</th>
    <th>Всего</th><th>Доступно</th><th></th>
  </tr></thead>
  <tbody>
  {% for b in books %}
    <tr>
      <td>{{ b.inventory_number }}</td>
      <td>{{ b.title }}</td>
      <td>{{ b.author }}</td>
      <td>{{ b.year or '' }}</td>
      <td>{{ b.total_copies }}</td>
      <td><span class="badge {{ 'bg-success' if b.available_copies>0 else 'bg-danger' }}">{{ b.available_copies }}</span></td>
      <td class="text-end">
        <a class="btn btn-sm btn-outline-primary" href="{{ url_for('book_edit', book_id=b.id) }}">✎</a>
        <form method="post" action="{{ url_for('book_delete', book_id=b.id) }}" class="d-inline" onsubmit="return confirm('Удалить книгу?')">
          <button class="btn btn-sm btn-outline-danger">🗑</button>
        </form>
      </td>
    </tr>
  {% else %}
    <tr><td colspan="7" class="text-center text-muted">Ничего не найдено</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

 📄 `templates/book_form.html`

```html
{% extends 'base.html' %}
{% block title %}{{ 'Редактирование' if book else 'Новая книга' }}{% endblock %}
{% block content %}
<h2 class="mb-3">{{ 'Редактировать книгу' if book else 'Добавить книгу' }}</h2>
<form method="post" class="bg-white p-4 rounded shadow-sm">
  <div class="row g-3">
    <div class="col-md-3"><label class="form-label">Инв. № *</label>
      <input name="inventory_number" required class="form-control" value="{{ book.inventory_number if book else '' }}"></div>
    <div class="col-md-3"><label class="form-label">ISBN</label>
      <input name="isbn" class="form-control" value="{{ book.isbn if book else '' }}"></div>
    <div class="col-md-6"><label class="form-label">Название *</label>
      <input name="title" required class="form-control" value="{{ book.title if book else '' }}"></div>
    <div class="col-md-6"><label class="form-label">Автор *</label>
      <input name="author" required class="form-control" value="{{ book.author if book else '' }}"></div>
    <div class="col-md-2"><label class="form-label">Год</label>
      <input name="year" type="number" class="form-control" value="{{ book.year if book and book.year else '' }}"></div>
    <div class="col-md-4"><label class="form-label">Жанр</label>
      <input name="genre" class="form-control" value="{{ book.genre if book else '' }}"></div>
    <div class="col-md-4"><label class="form-label">Издательство</label>
      <input name="publisher" class="form-control" value="{{ book.publisher if book else '' }}"></div>
    <div class="col-md-3"><label class="form-label">Кол-во экземпляров *</label>
      <input name="total_copies" type="number" min="1" class="form-control" value="{{ book.total_copies if book else 1 }}"></div>
    <div class="col-md-3"><label class="form-label">Полка</label>
      <input name="shelf" class="form-control" value="{{ book.shelf if book else '' }}"></div>
  </div>
  <div class="mt-4">
    <button class="btn btn-primary">Сохранить</button>
    <a href="{{ url_for('books') }}" class="btn btn-secondary">Отмена</a>
  </div>
</form>
{% endblock %}
```

 📄 `templates/readers.html`

```html
{% extends 'base.html' %}
{% block title %}Читатели{% endblock %}
{% block content %}
<div class="d-flex justify-content-between align-items-center mb-3">
  <h2>Читатели</h2>
  <a class="btn btn-success" href="{{ url_for('reader_add') }}">+ Добавить читателя</a>
</div>
<form class="row g-2 mb-3">
  <div class="col-auto"><input name="q" value="{{ q }}" class="form-control" placeholder="ФИО, номер билета, телефон"></div>
  <div class="col-auto"><button class="btn btn-outline-secondary">Найти</button></div>
</form>
<table class="table table-hover bg-white">
  <thead><tr><th>Билет</th><th>ФИО</th><th>Телефон</th><th>Email</th><th>Активных выдач</th><th></th></tr></thead>
  <tbody>
  {% for r in readers %}
    <tr>
      <td>{{ r.card_number }}</td>
      <td>{{ r.full_name }}</td>
      <td>{{ r.phone or '' }}</td>
      <td>{{ r.email or '' }}</td>
      <td>{{ r.loans | selectattr('returned_at', 'none') | list | length }}</td>
      <td class="text-end">
        <a class="btn btn-sm btn-outline-primary" href="{{ url_for('reader_edit', rid=r.id) }}">✎</a>
        <form method="post" action="{{ url_for('reader_delete', rid=r.id) }}" class="d-inline" onsubmit="return confirm('Удалить читателя?')">
          <button class="btn btn-sm btn-outline-danger">🗑</button>
        </form>
      </td>
    </tr>
  {% else %}
    <tr><td colspan="6" class="text-center text-muted">Ничего не найдено</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

 📄 `templates/reader_form.html`

```html
{% extends 'base.html' %}
{% block title %}{{ 'Редактирование' if reader else 'Новый читатель' }}{% endblock %}
{% block content %}
<h2 class="mb-3">{{ 'Редактировать читателя' if reader else 'Добавить читателя' }}</h2>
<form method="post" class="bg-white p-4 rounded shadow-sm">
  <div class="row g-3">
    <div class="col-md-4"><label class="form-label">№ билета *</label>
      <input name="card_number" required class="form-control" value="{{ reader.card_number if reader else '' }}"></div>
    <div class="col-md-8"><label class="form-label">ФИО *</label>
      <input name="full_name" required class="form-control" value="{{ reader.full_name if reader else '' }}"></div>
    <div class="col-md-4"><label class="form-label">Телефон</label>
      <input name="phone" class="form-control" value="{{ reader.phone if reader else '' }}"></div>
    <div class="col-md-4"><label class="form-label">Email</label>
      <input name="email" type="email" class="form-control" value="{{ reader.email if reader else '' }}"></div>
    <div class="col-md-4"><label class="form-label">Адрес</label>
      <input name="address" class="form-control" value="{{ reader.address if reader else '' }}"></div>
  </div>
  <div class="mt-4">
    <button class="btn btn-primary">Сохранить</button>
    <a href="{{ url_for('readers') }}" class="btn btn-secondary">Отмена</a>
  </div>
</form>
{% endblock %}
```

 📄 `templates/loans.html`

```html
{% extends 'base.html' %}
{% block title %}Выдачи{% endblock %}
{% block content %}
<div class="d-flex justify-content-between align-items-center mb-3">
  <h2>Выдачи книг</h2>
  <a class="btn btn-success" href="{{ url_for('loan_issue') }}">+ Выдать книгу</a>
</div>

<ul class="nav nav-pills mb-3">
  {% for st, lbl in [('active','Активные'),('overdue','Просроченные'),('returned','Возвращённые'),('all','Все')] %}
    <li class="nav-item">
      <a class="nav-link {{ 'active' if status==st }}" href="{{ url_for('loans', status=st) }}">{{ lbl }}</a>
    </li>
  {% endfor %}
</ul>

<table class="table table-hover bg-white">
  <thead><tr><th>Книга</th><th>Читатель</th><th>Выдана</th><th>Срок</th><th>Статус</th><th></th></tr></thead>
  <tbody>
  {% for l in loans %}
    <tr>
      <td>{{ l.book.title }}</td>
      <td>{{ l.reader.full_name }}</td>
      <td>{{ l.issued_at.strftime('%d.%m.%Y') }}</td>
      <td>{{ l.due_at.strftime('%d.%m.%Y') }}</td>
      <td>
        {% if l.status=='active' %}<span class="badge bg-primary">на руках</span>
        {% elif l.status=='overdue' %}<span class="badge bg-danger">просрочено</span>
        {% else %}<span class="badge bg-secondary">возвращена {{ l.returned_at.strftime('%d.%m.%Y') }}</span>{% endif %}
      </td>
      <td class="text-end">
        {% if not l.returned_at %}
          <form method="post" action="{{ url_for('loan_return', loan_id=l.id) }}" class="d-inline">
            <button class="btn btn-sm btn-outline-success">Вернуть</button>
          </form>
        {% endif %}
      </td>
    </tr>
  {% else %}
    <tr><td colspan="6" class="text-center text-muted">Нет записей</td></tr>
  {% endfor %}
  </tbody>
</table>
{% endblock %}
```

 📄 `templates/loan_form.html`

```html
{% extends 'base.html' %}
{% block title %}Выдача книги{% endblock %}
{% block content %}
<h2 class="mb-3">Выдать книгу</h2>
<form method="post" class="bg-white p-4 rounded shadow-sm" style="max-width:640px">
  <div class="mb-3">
    <label class="form-label">Книга *</label>
    <select name="book_id" class="form-select" required>
      <option value="">- выберите книгу -</option>
      {% for b in books %}
        <option value="{{ b.id }}">{{ b.title }} - {{ b.author }} (доступно: {{ b.available_copies }})</option>
      {% endfor %}
    </select>
  </div>
  <div class="mb-3">
    <label class="form-label">Читатель *</label>
    <select name="reader_id" class="form-select" required>
      <option value="">- выберите читателя -</option>
      {% for r in readers %}
        <option value="{{ r.id }}">{{ r.full_name }} (билет {{ r.card_number }})</option>
      {% endfor %}
    </select>
  </div>
  <div class="mb-3">
    <label class="form-label">Срок выдачи, дней</label>
    <input name="days" type="number" min="1" value="{{ default_days }}" class="form-control">
  </div>
  <button class="btn btn-primary">Выдать</button>
  <a href="{{ url_for('loans') }}" class="btn btn-secondary">Отмена</a>
</form>
{% endblock %}
```

 📄 `static/style.css`

```css
body { font-size: 0.95rem; }
.table td, .table th { vertical-align: middle; }
.card h2 { margin: 0; }
```

---

 🚀 Запуск

```bash
 1. Виртуальное окружение
python -m venv venv
source venv/bin/activate         Linux/macOS
venv\Scripts\activate            Windows

 2. Зависимости
pip install -r requirements.txt

 3. Запуск
python app.py
```

Откройте `http://127.0.0.1:5000`, войдите как admin / admin. База `library.db` создастся автоматически при первом запуске.

---

 Возможные направления развития

- 📦 Экспорт в Excel/PDF отчётов (через `openpyxl`/`reportlab`) - популярное требование в ТЗ по учёту фонда.
- 📧 Email-напоминания о просрочке (SMTP-модуль).
- 📊 Отчёт о движении фонда за период: выдачи/возвраты/списания.
- 🖨 Печать формуляров читателя и актов выдачи.
- 🔐 Разграничение ролей (администратор/библиотекарь/только просмотр).
- 🔍 Импорт книг из ISBN через публичные API (Google Books, OpenLibrary).
- 🐘 Миграция на PostgreSQL - заменить `SQLALCHEMY_DATABASE_URI` на `postgresql://...` и добавить `Flask-Migrate`.
- 🧪 Тесты (pytest + Flask test client).

Если нужен вариант под конкретное ТЗ (например, десктоп на C/WinForms, Django-версия или REST API без фронтенда) - скажите, переделаю под нужный стек.
