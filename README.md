# emby chat


## Общие параметры для запросов:

* (int) client_id - Идентификатор клиента
* (string) client_secret - Секретный ключ клиента для подписи запроса 
* (string) api_token - Токен запроса для валидации
* (hash) user - данные текущего пользователя
    * (string | number) id - идентификатор пользователя
    * (string) name - имя пользователя
    * (bool) is_bot - флаг, указывающий, что пользователь является ботом
* (array of hash) recipients данные остальных пользователей чата
    * (string | number) id - идентификатор пользователя
    * (string) name - имя пользователя
    * (bool) is_bot - флаг, указывающий, что пользователь является ботом
* (string) signature - подпись запроса, генерируется на основе строгой последовательности параметров client_id, client_secret, random number, user.id, user.name, recipient.id, recipient.name
* (array) extra - дополнительные параметры сообщения, которые могут пригодиться для бизнес логики чата. К примеру есть параметр isService, если он проставлен в true, такие сообщения отображаются серым цветом
* (array) buttons - кнопки в сообщении, при нажатии на которые можно активировать отсылать эвенты
    * (string.required) label - надпись на кнопке
    * (string) style: positive|negative|neutral - стиль в котором отображается кнопка
    * (string) type: local|remote - local кнопки шлют sendMessage в родительский window c параметром action, remote кнопки шлют сообщения на удаленный сервер посреством webhook
    * (string | array | object) action - параметр которые пересылается при нажатии на кнопку

В POST и PUT запросах параметры нужно отправлять в виде JSON в теле запроса

## Генерация ссылок

GET /?{request_uri}

```php
function generateEmbyUrl($chatId, User $user, $recipients, $extra = [])
{
    // получаем из конфигов идентифкатор клиента и секретный чат
    $clientId = Config::get('emby.client_id');
    $clientSecret = Config::get('emby.client_id');
    // базовая часть урла API (http://emby.markuper.com/)
    $baseUrl = Config::get('emby.base_url');


    // генерируем рандом для запроса
    $rnd = str_random(32);

    // в переменной $signatureParams хранится массив параметров на основе которых будет сгенерирована строка подписи
    // порядок важен параметров важен!
    $signatureParams = [
        $clientId,
        $clientSecret,
        $rnd,
        $user->id,
        $user->name
    ];

    // в переменной $queryParams хранится массив параметров запроса
    $queryParams = [
        'client_id' => $clientId, // идентификатор клиента
        'chat_id' => $chatId, // идентификатор чата
        'rnd' => $rnd, // рандомный параметр, для того что бы запрос всегда был с разной подписью
        'user' => [ // данные текущего пользователя, от имени которого будем писать в чат
            'id' => $user->id,
            'name' => $user->name
        ],
        'recipients' => [] // массив участников чата, не содержит данные текузего пользователя
    ];

    // проходим по массиву участников чата. Участники чата все члены чата, кроме того человека от чьего имени загружаем чат
    foreach ($recipients as $recipient)
    {
        // кладем в массив параметров подписи сначала айди участника
        $signatureParams[] = $recipient->id;
        // затем имя участника, порядок важен!
        $signatureParams[] = $recipient->name;

        // кладем в массив параметров запроса участника
        $queryParams['recipients'][] = [
            'id' => $recipient->id,
            'name' => $recipient->name
        ];
    }

    //последним параметром подписи является айди чата, у этого параметра ключ числовой
    $signatureParams[] = $chatId;

    // кладем в массив параметров запроса подпись на основе md5 хэш функции
    $queryParams['signature'] = md5(implode(',', $signatureParams));

    // кладем в массив параметров запроса дополнительные данные.
    // дополнительные данные не используются при генерации подписи!
    foreach ($extra as $key => $value)
    {
        $queryParams[$key] = $value;
    }

    // из массив параметров запроса получается uri строку запроса (request uri)
    $query = http_build_query($queryParams, null, '&');

    // вовзращаем строку с абсолюиным адресом запроса
    return $baseUrl . '?' . $query;
}
```

## Отправка сообщения

POST /api/v1/messages

```php
function sendMessage($chatId, User $user, $recipients, $message, array $extra = [], array $buttons = [])
{
    // получаем из конфигов токен апи и базовый урл эмби
    $apiToken = Config::get('emby.api_token');
    // базовая часть урла API (http://emby.markuper.com/)
    $baseUrl = Config::get('emby.base_url');

    // переменная $queryParams содержит параметры POST запроса
    $queryParams = [
        'api_token' => $apiToken,
        'user' => [
            'id' => $user->id,
            'name' => $user->name,
            'is_bot' => true
        ],
        'chat_id' => $chatId,
        'messages' => [
            [
               'text' => $message
            ]
        ],
        'recipients' => []
    ];

    // если переданные экстра параметры сообщения
    if (count($extra))
    {
        $queryParams['message']['extra'] = $extra;
    }

    // если переданы кнопки для сообщения
    if (count($buttons))
    {
        $queryParams['message']['buttons'] = $buttons;
    }

    foreach ($recipients as $recipient)
    {
        $queryParams['recipients'][] = [
            'id' => $recipient->id,
            'name' => $recipient->name
        ];
    }

    $client = new GuzzleClient();
    // $baseUrl = http://emby.markuper.com
    $url = $baseUrl . 'api/v1/messages';

    // отправляем POST запрос с параметрами
    $client->post($url, [
        'json' => $queryParams
    ]);
}
```

## Редактирование сообщения

PUT api/v1/messages/{message_id}

```php
/**
 * @param string $messageId
 * @param array $data Обновляемые поля сообщения  { text, extra, buttons, is_deleted }
 */
public function updateMessage($messageId, array $data)
{
    // получаем из конфигов токен апи и базовый урл эмби
    $apiToken = Config::get('emby.api_token');
    $baseUrl = Config::get('emby.base_url');

    // переменная $queryParams содержит параметры POST запроса
    $queryParams = [
        'api_token' => $apiToken,
        'message' => $data
    ]

    $client = new GuzzleClient();
    $url = $baseUrl . 'api/v1/message/{$messageId}';

    // отправляем PUT запрос с параметрами
    $client->put($url, [
        'json' => $queryParams
    ]);
}
```

## Webhook

В POST-запросе приходят данные в виде JSON в теле запроса:

* event: message_button|new_message
* params: содержит набор параметров, общие параметры для всех типов такие
    * client_id
    * chat_id
    * user_id
    * message_id

**Типы событий (event_name):**

* message_button - пользователь нажал на кнопку, в params дополнительно приходит поле
    * (object) button_action - параметры кнопки

* new_message - новое сообщение, в params дополнительно приходит поле
    * (string) message_text
    * (string) message_date
