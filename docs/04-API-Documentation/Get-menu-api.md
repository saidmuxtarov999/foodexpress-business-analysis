## 1. Nədir və Nə Üçün İstifadə Olunur?

Bu endpoint müştəri ana səhifədə hər hansı bir restoranın üzərinə kliklədikdə **həmin obyektin daxili səhifəsini və satılan bütün təamları formalaşdıran mərkəzi sorğudur**. Seçilmiş restoranın strukturunu, aktiv menyu kateqoriyalarını (məs: *Dönərlər, İçkilər, Şirniyyatlar*), yeməklərin adlarını, qiymətlərini və şəkillərini strukturlaşdırılmış şəkildə müştərinin ekranına gətirir.

Müştəri restoranın profili ekranına keçid etdiyi an mobil tətbiq bu API-a restoranın unikal ID-sini (`id`) ötürərək müraciət edir.

## 2. Biznes Loqikası və Sistemə Nəyə Lazımdır?

- **Anlıq Stok Kontrolu (Stop-List / Availability):** Restoranda həmin an tükənmiş olan bir yeməyin müştəri tərəfindən səhvən sifariş edilməsinin qarşısını alır. Əgər məhsulun bazadakı statusu `isAvailable = false` olarsa, frontend həmin məhsulun üzərinə klikləməyi bloklayır. Bu, restoranla müştəri arasında yaranacaq narazılıqları önləyir.
- **Kateqoriyalı Struktur (UX/UI):** Müştərinin xaotik şəkildə deyil, rahat şəkildə kateqoriyalara bölünmüş menyu daxilində naviqasiya etməsinə kömək edir. Bu da müştərinin səbətə mal əlavə etmə sürətini və həvəsini artırır.
- **Dinamik Qiymətləndirmə:** Kampaniya, endirim və ya standart qiymət dəyişiklikləri birbaşa bazada yeniləndiyi üçün müştəri hər an ən son və qüvvədə olan qiyməti görür.

## 3. Texniki Arxitektura və Data Axını

### Daxili Proses (Step-by-Step):

1. **URL Parsing:** API Gateway gələn URL daxilindəki `{id}` parametrini oxuyur (məs: `102`) və daxili sorğunu Menyu Servisinə (Menu Service) yönləndirir.
2. **Database Join Əməliyyatı:** Servis daxildə `itba.menu_items` və `itba.menu_categories` cədvəllərini bir-birinə bağlayaraq relasiyalı `SELECT` sorğusu icra edir.
    - *Daxili Məntiq:*SQL
        
        ```
        SELECT c.category_name, i.id, i.title, i.price, i.image_url, i.is_available
        FROM itba.menu_items i
        JOIN itba.menu_categories c ON i.category_id = c.id
        WHERE i.restaurant_id = :restaurantId;
        ```
        
3. **Data Localization:** Göndərilən `lang` parametrinə uyğun olaraq (az, eng, rus) yeməklərin adları və təsvirləri müvafiq dildə JSON strukturuna çevrilir.

## 4. Parameters (Parametrlər)

### Path Parameters (URL daxilində)

| **Parametr** | **Mütləqdir?** | **Data Tipi** | **Təsvir** |
| --- | --- | --- | --- |
| **`id`** | bəli | Integer | Menyusu çağırılacaq restoranın verilənlər bazasındakı unikal ID-si. |

### Query Parameters (Sorğu sətirində)

| **Parametr** | **Mütləqdir?** | **Data Tipi** | **Təsvir / Seçimlər** |
| --- | --- | --- | --- |
| **`lang`** | bəli | String | Kontentin hansı dildə gələcəyini təyin edir (`az`, `eng`, `rus`). |

## 5. Responses (Cavab Strukturları)

### Uğurlu Senari (200 OK)

Restoran və menyu məlumatları bazada tapıldıqda sistem bu iyerarxik strukturu qaytarır:

JSON

```json
{
  "statusCode": 1,
  "statusMessage": "Success",
  "menu": [
    {
      "categoryName": "Dönərlər",
      "items": [
        {
          "itemId": 501,
          "title": "Ət Dönəri Çörəkdə",
          "price": 4.50,
          "imageUrl": "https://cdn.foodexpress.az/doner.png",
          "isAvailable": true
        },
        {
          "itemId": 502,
          "title": "Toyuq Dönəri Lavaşda",
          "price": 4.00,
          "imageUrl": "https://cdn.foodexpress.az/lavas.png",
          "isAvailable": false
        }
      ]
    },
    {
      "categoryName": "İçkilər",
      "items": [
        {
          "itemId": 601,
          "title": "Coca-Cola 330ml",
          "price": 1.50,
          "imageUrl": "https://cdn.foodexpress.az/cola.png",
          "isAvailable": true
        }
      ]
    }
  ]
}
```

### Xəta Senarisi (400 Bad Request)

Əgər mütləq olan `lang` parametri unudulubsa və ya mövcud olmayan bir restoran ID-si yazılıbsa:

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

- `statusCode = 1` olduqda, gələn `menu` massivi daxilindəki hər bir obyekt üçün ekranda bir bölmə (Section) açılır. Bölmənin başlığı `categoryName` olur.
- Kateqoriyanın daxilindəki `items` massivi dövrə (loop) salınaraq yemək kartları render edilir.
- Əgər hər hansı bir məhsulda `isAvailable = false` gələrsə, frontend həmin kartı tutqun (greyed out) rəngə boyamalı, üzərinə "Tükənib" mətni yazmalı və səbətə əlavə etmək üçün istifadə olunan "+" düyməsini qeyri-aktiv etməlidir.
- Müştəri aktiv məhsulun üzərindəki "+" düyməsinə basdıqda, tətbiq `itemId` dəyərini götürərək **5-ci və 6-cı API-lara (Səbət əməliyyatları)** yönlənməlidir.

## 7. Code Example (Axios ilə Sorğu Nümunəsi)

JavaScript

```jsx
import axios from 'axios';

const options = {
  method: 'GET',
  url: 'https://api.foodexpress.az/api/v1/restaurants/102/menu', // Restoran ID: 102
  params: {
    lang: 'az' // Mütləq parametr
  },
  headers: {
    'accept': 'application/json',
    'Authorization': 'Bearer eyJhbGciOiJIUzI1Ni...'
  }
};

axios.request(options)
  .then(function (response){
    // Menyunu frontend state-inə ötürüb ekranda göstəririk
    console.log('Menyu strukturu yükləndi:', response.data.menu);
  })
  .catch(function (error){
    console.error('Menyu gətirilərkən xəta baş verdi:', error.response ? error.response.data : error.message);
  });
```
