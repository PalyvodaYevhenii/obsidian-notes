[[Pro Spring 5, 5th Edition 1.pdf]]
[[Pro Spring Boot 1.pdf]]

Полный структурированный гайд для подготовки к собеседованию на позицию **Senior Java / Spring Developer**. Каждая глава содержит ключевые темы, вопросы с Senior-собеседований, практическое ТЗ и **экспресс-тест из 3 концептуальных вопросов с разбором правильных ответов**.

---

# 🔴 ПРИОРИТЕТ 1: Ядро Spring (IoC, DI, Lifecycle) и Auto-Configuration

---

## 1. Pro Spring 5 — Глава 3: _Introducing IoC and DI in Spring_

### 📌 Ключевые темы

- Внедрение зависимостей (Constructor vs Setter vs Field injection).
- Области видимости бинов (Bean Scopes: Singleton, Prototype, Request, Session, Thread).
- Внедрение Prototype в Singleton (Method Injection, `@Lookup`, `ObjectProvider`).
- Связывание (Autowiring: `byName`, `byType`, `constructor`), разрешение неоднозначностей (`@Primary`, `@Qualifier`).

### ❓ Что спросят на Senior-собеседовании

1. Почему Field Injection считается антипаттерном и как Constructor Injection обеспечивает неизменяемость (immutability) и облегчает модульное тестирование?
2. Что произойдёт, если внедрить `@Scope("prototype")` бин напрямую через `@Autowired` в `@Scope("singleton")` сервис? Какие 3 способа решения этой проблемы есть в Spring?
3. Как контейнер Spring разрешает циклические зависимости (Circular Dependencies) и почему в Spring Boot 2.6+ они запрещены по умолчанию?

### 💻 Практическое задание (ТЗ)

Создайте singleton-сервис `ReportService` и prototype-бин `ReportGenerator`. Продемонстрируйте проблему зафиксированного экземпляра prototype-бина при обычном `@Autowired`. Исправьте это двумя способами: через `ObjectProvider<ReportGenerator>` и через `@Lookup`. Дополнительно реализуйте кастомный `BeanPostProcessor`, проверяющий время инициализации бинов.

### 🧪 Практический тест

**Вопрос 1.** Какой результат выведет метод `check()` при двух последовательных вызовах, если `PrototypeBean` внедрён в `SingletonBean` через `@Autowired` без дополнительных настроек?

```
@Component @Scope("prototype")
public class PrototypeBean { private final UUID id = UUID.randomUUID(); public UUID getId() { return id; } }

@Component
public class SingletonBean {
    @Autowired private PrototypeBean prototypeBean;
    public void check() { System.out.println(prototypeBean.getId()); }
}
```

- A) Каждый вызов выведет новый случайный `UUID`.
- B) Оба вызова выведут один и тот же `UUID`.
- C) Выбросится `BeanCreationException`.
- **Правильный ответ:** **B**. Singleton-бин создаётся один раз при старте контекста. Зависимость `PrototypeBean` внедряется в него единовременно в момент конструирования, поэтому при последующих вызовах используется один и тот же экземпляр.

**Вопрос 2.** Какая аннотация позволяет внедрять новый экземпляр prototype-бина в singleton-бин без прямого обращения к `ApplicationContext`?

- A) `@Resource`
- B) `@Lookup`
- C) `@PostProxy`
- **Правильный ответ:** **B**. Аннотация `@Lookup` заставляет Spring переопределить абстрактный или конкретный метод с помощью CGLIB-генерации подкласса, который при каждом вызове запрашивает новый экземпляр из контекста.

**Вопрос 3.** В какой момент жизненного цикла вызывается метод `postProcessBeforeInitialization` интерфейса `BeanPostProcessor`?

- A) До вызова конструктора бина.
- B) После установки свойств (setters), но до методов инициализации (`@PostConstruct` / `afterPropertiesSet`).
- C) После вызова `@PostConstruct`.
- **Правильный ответ:** **B**. `postProcessBeforeInitialization` выполняется сразу после внедрения зависимостей, но строго перед вызовом callback-методов инициализации (`@PostConstruct`, `InitializingBean.afterPropertiesSet`).

---

## 2. Pro Spring 5 — Глава 4: _Spring Configuration in Detail and Spring Boot_

### 📌 Ключевые темы

- Жизненный цикл бина (Bean Lifecycle: `BeanNameAware`, `ApplicationContextAware`, `BeanPostProcessor`, `@PostConstruct`, `@PreDestroy`).
- Java Configuration (`@Configuration`, `@Bean`), разница между Full Mode (`@Configuration`) и Lite Mode (`@Component`).
- Использование `Environment`, `PropertySource` и профайлов (`@Profile`).
- `FactoryBean` и регистрация сложных объектов.

