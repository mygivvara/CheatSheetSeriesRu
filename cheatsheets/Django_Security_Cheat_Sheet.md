# Шпаргалка по безопасности для Django

## Введение

Фреймворк Django — это мощный веб-фреймворк на Python, который поставляется с встроенными функциями безопасности, которые можно использовать прямо из коробки для предотвращения распространенных веб-уязвимостей. Эта шпаргалка перечисляет действия и советы по безопасности, которые разработчики могут предпринять для разработки безопасных приложений на Django. Он направлен на покрытие распространенных уязвимостей, чтобы повысить уровень безопасности вашего приложения на Django. Каждый пункт содержит краткое объяснение и соответствующие примеры кода, специфичные для среды Django.

Фреймворк Django предоставляет некоторые встроенные функции безопасности, которые нацелены на безопасность по умолчанию. Эти функции также гибкие, чтобы разработчик мог повторно использовать компоненты для сложных сценариев. Это открывает сценарии, где разработчики, не знакомые с внутренними workings компонентов, могут настраивать их небезопасным образом. Эта шпаргалка направлен на перечисление таких случаев.

## Общие рекомендации

- Всегда поддерживайте Django и зависимости вашего приложения в актуальном состоянии, чтобы справляться с уязвимостями безопасности.
- Убедитесь, что приложение никогда не находится в режиме `DEBUG` в рабочей среде. Никогда не запускайте `DEBUG = True` в продакшн.
- Используйте пакеты, такие как [`django_ratelimit`](https://django-ratelimit.readthedocs.io/en/stable/) или [`django-axes`](https://django-axes.readthedocs.io/en/latest/index.html), чтобы предотвратить атаки грубой силы.

## Аутентификация

- Используйте приложение `django.contrib.auth` для представлений и форм для операций аутентификации пользователей, таких как вход, выход, смена пароля и т. д. Включите модуль и его зависимости `django.contrib.contenttypes` и `django.contrib.sessions` в настройку `INSTALLED_APPS` в файле `settings.py`.

  ```python
  INSTALLED_APPS = [
      # ...
      'django.contrib.auth',
      'django.contrib.contenttypes',
      'django.contrib.sessions',
      # ...
  ]
  ```

- Используйте декоратор `@login_required`, чтобы убедиться, что только аутентифицированные пользователи могут получить доступ к представлению. Пример кода ниже иллюстрирует использование `@login_required`.

  ```python
  from django.contrib.auth.decorators import login_required

  # Пользователь перенаправляется на страницу входа по умолчанию, если не аутентифицирован.
  @login_required
  def my_view(request):
    # Логика представления

  # Пользователь перенаправляется на пользовательскую '/login-page/', если не аутентифицирован.
  @login_required(login_url='/login-page/')
  def my_view(request):
    # Логика представления
  ```

- Используйте валидаторы паролей для соблюдения политик паролей. Добавьте или обновите настройку `AUTH_PASSWORD_VALIDATORS` в файле `settings.py`, чтобы включить специфические валидаторы, необходимые вашему приложению.

  ```python
  AUTH_PASSWORD_VALIDATORS = [
    {
      # Проверяет сходство между паролем и набором атрибутов пользователя.
      'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
      'OPTIONS': {
        'user_attributes': ('username', 'email', 'first_name', 'last_name'),
        'max_similarity': 0.7,
      }
    },
    {
      # Проверяет, соответствует ли пароль минимальной длине.
      'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
      'OPTIONS': {
        'min_length': 8,
      }
    },
    {
      # Проверяет, присутствует ли пароль в списке общих паролей.
      'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
      # Проверяет, не является ли пароль полностью числовым.
      'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    }
  ]
  ```

- Храните пароли, используя утилиту `make_password`, чтобы хешировать обычный текстовый пароль.

  ```python
  from django.contrib.auth.hashers import make_password
  #...
  hashed_pwd = make_password('plaintext_password')
  ```

- Проверьте текстовый пароль на соответствие хешированному паролю, используя утилиту `check_password`.

  ```python
  from django.contrib.auth.hashers import check_password
  #...
  plain_pwd = 'plaintext_password'
  hashed_pwd = 'hashed_password_from_database'

  if check_password(plain_pwd, hashed_pwd):
    print("Пароль верен.")
  else:
    print("Пароль неверен.")
  ```

## Управление ключами

Параметр `SECRET_KEY` в `settings.py` используется для криптографической подписи и должен быть конфиденциальным. Рассмотрите следующие рекомендации:

- Генерируйте ключ длиной не менее 50 символов, содержащий смесь букв, цифр и символов.
- Убедитесь, что `SECRET_KEY` генерируется с использованием сильного генератора случайных чисел, такого как функция `get_random_secret_key()` в Django.
- Избегайте жесткой кодировки значения `SECRET_KEY` в `settings.py` или любом другом месте. Рассмотрите возможность хранения ключа в переменных окружения или менеджерах секретов.

  ```python
  import os
  SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY')
  ```

- Регулярно меняйте ключ, помня о том, что это действие может сделать недействительными сеансы, токены сброса пароля и т. д. Смените ключ немедленно, если он будет скомпрометирован.

## Заголовки

Включите модуль `django.middleware.security.SecurityMiddleware` в настройку `MIDDLEWARE` в `settings.py` вашего проекта, чтобы добавить заголовки, связанные с безопасностью, в ваши ответы. Этот модуль используется для установки следующих параметров:

- `SECURE_CONTENT_TYPE_NOSNIFF`: Установите этот параметр в `True`. Защищает от атак по типу MIME, включая заголовок `X-Content-Type-Options: nosniff`.
- `SECURE_BROWSER_XSS_FILTER`: Установите этот параметр в `True`. Включает фильтр XSS браузера, устанавливая заголовок `X-XSS-Protection: 1; mode=block`.
- `SECURE_HSTS_SECONDS`: Обеспечивает доступ к сайту только через HTTPS.

Включите модуль `django.middleware.clickjacking.XFrameOptionsMiddleware` в настройку `MIDDLEWARE` в `settings.py` вашего проекта (Этот модуль должен быть указан после модуля `django.middleware.security.SecurityMiddleware`, так как порядок имеет значение). Этот модуль используется для установки следующих параметров:

- `X_FRAME_OPTIONS`: Установите этот параметр в 'DENY' или 'SAMEORIGIN'. Эта настройка добавляет заголовок `X-Frame-Options` ко всем HTTP-ответам. Это защищает от атак clickjacking.

## Cookies

- `SESSION_COOKIE_SECURE`: Установите этот параметр в `True` в файле `settings.py`. Это будет отправлять cookie сеанса только через защищенные (HTTPS) соединения.
- `CSRF_COOKIE_SECURE`: Установите этот параметр в `True` в файле `settings.py`. Это обеспечит отправку CSRF cookie только через защищенные соединения.
- Когда вы устанавливаете пользовательскую cookie в представлении, используя метод `HttpResponse.set_cookie()`, убедитесь, что параметр `secure` установлен в `True`.

  ```python
  response = HttpResponse("Некоторый ответ")
  response.set_cookie('my_cookie', 'cookie_value', secure=True)
  ```

## Межсайтовая подделка запросов (CSRF)

- Включите модуль `django.middleware.csrf.CsrfViewMiddleware` в настройку `MIDDLEWARE` в `settings.py` вашего проекта, чтобы добавить заголовки, связанные с CSRF, в ваши ответы.
- В формах используйте тег шаблона `{% csrf_token %}`, чтобы включить CSRF токен. Пример показан ниже.

  ```html
  <form method="post">
      {% csrf_token %}
      <!-- Ваши поля формы здесь -->
  </form>
  ```

- Для AJAX-запросов необходимо извлечь CSRF токен для запроса до использования в AJAX-запросе.
- Дополнительные рекомендации и контрольные меры можно найти в документации Django по [защите от межсайтовой подделки запросов (CSRF)](https://docs.djangoproject.com/en/3.2/ref/csrf/).

## Межсайтовый скриптинг (XSS)

Рекомендации в этом разделе дополняют рекомендации по XSS, уже упомянутые ранее.

- Используйте встроенную систему шаблонов для рендеринга шаблонов в Django. Ознакомьтесь с документацией Django по [автоматическому экранированию HTML](https://docs.djangoproject.com/en/3.2/ref/templates/language/#automatic-html-escaping), чтобы узнать больше.
- Избегайте использования фильтров `safe`, `mark_safe` или `json_script` для отключения автоматического экранирования шаблонов

 Django. Эквивалентная функция в Python — это функция `make_safe()`. Ознакомьтесь с документацией фильтра шаблона [json_script](https://docs.djangoproject.com/en/3.2/ref/templates/builtins/#json-script0), чтобы узнать больше.
- Ознакомьтесь с документацией Django по [защите от межсайтовых скриптов (XSS)](https://docs.djangoproject.com/en/3.2/topics/security/#cross-site-scripting-xss-protection), чтобы узнать больше.

## HTTPS

- Включите модуль `django.middleware.security.SecurityMiddleware` в настройку `MIDDLEWARE` в `settings.py` вашего проекта, если он еще не добавлен.
- Установите `SECURE_SSL_REDIRECT = True` в файле `settings.py`, чтобы обеспечить, что вся связь осуществляется через HTTPS. Это автоматически перенаправит любые HTTP-запросы на HTTPS. Это также постоянное перенаправление 301, так что ваш браузер запомнит перенаправление для последующих запросов.
- Если ваше приложение Django находится за прокси-сервером или балансировщиком нагрузки, установите настройку `SECURE_PROXY_SSL_HEADER` в `TRUE`, чтобы Django мог определить протокол оригинального запроса. Для получения дополнительной информации см. [документацию по SECURE_PROXY_SSL_HEADER](https://docs.djangoproject.com/en/3.2/ref/settings/#secure-proxy-ssl-header).

## URL админ панели

Рекомендуется изменить URL по умолчанию, ведущий к админ панели (example.com/admin/), чтобы немного усложнить автоматизированные атаки. Вот как это сделать:

В папке с приложением по умолчанию в вашем проекте найдите файл `urls.py`, управляющий верхнеуровневыми URL. В файле измените переменную `urlpatterns`, список, так что URL, ведущий к `admin.site.urls`, отличается от "admin/". Этот подход добавляет дополнительный уровень безопасности, скрывая общий конечный пункт для административного доступа.

## Ссылки

Дополнительная документация -

- [Защита от Clickjacking](https://docs.djangoproject.com/en/3.2/topics/security/#clickjacking-protection)
- [Безопасность Middleware](https://docs.djangoproject.com/en/3.2/topics/security/#module-django.middleware.security)
