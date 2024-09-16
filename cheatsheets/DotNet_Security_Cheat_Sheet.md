# Шпаргалка по безопасности DotNet

## Введение

Эта страница предоставляет быстрые базовые советы по безопасности .NET для разработчиков.

### .NET Framework

.NET Framework — это основная платформа Microsoft для разработки корпоративных приложений. Это API, поддерживающая ASP.NET, настольные приложения Windows, службы Windows Communication Foundation, SharePoint, Visual Studio Tools for Office и другие технологии.

.NET Framework представляет собой набор API, который облегчает использование расширенной системы типов, управление данными, графикой, сетями, файловыми операциями и другими задачами, охватывая практически все требования для разработки корпоративных приложений в экосистеме Microsoft. Это почти повсеместно используемая библиотека, которая имеет сильные имена и версионируется на уровне сборки.

### Обновление платформы

.NET Framework поддерживается в актуальном состоянии Microsoft через службу обновления Windows. Разработчикам обычно не нужно запускать отдельные обновления для платформы. Службу обновления Windows можно открыть на сайте [Windows Update](http://windowsupdate.microsoft.com/) или через программу обновления Windows на компьютере.

Отдельные фреймворки можно обновлять с помощью [NuGet](https://docs.microsoft.com/en-us/nuget/). Когда Visual Studio предлагает обновления, включайте их в свой цикл разработки.

Помните, что сторонние библиотеки необходимо обновлять отдельно, и не все они используют NuGet. Например, ELMAH требует отдельного обновления.

### Сообщения о безопасности

Получайте уведомления о безопасности, нажав кнопку «Watch» в следующих репозиториях:

- [Объявления о безопасности .NET Core](https://github.com/dotnet/announcements/issues?q=is%3Aopen+is%3Aissue+label%3ASecurity)
- [Объявления о безопасности ASP.NET Core и Entity Framework Core](https://github.com/aspnet/Announcements/issues?q=is%3Aopen+is%3Aissue+label%3ASecurity)

## Общие рекомендации по .NET

Этот раздел содержит общие рекомендации для .NET приложений.
Это относится ко всем .NET приложениям, включая ASP.NET, WPF, WinForms и другие.

OWASP Top 10 представляет самые распространенные и опасные угрозы безопасности веб-приложений в мире на сегодняшний день и пересматривается каждые несколько лет с обновлением на основе последних данных о угрозах. Этот раздел шпаргалки основан на этом списке. Ваш подход к обеспечению безопасности веб-приложения должен начинаться с верхней угрозы A1 и продвигаться вниз; это обеспечит наиболее эффективное использование времени на обеспечение безопасности, начиная с самых серьезных угроз и затем с менее значительных. После покрытия Top 10 обычно рекомендуется оценить другие угрозы или провести профессиональный тест на проникновение.

### A01 Нарушенный контроль доступа

#### Слабое управление аккаунтами

Убедитесь, что файлы cookie отправляются с флагом HttpOnly, чтобы предотвратить доступ клиентских скриптов к cookie:

```csharp
CookieHttpOnly = true,
```

Сократите период времени, в течение которого сессия может быть украдена, уменьшая тайм-аут сессии и отключая скользящее истечение срока:

```csharp
ExpireTimeSpan = TimeSpan.FromMinutes(60),
SlidingExpiration = false
```

Пример полного кода старта можно найти [здесь](https://github.com/johnstaveley/SecurityEssentials/blob/master/SecurityEssentials/App_Start/Startup.Auth.cs).

Убедитесь, что файлы cookie отправляются через HTTPS в производственной среде. Это должно быть настроено в конфигурационных преобразованиях:

```xml
<httpCookies requireSSL="true" />
<authentication>
    <forms requireSSL="true" />
</authentication>
```

Защитите методы входа, регистрации и сброса пароля от атак методом грубой силы, ограничив количество запросов (см. код ниже). Также рассмотрите использование ReCaptcha.

```csharp
[HttpPost]
[AllowAnonymous]
[ValidateAntiForgeryToken]
[AllowXRequestsEveryXSecondsAttribute(Name = "LogOn",
Message = "Вы выполнили это действие более {x} раз за последние {n} секунд.",
Requests = 3, Seconds = 60)]
public async Task<ActionResult> LogOn(LogOnViewModel model, string returnUrl)
```

НЕ: Разрабатывайте собственную систему аутентификации или управления сессиями. Используйте готовую, предоставленную .NET.

НЕ: Сообщайте пользователю, существует ли учетная запись на этапе входа, регистрации или сброса пароля. Сообщайте что-то вроде «Логин или пароль неверны» или «Если эта учетная запись существует, на зарегистрированный email будет отправлен токен для сброса». Это защищает от подбора учетных записей.

Ответ пользователю должен быть одинаковым, существует ли учетная запись или нет, как по содержанию, так и по поведению. Например, если ответ занимает на 50% больше времени, когда учетная запись существует, это может позволить злоумышленникам угадывать и проверять существование учетных записей.

#### Отсутствие контроля доступа на уровне функций

Делайте: Авторизуйте пользователей на всех внешних конечных точках. .NET framework предоставляет множество способов авторизации пользователя, используйте их на уровне методов:

```csharp
[Authorize(Roles = "Admin")]
[HttpGet]
public ActionResult Index(int page = 1)
```

или еще лучше, на уровне контроллера:

```csharp
[Authorize]
public class UserController
```

Вы также можете проверять роли в коде, используя функции идентификации в .NET: `System.Web.Security.Roles.IsUserInRole(userName, roleName)`

Больше информации можно найти в [Шпаргалке по авторизации](Authorization_Cheat_Sheet.md) и [Шпаргалке по автоматизации тестирования авторизации](Authorization_Testing_Automation_Cheat_Sheet.md).

#### Небезопасные прямые ссылки на объекты

Если у вас есть ресурс (объект), доступ к которому предоставляется по ссылке (например, в коде ниже это `id`), вы должны убедиться, что пользователь имеет право на доступ к этому ресурсу.

```csharp
// Небезопасно
public ActionResult Edit(int id)
{
  var user = _context.Users.FirstOrDefault(e => e.Id == id);
  return View("Details", new UserViewModel(user);
}

// Безопасно
public ActionResult Edit(int id)
{
  var user = _context.Users.FirstOrDefault(e => e.Id == id);
  // Убедитесь, что пользователь имеет право редактировать данные
  if (user.Id != _userIdentity.GetUserId())
  {
        HandleErrorInfo error = new HandleErrorInfo(
            new Exception("INFO: У вас нет прав для редактирования этих данных"));
        return View("Error", error);
  }
  return View("Edit", new UserViewModel(user);
}
```

Дополнительную информацию можно найти в [Шпаргалке по предотвращению небезопасных прямых ссылок на объекты](Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.md).

### A02 Криптографические ошибки

#### Общие рекомендации по криптографии

- **Никогда не пишите свои криптографические функции.**
- По возможности избегайте написания криптографического кода. Вместо этого постарайтесь использовать готовые решения для управления секретами или решения, предоставляемые вашим облачным провайдером. Подробнее см. в [Шпаргалке по управлению секретами OWASP](Secrets_Management_Cheat_Sheet.md).
- Если вы не можете использовать готовое решение, постарайтесь использовать проверенную библиотеку, а не встроенные библиотеки .NET, так как с ними слишком легко допустить ошибки.
- Убедитесь, что ваше приложение или протокол могут легко поддерживать будущее изменение криптографических алгоритмов.

#### Хеширование

РЕКОМЕНДУЕТСЯ: Использовать сильный хеш-алгоритм.

- В .NET (как Framework, так и Core) самый сильный хеш-алгоритм для общих задач хеширования — это
  [System.Security.Cryptography.SHA512](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.sha512).
- В .NET Framework 4.6 и более ранних версиях самым сильным алгоритмом для хеширования паролей является PBKDF2, реализованный в
  [System.Security.Cryptography.Rfc2898DeriveBytes](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.rfc2898derivebytes).
- В .NET Framework 4.6.1 и более поздних версиях, а также в .NET Core самым сильным алгоритмом для хеширования паролей является PBKDF2, реализованный в
  [Microsoft.AspNetCore.Cryptography.KeyDerivation.Pbkdf2](https://docs.microsoft.com/en-us/aspnet/core/security/data-protection/consumer-apis/password-hashing), который имеет несколько значительных преимуществ по сравнению с `Rfc2898DeriveBytes`.
- При использовании хеш-функции для хеширования неуникальных данных, таких как пароли, используйте значение соли, добавленное к исходному значению перед хешированием.
- Подробности см. в [Шпаргалке по хранению паролей](Password_Storage_Cheat_Sheet.md).

#### Пароли

РЕКОМЕНДУЕТСЯ: Применять пароли с минимальной сложностью, которые смогут выдержать словарные атаки, т.е. более длинные пароли, использующие полный набор символов (цифры, символы и буквы) для увеличения энтропии.

#### Шифрование

РЕКОМЕНДУЕТСЯ: Использовать сильный алгоритм шифрования, такой как AES-512, для персональных данных, которые нужно восстановить в исходном формате.

РЕКОМЕНДУЕТСЯ: Защищать ключи шифрования больше, чем любые другие активы. Подробнее о хранении ключей шифрования в состоянии покоя см. в [Шпаргалке по управлению ключами](Key_Management_Cheat_Sheet.md#storage).

РЕКОМЕНДУЕТСЯ: Использовать TLS 1.2+ для всего сайта. Получите бесплатный сертификат [LetsEncrypt.org](https://letsencrypt.org/) и автоматизируйте его обновление.

НЕ РЕКОМЕНДУЕТСЯ: [Использовать SSL, так как он устарел](https://github.com/ssllabs/research/wiki/SSL-and-TLS-Deployment-Best-Practices).

РЕКОМЕНДУЕТСЯ: Иметь строгую политику TLS (см. [Лучшие практики SSL](https://www.ssllabs.com/projects/best-practices/index.html)), используйте TLS 1.2+ где возможно. Проверьте конфигурацию с помощью [SSL Test](https://www.ssllabs.com/ssltest/) или [TestSSL](https://testssl.sh/).

Дополнительную информацию о защите транспортного уровня можно найти в [Шпаргалке по защите транспортного уровня](Transport_Layer_Security_Cheat_Sheet.md).

РЕКОМЕНДУЕТСЯ: Убедиться, что заголовки не раскрывают информацию о вашем приложении. См. [HttpHeaders.cs](https://github.com/johnstaveley/SecurityEssentials/blob/master/SecurityEssentials/Core/HttpHeaders.cs), [Dionach StripHeaders](https://github.com/Dionach/StripHeaders/), отключите через `web.config` или [Startup.cs](https://medium.com/bugbountywriteup/security-headers-1c770105940b).

Пример Web.config:

```xml
<system.web>
    <httpRuntime enableVersionHeader="false"/>
</system.web>
<system.webServer>
    <security>
        <requestFiltering removeServerHeader="true" />
    </security>
    <httpProtocol>
        <customHeaders>
            <add name="X-Content-Type-Options" value="nosniff" />
            <add name="X-Frame-Options" value="DENY" />
            <add name="X-Permitted-Cross-Domain-Policies" value="master-only"/>
            <add name="X-XSS-Protection" value="0"/>
            <remove name="X-Powered-By"/>
        </customHeaders>
    </httpProtocol>
</system.webServer>
```

Пример Startup.cs:

```csharp
app.UseHsts(hsts => hsts.MaxAge(365).IncludeSubdomains());
app.UseXContentTypeOptions();
app.UseReferrerPolicy(opts => opts.NoReferrer());
app.UseXXssProtection(options => options.FilterDisabled());
app.UseXfo(options => options.Deny());

app.UseCsp(opts => opts
 .BlockAllMixedContent()
 .StyleSources(s => s.Self())
 .StyleSources(s => s.UnsafeInline())
 .FontSources(s => s.Self())
 .FormActions(s => s.Self())
 .FrameAncestors(s => s.Self())
 .ImageSources(s => s.Self())
 .ScriptSources(s => s.Self())
 );
```

Больше информации о заголовках можно найти в [Проекте безопасных заголовков OWASP](https://owasp.org/www-project-secure-headers/).

#### Шифрование для хранения данных

- Используйте [Windows Data Protection API (DPAPI)](https://docs.microsoft.com/en-us/dotnet/standard/security/how-to-use-data-protection) для безопасного локального хранения конфиденциальных данных.
- Если использование DPAPI невозможно, следуйте рекомендациям по выбору алгоритмов в [Шпаргалке по криптографическому хранению данных OWASP](Cryptographic_Storage_Cheat_Sheet.md#algorithms).

Ниже приведен пример использования AES-GCM для шифрования и дешифрования данных. Настоятельно рекомендуется, чтобы эксперт по криптографии проверил ваш окончательный дизайн и код, так как даже самая незначительная ошибка может значительно ослабить шифрование.

Пример кода взят отсюда: [https://www.scottbrady91.com/c-sharp/aes-gcm-dotnet](https://www.scottbrady91.com/c-sharp/aes-gcm-dotnet)

Некоторые ограничения и подводные камни в этом коде:

- В коде не учтено управление ротацией ключей, что является отдельной темой.
- Важно использовать уникальный nonce для каждой операции шифрования, даже если используется тот же ключ.
- Ключ должен храниться безопасно.

<details>
  <summary>Нажмите, чтобы просмотреть пример кода "Симметричное шифрование AES-GCM".</summary>

```csharp
// Код основан на примере отсюда:
// https://www.scottbrady91.com/c-sharp/aes-gcm-dotnet

public class AesGcmSimpleTest
{
    public static void Main()
    {
        // Ключ длиной 32 байта / 256 бит для AES
        var key = new byte[32];
        RandomNumberGenerator.Fill(key);

        // Размер nonce = 12 байт / 96 бит, всегда должен использоваться этот размер
        var nonce = new byte[AesGcm.NonceByteSizes.MaxSize];
        RandomNumberGenerator.Fill(nonce);

        // Тег для аутентифицированного шифрования
        var tag = new byte[AesGcm.TagByteSizes.MaxSize];

        var message = "Сообщение для шифрования";
        Console.WriteLine(message);

        // Шифруем сообщение
        var cipherText = AesGcmSimple.Encrypt(message, nonce, out tag, key);
        Console.WriteLine(Convert.ToBase64String(cipherText));

        // Дешифруем сообщение
        var message2 = AesGcmSimple.Decrypt(cipherText, nonce, tag, key);
        Console.WriteLine(message2);
    }
}


public static class AesGcmSimple
{
    public static byte[] Encrypt(string plaintext, byte[] nonce, out byte[] tag, byte[] key)
    {
        using (var aes = new AesGcm(key))
        {
            // Тег для аутентифицированного шифрования
            tag = new byte[AesGcm.TagByteSizes.MaxSize];

            // Преобразуем сообщение в байты
            var plaintextBytes = Encoding.UTF8.GetBytes(plaintext);

            // Длина шифротекста будет такой же, как и у исходного текста
            var ciphertext = new byte[plaintextBytes.Length];

            // Выполняем шифрование
            aes.Encrypt(nonce, plaintextBytes, ciphertext, tag);
            return ciphertext;
        }
    }

    public static string Decrypt(byte[] ciphertext, byte[] nonce, byte[] tag, byte[] key)
    {
        using (var aes = new AesGcm(key))
        {
            // Длина исходного текста будет такой же, как и у шифротекста
            var plaintextBytes = new byte[ciphertext.Length];

            // Выполняем дешифрование
            aes.Decrypt(nonce, ciphertext, tag, plaintextBytes);

            return Encoding.UTF8.GetString(plaintextBytes);
        }
    }
}

```

</details>

#### Шифрование для передачи данных

- Следуйте рекомендациям по выбору алгоритмов в [Шпаргалке по криптографическому хранению данных OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html#algorithms).

Пример ниже показывает использование эллиптической кривой/алгоритма Диффи-Хеллмана (ECDH) в сочетании с AES-GCM для шифрования и дешифрования данных между двумя сторонами без необходимости передачи симметричного ключа между ними. Вместо этого стороны обмениваются открытыми ключами и могут использовать ECDH для генерации общего секрета, который затем используется для симметричного шифрования.

Как и ранее, настоятельно рекомендуется, чтобы эксперт по криптографии проверил ваш окончательный дизайн и код, так как даже незначительная ошибка может ослабить безопасность шифрования.

Пример кода полагается на класс `AesGcmSimple` из предыдущего раздела.

Несколько ограничений/подводных камней этого кода:

- Он не учитывает ротацию или управление ключами, что является отдельной темой.
- Код преднамеренно требует нового nonce для каждой операции шифрования, но это должно управляться как отдельный элемент данных вместе с шифротекстом.
- Приватные ключи должны храниться безопасно.
- Код не учитывает проверку публичных ключей перед использованием.
- В целом, отсутствует проверка подлинности между двумя сторонами.

<details>
  <summary>Нажмите здесь, чтобы просмотреть фрагмент кода "ECDH асимметричное шифрование".</summary>

```csharp
public class ECDHSimpleTest
{
    public static void Main()
    {
        // Генерация пары ECC-ключей для Алисы
        var alice = new ECDHSimple();
        byte[] alicePublicKey = alice.PublicKey;

        // Генерация пары ECC-ключей для Боба
        var bob = new ECDHSimple();
        byte[] bobPublicKey = bob.PublicKey;

        string plaintext = "Привет, Боб! Как ты?";
        Console.WriteLine("Секрет, отправляемый от Алисы к Бобу: " + plaintext);

        // Обратите внимание, что новый nonce генерируется при каждой операции шифрования в соответствии с
        // моделью безопасности AES GCM
        byte[] tag;
        byte[] nonce;
        var cipherText = alice.Encrypt(bobPublicKey, plaintext, out nonce, out tag);
        Console.WriteLine("Шифротекст, nonce и tag, отправляемые от Алисы к Бобу: " + Convert.ToBase64String(cipherText) + " " + Convert.ToBase64String(nonce) + " " + Convert.ToBase64String(tag));

        var decrypted = bob.Decrypt(alicePublicKey, cipherText, nonce, tag);
        Console.WriteLine("Секрет, полученный Бобом от Алисы: " + decrypted);

        Console.WriteLine();

        string plaintext2 = "Привет, Алиса! Я в порядке, как ты?";
        Console.WriteLine("Секрет, отправляемый от Боба к Алисе: " + plaintext2);

        byte[] tag2;
        byte[] nonce2;
        var cipherText2 = bob.Encrypt(alicePublicKey, plaintext2, out nonce2, out tag2);
        Console.WriteLine("Шифротекст, nonce и tag, отправляемые от Боба к Алисе: " + Convert.ToBase64String(cipherText2) + " " + Convert.ToBase64String(nonce2) + " " + Convert.ToBase64String(tag2));

        var decrypted2 = alice.Decrypt(bobPublicKey, cipherText2, nonce2, tag2);
        Console.WriteLine("Секрет, полученный Алисой от Боба: " + decrypted2);
    }
}


public class ECDHSimple
{

    private ECDiffieHellmanCng ecdh = new ECDiffieHellmanCng();

    public byte[] PublicKey
    {
        get
        {
            return ecdh.PublicKey.ToByteArray();
        }
    }

    public byte[] Encrypt(byte[] partnerPublicKey, string message, out byte[] nonce, out byte[] tag)
    {
        // Генерация ключа AES и nonce
        var aesKey = GenerateAESKey(partnerPublicKey);

        // Tag для аутентифицированного шифрования
        tag = new byte[AesGcm.TagByteSizes.MaxSize];

        // MaxSize = 12 байт / 96 бит, и этот размер всегда должен использоваться.
        // Новый nonce генерируется при каждой операции шифрования в соответствии с
        // моделью безопасности AES GCM
        nonce = new byte[AesGcm.NonceByteSizes.MaxSize];
        RandomNumberGenerator.Fill(nonce);

        // возвращает зашифрованное значение
        return AesGcmSimple.Encrypt(message, nonce, out tag, aesKey);
    }


    public string Decrypt(byte[] partnerPublicKey, byte[] ciphertext, byte[] nonce, byte[] tag)
    {
        // Генерация ключа AES и nonce
        var aesKey = GenerateAESKey(partnerPublicKey);

        // возвращает расшифрованное значение
        return AesGcmSimple.Decrypt(ciphertext, nonce, tag, aesKey);
    }

    private byte[] GenerateAESKey(byte[] partnerPublicKey)
    {
        // Получение секрета на основе приватного ключа этой стороны и публичного ключа другой стороны
        byte[] secret = ecdh.DeriveKeyMaterial(CngKey.Import(partnerPublicKey, CngKeyBlobFormat.EccPublicBlob));

        byte[] aesKey = new byte[32]; // 256-битный ключ AES
        Array.Copy(secret, 0, aesKey, 0, 32); // Копирование первых 32 байтов в качестве ключа

        return aesKey;
    }
}
```

</details>

### A03 Инъекции

#### SQL Инъекция

**ДЕЛАЙТЕ:** Используйте объектно-реляционные мапперы (ORM) или хранимые процедуры, так как это наиболее эффективный способ защиты от уязвимости SQL-инъекции.

**ДЕЛАЙТЕ:** Используйте параметризованные запросы, если необходимо использовать прямой SQL-запрос. Дополнительную информацию можно найти в [Чек-листе по параметризации запросов](Query_Parameterization_Cheat_Sheet.md).

Пример использования Entity Framework:

```csharp
var sql = @"Update [User] SET FirstName = @FirstName WHERE Id = @Id";
context.Database.ExecuteSqlCommand(
    sql,
    new SqlParameter("@FirstName", firstname),
    new SqlParameter("@Id", id));
```

**ДЕЛАЙТЕ:** Конкатенируйте строки в любом месте вашего кода и выполняйте их против вашей базы данных (известно как *динамический SQL*).

Примечание: Даже с ORM или хранимыми процедурами можно случайно это сделать, поэтому проверяйте везде. Например:

```csharp
string sql = "SELECT * FROM Users WHERE UserName='" + txtUser.Text + "' AND Password='"
                + txtPassword.Text + "'";
context.Database.ExecuteSqlCommand(sql); // Уязвимость SQL-инъекции!
```

**ДЕЛАЙТЕ:** Практикуйте принцип наименьших привилегий – подключайтесь к базе данных, используя учетную запись с минимальным набором разрешений, необходимых для выполнения своей работы, а не учетную запись администратора базы данных.

#### Инъекции в ОС

Общие рекомендации по защите от OS-инъекций можно найти в [Чек-листе по защите от командных инъекций OS](OS_Command_Injection_Defense_Cheat_Sheet.md).

**ДЕЛАЙТЕ:** Используйте [System.Diagnostics.Process.Start](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process.start?view=netframework-4.7.2) для вызова функций ОС.

Пример:

```csharp
var process = new System.Diagnostics.Process();
var startInfo = new System.Diagnostics.ProcessStartInfo();
startInfo.FileName = "validatedCommand";
startInfo.Arguments = "validatedArg1 validatedArg2 validatedArg3";
process.StartInfo = startInfo;
process.Start();
```

**НЕ ДЕЛАЙТЕ:** Предполагается, что этот механизм защитит от вредоносного ввода, предназначенного для выхода из одного аргумента и последующего вмешательства в другой аргумент процесса. Это все еще будет возможно.

**ДЕЛАЙТЕ:** Используйте проверку в белом списке для всех пользовательских данных, где это возможно. Проверка ввода предотвращает попадание неправильно сформированных данных в информационную систему. Для получения дополнительной информации смотрите [Чек-лист по проверке ввода](Input_Validation_Cheat_Sheet.md).

Пример проверки пользовательского ввода с использованием [Метода IPAddress.TryParse](https://docs.microsoft.com/en-us/dotnet/api/system.net.ipaddress.tryparse?view=netframework-4.8):

```csharp
// Ввод пользователя
string ipAddress = "127.0.0.1";

// проверьте, что был предоставлен IP-адрес
if (!string.IsNullOrEmpty(ipAddress))
{
    // Создание экземпляра IPAddress для указанной строки адреса (в
    // форме dotted-quad или colon-hexadecimal notation).
    if (IPAddress.TryParse(ipAddress, out var address))
    {
        // Отображение адреса в стандартной нотации.
        return address.ToString();
    }
    else
    {
        // ipAddress не является типом IPAddress
        ...
    }
    ...
}
```

**НЕ ДЕЛАЙТЕ:** Пытайтесь принимать только простые алфавитно-цифровые символы.

**НЕ ДЕЛАЙТЕ:** Предполагается, что вы можете очищать специальные символы без их фактического удаления. Различные комбинации ```\```, ```'``` и ```@``` могут иметь непредвиденное воздействие на попытки очистки.

**НЕ ДЕЛАЙТЕ:** Полагайтесь на методы без гарантии безопасности.

Например, .NET Core 2.2 и новее, а также .NET 5 и новее поддерживают [ProcessStartInfo.ArgumentList](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.processstartinfo.argumentlist), который выполняет некоторую экранизацию символов, но объект содержит [отказ от ответственности о том, что он не безопасен для ненадежного ввода](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.processstartinfo.argumentlist#remarks).

**ДЕЛАЙТЕ:** Рассмотрите альтернативы передаче необработанных ненадежных аргументов через параметры командной строки, такие как кодирование с использованием Base64 (что безопасно закодирует любые специальные символы) и последующая декодировка параметров в принимаемом приложении.

#### LDAP Injection

Практически любые символы могут использоваться в Distinguished Names. Однако некоторые из них должны быть экранированы с помощью символа экранирования обратного слэша `\`. Таблица, показывающая, какие символы следует экранировать для Active Directory, можно найти в [Шпаргалке по предотвращению LDAP-инъекций](LDAP_Injection_Prevention_Cheat_Sheet.md).

Примечание: Пробел должен быть экранирован только в том случае, если он является начальным или конечным символом в имени компонента, таком как Common Name. Вложенные пробелы экранировать не нужно.

Дополнительную информацию можно найти в [Шпаргалке по предотвращению LDAP-инъекций](LDAP_Injection_Prevention_Cheat_Sheet.md).

### A04 Небезопасный дизайн

Небезопасный дизайн относится к сбоям безопасности в проектировании приложения или системы. Это отличается от других элементов списка OWASP Top 10, которые относятся к сбоям реализации. Тема безопасного дизайна не связана с конкретной технологией или языком и, следовательно, выходит за рамки этого чек-листа. Для получения дополнительной информации см. [Шпаргалке по безопасному дизайну продукта](Secure_Product_Design_Cheat_Sheet.md).

### A05 Неправильная конфигурация безопасности

#### Отладка и трассировка

Убедитесь, что отладка и трассировка отключены в производственной среде. Это можно обеспечить с помощью преобразований web.config:

```xml
<compilation xdt:Transform="RemoveAttributes(debug)" />
<trace enabled="false" xdt:Transform="Replace"/>
```

**НЕ ДЕЛАЙТЕ:** Используйте стандартные пароли

**ДЕЛАЙТЕ:** Перенаправляйте запросы, сделанные по HTTP, на HTTPS:

Пример, Global.asax.cs:

```csharp
protected void Application_BeginRequest()
{
    #if !DEBUG
    // SECURE: Убедитесь, что любой запрос возвращается по SSL/TLS в производственной среде
    if (!Request.IsLocal && !Context.Request.IsSecureConnection) {
        var redirect = Context.Request.Url.ToString()
                        .ToLower(CultureInfo.CurrentCulture)
                        .Replace("http:", "https:");
        Response.Redirect(redirect);
    }
    #endif
}
```

Пример, Startup.cs в `Configure()`:

```csharp
app.UseHttpsRedirection();
```

#### Межсайтовая подделка запросов (CSRF)

**НЕ ДЕЛАЙТЕ:** Отправляйте чувствительные данные без проверки Anti-Forgery-Tokens ([.NET](https://docs.microsoft.com/en-us/aspnet/web-api/overview/security/preventing-cross-site-request-forgery-csrf-attacks) / [.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/anti-request-forgery?view=aspnetcore-7.0#aspnet-core-antiforgery-configuration)).

**ДЕЛАЙТЕ:** Отправляйте токен защиты от подделки с каждым POST/PUT запросом:

##### Используя .NET Framework

```csharp
using (Html.BeginForm("LogOff", "Account", FormMethod.Post, new { id = "logoutForm",
                        @class = "pull-right" }))
{
    @Html.AntiForgeryToken()
    <ul class="nav nav-pills">
        <li role="presentation">
        Logged on as @User.Identity.Name
        </li>
        <li role="presentation">
        <a href="javascript:document.getElementById('logoutForm').submit()">Выйти</a>
        </li>
    </ul>
}
```

Затем проверьте его на уровне метода или предпочтительно на уровне контроллера:

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public ActionResult LogOff()
```

Убедитесь, что токены полностью удалены для аннулирования при выходе из системы.

```csharp
/// <summary>
/// SECURE: Удалить любые оставшиеся cookies, включая cookie Anti-CSRF
/// </summary>
public void RemoveAntiForgeryCookie(Controller controller)
{
    string[] allCookies = controller.Request.Cookies.AllKeys;
    foreach (string cookie in allCookies)
    {
        if (controller.Response.Cookies[cookie] != null &&
            cookie == "__RequestVerificationToken")
        {
            controller.Response.Cookies[cookie].Expires = DateTime.Now.AddDays(-1);
        }
    }
}
```

##### Использование .NET Core 2.0 и новее

Начиная с .NET Core 2.0, можно [автоматически генерировать и проверять токены защиты от подделки](https://docs.microsoft.com/en-us/aspnet/core/security/anti-request-forgery?view=aspnetcore-7.0#aspnet-core-antiforgery-configuration).

Если вы используете [tag-helpers](https://docs.microsoft.com/en-us/aspnet/core/mvc/views/tag-helpers/intro), что является стандартом для большинства шаблонов веб-проектов, то все формы автоматически отправляют токен защиты от подделки. Вы можете проверить, включены ли tag-helpers, проверив наличие следующей строки в вашем основном файле `_ViewImports.cshtml`:

```csharp
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

`IHtmlHelper.BeginForm` также автоматически отправляет токены защиты от подделки.

Если вы не используете tag-helpers или `IHtmlHelper.BeginForm`, необходимо использовать требуемый хелпер в формах, как показано здесь:

```html
<form action="RelevantAction" >
@Html.AntiForgeryToken()
</form>
```

Чтобы автоматически проверять все запросы, кроме GET, HEAD, OPTIONS и TRACE, необходимо добавить глобальный фильтр действий с атрибутом [AutoValidateAntiforgeryToken](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.autovalidateantiforgerytokenattribute?view=aspnetcore-7.0) в ваш `Startup.cs`, как указано в следующей [статье](https://andrewlock.net/automatically-validating-anti-forgery-tokens-in-asp-net-core-with-the-autovalidateantiforgerytokenattribute/):

```csharp
services.AddMvc(options =>
{
    options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute());
});
```

Если вам нужно отключить проверку атрибута для конкретного метода контроллера, вы можете добавить атрибут [IgnoreAntiforgeryToken](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.ignoreantiforgerytokenattribute?view=aspnetcore-7.0) к методу контроллера (для MVC-контроллеров) или к родительскому классу (для Razor-страниц):

```csharp
[IgnoreAntiforgeryToken]
[HttpDelete]
public IActionResult Delete()
```

```csharp
[IgnoreAntiforgeryToken]
public class UnsafeModel : PageModel
```

Если вам также нужно проверять токен для запросов GET, HEAD, OPTIONS и TRACE, вы можете добавить атрибут [ValidateAntiforgeryToken](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.validateantiforgerytokenattribute?view=aspnetcore-7.0) к методу контроллера (для MVC-контроллеров) или к родительскому классу (для Razor-страниц):

```csharp
[HttpGet]
[ValidateAntiforgeryToken]
public IActionResult DoSomethingDangerous()
```

```csharp
[HttpGet]
[ValidateAntiforgeryToken]
public class SafeModel : PageModel
```

В случае, если вы не можете использовать глобальный фильтр действий, добавьте атрибут [AutoValidateAntiforgeryToken](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.autovalidateantiforgerytokenattribute?view=aspnetcore-7.0) к вашим классам контроллеров или моделям Razor-страниц:

```csharp
[AutoValidateAntiforgeryToken]
public class UserController
```

```csharp
[AutoValidateAntiforgeryToken]
public class SafeModel : PageModel
```

##### Использование .NET Core или .NET Framework с AJAX

Необходимо прикрепить токен защиты от подделки к AJAX-запросам.

Если вы используете jQuery в представлении ASP.NET Core MVC, это можно сделать с помощью следующего фрагмента:

```javascript
@inject  Microsoft.AspNetCore.Antiforgery.IAntiforgery antiforgeryProvider
$.ajax(
{
    type: "POST",
    url: '@Url.Action("Action", "Controller")',
    contentType: "application/x-www-form-urlencoded; charset=utf-8",
    data: {
        id: id,
        '__RequestVerificationToken': '@antiforgeryProvider.GetAndStoreTokens(this.Context).RequestToken'
    }
})
```

Если вы используете .NET Framework, вы можете найти некоторые фрагменты кода [здесь](https://docs.microsoft.com/en-us/aspnet/web-api/overview/security/preventing-cross-site-request-forgery-csrf-attacks#anti-csrf-and-ajax).

Более подробную информацию можно найти в [Чек-листе по предотвращению подделки межсайтовых запросов](Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.md).

### A06 Уязвимые и устаревшие компоненты

ДЕЛАЙТЕ: Держите .NET Framework обновленным с последними патчами

ДЕЛАЙТЕ: Держите ваши [NuGet](https://docs.microsoft.com/en-us/nuget/) пакеты актуальными

ДЕЛАЙТЕ: Запускайте [OWASP Dependency Checker](Vulnerable_Dependency_Management_Cheat_Sheet.md#tools) против вашего приложения в рамках процесса сборки и реагируйте на любые уязвимости высокого или критического уровня.

ДЕЛАЙТЕ: Включите инструменты анализа состава программного обеспечения (SCA) в ваш CI/CD pipeline, чтобы обеспечить обнаружение и устранение новых уязвимостей в ваших зависимостях.

### A07 Ошибки идентификации и аутентификации

ДЕЛАЙТЕ: Используйте [ASP.NET Core Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-2.2&). Фреймворк ASP.NET Core Identity по умолчанию настроен на использование безопасных хешей паролей и индивидуальной соли. Identity использует функцию хеширования PBKDF2 для паролей и генерирует случайную соль для каждого пользователя.

ДЕЛАЙТЕ: Установите безопасную политику паролей

например, ASP.NET Core Identity

```csharp
//Startup.cs
services.Configure<IdentityOptions>(options =>
{
 // Настройки паролей
 options.Password.RequireDigit = true;
 options.Password.RequiredLength = 8;
 options.Password.RequireNonAlphanumeric = true;
 options.Password.RequireUppercase = true;
 options.Password.RequireLowercase = true;
 options.Password.RequiredUniqueChars = 6;

 options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(30);
 options.Lockout.MaxFailedAccessAttempts = 3;

 options.SignIn.RequireConfirmedEmail = true;

 options.User.RequireUniqueEmail = true;
});
```

ДЕЛАЙТЕ: Установите политику для куки-файлов

например,

```csharp
//Startup.cs
services.ConfigureApplicationCookie(options =>
{
 options.Cookie.HttpOnly = true;
 options.Cookie.Expiration = TimeSpan.FromHours(1);
 options.SlidingExpiration = true;
});
```

### A08 Ошибки целостности программного обеспечения и данных

ДЕЛАЙТЕ: Цифрово подписывайте сборки и исполняемые файлы

ДЕЛАЙТЕ: Используйте подпись пакетов NuGet

ДЕЛАЙТЕ: Проверяйте код и изменения конфигурации, чтобы избежать внедрения вредоносного кода или зависимостей

НЕ ДЕЛАЙТЕ: Отправляйте неподписанные или нешифрованные сериализованные объекты по сети

ДЕЛАЙТЕ: Выполняйте проверки целостности или проверяйте цифровые подписи на сериализованных объектах, полученных из сети

НЕ ДЕЛАЙТЕ: Используйте тип BinaryFormatter, который опасен и [не рекомендуется](https://learn.microsoft.com/en-us/dotnet/standard/serialization/binaryformatter-security-guide) для обработки данных. .NET предлагает несколько встроенных сериализаторов, которые могут безопасно обрабатывать ненадежные данные:

- XmlSerializer и DataContractSerializer для сериализации графов объектов в XML и из XML. Не путайте DataContractSerializer с NetDataContractSerializer.
- BinaryReader и BinaryWriter для XML и JSON.
- API System.Text.Json для сериализации графов объектов в JSON.

### A09 Ошибки в регистрации и мониторинге безопасности

ДЕЛАЙТЕ: Убедитесь, что все неудачные попытки входа, контроль доступа и проверки ввода на серверной стороне регистрируются с достаточным контекстом пользователя для идентификации подозрительных или вредоносных аккаунтов.

ДЕЛАЙТЕ: Установите эффективное мониторинг и оповещение, чтобы подозрительная активность могла быть обнаружена и на нее своевременно отреагировали.

НЕ ДЕЛАЙТЕ: Регистрируйте общие сообщения об ошибках, такие как: ```csharp Log.Error("Error was thrown");```. Вместо этого, регистрируйте трассировку стека, сообщение об ошибке и ID пользователя, вызвавшего ошибку.

НЕ ДЕЛАЙТЕ: Регистрируйте чувствительные данные, такие как пароли пользователей.

#### Регистрация

Что регистрировать и дополнительная информация о регистрации можно найти в [Шпаргалке по логированию](Logging_Cheat_Sheet.md).

.NET Core поставляется с LoggerFactory, который находится в Microsoft.Extensions.Logging. Более подробную информацию об ILogger можно найти [здесь](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.ilogger).

Вот как регистрировать все ошибки из `Startup.cs`, чтобы каждая ошибка, которая возникает, регистрировалась:

```csharp
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
    {
        _isDevelopment = true;
        app.UseDeveloperExceptionPage();
    }

    // Регистрируем все ошибки в приложении
    app.UseExceptionHandler(errorApp =>
    {
        errorApp.Run(async context =>
        {
            var errorFeature = context.Features.Get<IExceptionHandlerFeature>();
            var exception = errorFeature.Error;

            Log.Error(String.Format("Stacktrace of error: {0}", exception.StackTrace.ToString()));
        });
    });

    app.UseAuthentication();
    app.UseMvc();
}
```

Например, внедрение в конструктор класса, что упрощает написание модульных тестов. Это рекомендуется, если экземпляры класса будут создаваться с помощью внедрения зависимостей (например, контроллеры MVC). Пример ниже показывает регистрацию всех неудачных попыток входа.

```csharp
public class AccountsController : Controller
{
    private ILogger _Logger;

    public AccountsController(ILogger logger)
    {
        _Logger = logger;
    }

    [HttpPost]
    [AllowAnonymous]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Login(LoginViewModel model)
    {
        if (ModelState.IsValid)
        {
            var result = await _signInManager.PasswordSignInAsync(model.Email, model.Password, model.RememberMe, lockoutOnFailure: false);
            if (result.Succeeded)
            {
                // Регистрируем все успешные попытки входа
                Log.Information(String.Format("User: {0}, Successfully Logged in", model.Email));
                // Код для успешного входа
                //...
            }
            else
            {
                // Регистрируем все неправильные попытки входа
                Log.Information(String.Format("User: {0}, Incorrect Password", model.Email));
            }
        }
        ...
    }
}
```

#### Мониторинг

Мониторинг позволяет нам оценивать производительность и здоровье работающей системы через ключевые показатели эффективности.

В .NET отличным вариантом для добавления возможностей мониторинга является [Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/asp-net-core).

Более подробную информацию о регистрации и мониторинге можно найти [здесь](https://github.com/microsoft/code-with-engineering-playbook/blob/main/docs/observability/README.md).

### A10 Подделка запросов на стороне сервера (SSRF)

ДЕЛАЙТЕ: Проверяйте и очищайте все пользовательские вводы перед использованием их для выполнения запросов.

ДЕЛАЙТЕ: Используйте белый список разрешенных протоколов и доменов.

ДЕЛАЙТЕ: Используйте `IPAddress.TryParse()` и `Uri.CheckHostName()`, чтобы убедиться, что IP-адреса и доменные имена действительны.

НЕ ДЕЛАЙТЕ: Следуйте HTTP-редиректам.

НЕ ДЕЛАЙТЕ: Пересылайте необработанные HTTP-ответы пользователю.

Для получения дополнительной информации, пожалуйста, смотрите [Шпаргалка по предотвращению подделки запросов на стороне сервера](Server_Side_Request_Forgery_Prevention_Cheat_Sheet.md).

### OWASP 2013 и 2017

Ниже приведены уязвимости, включенные в списки OWASP Top 10 2013 и 2017 годов, которые не вошли в список 2021 года. Эти уязвимости все еще актуальны, но были исключены из списка 2021 года, поскольку они стали менее распространенными.

#### A04:2017 XML External Entities (XXE)

Атаки XXE возникают, когда XML-парсер неправильно обрабатывает пользовательский ввод, содержащий объявления внешних сущностей в doctype XML-пейлоада.

[Эта статья](https://docs.microsoft.com/en-us/dotnet/standard/data/xml/xml-processing-options) обсуждает наиболее распространенные варианты обработки XML в .NET.

Пожалуйста, обратитесь к [шпаргалке по предотвращению XXE](XML_External_Entity_Prevention_Cheat_Sheet.md#net) для получения более подробной информации о предотвращении XXE и других атак XML Denial of Service.

### A07:2017 Межсайтовый скриптинг (XSS)

НЕ ДЕЛАЙТЕ: Доверяйте любым данным, которые отправляет пользователь. Предпочитайте белые списки (всегда безопасные) вместо черных списков.

Вы получаете кодирование всего HTML-контента с помощью MVC3. Чтобы правильно кодировать весь контент, будь то HTML, JavaScript, CSS, LDAP и т. д., используйте библиотеку Microsoft AntiXSS:

`Install-Package AntiXSS`

Затем настройте в конфигурации:

```xml
<system.web>
<httpRuntime targetFramework="4.5"
enableVersionHeader="false"
encoderType="Microsoft.Security.Application.AntiXssEncoder, AntiXssLibrary"
maxRequestLength="4096" />
```

НЕ ДЕЛАЙТЕ: Используйте атрибут `[AllowHTML]` или вспомогательный класс `@Html.Raw`, если вы абсолютно не уверены, что контент, который вы записываете в браузер, безопасен и правильно экранирован.

ДЕЛАЙТЕ: Включите [Политику безопасности контента (CSP)](Content_Security_Policy_Cheat_Sheet.md#context). Это предотвратит доступ ваших страниц к ресурсам, к которым они не должны иметь доступа (например, вредоносным скриптам):

```xml
<system.webServer>
    <httpProtocol>
        <customHeaders>
            <add name="Content-Security-Policy"
                value="default-src 'none'; style-src 'self'; img-src 'self';
                font-src 'self'; script-src 'self'" />
```

Более подробную информацию можно найти в [шпаргалке по предотвращению межсайтовых скриптов (XSS)](Cross_Site_Scripting_Prevention_Cheat_Sheet.md).

### A08:2017 Небезопасная десериализация

НЕ ДЕЛАЙТЕ: Принимайте сериализованные объекты от ненадежных источников.

ДЕЛАЙТЕ: Проверяйте вводимые пользователем данные.

Злоумышленники могут использовать такие объекты, как cookies, чтобы вставлять вредоносную информацию для изменения ролей пользователей. В некоторых случаях хакеры могут повысить свои права до прав администратора, используя предварительно существующий или кэшированный хэш пароля из предыдущей сессии.

ДЕЛАЙТЕ: Предотвращайте десериализацию объектов домена.

ДЕЛАЙТЕ: Выполняйте код десериализации с ограниченными правами доступа.
Если десериализованный враждебный объект пытается инициировать системный процесс или получить доступ к ресурсу на сервере или в операционной системе хоста, доступ будет отклонен, и будет установлен флаг разрешения, чтобы администратор системы был уведомлен о любой аномальной активности на сервере.

Более подробную информацию о небезопасной десериализации можно найти в [шпаргалке по десериализации](Deserialization_Cheat_Sheet.md#net-csharp).

### A10:2013 Непроверенные перенаправления и переадресации

Защита от этого была введена в шаблоне MVC 3. Вот код:

```csharp
public async Task<ActionResult> LogOn(LogOnViewModel model, string returnUrl)
{
    if (ModelState.IsValid)
    {
        var logonResult = await _userManager.TryLogOnAsync(model.UserName, model.Password);
        if (logonResult.Success)
        {
            await _userManager.LogOnAsync(logonResult.UserName, model.RememberMe);  
            return RedirectToLocal(returnUrl);
...
```

```csharp
private ActionResult RedirectToLocal(string returnUrl)
{
    if (Url.IsLocalUrl(returnUrl))
    {
        return Redirect(returnUrl);
    }
    else
    {
        return RedirectToAction("Landing", "Account");
    }
}
```

### Другие рекомендации

- Защитите от атак Clickjacking и Man-in-the-Middle, которые могут перехватить начальный запрос без TLS: установите заголовки `X-Frame-Options` и `Strict-Transport-Security` (HSTS). Подробности [здесь](https://github.com/johnstaveley/SecurityEssentials/blob/master/SecurityEssentials/Core/HttpHeaders.cs)
- Защитите от атак Man-in-the-Middle для пользователей, которые впервые посещают ваш сайт. Зарегистрируйтесь для [предварительной загрузки HSTS](https://hstspreload.org/)
- Проводите регулярное тестирование безопасности и анализ сервисов Web API. Они скрыты внутри сайтов на MVC, но являются публичными частями сайта, которые могут быть найдены злоумышленником. Все рекомендации для MVC, а также значительная часть рекомендаций для WCF применимы к Web API.
- Также посмотрите [шпаргалку по непроверенным перенаправлениям и переадресациям](Unvalidated_Redirects_and_Forwards_Cheat_Sheet.md).

#### Пример проекта

Для получения дополнительной информации по вышеуказанным пунктам и примерам кода, встроенным в пример приложения MVC5 с расширенной базовой безопасностью, перейдите на проект [Security Essentials Baseline](http://github.com/johnstaveley/SecurityEssentials/).

## Рекомендации по конкретным темам

Этот раздел содержит рекомендации по конкретным темам в .NET.

### Конфигурация и развертывание

- Ограничьте доступ к конфигурационным файлам.
    - Удалите все аспекты конфигурации, которые не используются.
    - Зашифруйте конфиденциальные части файла `web.config` с помощью `aspnet_regiis -pe` ([помощь по командной строке](https://docs.microsoft.com/en-us/previous-versions/dotnet/netframework-2.0/k6h9cz8h(v=vs.80))).
- Для приложений ClickOnce обновите .NET Framework до последней версии, чтобы обеспечить поддержку TLS 1.2 или выше.

### Доступ к данным

- Используйте [параметризованные SQL](https://docs.microsoft.com/en-us/dotnet/api/system.data.sqlclient.sqlcommand.prepare?view=netframework-4.7.2) команды для всех операций с данными, без исключений.
- Не используйте [SqlCommand](https://docs.microsoft.com/en-us/dotnet/api/system.data.sqlclient.sqlcommand) с параметром в виде [конкатенированной SQL-строки](https://docs.microsoft.com/en-gb/visualstudio/code-quality/ca2100-review-sql-queries-for-security-vulnerabilities?view=vs-2017).
- Устанавливайте допустимые значения, поступающие от пользователя. Используйте перечисления (enums), [TryParse](https://docs.microsoft.com/en-us/dotnet/api/system.int32.tryparse#System_Int32_TryParse_System_String_System_Int32__) или справочные значения, чтобы убедиться, что данные от пользователя соответствуют ожидаемым.
    - Перечисления (enums) также уязвимы для неожиданных значений, поскольку .NET проверяет только успешное приведение к базовому типу данных (по умолчанию — целое число). [Enum.IsDefined](https://docs.microsoft.com/en-us/dotnet/api/system.enum.isdefined) может проверить, является ли входное значение допустимым в пределах списка определённых констант.
- Применяйте принцип наименьших привилегий при настройке пользователя базы данных в вашей системе. Пользователь базы данных должен иметь доступ только к тем элементам, которые необходимы для выполнения задачи.
- Использование [Entity Framework](https://docs.microsoft.com/en-us/ef/) является очень эффективным механизмом предотвращения [SQL-инъекций](SQL_Injection_Prevention_Cheat_Sheet.md). **Помните, что создание собственных запросов в Entity Framework так же уязвимо для SQLi, как и обычные SQL-запросы**.
- При использовании SQL Server отдавайте предпочтение [интегрированной аутентификации](https://learn.microsoft.com/en-us/sql/connect/odbc/linux-mac/using-integrated-authentication?view=sql-server-ver16) перед [аутентификацией SQL](https://learn.microsoft.com/en-us/sql/relational-databases/security/choose-an-authentication-mode?view=sql-server-ver16#connecting-through-sql-server-authentication).
- Используйте [Always Encrypted](https://docs.microsoft.com/en-us/sql/relational-databases/security/encryption/always-encrypted-database-engine), если возможно, для защиты конфиденциальных данных (SQL Server 2016+ и Azure SQL).

## Руководство по ASP.NET Web Forms

ASP.NET Web Forms — это оригинальный API для разработки браузерных приложений на платформе .NET Framework, и до сих пор является самой распространенной корпоративной платформой для разработки веб-приложений.

- Всегда используйте [HTTPS](http://support.microsoft.com/kb/324069).
- Включите параметр [requireSSL](https://docs.microsoft.com/en-us/dotnet/api/system.web.configuration.httpcookiessection.requiressl) для файлов cookie и элементов формы, а также параметр [HttpOnly](https://docs.microsoft.com/en-us/dotnet/api/system.web.configuration.httpcookiessection.httponlycookies) для файлов cookie в `web.config`.
- Реализуйте [customErrors](https://docs.microsoft.com/en-us/dotnet/api/system.web.configuration.customerror).
- Убедитесь, что [трассировка](http://www.iis.net/configreference/system.webserver/tracing) отключена.
- Хотя ViewState не всегда подходит для веб-разработки, его использование может обеспечить защиту от CSRF. Чтобы ViewState защищал от CSRF-атак, нужно установить [ViewStateUserKey](https://docs.microsoft.com/en-us/dotnet/api/system.web.ui.page.viewstateuserkey):

```csharp
protected override OnInit(EventArgs e) {
    base.OnInit(e);
    ViewStateUserKey = Session.SessionID;
}
```

Если вы не используете ViewState, то обратитесь к главной странице по умолчанию шаблона ASP.NET Web Forms для ручной реализации токена защиты от CSRF с помощью двойной отправки cookie.

```csharp
private const string AntiXsrfTokenKey = "__AntiXsrfToken";
private const string AntiXsrfUserNameKey = "__AntiXsrfUserName";
private string _antiXsrfTokenValue;

protected void Page_Init(object sender, EventArgs e)
{
    // Код ниже помогает защититься от атак XSRF
    var requestCookie = Request.Cookies[AntiXsrfTokenKey];
    Guid requestCookieGuidValue;
    if (requestCookie != null && Guid.TryParse(requestCookie.Value, out requestCookieGuidValue))
    {
        // Используйте токен Anti-XSRF из cookie
        _antiXsrfTokenValue = requestCookie.Value;
        Page.ViewStateUserKey = _antiXsrfTokenValue;
    }
    else
    {
        // Сгенерируйте новый токен Anti-XSRF и сохраните его в cookie
        _antiXsrfTokenValue = Guid.NewGuid().ToString("N");
        Page.ViewStateUserKey = _antiXsrfTokenValue;
        var responseCookie = new HttpCookie(AntiXsrfTokenKey)
        {
            HttpOnly = true,
            Value = _antiXsrfTokenValue
        };
        if (FormsAuthentication.RequireSSL && Request.IsSecureConnection)
        {
            responseCookie.Secure = true;
        }
        Response.Cookies.Set(responseCookie);
    }
    Page.PreLoad += master_Page_PreLoad;
}

protected void master_Page_PreLoad(object sender, EventArgs e)
{
    if (!IsPostBack)
    {
        // Установите токен Anti-XSRF
        ViewState[AntiXsrfTokenKey] = Page.ViewStateUserKey;
        ViewState[AntiXsrfUserNameKey] = Context.User.Identity.Name ?? String.Empty;
    }
    else
    {
        // Проверьте токен Anti-XSRF
        if ((string)ViewState[AntiXsrfTokenKey] != _antiXsrfTokenValue ||
            (string)ViewState[AntiXsrfUserNameKey] != (Context.User.Identity.Name ?? String.Empty))
        {
            throw new InvalidOperationException("Проверка токена Anti-XSRF не удалась.");
        }
    }
}
```

- Рассмотрите возможность использования [HSTS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security) в IIS. См. [здесь](https://support.microsoft.com/en-us/help/954002/how-to-add-a-custom-http-response-header-to-a-web-site-that-is-hosted) процедуру настройки.
- Вот рекомендуемая конфигурация `web.config`, которая обрабатывает HSTS и другие параметры.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.web>
    <httpRuntime enableVersionHeader="false"/>
  </system.web>
  <system.webServer>
    <security>
      <requestFiltering removeServerHeader="true" />
    </security>
    <staticContent>
      <clientCache cacheControlCustom="public"
                   cacheControlMode="UseMaxAge"
                   cacheControlMaxAge="1.00:00:00"
                   setEtag="true" />
    </staticContent>
    <httpProtocol>
      <customHeaders>
        <add name="Content-Security-Policy"
             value="default-src 'none'; style-src 'self'; img-src 'self'; font-src 'self'" />
        <add name="X-Content-Type-Options" value="NOSNIFF" />
        <add name="X-Frame-Options" value="DENY" />
        <add name="X-Permitted-Cross-Domain-Policies" value="master-only"/>
        <add name="X-XSS-Protection" value="0"/>
        <remove name="X-Powered-By"/>
      </customHeaders>
    </httpProtocol>
    <rewrite>
      <rules>
        <rule name="Redirect to https">
          <match url="(.*)"/>
          <conditions>
            <add input="{HTTPS}" pattern="Off"/>
            <add input="{REQUEST_METHOD}" pattern="^get$|^head$" />
          </conditions>
          <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent"/>
        </rule>
      </rules>
      <outboundRules>
        <rule name="Add HSTS Header" enabled="true">
          <match serverVariable="RESPONSE_Strict_Transport_Security" pattern=".*" />
          <conditions>
            <add input="{HTTPS}" pattern="on" ignoreCase="true" />
          </conditions>
          <action type="Rewrite" value="max-age=15768000" />
        </rule>
      </outboundRules>
    </rewrite>
  </system.webServer>
</configuration>
```

### Удалите заголовок версии, добавив следующую строку в файл `Machine.config`:

```xml
<httpRuntime enableVersionHeader="false" />
```

### Также удалите заголовок "Server", используя класс HttpContext в вашем коде:

```csharp
HttpContext.Current.Response.Headers.Remove("Server");
```

### Проверка HTTP и кодирование

- Не отключайте параметр [validateRequest](http://www.asp.net/whitepapers/request-validation) в файле `web.config` или настройках страницы. Этот параметр обеспечивает ограниченную защиту от XSS в ASP.NET и должен оставаться включённым, так как он обеспечивает частичную защиту от межсайтового скриптинга. Рекомендуется проводить полную проверку запросов в дополнение к встроенной защите.
- Версия .NET Framework 4.5 включает библиотеку [AntiXssEncoder](https://docs.microsoft.com/en-us/dotnet/api/system.web.security.antixss.antixssencoder?view=netframework-4.7.2), которая предоставляет расширенные возможности кодирования для предотвращения XSS-атак. Используйте её.
- Всегда указывайте допустимые значения при приёме данных от пользователя.
- Проверяйте формат URI с помощью метода [Uri.IsWellFormedUriString](https://docs.microsoft.com/en-us/dotnet/api/system.uri.iswellformeduristring).

### Аутентификация с использованием форм

- Используйте файлы cookie для сохранения состояния сессии, если это возможно. Аутентификация без cookie будет по умолчанию использовать [UseDeviceProfile](https://docs.microsoft.com/en-us/dotnet/api/system.web.httpcookiemode?view=netframework-4.7.2).
- Не доверяйте URI запроса для сохранения сессии или авторизации, так как он легко может быть подделан.
- Уменьшите время таймаута формы аутентификации с значения по умолчанию (20 минут) до минимального периода, подходящего для вашего приложения. Если используется [slidingExpiration](https://docs.microsoft.com/en-us/dotnet/api/system.web.security.formsauthentication.slidingexpiration?view=netframework-4.7.2), таймаут будет обновляться после каждого запроса, так что активные пользователи не пострадают.
- Если HTTPS не используется, [slidingExpiration](https://docs.microsoft.com/en-us/dotnet/api/system.web.security.formsauthentication.slidingexpiration?view=netframework-4.7.2) следует отключить. Рассмотрите отключение [slidingExpiration](https://docs.microsoft.com/en-us/dotnet/api/system.web.security.formsauthentication.slidingexpiration?view=netframework-4.7.2), даже если HTTPS используется.
- Всегда внедряйте корректные механизмы контроля доступа.
    - Сравнивайте имя пользователя, предоставленное пользователем, с `User.Identity.Name`.
    - Проверяйте роли через метод `User.Identity.IsInRole`.
- Используйте [поставщика членства и ролей ASP.NET](https://docs.microsoft.com/en-us/dotnet/framework/wcf/samples/membership-and-role-provider), но обратите внимание на хранение паролей. Хэширование паролей по умолчанию выполняется с помощью одного итерационного шага алгоритма SHA-1, что довольно слабо. В шаблоне ASP.NET MVC4 используется [ASP.NET Identity](http://www.asp.net/identity/overview/getting-started/introduction-to-aspnet-identity), который по умолчанию использует PBKDF2, что лучше. Ознакомьтесь с [шпаргалкой OWASP по хранению паролей](Password_Storage_Cheat_Sheet.md) для получения дополнительной информации.
- Явно авторизуйте запросы к ресурсам.
- Используйте авторизацию на основе ролей через метод `User.Identity.IsInRole`.

## Руководство по XAML

- Работайте в рамках ограничений безопасности зоны Интернета для вашего приложения.
- Используйте развертывание ClickOnce. Для повышения прав используйте повышение разрешений во время выполнения или при установке доверенного приложения.

## Руководство по Windows Forms

- Используйте частичное доверие, если это возможно. Частично доверенные приложения Windows уменьшают поверхность атаки приложения. Управляйте списком разрешений, необходимых вашему приложению, и запросите эти разрешения декларативно во время выполнения.
- Используйте развертывание ClickOnce. Для повышения прав используйте повышение разрешений во время выполнения или при установке доверенного приложения.

## Руководство по WCF

- Учтите, что единственный безопасный способ передачи запроса в RESTful сервисах — через метод `HTTP POST` с включённым TLS. Использование `HTTP GET` требует передачи данных в URL (например, в строке запроса), что делает их видимыми для пользователя, а также записывает их в историю браузера.
- Избегайте использования [BasicHttpBinding](https://docs.microsoft.com/en-us/dotnet/api/system.servicemodel.basichttpbinding?view=netframework-4.7.2), так как он не имеет конфигурации безопасности по умолчанию. Используйте [WSHttpBinding](https://docs.microsoft.com/en-us/dotnet/api/system.servicemodel.wshttpbinding?view=netframework-4.7.2) вместо него.
- Используйте как минимум два режима безопасности для вашего привязочного механизма. Безопасность сообщений включает защиту в заголовках. Транспортная безопасность подразумевает использование SSL. Комбинируйте оба, используя [TransportWithMessageCredential](https://docs.microsoft.com/en-us/dotnet/framework/wcf/samples/ws-transport-with-message-credential).
- Тестируйте вашу реализацию WCF с помощью фреймворков для тестирования безопасности, таких как [ZAP](https://www.zaproxy.org/).
