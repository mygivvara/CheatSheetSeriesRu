# Шпаргалка по автоматизации тестирования авторизации

## Введение

**При реализации мер защиты приложения одной из самых важных частей процесса является определение и внедрение авторизаций приложения.** Несмотря на все проверки и аудиты безопасности, проводимые на этапе создания, большинство проблем с авторизациями возникают из-за добавления или изменения функций в обновленных выпусках без определения их влияния на авторизации приложения (как правило, по причинам, связанным с затратами или временем).

Чтобы решить эту проблему, мы рекомендуем разработчикам автоматизировать оценку авторизаций и проводить тестирование при выпуске новой версии. Это гарантирует, что команда будет знать, будут ли изменения в приложении конфликтовать с определением и/или реализацией авторизации.

## Контекст

Авторизация обычно содержит два элемента (также называемых измерениями): **Функцию** и **Логическую роль**, которая получает к ней доступ. Иногда добавляется третье измерение под названием **Данные**, чтобы определить доступ, включающий фильтрацию на уровне бизнес-данных.

Как правило, два измерения каждой авторизации должны быть указаны в электронной таблице, которая называется **Матрицей авторизации**. При тестировании авторизаций логические роли иногда называют **Точкой зрения**.

## Цель

Эта шпаргалка предназначена для помощи в разработке собственных подходов к автоматизации тестирования авторизаций в матрице авторизаций. Так как разработчикам нужно будет разработать собственный подход к автоматизации тестирования авторизаций, **эта шпаргалка покажет возможный подход к автоматизации тестов авторизаций для одной из возможных реализаций приложения, предоставляющего REST-сервисы.**

## Предложение

### Подготовка к автоматизации матрицы авторизаций

Прежде чем начать автоматизацию тестирования матрицы авторизаций, необходимо выполнить следующие действия:

1. **Формализовать матрицу авторизаций в виде файла сводного формата, который позволит вам:**
    1. Легко обрабатывать матрицу программой.
    2. Дать возможность человеку читать и обновлять ее при необходимости отслеживания комбинаций авторизаций.
    3. Настроить иерархию авторизаций, что позволит легко создавать различные комбинации.
    4. Создать максимально возможную независимость от используемой технологии и дизайна приложения.

2. **Создать набор интеграционных тестов, которые полностью используют файл сводного формата матрицы авторизаций в качестве источника данных, что позволит вам оценить различные комбинации с следующими преимуществами:**
    1. Минимальное возможное обслуживание при обновлении файла сводного формата матрицы авторизаций.
    2. Четкое указание, в случае провала теста, комбинации авторизаций, которая не соответствует матрице авторизаций.

### Создание файла сводного формата матрицы авторизаций

**В этом примере мы используем формат XML для формализации матрицы авторизаций.**

Эта XML-структура включает три основных раздела (или узла):

- Узел **roles**: Описывает возможные логические роли, используемые в системе, предоставляет список ролей и объясняет различные уровни авторизации.
- Узел **services**: Предоставляет список доступных сервисов, предлагаемых системой, дает описание этих сервисов и определяет связанные логические роли, которые могут их вызывать.
- Узел **services-testing**: Предоставляет тестовую нагрузку для каждого сервиса, если сервис использует входные данные, отличные от тех, которые поступают из URL или пути.

**Этот пример демонстрирует, как может быть определена авторизация с использованием XML:**:

> Плейсхолдеры (значения в {} ) используются для обозначения местоположения, куда тестовые значения должны быть вставлены интеграционными тестами при необходимости.

``` xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!--
      Этот файл создает матрицу авторизации для различных 
      служб, предоставляемых системой:

      Тесты будут использовать ее в качестве источника входных данных для различных \
      тестовых примеров путем:
      1) Определения легитимного доступа и правильной реализации
      2) Выявления нелегитимного доступа (проблема с определением авторизации
      при реализации службы)

       Атрибут "name" используется для уникальной идентификации СЛУЖБЫ или РОЛИ.
  -->
  <authorization-matrix>

      <!-- Описание возможных логических ролей, используемых в системе, используется здесь для
        предоставления списка+пояснений
        различных ролей (уровень авторизации) -->
      <roles>
          <role name="ANONYMOUS"
          description="Indicate that no authorization is needed"/>
          <role name="BASIC"
          description="Role affecting a standard user (lowest access right just above anonymous)"/>
          <role name="ADMIN"
          description="Role affecting an administrator user (highest access right)"/>
      </roles>

      <!-- Перечислите и опишите доступные службы, предоставляемые системой, и связанные с ними
        логические роли, которые могут их вызывать -->
      <services>
          <service name="ReadSingleMessage" uri="/{messageId}" http-method="GET"
          http-response-code-for-access-allowed="200" http-response-code-for-access-denied="403">
              <role name="ANONYMOUS"/>
              <role name="BASIC"/>
              <role name="ADMIN"/>
          </service>
          <service name="ReadAllMessages" uri="/" http-method="GET"
          http-response-code-for-access-allowed="200" http-response-code-for-access-denied="403">
              <role name="ANONYMOUS"/>
              <role name="BASIC"/>
              <role name="ADMIN"/>
          </service>
          <service name="CreateMessage" uri="/" http-method="PUT"
          http-response-code-for-access-allowed="200" http-response-code-for-access-denied="403">
              <role name="BASIC"/>
              <role name="ADMIN"/>
          </service>
          <service name="DeleteMessage" uri="/{messageId}" http-method="DELETE"
          http-response-code-for-access-allowed="200" http-response-code-for-access-denied="403">
              <role name="ADMIN"/>
          </service>
      </services>

      <!-- При необходимости предоставьте тестовый пейлоад для каждой службы -->
      <services-testing>
          <service name="ReadSingleMessage">
              <payload/>
          </service>
          <service name="ReadAllMessages">
              <payload/>
          </service>
          <service name="CreateMessage">
              <payload content-type="application/json">
                  {"content":"test"}
              </payload>
          </service>
          <service name="DeleteMessage">
              <payload/>
          </service>
      </services-testing>

  </authorization-matrix>
```

### Реализация интеграционного теста

**Чтобы создать интеграционный тест, вы должны использовать максимально факторизованный код и один тестовый случай для каждой Точки Зрения (Point Of View, POV), чтобы верификации могли быть профилированы по уровню доступа (логическая роль). Это упростит отображение/идентификацию ошибок.**

В этом интеграционном тесте мы реализовали парсинг, сопоставление объектов и доступ к матрице авторизаций путем маршаллинга XML в объект Java и демаршаллинга объекта обратно в XML. Эти функции используются для реализации тестов (здесь применяется JAXB) и ограничивают объем кода, который должен разрабатывать разработчик, отвечающий за выполнение тестов.

**Пример реализации класса интеграционного тестового случая:**

