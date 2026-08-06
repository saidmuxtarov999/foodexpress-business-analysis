## Nədir və Nə Üçün İstifadə Olunur?

Bu endpoint Food Express platformasına daxil olmaq və ya qeydiyyatdan keçmək istəyən hər bir istifadəçinin (Müştəri və ya Kuryer) sistemdə atdığı **ilk addımdır**. Platformada ənənəvi şifrə mexanizmindən istifadə olunmadığı üçün, təhlükəsizlik və istifadəçi rahatlığı mərkəzi **OTP (One-Time Password)** sistemi üzərində qurulub.

İstifadəçi tətbiqi açır, mobil nömrəsini daxil edir və "Davam et" düyməsini sıxdıqda arxa fonda bu endpoint çağırılır. Endpoint-in əsas vəzifəsi daxil edilən nömrənin strukturunu yoxlamaq, unikal sessiya ID-si (`verificationId`) yaratmaq və xarici SMS Gateway provayderinə sorğu ataraq istifadəçinin telefonuna 4 rəqəmli şifrə göndərməkdir.

## 2. Biznes Loqikası və Sistemə Nəyə Lazımdır?

- **Təhlükəsizlik və Bot Qorunması:** Sistemə saxta (fake) nömrələrlə kütləvi qeydiyyatın (Spam/DDOS) qarşısını alır. Hər bir sorğu real mobil şəbəkə tərəfindən doğrulanır.
- **İstifadəçi Təcrübəsi (UX):** İstifadəçini uzun qeydiyyat formaları (Ad, soyad, şifrə, şifrə təkrarı) doldurmaq məcburiyyətindən azad edir. Sistem daxilində qeydiyyat müddətini 10 saniyəyə endirir.
- **Maliyyə Kontrolu (Rate Limiting):** SMS göndərişləri şirkət üçün birbaşa xərc (maliyyə) deməkdir. Bu API arxasında dayanan loqika eyni nömrəyə 1 dəqiqə ərzində ardıcıl SMS göndərilməsini bloklayır, bununla da şirkətin büdcəsini qoruyur.

## 3. Texniki Arxitektura və Data Axını

### Daxili Proses (Step-by-Step):

1. **Validation (Yoxlama):** API Gateway sorğunu qarşılayır və `phoneNumber` dəyərinin Azərbaycanın E.164 beynəlxalq standartına (`+994XXXXXXXXX`) uyğun olub-olmadığını Regex vasitəsilə yoxlayır.
2. **Sessiya Yaradılması:** `Identity Service` yaddaşda (In-memory cache) müvəqqəti olaraq 180 saniyəlik (3 dəqiqə) bir sətir açır. Bura təsadüfi yaradılmış `verificationId` (məs: `v-77a8-4cde-bc92`) və generatsiya olunmuş 4 rəqəmli kod (məs: `4815`) yazılır.
3. **Integration (İnteqrasiya):** Servis daxildə asinxron olaraq yerli SMS provayderin (məs: Layan, Azercell GW və s.) API-ına HTTP POST sorğusu göndərir: *"Bu nömrəyə bu mətni göndər: Food Express doğrulama kodu: 4815"*.
4. **Response (Cavab):** Xarici provayderdən "Uğurlu" cavabı gəldikdə, backend müştəri tətbiqinə təhlükəsizlik səbəbindən **heç bir halda real OTP kodu göndərmədən**, sadəcə `verificationId` və taymaut müddətini geri qaytarır.

## 4. Query Parameters

*Bu endpoint üçün query parametri mövcud deyil, məlumat təhlükəsizlik və struktur bütövlüyü baxımından birbaşa HTTP Request Body (application/json) daxilində ötürülür.*

## 5. Responses (Cavab Strukturları)

### Uğurlu Senari (200 OK)

Sistem nömrəni qəbul etdikdə, SMS-i yola saldıqda və keşdə sessiyanı uğurla başlatdıqda bu cavab dönür:

JSON

```json
{
  "statusCode": 1,
  "statusMessage": "Success",
  "data": {
    "verificationId": "v-77a8-4cde-bc92",
    "expiresInSeconds": 180
  }
}
```

### Xəta Senarisi (400 Bad Request)

Əgər istifadəçi nömrəni səhv formatda daxil edibsə (məsələn, rəqəm əvəzinə hərf yazıb və ya beynəlxalq kodu unudubsa):

JSON

```json
{
  "statusCode": 0,
  "statusMessage": "InvalidRequest",
  "errors": [
    {
      "code": "INVALID_PHONE_FORMAT",
      "message": "The 'phoneNumber' field must match Azerbaijan E.164 international standard."
    }
  ]
}
```

## 6. Expected Behavior (Frontend üçün gözlənilən loqika)

- `statusCode = 1` gəldiyi andan etibarən mobil tətbiq dərhal nömrə daxiletmə ekranını gizlətməli və 4 xanalı OTP giriş ekranını aktivləşdirməlidir.
- Gələn `verificationId` dəyəri tətbiqin lokal yaddaşında (state) saxlanılmalıdır, çünki növbəti (2-ci) API-a kodu göndərərkən bu ID mütləq bizə lazım olacaq.
- `expiresInSeconds` (180 saniyə) dəyərinə uyğun olaraq ekranda geriyə sayım (Countdown Timer) vizuallaşdırılmalıdır. Taymer sıfıra çatana qədər istifadəçiyə ikinci dəfə "SMS-i yenidən göndər" düyməsinə sıxmaq icazəsi verilməməlidir.

## 7. Code Example (Axios ilə Sorğu Nümunəsi)

JavaScript

```jsx
import axios from 'axios';

// Sorğunun konfiqurasiya parametrləri
const options = {
  method: 'POST',
  url: 'https://api.foodexpress.az/api/v1/auth/register-initiate',
  headers: {
    'accept': 'application/json',
    'content-type': 'application/json'
  },
  data: {
    phoneNumber: '+994501234567' // Müştərinin tətbiqdə yazdığı nömrə
  }
};

// Sorğunun icra edilməsi
axios.request(options)
  .then(function (response){
    // Uğurlu cavab gəldikdə bura işləyir
    console.log('Sessiya başladıldı:', response.data);
  })
  .catch(function (error){
    // Sistem və ya şəbəkə xətası baş verdikdə bura işləyir
    console.error('Qeydiyyat başlanğıcında xəta:', error.response ? error.response.data : error.message);
  });
```
