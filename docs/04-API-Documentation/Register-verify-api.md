## 1. Nədir və Nə Üçün İstifadə Olunur?

Bu endpoint istifadəçinin (Müştəri və ya Kuryer) telefonuna gələn 4 rəqəmli OTP doğrulama kodunu daxil etdikdən sonra icra olunan **təhlükəsizlik və sessiya rəsmiləşdirilməsi qapısıdır**. Birinci API tərəfindən başladılan qeydiyyat/giriş sessiyasını yekunlaşdırır.

Müştəri kodu yazıb təsdiqlədiyi and tətbiq bu API-a sorğu göndərir. Əgər kod düzgündürsə, sistem istifadəçinin şəxsiyyətini təsdiqləyir, yeni istifadəçidirsə verilənlər bazasında qeydiyyatını tamamlayır və tətbiqin sonrakı bütün sorğularda istifadə edəcəyi kriptoqrafik mərkəzi giriş icazələrini (**JWT Access və Refresh Token**) generasiya edir.

## 2. Biznes Loqikası və Sistemə Nəyə Lazımdır?

- **Şəxsiyyətin Vahid Doğrulanması (Authentication):** Şifrə unudulması ssenarilərini sıfıra endirir. İstifadəçinin mobil nömrəyə fiziki olaraq sahib olduğunu qəti şəkildə sübut edir.
- **Rol Bölünməsi (Role-Based Access Control):** Doğrulamadan keçən istifadəçinin sistemdəki statusunu müəyyən edir. Eyni interfeysdən gələn sorğunun müştəriyə (`ROLE_CUSTOMER`), restorana (`ROLE_RESTAURANT`) və ya kuryerə (`ROLE_COURIER`) aid olduğunu təyin edərək sistem daxilindəki səlahiyyətləri ayırır.
- **Sessiya Təhlükəsizliyi:** Generasiya olunan tokenlər vasitəsilə istifadəçinin hər dəfə nömrə yazmaq məcburiyyəti aradan qalxır, eyni zamanda kənar şəxslərin istifadəçi məlumatlarını ələ keçirməsinin qarşısını mərkəzi asinxron şifrələmə ilə alır.

## 3. Texniki Arxitektura və Data Axını

### Daxili Proses (Step-by-Step):

1. **İn-Memory Yoxlanış:** `Identity Service` gələn `verificationId` dəyərini keş yaddaşından (Redis/In-memory) axtarır. Əgər taymaut (180 saniyə) bitibsə və ya belə bir ID yoxdursa, dərhal xəta qaytarır.
2. **Kodun Müqayisəsi:** Keşdə saxlanılan real kodla istifadəçinin göndərdiyi `otpCode` riyazi olaraq qarşılaşdırılır. Səhv daxiletmə sayı (məsələn, max 3 cəhd) aşılarsa, sessiya bloklanır.
3. **Database İnteqrasiyası:** Kod düzgündürsə, sistem mərkəzi relyasion bazanın `itba.users` cədvəlinə sorğu atır. Əgər bu nömrə bazada yoxdursa, yeni sətir açaraq müştərini qeydiyyata alır (`INSERT`). Əgər nömrə artıq mövcuddursa, profil məlumatlarını oxuyur (`SELECT`).
4. **Token Generasiyası:** İstifadəçinin ID-si və rolu JWT strukturunun içərisinə (payload) qoyulur, sistemin gizli açarı (Secret Key) ilə imzalanır. 1 saatlıq `accessToken` və uzunmüddətli `refreshToken` yaradılaraq tətbiqə təhvil verilir. Keşdəki OTP sessiyası isə təhlükəsizlik üçün dərhal silinir (purge).

## 4. Request Body Parameters

| **Parametr** | **Mütləqdir?** | **Data Tipi** | **Təsvir** |
| --- | --- | --- | --- |
| **`verificationId`** | bəli | String | 1-ci API-nin cavabından alınan unikal sessiya ID-si. |
| **`otpCode`** | bəli | String | İstifadəçinin telefonuna gələn 4 rəqəmli doğrulama şifrəsi. |

## 5. Responses (Cavab Strukturları)

### Uğurlu Senari (200 OK)

Kod tamamilə düzgün olduqda və sistem sessiyanı rəsmiləşdirdikdə bu cavab strukturlaşdırılır:

JSON

```json
{
  "statusCode": 1,
  "statusMessage": "Success",
  "auth": {
    "tokenType": "Bearer",
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkZvb2REZXYiLCJyb2xlIjoiUk9MRV9DVVNUT01FUiJ9...",
    "refreshToken": "rX9_m2_kL0_pQ1_mN3bV",
    "expiresIn": 3600,
    "role": "ROLE_CUSTOMER"
  }
}
```

### Xəta Senarisi (401 Unauthorized)

Kod səhv yazıldıqda və ya 3 dəqiqəlik gözləmə müddəti keçdikdə (expired) yaranan biznes xətası:

JSON

```json
{
  "statusCode": 0,
  "statusMessage": "AuthenticationFailed",
  "errors": [
    {
      "code": "EXPIRED_OR_INVALID_OTP",
      "message": "The OTP code is incorrect or session has been expired."
    }
  ]
}
```

## 6. Expected Behavior (Frontend üçün gözlənilən loqika)

- `statusCode = 1` gəldikdə frontend daxil olan `accessToken` və `refreshToken` dəyərlərini tətbiqin təhlükəsiz lokal yaddaşında (Secure Storage / Encrypted SharedPreferences) saxlamalıdır.
- Sonrakı mərhələlərdə avtorizasiya (giriş) tələb edən hər hansı bir endpoint çağırıldıqda (Məs: Səbətə mal atmaq, sifariş keçmək), bu token HTTP Header hissəsinə mütləq şəkildə aşağıdakı formatda yerləşdirilməlidir:
    
    `Authorization: Bearer <accessToken>`
    
- `role` dəyərinə uyğun olaraq istifadəçi birbaşa tətbiqin ana səhifəsinə (Kataloq ekranına) yönləndirilir.

## 7. Code Example (Axios ilə Sorğu Nümunəsi)

JavaScript

```jsx
import axios from 'axios';

const options = {
  method: 'POST',
  url: 'https://api.foodexpress.az/api/v1/auth/register-verify',
  headers: {
    'accept': 'application/json',
    'content-type': 'application/json'
  },
  data: {
    verificationId: 'v-77a8-4cde-bc92', // 1-ci API-dən gələn ID
    otpCode: '4815' // İstifadəçinin ekranda yazdığı kod
  }
};

axios.request(options)
  .then(function (response){
    // Tokenləri yaddaşa vurub ana səhifəyə keçirik
    const token = response.data.auth.accessToken;
    console.log('Giriş uğurludur, Token alındı.');
  })
  .catch(function (error){
    console.error('Doğrulama xətası:', error.response ? error.response.data : error.message);
  });
```