``` java
  import org.owasp.pocauthztesting.enumeration.SecurityRole;
  import org.owasp.pocauthztesting.service.AuthService;
  import org.owasp.pocauthztesting.vo.AuthorizationMatrix;
  import org.apache.http.client.methods.CloseableHttpResponse;
  import org.apache.http.client.methods.HttpDelete;
  import org.apache.http.client.methods.HttpGet;
  import org.apache.http.client.methods.HttpPut;
  import org.apache.http.client.methods.HttpRequestBase;
  import org.apache.http.entity.StringEntity;
  import org.apache.http.impl.client.CloseableHttpClient;
  import org.apache.http.impl.client.HttpClients;
  import org.junit.Assert;
  import org.junit.BeforeClass;
  import org.junit.Test;
  import org.xml.sax.InputSource;
  import javax.xml.bind.JAXBContext;
  import javax.xml.parsers.SAXParserFactory;
  import javax.xml.transform.Source;
  import javax.xml.transform.sax.SAXSource;
  import java.io.File;
  import java.io.FileInputStream;
  import java.util.ArrayList;
  import java.util.List;
  import java.util.Optional;

  /**
   * Интеграционные тестовые случаи проверяют корректность реализации матрицы авторизаций.
   * Они создают тестовый случай для каждой логической роли, который проверяет доступ ко всем сервисам, предоставляемым системой.
   * Эта реализация ориентирована на читаемость кода.
   */
  public class AuthorizationMatrixIT {

      /**
       * Представление объекта матрицы авторизаций
       */
      private static AuthorizationMatrix AUTHZ_MATRIX;

      private static final String BASE_URL = "http://localhost:8080";


      /**
       * Загрузка матрицы авторизаций в древовидную структуру объектов
       *
       * @throws Exception Если произошла ошибка
       */
      @BeforeClass
      public static void globalInit() throws Exception {
          try (FileInputStream fis = new FileInputStream(new File("authorization-matrix.xml"))) {
              SAXParserFactory spf = SAXParserFactory.newInstance();
              spf.setFeature("http://xml.org/sax/features/external-general-entities", false);
              spf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
              spf.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
              Source xmlSource = new SAXSource(spf.newSAXParser().getXMLReader(), new InputSource(fis));
              JAXBContext jc = JAXBContext.newInstance(AuthorizationMatrix.class);
              AUTHZ_MATRIX = (AuthorizationMatrix) jc.createUnmarshaller().unmarshal(xmlSource);
          }
      }

      /**
       * Тестирование доступа к сервисам от анонимного пользователя.
       *
       * @throws Exception
       */
      @Test
      public void testAccessUsingAnonymousUserPointOfView() throws Exception {
          //Run the tests - No access token here
          List<String> errors = executeTestWithPointOfView(SecurityRole.ANONYMOUS, null);
          //Verify the test results
          Assert.assertEquals("Проблемы с доступом обнаружены используя Точку Зрения АНОНИМНОГО ПОЛЬЗОВАТЕЛЯ:\n" + formatErrorsList(errors), 0, errors.size());
      }

      /**
       * Тестирование доступа к сервисам от обычного пользователя.
       *
       * @throws Exception
       */
      @Test
      public void testAccessUsingBasicUserPointOfView() throws Exception {
          //Get access token representing the authorization for the associated point of view
          String accessToken = generateTestCaseAccessToken("basic", SecurityRole.BASIC);
          //Run the tests
          List<String> errors = executeTestWithPointOfView(SecurityRole.BASIC, accessToken);
          //Verify the test results
          Assert.assertEquals("Проблемы с доступом обнаружены используя Точку Зрения ОБЫЧНОГО ПОЛЬЗОВАТЕЛЯ:\n " + formatErrorsList(errors), 0, errors.size());
      }

      /**
       * Тестирование доступа к сервисам от пользователя с правами администратора.
       *
       * @throws Exception
       */
      @Test
      public void testAccessUsingAdministratorUserPointOfView() throws Exception {
          //Получение токена доступа, представляющего авторизацию для связанной точки зрения
          String accessToken = generateTestCaseAccessToken("admin", SecurityRole.ADMIN);
          //Запуск тестов
          List<String> errors = executeTestWithPointOfView(SecurityRole.ADMIN, accessToken);
          //Проверка результатов тестов
          Assert.assertEquals("Проблемы с доступом обнаружены используя Точку Зрения АДМИНИСТРАТОРА:\n" + formatErrorsList(errors), 0, errors.size());
      }

      /**
       * Оценка доступа ко всем сервисам с использованием указанной точки зрения (POV).
       *
       * @param pointOfView Используемая точка зрения
       * @param accessToken Токен доступа, связанный с точкой зрения с точки зрения авторизации.
       * @return Список обнаруженных ошибок
       * @throws Exception Если произошла ошибка
       */
      private List<String> executeTestWithPointOfView(SecurityRole pointOfView, String accessToken) throws Exception {
          List<String> errors = new ArrayList<>();
          String errorMessageTplForUnexpectedReturnCode = "Сервис '%s' при вызове с POV '%s' возвращает код ответа %s, который не является ожидаемым в случае разрешения или отказа.";
          String errorMessageTplForIncorrectReturnCode = "Сервис '%s' при вызове с POV '%s' возвращает код ответа %s, который не является ожидаемым (ожидался %s).";
          String fatalErrorMessageTpl = "Сервис '%s' при вызове с POV %s вызывает ошибку: %s";

          //Получение списка сервисов для вызова
          List<AuthorizationMatrix.Services.Service> services = AUTHZ_MATRIX.getServices().getService();

          //Получение списка тестовой нагрузки сервисов для использования
          List<AuthorizationMatrix.ServicesTesting.Service> servicesTestPayload = AUTHZ_MATRIX.getServicesTesting().getService();

          //Последовательный вызов всех сервисов (без особого акцента на производительность)
          services.forEach(service -> {
              //Получить тестовую нагрузку для текущего сервиса
              String payload = null;
              String payloadContentType = null;
              Optional<AuthorizationMatrix.ServicesTesting.Service> serviceTesting = servicesTestPayload.stream().filter(srvPld -> srvPld.getName().equals(service.getName())).findFirst();
              if (serviceTesting.isPresent()) {
                  payload = serviceTesting.get().getPayload().getValue();
                  payloadContentType = serviceTesting.get().getPayload().getContentType();
              }
              //Вызвать сервис и проверить, соответствует ли ответ ожиданиям
              try {
                  //Вызвать сервис
                  int serviceResponseCode = callService(service.getUri(), payload, payloadContentType, service.getHttpMethod(), accessToken);
                  //Проверить, определена ли роль, представленная указанной точкой зрения, для текущего сервиса
                  Optional<AuthorizationMatrix.Services.Service.Role> role = service.getRole().stream().filter(r -> r.getName().equals(pointOfView.name())).findFirst();
                  boolean accessIsGrantedInAuthorizationMatrix = role.isPresent();
                  //Проверить согласованность поведения в соответствии с возвращаемым кодом ответа и авторизацией, настроенной в матрице
                  if (serviceResponseCode == service.getHttpResponseCodeForAccessAllowed()) {
                      //Роль не входит в список ролей, которым разрешен доступ к сервису, следовательно, это ошибка
                      if (!accessIsGrantedInAuthorizationMatrix) {
                          errors.add(String.format(errorMessageTplForIncorrectReturnCode, service.getName(), pointOfView.name(), serviceResponseCode,
                           service.getHttpResponseCodeForAccessDenied()));
                      }
                  } else if (serviceResponseCode == service.getHttpResponseCodeForAccessDenied()) {
                      //Роль входит в список ролей, которым разрешен доступ к сервису, следовательно, это ошибка
                      if (accessIsGrantedInAuthorizationMatrix) {
                          errors.add(String.format(errorMessageTplForIncorrectReturnCode, service.getName(), pointOfView.name(), serviceResponseCode,
                           service.getHttpResponseCodeForAccessAllowed()));
                      }
                  } else {
                      errors.add(String.format(errorMessageTplForUnexpectedReturnCode, service.getName(), pointOfView.name(), serviceResponseCode));
                  }
              } catch (Exception e) {
                  errors.add(String.format(fatalErrorMessageTpl, service.getName(), pointOfView.name(), e.getMessage()));
              }


          });

          return errors;
      }

      /**
       * Вызвать сервис с определенной нагрузкой и вернуть HTTP-код ответа, который был получен.
       * Этот шаг был делегирован для облегчения поддержки тестовых случаев.
       *
       * @param uri                URI целевого сервиса
       * @param payloadContentType Тип содержимого нагрузки, которую нужно отправить
       * @param payload            Нагрузка, которую нужно отправить
       * @param httpMethod         HTTP-метод, который нужно использовать
       * @param accessToken        Токен доступа для указания идентичности вызывающего пользователя
       * @return Полученный HTTP-код ответ
       * @throws Exception Исключение возникает в случае любой ошибки
       */
      private int callService(String uri, String payload, String payloadContentType, String httpMethod, String accessToken) throws Exception {
          int rc;

          //Построить запрос — используйте Apache HTTP Client, чтобы обеспечить большую гибкость в комбинациях.
          HttpRequestBase request;
          String url = (BASE_URL + uri).replaceAll("\\{messageId\\}", "1");
          switch (httpMethod) {
              case "GET":
                  request = new HttpGet(url);
                  break;
              case "DELETE":
                  request = new HttpDelete(url);
                  break;
              case "PUT":
                  request = new HttpPut(url);
                  if (payload != null) {
                      request.setHeader("Content-Type", payloadContentType);
                      ((HttpPut) request).setEntity(new StringEntity(payload.trim()));
                  }
                  break;
              default:
                  throw new UnsupportedOperationException(httpMethod + " не поддерживается !");
          }
          request.setHeader("Авторизация", (accessToken != null) ? accessToken : "");


          //Отправить запрос и получить HTTP-код ответа.
          try (CloseableHttpClient httpClient = HttpClients.createDefault()) {
              try (CloseableHttpResponse httpResponse = httpClient.execute(request)) {
                  //Содержимое ответа здесь не имеет значения...
                  rc = httpResponse.getStatusLine().getStatusCode();
              }
          }

          return rc;
      }

      /**
       * Генерация JWT-токена для указанного пользователя и роли.
       *
       * @param login Логин пользователя
       * @param role  Логическая роль авторизации
       * @return JWT-токен
       * @throws Exception Исключение возникает в случае любой ошибки при создании
       */
      private String generateTestCaseAccessToken(String login, SecurityRole role) throws Exception {
          return new AuthService().issueAccessToken(login, role);
      }


      /**
       * Форматировать список ошибок в строку для печати.
       *
       * @param errors Список ошибок
       * @return Строка для печати
       */
      private String formatErrorsList(List<String> errors) {
          StringBuilder buffer = new StringBuilder();
          errors.forEach(e -> buffer.append(e).append("\n"));
          return buffer.toString();
      }
  }
```

