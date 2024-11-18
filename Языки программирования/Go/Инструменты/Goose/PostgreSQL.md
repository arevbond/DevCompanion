### Создать файл миграции
`goose -dir=[path] create [name] sql`

### Выполнить миграции
`goose -dir=[path] postgres [DSN] up`
[[DSN|PostgreSQL DSN]]