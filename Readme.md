Крч, сначала пройдемся по структуре проекта

```
.
├── auxiliaryfunc.py
├── bot.py
├── configStore.sh
├── db.sqlite
├── dbworker.py
├── helper
│   └── dbCreate.py
├── requirements.txt
└── yandexmap.py
```

| ФайлНейм | Описание |
| ------ | ------ |
| bot.py | тут вся логика тг бота |
| dbworker.py | Тут вся работа с бд |
| auxiliaryfunc.py | Тут доп функции для расчета лучшего дня и лучшей точки. Просто вынес доп логику в отдельный файл |
| yandexmap.py | Тут работа с api yandex map  |
| configStore.sh & helper/ db.Create.py | это вспомогательный скрипт для работы с бд sqlite3 при отладке |
| db.sqlite | сама бд |
| .env | енв файл с апи ключами |
--------------------------------------------
## .env 
##### Тут находятся апи ключи 
1) tgapikey - апи кей тг бота 
Что бы получить ключ создай тг бота тут(@BotFather в телеге)
2) yandexmapapikey - апи кей яндекс карт
сайтик: https://developer.tech.yandex.ru/services/
###### тыкаем step by step:
2.1 подключить api
2.2 javascript api и http геокодер
2.3 проходим анкетку и получаем апи ключ
```.sh
tgapikey='5336813001:AAGqdcg7hSO8zp6tIMBgHJviO493BxALtWc'
yandexmapapikey='3af8d255-1d29-4aba-bd93-cd169d90f494'
```

--------------------------------------------
### Теперь пойдем по файлам 

## 1) yandexmap.py
```.py
# Подгружаю  апи токен из .env
config = load_dotenv()
client = Client(os.getenv("yandexmapapikey"))
```

#### Адрес в координаты
```.py
# принмаю на вход адрес
def geo_decoder(address: str):
    try:
    #делаю запрос и получаю кортеж
        coordinates = client.coordinates(address)
        # распаршиваю его и возвращаю словарь с x и y кордами
        pos = {}
        pos['x'] = coordinates[1]
        pos['y'] = coordinates[0]
        return pos

    except Exception as e:
        return 'error'
```

##### Координаты по адресу
тут думаю даже объяснять ничего не надо
```.py
def get_address_by_coordinates(x: float, y: float):
    address = client.address(Decimal(f'{y}'), Decimal(f'{x}'))
    return address
```

## 2) auxiliaryfunc.py

##### Находим лучший день (Наилучший день тот, в которое общий вес мусора будет наибольший.)
```.py
# user_dict у нас примерно такого вида: {406149871: ['Понедельник', 'Среда', 'Пятница'], 1943316717: ['Понедельник', 'Среда', 'Суббота']}

def best_day(user_dict):
    days = dict()
    best_days = {
        'Понедельник': 0,
        'Вторник': 0,
        'Среда': 0,
        'Четверг': 0,
        'Пятница': 0,
        'Суббота': 0,
        'Воскресенье': 0,
    }
    
    #прохожусь по кортежу в первый раз и создаю в своем словарике нулевые значения веса мусора в соотвествии chat id из user_dict
    for key in user_dict:
        days[key] = {}
        for i in range(len(user_dict[key])):
            days[key][user_dict[key][i]] = 0
            
    # Прохожусь во второй раз и суммирую для каждого дня вес мусора
    for key in user_dict:
        for i in range(len(user_dict[key])):
            days[key][user_dict[key][i]] += dbworker.get_trash_weight(chat_id = key)
    
    # суммирую все веса для каждого отдельного дня
    for key in user_dict:
        for i in range(len(user_dict[key])):
            best_days[user_dict[key][i]] += days[key][user_dict[key][i]]
            
    # нахожу ключ(те нейминг дня недели) с максимальным весом мусора
    final_dict = dict([max(best_days.items(), key=lambda k_v: k_v[1])])
    # достаю ключ и преобразую его в строчку из словаря_ключей
    for key in final_dict:
        best_day = key
    
    return best_day

```

##### Это вспомогательная функция, где мы из кортежа соотвествий юзер чат айди/дни, которые он добавил возвращаем массив с чат айдишниками в вхождении лучшего дня
```.py
# user_dict у нас примерно такого вида: {406149871: ['Понедельник', 'Среда', 'Пятница'], 1943316717: ['Понедельник', 'Среда', 'Суббота']}

def chatids_with_best_day_match(user_dict, best_day: str):
    applied_chatids = []
    for key in user_dict:
        if best_day in user_dict[key]:
            applied_chatids.append(key)
    return applied_chatids
```

##### Это функция поиска лучшей точки (тут все по формуле из тз)
```.py
# applied_chatids вида: [406149871, 1943316717] (чат айишники телеги, у которых есть вхождение в лучший день)
def determine_the_best_position(applied_chatids):
    x_dividend = 0
    y_dividend = 0
    weight_divider = 0

    for i in range(len(applied_chatids)):
        output = dbworker.get_user_pos_and_trash_weight(chat_id = applied_chatids[i])
        x_dividend += output['x'] * output['trash_weight']
        y_dividend += output['y'] * output['trash_weight']
        weight_divider += output['trash_weight']

    x = x_dividend / weight_divider
    y = y_dividend / weight_divider

    return x, y
```

##### Функция поиска ближайшего адреса из юзеров, у которых есть вхождение в лучший день, к лучшей координате
тут я использую формулу sqrt((x1-x2)^2 + (y1-y2)^2) для определения расстояния между лучшей точкой и кордами юзера