### ❓ Что спросят на Senior-собеседовании

1. Чем отличается вызов `@Bean`-метода из другого `@Bean`-метода внутри класса с аннотацией `@Configuration` от аналогичного вызова внутри класса с `@Component`? Как CGLIB обеспечит Singleton-гарантию?
2. В каком порядке вызываются методы инициализации бина (`@PostConstruct`, `InitializingBean.afterPropertiesSet()`, `init-method`)?
3. Как работает `@Profile` "под капотом" и как устроен механизм `@Conditional`?

### 💻 Практическое задание (ТЗ)

Напишите `@Configuration`-класс с двумя `@Bean`-методами, где `beanB()` напрямую вызывает `beanA()`. Продемонстрируйте разницу в количестве созданных экземпляров `beanA` при замене `@Configuration` на `@Component`. Напишите кастомный `Condition`, который активирует конфигурацию только при наличии определенной переменной окружения.

### 🧪 Практический тест

**Вопрос 1.** Что произойдет при выполнении следующего кода?

```
@Component
public class AppConfig {
    @Bean public Engine engine() { return new Engine(); }
    @Bean public Car car() { return new Car(engine()); }
}
```

- A) В контексте будет ровно один бин `Engine`, который внедрится в `Car`.
- B) Будет создано два разных экземпляра `Engine`: один в контексте Spring, второй напрямую через оператор `new` внутри `car()`.
- C) Вызов `engine()` внутри `car()` вызовет `NullPointerException`.
- **Правильный ответ:** **B**. Поскольку класс аннотирован `@Component` (Lite Mode), CGLIB-проксирование класса конфигурации не применяется. Прямой вызов `engine()` сработает как обычный Java-метод и создаст второй объект `Engine`.

**Вопрос 2.** Какова последовательность вызовов при инициализации бина?

- A) Конструктор -> `@PostConstruct` -> `postProcessBeforeInitialization` -> `afterPropertiesSet`
- B) Конструктор -> `postProcessBeforeInitialization` -> `@PostConstruct` -> `afterPropertiesSet` -> `postProcessAfterInitialization`
- C) Конструктор -> `afterPropertiesSet` -> `@PostConstruct` -> `postProcessAfterInitialization`
- **Правильный ответ:** **B**. Сначала создается объект, затем `BeanPostProcessor` делает preprocessing (где и обрабатывается `@PostConstruct`), после этого вызывается `afterPropertiesSet()` от `InitializingBean`, и в конце `postProcessAfterInitialization`.

**Вопрос 3.** Зачем нужен интерфейс `FactoryBean<T>`?

- A) Для создания динамических прокси AspectJ.
- B) Для инкапсуляции сложной логики создания объектов (например, клиенты третьих библиотек), когда создание невозможно описать простым вызовом конструктора.
- C) Для автоматической генерации DTO.
- **Правильный ответ:** **B**. Если объект имеет сложную инициализацию, метод `getObject()` внутри `FactoryBean` берет эту логику на себя, а Spring регистрирует возвращаемый целевой объект как бин.

---

## 3. Pro Spring Boot — Глава 3: _Spring Boot Auto-Configuration, Features, and More_

### 📌 Ключевые темы

