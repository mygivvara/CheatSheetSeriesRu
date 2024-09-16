# Шпаргалка по безопасности Docker

## Введение

Docker — это самая популярная технология контейнеризации. При правильном использовании она может повысить безопасность по сравнению с запуском приложений непосредственно на хост-системе. Однако определенные некорректные настройки могут снизить уровень безопасности или создать новые уязвимости.

Цель данной шпаргалки — предоставить простую и понятную информацию о распространенных ошибках безопасности и лучших практиках для защиты ваших Docker-контейнеров.

## Правила

### ПРАВИЛО \#0 - Обновляйте хост и Docker

Для защиты от известных уязвимостей контейнеров, таких как [Leaky Vessels](https://snyk.io/blog/cve-2024-21626-runc-process-cwd-container-breakout/), которые обычно приводят к получению атакующим root-доступа к хосту, крайне важно поддерживать актуальность как хоста, так и Docker. Это включает регулярное обновление ядра хоста и Docker Engine.

Контейнеры используют ядро хоста. Если ядро хоста уязвимо, то уязвимы и контейнеры. Например, эксплуатация уязвимости повышения привилегий ядра [Dirty COW](https://github.com/scumjr/dirtycow-vdso), выполненная внутри хорошо изолированного контейнера, все равно приведет к получению root-доступа на уязвимом хосте.

### ПРАВИЛО \#1 - Не открывайте сокет демона Docker (даже для контейнеров)

Сокет Docker */var/run/docker.sock* — это UNIX-сокет, к которому Docker подключается. Это основная точка входа для Docker API. Владелец этого сокета — root. Предоставление доступа к нему эквивалентно предоставлению неограниченного root-доступа к вашему хосту.

**Не включайте *tcp* сокет демона Docker.** Если вы запускаете демон Docker с `-H tcp://0.0.0.0:XXX` или подобным образом, вы открываете нешифрованный и неаутентифицированный прямой доступ к демону Docker. Если хост подключен к интернету, это означает, что демон Docker на вашем компьютере может использоваться любым пользователем из интернета. Если вам действительно, **действительно** нужно это сделать, вы должны обеспечить его безопасность. Ознакомьтесь с [официальной документацией Docker](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-socket-option), чтобы узнать, как это сделать.

**Не открывайте */var/run/docker.sock* для других контейнеров.** Если вы запускаете ваш Docker-образ с `-v /var/run/docker.sock://var/run/docker.sock` или подобным образом, вам следует изменить это. Помните, что монтирование сокета только для чтения не является решением, а лишь усложняет эксплуатацию. Эквивалент в файле docker-compose выглядит следующим образом:

```yaml
    volumes:
    - "/var/run/docker.sock:/var/run/docker.sock"
```

### ПРАВИЛО \#2 - Установите пользователя

Настройка контейнера для использования непривилегированного пользователя — это лучший способ предотвратить атаки повышения привилегий. Это можно сделать тремя различными способами:

1. Во время выполнения с помощью опции `-u` команды `docker run`, например:

```bash
docker run -u 4000 alpine
```

2. Во время сборки. Просто добавьте пользователя в Dockerfile и используйте его. Например:

```dockerfile
FROM alpine
RUN groupadd -r myuser && useradd -r -g myuser myuser
#    <ЗДЕСЬ ВЫПОЛНИТЕ ВСЕ НЕОБХОДИМЫЕ ДЕЙСТВИЯ В КАЧЕСТВЕ ПОЛЬЗОВАТЕЛЯ ROOT, НАПРИМЕР, УСТАНОВКУ ПАКЕТОВ И Т.Д.>
USER myuser
```

3. Включите поддержку пространства имен пользователей (`--userns-remap=default`) в [демоне Docker](https://docs.docker.com/engine/security/userns-remap/#enable-userns-remap-on-the-daemon)

Больше информации об этой теме можно найти в [официальной документации Docker](https://docs.docker.com/engine/security/userns-remap/). Для дополнительной безопасности вы также можете запускать в режиме без root-доступа, который обсуждается в [Правиле \#11](#правило-11---запускайте-docker-в-режиме-без-прав-суперпользователя-rootless-mode).

В Kubernetes это можно настроить в [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) с помощью поля `runAsUser` с идентификатором пользователя, например:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  containers:
  - name: example
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      runAsUser: 4000 # <-- Это идентификатор пользователя пода
```

Как администратор кластера Kubernetes, вы можете настроить жесткие параметры по умолчанию, используя уровень [`Restricted`](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) с встроенным [Pod Security admission controller](https://kubernetes.io/docs/concepts/security/pod-security-admission/). Если требуется более высокая степень настройки, рассмотрите возможность использования [Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks) или [альтернативного варианта](https://kubernetes.io/docs/concepts/security/pod-security-standards/#alternatives).

### ПРАВИЛО \#3 - Ограничьте возможности (предоставляйте только конкретные возможности, необходимые контейнеру)

[Возможности ядра Linux](http://man7.org/linux/man-pages/man7/capabilities.7.html) — это набор привилегий, которые могут использоваться привилегированными пользователями. По умолчанию Docker работает только с подмножеством возможностей. Вы можете изменить это и убрать некоторые возможности (используя `--cap-drop`), чтобы усилить безопасность ваших контейнеров, или добавить некоторые возможности (используя `--cap-add`), если это необходимо. Помните, что не следует запускать контейнеры с флагом `--privileged` — это добавит ВСЕ возможности ядра Linux в контейнер.

Самый безопасный вариант — это убрать все возможности `--cap-drop all`, а затем добавить только необходимые. Например:

```bash
docker run --cap-drop all --cap-add CHOWN alpine
```

**И помните: не запускайте контейнеры с флагом *--privileged*!!!**

В Kubernetes это можно настроить в [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) с помощью поля `capabilities`, например:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  containers:
  - name: example
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
        capabilities:
            drop:
                - ALL
            add: ["CHOWN"]
```

Как администратор кластера Kubernetes, вы можете настроить жесткие параметры по умолчанию, используя уровень [`Restricted`](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) с встроенным [Pod Security admission controller](https://kubernetes.io/docs/concepts/security/pod-security-admission/). Если требуется более высокая степень настройки, рассмотрите возможность использования [Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks) или [альтернативного варианта](https://kubernetes.io/docs/concepts/security/pod-security-standards/#alternatives).

### ПРАВИЛО \#4 - Предотвращение повышения привилегий внутри контейнера

Всегда запускайте ваши Docker-образы с `--security-opt=no-new-privileges`, чтобы предотвратить повышение привилегий. Это предотвратит получение новых привилегий контейнером через бинарные файлы `setuid` или `setgid`.

В Kubernetes это можно настроить в [Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) с помощью поля `allowPrivilegeEscalation`, например:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  containers:
  - name: example
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      allowPrivilegeEscalation: false
```

Как администратор кластера Kubernetes, вы можете настроить жесткие параметры по умолчанию, используя уровень [`Restricted`](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) с встроенным [Pod Security admission controller](https://kubernetes.io/docs/concepts/security/pod-security-admission/). Если требуется более высокая степень настройки, рассмотрите возможность использования [Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks) или [альтернативного варианта](https://kubernetes.io/docs/concepts/security/pod-security-standards/#alternatives).

### ПРАВИЛО \#5 - Обратите внимание на взаимосвязь контейнеров

Взаимосвязь контейнеров (ICC) включена по умолчанию, что позволяет всем контейнерам обмениваться данными друг с другом через [мостовую сеть `docker0`](https://docs.docker.com/network/drivers/bridge/). Вместо использования флага `--icc=false` для демона Docker, который полностью отключает коммуникацию между контейнерами, рассмотрите возможность определения конкретных сетевых конфигураций. Это можно сделать, создав собственные сети Docker и указав, к каким из них должны быть подключены контейнеры. Этот метод предоставляет более детальный контроль над коммуникацией контейнеров.

Для подробных рекомендаций по настройке сетей Docker для коммуникации между контейнерами обратитесь к [документации Docker](https://docs.docker.com/network/#communication-between-containers).

В средах Kubernetes можно использовать [Политики сети](https://kubernetes.io/docs/concepts/services-networking/network-policies/) для определения правил, регулирующих взаимодействие подов внутри кластера. Эти политики предоставляют надежную структуру для контроля того, как поды общаются друг с другом и с другими сетевыми конечными точками. Дополнительно, [Network Policy Editor](https://networkpolicy.io/) упрощает создание и управление сетевыми политиками, делая определение сложных сетевых правил более доступным через удобный интерфейс.

### ПРАВИЛО \#6 - Используйте модули безопасности Linux (seccomp, AppArmor или SELinux)

**Прежде всего, не отключайте профиль безопасности по умолчанию!**

Рассмотрите возможность использования профиля безопасности, такого как [seccomp](https://docs.docker.com/engine/security/seccomp/) или [AppArmor](https://docs.docker.com/engine/security/apparmor/).

Инструкции по этому поводу в Kubernetes можно найти в [Настройка контекста безопасности для пода или контейнера](https://kubernetes.io/docs/tutorials/security/seccomp/).

### ПРАВИЛО \#7 - Ограничьте ресурсы (память, процессор, файловые дескрипторы, процессы, перезапуски)

Лучший способ избежать атак типа DoS — ограничить ресурсы. Вы можете ограничить [память](https://docs.docker.com/config/containers/resource_constraints/#memory), [процессор](https://docs.docker.com/config/containers/resource_constraints/#cpu), максимальное количество перезапусков (`--restart=on-failure:<number_of_restarts>`), максимальное количество файловых дескрипторов (`--ulimit nofile=<number>`) и максимальное количество процессов (`--ulimit nproc=<number>`).

[Проверьте документацию для получения более детальной информации о ulimits](https://docs.docker.com/engine/reference/commandline/run/#set-ulimits-in-container---ulimit).

Эти ограничения также можно установить в Kubernetes: [Назначение ресурсов памяти для контейнеров и подов](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/), [Назначение ресурсов процессора для контейнеров и подов](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/) и [Назначение расширенных ресурсов контейнеру](https://kubernetes.io/docs/tasks/configure-pod-container/extended-resource/).

### ПРАВИЛО \#8 - Установите файловую систему и тома в режим только для чтения

**Запускайте контейнеры с файловой системой только для чтения**, используя флаг `--read-only`. Например:

```bash
docker run --read-only alpine sh -c 'echo "что угодно" > /tmp'
```

Если приложение внутри контейнера должно временно сохранять что-то, объедините флаг `--read-only` с `--tmpfs`, как показано ниже:

```bash
docker run --read-only --tmpfs /tmp alpine sh -c 'echo "что угодно" > /tmp/file'
```

Эквивалент в Docker Compose `compose.yml` будет следующим:

```yaml
version: "3"
services:
  alpine:
    image: alpine
    read_only: true
```

Эквивалент в Kubernetes в [Контексте безопасности](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  containers:
  - name: example
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      readOnlyRootFilesystem: true
```

Кроме того, если том смонтирован только для чтения, **монтируйте его как только для чтения**. Это можно сделать, добавив `:ro` к `-v`, как показано ниже:

```bash
docker run -v volume-name:/path/in/container:ro alpine
```

Или используя опцию `--mount`:

```bash
docker run --mount source=volume-name,destination=/path/in/container,readonly alpine
```

### ПРАВИЛО \#9 - Интегрируйте инструменты сканирования контейнеров в ваш CI/CD конвейер

[CI/CD конвейеры](CI_CD_Security_Cheat_Sheet.md) являются ключевой частью жизненного цикла разработки программного обеспечения и должны включать различные проверки безопасности, такие как проверка синтаксиса, статический анализ кода и сканирование контейнеров.

Многие проблемы можно предотвратить, следуя лучшим практикам при написании Dockerfile. Однако добавление проверщика безопасности как шага в конвейере сборки может значительно облегчить работу. Некоторые часто проверяемые проблемы включают:

- Убедитесь, что указана директива `USER`
- Убедитесь, что версия базового образа зафиксирована
- Убедитесь, что версии пакетов ОС зафиксированы
- Избегайте использования `ADD` в пользу `COPY`
- Избегайте использования curl в директивах `RUN`

Ссылки:

- [Базовые рекомендации Docker на DevSec](https://dev-sec.io/baselines/docker/)
- [Используйте командную строку Docker](https://docs.docker.com/engine/reference/commandline/cli/)
- [Обзор CLI Docker Compose v2](https://docs.docker.com/compose/reference/overview/)
- [Настройка драйверов логирования](https://docs.docker.com/config/containers/logging/configure/)
- [Просмотр логов для контейнера или сервиса](https://docs.docker.com/config/containers/logging/)
- [Лучшие практики безопасности Dockerfile](https://cloudberry.engineering/article/dockerfile-security-best-practices/)

Инструменты сканирования контейнеров особенно важны как часть успешной стратегии безопасности. Они могут обнаруживать известные уязвимости, секреты и неправильные конфигурации в образах контейнеров и предоставлять отчет о находках с рекомендациями по их исправлению. Некоторые примеры популярных инструментов сканирования контейнеров:

- Бесплатные
    - [Clair](https://github.com/coreos/clair)
    - [ThreatMapper](https://github.com/deepfence/ThreatMapper)
    - [Trivy](https://github.com/aquasecurity/trivy)
- Коммерческие
    - [Snyk](https://snyk.io/) **(доступна опция с открытым исходным кодом и бесплатная версия)**
    - [Anchore](https://github.com/anchore/grype/) **(доступна опция с открытым исходным кодом и бесплатная версия)**
    - [Docker Scout](https://www.docker.com/products/docker-scout/) **(доступна опция с открытым исходным кодом и бесплатная версия)**
    - [JFrog XRay](https://jfrog.com/xray/)
    - [Qualys](https://www.qualys.com/apps/container-security/)

Для обнаружения секретов в изображениях:

- [ggshield](https://github.com/GitGuardian/ggshield) **(доступна опция с открытым исходным кодом и бесплатная версия)**
- [SecretScanner](https://github.com/deepfence/SecretScanner) **(версия с открытым исходным кодом)**

Для обнаружения мисконфигураций в Kubernetes

- [kubeaudit](https://github.com/Shopify/kubeaudit)
- [kubesec.io](https://kubesec.io/)
- [kube-bench](https://github.com/aquasecurity/kube-bench)

Для обнаружения мисконфигурций в Docker

- [inspec.io](https://www.inspec.io/docs/reference/resources/docker/)
- [dev-sec.io](https://dev-sec.io/baselines/docker/)
- [Docker Bench for Security](https://github.com/docker/docker-bench-security)

### ПРАВИЛО \#10 - Сохраняйте уровень логирования демона Docker на уровне `info`

По умолчанию демон Docker настроен на уровень логирования `info`. Это можно проверить, проверив файл конфигурации демона `/etc/docker/daemon.json` на наличие ключа `log-level`. Если ключ отсутствует, уровень логирования по умолчанию — `info`. Кроме того, если демон Docker запускается с опцией `--log-level`, значение ключа `log-level` в конфигурационном файле будет переопределено. Чтобы проверить, работает ли демон Docker с другим уровнем логирования, можно использовать следующую команду:

```bash
ps aux | grep '[d]ockerd.*--log-level' | awk '{for(i=1;i<=NF;i++) if ($i ~ /--log-level/) print $i}'
```

Установка подходящего уровня логирования настраивает демон Docker на запись событий, которые вы можете просмотреть позже. Базовый уровень логирования `info` и выше будет захватывать все логи, кроме отладочных. Пока это не требуется, не следует запускать демон Docker на уровне `debug`.

### ПРАВИЛО \#11 - Запускайте Docker в режиме без прав суперпользователя (rootless mode)

Режим без прав суперпользователя гарантирует, что демон Docker и контейнеры работают от имени непривилегированного пользователя, что означает, что даже если злоумышленник выйдет из контейнера, он не получит привилегий root на хосте, что значительно ограничивает поверхность атаки. Это отличается от режима [userns-remap](#rule-2---set-a-user), где демон все еще работает с привилегиями root.

Оцените [конкретные требования](Attack_Surface_Analysis_Cheat_Sheet.md) и [постуру безопасности](Threat_Modeling_Cheat_Sheet.md) вашей среды, чтобы определить, является ли режим без прав суперпользователя лучшим выбором для вас. Для сред, где безопасность является первоочередной задачей, и [ограничения режима без прав суперпользователя](https://docs.docker.com/engine/security/rootless/#known-limitations) не мешают оперативным требованиям, это настоятельно рекомендуется. В качестве альтернативы рассмотрите использование [Podman](#podman-as-an-alternative-to-docker) вместо Docker.

> Режим без прав суперпользователя позволяет запускать демон Docker и контейнеры от имени пользователя без прав root, чтобы снизить потенциальные уязвимости в демоне и контейнерном окружении.
> Режим без прав суперпользователя не требует прав root даже во время установки демона Docker, при условии выполнения [предварительных условий](https://docs.docker.com/engine/security/rootless/#prerequisites).

Читайте больше о режиме без прав суперпользователя и его ограничениях, инструкциях по установке и использованию на странице [документации Docker](https://docs.docker.com/engine/security/rootless/).

### ПРАВИЛО \#12 - Используйте Docker Secrets для управления конфиденциальными данными

Docker Secrets предоставляет безопасный способ хранения и управления конфиденциальными данными, такими как пароли, токены и ключи SSH. Использование Docker Secrets помогает избежать раскрытия конфиденциальных данных в образах контейнеров или в командах выполнения.

```bash
docker secret create my_secret /path/to/super-secret-data.txt
docker service create --name web --secret my_secret nginx:latest
```

Или для Docker Compose:

```yaml
version: "3.8"
secrets:
  my_secret:
    file: ./super-secret-data.txt
services:
  web:
    image: nginx:latest
    secrets:
      - my_secret
```

Хотя Docker Secrets обычно обеспечивает безопасное управление конфиденциальными данными в среде Docker, этот подход не рекомендуется для Kubernetes, где секреты по умолчанию хранятся в открытом виде. В Kubernetes рассмотрите возможность использования дополнительных мер безопасности, таких как шифрование etcd или сторонние инструменты. См. [Cheat Sheet по управлению секретами](Secrets_Management_Cheat_Sheet.md) для получения дополнительной информации.

### ПРАВИЛО \#13 - Усильте безопасность цепочки поставок

Исходя из принципов [Правила \#9](#правило-9---Интегрируйте-инструменты-сканирования-контейнеров-в-ваш-CICD-конвейер), усиление безопасности цепочки поставок включает внедрение дополнительных мер для защиты всего жизненного цикла образов контейнеров от создания до развертывания. Некоторые ключевые практики включают:

- [Происхождение образов](https://slsa.dev/spec/v1.0/provenance): Документируйте происхождение и историю образов контейнеров для обеспечения прослеживаемости и целостности.
- [Генерация SBOM](https://cyclonedx.org/guides/CycloneDX%20One%20Pager.pdf): Создавайте программный акт материалов (SBOM) для каждого образа, описывающий все компоненты, библиотеки и зависимости для прозрачности и управления уязвимостями.
- [Подпись образов](https://github.com/notaryproject/notary): Цифровая подпись образов для проверки их целостности и подлинности, установление доверия к их безопасности.
- [Доверенный реестр](https://snyk.io/learn/container-security/container-registry-security/): Храните документированные, подписанные образы с их SBOM в безопасном реестре, который обеспечивает строгие [контроль доступа](Access_Control_Cheat_Sheet.md) и поддерживает управление метаданными.
- [Безопасное развертывание](https://www.openpolicyagent.org/docs/latest/#overview): Реализуйте политики безопасного развертывания, такие как проверка образов, безопасность в реальном времени и непрерывный мониторинг, чтобы обеспечить безопасность развернутых образов.

## Podman как альтернатива Docker

[Podman](https://podman.io/) — это совместимый с OCI, открытый инструмент управления контейнерами, разработанный [Red Hat](https://www.redhat.com/en), который предоставляет интерфейс командной строки, совместимый с Docker, и настольное приложение для управления контейнерами. Он предназначен как более безопасная и легковесная альтернатива Docker, особенно для сред, где предпочтительны безопасные настройки по умолчанию. Некоторые преимущества безопасности Podman включают:

1. Архитектура без демона: В отличие от Docker, который требует центрального демона (dockerd) для создания, запуска и управления контейнерами, Podman использует модель fork-exec. Когда пользователь запрашивает запуск контейнера, Podman создаёт новый процесс, который затем запускает контейнерное окружение.
2. Контейнеры без прав суперпользователя: Модель fork-exec облегчает возможность Podman запускать контейнеры без прав root. Когда не-root пользователь запускает контейнер, Podman работает с правами этого пользователя.
3. Интеграция с SELinux: Podman разработан для работы с SELinux, что обеспечивает дополнительный уровень безопасности за счёт применения обязательного контроля доступа к контейнерам и их взаимодействию с хост-системой.

## Ссылки и дополнительное чтение

[OWASP Docker Top 10](https://github.com/OWASP/Docker-Security)  
[Лучшие практики безопасности Docker](https://docs.docker.com/develop/security-best-practices/)  
[Безопасность Docker Engine](https://docs.docker.com/engine/security/)  
[Cheat Sheet по безопасности Kubernetes](Kubernetes_Security_Cheat_Sheet.md)  
[SLSA - Уровни цепочки поставок для программных артефактов](https://slsa.dev/)  
[Sigstore](https://sigstore.dev/)  
[Аттестация сборок Docker](https://docs.docker.com/build/attestations/)  
[Доверие к содержимому Docker](https://docs.docker.com/engine/security/trust/)