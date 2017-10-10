# Работа с SQL в golang
Денис Исаев, BeepCar

---

### Обзор. 1. Классическая работа с SQL
Открываем коннект к базе
```go
db, err := sql.Open(driver, dataSourceName)
if err != nil {
  log.Fatalf(err)
}

if err := db.Ping(); err != nil {
  log.Fatal(err)
}
```


Выполняем запрос без возврата значений
```go
result, err := db.Exec(
	"INSERT INTO users (name, age) VALUES ($1, $2)",
	"gopher",
	27,
)
```


Селектим данные
```go
rows, err := db.Query("SELECT name FROM users "+
  "WHERE age = $1", age)
if err != nil {}

for rows.Next() {
	var name string
	if err := rows.Scan(&name); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("%s is %d\n", name, age)
}

if err := rows.Err(); err != nil {}
```

---

### Обзор. 2. SQL Query Builders
```go
import sq "github.com/Masterminds/squirrel"

users := sq.Select("*").
  From("users").
  Join("emails USING (email_id)")
active := users.Where(sq.Eq{"deleted_at": nil})
sql, args, err := active.ToSql()
```

```sql
SELECT *
  FROM users
  JOIN emails USING (email_id)
  WHERE deleted_at IS NULL
```


Может помогать в выполнении запросов
```go
stooges := users.Where(sq.Eq{"username": []string{
  "moe", "larry", "curly", "shemp"}})
three_stooges := stooges.Limit(3)
```

<pre><code class="lang-go hljs"><mark>rows, err := three_stooges.RunWith(db).Query()</mark></code></pre>

```go
// Behaves like:
rows, err := db.Query("SELECT * FROM users WHERE "+
  "username IN (?,?,?,?) LIMIT 3",
  "moe", "larry", "curly", "shemp")
```


## Самый популярный QB - [squirrel](https://github.com/Masterminds/squirrel).

### Список QB + бенчмарки: [https://github.com/elgris/golang-sql-builder-benchmark](https://github.com/elgris/golang-sql-builder-benchmark)

---

### Обзор. 3. ORM

sfasdf

Note:
Тут заметка для меня
