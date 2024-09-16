# Cheat Sheet по JAAS

## Введение - Что такое аутентификация JAAS

Процесс проверки личности пользователя или другой системы называется аутентификацией.

[JAAS](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jaas/JAASRefGuide.html), как фреймворк аутентификации, управляет личностью и учетными данными аутентифицированного пользователя от входа в систему до выхода.

Жизненный цикл аутентификации JAAS:

1. Создайте `LoginContext`.
2. Прочитайте конфигурационный файл для одного или нескольких `LoginModules` для инициализации.
3. Вызовите `LoginContext.initialize()` для каждого `LoginModule` для инициализации.
4. Вызовите `LoginContext.login()` для каждого `LoginModule`.
5. Если вход выполнен успешно, вызовите `LoginContext.commit()`, в противном случае вызовите `LoginContext.abort()`.

## Конфигурационный файл

Конфигурационный файл JAAS содержит секцию `LoginModule` для каждого доступного `LoginModule` для входа в приложение.

Пример секции из конфигурационного файла JAAS:

```text
Branches
{
    USNavy.AppLoginModule required
    debug=true
    succeeded=true;
}
```

Обратите внимание на размещение точек с запятой, которые завершает как записи `LoginModule`, так и секции.

Слово `required` указывает на то, что метод `login()` `LoginContext` должен быть успешным при входе пользователя. Значения, специфичные для `LoginModule`, такие как `debug` и `succeeded`, передаются в `LoginModule`.

Они определяются `LoginModule` и их использование управляется внутри `LoginModule`. Обратите внимание, что параметры настраиваются с помощью пар ключ-значение, таких как `debug="true"`, и ключ и значение должны быть разделены знаком `=`.

## Main.java (Клиент)

- Синтаксис выполнения:

```text
Java –Djava.security.auth.login.config==packageName/packageName.config
        packageName.Main Stanza1

Где:
    packageName - директория, содержащая конфигурационный файл.
    packageName.config - указывает конфигурационный файл в пакете Java, packageName.
    packageName.Main - указывает на Main.java в пакете Java, packageName.
    Stanza1 - имя секции, которую должен прочитать Main() из конфигурационного файла.
```

- При выполнении первый аргумент командной строки — это секция из конфигурационного файла. Секция указывает, какой `LoginModule` должен быть использован. Второй аргумент — это `CallbackHandler`.
- Создайте новый `LoginContext` с аргументами, переданными в `Main.java`:
    - `loginContext = new LoginContext(args[0], new AppCallbackHandler());`
- Вызовите модуль LoginContext:
    - `loginContext.login();`
- Значение в параметре `succeeded` возвращается из `loginContext.login()`.
- Если вход в систему прошел успешно, создается субъект.

## LoginModule.java

`LoginModule` должен иметь следующие методы аутентификации:

- `initialize()`
- `login()`
- `commit()`
- `abort()`
- `logout()`

### initialize()

В `Main()`, после того как `LoginContext` прочитает правильную секцию из конфигурационного файла, `LoginContext` создает экземпляр `LoginModule`, указанный в секции.

- Подпись метода `initialize()`:
    - `Public void initialize(Subject subject, CallbackHandler callbackHandler, Map sharedState, Map options)`
- Аргументы выше должны сохраняться следующим образом:
    - `this.subject = subject;`
    - `this.callbackHandler = callbackHandler;`
    - `this.sharedState = sharedState;`
    - `this.options = options;`
- Что делает метод `initialize()`:
    - Создает объект субъекта класса `Subject` при успешной аутентификации.
    - Устанавливает `CallbackHandler`, который взаимодействует с пользователем для сбора информации для входа.
    - Если `LoginContext` указывает 2 или более `LoginModules`, что допустимо, они могут обмениваться информацией через карту `sharedState`.
    - Сохраняет информацию о состоянии, такую как `debug` и `succeeded`, в карте `options`.

### login()

Метод `login()` захватывает информацию для входа, предоставленную пользователем. Приведенный ниже фрагмент кода объявляет массив из двух объектов обратного вызова, которые, переданные методу `callbackHandler.handle` в программе `callbackHandler.java`, будут заполнены именем пользователя и паролем, предоставленными пользователем интерактивно:

