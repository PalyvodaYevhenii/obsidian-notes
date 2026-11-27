
- **Теоретическая сводка:**
    
    - **Управление внешним ключом:** В реляционных БД связь всегда задается внешним ключом (Foreign Key) на стороне дочерней таблицы. В JPA наиболее эффективна связь `@ManyToOne`, так как она напрямую управляет значением внешнего ключа.
    - **Двунаправленная vs Однонаправленная `@OneToMany`:** Двунаправленная `@OneToMany(mappedBy = ...)` передает управление внешним ключом стороне `@ManyToOne`. Однонаправленная `@OneToMany` без `mappedBy` создает избыточную связующую таблицу (Junction Table) и генерирует лишние DML-операции (`DELETE` + `INSERT`) при удалении элементов из коллекции.
    - **Оптимизация `@OneToOne` и `@ManyToMany`:** Связь `@OneToOne` с совместным первичным ключом (`@MapsId`) исключает вторичные SQL-запросы и экономит память индексов. Классический `@ManyToMany` удаляет и перевставляет все записи связующей таблицы при изменении коллекции; его эффективнее заменять на промежуточную сущность (`PostTag`) с двумя связями `@OneToMany` / `@ManyToOne`.
- **Практическая сводка:**
    
    - **Сравнение DML-запросов `@OneToMany`:** Создать однонаправленный `@OneToMany` и при удалении одного дочернего элемента зафиксировать в логах очистку и повторную вставку записей связующей таблицы. Переписать связь на двунаправленную с `orphanRemoval = true` и убедиться, что генерируется ровно один запрос `DELETE`.
    - **Замена `@ManyToMany`:** Реализовать промежуточную сущность `PostTag` с составным ключом `@EmbeddedId` и аннотацией `@MapsId`, подтвердив одиночное точечное удаление строки из промежуточной таблицы.

