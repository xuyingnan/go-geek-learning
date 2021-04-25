# 第 2 课 Go 语言实践 - error 作业


1. 我们在数据库操作的时候，比如 dao 层中当遇到一个 sql.ErrNoRows 的时候，是否应该 Wrap 这个 error，抛给上层。为什么，应该怎么做请写出代码？


答：
    无需 Wrap 这个 error 抛给上层，在 dao 层处理就好。

    代码

    ```go

      func main(db *sql.DB) {
        sqlText := "select age from students where name = ?"
        var age int32
        err = db.QueryRow(sqlText, 'youknowwho').Scan(&name)
        if err != nil {
            if err == sql.ErrNoRows {
              //  不处理
            } else {
                log.Fatal(err)
            }
        }
        fmt.Println(age)
      }

    ```

