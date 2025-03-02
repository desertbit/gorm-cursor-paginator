**Forked from https://github.com/pilagod/gorm-cursor-paginator**

# gorm-cursor-paginator [![Build Status](https://travis-ci.org/pilagod/gorm-cursor-paginator.svg?branch=master)](https://travis-ci.org/pilagod/gorm-cursor-paginator) [![Coverage Status](https://coveralls.io/repos/github/pilagod/gorm-cursor-paginator/badge.svg?branch=master&kill_cache=1)](https://coveralls.io/github/pilagod/gorm-cursor-paginator?branch=master) [![Go Report Card](https://goreportcard.com/badge/github.com/desertbit/gorm-cursor-paginator)](https://goreportcard.com/report/github.com/desertbit/gorm-cursor-paginator)

A paginator doing cursor-based pagination based on [GORM](https://github.com/go-gorm/gorm)

> This doc is for v2, which uses [GORM v2](https://github.com/go-gorm/gorm). If you are using [GORM v1](https://github.com/jinzhu/gorm), please checkout [v1 doc](https://github.com/desertbit/gorm-cursor-paginator/tree/v1).

## Features

- Query extendable.
- Multiple paging keys.
- Pagination across custom types (e.g. JSON)
- Paging rule customization for each key.
- GORM `column` tag supported.
- Error handling enhancement.
- Exporting `cursor` module for advanced usage.

## Installation

```sh
go get -u github.com/desertbit/gorm-cursor-paginator/v2
```

## Usage By Example

```go
import (
   "github.com/desertbit/gorm-cursor-paginator/v2/paginator"
)
```

Given an `User` model for example:

```go
type User struct {
    ID          int
    JoinedAt    time.Time `gorm:"column:created_at"`
}
```

We first need to create a `paginator.Paginator` for `User`, here are some useful patterns:

1. Configure by `paginator.Option`, those functions with `With` prefix are factories for `paginator.Option`:

    ```go
    func CreateUserPaginator(
        cursor paginator.Cursor,
        order *paginator.Order,
        limit *int,
    ) *paginator.Paginator {
        opts := []paginator.Option{
            &paginator.Config{
                Keys: []string{"ID", "JoinedAt"},
                Limit: 10,
                Order: paginator.ASC,
            },
        }
        if limit != nil {
            opts = append(opts, paginator.WithLimit(*limit))
        }
        if order != nil {
            opts = append(opts, paginator.WithOrder(*order))
        }
        if cursor.After != nil {
            opts = append(opts, paginator.WithAfter(*cursor.After))
        }
        if cursor.Before != nil {
            opts = append(opts, paginator.WithBefore(*cursor.Before))
        }
        return paginator.New(opts...)
    }
    ```

2. Configure by setters on `paginator.Paginator`:

    ```go
    func CreateUserPaginator(
        cursor paginator.Cursor,
        order *paginator.Order,
        limit *int,
    ) *paginator.Paginator {
        p := paginator.New(
            &paginator.Config{
                Keys: []string{"ID", "JoinedAt"},
                Limit: 10,
                Order: paginator.ASC,
            },
        )
        if order != nil {
            p.SetOrder(*order)
        }
        if limit != nil {
            p.SetLimit(*limit)
        }
        if cursor.After != nil {
            p.SetAfterCursor(*cursor.After)
        }
        if cursor.Before != nil {
            p.SetBeforeCursor(*cursor.Before)
        }
        return p
    }
    ```

3. Configure by `paginator.Rule` for fine grained setting for each key:

    > Please refer to [Specification](#specification) for details of `paginator.Rule`.

    ```go
    func CreateUserPaginator(/* ... */) {
        p := paginator.New(
            &paginator.Config{
                Rules: []paginator.Rule{
                    {
                        Key: "ID",
                    },
                    {
                        Key: "JoinedAt",
                        Order: paginator.DESC,
                        SQLRepr: "users.created_at",
                        NULLReplacement: "1970-01-01",
                    },
                },
                Limit: 10,
                // Order here will apply to keys without order specified.
                // In this example paginator will order by "ID" ASC, "JoinedAt" DESC.
                Order: paginator.ASC, 
            },
        )
        // ...
        return p
    }
    ```

After knowing how to setup the paginator, we can start paginating `User` with GORM:

```go
func FindUsers(db *gorm.DB, query Query) ([]User, paginator.Cursor, error) {
    var users []User

    // extend query before paginating
    stmt := db.
        Select(/* fields */).
        Joins(/* joins */).
        Where(/* queries */)

    // create paginator for User model
    p := CreateUserPaginator(/* config */)

    // find users with pagination
    result, cursor, err := p.Paginate(stmt, &users)

    // this is paginator error, e.g., invalid cursor
    if err != nil {
        return nil, paginator.Cursor{}, err
    }

    // this is gorm error
    if result.Error != nil {
        return nil, paginator.Cursor{}, result.Error
    }

    return users, cursor, nil
}
```

The second value returned from `paginator.Paginator.Paginate` is a `paginator.Cursor` struct, which is same as `cursor.Cursor` struct:

```go
type Cursor struct {
    After  *string `json:"after" query:"after"`
    Before *string `json:"before" query:"before"`
}
```

That's all! Enjoy paginating in the GORM world. :tada:

> For more paginating examples, please checkout [exmaple/main.go](https://github.com/desertbit/gorm-cursor-paginator/blob/master/example/main.go) and [paginator/paginator_paginate_test.go](https://github.com/desertbit/gorm-cursor-paginator/blob/master/paginator/paginator_paginate_test.go)
>
> For manually encoding/decoding cursor exmaples, please checkout [cursor/encoding_test.go](https://github.com/desertbit/gorm-cursor-paginator/blob/master/cursor/encoding_test.go)

## Specification

### paginator.Paginator

Default options used by paginator when not specified:

- `Keys`: `[]string{"ID"}`
- `Limit`: `10`
- `Order`: `paginator.DESC`

### paginator.Rule

- `Key`: Field name in target model struct.
- `Order`: Order for this key only.
- `SQLRepr`: SQL representation used in raw SQL query.<br/>
    > This is especially useful when you have `JOIN` or table alias in your SQL query. If `SQLRepr` is not specified, paginator will get table name from model, plus table key derived by below rules to form the SQL query:
    > 1. Find GORM tag `column` on struct field.
    > 2. If tag not found, convert struct field name to snake case.
- `SQLType`: SQL type used for type casting in the raw SQL query.<br/>
    > This is especially useful when working with custom types (e.g. JSON).
- `NULLReplacement`(v2.2.0): Replacement for NULL value when paginating by nullable column.<br/>
    > If you paginate by nullable column, you will encounter [NULLS { FIRST | LAST } problems](https://learnsql.com/blog/how-to-order-rows-with-nulls/). This option let you decide how to order rows with NULL value. For instance, we can set this value to `1970-01-01` for a nullable `date` column, to ensure rows with NULL date will be placed at head when order is ASC, or at tail when order is DESC.
- `CustomType`: Extra information needed only when paginating across custom types (e.g. JSON). To support custom type pagination, the type needs to implement the `CustomType` interface:
  ```go
  type CustomType interface {
      // GetCustomTypeValue returns the value corresponding to the meta attribute inside the custom type.
      GetCustomTypeValue(meta interface{}) (interface{}, error)
  }
  ```
  and provide the following information:
  - `Meta`: meta attribute inside the custom type. The paginator will pass this meta attribute to the `GetCustomTypeValue` function, which should return the actual value corresponding to the meta attribute. For JSON, meta would contain the JSON key of the element inside JSON to be used for pagination.
  - `Type`: GoLang type of the meta attribute. 

  Also, when paginating across custom types, it is expected that the `SQLRepr` & `SQLType` are set.  `SQLRepr` should contain the SQL query to get the meta attribute value, while `SQLType` should be used for type casting if needed.

## Changelog

### v2.3.0

- Add `CustomType` to `paginator.Rule` to support [custom data types](https://gorm.io/docs/data_types.html), credit to [@nikicc](https://github.com/nikicc).
> There are some adjustments to the signatures of `cursor.NewEncoder` and `cursor.NewDecoder`. Be careful when upgrading if you use them directly.

### v2.2.0

- Add `NULLReplacement` to `paginator.Rule` to overcome [NULLS { FIRST | LAST } problems](https://learnsql.com/blog/how-to-order-rows-with-nulls/), credit to [@nikicc](https://github.com/nikicc).

### v2.1.0

- Let client control context, suggestion from [@khalilsarwari](https://github.com/khalilsarwari).

### v2.0.1

- Fix order flip bug when paginating backward, credit to [@sylviamoss](https://github.com/sylviamoss).

## License

© Cyan Ho (pilagod), 2018-NOW

Released under the [MIT License](https://github.com/desertbit/gorm-cursor-paginator/blob/master/LICENSE)
