# Памятка по десериализации

## Введение

Эта статья сосредоточена на предоставлении ясных, практических рекомендаций для безопасной десериализации ненадежных данных в ваших приложениях.

## Что такое десериализация

**Сериализация** - это процесс преобразования объекта в формат данных, который можно восстановить позже. Люди часто сериализуют объекты, чтобы сохранить их для хранения или отправить в рамках коммуникаций.

**Десериализация** - это обратный процесс, который принимает данные, структурированные в некотором формате, и восстанавливает их в объект. Сегодня наиболее популярный формат данных для сериализации - это JSON. Ранее использовался XML.

Однако многие языки программирования имеют нативные способы сериализации объектов. Эти нативные форматы обычно предлагают больше возможностей, чем JSON или XML, включая настройку процесса сериализации.

К сожалению, возможности этих нативных механизмов десериализации иногда могут быть использованы в злонамеренных целях при работе с ненадежными данными. Атаки на десериализаторы могут позволять атаки типа отказ в обслуживании, нарушения контроля доступа или удаленного выполнения кода (RCE).

## Рекомендации по безопасной десериализации объектов

Следующие рекомендации, специфичные для языков программирования, пытаются перечислить безопасные методологии для десериализации данных, которым нельзя доверять.

### PHP

#### Открытое исследование

