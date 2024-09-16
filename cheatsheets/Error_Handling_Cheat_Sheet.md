## Шпаргалка по Обработке Ошибок

### Введение

Обработка ошибок — это важная часть общей безопасности приложения. Как правило, атака начинается с фазы **Разведки**, когда злоумышленник пытается собрать как можно больше технической информации (например, *имя* и *версия* серверов, фреймворков и библиотек) о цели, таких как сервер приложений, используемые фреймворки и библиотеки.

Неперехваченные ошибки могут помочь злоумышленнику на этом этапе, что крайне важно для последующих атак.

Следующая [ссылка](https://web.archive.org/web/20230929111320/https://cipher.com/blog/a-complete-guide-to-the-phases-of-penetration-testing/) описывает различные фазы атаки.

### Контекст

Ошибки в обработке могут раскрыть много информации о цели, а также использоваться для выявления точек инъекции в функциональности приложения.

Пример раскрытия стека технологий (версий Struts2 и Tomcat) через исключение, отображаемое пользователю:

```text
HTTP Status 500 - For input string: "null"

type Exception report

message For input string: "null"

description The server encountered an internal error that prevented it from fulfilling this request.

exception

java.lang.NumberFormatException: For input string: "null"
    java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
    java.lang.Integer.parseInt(Integer.java:492)
    java.lang.Integer.parseInt(Integer.java:527)
    sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    ...
note: The full stack trace of the root cause is available in the Apache Tomcat/7.0.56 logs.
```

Пример раскрытия SQL-ошибки с путём установки сайта, что может быть использовано для нахождения уязвимых точек инъекции:

```text
Warning: odbc_fetch_array() expects parameter /1 to be resource, boolean given
in D:\app\index_new.php on line 188
```

[Руководство по тестированию OWASP](https://owasp.org/www-project-web-security-testing-guide/stable/4-Web_Application_Security_Testing/01-Information_Gathering/) предоставляет различные методы получения технической информации из приложения.

### Цель

Настройка глобального обработчика ошибок должна быть частью конфигурации выполнения вашего приложения. При возникновении непредвиденной ошибки приложение должно возвращать пользователю общий ответ, а детали ошибки логироваться на сервере для последующего расследования.

Ниже приведена схема предлагаемого подхода:

![Обзор](../assets/Error_Handling_Cheat_Sheet_Overview.png)

Поскольку большинство современных приложений основано на *API*, в данной статье предполагается, что бэкенд предоставляет только REST API и не содержит пользовательского интерфейса. Приложение должно стараться охватывать все возможные режимы отказа и использовать ошибки 5xx только для обозначения того, что запрос не может быть выполнен, при этом не предоставляя никакой информации, раскрывающей детали реализации. Для этого [RFC 7807 - Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807) определяет формат документа.  
Для самой операции логирования ошибок используйте [шпаргалку по логированию](Logging_Cheat_Sheet.md). Данная статья фокусируется на части обработки ошибок.

### Предложение

Для каждого стека технологий предлагаются следующие варианты конфигурации:

#### Стандартное веб-приложение на Java

Для таких приложений глобальный обработчик ошибок можно настроить на уровне дескриптора развертывания **web.xml**.

Пример конфигурации, которая может использоваться с версией спецификации Servlet *2.5* и выше.

С этой конфигурацией любая непредвиденная ошибка приведёт к перенаправлению на страницу **error.jsp**, где ошибка будет зарегистрирована, а пользователю вернётся общий ответ.

Конфигурация перенаправления в файле **web.xml**:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" ns="http://java.sun.com/xml/ns/javaee"
xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
version="3.0">
...
    <error-page>
        <exception-type>java.lang.Exception</exception-type>
        <location>/error.jsp</location>
    </error-page>
...
</web-app>
```

Содержимое файла **error.jsp**:

```java
<%@ page language="java" isErrorPage="true" contentType="application/json; charset=UTF-8"
    pageEncoding="UTF-8"%>
<%
String errorMessage = exception.getMessage();
// Логируем исключение через переменную "exception"
// ...
// Строим общий ответ в формате JSON, так как мы в контексте REST API приложения
// Также добавляем заголовок HTTP-ответа, чтобы указать клиенту, что произошла ошибка
response.setHeader("X-ERROR", "true");
// Используем ответ с кодом ошибки сервера
response.setStatus(500);
%>
{"message":"Произошла ошибка, повторите попытку"}
```

### Веб-приложение Java SpringMVC/SpringBoot

В SpringMVC или SpringBoot можно определить глобальный обработчик ошибок, реализовав следующий класс в вашем проекте. В Spring Framework 6 была введена поддержка обработки ошибок в соответствии с [RFC 7807](https://github.com/spring-projects/spring-framework/issues/27052) (Problem Details for HTTP APIs).

Мы указываем обработчику с помощью аннотации [@ExceptionHandler](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/bind/annotation/ExceptionHandler.html), что он должен обрабатывать любые исключения, наследуемые от класса *java.lang.Exception*, которые выбрасываются в приложении. Мы также используем класс [ProblemDetail](https://docs.spring.io/spring-framework/docs/6.0.0/javadoc-api/org/springframework/http/ProblemDetail.html) для создания объекта ответа.

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.context.request.WebRequest;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

/**
 * Глобальный обработчик ошибок, отвечающий за возврат общего ответа в случае неожиданных ошибок.
 */
@RestControllerAdvice
public class RestResponseEntityExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(value = {Exception.class})
    public ProblemDetail handleGlobalError(RuntimeException exception, WebRequest request) {
        // Логирование исключения через параметр "exception"
        // ...
        // Используем ответ с ошибкой сервера
        // В некоторых случаях разумно возвращать 4xx ошибки для некорректных запросов
        // Согласно спецификации, content-type может быть "application/problem+json" или "application/problem+xml"
        return ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR, "Произошла ошибка, повторите попытку");
    }
}
```

### Веб-приложение ASP.NET Core

В ASP.NET Core вы можете определить глобальный обработчик ошибок, указав, что обработчик исключений является отдельным контроллером API.

Пример содержимого контроллера API, предназначенного для обработки ошибок:

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;
using System;
using System.Collections.Generic;
using System.Net;

namespace MyProject.Controllers
{
    /// <summary>
    /// API контроллер для перехвата и обработки всех неожиданных исключений.
    /// </summary>
    [Route("api/[controller]")]
    [ApiController]
    [AllowAnonymous]
    public class ErrorController : ControllerBase
    {
        /// <summary>
        /// Метод, который будет вызван для обработки текущей ошибки.
        /// </summary>
        /// <returns>Возвращает общий ответ об ошибке в формате JSON, так как приложение использует REST API.</returns>
        [HttpGet]
        [HttpPost]
        [HttpHead]
        [HttpDelete]
        [HttpPut]
        [HttpOptions]
        [HttpPatch]
        public JsonResult Handle()
        {
            // Получение исключения, которое привело к вызову этого контроллера
            Exception exception = HttpContext.Features.Get<IExceptionHandlerFeature>()?.Error;
            // Логирование исключения, если оно не NULL
            // ...
            // Формируем общий ответ в формате JSON
            var responseBody = new Dictionary<String, String> {
                { "message", "Произошла ошибка, повторите попытку" }
            };
            JsonResult response = new JsonResult(responseBody);
            // Используем код ошибки сервера
            response.StatusCode = (int)HttpStatusCode.InternalServerError;
            Request.HttpContext.Response.Headers.Remove("X-ERROR");
            Request.HttpContext.Response.Headers.Add("X-ERROR", "true");
            return response;
        }
    }
}
```

### Определение в файле **Startup.cs** для сопоставления обработчика исключений с выделенным контроллером API для обработки ошибок:

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace MyProject
{
    public class Startup
    {
        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            // Сначала настраиваем промежуточное ПО для обработки ошибок!
            // Включаем глобальный обработчик ошибок в средах, отличных от DEV,
            // так как страницы отладки полезны при разработке
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }
            else
            {
                // Наш глобальный обработчик определён на URL "/api/error", 
                // поэтому указываем обработчику исключений вызывать этот API-контроллер
                // при любом неожиданном исключении, возникшем в приложении
                app.UseExceptionHandler("/api/error");

                // Чтобы настроить тип контента и текст ответа, используйте перегрузку
                // метода UseStatusCodePages, принимающую тип контента и строку формата.
                app.UseStatusCodePages("text/plain", "Страница статуса, код статуса: {0}");
            }

            // Настраиваем другие промежуточные ПО, не забывая, что порядок объявления важен...
            app.UseMvc();
        }
    }
}
```

### Веб-приложение ASP.NET Web API

В [ASP.NET Web API](https://www.asp.net/web-api) (для стандартного .NET Framework, а не .NET Core) можно определить и зарегистрировать обработчики для отслеживания и обработки любых ошибок, возникающих в приложении.

Определение обработчика для отслеживания деталей ошибки:

```csharp
using System;
using System.Web.Http.ExceptionHandling;

namespace MyProject.Security
{
    /// <summary>
    /// Глобальный логгер для отслеживания любых ошибок, возникающих на уровне всего приложения
    /// </summary>
    public class GlobalErrorLogger : ExceptionLogger
    {
        /// <summary>
        /// Метод, отвечающий за управление ошибками с точки зрения логирования
        /// </summary>
        /// <param name="context">Контекст, содержащий детали ошибки</param>
        public override void Log(ExceptionLoggerContext context)
        {
            // Получаем исключение
            Exception exception = context.Exception;
            // Логируем исключение через переменную "exception", если она не NULL
            // ...
        }
    }
}
```

### Регистрация глобального логгера в конфигурации Web API:

Чтобы зарегистрировать глобальный логгер, добавьте его в коллекцию служб `ExceptionLogger` в файле конфигурации Web API:

```csharp
public static class WebApiConfig
{
    public static void Register(HttpConfiguration config)
    {
        // Регистрация глобального логгера для всех ошибок
        config.Services.Add(typeof(IExceptionLogger), new GlobalErrorLogger());

        // Настройка маршрутов Web API
        config.MapHttpAttributeRoutes();

        config.Routes.MapHttpRoute(
            name: "DefaultApi",
            routeTemplate: "api/{controller}/{id}",
            defaults: new { id = RouteParameter.Optional }
        );
    }
}
```

### Определение обработчика для управления ошибками с целью возврата общего ответа:

```csharp
using Newtonsoft.Json;
using System;
using System.Collections.Generic;
using System.Net;
using System.Net.Http;
using System.Text;
using System.Threading;
using System.Threading.Tasks;
using System.Web.Http;
using System.Web.Http.ExceptionHandling;

namespace MyProject.Security
{
    /// <summary>
    /// Глобальный обработчик для обработки любых ошибок, возникающих на уровне всего приложения.
    /// </summary>
    public class GlobalErrorHandler : ExceptionHandler
    {
        /// <summary>
        /// Метод, отвечающий за управление общим ответом, отправляемым в случае ошибки.
        /// </summary>
        /// <param name="context">Контекст ошибки</param>
        public override void Handle(ExceptionHandlerContext context)
        {
            context.Result = new GenericResult();
        }

        /// <summary>
        /// Класс, представляющий общий ответ на ошибку.
        /// </summary>
        private class GenericResult : IHttpActionResult
        {
            /// <summary>
            /// Метод, отвечающий за создание общего ответа.
            /// </summary>
            /// <param name="cancellationToken">Объект для отмены задачи</param>
            /// <returns>Задача, отвечающая за отправку общего ответа</returns>
            public Task<HttpResponseMessage> ExecuteAsync(CancellationToken cancellationToken)
            {
                // Формируем общий ответ в формате JSON, так как это контекст REST API
                // Также добавляем заголовок HTTP-ответа, чтобы указать клиенту, что произошла ошибка
                var responseBody = new Dictionary<String, String>{
                    { "message", "Произошла ошибка, повторите попытку" }
                };
                // Используем ответ с кодом ошибки сервера (500)
                // В некоторых случаях разумно вернуть код ошибки 4xx, если проблема вызвана некорректными действиями клиента
                HttpResponseMessage response = new HttpResponseMessage(HttpStatusCode.InternalServerError);
                response.Headers.Add("X-ERROR", "true");
                response.Content = new StringContent(JsonConvert.SerializeObject(responseBody),
                                                     Encoding.UTF8, "application/json");
                return Task.FromResult(response);
            }
        }
    }
}
```

### Регистрация обоих обработчиков в файле **WebApiConfig.cs**:

```csharp
using MyProject.Security;
using System.Web.Http;
using System.Web.Http.ExceptionHandling;

namespace MyProject
{
    public static class WebApiConfig
    {
        public static void Register(HttpConfiguration config)
        {
            // Регистрация глобального логгера и обработчика ошибок
            config.Services.Replace(typeof(IExceptionLogger), new GlobalErrorLogger());
            config.Services.Replace(typeof(IExceptionHandler), new GlobalErrorHandler());
            // Остальная конфигурация
            //...
        }
    }
}
```

### Настройка секции `customErrors` в файле **Web.config** в пределах узла `<system.web>`:

```xml
<configuration>
    ...
    <system.web>
        <customErrors mode="RemoteOnly"
                      defaultRedirect="~/ErrorPages/Oops.aspx" />
        ...
    </system.web>
</configuration>
```

### Источники информации и ссылки:

- [Обработка исключений в ASP.Net Web API](https://exceptionnotfound.net/the-asp-net-web-api-exception-handling-pipeline-a-guided-tour/)
- [Обработка ошибок в ASP.NET](https://docs.microsoft.com/en-us/aspnet/web-forms/overview/getting-started/getting-started-with-aspnet-45-web-forms/aspnet-error-handling)

### Пример исходного кода:

Исходный код всех песочниц, созданных для поиска правильной конфигурации, доступен в этом [репозитории на GitHub](https://github.com/righettod/poc-error-handling).

### Приложение: HTTP ошибки

Референс HTTP ошибок можно найти здесь: [RFC 2616](https://www.ietf.org/rfc/rfc2616.txt). Использование сообщений об ошибках без предоставления деталей реализации важно для предотвращения утечек информации. В общем случае рекомендуется использовать коды ошибок 4xx для клиентских ошибок (например, неавторизованный доступ, слишком большой запрос), а коды 5xx для ошибок сервера, вызванных непредвиденными багами.