Если обнаружена проблема с авторизацией (или несколько проблем), вывод будет следующим:

```java
testAccessUsingAnonymousUserPointOfView(org.owasp.pocauthztesting.AuthorizationMatrixIT)
Time elapsed: 1.009 s  ### FAILURE
java.lang.AssertionError:
Access issues detected using the ANONYMOUS USER point of view:
    The service 'DeleteMessage' when called with POV 'ANONYMOUS' return
    a response code 200 that is not the expected one (403 expected).

    The service 'CreateMessage' when called with POV 'ANONYMOUS' return
    a response code 200 that is not the expected one (403 expected).

testAccessUsingBasicUserPointOfView(org.owasp.pocauthztesting.AuthorizationMatrixIT)
Time elapsed: 0.05 s  ### FAILURE!
java.lang.AssertionError:
Access issues detected using the BASIC USER point of view:
    The service 'DeleteMessage' when called with POV 'BASIC' return
    a response code 200 that is not the expected one (403 expected).
```

## Отображение матрицы авторизаций для аудита / проверки

Даже если матрица авторизаций хранится в формате, удобочитаемом для человека (например, в XML), возможно, вы захотите представить ее в динамически сгенерированном виде. Это поможет выявить потенциальные несоответствия и упростить проверку, аудит и обсуждение матрицы авторизаций.

