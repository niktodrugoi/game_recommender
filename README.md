<div align="center">

# 🎮 game_recommender

**Скажи, что любишь, и получи игры, в которые реально захочется играть.**

REST API, который подбирает видеоигры под вкус, кошелёк, платформу и возраст.

![Python](https://img.shields.io/badge/Python-3.10+-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-F7931E?logo=scikitlearn&logoColor=white)
![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy-D71F00)
![Tests](https://img.shields.io/badge/tests-pytest-0A9EDC?logo=pytest&logoColor=white)

</div>

---

## ✨ Что это

Обычные «топы игр» одинаковы для всех. Этот модуль работает иначе: он смотрит на **твои любимые жанры**, находит похожие игры в каталоге, а потом выкидывает всё, что тебе не подходит: не твоя платформа, слишком дорого, слишком долго, не по возрасту, не твой регион.

На выходе получаешь до 10 игр, у каждой есть оценка схожести и **человеческое объяснение, почему она тебе подошла**.

## 🧠 Как это работает

```
  жанры пользователя ─┐
                      ├─► TF-IDF ─► косинусная близость ─► ранжирование
  жанры игр ──────────┘                                         │
                                                                ▼
                                      фильтры: платформа · цена · время ·
                                               возраст · регион
                                                                │
                                      пусто? ─► fallback ◄──────┤
                                                                ▼
                                          топ-10 + объяснения для каждой
```

1. **Схожесть.** Описание каждой игры (название, жанры, платформа) и профиль пользователя (любимые жанры, платформа) превращаются в векторы через **TF-IDF**, а близость считается как **cosine similarity**.
2. **Контекст.** Строгие фильтры отсекают неподходящее: платформа и регион должны совпасть, цена, время прохождения и возрастной рейтинг не должны превышать лимиты пользователя.
3. **Запасной план.** Если фильтры вычистили всё, включается **fallback**: в выдачу попадают самые высокооценённые игры каталога, и ответ не будет пустым.
4. **Объяснение.** Для каждой игры генерируется текст, почему она в выдаче.

## 🚀 Быстрый старт

```bash
git clone https://github.com/niktodrugoi/game_recommender.git
cd game_recommender
pip install -r requirements.txt
cd ..

python -m game_recommender.seed_data          # 10 игр и тестовый пользователь
uvicorn game_recommender.main:app --reload    # запуск сервера
```

Готово. Открывай **http://127.0.0.1:8000/docs**: там интерактивный Swagger, где всё можно потыкать мышкой.

> 💡 Модуль использует относительные импорты, поэтому запускай команды **из папки выше** проекта. Если у папки другое имя, подставь его вместо `game_recommender`.

## 🎯 Пример

После `seed_data.py` в базе есть пользователь **ilya**: любит RPG, Action, Fantasy и Open-World, играет на PC в регионе RU, готов потратить до 2500 ₽, до 100 часов и до 18+.

```bash
curl -X POST http://127.0.0.1:8000/recommend/1
```

Из 10 игр каталога фильтры отсеют три:

| Игра | Почему не попала |
|---|---|
| FIFA 24 | другая платформа (PS5) и регион (EU) |
| God of War Ragnarok | другая платформа (PS5) и регион (EU) |
| Minecraft | 200 часов, а лимит 100 |

Остальные игры (The Witcher 3, Baldur's Gate 3, Elden Ring, Cyberpunk 2077 и другие) упорядочатся по схожести с вкусами пользователя. Ответ выглядит так:

```json
{
  "user_id": 1,
  "username": "ilya",
  "recommendations_count": 7,
  "total_games_checked": 10,
  "rejected_games_count": 3,
  "response_time_seconds": 0.01,
  "algorithm": "TF-IDF + cosine similarity + context filtering",
  "recommendations": [
    {
      "game_id": 2,
      "title": "The Witcher 3",
      "genres": "RPG Fantasy Open-World Adventure",
      "platform": "PC",
      "price": 1499.0,
      "similarity": 0.8,
      "explanation": "..."
    }
  ]
}
```

*Числа `similarity` и `response_time_seconds` в примере условные: у тебя будут свои.*

## 🔌 API

| Метод | Путь | Что делает |
|---|---|---|
| `GET` | `/`, `/health` | Проверка, что сервис жив |
| `POST` | `/games` | Добавить игру в каталог |
| `GET` | `/games` | Весь каталог |
| `POST` | `/users` | Создать профиль пользователя |
| `GET` | `/users/{user_id}` | Получить профиль |
| `POST` | `/recommend/{user_id}` | 🎯 **Получить рекомендации** |
| `GET` | `/export/json/{user_id}` | Сохранить рекомендации в JSON |
| `GET` | `/export/csv/{user_id}` | Сохранить рекомендации в CSV |

### 👤 Профиль пользователя

| Поле | Что значит | Пример |
|---|---|---|
| `username` | Имя | `"ilya"` |
| `favorite_genres` | Любимые жанры через пробел | `"RPG Action Fantasy"` |
| `preferred_platform` | Платформа | `"PC"` |
| `max_price` | Потолок цены | `2500` |
| `max_playtime` | Максимум часов на игру | `100` |
| `max_age_rating` | Максимальный возрастной рейтинг | `18` |
| `region` | Регион | `"RU"` |

### 🕹 Карточка игры

`title`, `genres`, `platform`, `price`, `rating`, `playtime`, `age_rating`, `region`

## 🗂 Структура проекта

| Файл | За что отвечает |
|---|---|
| `main.py` | FastAPI-приложение и эндпоинты |
| `recommender.py` | TF-IDF, косинусная близость, fallback |
| `filters.py` | Фильтрация по контексту |
| `explanation.py` | Текст «почему рекомендуем» |
| `models.py` / `schemas.py` | ORM-модели и Pydantic-схемы |
| `database.py` / `repositories.py` | База данных и доступ к данным |
| `exceptions.py` / `logger_config.py` | Обработка ошибок и логи |
| `seed_data.py` | Тестовые данные |
| `test_*.py` | Тесты API, рекомендаций и нагрузки |

## 🧪 Тесты

```bash
pytest
```

## 📝 Логи

Всё важное пишется в `app.log`: кто что запросил, сколько игр проверено, сколько отсеяно и за сколько секунд собран ответ.

## 🛠 Стек

**Python** · **FastAPI** · **SQLAlchemy** · **Pydantic** · **scikit-learn** · **pandas** · **NumPy** · **pytest** · **httpx**

## 🗺 Что можно улучшить

- Применять фильтры (возраст, регион, платформа) и к fallback-выдаче
- Учитывать в ранжировании рейтинг и популярность игр, а не только текстовую схожесть
- Нормальная БД (PostgreSQL) вместо локальной
- Обратная связь «понравилось / не понравилось» для дообучения рекомендаций
- Docker-образ для запуска одной командой

---

<div align="center">

Сделано с ❤️ и косинусной близостью

</div>