---
**```
# 🛠 Практический трекер: Релейшены и связи сущностей в JPA / Hibernate
*(По материалам Главы 11 «Relationships» и Главы 8 книги «High-Performance Java Persistence» Влада Михалчи)*

---

## 📌 Раздел: Релейшены (@ManyToOne, @OneToMany, @OneToOne, @ManyToMany)

### 🔹 Блок A: One-to-Many и Many-to-One
- [ ] **1. Избыточная Join-таблица в Unidirectional `@OneToMany`** #jpa #relationships #oneto-many
  - **Суть:** Создать сущности `Post` и `PostComment`. Настроить однонаправленную связь `@OneToMany List<PostComment> comments` без `@ManyToOne` на дочерней стороне и без `@JoinColumn`. Сохранить пост с двумя комментариями.
  - **Проверка:** По логам SQL зафиксировать создание промежуточной таблицы `post_post_comment` и лишние `INSERT` в junction-таблицу.
- [ ] **2. Лишние `UPDATE` при Unidirectional `@OneToMany` с `@JoinColumn`** #jpa #performance #sql-logging
  - **Суть:** Добавить `@JoinColumn(name = "post_id")` к `@OneToMany List<PostComment> comments`. Сохранить пост с двумя комментариями, затем удалить один комментарий через `comments.remove(0)`.
  - **Проверка:** При вставке увидеть цепочку: сначала `INSERT` комментариев с `NULL post_id`, затем лишние `UPDATE post_comment SET post_id = ?`. При удалении — лишний `UPDATE ... SET post_id = NULL` перед фактическим `DELETE`.
- [ ] **3. Эталонный Bidirectional `@OneToMany` и синхронизация обеих сторон** #jpa #best-practices
  - **Суть:** Реализовать двунаправленную связь: `@ManyToOne @JoinColumn(name = "post_id")` в `PostComment` (владелец FK) и `@OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)` в `Post`. Написать вспомогательные методы `addComment()` и `removeComment()`, синхронизирующие обе стороны.
  - **Проверка:** Убедиться по логам SQL, что при вставке и удалении дочерних записей выполняются только чистые прямые `INSERT` и `DELETE` без единого лишнего `UPDATE` или join-таблицы.
- [ ] **4. Проблема равенства сущностей (equals/hashCode) при удалении из коллекции** #hibernate #equality
  - **Суть:** Загрузить `Post` с комментариями в одном запросе (транзакции), закрыть контекст (сущности detached). В новой транзакции попытаться удалить комментарий `post.removeComment(detachedComment)` по бизнес-ключу (`createdBy` + `createdOn`).
  - **Проверка:** Проверить, что без переопределения `equals`/`hashCode` удаление не срабатывает (используется identity equality `==`), а реализация равенства по уникальному бизнес-ключу позволяет корректно отработать `orphanRemoval`.

---

### 🔹 Блок B: Особенности и подводные камни @OneToOne
- [ ] **5. Оптимизация `@OneToOne` через Derived Identifiers (`@MapsId`)** #jpa #oneto-one #maps-id
  - **Суть:** Связать `Post` и `PostDetails`. Вместо отдельного первичного ключа и внешнего ключа в `post_details` настроить разделяемый первичный ключ через `@OneToOne @MapsId private Post post;`.
  - **Проверка:** В схеме БД таблица `post_details` имеет первичный ключ, являющийся одновременно foreign key на `post.id` (экономия индекса). Загрузка `PostDetails` по ID поста через `find()` не требует отдельного JPQL/SQL Join и напрямую использует L2-кеш.
- [ ] **6. Ловушка `FetchType.LAZY` на стороне Parent в Bidirectional `@OneToOne`** #hibernate #n-plus-one #lazy-loading
  - **Суть:** Создать двунаправленный `@OneToOne(mappedBy = "post", fetch = FetchType.LAZY) private PostDetails details;`. Выполнить `entityManager.find(Post.class, 1L)`.
  - **Проверка:** Увидеть по логам SQL, что Hibernate **игнорирует LAZY** и сразу выполняет второй `SELECT` в `post_details` (Hibernate не может поместить Proxy без проверки существования записи в БД, так как поле может быть `null`).

---

### 🔹 Блок C: Many-to-Many и Bag Semantics
- [ ] **7. Катастрофа удаления в Unidirectional `@ManyToMany` со списком `List`** #jpa #manyto-many #performance
  - **Суть:** Создать отношение `@ManyToMany` между `Post` и `Tag` через `List<Tag> tags`. Добавить к посту 3 тега. Затем вызвать `post.getTags().remove(0)`.
  - **Проверка:** Убедиться по логам SQL, что Hibernate из-за семантики Bag выполняет `DELETE FROM post_tag WHERE post_id = ?` (удаляет **все** связи поста) и заново вставляет оставшиеся теги через `INSERT`.
- [ ] **8. Разрешение Many-to-Many через промежуточную сущность (`PostTag`)** #jpa #architecture #composite-key
  - **Суть:** Заменить классический `@ManyToMany` на промежуточную сущность `PostTag` с составным ключом (`@EmbeddedId PostTagId` или `@IdClass`) и двумя связями `@ManyToOne @MapsId`.
  - **Проверка:** При удалении одного тега Hibernate генерирует ровно один точечный `DELETE FROM post_tag WHERE post_id = ? AND tag_id = ?`. Появляется возможность добавить в таблицу связи полезные бизнес-колонки (`created_on`, `created_by`).

---

### 🔹 Блок D: Каскады, Типы коллекций и Выборка
- [ ] **9. Поведение удаления в `@ElementCollection`: List против `@OrderColumn`** #hibernate #element-collection
  - **Суть:** Создать сущность `Post` с `@ElementCollection List<String> comments`. Сохранить 3 комментария, затем удалить первый элемент `comments.remove(0)`. Затем повторить эксперимент, добавив аннотацию `@OrderColumn(name = "order_id")`.
  - **Проверка:** Без `@OrderColumn` Hibernate удаляет все строки и вставляет оставшиеся. С `@OrderColumn` при удалении с хвоста выполняется одиночный `DELETE`, а из середины/головы — `DELETE` и точечный `UPDATE` индексов.
- [ ] **10. Искоренение `EAGER` и предотвращение N+1 при выборке связей** #hibernate #fetch-type #n-plus-one
  - **Суть:** Оставить для `@ManyToOne` дефолтный `FetchType.EAGER` и загрузить 10 комментариев через `findAll()` (получить 10 дополнительных `SELECT` постов — N+1). Переключить связь на `FetchType.LAZY` и написать запрос с динамическим `JOIN FETCH` (или `@EntityGraph`).
  - **Проверка:** С `FetchType.LAZY` + `JOIN FETCH` формируется ровно 1 SQL-запрос с `INNER JOIN`, полностью устраняющий проблему N+1.
```

---

💡 Хотите разобрать эталонный шаблон связывания сущностей для задачи 8 (промежуточная сущность `PostTag` с `@MapsId` и `equals/hashCode`), который Влад Михалча рекомендует как лучший паттерн для Many-to-Many?**