получаю наименьшее расстояние и его чайт айдишник. делаю запрос к бд по чат айдишнику и она ретернит его адрес
```.py
def nearest_address(applied_chatids, x: float, y: float):
    best_distance = 100000000
    chat_id = 0
    for i in range(len(applied_chatids)):
        output = dbworker.get_user_pos_and_trash_weight(chat_id = applied_chatids[i])
        distance = sqrt( pow(x - output['x'],2) + pow(y - output['y'],2))

        if distance < best_distance:
            chat_id = applied_chatids[i]
            best_distance = distance

    nearest_address = dbworker.get_user_address(chat_id = chat_id)

    return nearest_address
```

## 3) dbworker.py

##### Инитим sql алхимию и установим ей конфиги (sqlAlchemy это ORM для работы с sql)
``` .py
basedir = os.path.abspath(os.path.dirname(__file__))
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + os.path.join(basedir, 'db.sqlite')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)
```

##### Тут мы инитим таблицу (думаю тут все понятно, название колонки = колонка(тип данных колонки))
```.py
class UserInfo(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    chat_id = db.Column(db.Integer())
    address = db.Column(db.String(1024))
    x = db.Column(db.Float())
    y = db.Column(db.Float())
    trash_weight = db.Column(db.Float())
```
    
###### В коде самих функций все пределбно понятно, так что пройдусь просто по неймингам и за что отвечают 
    
    * init_user - инитит юзера в бд
    * set_position - устанавливает корды по соответсвию chat_id для юзера
    * set_trash_weight - устанавливает вес мусора по соответсвию chat_id для юзера
    * get_user_pos_and_trash_weight - по чат айдишнику отдает данные об юзере (корды x y и вес мусора)
    * get_users - возвращает всех юзеров (ретернит список, но каждый объект это экземпляр таблицы UserInfo так что можно обращаться к каждому элементу по названию поля типо obj[i].chat_id)
    * get_users_chatids - отдает чайтайдишник всех юзеров
    * get_user_address - отдает адрес юзера по chat_id
    * get_trash_weight - отдает вес мусора юзера по chat_id

## 4) bot.py
Крч, тут много когда, но он как пазл и состоит из одних и тех же частей. Я тут просто перечислю основные моменты из фреймворка телебот и тогда логика должна стать понятной.

##### это декоратор телебота для обработки команд. возвращает экземпляр message который нужно пихнуть в арг функции
```.py
@bot.message_handler(commands=['start', 'menu'])
def menu(message):
```
##### Так создается вложенная клава
```.py
        markup = types.ReplyKeyboardMarkup(resize_keyboard=True, row_width=2)
        itembtn1 = types.KeyboardButton('/add_convenient_day')
        itembtn2 = types.KeyboardButton('/set_address')
        itembtn3 = types.KeyboardButton('/enter_trash_weight')
        itembtn4 = types.KeyboardButton('/profile')
        markup.add(itembtn1,itembtn2,itembtn3,itembtn4)
```

##### Так отправляется сообщение
```.py
bot.send_message(чат_айди, 'текст сообщения', reply_markup=markup(клава), parse_mode='MARKDOWN' (разметка, так же может быть html плейн текст и еще что-то)
```

##### Функция которая перекидывает на следющий логический блок после ответа юзера. Не сработает пока юзер не отправит сообщение(тк на вход в свои арги хавает сообщение от юзера и нейминг функции, на которую нужно перебросить)

```.py
bot.register_next_step_handler(msg, set_coordinates)
```

##### Inline кнопки создаются практически аналогично вложенной клаве, только к каждой кнопке нужно добавить call-back

```.py
markup_inline = types.InlineKeyboardMarkup(row_width=1)

        monday = types.InlineKeyboardButton(text = 'Понедельник', callback_data = 'Monday')
        tuesday = types.InlineKeyboardButton(text = 'Вторник', callback_data = 'Tuesday')
        wednesday = types.InlineKeyboardButton(text = 'Среда', callback_data = 'Wednesday')
        thursday = types.InlineKeyboardButton(text = 'Четверг', callback_data = 'Thursday')
        friday = types.InlineKeyboardButton(text = 'Пятница', callback_data = 'Friday')
        saturday = types.InlineKeyboardButton(text = 'Суббота', callback_data = 'Saturday')
        sunday = types.InlineKeyboardButton(text = 'Воскресенье', callback_data = 'Sunday')
        update = types.InlineKeyboardButton(text = 'Обновить 🔄', callback_data = 'update')
        
        markup_inline.add(monday, tuesday, wednesday, thursday, friday, saturday, sunday,update)
```

##### Обработчик кол-бэков от инлайн клавиатур 
Тут в отличии от месседж хэндлера вместо экземпляра message в него прилетает call и у него немного другие поля и методы(те если в месседж хэндлере что бы получить чат айдишник нужно делать message.chat.id, то тут call.from_user.id)
```.py
@bot.callback_query_handler(func = lambda call:True)
def callback_handler(call):
```

##### Функции изменений текста, клавиатуры и алерт сообщение
```.py
#Меняет текст сообщения по переданному message_id
bot.edit_message_text(chat_id=call.message.chat.id,message_id=call.message.message_id, text=msg)

#Меняет инлайн клаву по переданному message_id
bot.edit_message_reply_markup(chat_id=call.message.chat.id,message_id=call.message.message_id, reply_markup=markup_inline)

# Делает алерт сообщение при нажатии на inline кнопку
bot.answer_callback_query(call.id, text="updated!")
```