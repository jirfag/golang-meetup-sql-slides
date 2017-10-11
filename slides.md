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


SQLX
[https://github.com/jmoiron/sqlx](https://github.com/jmoiron/sqlx)
Хелперы, например, селект в структуру.

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


В *BeepCar* используем beego.orm как QueryBuilder.

```go
qb, _ := orm.NewQueryBuilder(db.SQLDriver)
qb.Select(
  "AVG(profiles.rating / profiles.reviews_count), "+
  "SUM(profiles.reviews_count)").
  From("trip_routes tr").
  InnerJoin("driver_trips dt").
  On("dt.driver_trip_id = tr.driver_trip_id").
  InnerJoin("profiles").
  On("profiles.id = dt.driver_id").
  Where("trip_routes.from_location_id = ?").
  And("trip_routes.to_location_id = ?").
  And("profiles.rating <> 0")
```

---

### Обзор. 3. ORM

TOP ORM:
1. [gorm](https://github.com/jinzhu/gorm) - 6k stars
2. [gorp](https://github.com/go-gorp/gorp) - 2k stars
3. [xorm](https://github.com/go-xorm/xorm) - 2k stars
4. [beego.orm](https://github.com/astaxie/beego/tree/master/orm)

---

### Benchmark
```
80000 times - Read
      raw:    22.54s       281776 ns/op     896 B/op     29 allocs/op
       pg:    37.49s       468683 ns/op     960 B/op     38 allocs/op
     xorm:    84.79s      1059842 ns/op    7812 B/op    241 allocs/op
beego_orm:    86.27s      1078420 ns/op    3081 B/op    108 allocs/op
     gorm:    87.23s      1090387 ns/op    8196 B/op    194 allocs/op
```

Полный тут: [https://github.com/milkpod29/orm-benchmark](https://github.com/milkpod29/orm-benchmark)


### Beego.ORM + Beepcar. Insert
```go
type DriverTrip struct {
	DriverTripId uint64   `orm:"pk;auto"`
	Car          *Car     `orm:"rel(fk);null"`
	Driver       *Profile `orm:"index;rel(fk)"`
	SeatsTotal   NSeats
	Description  string `orm:"type(text)"`
	WomenOnly    bool
	Distance     uint
	TripPoints   []*MyTripPoint `orm:"reverse(many)"`
	PassTrips    []PassTrip     `orm:"-"`
	DeletedAt    *time.Time     `orm:"null;index"`
}
```


Insert by beego.ORM
```go
trip := DriverTrip{
  CarId: 1,
  DriverId: 2,
  SeatsTotal: NSeats(3),
  Description: "lala",
}
_, err := o.Insert(trip)
```


Insert by squirrel
```go
sql, args, err := sq.
    Insert("driver_trips").
    Columns("car_id", "driver_id").
    Values(trip.CarId, trip.DriverId).
    RunWith(db)
```


Insert by Raw SQL
```go
result, err := db.Exec(
	"INSERT INTO driver_trips"+
    "(car_id, driver_id, ...) "+
    "VALUES ($1, $2, ...)",
	trip.CarId, trip.DriverId, ...
)
```

---

### Beego.ORM + BeepCar. Select

```go
var dt DriverTrip
err := o.QueryTable("driver_trips").
  Filter("driver_id__eq", userID).
  Filter("deleted_at__isnull", true)
  Filter("car__deleted_at__isnull", true).
  OrderBy("start_time DESC").
  Limit(1).
  One(&dt)
```

---

### GORM

```go
type User struct {
	gorm.Model
	Rating      int
	RatingMarks int
}

var users []User
err := getGormDB().
  Where("created_at >= ?", getTodayBegin()).
  Limit(limit).
  Find(&users).Error
```

---

### Обзор. 4. ORM с кодогенерацией

|имя|основа|звезд|описание|
|---|---|---|---|
|[go-queryset](https://github.com/jirfag/go-queryset)|struct|200|основано на `gorm`|
|[go-kallax](https://github.com/src-d/go-kallax)|struct|500|только `postgresql`|
|[reform](https://github.com/go-reform/reform)|struct|600||
|[sqlboiler](https://github.com/volatiletech/sqlboiler)|schema|800|нет поддержки `sqlite`|


Пример. Модель
```go
//go:generate goqueryset -in models.go

// gen:qs
type User struct {
	gorm.Model
	Rating      int
	RatingMarks int
}
```

```bash
go generate ./...
```


Сгенерированные методы
```go
func (qs UserQuerySet) RatingGt(rating int) UserQuerySet {
	return qs.w(qs.db.Where("rating > ?", rating))
}
func (qs UserQuerySet) IDEq(ID uint) UserQuerySet {
	return qs.w(qs.db.Where("id = ?", ID))
}
func (qs UserQuerySet) DeletedAtIsNull() UserQuerySet {
	return qs.w(qs.db.Where("deleted_at IS NULL"))
}
func (o *User) Delete(db *gorm.DB) error {
	return db.Delete(o).Error
}
func (qs UserQuerySet) OrderAscByCreatedAt() UserQuerySet {
	return qs.w(qs.db.Order("created_at ASC"))
}
```


Создаем пользователя
```go
u := User{
	Rating: 5,
	RatingMarks: 0,
}
err := u.Create(getGormDB())
```


Селектим юзеров
```go
var users []User
err := NewUserQuerySet(getGormDB()).
	RatingMarksGte(minMarks).
	OrderDescByRating().
	Limit(N)
	All(&users)
```


Апдейтим юзера
```go
u := User{
	Model: gorm.Model{
		ID: uint(7),
	},
	Rating: 1,
}
err := u.Update(getGormDB(), UserDBSchema.Rating)
```

```sql
UPDATE `users`
  SET `rating` = ?
  WHERE `users`.deleted_at IS NULL
    AND `users`.`id` = ?
```


Апдейтим юзеров
```go
err := NewUserQuerySet(getGormDB()).
	RatingLt(1).
	GetUpdater().
	SetRatingMarks(0).
	Update()
```

```sql
UPDATE `users`
  SET `rating_marks` = ?
  WHERE `users`.deleted_at IS NULL
    AND ((rating < ?))
```

---

### На что спотыкались в проде

<img
  width="30%"
  src="https://upload.wikimedia.org/wikipedia/commons/thumb/d/df/StudioWiring.png/1024px-StudioWiring.png">

```go
db, err := sql.Open(db.SQLDriver, connStr)
db.SetMaxOpenConns(openConns)
db.SetMaxIdleConns(idleConns)
```


Типобезопасность
```go
var users []User
db := getGormDB()
db.Find(&users).Error
db.Find(users).Error
db.Find(1).Error
```


<img
  width="50%"
  src="http://memeshappen.com/media/created/SAY-COPY-AND-PASTE-again-Say-That-Again-I-Dare-You-meme-32235.jpg">



```go
func (qs UserQuerySet) WithMaxRating(
  minMarks int) UserQuerySet {

	return qs.RatingMarksGte(minMarks).OrderDescByRating()
}

func (qs UserQuerySet) RegisteredToday() UserQuerySet {
	return qs.CreatedAtGte(getTodayBegin())
}
```


<pre><code class="lang-go hljs">
func GetUsersRegisteredTodayWithMaxRating(
  limit int) ([]User, error) {

	var users []User
	err := NewUserQuerySet(getGormDB()).
		<mark>RegisteredToday().             // reuse our method</mark>
		<mark>WithMaxRating(minRatingMarks). // reuse our method</mark>
		Limit(limit).
		All(&users) // typesafe method All(\*[]User)
	if err != nil {
		return nil, err
	}
	return users, nil
}
</code></pre>


1. Переход mysql -> postgresql
2. Крэши mysql
3. Различия в версиях mysql dev/prod
4. Репликация боя в stage


NULL vs пустое значение

```go
type NullString struct {
        String string
        Valid  bool // Valid is true if String is not NULL
}
```

```go
var s NullString
err := db.QueryRow(
  "SELECT name FROM foo WHERE id=?", id).
  Scan(&s)
if s.Valid {
   // use s.String
} else {
   // NULL value
}
```


Свои типы данных
```go
type Scanner interface {
        // Scan assigns a value from a database driver.
        // The src value will be of one of the
        // following types:
        //    int64
        //    float64
        //    bool
        //    []byte
        //    string
        //    time.Time
        //    nil - for NULL values
        // An error should be returned if the value
        // cannot be stored
        // without loss of information.
        Scan(src interface{}) error
}
```


```go
// Value is for sql driver
func (p P) Value() (driver.Value, error) {

    return p.Marshal()
}

// Scan is for sql driver
func (p *P) Scan(v interface{}) error {
	switch v := v.(type) {
	case []uint8:
		return json.Unmarshal(v, p)
	default:
		return fmt.Errorf("invalid type %T: %v", v, v)
	}
}
```
