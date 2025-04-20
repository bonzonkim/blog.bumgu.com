---
title: "Go와 GORM으로 DB연결해보기"
author: Bumgu
date: 2024-09-18
categories: 
    - Developer
tags: 
    - Go
    - GORM
slug: "go_gorm_db"
---
최근 회사 서비스 어플리케이션 커스텀 Exporter를 만들기 위해 Go를 공부하고 있습니다. 
Go진영 ORM인 `GORM`으로 DB연결후 데이터 삽입까지 간단히 해봤습니다. 

---
### 1. 세팅
#### 1-1. go 세팅
```bash
go mod init // 프로젝트 초기화
go get -u gorm.io/gorm // gorm 패키지 다운
go get -u gorm.io/driver/mysql // mysql 드라이버 다운
```


#### 1-2. Docker로 Mariadb 띄우기
```bash
docker run -d -p 3306:3306 --name maria \
-e MYSQL_ALLOW_EMPTY_PASSWORD=true mariadb:10.9
```




### 2. DB 연결
이제 GORM을 연결해 DB연결을 해보겠습니다.

* database 생성
`create database test;`

```go
package main

import (
	"fmt"
	"log"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)


func main() {
	dsn := "root@tcp(127.0.0.1:3306)/test"

	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})

	if err != nil {
		log.Fatalf("Error opening GORM connection %v", err)
	}

	fmt.Println("Successfully connected GORM : ", db)
}
```


딱히 설명이 필요없는 코드이기 때문에 바로 실행하겠습니다.

![](/images/post/13-gorm/1.png)


이제 mariadb에서 user테이블을 생성하도록 하겠습니다.



```sql
CREATE TABLE user (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    age INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

성공적으로 생성이 됐다면 `show tables;` 로 확인해봅니다.



### 3. 데이터 삽입

#### 3-1. User 타입 정의
먼저 User타입이 필요합니다.
```go
type User struct {
	gorm.Model
	Name  string
	Email string
	Age   int
}
```

gorm의 기본 구조체인 Model을 따릅니다.
Name, Email, Age 필드를 가지고 있습니다.

이제 `User`타입의 테스트 유저를 만들고 그 테스트 유저의 데이터를 삽입하겠습니다.

```go
package main

import (
	"fmt"
	"log"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

type User struct {
	gorm.Model
	Name string
	Email string
	Age int
}


func main() {
	dsn := "root@tcp(127.0.0.1:3306)/test"

	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})

	if err != nil {
		log.Fatalf("Error opening GORM connection %v", err)
	}

	fmt.Println("Successfully connected GORM : ", db)

	testUser := User{Name: "kelly", Email: "kelly@gmail.com", Age: 25}

	result := db.Create(&testUser)

	if result.Error != nil {
		log.Fatalf("Error inserting date to database %v", result.Error)
	}

	fmt.Printf("Test User Created ID: %d", testUser.ID)
 ```
 
`gorm` 으로 생성된 `db`변수의 `Create`를 사용해 데이터를 삽입합니다.
![](/images/post/13-gorm/2.png)

`Create`는 
>  삽입 값을 생성하여 삽입된 데이터의 기본 키를 값의 ID로 반환합니다.

라고 설명하고 있습니다. 즉 삽입하고 나서 그 데이터의 PK를 반환합니다.



#### 3-2 Users 테이블에 삽입실패 - 오류

한번 실행해 보겠습니다.

![](/images/post/13-gorm/3.png)

인서트에 실패했습니다.

오류는 `test.users`가 존재하지 않아서 그렇습니다.

저는 `user`라는 테이블을 생성하였으나, `gorm`의 기본 구조체를 사용했기때문에 `gorm`은 당연하게 `users`라는 테이블에 삽입을 하려고 하고, `users`라는 테이블이 존재하지 않기 때문에 실패했습니다.

```go
func (User) TableName() string {
	return "user"
}
```

`User`를 리시버로 두고 테이블이름을 리턴해주는 `TableName`함수를 생성합니다.



#### 3-3 deleted_at 필드 존재하지 않음 - 오류
다시한번 실행하겠습니다.

![](/images/post/13-gorm/4.png)


```bash
[1.994ms] [rows:0] INSERT INTO `user` (`created_at`,`updated_at`,`deleted_at`,`name`,`email`,`age`) VALUES ('2024-09-18 10:09:14.347','2024-09-18 10:09:14.347',NULL,'kelly','kelly@gmail.com',25) RETURNING `id`
2024/09/18 19:09:14 Error inserting date to database Error 1054 (42S22): Unknown column 'deleted_at' in 'field list'
```

이번엔 `deleted_at`필드가 없다고 에러가 발생했습니다. `user`테이블에 `deleted_at`필드를 생성안했으니 당연하다만  `deleted_at` 필드를 찾는 이유는 위의 `users`테이블을 찾는 이유와 비슷합니다.

`User`타입을 보면,
```go
type User struct {
	gorm.Model
	Name string
	Email string
	Age int
}
```

`gorm.Model`을 사용했는데, 이는 `gorm`의 기본 `users` Model을 사용하겠다는 의미이며 `gorm`의 model에는 `deleted_at`이 기본으로 존재하기 때문에 그렇습니다.


`User`타입을 수정하도록 하겠습니다.


```go
type User struct {
    ID        uint      `gorm:"primaryKey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    Name      string
    Email     string
    Age       int
}
```

DB의 `user`테이블과 컬럼을 맞춰줍니다. 

이부분만 수정하고 실행하겠습니다.


### 4. 결과

![](/images/post/13-gorm/5.png)

성공적으로 생성했다고 로그를 남겼습니다. db에서도 확인해보겠습니다.
![](/images/post/13-gorm/6.png)


성공적으로 데이터가 삽입된것을 확인할 수 있습니다.

---
앞으로 Go를 공부하면서 배우는점을 차근차근 포스팅 하도록 하겠습니다. 감사합니다.
