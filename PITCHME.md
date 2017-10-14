# Кодогенерация в Go

Илья Сауленко, Avito

---

## Введение

Работаю в юните Architecture в Avito.

Делаю платформенные штуки.

Пишу на Go, PHP, Python и YAML 😭

---

# Теория

+++

## Что такое кодогенерация?

Это когда код за тебя пишет компьютер.

Метапрограммирование, шаблоны кода.

- CMake
- C++ Templates
- LLVM
- Yacc

+++

## Зачем нужна кодогенерация?

- Уменьшение рутины/копипасты
- Compile-time safety
- Скорость выполнения
- Ограничения языка

---

# Проблема

+++

## Проблема

- Есть сервис хранения метаданных о картинках
- Метаданные описываются в Go-структурах. У структур много вложенных структур
- Все вложенные структуры - опциональны

+++

## Пример документа

> GET /images/42

```json
{
    "exif": {
        "camera": {
            "make": "Nikon",
            "model": "D5"
        },
        "gps": {
            "coordinates": [55, 80]
        },
        "created": "2017-10-14T16:42:00"
    },
    //...
}
```

+++

## Клиенты приходят и обновляют часть данных

> PUT /images/42

```json
{
    "exif": {
        "gps": {
            "country": "Russia",
            "city": "Moscow"
        }
    }
}
```

+++

## Желаемый результат

> GET /images/42

```json
{
    "exif": {
        "camera": {
            "make": "Nikon",
            "model": "D5"
        },
        "gps": {
            "coordinates": [55, 80],
            "country": "Russia",
            "city": "Moscow"
        },
        "created": "2017-10-14T16:42:00"
    },
    //...
}
```

+++

## Сервис должен корректно мержить все эти данные

```go
type Exif struct {
	Gps     *Gps       `json="gps,omitempty"`
	Camera  *Camera    `json="camera,omitempty"`
	Created *time.Time `json="created,omitempty"`
}

type Camera struct {
	Make  *string `json="make,omitempty"`
	Model *string `json="model,omitempty"`
}

type Gps struct {
	Coordinates *Coordinates `json="coordinates,omitempty"`
	Country     *string      `json="country,omitempty"`
	City        *string      `json="gps,omitempty"`
}

type Coordinates *[2]float32
```

---

# Решение #0

+++

## Решение #0

Взять существующую библиотеку:

