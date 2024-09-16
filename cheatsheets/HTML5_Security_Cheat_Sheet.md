# Шпаргалка по безопасности HTML5

## Введение

Эта шпаргалка служит руководством по безопасной реализации HTML5.

## API для коммуникации

### Веб-сообщения

Веб-сообщения (также известные как междоменные сообщения) предоставляют способ обмена сообщениями между документами из разных источников, что обычно безопаснее, чем множество хака, используемых ранее для этой задачи. Тем не менее, есть несколько рекомендаций, которые следует учитывать:

- При отправке сообщения явно указывайте ожидаемый источник в качестве второго аргумента для `postMessage`, а не `*`, чтобы предотвратить отправку сообщения на неизвестный источник после перенаправления или другого изменения источника целевого окна.
- Получающая страница **всегда** должна:
    - Проверять атрибут `origin` отправителя, чтобы убедиться, что данные поступают из ожидаемого источника.
    - Выполнять проверку входных данных на атрибуте `data` события, чтобы убедиться, что данные имеют нужный формат.
- Не предполагавайте, что у вас есть контроль над атрибутом `data`. Единственный [флэш Cross Site Scripting](Cross_Site_Scripting_Prevention_Cheat_Sheet.md) на странице отправителя позволяет злоумышленнику отправлять сообщения в любом формате.
- Обе страницы должны интерпретировать передаваемые сообщения только как **данные**. Никогда не оценивайте переданные сообщения как код (например, с помощью `eval()`) или не вставляйте их в DOM страницы (например, с помощью `innerHTML`), так как это создаст уязвимость DOM-based XSS. Для получения дополнительной информации см. [Шпаргалка по предотвращению DOM-based XSS](DOM_based_XSS_Prevention_Cheat_Sheet.md).
- Для присвоения значения данных элементу, вместо использования небезопасного метода, такого как `element.innerHTML=data;`, используйте более безопасный вариант: `element.textContent=data;`
- Тщательно проверяйте происхождение, чтобы оно точно совпадало с ожидаемыми FQDN. Обратите внимание, что следующий код: `if(message.origin.indexOf(".owasp.org")!=-1) { /* ... */ }` является очень небезопасным и не будет иметь желаемого поведения, так как `owasp.org.attacker.com` будет совпадать.
- Если вам нужно встроить внешний контент/ненадежные гаджеты и разрешить пользовательские скрипты (что крайне не рекомендуется), пожалуйста, ознакомьтесь с информацией о [песочницах](HTML5_Security_Cheat_Sheet.md#sandboxed-frames).

### Междоменный доступ к ресурсам

- Проверьте URL, передаваемые в `XMLHttpRequest.open`. Современные браузеры позволяют этим URL быть междоменными; это поведение может привести к инъекциям кода удаленным злоумышленником. Особое внимание уделите абсолютным URL.
- Убедитесь, что URL, отвечающие `Access-Control-Allow-Origin: *`, не содержат чувствительного контента или информации, которая может помочь злоумышленнику в последующих атаках. Используйте заголовок `Access-Control-Allow-Origin` только на выбранных URL, которым требуется доступ из другого домена. Не используйте заголовок для всего домена.
- Разрешайте только выбранные, доверенные домены в заголовке `Access-Control-Allow-Origin`. Предпочтительнее разрешать конкретные домены, чем блокировать или разрешать любой домен (не используйте `*` и не возвращайте содержимое заголовка `Origin` без проверки).
- Помните, что CORS не предотвращает передачу запрашиваемых данных на несанкционированное место. Важно, чтобы сервер выполнял обычную проверку [CSRF](Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.md).
- Хотя [Стандарт Fetch](https://fetch.spec.whatwg.org/#http-cors-protocol) рекомендует предварительный запрос с методом `OPTIONS`, текущие реализации могут не выполнять этот запрос, поэтому важно, чтобы "обычные" (`GET` и `POST`) запросы выполняли необходимое управление доступом.
- Отбрасывайте запросы, полученные по обычному HTTP, при наличии HTTPS-источников, чтобы предотвратить ошибки смешанного контента.
- Не полагайтесь только на заголовок Origin для проверки доступа. Браузер всегда отправляет этот заголовок в запросах CORS, но он может быть подделан вне браузера. Протоколы на уровне приложения должны использоваться для защиты чувствительных данных.

### WebSockets

- Откажитесь от поддержки старых версий протокола на клиентах/серверах и используйте только версии протокола выше hybi-00. Популярная версия Hixie-76 (hiby-00) и более старые устарели и небезопасны.
- Рекомендуемая версия, поддерживаемая в последних версиях всех современных браузеров, - [RFC 6455](http://tools.ietf.org/html/rfc6455) (поддерживается Firefox 11+, Chrome 16+, Safari 6, Opera 12.50 и IE10).
- Хотя относительно легко пробрасывать TCP-сервисы через WebSockets (например, VNC, FTP), это позволяет злоумышленнику в браузере получить доступ к этим проброшенным сервисам в случае атаки Cross Site Scripting. Эти сервисы также могут быть вызваны напрямую с вредоносной страницы или программы.
- Протокол не обрабатывает авторизацию и/или аутентификацию. Протоколы на уровне приложения должны обрабатывать это отдельно в случае передачи чувствительных данных.
- Обрабатывайте сообщения, полученные через WebSocket, как данные. Не пытайтесь присвоить их напрямую DOM или оценивать как код. Если ответ - JSON, никогда не используйте небезопасную функцию `eval()`; используйте безопасный вариант JSON.parse().
- Точки доступа, открытые через протокол `ws://`, легко могут быть преобразованы в простой текст. Используйте только `wss://` (WebSockets через SSL/TLS) для защиты от атак типа Man-In-The-Middle.
- Подделка клиента возможна вне браузера, поэтому сервер WebSockets должен быть способен обрабатывать некорректные/вредоносные входные данные. Всегда проверяйте входные данные, поступающие с удаленного сайта, так как они могли быть изменены.
- При реализации серверов проверяйте заголовок `Origin:` в рукопожатии WebSockets. Хотя он может быть подделан вне браузера, браузеры всегда добавляют Origin страницы, которая инициировала соединение WebSockets.
- Так как клиент WebSockets в браузере доступен через вызовы JavaScript, все коммуникации WebSockets могут быть подделаны или перехвачены через [Cross Site Scripting](https://owasp.org/www-community/attacks/xss/). Всегда проверяйте данные, поступающие через соединение WebSockets.

### События, отправленные сервером

- Проверяйте URL, передаваемые в конструктор `EventSource`, даже если разрешены только URL того же источника.
- Как упоминалось ранее, обрабатывайте сообщения (`event.data`) как данные и никогда не оценивайте их содержимое как HTML или скриптовый код.
- Всегда проверяйте атрибут происхождения сообщения (`event.origin`), чтобы убедиться, что сообщение поступает из доверенного домена. Используйте подход с белым списком.

## API для хранения

### Локальное хранилище

- Также известно как Offline Storage, Web Storage. Основной механизм хранения может варьироваться от одного пользовательского агента к другому. Другими словами, любая аутентификация, требуемая вашим приложением, может быть обойдена пользователем с локальными привилегиями на машине, на которой хранятся данные. Поэтому рекомендуется избегать хранения чувствительной информации в локальном хранилище, где предполагается аутентификация.
- Благодаря гарантиям безопасности браузера подходяще использовать локальное хранилище, когда доступ к данным не предполагает аутентификации или авторизации.
- Используйте объект sessionStorage вместо localStorage, если постоянное хранилище не требуется. Объект sessionStorage доступен только для этого окна/вкладки до тех пор, пока окно не будет закрыто.
- Один [Cross Site Scripting](https://owasp.org/www-community/attacks/xss/) может быть использован для кражи всех данных в этих объектах, поэтому снова рекомендуется не хранить чувствительную информацию в локальном хранилище.
- Один [Cross Site Scripting](https://owasp.org/www-community/attacks/xss/) может быть использован для загрузки вредоносных данных в эти объекты, поэтому не рассматривайте объекты в них как доверенные.
- Особое внимание уделяйте вызовам "localStorage.getItem" и "setItem", реализованным на странице HTML5. Это поможет обнаружить случаи, когда разработчики создают решения, которые помещают чувствительную информацию в локальное хранилище, что может быть серьезным риском, если аутентификация или авторизация к этим данным предполагается неверно.
- Не храните идентификаторы сеансов в локальном хранилище, так как данные всегда доступны через JavaScript. Cookies могут смягчить этот риск с помощью флага `httpOnly`.
- Нет способа ограничить видимость объекта до конкретного пути, как это делается с атрибутом path HTTP Cookies, каждый объект делится внутри источника и защищен Политикой одного источника. Избегайте размещения нескольких приложений на одном источнике, все они будут делиться одним и тем же объектом localStorage, используйте вместо этого разные субдомены.

### Клиентские базы данных

- В ноябре 2010 года W3C объявил Web SQL Database (реляционная SQL база данных) устаревшей спецификацией. Новый стандарт Indexed Database API или IndexedDB (ранее WebSimpleDB) активно разрабатывается, и предоставляет хранилище ключ-значение и методы для выполнения расширенных запросов.
- Основные механизмы хранения могут варьироваться от одного пользовательского агента к другому. Другими словами, любая аутентификация, требуемая вашим приложением, может быть обойдена пользователем с локальными привилегиями на машине, на которой хранятся данные. Поэтому рекомендуется не хранить чувствительную информацию в локальном хранилище.
- Если используется, контент WebDatabase на стороне клиента может быть уязвим к SQL-инъекциям и требует надлежащей проверки и параметризации.
- Как и локальное хранилище, один [Cross Site Scripting](https://owasp.org/www-community/attacks/xss/) может быть использован для загрузки вредоносных данных в веб-базу данных. Не рассматривайте данные в этих базах как доверенные.

## Геолокация

- [API Геолокации](https://www.w3.org/TR/2021/WD-geolocation-20211124/#security) требует, чтобы пользовательские агенты запрашивали разрешение пользователя перед вычислением местоположения. Как это решение запоминается, варьируется от браузера к браузеру. Некоторые пользовательские агенты требуют, чтобы пользователь снова посетил страницу, чтобы отключить возможность получения местоположения пользователя без запроса, поэтому по соображениям конфиденциальности рекомендуется требовать ввода пользователя перед вызовом `getCurrentPosition` или `watchPosition`.

## Web Workers

- Web Workers могут использовать объект `XMLHttpRequest` для выполнения запросов в домене и междоменных запросов. Смотрите соответствующий раздел этой Шпаргалки, чтобы обеспечить безопасность CORS.
- Хотя Web Workers не имеют доступа к DOM вызывающей страницы, вредоносные Web Workers могут использовать чрезмерное количество процессорного времени для вычислений, что может привести к отказу в обслуживании или злоупотреблению Cross Origin Resource Sharing для дальнейшей эксплуатации. Убедитесь, что код во всех скриптах Web Workers не является зловредным. Не разрешайте создание скриптов Web Worker из входных данных пользователя.
- Проверяйте сообщения, обмененные с Web Worker. Не пытайтесь обмениваться фрагментами JavaScript для оценки, например, с помощью `eval()`, так как это может создать [DOM Based XSS](DOM_based_XSS_Prevention_Cheat_Sheet.md) уязвимость.

## Tabnabbing

Атака описана в [этой статье](https://owasp.org/www-community/attacks/Reverse_Tabnabbing).

Вкратце, это способность действовать на содержимое или местоположение родительской страницы из вновь открытой страницы через ссылку "назад", которую предоставляет объект **opener** JavaScript.

Это относится к HTML-ссылке или функции JavaScript `window.open`, использующим атрибут/инструкцию `target`, чтобы указать [место загрузки](https://www.w3schools.com/tags/att_a_target.asp), которое не заменяет текущее местоположение, а затем делает текущее окно/вкладку доступным.

Чтобы предотвратить эту проблему, доступны следующие действия:

Отключите ссылку "назад" между родительскими и дочерними страницами:

- Для HTML-ссылок:
    - Чтобы отключить эту ссылку, добавьте атрибут `rel="noopener"` в тег, используемый для создания ссылки с родительской страницы на дочернюю страницу. Значение этого атрибута отключает ссылку, но в зависимости от браузера может оставить информацию о реферере в запросе к дочерней странице.
    - Чтобы также удалить информацию о реферере, используйте значение атрибута: `rel="noopener noreferrer"`.
- Для функции JavaScript `window.open`, добавьте значения `noopener,noreferrer` в параметр [windowFeatures](https://developer.mozilla.org/en-US/docs/Web/API/Window/open) функции `window.open`.

Так как поведение при использовании указанных элементов различается между браузерами, используйте HTML-ссылку или JavaScript для открытия окна (или вкладки), а затем используйте эту конфигурацию для максимальной кросс-браузерной поддержки:

- Для [HTML-ссылок](https://www.scaler.com/topics/html/html-links/), добавьте атрибут `rel="noopener noreferrer"` ко всем ссылкам.
- Для JavaScript используйте эту функцию для открытия окна (или вкладки):

```javascript
function openPopup(url, name, windowFeatures) {
  // Откройте всплывающее окно и установите инструкции по политике opener и реферера
  var newWindow = window.open(url, name, 'noopener,noreferrer,' + windowFeatures);
  // Сбросьте ссылку opener
  newWindow.opener = null;
}
```

- Добавьте заголовок HTTP-ответа `Referrer-Policy: no-referrer` ко всем HTTP-ответам, отправляемым приложением ([Информация о заголовке Referrer-Policy](https://owasp.org/www-project-secure-headers/)). Эта конфигурация обеспечит отсутствие информации о реферере в запросах со страницы.

Матрица совместимости:

- [noopener](https://caniuse.com/#search=noopener)
- [noreferrer](https://caniuse.com/#search=noreferrer)
- [referrer-policy](https://caniuse.com/#feat=referrer-policy)

## Сэндбоксированные фреймы

- Используйте атрибут `sandbox` у `iframe` для ненадежного контента.
- Атрибут `sandbox` у `iframe` включает ограничения на контент внутри `iframe`. При установке атрибута `sandbox` действуют следующие ограничения:
    1. Все разметки рассматриваются как относящиеся к уникальному источнику.
    2. Все формы и скрипты отключены.
    3. Все ссылки не могут нацеливаться на другие контексты просмотра.
    4. Все функции, которые срабатывают автоматически, заблокированы.
    5. Все плагины отключены.

Можно получить [тонкую настройку](https://html.spec.whatwg.org/multipage/iframe-embed-object.html#attr-iframe-sandbox) возможностей `iframe`, используя значение атрибута `sandbox`.

- В старых версиях пользовательских агентов, где эта функция не поддерживается, этот атрибут будет игнорироваться. Используйте эту функцию как дополнительный слой защиты или проверьте, поддерживает ли браузер сэндбоксированные фреймы, и показывайте ненадежный контент только в случае поддержки.
- Помимо этого атрибута, для предотвращения атак Clickjacking и нежелательного фрейминга рекомендуется использовать заголовок `X-Frame-Options`, который поддерживает значения `deny` и `same-origin`. Другие решения, такие как framebusting (`if(window!==window.top) { window.top.location=location;}`), не рекомендуются.

## Подсказки по вводу учетных данных и персонально идентифицируемой информации (PII)

- Защищайте значения ввода от кэширования браузером.

> Доступ к финансовому счету с общего компьютера. Даже если вы вышли из системы, следующий пользователь, использующий машину, может войти в систему из-за функции автозаполнения браузера. Чтобы смягчить это, мы говорим полям ввода не помогать в этом.

```html
<input type="text" spellcheck="false" autocomplete="off" autocorrect="off" autocapitalize="off"></input>
```

Текстовые области и поля ввода для PII (имя, электронная почта, адрес, номер телефона) и учетных данных для входа (имя пользователя, пароль) должны предотвращать хранение в браузере. Используйте эти атрибуты HTML5, чтобы предотвратить хранение PII в вашей форме:

- `spellcheck="false"`
- `autocomplete="off"`
- `autocorrect="off"`
- `autocapitalize="off"`

## Оффлайн-приложения

- Порядок, в котором пользовательский агент запрашивает разрешение у пользователя на хранение данных для оффлайн-просмотра и когда этот кэш удаляется, варьируется от одного браузера к другому. Отравление кэша представляет проблему, если пользователь подключается через небезопасные сети, поэтому по соображениям конфиденциальности рекомендуется требовать ввода пользователя перед отправкой любого файла `manifest`.
- Пользователи должны кэшировать только доверенные веб-сайты и очищать кэш после просмотра через открытые или небезопасные сети.

## Прогрессивные улучшения и риски плавной деградации

- Лучшая практика на данный момент — определить возможности, которые поддерживает браузер, и дополнить их некоторым заменителем для возможностей, которые не поддерживаются напрямую. Это может означать использование элемента, подобного луковице, например, переход к Flash Player, если тег `<video>` не поддерживается, или это может означать дополнительный скриптовый код из различных источников, который должен быть проверен на предмет безопасности.

## HTTP заголовки для повышения безопасности

Обратитесь к проекту [OWASP Secure Headers](https://owasp.org/www-project-secure-headers/), чтобы получить список HTTP заголовков безопасности, которые приложение должно использовать для включения защит на уровне браузера.

## Рекомендации по реализации WebSocket

В дополнение к вышеупомянутым элементам, приведен список областей, к которым нужно проявлять осторожность при реализации.

- Фильтрация доступа через HTTP заголовок "Origin"
- Валидация ввода / вывода
- Аутентификация
- Авторизация
- Явная аннулизация токенов доступа
- Конфиденциальность и целостность

Ниже будут предложены некоторые рекомендации по реализации для каждой области, а также приведен пример приложения, демонстрирующий все описанные пункты.

Полный исходный код примера приложения доступен [здесь](https://github.com/righettod/poc-websocket).

### Фильтрация доступа

Во время инициации канала WebSocket браузер отправляет HTTP заголовок **Origin**, который содержит исходный домен для запроса на установление соединения. Хотя этот заголовок можно подделать в поддельном HTTP запросе (не на базе браузера), его нельзя переопределить или принудительно изменить в контексте браузера. Поэтому он представляет собой хороший кандидат для применения фильтрации в соответствии с ожидаемым значением.

Пример атаки с использованием этого вектора, называемый *Cross-Site WebSocket Hijacking (CSWSH)*, описан [здесь](https://www.christian-schneider.net/CrossSiteWebSocketHijacking.html).

Код ниже определяет конфигурацию, которая применяет фильтрацию на основе "белого списка" источников. Это гарантирует, что только разрешенные источники могут установить полное соединение:

```java
import org.owasp.encoder.Encode;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.websocket.server.ServerEndpointConfig;
import java.util.Arrays;
import java.util.List;

/**
 * Настройка правил рукопожатия, применяемых ко всем конечным точкам WebSocket приложения.
 * Используется для настройки фильтрации доступа с использованием HTTP заголовка "Origin" в качестве входной информации.
 *
 * @see "http://docs.oracle.com/javaee/7/api/index.html?javax/websocket/server/
 * ServerEndpointConfig.Configurator.html"
 * @see "https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin"
 */
public class EndpointConfigurator extends ServerEndpointConfig.Configurator {

    /**
     * Логгер
     */
    private static final Logger LOG = LoggerFactory.getLogger(EndpointConfigurator.class);

    /**
     * Получение ожидаемых исходных доменов из свойства JVM для возможности внешней конфигурации
     */
    private static final List<String> EXPECTED_ORIGINS =  Arrays.asList(System.getProperty("source.origins")
                                                          .split(";"));

    /**
     * {@inheritDoc}
     */
    @Override
    public boolean checkOrigin(String originHeaderValue) {
        boolean isAllowed = EXPECTED_ORIGINS.contains(originHeaderValue);
        String safeOriginValue = Encode.forHtmlContent(originHeaderValue);
        if (isAllowed) {
            LOG.info("[EndpointConfigurator] Новый запрос на рукопожатие получен от {} и был принят.",
                      safeOriginValue);
        } else {
            LOG.warn("[EndpointConfigurator] Новый запрос на рукопожатие получен от {} и был отклонен!",
                      safeOriginValue);
        }
        return isAllowed;
    }

}
```

### Аутентификация и валидация ввода/вывода

При использовании WebSocket в качестве канала связи важно применять метод аутентификации, который позволяет пользователю получать токен доступа, который не отправляется автоматически браузером и затем должен быть явно отправлен клиентским кодом при каждом обмене.

HMAC дайджесты являются самым простым методом, а [JSON Web Token](https://jwt.io/introduction/) является хорошей альтернативой с богатыми возможностями, так как он позволяет передавать информацию о доступе в безгосударственном и неизменяемом виде. Более того, он определяет срок действия токена. Дополнительную информацию о усилении безопасности JWT можно найти в этой [шпаргалке](JSON_Web_Token_for_Java_Cheat_Sheet.md).

[JSON Validation Schema](http://json-schema.org/) используются для определения и проверки ожидаемого содержания во входных и выходных сообщениях.

Ниже представлен код, который определяет полный поток обработки сообщений аутентификации:

**Точка подключения WebSocket для аутентификации** - Обеспечивает WS конечную точку, которая позволяет обмениваться данными для аутентификации

```java
import org.owasp.pocwebsocket.configurator.EndpointConfigurator;
import org.owasp.pocwebsocket.decoder.AuthenticationRequestDecoder;
import org.owasp.pocwebsocket.encoder.AuthenticationResponseEncoder;
import org.owasp.pocwebsocket.handler.AuthenticationMessageHandler;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.websocket.CloseReason;
import javax.websocket.OnClose;
import javax.websocket.OnError;
import javax.websocket.OnOpen;
import javax.websocket.Session;
import javax.websocket.server.ServerEndpoint;

/**
 * Класс, ответственный за управление аутентификацией клиента.
 *
 * @see "http://docs.oracle.com/javaee/7/api/javax/websocket/server/ServerEndpointConfig.Configurator.html"
 * @see "http://svn.apache.org/viewvc/tomcat/trunk/webapps/examples/WEB-INF/classes/websocket/"
 */
@ServerEndpoint(value = "/auth", configurator = EndpointConfigurator.class,
subprotocols = {"authentication"}, encoders = {AuthenticationResponseEncoder.class},
decoders = {AuthenticationRequestDecoder.class})
public class AuthenticationEndpoint {

    /**
     * Логгер
     */
    private static final Logger LOG = LoggerFactory.getLogger(AuthenticationEndpoint.class);

    /**
     * Обработка начала обмена
     *
     * @param session Информация о сессии обмена
     */
    @OnOpen
    public void start(Session session) {
        // Определение тайм-аута бездействия соединения и ограничений на размер сообщений для минимизации
        // атак типа DOS, связанных с массовым открытием соединений или отправкой больших сообщений
        int msgMaxSize = 1024 * 1024; // 1 МБ
        session.setMaxIdleTimeout(60000); // 1 минута
        session.setMaxTextMessageBufferSize(msgMaxSize);
        session.setMaxBinaryMessageBufferSize(msgMaxSize);
        // Логирование начала обмена
        LOG.info("[AuthenticationEndpoint] Сессия {} началась", session.getId());
        // Присвоение нового обработчика сообщений для обработки обмена
        session.addMessageHandler(new AuthenticationMessageHandler(session.getBasicRemote()));
        LOG.info("[AuthenticationEndpoint] Обработчик сообщений для сессии {} назначен для обработки",
                  session.getId());
    }

    /**
     * Обработка ошибок
     *
     * @param session Информация о сессии обмена
     * @param thr     Подробности об ошибке
     */
    @OnError
    public void onError(Session session, Throwable thr) {
        LOG.error("[AuthenticationEndpoint] Ошибка в сессии {}", session.getId(), thr);
    }

    /**
     * Обработка события закрытия
     *
     * @param session     Информация о сессии обмена
     * @param closeReason Причина закрытия обмена
     */
    @OnClose
    public void onClose(Session session, CloseReason closeReason) {
        LOG.info("[AuthenticationEndpoint] Сессия {} закрыта: {}", session.getId(),
                  closeReason.getReasonPhrase());
    }

}
```

**Обработчик сообщений аутентификации** - Обработка всех запросов на аутентификацию

```java
import org.owasp.pocwebsocket.enumeration.AccessLevel;
import org.owasp.pocwebsocket.util.AuthenticationUtils;
import org.owasp.pocwebsocket.vo.AuthenticationRequest;
import org.owasp.pocwebsocket.vo.AuthenticationResponse;
import org.owasp.encoder.Encode;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.websocket.EncodeException;
import javax.websocket.MessageHandler;
import javax.websocket.RemoteEndpoint;
import java.io.IOException;

/**
 * Обрабатывает поток сообщений аутентификации
 */
public class AuthenticationMessageHandler implements MessageHandler.Whole<AuthenticationRequest> {

    private static final Logger LOG = LoggerFactory.getLogger(AuthenticationMessageHandler.class);

    /**
     * Ссылка на канал связи с клиентом
     */
    private RemoteEndpoint.Basic clientConnection;

    /**
     * Конструктор
     *
     * @param clientConnection Ссылка на канал связи с клиентом
     */
    public AuthenticationMessageHandler(RemoteEndpoint.Basic clientConnection) {
        this.clientConnection = clientConnection;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public void onMessage(AuthenticationRequest message) {
        AuthenticationResponse response = null;
        try {
            // Аутентификация
            String authenticationToken = "";
            String accessLevel = this.authenticate(message.getLogin(), message.getPassword());
            if (accessLevel != null) {
                // Создание простого JSON токена, представляющего профиль аутентификации
                authenticationToken = AuthenticationUtils.issueToken(message.getLogin(), accessLevel);
            }
            // Формирование объекта ответа
            String safeLoginValue = Encode.forHtmlContent(message.getLogin());
            if (!authenticationToken.isEmpty()) {
                response = new AuthenticationResponse(true, authenticationToken, "Аутентификация успешна!");
                LOG.info("[AuthenticationMessageHandler] Пользователь {} аутентифицирован успешно.", safeLoginValue);
            } else {
                response = new AuthenticationResponse(false, authenticationToken, "Ошибка аутентификации!");
                LOG.warn("[AuthenticationMessageHandler] Ошибка аутентификации пользователя {}.", safeLoginValue);
            }
        } catch (Exception e) {
            LOG.error("[AuthenticationMessageHandler] Ошибка в процессе аутентификации.", e);
            // Формирование объекта ответа, указывающего на сбой аутентификации
            response = new AuthenticationResponse(false, "", "Ошибка аутентификации!");
        } finally {
            // Отправка ответа
            try {
                this.clientConnection.sendObject(response);
            } catch (IOException | EncodeException e) {
                LOG.error("[AuthenticationMessageHandler] Ошибка при отправке объекта ответа.", e);
            }
        }
    }

    /**
     * Аутентификация пользователя
     *
     * @param login    Логин пользователя
     * @param password Пароль пользователя
     * @return Уровень доступа, если аутентификация прошла успешно, или NULL, если аутентификация не удалась
     */
    private String authenticate(String login, String password) {
        ...
    }
}
```

**Утилитный класс для управления JWT** - Обработка выдачи и проверки токенов доступа. В примере использован простой JWT, здесь сосредоточено внимание на реализации глобальной WS конечной точки, без дополнительного усиления безопасности (см. эту [шпаргалку](JSON_Web_Token_for_Java_Cheat_Sheet.md) для применения дополнительных мер защиты JWT)

```java
import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTVerifier;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.interfaces.DecodedJWT;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Calendar;
import java.util.Locale;

/**
 * Утилитный класс для управления JWT токенами аутентификации
 */
public class AuthenticationUtils {

    /**
     * Создает JWT токен для пользователя
     *
     * @param login       Логин пользователя
     * @param accessLevel Уровень доступа пользователя
     * @return Закодированный в Base64 JWT токен
     * @throws Exception Если возникает ошибка при создании токена
     */
    public static String issueToken(String login, String accessLevel) throws Exception {
        // Создание JWT токена с действительностью 30 минут
        Algorithm algorithm = Algorithm.HMAC256(loadSecret());
        Calendar c = Calendar.getInstance();
        c.add(Calendar.MINUTE, 30);
        return JWT.create()
                  .withIssuer("WEBSOCKET-SERVER")
                  .withSubject(login)
                  .withExpiresAt(c.getTime())
                  .withClaim("access_level", accessLevel.trim().toUpperCase(Locale.US))
                  .sign(algorithm);
    }

    /**
     * Проверяет действительность предоставленного JWT токена
     *
     * @param token Закодированный JWT токен для проверки
     * @return Проверенный и декодированный токен с информацией о пользователе и
     * авторизации (уровень доступа)
     * @throws Exception Если возникает ошибка при проверке токена
     */
    public static DecodedJWT validateToken(String token) throws Exception {
        Algorithm algorithm = Algorithm.HMAC256(loadSecret());
        JWTVerifier verifier = JWT.require(algorithm).withIssuer("WEBSOCKET-SERVER").build();
        return verifier.verify(token);
    }

    /**
     * Загружает секрет JWT, используемый для подписи токена, используя байтовый массив для хранения секрета
     * с целью избегания постоянного хранения строки в памяти
     *
     * @return Секрет в виде байтового массива
     * @throws IOException Если возникает ошибка при загрузке секрета
     */
    private static byte[] loadSecret() throws IOException {
        return Files.readAllBytes(Paths.get("src", "main", "resources", "jwt-secret.txt"));
    }
}
```

**JSON схема входного и выходного сообщения аутентификации** - Определяет ожидаемую структуру входных и выходных сообщений с точки зрения конечной точки аутентификации.

```json
{
    "$schema": "http://json-schema.org/schema#",
    "title": "AuthenticationRequest",
    "type": "object",
    "properties": {
        "login": {
            "type": "string",
            "pattern": "^[a-zA-Z]{1,10}$"
        },
        "password": {
            "type": "string"
        }
    },
    "required": [
        "login",
        "password"
    ]
}

{
    "$schema": "http://json-schema.org/schema#",
    "title": "AuthenticationResponse",
    "type": "object",
    "properties": {
        "isSuccess": {
            "type": "boolean"
        },
        "token": {
            "type": "string",
            "pattern": "^[a-zA-Z0-9+/=\\._-]{0,500}$"
        },
        "message": {
            "type": "string",
            "pattern": "^[a-zA-Z0-9!\\s]{0,100}$"
        }
    },
    "required": [
        "isSuccess",
        "token",
        "message"
    ]
}
```

**Декодер и кодировщик сообщений аутентификации** - Выполняет сериализацию/десериализацию JSON и проверку входных/выходных данных с использованием специальной JSON-схемы. Это позволяет систематически обеспечивать, чтобы все сообщения, полученные и отправленные конечной точкой, строго соответствовали ожидаемой структуре и содержимому.

``` java
import com.fasterxml.jackson.databind.JsonNode;
import com.github.fge.jackson.JsonLoader;
import com.github.fge.jsonschema.core.exceptions.ProcessingException;
import com.github.fge.jsonschema.core.report.ProcessingReport;
import com.github.fge.jsonschema.main.JsonSchema;
import com.github.fge.jsonschema.main.JsonSchemaFactory;
import com.google.gson.Gson;
import org.owasp.pocwebsocket.vo.AuthenticationRequest;

import javax.websocket.DecodeException;
import javax.websocket.Decoder;
import javax.websocket.EndpointConfig;
import java.io.File;
import java.io.IOException;

/**
 * Декодирует текстовое представление JSON в объект AuthenticationRequest
 * <p>
 * Поскольку существует один экземпляр класса декодера на сеанс конечной точки, можно использовать
 * JsonSchema в качестве переменной экземпляра декодера.
 */
public class AuthenticationRequestDecoder implements Decoder.Text<AuthenticationRequest> {

    /**
     * JSON-схема проверки, связанная с этим типом сообщения
     */
    private JsonSchema validationSchema = null;

    /**
     * Инициализирует декодер и связанную JSON-схему проверки
     *
     * @throws IOException Если возникает ошибка при создании объекта
     * @throws ProcessingException Если возникает ошибка при загрузке схемы
     */
    public AuthenticationRequestDecoder() throws IOException, ProcessingException {
        JsonNode node = JsonLoader.fromFile(
                        new File("src/main/resources/authentication-request-schema.json"));
        this.validationSchema = JsonSchemaFactory.byDefault().getJsonSchema(node);
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public AuthenticationRequest decode(String s) throws DecodeException {
        try {
            // Проверка предоставленного представления по отношению к специальной схеме
            // Используйте режим проверки с отчетом, чтобы включить дальнейшее исследование/отслеживание
            // деталей ошибок
            // Более того, метод проверки "validInstance()" генерирует NullPointerException
            // если представление не соответствует ожидаемой схеме
            // поэтому правильнее использовать метод проверки с отчетом
            ProcessingReport validationReport = this.validationSchema.validate(JsonLoader.fromString(s),
                                                                               true);
            // Убедитесь, что нет ошибок
            if (!validationReport.isSuccess()) {
                // Просто отклоните сообщение здесь: не обращая внимания на детали ошибок...
                throw new DecodeException(s, "Проверка предоставленного представления не удалась!");
            }
        } catch (IOException | ProcessingException e) {
            throw new DecodeException(s, "Не удалось проверить предоставленное представление на валидность JSON!", e);
        }

        return new Gson().fromJson(s, AuthenticationRequest.class);
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public boolean willDecode(String s) {
        boolean canDecode = false;

        // Если предоставленное JSON-представление пустое/нулевое, то мы указываем, что
        // представление не может быть декодировано в наш ожидаемый объект
        if (s == null || s.trim().isEmpty()) {
            return canDecode;
        }

        // Попробуйте преобразовать предоставленное JSON-представление в наш объект, чтобы проверить хотя бы
        // структуру (проверка содержания проводится во время декодирования)
        try {
            AuthenticationRequest test = new Gson().fromJson(s, AuthenticationRequest.class);
            canDecode = (test != null);
        } catch (Exception e) {
            // Явно игнорируйте любые ошибки преобразования...
        }

        return canDecode;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public void init(EndpointConfig config) {
        // Не используется
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public void destroy() {
        // Не используется
    }
}
```

``` java
import com.fasterxml.jackson.databind.JsonNode;
import com.github.fge.jackson.JsonLoader;
import com.github.fge.jsonschema.core.exceptions.ProcessingException;
import com.github.fge.jsonschema.core.report.ProcessingReport;
import com.github.fge.jsonschema.main.JsonSchema;
import com.github.fge.jsonschema.main.JsonSchemaFactory;
import com.google.gson.Gson;
import org.owasp.pocwebsocket.vo.AuthenticationResponse;

import javax.websocket.EncodeException;
import javax.websocket.Encoder;
import javax.websocket.EndpointConfig;
import java.io.File;
import java.io.IOException;

/**
 * Кодирует объект AuthenticationResponse в текстовое представление JSON.
 * <p>
 * Поскольку существует один экземпляр класса кодировщика на сеанс конечной точки, можно использовать
 * JsonSchema в качестве переменной экземпляра кодировщика.
 */
public class AuthenticationResponseEncoder implements Encoder.Text<AuthenticationResponse> {

    /**
     * JSON-схема проверки, связанная с этим типом сообщения
     */
    private JsonSchema validationSchema = null;

    /**
     * Инициализирует кодировщик и связанную JSON-схему проверки
     *
     * @throws IOException Если возникает ошибка при создании объекта
     * @throws ProcessingException Если возникает ошибка при загрузке схемы
     */
    public AuthenticationResponseEncoder() throws IOException, ProcessingException {
        JsonNode node = JsonLoader.fromFile(
                        new File("src/main/resources/authentication-response-schema.json"));
        this.validationSchema = JsonSchemaFactory.byDefault().getJsonSchema(node);
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public String encode(AuthenticationResponse object) throws EncodeException {
        // Генерирует представление JSON
        String json = new Gson().toJson(object);
        try {
            // Проверяет сгенерированное представление по отношению к специальной схеме
            // Используйте режим проверки с отчетом, чтобы включить дальнейшее исследование/отслеживание
            // деталей ошибок
            // Более того, метод проверки "validInstance()" генерирует NullPointerException
            // если представление не соответствует ожидаемой схеме
            // поэтому правильнее использовать метод проверки с отчетом
            ProcessingReport validationReport = this.validationSchema.validate(JsonLoader.fromString(json),
                                                                                true);
            // Убедитесь, что нет ошибок
            if (!validationReport.isSuccess()) {
                // Просто отклоните сообщение здесь: не обращая внимания на детали ошибок...
                throw new EncodeException(object, "Проверка сгенерированного представления не удалась!");
            }
        } catch (IOException | ProcessingException e) {
            throw new EncodeException(object, "Не удалось проверить сгенерированное представление на валидность JSON!", e);
        }

        return json;
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public void init(EndpointConfig config) {
        // Не используется
    }

    /**
     * {@inheritDoc}
     */
    @Override
    public void destroy() {
        // Не используется
    }

}
```

Обратите внимание, что тот же подход используется в части обработки сообщений POC. Все сообщения, обмененные между клиентом и сервером, систематически проверяются таким же способом, с использованием выделенных JSON-схем, связанных с кодировщиками/декодировщиками сообщений (сериализация/десериализация).

### Авторизация и явное аннулирование токена доступа

Информация об авторизации хранится в токене доступа с использованием функции JWT *Claim* (в POC имя этого утверждения — *access_level*). Авторизация проверяется при получении запроса и перед выполнением любых других действий, использующих информацию от пользователя.

Токен доступа передается с каждым сообщением, отправленным в конечную точку сообщений, и используется черный список, чтобы позволить пользователю запросить явную аннулирование токена.

Явная аннулирование токена интересно с точки зрения пользователя, поскольку, когда используются токены, срок их действия часто довольно длинный (обычно срок действия превышает 1 час), поэтому важно предоставить пользователю способ сообщить системе "Хорошо, я завершил обмен с вами, так что вы можете закрыть нашу сессию обмена и очистить связанные ссылки".

Это также помогает пользователю отозвать текущий доступ, если обнаружен злонамеренный одновременный доступ с использованием того же токена (случай кражи токена).

**Черный список токенов** - Поддерживайте временный список, используя кэширование в памяти с ограничением по времени для хэшей токенов, которые больше не могут использоваться

``` java
import org.apache.commons.jcs.JCS;
import org.apache.commons.jcs.access.CacheAccess;
import org.apache.commons.jcs.access.exception.CacheException;

import javax.xml.bind.DatatypeConverter;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

/**
 * Утилитный класс для управления токенами доступа, которые были объявлены как больше не подлежащие использованию (явный выход пользователя)
 */
public class AccessTokenBlocklistUtils {
    /**
     * Содержимое сообщения, отправленного пользователем, которое указывает, что токен доступа,
     * сопровождающий сообщение, должен быть добавлен в черный список для дальнейшего использования
     */
    public static final String MESSAGE_ACCESS_TOKEN_INVALIDATION_FLAG = "INVALIDATE_TOKEN";

    /**
     * Используйте кэш для хранения хэшей токенов в черном списке, чтобы избежать исчерпания памяти и быть последовательным,
     * поскольку токены действительны 30 минут, элементы остаются в кэше 60 минут
     */
    private static final CacheAccess<String, String> TOKEN_CACHE;

    static {
        try {
            TOKEN_CACHE = JCS.getInstance("default");
        } catch (CacheException e) {
            throw new RuntimeException("Не удалось инициализировать кэш токенов!", e);
        }
    }

    /**
     * Добавить токен в черный список
     *
     * @param token Токен, хэш которого должен быть добавлен
     * @throws NoSuchAlgorithmException Если SHA256 недоступен
     */
    public static void addToken(String token) throws NoSuchAlgorithmException {
        if (token != null && !token.trim().isEmpty()) {
            String hashHex = computeHash(token);
            if (TOKEN_CACHE.get(hashHex) == null) {
                TOKEN_CACHE.putSafe(hashHex, hashHex);
            }
        }
    }

    /**
     * Проверить, присутствует ли токен в черном списке
     *
     * @param token Токен, присутствие хэша которого необходимо проверить
     * @return TRUE, если токен в черном списке
     * @throws NoSuchAlgorithmException Если SHA256 недоступен
     */
    public static boolean isBlocklisted(String token) throws NoSuchAlgorithmException {
        boolean exists = false;
        if (token != null && !token.trim().isEmpty()) {
            String hashHex = computeHash(token);
            exists = (TOKEN_CACHE.get(hashHex) != null);
        }
        return exists;
    }

    /**
     * Вычислить SHA256 хэш токена
     *
     * @param token Токен, для которого необходимо вычислить хэш
     * @return Хэш в HEX-кодировке
     * @throws NoSuchAlgorithmException Если SHA256 недоступен
     */
    private static String computeHash(String token) throws NoSuchAlgorithmException {
        String hashHex = null;
        if (token != null && !token.trim().isEmpty()) {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] hash = md.digest(token.getBytes());
            hashHex = DatatypeConverter.printHexBinary(hash);
        }
        return hashHex;
    }

}
```

**Обработка сообщений** - Обработка запроса от пользователя на добавление сообщения в список. Пример подхода к проверке авторизации.

``` java
import com.auth0.jwt.interfaces.Claim;
import com.auth0.jwt.interfaces.DecodedJWT;
import org.owasp.pocwebsocket.enumeration.AccessLevel;
import org.owasp.pocwebsocket.util.AccessTokenBlocklistUtils;
import org.owasp.pocwebsocket.util.AuthenticationUtils;
import org.owasp.pocwebsocket.util.MessageUtils;
import org.owasp.pocwebsocket.vo.MessageRequest;
import org.owasp.pocwebsocket.vo.MessageResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.websocket.EncodeException;
import javax.websocket.RemoteEndpoint;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

/**
 * Обработка потока сообщений
 */
public class MessageHandler implements javax.websocket.MessageHandler.Whole<MessageRequest> {

    private static final Logger LOG = LoggerFactory.getLogger(MessageHandler.class);

    /**
     * Ссылка на канал связи с клиентом
     */
    private RemoteEndpoint.Basic clientConnection;

    /**
     * Конструктор
     *
     * @param clientConnection Ссылка на канал связи с клиентом
     */
    public MessageHandler(RemoteEndpoint.Basic clientConnection) {
        this.clientConnection = clientConnection;
    }


    /**
     * {@inheritDoc}
     */
    @Override
    public void onMessage(MessageRequest message) {
        MessageResponse response = null;
        try {
            /*Шаг 1: Проверка токена*/
            String token = message.getToken();
            // Проверка наличия токена в черном списке
            if (AccessTokenBlocklistUtils.isBlocklisted(token)) {
                throw new IllegalAccessException("Токен находится в черном списке!");
            }

            // Проверка подписи токена
            DecodedJWT decodedToken = AuthenticationUtils.validateToken(token);

            /*Шаг 2: Проверка авторизации (уровень доступа)*/
            Claim accessLevel = decodedToken.getClaim("access_level");
            if (accessLevel == null || AccessLevel.valueOf(accessLevel.asString()) == null) {
                throw new IllegalAccessException("Токен содержит недопустимый уровень доступа!");
            }

            /*Шаг 3: Выполнение ожидаемой обработки*/
            // Инициализация списка сообщений для текущего пользователя
            if (!MessageUtils.MESSAGES_DB.containsKey(decodedToken.getSubject())) {
                MessageUtils.MESSAGES_DB.put(decodedToken.getSubject(), new ArrayList<>());
            }

            // Добавление сообщения в список сообщений пользователя, если это не запрос на аннулирование токена
            // В противном случае добавление токена в черный список
            if (AccessTokenBlocklistUtils.MESSAGE_ACCESS_TOKEN_INVALIDATION_FLAG
                .equalsIgnoreCase(message.getContent().trim())) {
                AccessTokenBlocklistUtils.addToken(message.getToken());
            } else {
                MessageUtils.MESSAGES_DB.get(decodedToken.getSubject()).add(message.getContent());
            }

            // В зависимости от уровня доступа пользователя возвращать только его сообщения или все сообщения
            List<String> messages = new ArrayList<>();
            if (accessLevel.asString().equals(AccessLevel.USER.name())) {
                MessageUtils.MESSAGES_DB.get(decodedToken.getSubject())
                .forEach(s -> messages.add(String.format("(%s): %s", decodedToken.getSubject(), s)));
            } else if (accessLevel.asString().equals(AccessLevel.ADMIN.name())) {
                MessageUtils.MESSAGES_DB.forEach((k, v) ->
                v.forEach(s -> messages.add(String.format("(%s): %s", k, s))));
            }

            // Формирование объекта ответа, указывающего на успешное завершение обмена
            if (AccessTokenBlocklistUtils.MESSAGE_ACCESS_TOKEN_INVALIDATION_FLAG
                .equalsIgnoreCase(message.getContent().trim())) {
                response = new MessageResponse(true, messages, "Токен добавлен в черный список");
            } else {
                response = new MessageResponse(true, messages, "");
            }

        } catch (Exception e) {
            LOG.error("[MessageHandler] Ошибка в процессе обмена.", e);
            // Формирование объекта ответа, указывающего на ошибку обмена
            // Мы отправляем детали ошибки клиенту, потому что это POC (в реальном приложении это не так)
            response = new MessageResponse(false, new ArrayList<>(), "Ошибка в процессе обмена: "
                       + e.getMessage());
        } finally {
            // Отправка ответа
            try {
                this.clientConnection.sendObject(response);
            } catch (IOException | EncodeException e) {
                LOG.error("[MessageHandler] Ошибка при отправке объекта ответа.", e);
            }
        }
    }
}
```

### Конфиденциальность и Целостность

Если используется исходная версия протокола (протокол `ws://`), то передаваемые данные подвержены перехвату и потенциальной изменению в процессе передачи.

Пример захвата с помощью [Wireshark](https://www.wireshark.org/) и поиска обменов паролями в сохраненном файле PCAP, символы, не печатаемые в командном результате, были явно удалены:

``` shell
$ grep -aE '(password)' capture.pcap
{"login":"bob","password":"bob123"}
```

Существует способ проверки на уровне WebSocket конечной точки, является ли канал защищенным, вызвав метод `isSecure()` на объекте *session*.

Пример реализации в методе конечной точки, отвечающей за установку сеанса и воздействующей на обработчик сообщений:

``` java
/**
 * Обработка начала обмена
 *
 * @param session Информация о сеансе обмена
 */
@OnOpen
public void start(Session session) {
    ...
    // Присвоить новый экземпляр обработчика сообщений для обработки обмена только в случае, если канал защищен
    if(session.isSecure()) {
        session.addMessageHandler(new AuthenticationMessageHandler(session.getBasicRemote()));
    } else {
        LOG.info("[AuthenticationEndpoint] Сеанс {} не использует защищенный канал, поэтому обработчик сообщений " +
                 "не был назначен для обработки, и сеанс был явно закрыт!", session.getId());
        try {
            session.close(new CloseReason(CloseReason.CloseCodes.CANNOT_ACCEPT, "Использован небезопасный канал!"));
        } catch (IOException e) {
            LOG.error("[AuthenticationEndpoint] Сеанс {} не может быть явно закрыт!", session.getId(), e);
        }
    }
    LOG.info("[AuthenticationEndpoint] Обработчик сообщений назначен для обработки сеанса {}", session.getId());
}
```

Открывайте WebSocket конечные точки только на протоколе [wss://](https://kaazing.com/html5-websocket-security-is-strong/) (WebSocket через SSL/TLS), чтобы обеспечить *Конфиденциальность* и *Целостность* трафика, как и использование HTTP через SSL/TLS для защиты HTTP обменов.
