# Шпаргалка по безопасности Java

## Предотвращение инъекций в Java

В этом разделе представлены советы по обработке *инъекций* в коде Java-приложений.

Пример кода, используемый в советах, можно найти [здесь](https://github.com/righettod/injection-cheat-sheets).

### Что такое инъекция

[Инъекция](https://owasp.org/www-project-top-ten/OWASP_Top_Ten_2017/Top_10-2017_A1-Injection) в OWASP Top 10 определяется следующим образом:

*Рассматривайте любого, кто может отправить непроверенные данные в систему, включая внешних пользователей, внутренних пользователей и администраторов.*

### Общие советы по предотвращению инъекций

Следующие рекомендации можно применять в общем виде для предотвращения проблемы *инъекции*:

1. Применяйте **Проверку Входных Данных** (используя подход с белым списком) в сочетании с **Очисткой и Экранированием Выходных Данных** для ввода/вывода пользователя.
2. Если вам нужно взаимодействовать с системой, старайтесь использовать возможности API, предоставляемые вашим технологическим стеком (Java / .Net / PHP...), вместо того чтобы строить команды самостоятельно.

Дополнительные советы можно найти в этой [шпаргалке](Input_Validation_Cheat_Sheet.md).

## Конкретные типы инъекций

*Примеры в этом разделе будут предоставлены на технологии Java (см. связанный проект Maven), но советы применимы и к другим технологиям, таким как .Net / PHP / Ruby / Python...*

### SQL

#### Симптом

Инъекция этого типа происходит, когда приложение использует непроверенные входные данные для построения SQL-запроса с помощью строки и выполнения его.

#### Как предотвратить

Используйте *Параметризацию Запросов* для предотвращения инъекций.

#### Пример

```java
/*Не используется фреймворк БД, чтобы показать реальное использование
  Prepared Statement из Java API*/
/*Открываем соединение с базой данных H2 и используем её*/
Class.forName("org.h2.Driver");
String jdbcUrl = "jdbc:h2:file:" + new File(".").getAbsolutePath() + "/target/db";
try (Connection con = DriverManager.getConnection(jdbcUrl)) {

    /* Пример A: Выбор данных с использованием Prepared Statement*/
    String query = "select * from color where friendly_name = ?";
    List<String> colors = new ArrayList<>();
    try (PreparedStatement pStatement = con.prepareStatement(query)) {
        pStatement.setString(1, "yellow");
        try (ResultSet rSet = pStatement.executeQuery()) {
            while (rSet.next()) {
                colors.add(rSet.getString(1));
            }
        }
    }

    /* Пример B: Вставка данных с использованием Prepared Statement*/
    query = "insert into color(friendly_name, red, green, blue) values(?, ?, ?, ?)";
    int insertedRecordCount;
    try (PreparedStatement pStatement = con.prepareStatement(query)) {
        pStatement.setString(1, "orange");
        pStatement.setInt(2, 239);
        pStatement.setInt(3, 125);
        pStatement.setInt(4, 11);
        insertedRecordCount = pStatement.executeUpdate();
    }

   /* Пример C: Обновление данных с использованием Prepared Statement*/
    query = "update color set blue = ? where friendly_name = ?";
    int updatedRecordCount;
    try (PreparedStatement pStatement = con.prepareStatement(query)) {
        pStatement.setInt(1, 10);
        pStatement.setString(2, "orange");
        updatedRecordCount = pStatement.executeUpdate();
    }

   /* Пример D: Удаление данных с использованием Prepared Statement*/
    query = "delete from color where friendly_name = ?";
    int deletedRecordCount;
    try (PreparedStatement pStatement = con.prepareStatement(query)) {
        pStatement.setString(1, "orange");
        deletedRecordCount = pStatement.executeUpdate();
    }

}
```

#### Ссылки

- [Шпаргалка по предотвращению SQL-инъекций](SQL_Injection_Prevention_Cheat_Sheet.md)

### JPA

#### Симптом

Инъекция этого типа происходит, когда приложение использует непроверенные входные данные для построения JPA-запроса с помощью строки и выполнения его. Это похоже на SQL-инъекцию, но здесь изменённый язык — не SQL, а JPA QL.

#### Как предотвратить

Используйте **Параметризацию Запросов** на языке запросов Java Persistence для предотвращения инъекций.

#### Пример

```java
EntityManager entityManager = null;
try {
    /* Получаем ссылку на EntityManager для доступа к БД */
    entityManager = Persistence.createEntityManagerFactory("testJPA").createEntityManager();

    /* Определяем параметризованный прототип запроса, используя именованный параметр для улучшения читаемости */
    String queryPrototype = "select c from Color c where c.friendlyName = :colorName";

    /* Создаем запрос, устанавливаем именованный параметр и выполняем запрос */
    Query queryObject = entityManager.createQuery(queryPrototype);
    Color c = (Color) queryObject.setParameter("colorName", "yellow").getSingleResult();

} finally {
    if (entityManager != null && entityManager.isOpen()) {
        entityManager.close();
    }
}
```

#### Ссылки

- [SQLi и JPA](https://software-security.sans.org/developer-how-to/fix-sql-injection-in-java-persistence-api-jpa)

### Операционная система

#### Симптом

Инъекция этого типа происходит, когда приложение использует непроверенные входные данные для построения команды операционной системы с помощью строки и её выполнения.

#### Как предотвратить

Используйте **API** вашего технологического стека для предотвращения инъекций.

#### Пример

```java
/* В данном контексте выполняется, например, PING-команда к компьютеру.
* Профилактика заключается в использовании возможностей, предоставляемых API Java, 
* вместо создания системной команды в виде строки и её выполнения */
InetAddress host = InetAddress.getByName("localhost");
var reachable = host.isReachable(5000);
```

#### Ссылки

- [Инъекция команд](https://owasp.org/www-community/attacks/Command_Injection)

### XML: Инъекция XPath

#### Симптом

Инъекция этого типа происходит, когда приложение использует непроверенные входные данные для построения XPath-запроса с помощью строки и его выполнения.

#### Как предотвратить

Используйте **XPath Variable Resolver** для предотвращения инъекций.

#### Пример

**Реализация Variable Resolver**.

```java
/**
 * Резолвер для определения параметра для XPATH-выражения.
 *
 */
public class SimpleVariableResolver implements XPathVariableResolver {

    private final Map<QName, Object> vars = new HashMap<QName, Object>();

    /**
     * Внешний метод для добавления параметра
     *
     * @param name Имя параметра
     * @param value Значение параметра
     */
    public void addVariable(QName name, Object value) {
        vars.put(name, value);
    }

    /**
     * {@inheritDoc}
     *
     * @see javax.xml.xpath.XPathVariableResolver#resolveVariable(javax.xml.namespace.QName)
     */
    public Object resolveVariable(QName variableName) {
        return vars.get(variableName);
    }
}
```

Код, использующий его для выполнения XPath-запроса.

```java
/*Создаем фабрику для построения XML-документов*/
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();

/*Отключаем разрешение внешних сущностей для различных случаев*/
//Не выполняется здесь, чтобы сосредоточиться на коде резолвера переменных
//но сделайте это для продакшн-кода!

/*Загружаем XML-файл*/
DocumentBuilder builder = dbf.newDocumentBuilder();
Document doc = builder.parse(new File("src/test/resources/SampleXPath.xml"));

/* Создаем и настраиваем резолвер параметров */
String bid = "bk102";
SimpleVariableResolver variableResolver = new SimpleVariableResolver();
variableResolver.addVariable(new QName("bookId"), bid);

/*Создаем и настраиваем XPATH-выражение*/
XPath xpath = XPathFactory.newInstance().newXPath();
xpath.setXPathVariableResolver(variableResolver);
XPathExpression xPathExpression = xpath.compile("//book[@id=$bookId]");

/* Применяем выражение к XML-документу */
Object nodes = xPathExpression.evaluate(doc, XPathConstants.NODESET);
NodeList nodesList = (NodeList) nodes;
Element book = (Element)nodesList.item(0);
var containsRalls = book.getTextContent().contains("Ralls, Kim");
```

#### Ссылки

- [Инъекция XPath](https://owasp.org/www-community/attacks/XPATH_Injection)

### HTML/JavaScript/CSS

#### Симптом

Инъекция этого типа происходит, когда приложение использует непроверенные входные данные для формирования HTTP-ответа и отправляет его в браузер.

#### Как предотвратить

Либо применяйте строгую проверку входных данных (подход с белым списком), либо используйте очистку и экранирование выходных данных, если проверка входных данных невозможна (комбинируйте оба подхода, когда это возможно).

#### Пример

```java
/*
СПОСОБ ВВОДА: Получение данных от пользователя
Здесь рекомендуется использовать строгую проверку входных данных с использованием подхода с белым списком.
Фактически, вы гарантируете, что только разрешенные символы входят в полученные данные.
*/

String userInput = "You user login is owasp-user01";

/* Сначала проверяем, что значение содержит только ожидаемые символы */
if (!Pattern.matches("[a-zA-Z0-9\\s\\-]{1,50}", userInput)) {
    return false;
}

/* Если первая проверка прошла, убедитесь, что потенциально опасные символы,
которые мы разрешили для бизнес-требований, не используются опасным образом.
Например, здесь мы разрешили символ '-', который может
использоваться в SQL-инъекции, поэтому мы
убедимся, что этот символ не используется в непрерывной форме.
Используйте API COMMONS LANG v3 для анализа строки... */
if (0 != StringUtils.countMatches(userInput.replace(" ", ""), "--")) {
    return false;
}

/*
СПОСОБ ВЫВОДА: Отправка данных пользователю
Здесь мы экранируем и очищаем любые данные, отправляемые пользователю.
Используйте API OWASP Java HTML Sanitizer для обработки очистки.
Используйте API OWASP Java Encoder для обработки кодирования (экранирования) HTML-тегов.
*/

String outputToUser = "You <p>user login</p> is <strong>owasp-user01</strong>";
outputToUser += "<script>alert(22);</script><img src='#' onload='javascript:alert(23);'>";

/* Создаем политику очистки, которая разрешает только теги '<p>' и '<strong>' */
PolicyFactory policy = new HtmlPolicyBuilder().allowElements("p", "strong").toFactory();

/* Очищаем вывод, который будет отправлен пользователю */
String safeOutput = policy.sanitize(outputToUser);

/* Кодируем HTML-теги */
safeOutput = Encode.forHtml(safeOutput);
String finalSafeOutputExpected = "You <p>user login</p> is <strong>owasp-user01</strong>";
if (!finalSafeOutputExpected.equals(safeOutput)) {
    return false;
}
```

#### Ссылки

- [XSS](https://owasp.org/www-community/attacks/xss/)
- [OWASP Java HTML Sanitizer](https://github.com/owasp/java-html-sanitizer)
- [OWASP Java Encoder](https://github.com/owasp/owasp-java-encoder)
- [Java RegEx](https://docs.oracle.com/javase/8/docs/api/java/util/regex/Pattern.html)

### LDAP

Создана специализированная [шпаргалка](LDAP_Injection_Prevention_Cheat_Sheet.md).

### NoSQL

#### Симптом

Инъекция этого типа происходит, когда приложение использует непроверенные входные данные для построения выражения вызова NoSQL API.

#### Как предотвратить

Поскольку существует множество систем баз данных NoSQL, каждая из которых использует свой API для вызовов, важно убедиться, что полученные от пользователя входные данные, используемые для построения выражения вызова API, не содержат символов, имеющих специальное значение в синтаксисе целевого API. Это необходимо для предотвращения использования этих символов для изменения исходного выражения вызова с целью создания другого на основе подделанных входных данных пользователя. Также важно не использовать конкатенацию строк для построения выражений вызова API, а использовать API для создания выражения.

#### Пример - MongoDB

```java
 /* Здесь используется MongoDB в качестве целевой NoSQL БД */
String userInput = "Brooklyn";

/* Сначала убедитесь, что входные данные не содержат специальных символов
для текущего NoSQL DB API,
здесь они следующие: ' " \ ; { } $
*/
// Избегайте регулярных выражений в этот раз для упрощения кода проверки
// и повышения читаемости...
ArrayList<String> specialCharsList = new ArrayList<String>() {
    {
        add("'");
        add("\"");
        add("\\");
        add(";");
        add("{");
        add("}");
        add("$");
    }
};

for (String specChar : specialCharsList) {
    if (userInput.contains(specChar)) {
        return false;
    }
}

// Также добавьте проверку максимального размера входных данных
if (userInput.length() > 50) {
    return false;
}

/* Затем выполните запрос к базе данных, используя API для построения выражения */
// Подключаемся к локальному экземпляру MongoDB
try (MongoClient mongoClient = new MongoClient()) {
    MongoDatabase db = mongoClient.getDatabase("test");
    // Используем API конструктора запросов для создания выражения
    // Создаем выражение
    Bson expression = eq("borough", userInput);
    // Выполняем запрос
    FindIterable<org.bson.Document> restaurants = db.getCollection("restaurants").find(expression);
    // Проверяем согласованность результата
    restaurants.forEach(new Block<org.bson.Document>() {
        @Override
        public void apply(final org.bson.Document doc) {
            String restBorough = (String) doc.get("borough");
            if (!"Brooklyn".equals(restBorough)) {
                return false;
            }
        }
    });
}
```

#### Ссылки

- [Тестирование на инъекции NoSQL](https://owasp.org/www-project-web-security-testing-guide/stable/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05.6-Testing_for_NoSQL_Injection.html)
- [SQL и NoSQL инъекции](https://ckarande.gitbooks.io/owasp-nodegoat-tutorial/content/tutorial/a1_-_sql_and_nosql_injection.html)
- [Нет SQL, нет инъекции?](https://arxiv.org/ftp/arxiv/papers/1506/1506.04082.pdf)

### Log Injection

#### Симптом

[Инъекция в журнал (Log Injection)](https://owasp.org/www-community/attacks/Log_Injection) происходит, когда приложение включает непроверенные данные в сообщение журнала приложения (например, злоумышленник может создать дополнительную запись в журнале, которая будет выглядеть так, как будто она пришла от совершенно другого пользователя, если он сможет внедрить символы CRLF в непроверенные данные). Дополнительную информацию об этой атаке можно найти на странице OWASP [Log Injection](https://owasp.org/www-community/attacks/Log_Injection).

#### Как предотвратить

Чтобы предотвратить запись вредоносного содержимого в журнал приложения, применяйте следующие меры защиты:

- Фильтруйте входные данные пользователя, чтобы предотвратить внедрение символов **C**arriage **R**eturn (CR) или **L**ine **F**eed (LF).
- Ограничьте размер значения входных данных пользователя, используемого для создания сообщения журнала.
- Убедитесь, что применяются [все меры защиты от XSS](Cross_Site_Scripting_Prevention_Cheat_Sheet.md) при просмотре файлов журналов в веб-браузере.

#### Пример с использованием Log4j2

Конфигурация политики ведения журнала для создания 10 файлов по 5 МБ каждый и кодирования/ограничения сообщения журнала с использованием [Pattern *encode{}{CRLF}*](https://logging.apache.org/log4j/2.x/manual/layouts.html#PatternLayout%7CLog4j2), введенного в [Log4j2 v2.10.0](https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-api), и ограничения размера сообщения *-500m*:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="error" name="SecureLoggingPolicy">
    <Appenders>
        <RollingFile name="RollingFile" fileName="App.log" filePattern="App-%i.log" ignoreExceptions="false">
            <PatternLayout>
                <!-- Кодируем любые символы CRLF в сообщении и ограничиваем его
                     максимальный размер до 500 символов -->
                <Pattern>%d{ISO8601} %-5p - %encode{ %.-500m }{CRLF}%n</Pattern>
            </PatternLayout>
            <Policies>
                <SizeBasedTriggeringPolicy size="5MB"/>
            </Policies>
            <DefaultRolloverStrategy max="10"/>
        </RollingFile>
    </Appenders>
    <Loggers>
        <Root level="debug">
            <AppenderRef ref="RollingFile"/>
        </Root>
    </Loggers>
</Configuration>
```

Использование логгера на уровне кода:

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
...
// Специальные действия не требуются, поскольку меры безопасности
// выполняются на уровне политики ведения журнала
Logger logger = LogManager.getLogger(MyClass.class);
logger.info(logMessage);
...
```

#### Пример с использованием Logback с библиотекой OWASP Security Logging

Конфигурация политики ведения журнала для создания 10 файлов по 5 МБ каждый и кодирования/ограничения сообщения журнала с использованием [CRLFConverter](https://github.com/augustd/owasp-security-logging/wiki/Log-Forging), предоставляемого **больше не активным** [OWASP Security Logging Project](https://github.com/augustd/owasp-security-logging/wiki), и ограничения размера сообщения *-500msg*:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- Определите CRLFConverter -->
    <conversionRule conversionWord="crlf" converterClass="org.owasp.security.logging.mask.CRLFConverter" />
    <appender name="RollingFile" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>App.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>App-%i.log</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>10</maxIndex>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>5MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <!-- Кодируем любые символы CRLF в сообщении и ограничиваем
                 его максимальный размер до 500 символов -->
            <pattern>%relative [%thread] %-5level %logger{35} - %crlf(%.-500msg) %n</pattern>
        </encoder>
    </appender>
    <root level="debug">
        <appender-ref ref="RollingFile" />
    </root>
</configuration>
```

Также необходимо добавить зависимость [OWASP Security Logging](https://github.com/augustd/owasp-security-logging/wiki/Usage-with-Logback) в ваш проект.

Использование логгера на уровне кода:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
...
// Специальные действия не требуются, поскольку меры безопасности
// выполняются на уровне политики ведения журнала
Logger logger = LoggerFactory.getLogger(MyClass.class);
logger.info(logMessage);
...
```

#### Ссылки

- [PatternLayout](https://logging.apache.org/log4j/2.x/manual/layouts.html#PatternLayout) (См. функцию `encode{}{CRLF}`)

```text
Обратите внимание, что стандартный кодировщик Log4j2 encode{} является HTML, что НЕ предотвращает инъекцию в журнал.

Он предотвращает атаки XSS при просмотре журналов через браузер.

OWASP рекомендует защищаться от атак XSS в таких ситуациях в самом приложении для просмотра журналов, а не предварительно кодировать все сообщения журнала HTML-кодированием, поскольку такие записи журнала могут использоваться/просматриваться во многих других инструментах для просмотра/анализа журналов, которые не ожидают, что данные журнала будут предварительно HTML-кодированы.
```

- [Конфигурация LOG4J](https://logging.apache.org/log4j/2.x/manual/configuration.html)
- [LOG4J Appender](https://logging.apache.org/log4j/2.x/manual/appenders.html)
- [Подделка журнала (Log Forging)](https://github.com/javabeanz/owasp-security-logging/wiki/Log-Forging) — См. раздел Logback о `CRLFConverter`, который предоставляет эта библиотека.
- [Использование OWASP Security Logging с Logback](https://github.com/javabeanz/owasp-security-logging/wiki/Usage-with-Logback)

## Криптография

### Общие рекомендации по криптографии

- **Никогда не пишите свои собственные криптографические функции.**
- По возможности старайтесь избегать написания любого криптографического кода. Вместо этого используйте существующие решения по управлению секретами или решение по управлению секретами, предоставляемое вашим облачным провайдером. Для получения дополнительной информации см. [Шпаргалке по управлению секретами](Secrets_Management_Cheat_Sheet.md).
- Если вы не можете использовать существующее решение по управлению секретами, попробуйте использовать проверенную и хорошо известную библиотеку реализации, а не библиотеки, встроенные в JCA/JCE, так как с ними слишком легко совершать криптографические ошибки.
- Убедитесь, что ваше приложение или протокол легко поддерживают будущие изменения криптографических алгоритмов.
- Используйте ваш менеджер пакетов, чтобы поддерживать все ваши пакеты в актуальном состоянии. Следите за обновлениями в вашей среде разработки и планируйте обновления ваших приложений соответственно.
- В приведенных ниже примерах будет использоваться Google Tink, библиотека, созданная экспертами в области криптографии для безопасного использования криптографии (в смысле минимизации общих ошибок при использовании стандартных криптографических библиотек).

### Шифрование для хранения

Следуйте рекомендациям по алгоритмам в [Шпаргалке по криптографическому хранению](Cryptographic_Storage_Cheat_Sheet.md#algorithms).

#### Пример симметричного шифрования с использованием Google Tink

Google Tink предоставляет документацию по выполнению общих задач.

Например, на этой странице (с сайта Google) показано, [как выполнить простое симметричное шифрование](https://developers.google.com/tink/encrypt-data).

Следующий фрагмент кода демонстрирует инкапсулированное использование этой функциональности:

<details>
  <summary>Нажмите здесь, чтобы просмотреть фрагмент кода "Tink симметричное шифрование".</summary>

```java
import static java.nio.charset.StandardCharsets.UTF_8;

import com.google.crypto.tink.Aead;
import com.google.crypto.tink.InsecureSecretKeyAccess;
import com.google.crypto.tink.KeysetHandle;
import com.google.crypto.tink.TinkJsonProtoKeysetFormat;
import com.google.crypto.tink.aead.AeadConfig;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Base64;

// AesGcmSimpleTest
public class App {

    // Исходный пример из:
    // https://github.com/tink-crypto/tink-java/tree/main/examples/aead

    public static void main(String[] args) throws Exception {

        // Ключ был сгенерирован с использованием:
        // tinkey create-keyset --key-template AES128_GCM --out-format JSON --out aead_test_keyset.json

        // Регистрация всех ключевых типов AEAD в среде выполнения Tink.
        AeadConfig.register();

        // Чтение набора ключей в KeysetHandle.
        KeysetHandle handle =
        TinkJsonProtoKeysetFormat.parseKeyset(
            new String(Files.readAllBytes(Paths.get("/home/fredbloggs/aead_test_keyset.json")), UTF_8), InsecureSecretKeyAccess.get());

        String message = "Это сообщение для шифрования";
        System.out.println(message);

        // Добавление некоторого контекста к зашифрованным данным, который следует проверить
        // при расшифровке
        String metadata = "Отправитель: fredbloggs@example.com";

        // Шифрование сообщения
        byte[] cipherText = AesGcmSimple.encrypt(message, metadata, handle);
        System.out.println(Base64.getEncoder().encodeToString(cipherText));

        // Расшифрование сообщения
        String message2 = AesGcmSimple.decrypt(cipherText, metadata, handle);
        System.out.println(message2);
    }
}

class AesGcmSimple {

    public static byte[] encrypt(String plaintext, String metadata, KeysetHandle handle) throws Exception {
        // Получение примитива.
        Aead aead = handle.getPrimitive(Aead.class);
        return aead.encrypt(plaintext.getBytes(UTF_8), metadata.getBytes(UTF_8));
    }

    public static String decrypt(byte[] ciphertext, String metadata, KeysetHandle handle) throws Exception {
        // Получение примитива.
        Aead aead = handle.getPrimitive(Aead.class);
        return new String(aead.decrypt(ciphertext, metadata.getBytes(UTF_8)), UTF_8);
    }
}
```

</details>

#### Пример симметричного шифрования с использованием встроенных классов JCA/JCE

Если вы абсолютно не можете использовать отдельную библиотеку, вы все равно можете использовать встроенные классы JCA/JCE, но настоятельно рекомендуется, чтобы криптографический эксперт проверил полный дизайн и код, так как даже самая тривиальная ошибка может серьезно ослабить ваше шифрование.

Следующий фрагмент кода демонстрирует использование AES-GCM для выполнения шифрования/расшифрования данных.

Некоторые ограничения/подводные камни этого кода:

- Он не учитывает управление или ротацию ключей, что является отдельной темой.
- Важно использовать уникальный nonce для каждой операции шифрования, особенно если используется один и тот же ключ. Дополнительную информацию можно найти [в этом ответе на Cryptography Stack Exchange](https://crypto.stackexchange.com/a/66500).
- Ключ нужно хранить надежно.

<details>
  <summary>Нажмите здесь, чтобы просмотреть фрагмент кода "JCA/JCE симметричное шифрование".</summary>

```java
import java.nio.charset.StandardCharsets;
import java.security.SecureRandom;
import javax.crypto.spec.*;
import javax.crypto.*;
import java.util.Base64;

// AesGcmSimpleTest
class Main {

    public static void main(String[] args) throws Exception {
        // Ключ из 32 байт / 256 бит для AES
        KeyGenerator keyGen = KeyGenerator.getInstance(AesGcmSimple.ALGORITHM);
        keyGen.init(AesGcmSimple.KEY_SIZE, new SecureRandom());
        SecretKey secretKey = keyGen.generateKey();

        // Nonce из 12 байт / 96 бит и этот размер всегда должен использоваться.
        // Критически важно для AES-GCM использовать уникальный nonce для каждой криптографической операции.
        byte[] nonce = new byte[AesGcmSimple.IV_LENGTH];
        SecureRandom random = new SecureRandom();
        random.nextBytes(nonce);

        var message = "Это сообщение для шифрования";
        System.out.println(message);

        // Шифрование сообщения
        byte[] cipherText = AesGcmSimple.encrypt(message, nonce, secretKey);
        System.out.println(Base64.getEncoder().encodeToString(cipherText));

        // Расшифрование сообщения
        var message2 = AesGcmSimple.decrypt(cipherText, nonce, secretKey);
        System.out.println(message2);
    }
}

class AesGcmSimple {

    public static final String ALGORITHM = "AES";
    public static final String CIPHER_ALGORITHM = "AES/GCM/NoPadding";
    public static final int KEY_SIZE = 256;
    public static final int TAG_LENGTH = 128;
    public static final int IV_LENGTH = 12;

    public static byte[] encrypt(String plaintext, byte[] nonce, SecretKey secretKey) throws Exception {
        return cryptoOperation(plaintext.getBytes(StandardCharsets.UTF_8), nonce, secretKey, Cipher.ENCRYPT_MODE);
    }

    public static String decrypt(byte[] ciphertext, byte[] nonce, SecretKey secretKey) throws Exception {
        return new String(cryptoOperation(ciphertext, nonce, secretKey, Cipher.DECRYPT_MODE), StandardCharsets.UTF_8);
    }

    private static byte[] cryptoOperation(byte[] text, byte[] nonce, SecretKey secretKey, int mode) throws Exception {
        Cipher cipher = Cipher.getInstance(CIPHER_ALGORITHM);
        GCMParameterSpec gcmParameterSpec = new GCMParameterSpec(TAG_LENGTH, nonce);
        cipher.init(mode, secretKey, gcmParameterSpec);
        return cipher.doFinal(text);
    }

}
```

</details>

#### Шифрование для передачи

Опять же, следуйте рекомендациям по алгоритмам, приведенным в [Шпаргалке по криптографическому хранилищу](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html#algorithms).

#### Пример ассиметричного шифрования с использованием Google Tink

Google Tink предоставляет документацию по выполнению распространенных задач.

Например, на этой странице (с сайта Google) показано, [как выполнить гибридный процесс шифрования](https://developers.google.com/tink/exchange-data), где две стороны хотят обмениваться данными на основе своих ассиметричных ключей.

Следующий фрагмент кода демонстрирует, как эту функциональность можно использовать для обмена секретами между Алисом и Бобом:

<details>
  <summary>Нажмите здесь, чтобы просмотреть фрагмент кода "Гибридное шифрование Tink".</summary>

```java
import static java.nio.charset.StandardCharsets.UTF_8;

import com.google.crypto.tink.HybridDecrypt;
import com.google.crypto.tink.HybridEncrypt;
import com.google.crypto.tink.InsecureSecretKeyAccess;
import com.google.crypto.tink.KeysetHandle;
import com.google.crypto.tink.TinkJsonProtoKeysetFormat;
import com.google.crypto.tink.hybrid.HybridConfig;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Base64;

// HybridReplaceTest
class App {
    public static void main(String[] args) throws Exception {
        /*

        Сгенерированы пары публичных/приватных ключей для Боба и Алисы с помощью следующих команд tinkey:

        ./tinkey create-keyset \
        --key-template DHKEM_X25519_HKDF_SHA256_HKDF_SHA256_AES_256_GCM \
        --out-format JSON --out alice_private_keyset.json

        ./tinkey create-keyset \
        --key-template DHKEM_X25519_HKDF_SHA256_HKDF_SHA256_AES_256_GCM \
        --out-format JSON --out bob_private_keyset.json

        ./tinkey create-public-keyset --in alice_private_keyset.json \
        --in-format JSON --out-format JSON --out alice_public_keyset.json

        ./tinkey create-public-keyset --in bob_private_keyset.json \
        --in-format JSON --out-format JSON --out bob_public_keyset.json
        */

        HybridConfig.register();

        // Генерация ECC ключевой пары для Алисы
        var alice = new HybridSimple(
                getKeysetHandle("/home/alicesmith/private_keyset.json"),
                getKeysetHandle("/home/alicesmith/public_keyset.json")

        );

        KeysetHandle alicePublicKey = alice.getPublicKey();

        // Генерация ECC ключевой пары для Боба
        var bob = new HybridSimple(
                getKeysetHandle("/home/bobjones/private_keyset.json"),
                getKeysetHandle("/home/bobjones/public_keyset.json")

        );

        KeysetHandle bobPublicKey = bob.getPublicKey();

        // Генерация ключевых пар должна выполняться периодически, чтобы
        // получить новый общий секрет и избежать длительного использования одного и того же секрета.

        // Алиса шифрует сообщение, чтобы отправить Бобу
        String plaintext = "Привет, Боб!";

        // Добавление контекста о зашифрованных данных, который должен быть проверен
        // при расшифровании
        String metadata = "Отправитель: alicesmith@example.com";

        System.out.println("Секрет, отправляемый от Алисы к Бобу: " + plaintext);
        var cipherText = alice.encrypt(bobPublicKey, plaintext, metadata);
        System.out.println("Шифротекст, отправляемый от Алисы к Бобу: " + Base64.getEncoder().encodeToString(cipherText));


        // Боб расшифровывает сообщение
        var decrypted = bob.decrypt(cipherText, metadata);
        System.out.println("Секрет, полученный Бобом от Алисы: " + decrypted);
        System.out.println();

        // Боб шифрует сообщение, чтобы отправить Алисе
        String plaintext2 = "Привет, Алиса!";

        // Добавление контекста о зашифрованных данных, который должен быть проверен
        // при расшифровании
        String metadata2 = "Отправитель: bobjones@example.com";

        System.out.println("Секрет, отправляемый от Боба к Алисе: " + plaintext2);
        var cipherText2 = bob.encrypt(alicePublicKey, plaintext2, metadata2);
        System.out.println("Шифротекст, отправляемый от Боба к Алисе: " + Base64.getEncoder().encodeToString(cipherText2));

        // Алиса расшифровывает сообщение
        var decrypted2 = alice.decrypt(cipherText2, metadata2);
        System.out.println("Секрет, полученный Алисой от Боба: " + decrypted2);
    }

    private static KeysetHandle getKeysetHandle(String filename) throws Exception
    {
        return TinkJsonProtoKeysetFormat.parseKeyset(
                new String(Files.readAllBytes( Paths.get(filename)), UTF_8), InsecureSecretKeyAccess.get());
    }
}
class HybridSimple {

    private KeysetHandle privateKey;
    private KeysetHandle publicKey;


    public HybridSimple(KeysetHandle privateKeyIn, KeysetHandle publicKeyIn) throws Exception {
        privateKey = privateKeyIn;
        publicKey = publicKeyIn;
    }

    public KeysetHandle getPublicKey() {
        return publicKey;
    }

    public byte[] encrypt(KeysetHandle partnerPublicKey, String message, String metadata) throws Exception {

        HybridEncrypt encryptor = partnerPublicKey.getPrimitive(HybridEncrypt.class);

        // возвращаем зашифрованное значение
        return encryptor.encrypt(message.getBytes(UTF_8), metadata.getBytes(UTF_8));
    }
    public String decrypt(byte[] ciphertext, String metadata) throws Exception {

        HybridDecrypt decryptor = privateKey.getPrimitive(HybridDecrypt.class);

        // возвращаем расшифрованное значение
        return new String(decryptor.decrypt(ciphertext, metadata.getBytes(UTF_8)),UTF_8);
    }


}
```

</details>

#### Пример ассиметричного шифрования с использованием встроенных классов JCA/JCE

Если вы не можете использовать отдельную библиотеку, можно использовать встроенные классы JCA/JCE, но настоятельно рекомендуется, чтобы криптографический эксперт проверил полный проект и код, так как даже незначительная ошибка может сильно ослабить ваше шифрование.

Следующий фрагмент кода демонстрирует использование Элиптической Кривой/Диффи-Хеллмана (ECDH) вместе с AES-GCM для выполнения шифрования/расшифрования данных между двумя сторонами без необходимости передавать симметричный ключ между сторонами. Вместо этого стороны обмениваются публичными ключами и могут использовать ECDH для генерации общего секрета, который затем можно использовать для симметричного шифрования.

Обратите внимание, что этот пример кода использует класс AesGcmSimple из [предыдущего раздела](#symmetric-example-using-built-in-jcajce-classes).

Некоторые ограничения и подводные камни этого кода:

- Не учитывается ротация или управление ключами, что является отдельной темой.
- Код намеренно использует новый nonce для каждой операции шифрования, но это должно управляться как отдельный элемент данных вместе с шифротекстом.
- Приватные ключи должны храниться надежно.
- Код не рассматривает проверку публичных ключей перед использованием.
- В целом, нет проверки подлинности между двумя сторонами.

<details>
  <summary>Нажмите здесь, чтобы просмотреть фрагмент кода "Гибридное шифрование JCA/JCE".</summary>

```java
import java.nio.charset.StandardCharsets;
import java.security.SecureRandom;
import javax.crypto.spec.*;
import javax.crypto.*;
import java.util.*;
import java.security.*;
import java.security.spec.*;
import java.util.Arrays;

// ECDHSimpleTest
class Main {
    public static void main(String[] args) throws Exception {

        // Генерация ECC ключевой пары для Алисы
        var alice = new ECDHSimple();
        Key alicePublicKey = alice.getPublicKey();

        // Генерация ECC ключевой пары для Боба
        var bob = new ECDHSimple();
        Key bobPublicKey = bob.getPublicKey();

        // Генерация ключевых пар должна выполняться периодически, чтобы
        // получить новый общий секрет и избежать длительного использования одного и того же секрета.

        // Алиса шифрует сообщение, чтобы отправить Бобу
        String plaintext = "Привет"; //, Боб!";
        System.out.println("Секрет, отправляемый от Алисы к Бобу: " + plaintext);

        var retPair = alice.encrypt(bobPublicKey, plaintext);
        var nonce = retPair.getKey();
        var cipherText = retPair.getValue();

        System.out.println("Оба значения cipherText и nonce, отправляемые от Алисы к Бобу: " + Base64.getEncoder().encodeToString(cipherText) + " " + Base64.getEncoder().encodeToString(nonce));


        // Боб расшифровывает сообщение
        var decrypted = bob.decrypt(alicePublicKey, cipherText, nonce);
        System.out.println("Секрет, полученный Бобом от Алисы: " + decrypted);
        System.out.println();

        // Боб шифрует сообщение, чтобы отправить Алисе
        String plaintext2 = "Привет"; //, Алиса!";
        System.out.println("Секрет, отправляемый от Боба к Алисе: " + plaintext2);

        var retPair2 = bob.encrypt(alicePublicKey, plaintext2);
        var nonce2 = retPair2.getKey();
        var cipherText2 = retPair2.getValue();
        System.out.println("Оба значения cipherText2 и nonce2, отправляемые от Боба к Алисе: " + Base64.getEncoder().encodeToString(cipherText2) + " " + Base64.getEncoder().encodeToString(nonce2));

        // Алиса расшифровывает сообщение
        var decrypted2 = alice.decrypt(bobPublicKey, cipherText2, nonce2);
        System.out.println("Секрет, полученный Алисой от Боба: " + decrypted2);
    }
}
class ECDHSimple {
    private KeyPair keyPair;

    public class AesKeyNonce {
        public SecretKey Key;
        public byte[] Nonce;
    }

    public ECDHSimple() throws Exception {
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("EC");
        ECGenParameterSpec ecSpec = new ECGenParameterSpec("secp256r1"); // Использование кривой secp256r1
        keyPairGenerator.initialize(ecSpec);
        keyPair = keyPairGenerator.generateKeyPair();
    }

    public Key getPublicKey() {
        return keyPair.getPublic();
    }

    public AbstractMap.SimpleEntry<byte[], byte[]> encrypt(Key partnerPublicKey, String message) throws Exception {

        // Генерация ключа AES и nonce
        AesKeyNonce aesParams = generateAESParams(partnerPublicKey);

        // возвращаем зашифрованное значение
        return new AbstractMap.SimpleEntry<>( 
            aesParams.Nonce,
            AesGcmSimple.encrypt(message, aesParams.Nonce, aesParams.Key)
            );
    }
    public String decrypt(Key partnerPublicKey, byte[] ciphertext, byte[] nonce) throws Exception {

        // Генерация ключа AES и nonce
        AesKeyNonce aesParams = generateAESParams(partnerPublicKey, nonce);

        // возвращаем расшифрованное значение
        return AesGcmSimple.decrypt(ciphertext, aesParams.Nonce, aesParams.Key);
    }

    private AesKeyNonce generateAESParams(Key partnerPublicKey, byte[] nonce) throws Exception {

        // Генерация секрета на основе приватного ключа этой стороны и публичного ключа другой стороны
        KeyAgreement keyAgreement = KeyAgreement.getInstance("ECDH");
        keyAgreement.init(keyPair.getPrivate());
        keyAgreement.doPhase(partnerPublicKey, true);
        byte[] secret = keyAgreement.generateSecret();

        AesKeyNonce aesKeyNonce = new AesKeyNonce();

        // Копирование первых 32 байт в качестве ключа
        byte[] key = Arrays.copyOfRange(secret, 0, (AesGcmSimple.KEY_SIZE / 8));
        aesKeyNonce.Key = new SecretKeySpec(key, 0, key.length, "AES");

        // Переданный nonce будет использоваться.
        aesKeyNonce.Nonce = nonce;
        return aesKeyNonce;

    }

    private AesKeyNonce generateAESParams(Key partnerPublicKey) throws Exception {

        // Nonce размером 12 байт / 96 бит и этот размер всегда должен использоваться.
        // Важно для AES-GCM, чтобы уникальный nonce использовался для каждой криптографической операции.
        // Поэтому он не генерируется из общего секрета
        byte[] nonce = new byte[AesGcmSimple.IV_LENGTH];
        SecureRandom random = new SecureRandom();
        random.nextBytes(nonce);
        return generateAESParams(partnerPublicKey, nonce);

    }
}
```

</details>