Проверьте использование функции [`unserialize()`](https://www.php.net/manual/en/function.unserialize.php) и проанализируйте, как принимаются внешние параметры. Используйте безопасный, стандартный формат обмена данными, такой как JSON (через `json_decode()` и `json_encode()`), если вам нужно передать сериализованные данные пользователю.

### Python

#### Непрозрачное исследование

Если в данных трафика есть символ точка `.` в конце, это очень вероятно указывает на то, что данные были отправлены в сериализованном виде. Это будет верно только если данные не закодированы с использованием схем Base64 или шестнадцатеричной системы. Если данные кодируются, лучше проверить, происходит ли сериализация, посмотрев на начальные символы значения параметра. Например, если данные закодированы в Base64, они, скорее всего, начнутся с `gASV`.

#### Открытое исследование

Следующие API в Python будут уязвимы к атакам через сериализацию. Поиск в коде по следующему шаблону.

1. Использование `pickle/c_pickle/_pickle` с `load/loads`:

```python
import pickle
data = """ cos.system(S'dir')tR. """
pickle.loads(data)

```

2. Использование `PyYAML` с`load`:

```python
import yaml
document = "!!python/object/apply:os.system ['ipconfig']"
print(yaml.load(document))
```

3. Использование `jsonpickle` с `encode` или `store`.

### Java

Следующие методы являются хорошими для предотвращения атак на десериализацию для [формата Serializable Java](https://docs.oracle.com/javase/7/docs/api/java/io/Serializable.html).

Рекомендации по реализации:

- В вашем коде переопределите метод `ObjectInputStream#resolveClass()`, чтобы предотвратить десериализацию произвольных классов. Это безопасное поведение можно обернуть в библиотеку, такую как [SerialKiller](https://github.com/ikkisoft/SerialKiller).
- Используйте безопасную замену для метода `readObject()`, как показано здесь. Обратите внимание, что это защищает от атак типа "[биллион смехов](https://en.wikipedia.org/wiki/Billion_laughs_attack)" путем проверки длины входных данных и количества десериализованных объектов.

#### Открытое исследование

Будьте внимательны к следующим использованиям Java API, которые могут указывать на потенциальную уязвимость сериализации.

1. `XMLdecoder` с внешними пользовательскими параметрами

2. `XStream` с методом `fromXML` (версия xstream <= v1.4.6 уязвима к проблеме сериализации)

3. `ObjectInputStream` с `readObject`

4. Использование `readObject`, `readObjectNoData`, `readResolve` или `readExternal`

5. `ObjectInputStream.readUnshared`

6. `Serializable`

#### Непрозрачное исследование

Если захваченные данные трафика включают следующие шаблоны, это может указывать на то, что данные были отправлены в потоках сериализации Java:

- `AC ED 00 05` в шестнадцатеричном виде
- `rO0` в Base64
- Заголовок `Content-type` HTTP-ответа установлен на `application/x-java-serialized-object`

#### Предотвращение утечки данных и перезаписи доверенных полей

Если в объекте есть члены данных, которые никогда не должны контролироваться пользователями во время десериализации или быть открытыми для пользователей во время сериализации, они должны быть объявлены с помощью [ключевого слова `transient`](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/serial-arch.html#7231) (раздел *Защита конфиденциальной информации*).

Для класса, определенного как Serializable, переменная с конфиденциальной информацией должна быть объявлена как `private transient`.

Например, для класса `myAccount`, переменные 'profit' и 'margin' были объявлены как transient, чтобы предотвратить их сериализацию.

```java
public class myAccount implements Serializable
{
    private transient double profit; // объявлено как transient

    private transient double margin; // объявлено как transient
    ...
...
```

#### Предотвращение десериализации доменных объектов

Некоторые объекты вашего приложения могут быть вынуждены реализовывать `Serializable` из-за их иерархии. Чтобы гарантировать, что объекты вашего приложения не могут быть десериализованы, следует объявить метод `readObject()` (с модификатором `final`), который всегда выбрасывает исключение:

```java
private final void readObject(ObjectInputStream in) throws java.io.IOException {
    throw new java.io.IOException("Cannot be deserialized");
}
```

#### Укрепление собственного java.io.ObjectInputStream

Класс `java.io.ObjectInputStream` используется для десериализации объектов. Его поведение можно усилить, создав подкласс. Это лучшее решение, если:

- вы можете изменить код, который выполняет десериализацию;
- вы знаете, какие классы вы ожидаете десериализовать.

Общая идея заключается в переопределении [`ObjectInputStream.html#resolveClass()`](http://docs.oracle.com/javase/7/docs/api/java/io/ObjectInputStream.html#resolveClass(java.io.ObjectStreamClass)) для ограничения того, какие классы могут быть десериализованы.

Поскольку этот вызов происходит до вызова `readObject()`, вы можете быть уверены, что никакая активность десериализации не произойдет, если тип не является разрешенным.

Простой пример показан здесь: класс `LookAheadObjectInputStream` гарантирует, что он **не** десериализует никакой другой тип, кроме класса `Bicycle`:

```java
public class LookAheadObjectInputStream extends ObjectInputStream {

    public LookAheadObjectInputStream(InputStream inputStream) throws IOException {
        super(inputStream);
    }

    /**
    * Десериализовать только экземпляры ожидаемого класса Bicycle
    */
    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
        if (!desc.getName().equals(Bicycle.class.getName())) {
            throw new InvalidClassException("Unauthorized deserialization attempt", desc.getName());
        }
        return super.resolveClass(desc);
    }
}
```

Более полные реализации этого подхода были предложены различными участниками сообщества:

- [NibbleSec](https://github.com/ikkisoft/SerialKiller) - библиотека, которая позволяет создавать списки классов, которые можно десериализовать
- [IBM](https://www.ibm.com/developerworks/library/se-lookahead/) - первичное средство защиты, написанное за много лет до того, как были предсказаны наиболее разрушительные сценарии эксплуатации.
- [Apache Commons IO classes](https://commons.apache.org/proper/commons-io/javadocs/api-2.5/org/apache/commons/io/serialization/ValidatingObjectInputStream.html)

#### Укрепление использования всех java.io.ObjectInputStream с помощью агента

Как упоминалось выше, класс `java.io.ObjectInputStream` используется для десериализации объектов. Его поведение можно усилить, создав подкласс. Однако, если вы не владеете кодом или не можете дождаться патча, использование агента для внедрения защиты в `java.io.ObjectInputStream` является лучшим решением.

Глобальное изменение `ObjectInputStream` безопасно только для блокировки известных вредоносных типов, поскольку невозможно знать, какие классы ожидаются для десериализации во всех приложениях. К счастью, в настоящее время существует очень мало классов, которые нужно включить в список отказов, чтобы быть защищенным от всех известных векторов атак.

Неизбежно, что будут обнаружены новые "гаджеты", которые могут быть злоупотреблены. Тем не менее, существует огромное количество уязвимого программного обеспечения, которое требует исправления. В некоторых случаях "исправление" уязвимости может включать переработку систем обмена сообщениями и нарушение обратной совместимости, поскольку разработчики переходят к отказу от принятия сериализованных объектов.

Чтобы включить эти агенты, просто добавьте новый параметр JVM:

```text
-javaagent:name-of-agent.jar
```

Агенты, использующие этот подход, были выпущены различными участниками сообщества:

- [rO0 от Contrast Security](https://github.com/Contrast-Security-OSS/contrast-rO0)

Похожие, но менее масштабируемые подходы включают ручное исправление и загрузку `ObjectInputStream` вашего JVM. Руководство по этому подходу доступно [здесь](https://github.com/wsargent/paranoid-java-serialization).

#### Другие библиотеки и форматы десериализации

Хотя приведенные выше советы сосредоточены на формате [Serializable Java](https://docs.oracle.com/javase/7/docs/api/java/io/Serializable.html), существует множество других библиотек, использующих другие форматы для десериализации. Многие из этих библиотек могут иметь аналогичные проблемы безопасности, если не настроены правильно. В этом разделе перечислены некоторые из этих библиотек и рекомендуемые параметры конфигурации для избежания проблем безопасности при десериализации непроверенных данных:

**Может использоваться безопасно с настройками по умолчанию:**

Следующие библиотеки могут использоваться безопасно с настройками по умолчанию:

- **[fastjson2](https://github.com/alibaba/fastjson2)** (JSON) - может использоваться безопасно, если опция [**autotype**](https://github.com/alibaba/fastjson2/wiki/fastjson2_autotype_cn) не включена
- **[jackson-databind](https://github.com/FasterXML/jackson-databind)** (JSON) - может использоваться безопасно, если не используется полиморфизм ([см. блог](https://cowtowncoder.medium.com/on-jackson-cves-dont-panic-here-is-what-you-need-to-know-54cd0d6e8062))
- **[Kryo v5.0.0+](https://github.com/EsotericSoftware/kryo)** (пользовательский формат) - может использоваться безопасно, если регистрация классов не отключена ([см. документацию](https://github.com/EsotericSoftware/kryo#optional-registration) и [эту проблему](https://github.com/EsotericSoftware/kryo/issues/929))
- **[YamlBeans v1.16+](https://github.com/EsotericSoftware/yamlbeans)** (YAML) - может использоваться безопасно, если класс **UnsafeYamlConfig** не используется (см. [этот коммит](https://github.com/EsotericSoftware/yamlbeans/commit/b1122588e7610ae4e0d516c50d08c94ee87946e6))
    - _ПРИМЕЧАНИЕ: поскольку эти версии недоступны в Maven Central, существует [форк](https://github.com/Contrast-Security-OSS/yamlbeans), который можно использовать вместо этого._
- **[XStream v1.4.17+](https://x-stream.github.io/)** (JSON и XML) - может использоваться безопасно, если список разрешенных и другие средства защиты не ослаблены ([см. документацию](https://x-stream.github.io/security.html))

**Требует настройки перед безопасным использованием:**

Следующие библиотеки требуют настройки параметров перед тем, как их можно будет использовать безопасно:

- **[fastjson v1.2.68+](https://github.com/alibaba/fastjson)** (JSON) - не может использоваться безопасно, если опция [**safemode**](https://github.com/alibaba/fastjson/wiki/fastjson_safemode_en) не включена, что отключает десериализацию любого класса ([см. документацию](https://github.com/alibaba/fastjson/wiki/enable_autotype)). Более ранние версии небезопасны.
- **[json-io](https://github.com/jdereg/json-io)** (JSON) - не может использоваться безопасно, поскольку использование свойства **@type** в JSON позволяет десериализовать любой класс. Можно использовать безопасно только в следующих ситуациях:
    - В [не типизированном режиме](https://github.com/jdereg/json-io/blob/master/user-guide.md#non-typed-usage) с использованием настройки **JsonReader.USE_MAPS**, которая отключает десериализацию обобщенных объектов.
    - [С использованием пользовательского десериализатора](https://github.com/jdereg/json-io/blob/master/user-guide.md#customization-technique-4-custom-serializer), контролирующего, какие классы будут десериализованы.
- **[Kryo < v5.0.0](https://github.com/EsotericSoftware/kryo)** (пользовательский формат) - не может использоваться безопасно, если регистрация классов не включена, что отключает десериализацию любого класса ([см. документацию](https://github.com/EsotericSoftware/kryo#optional-registration) и [эту проблему](https://github.com/EsotericSoftware/kryo/issues/929))
    - _ПРИМЕЧАНИЕ: существуют другие обертки вокруг Kryo, такие как [Chill](https://github.com/twitter/chill), которые также могут иметь регистрацию классов, не требуемую по умолчанию, независимо от используемой версии Kryo._
- **[SnakeYAML](https://bitbucket.org/snakeyaml/snakeyaml/src)** (YAML) - не может использоваться безопасно, если не используется класс **org.yaml.snakeyaml.constructor.SafeConstructor**, который отключает десериализацию любого класса ([см. документацию](https://bitbucket.org/snakeyaml/snakeyaml/wiki/CVE-2022-1471))

**Не может использоваться безопасно:**

Следующие библиотеки больше не поддерживаются или не могут использоваться безопасно с непроверенными входными данными:

- **[Castor](https://github.com/castor-data-binding/castor)** (XML) - похоже, что она заброшена, нет коммитов с 2016 года
- **[fastjson < v1.2.68](https://github.com/alibaba/fastjson)** (JSON) - эти версии позволяют десериализовать любой класс ([см. документацию](https://github.com/alibaba/fastjson/wiki/enable_autotype))
- **[XMLDecoder в JDK](https://docs.oracle.com/javase/8/docs/api/java/beans/XMLDecoder.html)** (XML) - _"почти невозможно безопасно десериализовать объекты Java в этом формате из непроверенных входных данных"_
("Руководство по защищенному программированию от Red Hat", [конец раздела 2.6.5](https://redhat-crypto.gitlab.io/defensive-coding-guide/#sect-Defensive_Coding-Tasks-Serialization-XML))
- **[XStream < v1.4.17](https://x-stream.github.io/)** (JSON и XML) - эти версии позволяют десериализовать любой класс (см. [документацию](https://x-stream.github.io/security.html#explicit))
- **[YamlBeans < v1.16](https://github.com/EsotericSoftware/yamlbeans)** (YAML) - эти версии позволяют десериализовать любой класс (см. [этот документ](https://github.com/Contrast-Security-OSS/yamlbeans/blob/main/SECURITY.md))

### .Net CSharp

#### Проверка исходного кода

Поиск в исходном коде следующих терминов:

1. `TypeNameHandling`
2. `JavaScriptTypeResolver`

Ищите сериализаторы, где тип устанавливается переменной, контролируемой пользователем.

#### Проверка в закрытом ящике

Ищите закодированный в base64 контент, начинающийся с:

```text
AAEAAAD/////
```

Ищите контент со следующим текстом:

1. `TypeObject`
2. `$type:`

#### Общие меры предосторожности

Microsoft заявила, что тип `BinaryFormatter` опасен и не может быть защищен. Поэтому его не следует использовать. Полные детали приведены в [руководстве по безопасности BinaryFormatter](https://docs.microsoft.com/en-us/dotnet/standard/serialization/binaryformatter-security-guide).

Не позволяйте потоку данных определять тип объекта, который будет десериализован. Вы можете предотвратить это, например, используя `DataContractSerializer` или `XmlSerializer`, если это возможно.

Если используется `JSON.Net`, убедитесь, что `TypeNameHandling` установлен только в `None`.

```csharp
TypeNameHandling = TypeNameHandling.None
```

Если используется `JavaScriptSerializer`, не используйте его с `JavaScriptTypeResolver`.

Если вам необходимо десериализовать потоки данных, которые определяют свой тип, ограничьте типы, которые могут быть десериализованы. Необходимо учитывать, что это все равно рискованно, поскольку многие встроенные типы .Net потенциально опасны сами по себе. Например:

```csharp
System.IO.FileInfo
```

Объекты `FileInfo`, которые ссылаются на файлы, фактически находящиеся на сервере, могут изменить свойства этих файлов, например, сделать их доступными только для чтения, что создаёт потенциальную атаку типа отказ в обслуживании.

Даже если вы ограничили типы, которые можно десериализовать, помните, что некоторые типы имеют свойства, которые могут быть рискованными. Например, `System.ComponentModel.DataAnnotations.ValidationException` имеет свойство `Value` типа `Object`. Если этот тип разрешен для десериализации, злоумышленник может установить свойство `Value` на любой выбранный им тип объекта.

Не позволяйте злоумышленникам управлять типом, который будет создан. Если это возможно, даже `DataContractSerializer` или `XmlSerializer` могут быть использованы злоумышленниками, например:

```csharp
// Действие ниже опасно, если злоумышленник может изменить данные в базе данных
var typename = GetTransactionTypeFromDatabase();

var serializer = new DataContractJsonSerializer(Type.GetType(typename));

var obj = serializer.ReadObject(ms);
```

Исполнение может произойти в некоторых типах .Net во время десериализации. Создание контроля, как показано ниже, неэффективно.

```csharp
var suspectObject = myBinaryFormatter.Deserialize(untrustedData);

// Проверка ниже слишком поздно! Исполнение могло уже произойти.
if (suspectObject is SomeDangerousObjectType)
{
    // Генерировать предупреждения и удалить suspectObject
}
```

Для `JSON.Net` возможно создать более безопасную форму контроля с использованием пользовательского `SerializationBinder`.

Старайтесь быть в курсе известных уязвимостей десериализации .NET и обращайте особое внимание на места, где такие типы могут быть созданы вашими процессами десериализации. **Десериализатор может создавать только те типы, о которых он знает**.

Старайтесь держать любой код, который может создавать потенциальные гаджеты, отдельно от кода, имеющего подключение к интернету. Например, `System.Windows.Data.ObjectDataProvider`, используемый в приложениях WPF, является известным гаджетом, который позволяет произвольное выполнение методов. Рискованно иметь ссылку на эту сборку в проекте REST-сервиса, который десериализует непроверенные данные.

#### Известные гаджеты RCE для .NET

- `System.Configuration.Install.AssemblyInstaller`
- `System.Activities.Presentation.WorkflowDesigner`
- `System.Windows.ResourceDictionary`
- `System.Windows.Data.ObjectDataProvider`
- `System.Windows.Forms.BindingSource`
- `Microsoft.Exchange.Management.SystemManager.WinForms.ExchangeSettingsProvider`
- `System.Data.DataViewManager, System.Xml.XmlDocument/XmlDataDocument`
- `System.Management.Automation.PSObject`

## Универсальные методы безопасной десериализации

### Использование альтернативных форматов данных

Значительное снижение риска достигается путем избегания родных (де)сериализационных форматов. Переключившись на чистый формат данных, такой как JSON или XML, вы снижаете вероятность того, что логика кастомной десериализации будет использована в злонамеренных целях.

Многие приложения полагаются на [паттерн передачи данных](https://en.wikipedia.org/wiki/Data_transfer_object), который включает создание отдельной области объектов для явной передачи данных. Конечно, всё ещё возможно, что приложение сделает ошибки безопасности после того, как чистый объект данных будет разобран.

### Десериализация только подписанных данных

Если приложение знает заранее, какие сообщения будут обработаны, оно может подписать их как часть процесса сериализации. Тогда приложение может выбрать не десериализовать ни одно сообщение, которое не имеет аутентифицированной подписи.

## Инструменты/библиотеки для предотвращения

- [Библиотека безопасности десериализации Java](https://github.com/ikkisoft/SerialKiller)
- [SWAT - инструмент для создания разрешенных списков](https://github.com/cschneider4711/SWAT)
- [NotSoSerial](https://github.com/kantega/notsoserial)

## Инструменты для обнаружения

- [Чек-лист десериализации Java, ориентированный на тестеров](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet)
- [Инструмент демонстрационного примера для генерации полезных нагрузок, которые используют небезопасную десериализацию объектов Java.](https://github.com/frohoff/ysoserial)
- [Наборы инструментов для десериализации Java](https://github.com/brianwrf/hackUtils)
- [Инструмент десериализации Java](https://github.com/frohoff/ysoserial)
- [Генератор полезных нагрузок .Net](https://github.com/pwntester/ysoserial.net)
- [Расширение Burp Suite](https://github.com/federicodotta/Java-Deserialization-Scanner/releases)
- [Библиотека безопасности десериализации Java](https://github.com/ikkisoft/SerialKiller)
- [Serianalyzer - статический анализатор байт-кода для десериализации](https://github.com/mbechler/serianalyzer)
- [Генератор полезных нагрузок](https://github.com/mbechler/marshalsec)
- [Тестировщик уязвимостей десериализации Android Java](https://github.com/modzero/modjoda)
- Расширения Burp Suite
    - [JavaSerialKiller](https://github.com/NetSPI/JavaSerialKiller)
    - [Java Deserialization Scanner](https://github.com/federicodotta/Java-Deserialization-Scanner)
    - [Burp-ysoserial](https://github.com/summitt/burp-ysoserial)
    - [SuperSerial](https://github.com/DirectDefense/SuperSerial)
    - [SuperSerial-Active](https://github.com/DirectDefense/SuperSerial-Active)

## Ссылки

- [Чек-лист по десериализации Java](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet)
- [Десериализация непроверенных данных](https://owasp.org/www-community/vulnerabilities/Deserialization_of_untrusted_data)
- [Атаки на десериализацию Java - German OWASP Day 2016](../assets/Deserialization_Cheat_Sheet_GOD16Deserialization.pdf)
- [AppSecCali 2015 - Marshalling Pickles](http://www.slideshare.net/frohoff1/appseccali-2015-marshalling-pickles)
- [FoxGlove Security - Объявление о уязвимости](http://foxglovesecurity.com/2015/11/06/what-do-weblogic-websphere-jboss-jenkins-opennms-and-your-application-have-in-common-this-vulnerability/#websphere)
- [Чек-лист по десериализации Java, ориентированный на тестеров](https://github.com/GrrrDog/Java-Deserialization-Cheat-Sheet)
- [Инструмент демонстрационного примера для генерации полезных нагрузок, которые используют небезопасную десериализацию объектов Java.](https://github.com/frohoff/ysoserial)
- [Наборы инструментов для десериализации Java](https://github.com/brianwrf/hackUtils)
- [Инструмент десериализации Java](https://github.com/frohoff/ysoserial)
- [Расширение Burp Suite](https://github.com/federicodotta/Java-Deserialization-Scanner/releases)
- [Библиотека безопасности десериализации Java](https://github.com/ikkisoft/SerialKiller)
- [Serianalyzer - статический анализатор байт-кода для десериализации](https://github.com/mbechler/serianalyzer)
- [Генератор полезных нагрузок](https://github.com/mbechler/marshalsec)
- [Тестировщик уязвимостей десериализации Android Java](https://github.com/modzero/modjoda)
- Расширения Burp Suite
    - [JavaSerialKiller](https://github.com/NetSPI/JavaSerialKiller)
    - [Java Deserialization Scanner](https://github.com/federicodotta/Java-Deserialization-Scanner)
    - [Burp-ysoserial](https://github.com/summitt/burp-ysoserial)
    - [SuperSerial](https://github.com/DirectDefense/SuperSerial)
    - [SuperSerial-Active](https://github.com/DirectDefense/SuperSerial-Active)
- .Net
    - [Alvaro Muñoz: .NET Serialization: Detecting and defending vulnerable endpoints](https://www.youtube.com/watch?v=qDoBlLwREYk)
    - [James Forshaw - Black Hat USA 2012 - Are You My Type? Breaking .net Sandboxes Through Serialization](https://www.youtube.com/watch?v=Xfbu-pQ1tIc)
    - [Jonathan Birch BlueHat v17 - Dangerous Contents - Securing .Net Deserialization](https://www.youtube.com/watch?v=oxlD8VWWHE8)
    - [Alvaro Muñoz & Oleksandr Mirosh - Friday the 13th: Attacking JSON - AppSecUSA 2017](https://www.youtube.com/watch?v=NqHsaVhlxAQ)
- Python
    - [Эксплуатация уязвимостей небезопасной десериализации, обнаруженных в дикой природе (Python Pickles)](https://macrosec.tech/index.php/2021/06/29/exploiting-insecuredeserialization-bugs-found-in-the-wild-python-pickles.)