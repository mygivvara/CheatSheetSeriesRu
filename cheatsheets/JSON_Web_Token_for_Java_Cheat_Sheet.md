# Шпаргалка по JSON Web Tokens для Java

## Введение

Многие приложения используют **JSON Web Tokens** (JWT), чтобы клиент мог указать свою личность для дальнейшего обмена после аутентификации.

Из [JWT.IO](https://jwt.io/introduction):

> JSON Web Token (JWT) — это открытый стандарт (RFC 7519), который определяет компактный и автономный способ безопасной передачи информации между сторонами в виде JSON-объекта. Эта информация может быть проверена и доверена, поскольку она цифровым образом подписана. JWT могут быть подписаны с использованием секрета (с помощью алгоритма HMAC) или пары открытого/закрытого ключа с использованием RSA.

JSON Web Token используется для передачи информации, связанной с личностью и характеристиками (утверждениями) клиента. Эта информация подписывается сервером, чтобы он мог определить, была ли она изменена после отправки клиенту. Это предотвратит изменение личности или каких-либо характеристик (например, изменение роли с простого пользователя на администратора или изменение логина клиента) злоумышленником.

Этот токен создается во время аутентификации (предоставляется в случае успешной аутентификации) и проверяется сервером до любого процесса. Он используется приложением, чтобы клиент мог представить токен, представляющий "удостоверение личности" пользователя серверу, и позволить серверу проверить действительность и целостность токена безопасным способом, всё это в безд状态ном и переносимом подходе (переносимом в том смысле, что технологии клиента и сервера могут быть различными, включая также транспортный канал, даже если HTTP используется наиболее часто).

## Структура токена

Пример структуры токена взят из [JWT.IO](https://jwt.io/#debugger):

`[Base64(HEADER)].[Base64(PAYLOAD)].[Base64(SIGNATURE)]`

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.
TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

Часть 1: **Заголовок**

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Часть 2: **Полезная нагрузка**

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

Часть 3: **Подпись**

```javascript
HMACSHA256( base64UrlEncode(header) + "." + base64UrlEncode(payload), KEY )
```

## Цель

Эта шпаргалка предоставляет советы по предотвращению распространенных проблем безопасности при использовании JSON Web Tokens (JWT) с Java.

Советы, представленные в этой статье, являются частью проекта на Java, созданного для демонстрации правильного подхода к созданию и проверке JSON Web Tokens.

Вы можете найти проект на Java [здесь](https://github.com/righettod/poc-jwt), он использует официальную [JWT библиотеку](https://jwt.io/#libraries).

В остальной части статьи термин **токен** относится к **JSON Web Tokens** (JWT).

## Соображения при использовании JWT

Хотя JWT "легко" использовать и они позволяют экспонировать сервисы (в основном в стиле REST) в stateless режиме, это не решение, которое подходит для всех приложений, поскольку оно имеет некоторые подводные камни, такие как, например, вопрос хранения токена (который рассматривается в этой шпаргалке) и другие...

Если вашему приложению не требуется быть полностью stateless, вы можете рассмотреть использование традиционной системы сессий, предоставляемой всеми веб-фреймворками, и следовать советам из специальной [шпаргалки по управлению сессиями](Session_Management_Cheat_Sheet.md). Однако для stateless приложений, при правильной реализации, это хороший кандидат.

## Проблемы

### Отсутствие алгоритма хэширования

#### Симптом

Эта атака, описанная [здесь](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/), происходит, когда злоумышленник изменяет токен и меняет алгоритм хэширования на указание, через ключевое слово *none*, что целостность токена уже была проверена. Как объясняется в указанной ссылке, *некоторые библиотеки рассматривали токены, подписанные с использованием алгоритма none, как действительные токены с проверенной подписью*, поэтому злоумышленник может изменить утверждения токена, и модифицированный токен будет по-прежнему доверен приложением.

#### Как предотвратить

Во-первых, используйте библиотеку JWT, которая не подвержена этой уязвимости.

Во-вторых, при проверке токена явно требуйте, чтобы использовался ожидаемый алгоритм.

#### Пример реализации

```java
// Ключ HMAC - Блокирование сериализации и хранения в виде строки в памяти JVM
private transient byte[] keyHMAC = ...;

...

// Создайте контекст проверки для токена, требуя
// явно использование алгоритма хэширования HMAC-256
JWTVerifier verifier = JWT.require(Algorithm.HMAC256(keyHMAC)).build();

// Проверьте токен, если проверка не удалась, будет выброшено исключение
DecodedJWT decodedToken = verifier.verify(token);
```

### Перехват токенов

#### Симптом

Эта атака происходит, когда злоумышленник перехватывает/ворует токен и использует его для получения доступа к системе, используя личность целевого пользователя.

#### Как предотвратить

Один из способов предотвращения — добавить "контекст пользователя" в токен. Контекст пользователя будет состоять из следующей информации:

- Случайная строка, которая будет сгенерирована во время фазы аутентификации. Она будет отправлена клиенту в виде защищенного cookie (флаги: [HttpOnly + Secure](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#Secure_and_HttpOnly_cookies) + [SameSite](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies#SameSite_cookies) + [Max-Age](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie) + [cookie prefixes](https://googlechrome.github.io/samples/cookie-prefixes/)). Избегайте установки заголовка *expires*, чтобы cookie очищался при закрытии браузера. Установите *Max-Age* на значение меньшее или равное значению срока действия JWT, но никогда не больше.
- SHA256-хэш случайной строки будет храниться в токене (вместо ее необработанного значения), чтобы предотвратить любые XSS-уязвимости, позволяющие злоумышленнику прочитать значение случайной строки и установить ожидаемое cookie.

IP-адреса не следует использовать, поскольку могут возникнуть легитимные ситуации, в которых IP-адрес может измениться в течение одной сессии. Например, когда пользователь получает доступ к приложению через мобильное устройство, и оператор мобильной связи меняется в процессе обмена, IP-адрес может (часто) измениться. Более того, использование IP-адреса может потенциально вызвать проблемы с соблюдением [GDPR](https://gdpr.eu/).

Во время проверки токена, если полученный токен не содержит правильного контекста (например, если он был повторно использован), он должен быть отклонен.

#### Пример реализации

Код для создания токена после успешной аутентификации.

```java
// Ключ HMAC - Блокирование сериализации и хранения в виде строки в памяти JVM
private transient byte[] keyHMAC = ...;
// Генератор случайных данных
private SecureRandom secureRandom = new SecureRandom();

...

// Сгенерируйте случайную строку, которая будет являться отпечатком для этого пользователя
byte[] randomFgp = new byte[50];
secureRandom.nextBytes(randomFgp);
String userFingerprint = DatatypeConverter.printHexBinary(randomFgp);

// Добавьте отпечаток в защищенное cookie - Добавьте cookie вручную, так как
// атрибут SameSite не поддерживается классом javax.servlet.http.Cookie
String fingerprintCookie = "__Secure-Fgp=" + userFingerprint
                           + "; SameSite=Strict; HttpOnly; Secure";
response.addHeader("Set-Cookie", fingerprintCookie);

// Вычислите SHA256-хэш отпечатка, чтобы сохранить
// хэш отпечатка (вместо необработанного значения) в токене
// для предотвращения XSS-уязвимостей, позволяющих
// злоумышленнику прочитать отпечаток и установить ожидаемое cookie
MessageDigest digest = MessageDigest.getInstance("SHA-256");
byte[] userFingerprintDigest = digest.digest(userFingerprint.getBytes("utf-8"));
String userFingerprintHash = DatatypeConverter.printHexBinary(userFingerprintDigest);

// Создайте токен с действительностью 15 минут и информацией о контексте клиента (отпечатке)
Calendar c = Calendar.getInstance();
Date now = c.getTime();
c.add(Calendar.MINUTE, 15);
Date expirationDate = c.getTime();
Map<String, Object> headerClaims = new HashMap<>();
headerClaims.put("typ", "JWT");
String token = JWT.create().withSubject(login)
   .withExpiresAt(expirationDate)
   .withIssuer(this.issuerID)
   .withIssuedAt(now)
   .withNotBefore(now)
   .withClaim("userFingerprint", userFingerprintHash)
   .withHeader(headerClaims)
   .sign(Algorithm.HMAC256(this.keyHMAC));
```

Код для проверки токена.

```java
// Ключ HMAC - Блокирование сериализации и хранения в виде строки в памяти JVM
private transient byte[] keyHMAC = ...;

...

// Получите отпечаток пользователя из специального cookie
String userFingerprint = null;
if (request.getCookies() != null && request.getCookies().length > 0) {
 List<Cookie> cookies = Arrays.stream(request.getCookies()).collect(Collectors.toList());
 Optional<Cookie> cookie = cookies.stream().filter(c -> "__Secure-Fgp"
                                            .equals(c.getName())).findFirst();
 if (cookie.isPresent()) {
   userFingerprint = cookie.get().getValue();
 }
}

// Вычислите SHA256-хэш полученного отпечатка в cookie для сравнения
// с хэшем отпечатка, хранящимся в токене
MessageDigest digest = MessageDigest.getInstance("SHA-256");
byte[] userFingerprintDigest = digest.digest(userFingerprint.getBytes("utf-8"));
String userFingerprintHash = DatatypeConverter.printHexBinary(userFingerprintDigest);

// Создайте контекст проверки для токена
JWTVerifier verifier = JWT.require(Algorithm.HMAC256(keyHMAC))
                              .withIssuer(issuerID)
                              .withClaim("userFingerprint", userFingerprintHash)
                              .build();

// Проверьте токен, если проверка не удалась, будет выброшено исключение
DecodedJWT decodedToken = verifier.verify(token);
```

### Отсутствие встроенной функции отзыва токенов пользователем

#### Симптом

Эта проблема присуща JWT, поскольку токен становится недействительным только по истечении срока его действия. У пользователя нет встроенной функции для явного отзыва действительности токена. Это означает, что если токен украден, пользователь не может сам отозвать токен, тем самым блокируя злоумышленника.

#### Как предотвратить

Поскольку JWT является stateless, на сервере не поддерживается сессия для обработки запросов клиента. Таким образом, нет сессии, которую нужно инвалидировать на серверной стороне. Хорошо реализованное решение для предотвращения перехвата токенов (как объяснено выше) должно устранить необходимость поддерживать denylist на стороне сервера. Это связано с тем, что защищенное cookie, используемое в решении для предотвращения перехвата токенов, можно считать столь же безопасным, как идентификатор сессии, используемый в традиционной системе сессий, и если оба, и cookie, и JWT, не будут перехвачены/украдены, JWT будет бесполезен. Выход из системы может быть "смоделирован" путем очистки JWT из хранилища сессий. Если пользователь решит закрыть браузер, то как cookie, так и sessionStorage будут автоматически очищены.

Другой способ защиты — реализовать denylist токенов, которая будет использоваться для имитации функции "выхода" из системы, существующей в традиционной системе управления сессиями.

Denylist будет хранить дайджест (SHA-256, закодированный в HEX) токена с датой отзыва. Эта запись должна существовать как минимум до истечения срока действия токена.

Когда пользователь хочет "выйти", он вызывает специальный сервис, который добавляет предоставленный токен пользователя в denylist, что приводит к немедленной инвалидизации токена для дальнейшего использования в приложении.

#### Пример реализации

##### Хранение черного списка

Для хранения denylist будет использоваться таблица базы данных со следующей структурой.

```sql
create table if not exists revoked_token(
    jwt_token_digest varchar(255) primary key,
    revocation_date timestamp default now()
);
```

##### Управление отзывами токенов

Код, отвечающий за добавление токена в denylist и проверку, отозван ли токен.

```java
/**
 * Обработка отзыва токена (выход из системы).
 * Используйте базу данных, чтобы несколько экземпляров могли проверять отозванные токены
 * и позволять очистку на уровне централизованной базы данных.
 */
public class TokenRevoker {

 /** Соединение с БД */
 @Resource("jdbc/storeDS")
 private DataSource storeDS;

 /**
  * Проверяет, присутствует ли дайджест токена, закодированный в HEX, в таблице отзыва
  *
  * @param jwtInHex Токен, закодированный в HEX
  * @return Флаг присутствия
  * @throws Exception Если возникнут проблемы при общении с БД
  */
 public boolean isTokenRevoked(String jwtInHex) throws Exception {
     boolean tokenIsPresent = false;
     if (jwtInHex != null && !jwtInHex.trim().isEmpty()) {
         // Декодирование зашифрованного токена
         byte[] cipheredToken = DatatypeConverter.parseHexBinary(jwtInHex);

         // Вычисление SHA256 хэша зашифрованного токена
         MessageDigest digest = MessageDigest.getInstance("SHA-256");
         byte[] cipheredTokenDigest = digest.digest(cipheredToken);
         String jwtTokenDigestInHex = DatatypeConverter.printHexBinary(cipheredTokenDigest);

         // Поиск дайджеста токена в БД
         try (Connection con = this.storeDS.getConnection()) {
             String query = "select jwt_token_digest from revoked_token where jwt_token_digest = ?";
             try (PreparedStatement pStatement = con.prepareStatement(query)) {
                 pStatement.setString(1, jwtTokenDigestInHex);
                 try (ResultSet rSet = pStatement.executeQuery()) {
                     tokenIsPresent = rSet.next();
                 }
             }
         }
     }

     return tokenIsPresent;
 }


 /**
  * Добавляет дайджест, закодированный в HEX, зашифрованного токена в таблицу отозванных токенов
  *
  * @param jwtInHex Токен, закодированный в HEX
  * @throws Exception Если возникнут проблемы при общении с БД
  */
 public void revokeToken(String jwtInHex) throws Exception {
     if (jwtInHex != null && !jwtInHex.trim().isEmpty()) {
         // Декодирование зашифрованного токена
         byte[] cipheredToken = DatatypeConverter.parseHexBinary(jwtInHex);

         // Вычисление SHA256 хэша зашифрованного токена
         MessageDigest digest = MessageDigest.getInstance("SHA-256");
         byte[] cipheredTokenDigest = digest.digest(cipheredToken);
         String jwtTokenDigestInHex = DatatypeConverter.printHexBinary(cipheredTokenDigest);

         // Проверка, присутствует ли дайджест токена в HEX уже в БД, и добавление его, если он отсутствует
         if (!this.isTokenRevoked(jwtInHex)) {
             try (Connection con = this.storeDS.getConnection()) {
                 String query = "insert into revoked_token(jwt_token_digest) values(?)";
                 int insertedRecordCount;
                 try (PreparedStatement pStatement = con.prepareStatement(query)) {
                     pStatement.setString(1, jwtTokenDigestInHex);
                     insertedRecordCount = pStatement.executeUpdate();
                 }
                 if (insertedRecordCount != 1) {
                     throw new IllegalStateException("Количество вставленных записей неверно, ожидалось 1, но есть " + insertedRecordCount);
                 }
             }
         }

     }
 }
```

### Разглашение информации о токене

#### Симптом

Эта атака происходит, когда злоумышленник имеет доступ к токену (или набору токенов) и извлекает из него информацию (содержимое JWT закодировано в base64, но по умолчанию не зашифровано), чтобы получить данные о системе. Информация может включать, например, роли безопасности, формат логина и т.д.

#### Как предотвратить

Один из способов защиты от этой атаки — зашифровать токен, используя, например, симметричный алгоритм.

Также важно защитить зашифрованные данные от атак, таких как [Padding Oracle](https://owasp.org/www-project-web-security-testing-guide/stable/4-Web_Application_Security_Testing/09-Testing_for_Weak_Cryptography/02-Testing_for_Padding_Oracle.html) или других атак с использованием криптоанализа.

Для достижения всех этих целей используется алгоритм *AES-[GCM](https://en.wikipedia.org/wiki/Galois/Counter_Mode)*, который обеспечивает *Authenticated Encryption with Associated Data* (Аутентифицированное шифрование с ассоциированными данными).

Подробнее см. [здесь](https://github.com/google/tink/blob/master/docs/PRIMITIVES.md#deterministic-authenticated-encryption-with-associated-data):

```text
AEAD primitive (Authenticated Encryption with Associated Data) предоставляет функциональность симметричного
аутентифицированного шифрования.

Реализации этого примитива защищены от атак с адаптивным выбором шифротекста.

При шифровании открытого текста можно опционально предоставить ассоциированные данные, которые должны быть аутентифицированы,
но не зашифрованы.

То есть шифрование с ассоциированными данными обеспечивает подлинность (то есть, кто отправитель) и
целостность (то есть, данные не были изменены) этих данных, но не их конфиденциальность.

См. RFC5116: https://tools.ietf.org/html/rfc5116
```

**Примечание:**

Зашифровка добавлена в основном для скрытия внутренней информации, но очень важно помнить, что первой защитой от подделки JWT является подпись. Поэтому подпись токена и её проверка должны всегда быть на месте.

#### Пример реализации

##### Шифрование токена

Код, отвечающий за управление шифрованием и расшифрованием токена. Используется криптографическая библиотека [Google Tink](https://github.com/google/tink) для обработки операций шифрования, чтобы воспользоваться встроенными лучшими практиками, предоставляемыми этой библиотекой.

``` java
/**
 * Обработка шифрования и расшифрования токена с использованием AES-GCM.
 *
 * @see "https://github.com/google/tink/blob/master/docs/JAVA-HOWTO.md"
 */
public class TokenCipher {

    /**
     * Конструктор - Регистрация конфигурации AEAD
     *
     * @throws Exception Если возникают проблемы при регистрации конфигурации AEAD
     */
    public TokenCipher() throws Exception {
        AeadConfig.register();
    }

    /**
     * Шифрование JWT
     *
     * @param jwt          Токен для шифрования
     * @param keysetHandle Указатель на ключевой набор
     * @return Зашифрованная версия токена в кодировке HEX
     * @throws Exception Если возникают проблемы при операции шифрования токена
     */
    public String cipherToken(String jwt, KeysetHandle keysetHandle) throws Exception {
        //Проверка параметров
        if (jwt == null || jwt.isEmpty() || keysetHandle == null) {
            throw new IllegalArgumentException("Оба параметра должны быть указаны!");
        }

        //Получение примитива
        Aead aead = AeadFactory.getPrimitive(keysetHandle);

        //Шифрование токена
        byte[] cipheredToken = aead.encrypt(jwt.getBytes(), null);

        return DatatypeConverter.printHexBinary(cipheredToken);
    }

    /**
     * Расшифрование JWT
     *
     * @param jwtInHex     Токен для расшифрования в кодировке HEX
     * @param keysetHandle Указатель на ключевой набор
     * @return Токен в открытом виде
     * @throws Exception Если возникают проблемы при операции расшифрования токена
     */
    public String decipherToken(String jwtInHex, KeysetHandle keysetHandle) throws Exception {
        //Проверка параметров
        if (jwtInHex == null || jwtInHex.isEmpty() || keysetHandle == null) {
            throw new IllegalArgumentException("Оба параметра должны быть указаны!");
        }

        //Декодирование зашифрованного токена
        byte[] cipheredToken = DatatypeConverter.parseHexBinary(jwtInHex);

        //Получение примитива
        Aead aead = AeadFactory.getPrimitive(keysetHandle);

        //Расшифрование токена
        byte[] decipheredToken = aead.decrypt(cipheredToken, null);

        return new String(decipheredToken);
    }
}
```

##### Создание / Проверка токена

Используйте обработчик шифрования токена при создании и проверке токена.

Загрузите ключи (ключ для шифрования был сгенерирован и сохранен с использованием [Google Tink](https://github.com/google/tink/blob/master/docs/JAVA-HOWTO.md#generating-new-keysets)) и настройте шифрование.

``` java
//Загрузка ключей из конфигурационных текстовых/json файлов, чтобы избежать хранения ключей как строки в памяти JVM
private transient byte[] keyHMAC = Files.readAllBytes(Paths.get("src", "main", "conf", "key-hmac.txt"));
private transient KeysetHandle keyCiphering = CleartextKeysetHandle.read(JsonKeysetReader.withFile(
Paths.get("src", "main", "conf", "key-ciphering.json").toFile()));

...

//Инициализация обработчика шифрования токена
TokenCipher tokenCipher = new TokenCipher();
```

Создание токена.

``` java
//Создание JWT токена с использованием JWT API...
//Шифрование токена (строковое представление JSON)
String cipheredToken = tokenCipher.cipherToken(token, this.keyCiphering);
//Отправка зашифрованного токена в кодировке HEX клиенту в HTTP ответе...
```

Проверка токена.

``` java
//Получение зашифрованного токена в кодировке HEX из HTTP запроса...
//Расшифрование токена
String token = tokenCipher.decipherToken(cipheredToken, this.keyCiphering);
//Проверка токена с использованием JWT API...
//Проверка доступа...
```

### Хранение токена на стороне клиента

#### Симптом

Это происходит, когда приложение хранит токен следующим образом:

- Автоматически отправляемый браузером (хранение в *Cookie*).
- Доступен даже после перезапуска браузера (использование контейнера браузера *localStorage*).
- Доступен в случае атаки [XSS](Cross_Site_Scripting_Prevention_Cheat_Sheet.md) (Cookie доступен для кода JavaScript или токен хранится в локальном/сессионном хранилище браузера).

#### Как предотвратить

1. Храните токен в контейнере *sessionStorage* браузера или используйте JavaScript *closures* с *private* переменными.
2. Добавляйте токен как *Bearer* HTTP `Authentication` заголовок при вызове сервисов.
3. Добавьте информацию о [отпечатке](JSON_Web_Token_for_Java_Cheat_Sheet.md#token-sidejacking) в токен.

Хранение токена в контейнере *sessionStorage* браузера подвержено риску кражи через атаку XSS. Однако, добавление отпечатков в токен предотвращает повторное использование украденного токена злоумышленником на их машине. Чтобы закрыть максимальное количество точек эксплойта для злоумышленника, добавьте в браузер [Политику Безопасности Контента (CSP)](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html), чтобы усилить контекст выполнения.

Альтернативой хранению токена в *sessionStorage* браузера является использование частных переменных JavaScript или замыканий. В этом случае доступ ко всем веб-запросам осуществляется через JavaScript-модуль, который инкапсулирует токен в частной переменной, доступной только изнутри модуля.

*Примечание:*

- Оставшийся случай — это когда злоумышленник использует контекст браузера пользователя как прокси для использования целевого приложения через законного пользователя, но Политика Безопасности Контента может предотвратить связь с не ожидаемыми доменами.
- Также возможно реализовать службу аутентификации таким образом, что токен выдается в защищенном cookie, но в этом случае необходимо реализовать защиту от атак [CSRF](Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.md).

### Пример реализации

JavaScript код для хранения токена после аутентификации.

``` javascript
/* Обработка запроса для получения JWT токена и хранение в sessionStorage */
function authenticate() {
    const login = $("#login").val();
    const postData = "login=" + encodeURIComponent(login) + "&password=test";

    $.post("/services/authenticate", postData, function (data) {
        if (data.status == "Authentication successful!") {
            ...
            sessionStorage.setItem("token", data.token);
        }
        else {
            ...
            sessionStorage.removeItem("token");
        }
    })
    .fail(function (jqXHR, textStatus, error) {
        ...
        sessionStorage.removeItem("token");
    });
}
```

JavaScript код для добавления токена в HTTP заголовок *Bearer* при вызове сервиса, например, для проверки токена.

``` javascript
/* Обработка запроса для проверки JWT токена */
function validateToken() {
    var token = sessionStorage.getItem("token");

    if (token == undefined || token == "") {
        $("#infoZone").removeClass();
        $("#infoZone").addClass("alert alert-warning");
        $("#infoZone").text("Obtain a JWT token first :)");
        return;
    }

    $.ajax({
        url: "/services/validate",
        type: "POST",
        beforeSend: function (xhr) {
            xhr.setRequestHeader("Authorization", "bearer " + token);
        },
        success: function (data) {
            ...
        },
        error: function (jqXHR, textStatus, error) {
            ...
        },
    });
}
```

JavaScript код для реализации замыканий с частными переменными:

``` javascript
function myFetchModule() {
    // Защита оригинального 'fetch' от перезаписи через XSS
    const fetch = window.fetch;

    const authOrigins = ["https://yourorigin", "http://localhost"];
    let token = '';

    this.setToken = (value) => {
        token = value;
    }

    this.fetch = (resource, options) => {
        let req = new Request(resource, options);
        const destOrigin = new URL(req.url).origin;
        if (token && authOrigins.includes(destOrigin)) {
            req.headers.set('Authorization', token);
        }
        return fetch(req);
    }
}

... 

// использование:
const myFetch = new myFetchModule();

function login() {
    fetch("/api/login")
        .then((res) => {
            if (res.status == 200) {
                return res.json();
            } else {
                throw Error(res.statusText);
            }
        })
        .then(data => {
            myFetch.setToken(data.token);
            console.log("Token received and stored.");
        })
        .catch(console.error);
}

... 

// после аутентификации, последующие вызовы API:
function makeRequest() {
    myFetch.fetch("/api/hello", {headers: {"MyHeader": "foobar"}})
        .then((res) => {
            if (res.status == 200) {
                return res.text();
            } else {
                throw Error(res.statusText);
            }
        })
        .then(responseText => console.log("helloResponse", responseText))
        .catch(console.error);
}
```

### Слабый секрет токена

#### Симптом

Когда токен защищен с использованием алгоритма на основе HMAC, безопасность токена полностью зависит от прочности секрета, используемого с HMAC. Если злоумышленник сможет получить действительный JWT, он может провести оффлайн-атаку и попытаться взломать секрет с помощью инструментов, таких как [John the Ripper](https://github.com/magnumripper/JohnTheRipper) или [Hashcat](https://github.com/hashcat/hashcat).

Если им удастся, они смогут модифицировать токен и подписать его с использованием полученного ключа. Это может позволить им эскалацию привилегий, компрометацию учетных записей других пользователей или выполнение других действий в зависимости от содержимого JWT.

Существует множество [руководств](https://www.notsosecure.com/crafting-way-json-web-tokens/), которые детализируют этот процесс.

#### Как предотвратить

Самый простой способ предотвратить эту атаку — убедиться, что секрет, используемый для подписи JWT, является сильным и уникальным, чтобы усложнить его взлом. Поскольку этот секрет никогда не потребуется для ввода человеком, он должен быть как минимум 64 символов и генерироваться с использованием [безопасного источника случайности](Cryptographic_Storage_Cheat_Sheet.md#secure-random-number-generation).

Альтернативно, рассмотрите использование токенов, которые подписаны с помощью RSA вместо использования HMAC и секретного ключа.

#### Дополнительное чтение

- [{JWT}.{Attack}.Playbook](https://github.com/ticarpi/jwt_tool/wiki) - Проект, который документирует известные атаки и потенциальные уязвимости и ошибки конфигурации JSON Web Tokens.
- [JWT Best Practices Internet Draft](https://datatracker.ietf.org/doc/draft-ietf-oauth-jwt-bcp/)