Для выполнения этой задачи вы можете использовать следующий XSL-стилистический лист:

``` xslt
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet xmlns:xsl="http://www.w3.org/1999/XSL/Transform" version="1.0">
  <xsl:template match="/">
    <html>
      <head>
        <title>Authorization Matrix</title>
        <link rel="stylesheet"
        href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-alpha.6/css/bootstrap.min.css"
        integrity="sha384-rwoIResjU2yc3z8GV/NPeZWAv56rSmLldC3R/AZzGRnGxQQKnKkoFVhFQhNUwEyJ"
        crossorigin="anonymous" />
      </head>
      <body>
        <h3>Roles</h3>
        <ul>
          <xsl:for-each select="authorization-matrix/roles/role">
            <xsl:choose>
              <xsl:when test="@name = 'ADMIN'">
                <div class="alert alert-warning" role="alert">
                  <strong>
                    <xsl:value-of select="@name" />
                  </strong>
                  :
                  <xsl:value-of select="@description" />
                </div>
              </xsl:when>
              <xsl:when test="@name = 'BASIC'">
                <div class="alert alert-info" role="alert">
                  <strong>
                    <xsl:value-of select="@name" />
                  </strong>
                  :
                  <xsl:value-of select="@description" />
                </div>
              </xsl:when>
              <xsl:otherwise>
                <div class="alert alert-danger" role="alert">
                  <strong>
                    <xsl:value-of select="@name" />
                  </strong>
                  :
                  <xsl:value-of select="@description" />
                </div>
              </xsl:otherwise>
            </xsl:choose>
          </xsl:for-each>
        </ul>
        <h3>Authorizations</h3>
        <table class="table table-hover table-sm">
          <thead class="thead-inverse">
            <tr>
              <th>Service</th>
              <th>URI</th>
              <th>Method</th>
              <th>Role</th>
            </tr>
          </thead>
          <tbody>
            <xsl:for-each select="authorization-matrix/services/service">
              <xsl:variable name="service-name" select="@name" />
              <xsl:variable name="service-uri" select="@uri" />
              <xsl:variable name="service-method" select="@http-method" />
              <xsl:for-each select="role">
                <tr>
                  <td scope="row">
                    <xsl:value-of select="$service-name" />
                  </td>
                  <td>
                    <xsl:value-of select="$service-uri" />
                  </td>
                  <td>
                    <xsl:value-of select="$service-method" />
                  </td>
                  <td>
                    <xsl:variable name="service-role-name" select="@name" />
                    <xsl:choose>
                      <xsl:when test="@name = 'ADMIN'">
                        <div class="alert alert-warning" role="alert">
                          <xsl:value-of select="@name" />
                        </div>
                      </xsl:when>
                      <xsl:when test="@name = 'BASIC'">
                        <div class="alert alert-info" role="alert">
                          <xsl:value-of select="@name" />
                        </div>
                      </xsl:when>
                      <xsl:otherwise>
                        <div class="alert alert-danger" role="alert">
                          <xsl:value-of select="@name" />
                        </div>
                      </xsl:otherwise>
                    </xsl:choose>
                  </td>
                </tr>
              </xsl:for-each>
            </xsl:for-each>
          </tbody>
        </table>
      </body>
    </html>
  </xsl:template>
</xsl:stylesheet>
```

Пример отображения

![RenderingExample](../assets/Authorization_Testing_Automation_AutomationRendering.png)

## Источники для прототипа

[GitHub репозиторий](https://github.com/righettod/poc-authz-testing)
