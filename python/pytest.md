________________________________________________________________________

# СОДЕРЖАНИЕ #

- > **[ОБЩАЯ ТЕОРИЯ](#общая-теория)**
- > **[ПЕРВЫЕ ШАГИ](#первые-шаги)**
- > **[ОСНОВНОЙ ФУНКЦИОНАЛ](#основной-функционал)**
    - > **[МЕТКИ, МАРКЕРЫ, ФИЛЬТРАЦИЯ](#метки-маркеры-фильтрация)**
    - > **[ПАРАМЕТРИЗАЦИЯ](#параметризация)**
    - > **[ПРОВЕРКА ИСКЛЮЧЕНИЙ](#проверка-исключений)**
    - > **[КОНФИГУРИРОВАНИЕ (CONFTEST.PY)](#конфигурирование-conftestpy)**
        - > **[Фикстуры](#фикстуры)**
        - > **[Хуки](#хуки)**
    - > **[МОКИ](#моки)**
    - > **[ПОКРЫТИЕ И CI]()**
- > **[ИСТОЧНИКИ И ДОП. МАТЕРИАЛЫ](#источники-и-доп-материалы)**
________________________________________________________________________

# ОБЩАЯ ТЕОРИЯ #

*pytest* - фреймворк для тестирования на Python, пришедший на замену
библиотеке unittest и призванный упростить тест-е.

Принципы:

- *Простота синтаксиса*: тест - это обычная ф-ция и обычный assert.
- *Лучшие сообщения об ошибках*: расшифровка выражений в assert и
    наглядные tracebacks.
- *Фикстуры вместо классовой магии*: декларативная система
    подготовки/очистки состояния без наследования.
- *Параметризация*: один тест - много входов/ожидаемых выходов.
- *Плагины*: архитектура, которую можно расширять под любые сценарии
    (распараллеливание, ретраи, бенчмарки, отчеты и т. д.).
________________________________________________________________________

Термины:

- *Assert-интроспекция* - это обычная инструкция Python вида
    `assert <условие>[, <сообщение>]`, которая при падении показывает
    разбор выражения: левую/правую части, значения подвыражений,
    красивый дифф коллекций/строк.

- *Traceback* - это отчет о том, по какой цепочке ф-ций Python дошел до
    места, где произошло искл-е.

- *Фикстуры* - в pytest это ф-ции, которые готовят ресурсы для тестов
    (данные, подключения, конфиги), отдают его тесту как аргумент по
    имени и затем корректно убирают. Это удобный способ интегрировать в
    тест необх. зав-ти.

- *Mark‑функции* - это ярлыки для тестов в виде ф-ций‑декораторов. Ими
    помечают ф-ции тестов, чтобы искл-ть/выбирать при запуске, условно
    пропускать или ожидать падения, группировать и т.д.

- *Хуки* - это специальные ф-ции, с помощью которых можно настраивать
    поведение pytest без изменения самих тестов.

- *Моки* - это подменные объекты, которые имитируют поведение внешних
    зав-тей в тестах: БД, сеть, время, файл-система и т.п. С моками вы
    проверяете как ваш код взаимодействует с зав-тью (какие ф-ции
    вызвал, с какими аргументами), не трогая реал. мир.
________________________________________________________________________

# ПЕРВЫЕ ШАГИ #

Установка: `pip install pytest`.

Настройка через файл `pyproject.toml`:

```py
[tool.pytest.ini_options]
minversion = "6.0"
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
  "-vv",				        # подробный вывод хода теста
  "--import-mode=importlib",    # меньше проблем с импортами
  "-m \"not slow\""             # выбор (или исключение) маркеров
]
markers = ["slow: медленные тесты", "integration: тесты с БД"]
norecursedirs = ["*.egg", ".eggs", "dist", "build"]
cache_dir = ".pytest_cache"
```

Здесь:

- *minversion* - требуемая мин. версия pytest. 
- *testpaths* - дир-и, в которых pytest будет искать тесты. 
- *python_files*, *python_classes*, *python_functions* - шаблоны имен
    файлов, классов и ф-ций соотв., которые pytest будет считать
- *addopts* - доп. аргументы командной строки.
- *markers* - регистрация маркеров. Это позволяет удобно фильтровать
    тесты при запуске (например, `pytest -m "not slow"`). 
- *norecursedirs* - дир-и, которые pytest пропустит при сборе тестов
    (исключит из поиска). 
- *cache_dir* - дир-я для кэша pytest.
________________________________________________________________________

Простейший пример assert-теста:

```py
# tests/test_example.py

import pytest 

def add(a, b): 
    return a + b

def test_math():
    assert add(2, 3) == 5
```

И запуск: `pytest tests/test_example.py`.
________________________________________________________________________

*assert* - станд. Python-инструкция проверки соотв-я в отладочных и
тестовых целях. Поддерживает станд. операции сравнения, предикаты
принадлежности и т.д. по условиям.

При выводе pytest + assert пытаются вывести как можно более точный
отчет, где произошла ошибка, в т.ч. точное место в списке или строке,
где произошло расхождение. Поэтому в т.ч. не следует исп-ть доступный
для assert функционал кастомного сообщения - это заменит подробный отчет
от pytest и assert.

Кроме того, есть следующие советы:

- Явные условия в assert.
- Подробные `__repr__` для сложных объектов.
- Один тест - одна идея. Исп-е нескольких assert допустимо, но несколько
    сценариев в одном тесте - моветон.
________________________________________________________________________

# ОСНОВНОЙ ФУНКЦИОНАЛ #

### МЕТКИ, МАРКЕРЫ, ФИЛЬТРАЦИЯ ###

В pyproject.toml есть возм-ть указать маркеры, которыми затем можно
группировать и фильтровать тесты для текущего исполнения.

Например, если в pyproject.toml есть запись:

```py
...
markers = [
  "slow: долгие тесты",
  "integration: интеграционные тесты",
  "network: сетевые тесты"
]
...
```

Тогда маркировать можно тесты следующими декораторами:

```py
@pytest.mark.slow
@pytest.mark.integration
@pytest.mark.network
```

А затем можно запустить определенные группы тестов и проигнорировать
другие. Например, команда `pytest -m "slow and not network"` запустит
долгие тесты с маркером slow, которые при этом не являются сетевыми с
маркером network.

Список маркеров доступен по команде `pytest --markers`.
________________________________________________________________________

Помимо именованных пользовательских маркеров есть встроенные служебные:

- `@pytest.mark.skip(reason="...")` - пропустить тест всегда.
- `@pytest.mark.skipif(условие, reason="...")` - пропустить при
    вып-и условия.
- `@pytest.mark.xfail(reason="...", strict=False)` - ожидаемый провал;
    не ломает прогон. strict=True делает "неожиданный успех" (XPASS)
    ошибкой.
- `@pytest.mark.parametrize(...)` - параметризация теста/фикстуры; можно
    помечать отдельные параметры через pytest.param(..., marks=...).
    Описан подробнее в разделе "Основной функционал" ->
    "Параметризация".
- `@pytest.mark.usefixtures("fix1", "fix2")` - подцепить фикстуры без
    явных аргументов.
- `@pytest.mark.filterwarnings("ignore::WarningType")` - локально
    подавить/поднять уровень предупреждений.
________________________________________________________________________

Можно также промаркировать тестовый класс или скрипт целиком. Например:

```py
pytestmark = [pytest.mark.integration]

class TestAPI:
    pytestmark = pytest.mark.network
```
________________________________________________________________________

### ПАРАМЕТРИЗАЦИЯ ###

Параметризация позволяет произвести одни и те же действия для разных
наборов данных без необх-ти писать отдельные тесты под каждый из них.

Это делается с помощью декоратора `pytest.mark.parametrize`, который
принимает строку с перечислением пар-ров ф-ции и список с тюплами
наборов принимаемых значений - по сути отдельными тестами.

Пример:

```py
import pytest

@pytest.mark.parametrize("a,b,expected", [
    (1, 1, 2),
    (2, 5, 7),
    (-1, 1, 0),
])
def test_add(a, b, expected):
    assert a + b == expected
```
________________________________________________________________________

Тестам можно присвоить элиасы:

```py
@pytest.mark.parametrize("a,b,expected", [
    (1, 1, 2),
    (2, 5, 7),
    (-1, 1, 0),
],
ids=["first", "second", "third"])
```

Или элиасы, вычисляемые с помощью ф-ции:

```py
def idfn(v) -> str:
    ...

@pytest.mark.parametrize("val", [
    1,
    4094,
    {"name": "alice", "role": "admin"},
], ids=idfn)
def test_example(val):
    assert val is not None
```
________________________________________________________________________

Есть возм-ть параметризировать через декартово произведение значений
пар-ров:

```py
@pytest.mark.parametrize("num1", [1, 2, 3])
@pytest.mark.parametrize("num2", [1, 2, 3])
def test_service(num1, num2):
    assert num1 + num2 == num1 + num2
```
________________________________________________________________________

Можно промаркировать и описать отедльные тест-кейсы:

```py
@pytest.mark.parametrize("mode", [
    "fast",
    pytest.param("slow", marks=pytest.mark.slow),
    pytest.param("offline", marks=pytest.mark.skip(reason="нет офлайна")),
])
def test_modes(mode):
    ...
```
________________________________________________________________________

### ПРОВЕРКА ИСКЛЮЧЕНИЙ ###

Проверка поднятия конкретного искл-я производится с помощью контекстного
менеджера `pytest.raises`.

Пример:

```py
import pytest

def check_num(num):
    if not isinstance(num, int):
        raise ValueError(f"invalid value {num}")

def test_raises():
    with pytest.raises(ValueError, match=r"invalid value \d+"):
        check_num('123123')
```
________________________________________________________________________

Для предупреждений - `pytest.warns`.

Пример:

```py
def test_warns():
    with pytest.warns(UserWarning, match="deprecated"):
        warnings.warn("deprecated API", UserWarning)
```
________________________________________________________________________

### КОНФИГУРИРОВАНИЕ (CONFTEST.PY) ###

Это локальный плагин pytest, который автоматически подхватывается для
каталога в котором лежит и для всех его подкаталогов. В нем обычно
держат фикстуры, хуки, свои CLI-опции, общие для этой группы тестов
настройки и проч.

Важно отметить, что этот файл не импортируется из тестов напрямую -
pytest подгружает его самостоятельно. Но в целом никто не мешает держать
свои фикстуры и ф-ции в отдельном модуле и импортировать вручную.

В conftest.py определенно не стоит класть тяжелые импорты и код с
побочными эффектами на уровне модуля. Ресурсы рекомендуется загружать
только с помощью фикстур. Также любые пользовательские утилиты не стоит
туда класть и лучше вынести в отдельный импортируемый модуль. 

То есть этот файл - это место, где складываются "эл-ты инфраструктуры"
для вып-я тестов на уровне текущей дир-и.
________________________________________________________________________

### <u> Фикстуры </u> ###

*Фикстуры* исп-ся для упрощения подготовки предварительных компонентов,
данных, подключений, конфигов и прочего. Это главный инструмент
подготовки к тестам. Эти данные отдаются в тест как аргумент по имени и
затем корректно убирает после себя.

Если pytest видит, что в ф-ции теста необходим аргумент с именем name,
то он идет искать фикстуру с таким именем. После осуществляет ее вызов
(один раз на область видимости), кэширует результат и передает его ч/
return/yield в тест. И после теста/модуля/сессии вып-ет teardown, т.е.
все что после yield или через addfinalizer.

Объявляется ч/ декоратор `pytest.fixture`.

Простой пример:

```py
@pytest.fixture
def ab():
    # фикстура готовит данные и возвращает их тестам
    return 2, 3

def test_add(ab):
    a, b = ab
    assert a + b == 5

def test_mul(ab):
    a, b = ab
    assert a * b == 6
```
________________________________________________________________________

Более приближенный к реал-ти пример:

```py
@pytest.fixture(scope="session")
def ssh_params():
    return {
        "host": "127.0.0.1",
        "user": "admin",
        "password": "pass",
        "port": "22"
    }

@pytest.fixture(scope="session")
def ssh(ssh_params):
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    kw = dict(
        hostname=ssh_params["host"],
        username=ssh_params["user"],
        port=ssh_params["port"],
        timeout=10,
        allow_agent=True,
        look_for_keys=False,
    )

    if ssh_params["password"]:
        kw["password"] = ssh_params["password"]

    client.connect(**kw)
    try:
        yield client  # передаем объект в тест
    finally:
        client.close()  # после теста закрываем сессию

@pytest.fixture
def run_ssh(ssh):
    """Функция-запускалка команд по SSH.
    
    Возвращает (rc, stdout, stderr).
    """
    def _run(cmd, timeout=10):
        stdin, stdout, stderr = ssh.exec_command(cmd, timeout=timeout)
        out = stdout.read().decode(errors="replace").strip()
        err = stderr.read().decode(errors="replace").strip()
        rc = stdout.channel.recv_exit_status()
        return rc, out, err
    return _run()

def test_hostname_matches_expected(run_ssh):
    expected_hostname = "localhost"
    rc, out, err = run_ssh("hostname")
    assert rc == 0, f"'hostname' завершилась с rc={rc}: {err}"
    assert out, "пустой вывод hostname"

    if expected_hostname:
        assert out == expected_hostname, f"ожидали {expected_hostname}, получили {out}"
    else:
        # если ожидание не задано - проверим "здравый" паттерн имени
        assert re.fullmatch(r"[A-Za-z0-9][A-Za-z0-9._-]*", out)
```
________________________________________________________________________

Параметр `scope` отвечает за область жизни фикстуры, т.е. как часто ее
нужно создавать и когда уничтожать. pytest кэширует результат фикстуры в
рамках ее scope. Выполнение кода после yield происходит когда область
жизни заканчивается.

Возможные значения scope:

- `function` (по умолчанию) - новая фикстура для каждого теста.
- `class` - одна фикстура на класс тестов (Test...).
- `module` - одна на скрипт (модуль) с тестами.
- `package` - одна на пакет (каталог с \_\_init\_\_.py и его подпакеты).
- `session` - одна на весь прогон pytest (сессию).

Если это дешевый/одноразовый ресурс - то используем function. Если это
"дорогие" подключения, типа SSH, к БД, HTTP-сессия - то лучше сделать
module и выше. Если это данные, которые нельзя переиспользовать между
тестами - то лучше оставить function или сделать ф-цию-конструктор,
которая будет создавать свежий, изолированный экземпляр и не будет
создавать "утечки состояния" между тест-кейсами.
________________________________________________________________________

### <u> Хуки </u> ###

*Хуки* - это спец. ф-ции-обработчики жизненного цикла тестов. pytest сам
вызывает их в определенные моменты (старт сессии, парсинг CLI, коллекция
тестов, параметризация, запуск, репортинг). С помощью них можно
настраивать поведение pytest без изменения тестов:
фильтровать/переименовывать тесты, добавлять опции командной строки,
параметризовать «на лету», вмешиваться в отчеты и т.д. К слову
parametrize - это частный образец такого хука.

Ф-ции называются по шаблону pytest_<имя_хука>, представляют собой
конкретные названия ф-ций и пишутся в conftest.py или плагинах.

Рукомендации:

- Их нужно делать быстрыми поскольку они вызываются очень часто.
- Всю логику, связанную с ресурсами, необходимо держать в фикстурах, а
    не в хуках.
- Хуки рекомендуется исп-ть только для глобальной политики проведения
    тестов, отчетности.
________________________________________________________________________

`pytest_addoption(parser)` - для добавления кастомных ключей запуска
тестов.

Пример:

```py
def pytest_addoption(parser):
    grp = parser.getgroup("ssh")
    grp.addoption("--ssh-host", help="SSH host")
    grp.addoption("--ssh-user", help="SSH username")
    grp.addoption("--ssh-password", help="SSH password")
    grp.addoption("--ssh-port", type=int, default=22, help="SSH port")
    grp.addoption("--expected-hostname", help="Expected hostname to assert")

@pytest.fixture(scope="session")
def ssh_params(pytestconfig):
    host = pytestconfig.getoption("--ssh-host") 
    user = pytestconfig.getoption("--ssh-user") 
    password = pytestconfig.getoption("--ssh-password") 
    port = pytestconfig.getoption("--ssh-port") 
    expected = pytestconfig.getoption("--expected-hostname")

    if not host or not user:
      pytest.skip("Нужно задать --ssh-host и --ssh-user")

    return {
        "host": host,
        "user": user,
        "password": password,
        "port": port,
        "expected": expected
    }
```

Здесь добавляются SSH-связанные ключи, а затем исп-ся в фикстуре.
________________________________________________________________________

`pytest_configure(config)` и `pytest_unconfigure(config)` исп-ся для
инициализации и освобождения соотв. глобальных объектов (например,
лог-файла, временной папки и т.п.).

Пример:

```py
import tempfile, os

global_tmp_dir = None

def pytest_configure(config):
    """Выполняется при старте pytest.
    
    Создаем временный каталог и сохраняем в объект config.
    """
    global global_tmp_dir
    global_tmp_dir = tempfile.mkdtemp(prefix="pytest_global_")
    config._global_tmp_dir = global_tmp_dir
    # Можно также регистрировать ini-lines: config.addinivalue_line(...)

def pytest_unconfigure(config):
    """Выполняется при завершении pytest.
    
    Чистим временный каталог.
    """
    global global_tmp_dir
    if global_tmp_dir and os.path.isdir(global_tmp_dir):
        try:
            import shutil
            shutil.rmtree(global_tmp_dir)
        finally:
            global_tmp_dir = None
```
________________________________________________________________________

`pytest_collection_modifyitems(config, items)` позволяет отфильтровать,
изменить, переупорядочить найденные тесты.

Например, можно изменить поведение в зависимости от наличия переданного
кастомного или встроенного ключа запуска тестов.

Другой пример - изменение порядка запуска тестов так, чтобы медленные
запускались последними с сохранением относительного порядка:

```py
def pytest_collection_modifyitems(config, items):
    """Переупорядочить тесты.

    Простая сортировка: быстрые первыми, потом медленные (с маркером
    'slow'). Стабильность порядка обеспечивается сравнением по пути и
    имени.
    """
    items.sort(
        key=lambda it: ("slow" in it.keywords, str(it.fspath), it.name)
    )
```
________________________________________________________________________

`pytest_generate_tests(metafunc)` позволяет осуществлять динамическую
параметризацию.

Пример:

```py
def pytest_generate_tests(metafunc):
    """Динамическая параметризация.

    Если тест запрашивает a, b и expected - подставляем набор простых
    арифметических кейсов.
    """
    if {"a", "b", "expected"} <= set(metafunc.fixturenames):
        cases = [
            (1, 2, 3),
            (2, 2, 4),
            (10, 5, 15),
            (0, 5, 5),
            (-1, 1, 0),
        ]
        ids = [f"{a}+{b}={exp}" for a, b, exp in cases]
        metafunc.parametrize(("a", "b", "expected"), cases, ids=ids)
```
________________________________________________________________________

`pytest_runtest_setup(item)`, `pytest_runtest_teardown(item)` -
вып-ются до и после всей сессии тест-я соотв. Т.е. можно настроить
предварительную подготовку или финальную чистку.

`pytest_runtest_call(item)` - вып-ется вокруг тестов, как бы оборачивая
его без декорирования, но с помощью доп. декоратора
`@pytest.hookimpl(hookwrapper=True)`.

`@pytest.hookimpl(hookwrapper=True)` - превращает весь хук в "обертку"
вокруг всей цепочки реализаций этого же хука - то есть можно исполнить
код до и после вып-я всех остальных обработчиков хука. Это делается
с помощью yield и все что до yield вып-ется перед основной работой хука,
а все, что после yield - после нее. То есть это своеобразный способ
сделать "around" обертку вокруг хука. С помощью него можно также
контролировать порядок вызова хуков.

Пример:

```py
import time

def pytest_runtest_setup(item):
    """Сетап-хук.

    Вызывается перед setup-частью теста. Здесь можно подготавливать
    окружение, проверять маркеры и т.п. Мы просто запомним время старта
    и напечатаем, что тест собирается запускаться.
    """
    item._runtest_start = time.time()
    print(f"\n[HOOK setup] Preparing to run: {item.nodeid}")


@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_call(item):
    """Хук-обертка.

    Обертка вокруг вып-я самого теста. Код до yield вып-ется до теста,
    код после yield - после теста. Это удобное место для измерения
    времени, перехвата искл-й и т.п.
    """
    print(f"[HOOK call] About to call test: {item.nodeid}")
    start = time.time()
    outcome = yield  # вып-е реал. теста происходит здесь
    duration = time.time() - start

    # Получаем информацию об исполнении (искл-е/успех) ч/
    # outcome.get_result() не дает report, но мы можем посмотреть, упало
    # ли искл-е во время вызова (ч/ outcome.exception() в new pytest).
    # Простая проверка: если тест выбросил искл-е, оно будет проброшено
    # дальше, но мы все равно логируем.
    print(f"[HOOK call] Finished call: {item.nodeid} (duration: {duration:.4f}s)")


def pytest_runtest_teardown(item, nextitem):
    """Тирдаун-хук.

    Вызывается после teardown-части теста. Здесь можно сделать финальную
    отчистку или логирование итоговой длительности.
    """
    total = None
    if hasattr(item, "_runtest_start"):
        total = time.time() - item._runtest_start
    print(f"[HOOK teardown] Completed: {item.nodeid} (total: {total:.4f}s)")
```
________________________________________________________________________

`pytest_runtest_makereport(item, call)` позволяет добавить кастомный лог
в объект отчета для каждого теста и, например, добавлять свои разделы.
Также исп-ся с декоратором `@pytest.hookimpl(hookwrapper=True)`.

Пример:

```py
@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    # hookwrapper дает возможность вып-ть код до/после получения rep
    outcome = yield
    rep = outcome.get_result()  # pytest.TestReport
    # интересует фаза "call" (сам тест), не setup/teardown
    if rep.when == "call" and rep.failed:
        # если тест использовал фикстуру 'run_ssh' - добавим ее
        # содержимое в секцию отчета
        if "run_ssh" in getattr(item, "funcargs", {}):
            fn = item.funcargs.get("run_ssh")
            if fn and hasattr(fn, "_paramiko_buf"):
                text = fn._paramiko_buf.getvalue()
                rep.sections.append((
                    "paramiko log",
                    text if text.strip() else "<empty>"
                ))
            else:
                rep.sections.append((
                    "paramiko log",
                    "run_ssh fixture not used"
                ))
```
________________________________________________________________________

`pytest_report_header(config)` - добавляет заголовок в начало запуска.

`pytest_terminal_summary(terminalreporter, exitstatus, config)` -
добавляет сводку в конце запуска.

Пример:

```py
def pytest_report_header(config):
    return f"Project: MyApp | ENV={config.getoption('--env') if config.getoption('--env', None) else 'default'}"

def pytest_terminal_summary(terminalreporter, exitstatus, config):
    tr = terminalreporter
    total = tr._numcollected if hasattr(tr, "_numcollected") else "?"
    passed = len(tr.stats.get("passed", []))
    failed = len(tr.stats.get("failed", []))
    skipped = len(tr.stats.get("skipped", []))
    tr.write_sep("-", f"Summary: collected={total} passed={passed} failed={failed} skipped={skipped}")
```
________________________________________________________________________

`pytest_assertrepr_compare(op, left, right)` позволяет переопределить
вывод assert на вариант, который нужен именно вам.

Пример:

```py
def pytest_assertrepr_compare(op, left, right):
    """Кастомный assert.

    Делает падения assert для чисел более информативными. Показываем
    левое/правое, разницу и относительную погрешность для float.
    """
    if op == "==" and isinstance(left, (int, float)) and isinstance(right, (int, float)):
        lines = ["numbers differ:"]
        lines.append(f"  left : {left!r}")
        lines.append(f"  right: {right!r}")
        diff = right - left
        lines.append(f"  diff (right - left): {diff!r}")
        if isinstance(left, float) or isinstance(right, float):
            denom = max(1.0, abs(right))  # чтобы не делить на 0
            rel = abs(diff) / denom
            lines.append(f"  abs diff: {abs(diff):.17g}, rel≈{rel:.3e} (vs right)")
            lines.append("  tip: for floats use pytest.approx(...)")
        return lines
```
________________________________________________________________________

### МОКИ ###

*Мок* - "заглушка" на место реал. внешнего объекта или источника,
которая имитирует прием, обработку и возврат ответа, а также помогает
анализировать поведение внешних объектов.

Для работы нужно установить pytest-mock: `pip install pytest-mock`.

Активируется при исп-нии автоматически без доп. импортов и передается ч/
фикстуру mocker.
________________________________________________________________________

Первое применение - исп-ние для простой "заглушки" с заменой на
фиксированное значение ч/ `mocker.stub`.

Главное правило здесь и для патчинга (см. далее) - необходимо применять
его там, где объект используется (импортирован), а не там, где он
определен. То есть если в app.py написано "from math import sqrt", то
цель будет "app.sqrt", а не "math.sqrt".

Пример с заменой на фиксированное время:

```py
# app_time.py
import time

def seconds_since_epoch() -> int:
    return int(time.time())


# tests/test_app_time.py
from app_time import seconds_since_epoch

def test_seconds_since_epoch_with_stub(mocker):
    time_stub = mocker.stub(name="time")
    time_stub.return_value = 1_700_000_000  # фиксированное "время"

    # в модуле app_time мы делали 'import time', значит патчим здесь
    mocker.patch("app_time.time.time", time_stub)

    assert seconds_since_epoch() == 1_700_000_000
    time_stub.assert_called_once_with()
```
________________________________________________________________________

Второе применение - *патчинг*, т.е. временная подмена
атрибута/функции/класса в точке исп-я на время теста. Т.е. более
продвинутая "заглушка". Исп-ся ч/ `mocker.patch`.

Пример замены JSON, получаемого через requests:

```py
# webapp.py
import requests

def get_title(url: str) -> str:
    resp = requests.get(url, timeout=1)
    resp.raise_for_status()
    return resp.json()["title"]


# tests/test_webapp.py
from webapp import get_title

def test_get_title_ok(mocker):
    # Патчим "там, где используется": webapp.requests.get
    mock_get = mocker.patch("webapp.requests.get")

    # Настраиваем фейковый ответ
    mock_resp = mocker.Mock()
    mock_resp.raise_for_status.return_value = None
    mock_resp.json.return_value = {"title": "Hello"}
    mock_get.return_value = mock_resp

    # Проверяем поведение
    assert get_title("https://example.com/api") == "Hello"
    mock_get.assert_called_once_with("https://example.com/api", timeout=1)
    mock_resp.raise_for_status.assert_called_once()
```
________________________________________________________________________

Третье применение - шпион, который обвешивает вызываемую ф-цию
средствами наблюдения без изменения реализации. Исп-ся ч/ `mocker.spy`.

Пример:

```py
import math

def use_sqrt(x: float) -> float:
    return math.sqrt(x)

def test_spy(mocker):
    spy = mocker.spy(math, "sqrt")
    assert use_sqrt(9) == 3
    assert spy.call_count == 1
    assert spy.spy_return == 3
```

Отедельные св-ва для отслеживания - см. документацию.
________________________________________________________________________

### ПОКРЫТИЕ И CI ###

- *Покрытие* - это процент кода, который тесты действительно прогоняют.
- *CI (Continuous Integration)* - часть CI/CD концепции, отвечающая за
    то, чтобы тесты автоматически запускались на каждый push, а не
    вручную.
________________________________________________________________________

Покрытие - не самоцель, но лишь хар-ка, насколько проект покрыт тестами.

Реализуется с помощью `pytest-cov`: `pip install pytest-cov`. Исп-ся с
помощью флага `--cov=<папка_проекта>` при запуске.

Выведет отчет при запуске с пар-рами:

- Stmts - сколько строк кода в файле.
- Miss - сколько не выполнилось во время тестов.
- Cover - процент покрытия.
________________________________________________________________________

CI - это автоматический запуск тестов (и любых других проверок) при
изменениях в репозитории. На GitHub это делает GitHub Actions, в GitLab
\- встроенный GitLab CI, есть еще CircleCI, Jenkins и др. Принцип у всех
один: на каждый git push запускается заданный pipeline.

В большинстве компаний для CI/CD исп-ся Jenkins и ему подобные, но в
контексте темы минимальный CI можно сделать ч/ GitHub Actions. Для этого
по пути `.github/workflows/tests.yml` в репозитории записать:

```
name: tests

on:
    push:
        branches: [main]
    pull_request:
        branches: [main]

jobs:
    test:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v4
            - uses: actions/setup-python@v5
              with:
                  python-version: '3.12'
            - run: pip install -r requirements.txt
            - run: pytest --cov=app
```

Эта запись говорит, что при пуше или пулл-реквесте на ветку main будет
происходить тест-е на Ubuntu след. шагами:

- Проверка.
- Установка Python 3.12.
- Установка зав-тей из requirements.txt.
- Запуск pytest.

Если хоть один тест не прошел при правильной настройке репозитория -
тогда не даст произвести пуш или пулл-реквест.
________________________________________________________________________

# ИСТОЧНИКИ И ДОП. МАТЕРИАЛЫ #

1. [Сайт] **Документация pytest**:
    [ссылка](https://docs.pytest.org/en/stable/index.html).
1. [Сайт] **Гайд на Хабре**:
    [ссылка](https://habr.com/ru/companies/beget/articles/948806/).
1. [Сайт] **Часть курса по Python про тестирование**:
    [ссылка](https://python-academy.org/ru/guide/pytest-basics).
________________________________________________________________________