- Механизм Автоконфигурации (`@EnableAutoConfiguration`, `@SpringBootApplication`).
- Поиск конфигураций через `META-INF/spring.factories` (или `AutoConfiguration.imports` в свежих версиях Boot).
- Условные аннотации (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`).
- Иерархия источников конфигурации (`application.yml`, переменные окружения, аргументы CLI).

### ❓ Что спросят на Senior-собеседовании

1. Как Spring Boot определяет, какие автоконфигурации нужно применить при старте приложения?
2. Каков приоритет загрузки конфигурационных параметров (Command Line arguments vs `application.yml` vs Environment variables)?
3. Как написать собственный **Custom Spring Boot Starter** и правильно настроить условное создание бинов (`@ConditionalOnMissingBean`)?

### 💻 Практическое задание (ТЗ)

Разработайте собственный Spring Boot Starter (`rate-limiter-spring-boot-starter`). Реализуйте автоконфигурацию, которая создает бин `RateLimiterFilter`, если в `application.yml` установлено `app.rate-limiter.enabled=true`. Если пользователь объявил свой бин такого типа, автоконфигурация должна уступить место пользовательскому бину (`@ConditionalOnMissingBean`).

### 🧪 Практический тест

**Вопрос 1.** Какой класс отвечает за считывание автоконфигураций из служебных файлов при старте Spring Boot?

- A) `AutoConfigurationManager`
- B) `EnableAutoConfigurationImportSelector` (или `AutoConfigurationImportSelector`)
- C) `SpringApplicationContextRegistry`
- **Правильный ответ:** **B**. `AutoConfigurationImportSelector` использует `SpringFactoriesLoader` для чтения записей `EnableAutoConfiguration` из файлов `META-INF/spring.factories`.

**Вопрос 2.** Какая аннотация гарантирует, что автоконфигурационный бин создастся только в том случае, если пользователь НЕ объявлял собственный бин того же типа?

- A) `@ConditionalOnBean`
- B) `@ConditionalOnMissingBean`
- C) `@ConditionalOnResource`
- **Правильный answer:** **B**. `@ConditionalOnMissingBean` проверяет наличие контекстного бина указанного типа и отменяет создание автоконфигурационного бина, если пользователь уже создал свой.

**Вопрос 3.** Какой источник параметров конфигурации имеет НАИВЫСШИЙ приоритет в Spring Boot?

- A) `[подозрительная ссылка удалена]` внутри JAR
- B) Переменные окружения ОС (OS Environment Variables)
- C) Аргументы командной строки (Command-line arguments)
- **Правильный ответ:** **C**. Аргументы командной строки переопределяют все остальные источники конфигурации (включая OS Environment variables и `application.yml`).

---

# 🔴 ПРИОРИТЕТ 2: Транзакции и работа с базами данных (JPA / Hibernate)

---

## 4. Pro Spring 5 — Глава 9: _Transaction Management_

### 📌 Ключевые темы

- Декларативные транзакции (`@Transactional`) и `PlatformTransactionManager`.
- Уровни распространения транзакций (Propagation: `REQUIRED`, `REQUIRES_NEW`, `NESTED`, `MANDATORY` и др.).
- Уровни изоляции (Isolation Levels) и правила отката (Rollback rules: Checked vs Unchecked).
- Проблема самовызова (Self-invocation) при использовании AOP-прокси.

### ❓ Что спросят на Senior-собеседовании

1. Почему `@Transactional` не срабатывает при вызове метода из другого метода того же класса (`this.method()`)? Как это связано с AOP-прокси?
2. Что произойдет при возбуждении `Checked Exception` (наследуемого от `Exception`) внутри метода с `@Transactional` по умолчанию?
3. В чём различие между `REQUIRES_NEW` и `NESTED` распространением транзакций?

### 💻 Практическое задание (ТЗ)

Смоделируйте ошибку с "потерянной" транзакцией при внутреннем вызове. В сервисе `OrderService` создайте метод `processBatch()`, который без аннотации `@Transactional` вызывает метод `processItem()` того же класса с `@Transactional(propagation = Propagation.REQUIRES_NEW)`. Продемонстрируйте отсутствие создания новой транзакции и исправьте архитектуру путем выделения отдельного сервиса.

### 🧪 Практический тест

**Вопрос 1.** Будет ли откатана транзакция по умолчанию, если метод с `@Transactional` выбросит `IOException`?

- A) Да, любые исключения откатывают транзакцию.
- B) Нет, по умолчанию транзакция откатывается только при `RuntimeException` (Unchecked) и `Error`.
- C) Да, если указан `readOnly = true`.
- **Правильный ответ:** **B**. По умолчанию Spring откатывает транзакцию только при необработанных `Unchecked Exception`. Для отката при `Checked Exception` нужно явно указывать `@Transactional(rollbackFor = IOException.class)`.

**Вопрос 2.** Чем отличается `Propagation.REQUIRES_NEW` от `Propagation.NESTED`?

- A) Ничем, это синонимы.
- B) `REQUIRES_NEW` всегда приостанавливает внешнюю транзакцию и создает новую независимую физическую транзакцию; `NESTED` использует точек сохранения (Savepoints) в рамках существующей физической транзакции.
- C) `NESTED` работает только с NoSQL базами данных.
- **Правильный ответ:** **B**. `REQUIRES_NEW` создает полностью независимую физическую транзакцию в БД. `NESTED` открывает вложенную транзакцию с помощью БД Savepoint: откат вложенной транзакции не приводит к автоматическому откату внешней.

**Вопрос 3.** В методе `A` (`REQUIRED`) вызывается метод `B` (`REQUIRES_NEW`). В методе `B` выбрасывается `RuntimeException`. Метод `A` перехватывает исключение через `try-catch`. Что произойдет при завершении метода `A`?

- A) Транзакция метода `B` откатится, а транзакция метода `A` успешно закоммитится.
- B) Вся транзакция откатится с ошибкой `UnexpectedRollbackException`.
- C) Обе транзакции упадут с `DeadlockException`.
- **Правильный ответ:** **A**. Так как `B` выполнялся в отдельной физической транзакции (`REQUIRES_NEW`), его откат не пометил транзакцию `A` как `rollback-only`. Посколько `A` перехватил исключение, его транзакция закоммитится успешно.

---

## 5. Pro Spring 5 — Глава 8 & 7: _Data Access in Spring with JPA2 / Hibernate_

### 📌 Ключевые темы

- `EntityManagerFactory`, `EntityManager` и управление состояниями сущностей (Transient, Persistent, Detached, Removed).
- Проблема N+1 и способы её решения (`JOIN FETCH`, `@EntityGraph`, DTO Projections).
- Кэширование L1 (Persistence Context) и L2 (Ehcache/Hazelcast).
- Блокировки: Оптимистические (`@Version`) и Пессимистические (`LockModeType`).

### ❓ Что спросят на Senior-собеседовании

1. Как возникнет проблема N+1 при вызове `repository.findAll()` для сущности со связью `@OneToMany(fetch = FetchType.LAZY)` и как её решить с помощью `@EntityGraph`?
2. Как работает механизм Dirty Checking в Hibernate и когда именно происходят SQL-запросы `UPDATE`?
3. Что происходит при возникновении `OptimisticLockException` и как организовать повторную попытку (retry) обработки операции?

### 💻 Практическое задание (ТЗ)

Создайте сущности `Author` (1) и `Book` (N). Воспроизведите проблему N+1 при вызове `authorRepository.findAll()`. Напишите 3 варианта решения: `JOIN FETCH` в JPQL, `@EntityGraph` и DTO-проекцию. Добавьте в `Book` поле `@Version` и напишите многопоточный тест, стимулирующий параллельное изменение записи двумя потоками с обработкой `ObjectOptimisticLockingFailureException`.

### 🧪 Практический тест

**Вопрос 1.** В каком случае произойдет ошибка `LazyInitializationException`?

- A) При попытке обратиться к Lazy-коллекции сущности, когда Persistence Context (сессия `EntityManager`) уже закрыт.
- B) При получении сущности с ошибочной `@Version`.
- C) При вызове метода `save()` для Detached сущности.
- **Правильный ответ:** **A**. LAZY-загрузка требует активной сессии JPA. Если сессия закрыта (например, за пределами `@Transactional` метода), обращение к непроинициализированной коллекционной связи вызовет `LazyInitializationException`.

**Вопрос 2.** Какой SQL-запрос сгенерирует Spring Data JPA для метода `findAll()` при наличии аннотации `@EntityGraph(attributePaths = {"books"})` над методом репозитория?

- A) `SELECT * FROM author` и N отдельных запросов `SELECT * FROM book WHERE author_id = ?`
- B) Один запрос с `LEFT OUTER JOIN book ...`.
- C) Два запроса: один для всех авторов и один для всех книг с условием `WHERE author_id IN (...)`.
- **Правильный ответ:** **B**. `@EntityGraph` даёт указание провайдеру JPA загрузить указанные графы связей в рамках одного SQL-запроса через JOIN.

**Вопрос 3.** Что делает аннотация `@Version` на поле сущности JPA?

- A) Автоматически инкрементирует версию схемы базы данных при Flyway-миграциях.
- B) Включает оптимистическую блокировку, проверяя значение поля версии в условии `WHERE id = ? AND version = ?` при каждом `UPDATE`.
- C) Обеспечивает сохранение истории изменений сущности в отдельную таблицу аудита.
- **Правильный ответ:** **B**. Это механизм оптимистической блокировки (Optimistic Locking). Если версия в БД изменилась другим потоком, сгенерируется `OptimisticLockException`.

---

# 🔴 ПРИОРИТЕТ 3: AOP и Проксирование

---

## 6. Pro Spring 5 — Глава 5: _Introducing Spring AOP_

### 📌 Ключевые темы

- Понятия AOP: Aspect, Joinpoint, Pointcut, Advice (`@Before`, `@After`, `@Around`, `@AfterThrowing`).
- JDK Dynamic Proxies (на основе интерфейсов) vs CGLIB Proxies (на основе наследования классов).
- Ограничения Spring AOP (отсутствие interception для private/final методов и вызовов внутри класса).
- Интеграция с AspectJ (Compile-time / Load-time weaving).

### ❓ Что спросят на Senior-собеседовании

1. Каковы фундаментальные различия между JDK Dynamic Proxy и CGLIB в Spring AOP? Какой прокси используется по умолчанию в Spring Boot 2.x+?
2. Почему Spring AOP не перехватывает вызовы `private`-методов или вызовы методов внутри одного и того же класса?
3. Чем отличается Spring AOP (Proxy-based) от AspectJ (Bytecode Weaving)?

### 💻 Практическое задание (ТЗ)

Напишите кастомный аспект с аннотацией `@LogExecutionTime(thresholdMs = 500)`. Аспект должен перехватывать вызовы методов с этой аннотацией, замерять время с помощью `StopWatch` и, если оно превышает `thresholdMs`, логировать предупреждение. Продемонстрируйте работу аспекта для интерфейса (JDK Proxy) и для класса без интерфейсов (CGLIB).

### 🧪 Практический тест

**Вопрос 1.** Какое ограничение имеет проксирование на основе CGLIB?

- A) Класс целевого бина должен обязательно реализовывать хотя бы один интерфейс.
- B) Целевой класс и его целевые методы не могут быть объявлены как `final`.
- C) CGLIB не поддерживает `@Around` советы (Advices).
- **Правильный ответ:** **B**. CGLIB генерирует подкласс на лету. В Java невозможно унаследоваться от `final`-класса или переопределить `final`-метод.

**Вопрос 2.** Какое выражение AspectJ Pointcut выберет все публичные методы в сервисах пакета `[подозрительная ссылка удалена].service`?

- A) `execution(public * [подозрительная ссылка удалена].service.*.*(..))`
- B) `within([подозрительная ссылка удалена].service..*)`
- C) `@annotation([подозрительная ссылка удалена].service.Public)`
- **Правильный ответ:** **A**. Синтаксис `execution(модификатор_доступа возвращаемый_тип пакет.класс.метод(параметры))` с шаблоном `public * [подозрительная ссылка удалена].service.*.*(..)` выберет все публичные методы любых классов данного пакета с любыми аргументами.

**Вопрос 3.** Какую роль выполняет объект `ProceedingJoinPoint` в аспектном методе с аннотацией `@Around`?

- A) Позволяет изменить аннотации целевого класса.
- B) Предоставляет метод `proceed()`, контролирующий непосредственный вызов целевого метода или следующего перехватчика в цепочке.
- C) Используется только для получения имени класса.
- **Правильный ответ:** **B**. `@Around` дает полный контроль над исполнением: метод `proceed()` можно вызывать, не вызывать вообще (заблокировав метод) или вызывать в блоке `try-catch` / цикле.

---

# 🔴 ПРИОРИТЕТ 4: Web-стек, REST и Реактивный стек

---

## 7. Pro Spring 5 — Глава 16 & Pro Spring Boot — Глава 8: _Web Applications (Spring MVC)_

### 📌 Ключевые темы

- Архитектура Spring MVC и жизненный цикл запроса (`DispatcherServlet`, `HandlerMapping`, `HandlerAdapter`, `ViewResolver`).
- RESTful контроллеры (`@Controller` vs `@RestController`, `@ResponseBody`).
- Глобальная обработка ошибок (`@ControllerAdvice`, `@ExceptionHandler`, RFC 7807 `ProblemDetail`).
- Валидация данных (`@Valid`, `@Validated`, `BindingResult`).

### ❓ Что спросят на Senior-собеседовании

1. Опишите полный путь HTTP-запроса внутри `DispatcherServlet` от поступления на сервер до формирования JSON-ответа.
2. В чём различие между контроллером `@Controller` и `@RestController`? Как работает `HttpMessageConverter`?
3. Как правильно организовать обработку ошибок в REST API с помощью `@RestControllerAdvice`?

### 💻 Практическое задание (ТЗ)

Реализуйте глобальный обработчик ошибок REST API на базе `@RestControllerAdvice`. Перехватывайте `MethodArgumentNotValidException` (ошибки валидации DTO) и кастомное `ResourceNotFoundException`. Возвращайте клиенту структурированный JSON формата **RFC 7807 (ProblemDetail)** с указанием HTTP-статуса, таймстемпа и списка некорректных полей.

### 🧪 Практический тест

**Вопрос 1.** Какой компонент Spring MVC отвечает за преобразование возвращаемого объектом Java в формат JSON или XML для отправки в тело ответа?

- A) `ViewResolver`
- B) `HttpMessageConverter` (например, `MappingJackson2HttpMessageConverter`)
- C) `HandlerInterceptor`
- **Правильный ответ:** **B**. При наличии `@ResponseBody` (или использовании `@RestController`) за дело берётся `HttpMessageConverter`, считывающий/записывающий данные напрямую в HTTP stream.

**Вопрос 2.** Чем отличается аннотация `@Valid` от `@Validated`?

- A) `@Valid` входит в стандарт JSR-303/JSR-380 (Bean Validation), а `@Validated` — аннотация Spring, поддерживающая группы валидации (Validation Groups).
- B) `@Validated` работает только в Spring WebFlux.
- C) Разницы нет, это абсолютные синонимы.
- **Правильный ответ:** **A**. `@Valid` — стандартная аннотация SE/EE. `@Validated` — расширение от Spring, позволяющее указывать группы валидации на уровне параметров метода.

**Вопрос 3.** Что произойдёт, если метод `@ExceptionHandler` объявлен внутри класса с аннотацией `@RestControllerAdvice`?

- A) Он будет обрабатывать исключения только внутри одного контроллера.
- B) Он станет перехватывать указанные исключения глобально для всех контроллеров приложения.
- C) Он перехватит ошибки только на этапе инициализации Spring Context.
- **Правильный ответ:** **B**. `@RestControllerAdvice` является специализацией `@Component` и регистрирует глобальный перехватчик ошибок для всех контроллеров.

---

## 8. Pro Spring 5 — Глава 18: _Spring WebFlux & Reactive Streams_

### 📌 Ключевые темы

- Реактивное программирование и спецификация Reactive Streams (Publisher, Subscriber, Subscription, Processor).
- Project Reactor (`Mono` [0..1] и `Flux` [0..N]).
- Архитектура Event Loop (Netty) vs Thread-per-request (Tomcat).
- Реактивный HTTP-клиент `WebClient`.

### ❓ Что спросят на Senior-собеседовании

1. В чём ключевое различие модели Threads-per-request (Spring MVC) и Event Loop (Spring WebFlux / Netty)?
2. Что происходит, если внутри реактивного цепочки вызовов WebFlux вызвать блокирующий код (например, JDBC или `Thread.sleep()`)? Как это исправить (`publishOn`/`subscribeOn`)?
3. Какова различие между типами `Mono` и `Flux` в Project Reactor?

### 💻 Практическое задание (ТЗ)

Реализуйте неблокирующий реактивный сервис. Используя `WebClient`, отправьте два параллельных запроса к сторонним API, объедините их результаты с помощью `[подозрительная ссылка удалена]()`, задайте жесткий таймаут 2 секунды (`.timeout()`) и обработайте фоллбэк с помощью `.onErrorReturn()` при возникновении ошибки.

### 🧪 Практический тест

**Вопрос 1.** Что произойдет с реактивным потоком Reactor (`Mono`/`Flux`), если над ним вызвать цепочку операторов `map()` или `flatMap()`, но НЕ подписаться на него (`.subscribe()`)?

- A) Запрос выполнится асинхронно в фоновом потоке.
- B) Ничего не произойдет (поток не выполнится), так как реактивные потоки ленивые ("Nothing happens until you subscribe").
- C) Выбросится `IllegalStateException`.
- **Правильный ответ:** **B**. В Reactive Streams сборка цепочки операторов лишь описывает граф обработки. Исполнение кода начинается исключительно в момент оформления подписки (`subscribe()`).

**Вопрос 2.** Какое правило следует соблюдать при работе с `WebClient` в Spring WebFlux?

- A) `WebClient` должен создаваться заново на каждый HTTP-запрос.
- B) `WebClient` является потокобезопасным (Thread-safe) и переиспользуемым компонентом; следует создавать один экземпляр на приложение.
- C) `WebClient` работает только с синхронными контроллерами.
- **Правильный ответ:** **B**. `WebClient` спроектирован как неблокирующий, иммутабельный и потокобезопасный клиент.

**Вопрос 3.** Какой оператор Reactor следует использовать для выноса блокирующего вызова (например, обращения к JDBC) на специальный пул потоков?

- A) `subscribeOn(Schedulers.boundedElastic())`
- B) `doOnError()`
- C) `retry()`
- **Правильный ответ:** **A**. Пул `Schedulers.boundedElastic()` специально предназначен для выноса блокирующих I/O операций из основного Event Loop потока WebFlux.

---

# 🔴 ПРИОРИТЕТ 5: Безопасность и Production-Ready механизмы

---

## 9. Pro Spring Boot — Глава 9: _Security with Spring Boot_

### 📌 Ключевые темы

- Архитектура Spring Security (`DelegatingFilterProxy`, `FilterChainProxy`, `SecurityFilterChain`).
- Контекст безопасности: `SecurityContextHolder`, `SecurityContext`, `Authentication`.
- Stateless REST API: Аутентификация на базе JWT (JSON Web Tokens).
- Защита от атак (CSRF, CORS) и авторизация методов (`@PreAuthorize`, `@Secured`).

### ❓ Что спросят на Senior-собеседовании

1. Как `DelegatingFilterProxy` связывает стандартные Сервлет-фильтры контейнера приложений с цепочкой фильтров Spring Security?
2. Почему для Stateless REST API с аутентификацией по JWT следует отключать защиту CSRF?
3. Каким образом `SecurityContextHolder` хранит данные пользователя по умолчанию (`MODE_THREADLOCAL`) и в чём нюансы передачи контекста при асинхронных вызовах (`@Async`)?

### 💻 Практическое задание (ТЗ)

Настройте конфигурацию `SecurityFilterChain` для Stateless REST API. Реализуйте кастомный фильтр `OncePerRequestFilter`, который извлекает JWT из заголовка `Authorization`, валидирует его, формирует `UsernamePasswordAuthenticationToken` и сохраняет его в `SecurityContextHolder`. Ограничьте доступ к эндпоинтам по ролям (`ROLE_ADMIN` и `ROLE_USER`).

### 🧪 Практический тест

**Вопрос 1.** Какая стратегия хранения `SecurityContext` используется в `SecurityContextHolder` по умолчанию?

- A) `MODE_GLOBAL`
- B) `MODE_THREADLOCAL`
- C) `MODE_INHERITABLETHREADLOCAL`
- **Правильный ответ:** **B**. По умолчанию контекст сохраняется в `ThreadLocal`, что делает его доступным для всех методов в рамках текущего потока обработки запроса.

**Вопрос 2.** Зачем в REST API с JWT-авторизацией явно отключают CSRF (`csrf.disable()`)?

- A) CSRF-атаки невозможны при авторизации через токены в заголовках, так как браузеры не подставляют кастомные заголовки `Authorization` автоматически (в отличие от Cookies).
- B) Отключение CSRF ускоряет шифрование паролей.
- C) CSRF отключит CORS проверки.
- **Правильный ответ:** **A**. CSRF-атака основана на автоматической отправке браузером сессионных кук (Session Cookies). Если API использует токены в заголовках, CSRF-защита не требуется.

**Вопрос 3.** Базовым интерфейсом для объявления цепочки правил безопасности в современных версиях Spring Security является:

- A) `WebSecurityConfigurerAdapter`
- B) `SecurityFilterChain`
- C) `AuthenticationManagerBuilder`
- **Правильный ответ:** **B**. Начиная со Spring Security 5.7 / Spring Boot 2.7+, класс `WebSecurityConfigurerAdapter` признан устаревшим, а конфигурация строится вокруг бина `SecurityFilterChain`.

---

## 10. Pro Spring Boot — Глава 11: _Spring Boot Actuator_

### 📌 Ключевые темы

- Production-ready эндпоинты (`/actuator/health`, `/actuator/metrics`, `/actuator/env`, `/actuator/prometheus`).
- Создание кастомных `HealthIndicator`.
- Сбор метрик с использованием Micrometer (`Counter`, `Timer`, `Gauge`).
- Безопасность и управление доступом к эндпоинтам Actuator.

### ❓ Что спросят на Senior-собеседовании

1. Какова структура и назначение Actuator-эндпоинтов в продакшен-среде?
2. Как написать кастомный `HealthIndicator` для проверки подключения к внешнему сервису/БД?
3. Как правильно разграничить доступ к чувствительным эндпоинтам Actuator (`/env`, `/heapdump`) в конфигурации безопасности?

### 💻 Практическое задание (ТЗ)

Реализуйте кастомный `HealthIndicator`, проверяющий доступность стороннего сервиса. Добавьте кастомные бизнес-метрики с помощью библиотеки Micrometer: `Counter` (счётчик заказов) и `Timer` (время обработки запроса). Настройте безопасность Actuator так, чтобы эндпоинты `/actuator/prometheus` и `/actuator/health` были открыты для систем мониторинга, а остальные заблокированы.

### 🧪 Практический тест

**Вопрос 1.** Какой интерфейс нужно реализовать для создания собственного индикатора здоровья приложения в Spring Boot Actuator?

- A) `HealthIndicator` (или унаследоваться от `AbstractHealthIndicator`)
- B) `MetricsContributor`
- C) `StatusChecker`
- **Правильный ответ:** **A**. Реализация метода `health()` интерфейса `HealthIndicator` позволяет вернуть статус `Health.up()` или `Health.down()` с дополнительными деталями.

**Вопрос 2.** Какая форма сбора метрик в Micrometer отвечает за измерение текущего значения (например, размер очереди в памяти или задействованная память JVM)?

- A) `Counter`
- B) `Gauge`
- C) `Timer`
- **Правильный ответ:** **B**. `Gauge` показывает текущее мгновенное значение величины. `Counter` умеет только увеличиваться, а `Timer` измеряет длительность и частоту событий.

**Вопрос 3.** Какое свойство в `application.yml` открывает доступ ко всем эндпоинтам Actuator по HTTP в Spring Boot по умолчанию?

- A) `[подозрительная ссылка удалена].exposure.include="*"`
- B) `management.actuator.enable-all=true`
- C) `[подозрительная ссылка удалена]=true`
- **Правильный ответ:** **A**. По умолчанию из соображений безопасности открыты только `/health`. Для экспозиции других эндпоинтов используется `[подозрительная ссылка удалена].exposure.include`.

---

# 🔴 ПРИОРИТЕТ 6: Тестирование

---

## 11. Pro Spring 5 — Глава 13 & Pro Spring Boot — Глава 6: _Testing with Spring & Spring Boot_

### 📌 Ключевые темы

- Юнит-тестирование vs Интеграционное тестирование.
- Тестовые срезы (Test Slices): `@WebMvcTest`, `@DataJpaTest`, `@JsonTest`.
- Мокание зависимостей в контексте Spring: `@MockBean` / `@SpyBean`.
- Интеграционное тестирование с контейнерами: **Testcontainers** и `@SpringBootTest`.

### ❓ Что спросят на Senior-собеседовании

1. Чем отличается сквозной тест `@SpringBootTest` от изолированного среза `@WebMvcTest`?
2. В чём различие между аннотациями `@Mock` (Mockito) и `@MockBean` (Spring Test)?
3. Как предотвратить частый перезапуск Spring Context при прогоне большого количества интеграционных тестов (кэширование контекста)?

### 💻 Практическое задание (ТЗ)

Напишите 2 уровня тестов для REST-сервиса:

1. Срез-тест `@WebMvcTest` с использованием `@MockBean` для проверки контроллера и HTTP-валидации DTO.
2. Интеграционный тест `@SpringBootTest` с использованием **Testcontainers** (поднятие реального контейнера PostgreSQL), проверяющий корректность записи в БД и транзакционную изоляцию тестов.

### 🧪 Практический тест

**Вопрос 1.** Какое преимущество дает использование аннотации `@WebMvcTest` по сравнению с `@SpringBootTest`?

- A) Она поднимает полноценный сервер Embedded Tomcat.
- B) Она подгружает только Web-слой (контроллеры, конвертеры, фильтры), что значительно ускоряет выполнение тестов за счёт отказа от загрузки базы данных и сервисов.
- C) Она автоматически мокает все репозитории без применения дополнительных аннотаций.
- **Правильный ответ:** **B**. `@WebMvcTest` — это срез (Test Slice), создающий минимальный контекст, необходимый исключительно для тестирования MVC-компонентов.

**Вопрос 2.** Что делает аннотация `@MockBean` в тестовом классе Spring Boot?

- A) Создает обычный Mockito-мок, не связанный со Spring.
- B) Создает Mockito-мок объекта и заменяет им существующий бин того же типа внутри Spring `ApplicationContext`.
- C) Помечает метод контроллера как заблокированный для вызова.
- **Правильный ответ:** **B**. `@MockBean` интегрирует мок-объект непосредственно в контекст Spring, заменяя оригинальный бин на время выполнения теста.

**Вопрос 3.** Какое поведение по умолчанию демонстрируют интеграционные тесты с аннотацией `@Transactional` при работе с БД?

- A) Коммитят все данные в базу по завершении каждого метода теста.
- B) Автоматически откатывают (rollback) все изменения в базе данных в конце каждого тестового метода.
- C) Блокируют базу данных от записи до перезапуска тестов.
- **Правильный ответ:** **B**. В тестовом окружении `@Transactional` по умолчанию откатывает транзакцию после каждого теста, обеспечивая изоляцию и чистоту данных.

---

💡 _Хотите подробно разобрать код одного из практических заданий или разобрать конкретный сложный кейс (например, настройку Testcontainers или написание Custom Starter)?_