```java
NameCallback nameCB = new NameCallback("Username");
PasswordCallback passwordCB = new PasswordCallback("Password", false);
Callback[] callbacks = new Callback[] { nameCB, passwordCB };
callbackHandler.handle(callbacks);
```

- Аутентифицирует пользователя.
- Извлекает информацию, предоставленную пользователем, из объектов обратного вызова:
    - `String ID = nameCallback.getName();`
    - `char[] tempPW = passwordCallback.getPassword();`
- Сравнивает `name` и `tempPW` со значениями, хранящимися в репозитории, таком как LDAP.
- Устанавливает значение переменной `succeeded` и возвращается в `Main()`.

### commit()

После успешной проверки учетных данных пользователя в методе `login()`, фреймворк JAAS связывает учетные данные с субъектом, если это необходимо.

Существуют два типа учетных данных: **Публичные** и **Приватные**:

- Публичные учетные данные включают в себя публичные ключи.
- Приватные учетные данные включают в себя пароли и публичные ключи.

Принципы (т.е. идентификаторы, которые субъект имеет помимо имени для входа), такие как номер сотрудника или ID в группе пользователей, добавляются к субъекту.

Ниже приведен пример метода `commit()`, в котором сначала для каждой группы, в которой аутентифицированный пользователь имеет членство, имя группы добавляется в качестве принципала к субъекту. Затем имя пользователя добавляется к их публичным учетным данным.

Фрагмент кода, устанавливающий и добавляющий любые принципы и публичные учетные данные к субъекту:

```java
public boolean commit() {
    if (userAuthenticated) {
        Set groups = UserService.findGroups(username);
        for (Iterator itr = groups.iterator(); itr.hasNext(); ) {
            String groupName = (String) itr.next();
            UserGroupPrincipal group = new UserGroupPrincipal(groupName);
            subject.getPrincipals().add(group);
        }
        UsernameCredential cred = new UsernameCredential(username);
        subject.getPublicCredentials().add(cred);
    }
}
```

### abort()

Метод `abort()` вызывается, когда аутентификация не удалась. Перед тем как метод `abort()` завершит работу `LoginModule`, необходимо сбросить состояние, включая поля ввода имени пользователя и пароля.

### logout()

Метод `logout()` освобождает принципы и учетные данные пользователя, когда вызывается `LoginContext.logout`:

```java
public boolean logout() {
    if (!subject.isReadOnly()) {
        Set principals = subject.getPrincipals(UserGroupPrincipal.class);
        subject.getPrincipals().removeAll(principals);
        Set creds = subject.getPublicCredentials(UsernameCredential.class);
        subject.getPublicCredentials().removeAll(creds);
        return true;
    } else {
        return false;
    }
}
```

## CallbackHandler.java

`CallbackHandler` находится в отдельном исходном (.java) файле, отличном от любого конкретного `LoginModule`, чтобы он мог обслуживать множество `LoginModules` с различными объектами обратного вызова:

- Создает экземпляр класса `CallbackHandler` и имеет только один метод, `handle()`.
- `CallbackHandler`, обслуживающий `LoginModule`, требующий имя пользователя и пароль для входа:

```java
public void handle(Callback[] callbacks) {
    for (int i = 0; i < callbacks.length; i++) {
        Callback callback = callbacks[i];
        if (callback instanceof NameCallback) {
            NameCallback nameCallBack = (NameCallback) callback;
            nameCallBack.setName(username);
        } else if (callback instanceof PasswordCallback) {
            PasswordCallback passwordCallBack = (PasswordCallback) callback;
            passwordCallBack.setPassword(password.toCharArray());
        }
    }
}
```

## Связанные статьи

- [JAAS in Action](https://jaasbook.wordpress.com/2009/09/27/intro/), Michael Coté, опубликовано 27 сентября 2009 года, URL по состоянию на 5/14/2012.
- Пистоя Марко, Нагаратнам Натарадж, Ковед Ларри, Надалин Энтони из книги ["Enterprise Java Security" - Addison-Wesley, 2004](https://www.oreilly.com/library/view/enterprise-javatm-security/0321118898/).

## Примечание

Весь код в этой шпаргалке был скопирован дословно из этого [бесплатного источника](https://jaasbook.wordpress.com/2009/09/27/intro/).
