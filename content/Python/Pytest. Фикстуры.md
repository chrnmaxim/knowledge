---
title: Pytest
draft: false
tags:
  - Python
  - pytest
  - asyncio
---
Здесь собраны полезные фикстуры Pytest для создания тестов FastAPI приложения в асинхронном режиме с подключением к тестовой БД PostgreSQL.
## Работа с БД
### Создание экземпляра движка

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import NullPool

from src.database import Base


engine_test = create_async_engine(
    url=<DATABASE_URL>,
    poolclass=NullPool,
)

async_session_maker = sessionmaker(
    bind=engine_test,
    class_=AsyncSession,
    expire_on_commit=False,
    autocommit=False,
    autoflush=False,
)
```

### Фикстура`prepare_database`
```python
@pytest.fixture(
    autouse=True,
    scope="module",
)

async def prepare_database():
    """
    Сбрасывает и создает таблицы в БД перед запуском каждого модуля с тестами.
   
    После выполнения всех тестов в модуле снова сбрасывает таблицы в БД.
    """

    async with engine_test.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with engine_test.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
```

#### Описание

- **`autouse=True`** — фикстура автоматически применяется ко всем тестам в модуле, её не нужно явно указывать в тестах.
- **`scope="module"`** — фикстура выполняется один раз перед запуском всех тестов внутри модуля и завершается после выполнения всех тестов из этого модуля.

#### Что делает

- **Перед запуском тестов**:
    - Подключается к базе данных через `engine_test`.
    - Удаляет все таблицы в БД с помощью `Base.metadata.drop_all`.
    - Создаёт новые таблицы с помощью `Base.metadata.create_all`.
- **После выполнения всех тестов в модуле**:
    - Снова удаляет все таблицы в базе данных.

Таким образом, фикстура гарантирует, что база данных всегда находится в чистом состоянии перед запуском и после завершения тестов в модуле

### Фикстура `session`

```python
@pytest.fixture(scope="function")

async def session():
	"""
    Подключается к Postgres, начинает транзакцию, затем
    связывает её с сессией с вложенной транзакцией.

    Вложенная транзакция позволяет изолировать изменения,
    так что они видны только в рамках текущего теста, но не сохраняются в БД.
	"""



    async with engine_test.connect() as conn:
        tsx = await conn.begin()
        async with async_session_maker(bind=conn) as _session:
            nested_tsx = await conn.begin_nested()
           
            yield _session
           
            if nested_tsx.is_active:
                await nested_tsx.rollback()
            await tsx.rollback()
```

#### Описание

- **`scope="function"`** — фикстура запускается перед каждым тестом и завершается после каждого теста.

#### Что делает

1. Открывает подключение к базе данных с помощью `engine_test.connect()`.
2. Запускает внешнюю транзакцию через `conn.begin()`.
3. Создаёт сессию с привязкой к этому соединению через `async_session_maker(bind=conn)`.
4. Запускает вложенную транзакцию через `conn.begin_nested()`. Вложенные транзакции полезны для того, чтобы изменения были видны только в текущем тесте, но не влияли на базу данных в целом.
5. Возвращает созданную сессию для использования в тесте.
6. После завершения теста:
    - Если вложенная транзакция ещё активна, она откатывается.
    - Откатывается внешняя транзакция, что гарантирует отсутствие изменений в базе данных после выполнения теста.

Эта фикстура обеспечивает изоляцию данных для каждого теста — изменения, сделанные в ходе выполнения теста, не сохраняются в базе данных.

----
📂 [[Python]]

Последнее изменение: 01.10.2024 16:22