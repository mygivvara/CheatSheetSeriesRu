# Шпаргалка по Bean Validation

## Введение

Эта статья направлена на предоставление четких, простых и практических рекомендаций по обеспечению безопасности с помощью Java Bean Validation в ваших приложениях.

Валидация бинов (JSR303, также известная как [Bean Validation 1.0](https://beanvalidation.org/1.0/spec/) / JSR349, также известная как [Bean Validation 1.1](https://beanvalidation.org/1.1/spec/)) является одним из наиболее распространенных способов выполнения [валидации ввода](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html) в Java. Это независимая от уровня приложения спецификация валидации, которая предоставляет разработчику возможность определять набор ограничений валидации для доменной модели и затем выполнять валидацию этих ограничений на различных уровнях приложения.

Одним из преимуществ этого подхода является то, что ограничения валидации и соответствующие валидаторы пишутся только один раз, что уменьшает дублирование усилий и обеспечивает единообразие:

### Типичная валидация

![Typical](../assets/Bean_Validation_Cheat_Sheet_Typical.png)

### Валидация бинов

![JSR](../assets/Bean_Validation_Cheat_Sheet_JSR.png)

## Настройка

Примеры в этом руководстве используют Hibernate Validator (референсную реализацию для Bean Validation 1.1).

Добавьте Hibernate Validator в ваш файл **pom.xml**:

```xml
<dependency>
   <groupId>org.hibernate</groupId>
   <artifactId>hibernate-validator</artifactId>
   <version>5.2.4.Final</version>
</dependency>
```

Включите поддержку валидации бинов в **context.xml** Spring:

```xml
<beans:beans ...
   ...
   <mvc:annotation-driven />
   ...
</beans:beans>
```

Для получения дополнительной информации, пожалуйста, ознакомьтесь с [руководством по настройке](https://hibernate.org/validator/documentation/getting-started/).

## Basics

Чтобы начать использовать Bean Validation, необходимо добавить ограничения валидации (`@Pattern`, `@Digits`, `@Min`, `@Max`, `@Size`, `@Past`, `@Future`, `@CreditCardNumber`, `@Email`, `@URL`, и др.) в вашу модель и использовать аннотацию `@Valid` при передаче вашей модели на различных уровнях приложения.

Ограничения могут быть применены в нескольких местах:

- Поля
- Свойства
- Классы

Для Bean Validation 1.1 также в:

- Параметрах
- Возвращаемых значениях
- Конструкторах

Для упрощения все приведенные ниже примеры используют ограничения для полей, и вся валидация инициируется контроллером. Обратитесь к документации по Bean Validation для полного списка примеров.

Что касается обработки ошибок, Hibernate Validator возвращает объект `BindingResult` который содержит `List<ObjectError>`. Примеры ниже содержат упрощенную обработку ошибок, тогда как готовое к использованию приложение должно иметь более сложный дизайн, учитывающий логирование и перенаправление на страницы ошибок.

## Предопределенные ограничения

### @Pattern

**Аннотация**:

`@Pattern(regex=,flag=)`

**Тип данных:**:

`CharSequence`

**Использование:**:

Проверяет, соответствует ли аннотированная строка регулярному выражению `regex` с учетом заданного флага соответствия. Пожалуйста, посетите [OWASP Validation Regex Repository](https://owasp.org/www-community/OWASP_Validation_Regex_Repository) для других полезных регулярных выражений.

**Ссылка**:

[Документация](https://docs.jboss.org/hibernate/validator/5.2/reference/en-US/html/ch02.html#section-builtin-constraints)

**Модель**:

```java
import org.hibernate.validator.constraints.Pattern;

public class Article  {
 //Ограничение: Буквенно-цифровые названия статей, в которых используются только регулярные выражения
 @Pattern(regexp = "[a-zA-Z0-9 ]")
 private String articleTitle;
 public String getArticleTitle()  {
  return  articleTitle;
 }
 public void setArticleTitle(String  articleTitle)  {
   this.articleTitle  =  articleTitle;
  }

  ...

}
```

**Контроллер**:

```java
import javax.validation.Valid;
import com.company.app.model.Article;

@Controller
public class ArticleController  {

 ...

 @RequestMapping(value = "/postArticle",  method = RequestMethod.POST)
 public @ResponseBody String postArticle(@Valid  Article  article,  BindingResult  result,
 HttpServletResponse  response) {
  if (result.hasErrors()) {
   String errorMessage  =  "";
   response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
   List<ObjectError> errors = result.getAllErrors();
   for(ObjectError  e :  errors) {
    errorMessage += "ERROR: " +  e.getDefaultMessage();
   }
   return  errorMessage;
  } else {
   return  "Validation Successful";
  }
 }
}
```

### @Digits

**Аннотация**:

`@Digits(integer=,fraction=)`

**Тип данных**:

`BigDecimal`, `BigInteger`, `CharSequence`, `byte`, `short`, `int`, `long` и соответствующие обертки примитивных типов; Также поддерживаются Hibernate Validator: любые подтипы `Number`

**Использование**:

Проверяет, является ли аннотированное значение числом, имеющим до `integer` цифр в целой части и `fraction` цифр в дробной части.

**Ссылка**:

[Документация](https://docs.jboss.org/hibernate/validator/5.2/reference/en-US/html/ch02.html#section-builtin-constraints)

**Модель:**:

```java
import org.hibernate.validator.constraints.Digits;

public class Customer {
  //Ограничение: возраст может состоять максимум из 3 цифр
  @Digits(integer = 3, fraction = 0)
  private int age;

  public String getAge()  {
    return age;
  }

  public void setAge(String age)  {
      this.age = age;
    }

    ...
}
```

**Контроллер**:

```java
import javax.validation.Valid;
import com.company.app.model.Customer;

@Controller
public class CustomerController  {

 ...

 @RequestMapping(value = "/registerCustomer",  method = RequestMethod.POST)
 public @ResponseBody String registerCustomer(@Valid Customer customer, BindingResult result,
 HttpServletResponse  response) {

  if (result.hasErrors()) {
   String errorMessage = "";
   response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
   List<ObjectError> errors = result.getAllErrors();

   for( ObjectError  e :  errors) {
    errorMessage += "ERROR: "  +  e.getDefaultMessage();
   }
   return  errorMessage;
  } else {
   return  "Validation Successful";
  }
 }
}
```

### @Size

**Аннотация**:

`@Size(min=,` `max=)`

**Тип данных**:

`CharSequence`, `Collection`, `Map` and `Arrays`

**Использование**:

Проверяет, что размер аннотированного элемента находится в пределах от `min` and `max` (включительно)

**Ссылка**:

[Документация](https://docs.jboss.org/hibernate/validator/5.2/reference/en-US/html/ch02.html#section-builtin-constraints)

**Модель**:

```java
import org.hibernate.validator.constraints.Size;

public class Message {

   //Ограничение: сообщение должно быть длиной минимум 10 символов, но не более 500
   @Size(min = 10, max = 500)
   private String message;

   public String getMessage() {
      return message;
   }

   public void setMessage(String message) {
      this.message = message;
   }

...
}
```

**Контроллер**:

```java
import javax.validation.Valid;
import com.company.app.model.Message;

@Controller
public class MessageController {

...

@RequestMapping(value="/sendMessage", method=RequestMethod.POST)
public @ResponseBody String sendMessage(@Valid Message message, BindingResult result,
HttpServletResponse response){

   if(result.hasErrors()){
      String errorMessage = "";
      response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
      List<ObjectError> errors = result.getAllErrors();
      for( ObjectError e : errors){
         errorMessage+= "ERROR: " + e.getDefaultMessage();
      }
      return errorMessage;
   }
   else{
      return "Validation Successful";
   }
}
}
```

### @Past / @Future

**Аннотация**:

`@Past,` `@Future`

**Тип данных**:

`java.util.Date`, `java.util.Calendar`, `java.time.chrono.ChronoZonedDateTime`, `java.time.Instant`, `java.time.OffsetDateTime`

**Использование**:

Проверяет, находится ли аннотированная дата в прошлом или будущем.

**Ссылка**:

[Документация](https://docs.jboss.org/hibernate/validator/5.2/reference/en-US/html/ch02.html#section-builtin-constraints)

**Модель**:

```java
import org.hibernate.validator.constraints.Past;
import org.hibernate.validator.constraints.Future;

public class DoctorVisit {

   //Ограничение: Дата рождения должна быть в прошлом
   @Past
   private Date birthDate;

   public Date getBirthDate() {
      return birthDate;
   }

   public void setBirthDate(Date birthDate) {
      this.birthDate = birthDate;
   }

   //Ограничение: Дата запланированного визита должна быть в будущем
   @Future
   private String scheduledVisitDate;

   public String getScheduledVisitDate() {
      return scheduledVisitDate;
   }

   public void setScheduledVisitDate(String scheduledVisitDate) {
      this.scheduledVisitDate = scheduledVisitDate;
   }

...
}
```

**Контроллер**:

```java
import javax.validation.Valid;
import com.company.app.model.DoctorVisit;

@Controller
public class DoctorVisitController {

   ...

   @RequestMapping(value="/scheduleVisit", method=RequestMethod.POST)
   public @ResponseBody String scheduleVisit(@Valid DoctorVisit doctorvisit, BindingResult result,
   HttpServletResponse response){

      if(result.hasErrors()){
         String errorMessage = "";
         response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
         List<ObjectError> errors = result.getAllErrors();
         for( ObjectError e : errors){
            errorMessage+= "ERROR: " + e.getDefaultMessage();
         }
         return errorMessage;
      }
      else{
         return "Validation Successful";
      }
   }
}
```

### Сочетание ограничений

Аннотации валидации могут быть скомбинированы любым подходящим образом. Например, чтобы указать допустимое значение рейтинга отзыва в пределах от 1 до 5, задайте валидацию следующим образом:

**Аннотация**:

`@Min(value=),` `@Max(value=)`

**Тип данных**:

`BigDecimal`, `BigInteger`, `byte`, `short`, `int`, `long` и соответствующие обертки примитивных типов; Также поддерживается Hibernate Validator: любые подтипы `CharSequence` (числовое значение, представленное последовательностью символов, оценивается), любые подтипы `Number`

**Использование**:

Проверяет, является ли аннотированное значение больше/меньше или равно заданному минимальному/максимальному значению.

**Ссылка**:

[Документация](https://docs.jboss.org/hibernate/validator/5.2/reference/en-US/html/ch02.html#section-builtin-constraints)

**Модель**:

```java
import org.hibernate.validator.constraints.Min;
import org.hibernate.validator.constraints.Max;

public class Review {

 //Ограничение: Рейтинг отзыва должен быть от 1 до 5
 @Min(1)
 @Max(5)
 private int reviewRating;

 public int getReviewRating() {
   return reviewRating;
 }

 public void setReviewRating(int reviewRating) {
   this.reviewRating = reviewRating;
}
 ...
}
```

**Контроллер**:

```java
import javax.validation.Valid;
import com.company.app.model.ReviewRating;

@Controller
public class ReviewController {

   ...

   @RequestMapping(value="/postReview", method=RequestMethod.POST)
   public @ResponseBody String postReview(@Valid Review review, BindingResult result,
   HttpServletResponse response){

      if(result.hasErrors()){
         String errorMessage = "";
         response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
         List<ObjectError> errors = result.getAllErrors();
         for( ObjectError e : errors){
            errorMessage+= "ERROR: " + e.getDefaultMessage();
         }
         return errorMessage;
      }
       else{
         return "Validation Successful";
      }
   }
}
```

### Каскадные ограничения

Проверка одного Java Bean — это хороший старт, но часто Bean-ы вложены или образуют полный граф Bean-ов. Чтобы проверить этот граф за один раз, применяйте каскадную валидацию с аннотацией [@Valid](https://docs.jboss.org/hibernate/validator/5.2/reference/en-US/html/ch03.html#_cascaded_validation)

### Дополнительные ограничения

Помимо полного набора ограничений JSR303, Hibernate Validator также определяет некоторые дополнительные ограничения для удобства:

- `@CreditCardNumber`
- `@EAN`
- `@Email`
- `@Length`
- `@Range`
- `@ScriptAssert`
- `@URL`

Просмотрите эту [таблицу](https://docs.jboss.org/hibernate/validator/5.2/reference/en-US/html/ch02.html#table-custom-constraints) для полного списка.

Обратите внимание, что `@SafeHtml`, ранее допустимое ограничение, было устаревшим в соответствии с [блогом о релизе Hibernate Validator 6.1.0.Final и 6.0.18.Final](https://in.relation.to/2019/11/20/hibernate-validator-610-6018-released/). Пожалуйста, воздержитесь от использования ограничения `@SafeHtml`.

## Пользовательские ограничения

Одной из самых мощных функций валидации бинов является возможность определять собственные ограничения, которые выходят за рамки простой валидации, предлагаемой встроенными ограничениями.

Создание пользовательских ограничений выходит за рамки данного руководства. Ознакомьтесь с [документацией](https://docs.jboss.org/hibernate/validator/5.2/reference/en-US/html/ch06.html) для получения подробной информации.

## Сообщения об ошибках

Можно указать ID сообщения с аннотацией валидации, чтобы настраивать сообщения об ошибках:

```java
@Pattern(regexp = "[a-zA-Z0-9 ]", message="article.title.error")
private String articleTitle;
```

Spring MVC затем будет искать сообщение с ID *article.title.error* в определенном `MessageSource`. Подробнее об этом читайте в [документации](https://www.silverbaytech.com/2013/04/16/custom-messages-in-spring-validation/).
