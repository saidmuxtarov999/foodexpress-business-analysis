# 1. Nədir və Nə Üçün İstifadə Olunur?

Bu endpoint loqistika mühərrikinin (**Dispatcher Engine**) restorana ən yaxın radiusda təyin etdiyi kuryerə göndərdiyi sifariş təklifini **kuryerin rəsmən qəbul etməsini təmin edən təsdiq qapısıdır**. Kuryer tətbiqinə yeni bir sifariş düşdükdə, ekranda "Qəbul et" düyməsi çıxır və kuryer həmin düyməyə basdıqda bu API işə düşür.

Sifarişin sahibsiz statusdan çıxıb, fiziki bir kuryerə bağlanması (assignment) tam olaraq bu endpoint-in çağırılması ilə reallaşır.

## 2. Biznes Loqikası və Sistemə Nəyə Lazımdır?

- **Ciddi 60 Saniyəlik Loop Kontrolu:** Sistem kuryerlərin operativliyini qorumaq üçün hər bir təklifə tam **60 saniyə** cavab müddəti tanıyır. Bu endpoint həmin 60 saniyə tamamlanmadan kuryer tərəfindən çağırılmalıdır. Əgər kuryer düyməni vaxtında sıxsa, sifarişi rəsmən öz üzərinə götürür və digər kuryerlərin ekranından bu təklif yox olur.
- **Logistik Kilidləmə (Order Locking):** Bir sifarişin eyni anda səhvən iki fərqli kuryerə təhvil verilməsinin qarşısını mütləq şəkildə alır. İlk klikləyən və tranzaksiyanı uğurla tamamlayan kuryer sifarişin rəsmi daşıyıcısı təyin edilir.
- **Çatdırılma SLA-nin Başlaması:** Kuryer bu təklifi qəbul etdiyi andan etibarən onun restorana tərəf hərəkət müddəti (Pick-up SLA) hesablanmağa başlayır. Bu da logistik səmərəliliyi maksimallaşdırır.

## 3. Texniki Arxitektura və Data Axını

### Daxili Proses (Step-by-Step):

1. **Concurrency and Availability Check:** `Logistics Service` mərkəzi yaddaşda (In-memory/Redis) həmin sifarişin hələ də boşda (boş statusda) olub-olmadığını və 60 saniyəlik təklif taymautunun bitib-bitmədiyini milisaniyələr daxilində yoxlayır.
2. **Database Update (Assignment):** Əgər sifariş hələ də aktivdirsə, mərkəzi `itba.orders` cədvəlində müvafiq sifariş sətirində kuryer ID-si qeydiyyata alınır və sifarişin statusu növbəti logistik mərhələyə keçirilir:SQL
    
    ```
    UPDATE itba.orders
    SET courier_id = :courierId, status = 'ACCEPTED_BY_COURIER', updated_at = NOW()
    WHERE id = :orderId AND courier_id IS NULL;
    ```
    
3. **State Evacuation:** Kuryerlər üçün açılmış aktiv dispetçer axtarış dövrü (search loop) və digər namizədlərə göndərilmiş bildirişlər dərhal ləğv edilir (purge).
4. **Customer Notification:** Sistem müştəri tərəfə WebSocket kanalı üzərindən dərhal paket ötürür: *"Kuryer tapıldı! Sifarişiniz [Kuryer Adı] tərəfindən restorandan təslim alınmağa doğru gedir"*.

## 4. Request Body Parameters

Göndərdiyin sənədləşdirmə strukturuna əsasən, dispetçer qəbul body-si aşağıdakı mütləq parametrləri ehtiva edir:

| **Parametr** | **Mütləqdir?** | **Data Tipi** | **Təsvir** |
| --- | --- | --- | --- |
| **`courierId`** | bəli | String | Təklifi qəbul edən kuryerin sistemdəki unikal identifikatoru. |
| **`orderId`** | bəli | String | Kuryerin üzərinə götürdüyü aktiv sifarişin ID-si. |

## 5. Responses (Cavab Strukturları)

### Uğurlu Senari (200 OK)

Kuryer təklifi vaxtında tutduqda və sistem rəsmi bağlamanı etdikdə bu cavab strukturlaşdırılır:

JSON

```json
{
  "statusCode": 1,
  "statusMessage": "CourierAssignedSuccessfully"
}
```

### Xəta Senarisi (400 Bad Request - Taymaut və ya Başqası Tutdu Xətası)

Əgər kuryer 60 saniyədən gec klikləyibsə və ya o klikləyən ana qədər sifariş avtomatik olaraq digər kuryerə ötürülübsə:

JSON

```json
{
  "statusCode": 0,
  "statusMessage": "InvalidRequest",
  "errors": [
    {
      "code": "OFFER_EXPIRED_OR_TAKEN",
      "message": "This order offer has expired or has been accepted by another courier."
    }
  ]
}
```

## 6. Expected Behavior (Frontend üçün gözlənilən loqika)

- `statusCode = 1` gəldikdə, kuryerin tətbiqindəki təklif ekranı (Pop-up/Modal) bağlanır və onun əvəzinə dərhal "Aktiv Sifariş Naviqasiyası" ekranı açılır. Bu ekranda kuryerə restoranın ünvanı, xəritə marşrutu və yeməyi götürməli olduğu daxili pin göstərilir.
- Əgər xəta kodu `OFFER_EXPIRED_OR_TAKEN` gələrsə, tətbiq ekrandakı təklif kartını dərhal silir, kuryerə səsli xəbərdarlıq verir və onu yenidən boş statusda yeni sifarişlər gözləmə rejimine (Online/Idle dashboard) qaytarır.

## 7. Code Example (Axios ilə Sorğu Nümunəsi)

JavaScript

```jsx
import axios from 'axios';

const options = {
  method: 'POST',
  url: 'https://api.foodexpress.az/api/v1/couriers/dispatch/accept',
  headers: {
    'content-type': 'application/json',
    'Authorization': 'Bearer eyJhbGciOiJIUzI1Ni...' // Kuryerin mərkəzi JWT tokeni
  },
  data: {
    courierId: 'cur-7721', // Təklifi qəbul edən kuryer
    orderId: 'ord-99102'   // Sifariş ID
  }
};

axios.request(options)
  .then(function (response){
    // Sifariş rəsən kuryerə bağlandı, naviqasiyanı açırıq
    console.log('Dispetçer təsdiqləndi:', response.data.statusMessage);
  })
  .catch(function (error){
    console.error('Təklif qəbul edilərkən xəta:', error.response ? error.response.data : error.message);
  });
```
