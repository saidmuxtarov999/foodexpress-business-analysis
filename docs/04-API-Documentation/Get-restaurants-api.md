## Nədir və Nə Üçün İstifadə Olunur?

Bu endpoint müştərinin tətbiqə daxil olduğu an qarşısına çıxan **əsas səhifəni (Feed/Kataloq) formalaşdıran mərkəzi sorğudur**. Sistemdə qeydiyyatdan keçmiş və anlıq olaraq sifariş qəbul etməyə hazır olan bütün restoranların siyahısını müəyyən edilmiş kriteriyalara (mətbəx növü, reytinq, çatdırılma məsafəsi) əsasən süzərək müştəriyə təqdim edir.

Müştəri ana səhifədə hər hansı bir filtr seçdikdə (Məsələn: *"Yalnız Fastfood və reytinqi 4.5-dən yuxarı olanlar"*) mobil tətbiq arxa fonda bu API-a müvafiq query parametrləri ilə müraciət edir.

## 2. Biznes Loqikası və Sistemə Nəyə Lazımdır?

- **Dinamik Kontent İdarəçiliyi (Geo-fencing):** Siyahı kor-koranə gətirilmir. Sistem arxa fonda müştərinin yerləşdiyi mövqeyi əsas götürərək, yalnız onun çatdırılma radiusunda (məsələn, max 5-7 KM məsafədə) yerləşən restoranları göstərir. Bu, uzaq məsafədən sifariş gəlməsinin və logistik kəsintilərin qarşısını alır.
- **Satışın Stimullaşdırılması (Conversion Rate Optimization):** Reytinq və populyarlıq filtrləri vasitəsilə müştərinin ən keyfiyyətli xidmət göstərən restoranları tez tapmasını təmin edir. Bu da müştəri məmnuniyyətini birbaşa artırır.
- **Əməliyyat Statusunun Yoxlanılması:** Bağlı olan, kuryer çatışmazlığı yaşayan və ya iş saatı bitmiş restoranlar dərhal siyahıdan kənarlaşdırılır (Stop-list məntiqi).

## 3. Texniki Arxitektura və Data Axını

### Daxili Proses (Step-by-Step):

1. **Authorization Verification (Token Yoxlanışı):** API Gateway gələn sorğunun header hissəsindəki JWT tokeni yoxlayır. Əgər `ROLE_CUSTOMER` səlahiyyəti düzgündürsə, sorğu kataloq xidmətinə (Catalog Service) ötürülür.
2. **Database Query (SQL Səviyyəsi):** Servis mərkəzi verilənlər bazasındakı `itba.restaurants` cədvəlinə dinamik `SELECT` sorğusu göndərir.
    - *Daxili Məntiq:*SQL
        
        ```
        SELECT id, name, rating, estimated_delivery_time, min_order, delivery_fee
        FROM itba.restaurants
        WHERE is_active = true
          AND cuisine_type = :cuisine
          AND rating >= :minRating;
        ```
        
3. **Caching (Sürətləndirmə):** Restoran kataloqu çox sıx çağırılan endpoint olduğu üçün, verilən cavablar qısa müddətli (məsələn, 30 saniyəlik) keş yaddaşında saxlanılır ki, hər saniyə bazaya birbaşa sorğu gedib performansı aşağı salmasın.

## 4. Query Parameters (Sorğu Parametrləri)

Göndərdiyin Notion/Database şablonunun strukturuna uyğun olaraq parametrlər aşağıdakı şəkildə ötürülür:

| **Parametr** | **Mütləqdir?** | **Data Tipi** | **Təsvir / Seçimlər** |
| --- | --- | --- | --- |
| **`lang`** | bəli | String | Kontentin qaytarılacağı dil versiyası (`az`, `eng`, `rus`). |
| **`cuisine`** | xeyr | String | Mətbəx növünə görə filtr (`fastfood`, `milli`, `doner`, `sushi`). |
| **`minRating`** | xeyr | Float | Minimum reytinq limiti (`1.0` - `5.0` arası dəyər). |

## 5. Responses (Cavab Strukturları)

### Uğurlu Senari (200 OK)

Filtrlərə uyğun restoranlar tapıldıqda sistem bu strukturu geri qaytarır:

JSON

```json
{
  "statusCode": 1,
  "statusMessage": "Success",
  "restaurants": [
    {
      "id": 102,
      "name": "Shaurma N1",
      "rating": 4.7,
      "estimatedDeliveryTime": "25-35 min",
      "minOrder": 5.00,
      "dynamicDeliveryFee": 1.50
    },
    {
      "id": 105,
      "name": "Mado",
      "rating": 4.5,
      "estimatedDeliveryTime": "30-40 min",
      "minOrder": 10.00,
      "dynamicDeliveryFee": 2.00
    }
  ]
}
```

### Xəta Senarisi (400 Bad Request)

Əgər mütləq olan `lang` parametri göndərilməyibsə və ya filtr dəyərləri düzgün tipdə deyiləsə:

JSON

```json
{
  "statusCode": 0,
  "statusMessage": "InvalidRequest",
  "errors": [
    {
      "code": "MISSING_PARAMETER",
      "message": "The 'lang' parameter is required."
    }
  ]
}
```

## 6. Expected Behavior (Frontend üçün gözlənilən loqika)

- `statusCode = 1` olduqda, gələn `restaurants` massivi (array) ana səhifədəki "Restoranlar" bölməsində kart dizaynı formasında render edilməlidir.
- Hər bir kartın üzərində `name`, `rating`, `estimatedDeliveryTime` və `dynamicDeliveryFee` vizual olaraq göstərilir.
- İstifadəçi siyahıdakı hər hansı bir restoran kartına kliklədikdə, tətbiq həmin restoranın daxili `id` dəyərini götürərək növbəti (4-cü) API-a (Menyu Çağırılması) keçid etməlidir.

## 7. Code Example (Axios ilə Sorğu Nümunəsi)

JavaScript

```jsx
import axios from 'axios';

const options = {
  method: 'GET',
  url: 'https://api.foodexpress.az/api/v1/restaurants',
  params: {
    lang: 'az',            // Mütləq parametr
    cuisine: 'fastfood',   // Könüllü filtr
    minRating: '4.5'       // Könüllü filtr
  },
  headers: {
    'accept': 'application/json',
    'Authorization': 'Bearer eyJhbGciOiJIUzI1Ni...' // 2-ci API-dən aldığımız token
  }
};

axios.request(options)
  .then(function (response){
    // Siyahını frontend state-inə set edirik
    console.log('Restoranlar uğurla gətirildi:', response.data.restaurants);
  })
  .catch(function (error){
    console.error('Kataloq yüklənərkən xəta baş verdi:', error.response ? error.response.data : error.message);
  });
```