- [github.com/imdario/mergo](https://github.com/imdario/mergo)
- [github.com/divideandconquer/go-merge](https://github.com/divideandconquer/go-merge)

Проблема: никто не любит мержить указатели на структуры.

+++

## Решение #0

Указатели на структуры нельзя просто так мержить из-за приватных полей:

- `time.Time`
- Реализации `hash.Hash`

---

# Решение #1

+++

## Решение #1

Реализуем на структурках метод `MergeWith()`:

```go
func (t *Gps) MergeWith(s *Gps) {
	if s == nil {
		return
	}

	if t == nil {
		*t = *s
		return
	}

	if s.Coordinates != nil {
		t.Coordinates = s.Coordinates
	}

	if s.Country != nil {
		t.Country = s.Country
	}
	
	if s.City != nil {
        t.City = s.City
    }
}
```

+++

## Решение #1

Реализуем на структурках метод `MergeWith()`:

```go
func (t *Exif) MergeWith(s *Exif) {
	if s == nil {
		return
	}

	if t == nil {
		*t = *s
		return
	}

	if t.Created != nil {
		s.Created = t.Created
	}

	t.Gps.MergeWith(s.Gps)
	t.Camera.MergeWith(s.Camera)
}
```

+++

## Решение #1

#### Плюсы:

- Compile-time safety
- Быстро работает

#### Минусы:

- Много кода
- При расширении структур надо не забыть добавить код в `MergeWith()`

---

# Решение #2

+++

## Решение #2

Возьмём `reflect` и напишем общую реализацию метода `Merge()`:

```go
func Merge(dst, src interface{}) {
	// resolve pointer for dst:
    vDst := reflect.ValueOf(dst).Elem()
    vSrc := reflect.ValueOf(src)

    merge(vDst, vSrc)
}
```

+++

```go
func merge(dst, src reflect.Value) error {
	// error handling skipped
	switch dst.Kind() {
    case reflect.Struct:
        for i, n := 0, dst.NumField(); i < n; i++ {
            if err := merge(dst.Field(i), src.Field(i)); err != nil {
                return err
            }
        }
    case reflect.Ptr:
        // ...
    case reflect.Interface:
        // ...
    default:
        if dst.CanSet() {
            dst.Set(src)
        }
    }

    return nil
}
```

+++

## Решение #2

Решение проблемы с `time.Time`:

У `reflect.StructField` есть поле `Tag` типа `reflect.StructTag`.

```go
type Exif struct {
	Gps     *Gps       `json="gps,omitempty"`
	Camera  *Camera    `json="camera,omitempty"`
	Created *time.Time `json="created,omitempty",merge="replace"`
}
```

+++

## Решение #2

#### Плюсы:

- Один раз написал — работает

#### Минусы:

- Метапрограммирование, но в рантайме
- nil pointer dereferencing

+++

## Решение #2

Пример из `text/template`:

Работает:

```
{{ .non_existent_key.foo }}
// <no value>
```

Паникует:

```
{{ (.non_existent_key).foo }}
// panic: reflect: Zero(nil) [recovered]
```

[golang/go issue#21171](https://github.com/golang/go/issues/21171)

---

# Решение #3

+++

## Решение #3

Генерировать код метода `MergeWith()` автоматически из Go-структур.

Парсить код в AST с помощью `go/parser` и `go/ast`, генерировать код как в решении #1.

+++

## Решение #3

Что такое AST?

[goast.yuroyoro.net](http://goast.yuroyoro.net/)

![goast](./assets/md/assets/goast.png)

+++

## Решение #3

`go/parser`

Метод `parser.ParseDir()` возвращает `map[string]*ast.Package`.

`ast.Package`

```go
type Package struct {
        Name    string             // package name
        Scope   *Scope             // package scope across all files
        Imports map[string]*Object // map of package id -> package object
        Files   map[string]*File   // Go source files by filename
}
```

+++

## Решение #3

`ast.File`

```go
type File struct {
        Doc        *CommentGroup   // associated documentation; or nil
        Package    token.Pos       // position of "package" keyword
        Name       *Ident          // package name
        Decls      []Decl          // top-level declarations; or nil
        Scope      *Scope          // package scope (this file only)
        Imports    []*ImportSpec   // imports in this file
        Unresolved []*Ident        // unresolved identifiers in this file
        Comments   []*CommentGroup // list of all comments in the source file
}
```

+++

## Решение #3

`ast.Decl` ~ `ast.Node`

- ast.GenDecl
- ast.FuncDecl
- ast.BlockStmt
- ast.ExprStmt
- ast.GoStmt
- etc

`ast.PackageExports()` оставляет только экспортированные типы/методы.

+++

## Решение #3

`ast.Walk()` / `ast.Inspect()` для итерации по дереву:

```go
func Walk(v Visitor, node Node)
```

```go
type Visitor interface {
	Visit(node Node) (w Visitor)
}
```

```go
func Inspect(node Node, f func(Node) bool)
```

+++

## Решение #3

```go
type StructsVisitor struct{}

func (s *StructsVisitor) Visit(node ast.Node) (w ast.Visitor) {
	switch t := node.(type) {
	case *ast.TypeSpec:
		if st, ok := t.Type.(*ast.StructType); ok {
			if st.Fields == nil {
				return s
			}

			for _, field := range st.Fields.List {
				typ, ptr := getType(field)
				tag := getTag(field)

				for _, name := range field.Names {
					fmt.Printf("name: %s, type: %s, isPointer: %#v, tag: %s", name.Name, typ, ptr, tag)
				}
			}
		}
	}

	return s
}
```

+++

## Решение #3

```go
func getType(field *ast.Field) (string, bool) {
	switch s := field.Type.(type) {
	case *ast.StarExpr:
		if v, ok := s.X.(*ast.Ident); ok {
			return v.Name, true
		}
	case *ast.Ident:
		return s.Name, false
	}

	return "", false
}
```

+++

## Решение #3

```go
func getTag(field *ast.Field) string {
	if field.Tag == nil {
		return ""
	}

	return field.Tag.Value
}
```

+++

## Решение #3

При желании можно обойтись без `Walk()` или использовать шорткат `Scope.Objects`:

```go
for _, obj := range pkg.Scope.Objects {
    if t, ok := obj.Decl.(ast.StructType); ok {
        for _, field := range t.Fields.List {
            typ, ptr := getType(field)
            tag := getTag(field)

            for _, name := range field.Names {
                fmt.Printf("name: %s, type: %s, isPointer: %#v, tag: %s", name.Name, typ, ptr, tag)
            }
        }
    }
}
```

+++

## Решение #3

Собственно, генерация самого кода:

`go/ast` vs шаблонизация строк

- Генерировать AST — надёжней и предсказуемей
- Строки шаблонизировать проще

+++

## Решение #3

`go/format` — библиотека, предоставляющаяя функционал `go fmt`.

Форматирует ноду (`ast.File`, `ast.Decl` etc) в `io.Writer`:
```go
func Node(dst io.Writer, fset *token.FileSet, node interface{}) error
```

Форматирует слайс байтов в слайс байтов:
```go
func Source(src []byte) ([]byte, error)
```

+++

## Решение #3

Ок, код написали, но как его запускать?

```bash
go format ./...
```

`go format` просто делает exec:

```go
package main

import "fmt"

//go:generate echo "123"

func main() {
	fmt.Printf("git version: %s\n", GitVersion)
}
```

+++

## Решение #3

Пример для генерации `version.go` вместо ld-флагов:

```
//go:generate version_from_git.sh
```

и version_from_git.sh:

```bash
##!/bin/bash

echo "package main" > version.go
echo "const GitVersion = '"$(git rev-parse HEAD)"'" >> version.go
```

+++

## Решение #3

#### Плюсы:

- Один раз написал — работает
- Compile-time safety
- Скорость выполнения
- Можно настраивать поведение

#### Минусы:

- Много кода для работы с AST

---

# Решение #4

+++

## Решение #4

Генерировать структуры и методы по JSON Schema.

---

## Сравнительная табличка

| Решение                       | Компиляция  | Скорость  | Сложность написания | Сложность поддержки |
|-------------------------------|-------------|-----------|---------------------|---------------------|
| самописный MergeWith()        | ✔️          | ✔️        | 😀                   | 😭                   |
| reflect                       | ❌          | ❌        | 😨                   | 😀                   |
| кодогенерация на go/ast       | ✔️          | ✔️        | 😭                   | 😀                   |
| кодогенерация на JSON Schema  | ✔️          | ✔️        | 😨                   | 😀                   |

<small>Сложность по шкале от 😀 до 😭</small>

---

## Итоги

Юзаем решение #3 с генерацией кода по AST и текстовой шаблонизацией.

Через теги структур можно настраивать поведение.

Планируем выложить утилиту в опенсорс.

---

# Бонус: решение #5

+++

## Бонус: решение #5

Генерировать код на текстовых шаблонах.

Можно писать на других языках.

Хорошо подходит для дженериков:

- List
- Queue

```
//go:generate generate-generic -name Exif -type Queue -output exif_list.go
```

+++

## Бонус: решение #5

```go
package {{.Package}}

type {{.MyType}}Queue struct {
    q []{{.MyType}}
}

func New{{.MyType}}Queue() *{{.MyType}}Queue {
    return &{{.MyType}}Queue{
        q: []{{.MyType}}{},
    }
}

func (o *{{.MyType}}Queue) Insert(v {{.MyType}}) {
    o.q = append(o.q, v)
}

func (o *{{.MyType}}Queue) Remove() {{.MyType}} {
    if len(o.q) == 0 {
        panic("Oops.")
    }
    rst := o.q[0]
    o.q = o.q[1:]
    return  rst
}
```

---

## Бонус: миграции кода

Миграции на уровне AST как в jscodeshift.

Было:
```go
func foo(a, b, c string)
```

Стало:
```go
func foo(a, b, c, d string)
```

---

## Бонус: библиотеки

- [github.com/joeshaw/gengen](https://github.com/joeshaw/gengen)
- [github.com/go-goast/goast](https://github.com/go-goast/goast)
- [github.com/fatih/astrewrite](https://github.com/fatih/astrewrite)

---

# Спасибо!

## Вопросы?

Илья Сауленко, Avito

- [hi@mynameiswhm.ru](mailto:hi@mynameiswhm.ru)
- [@mynameiswhm](https://twitter.com/mynameiswhm)
