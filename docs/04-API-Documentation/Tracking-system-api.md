## 1. Nədir və Nə Üçün İstifadə Olunur?

Bu endpoint platformanın mərkəzi **Canlı Data Axını (Real-Time Streaming) qapısıdır**. Siyahıdakı digər bütün API-lardan fərqli olaraq, bu endpoint standart HTTP protokolu ilə deyil, **WebSocket (WS)** protokolu ilə işləyir.

Müştəri tətbiqdə "Sifariş İzləmə" (Order Tracking) ekranını açdığı an, frontend serverlə kəsilməz və ikitərəfli açıq bir rabitə kanalı qurur. Bu kanalın yeganə və mərkəzi vəzifəsi — 11-ci API-dən kuryerin göndərdiyi anlıq GPS koordinatlarını və kuryerin dönmə bucağını (bearing) heç bir gecikmə (delay) olmadan **canlı olaraq müştərinin telefonuna axıtmaqdır**.

## 2. Biznes Loqikası və Sistemə Nəyə Lazımdır?

- **Müştəri Həyəcanının və Dəstək Yükünün Azaldılması:** Müştəri yeməyin yolda olduğunu və kuryerin onun binasına tərəf döndüyünü canlı xəritədə saniyə-saniyə izləyə bildikdə özünü rahat hiss edir. Bu, müştəri xidmətlərinə gələn *"Sifarişim haradadır?"*, *"Kuryer niyə gecikdi?"* tipli zənglərin sayını 80% azaldır.
- **Dinamik Çatdırılma Saniyəsinin Hesablanması (`estimatedArrivalSeconds`):** Kuryer hər hərəkət etdikdə, arxa fondakı logistika mühərriki müştəri ilə kuryer arasındakı məsafəni yenidən ölçür. Tıxac və sürət amillərini nəzərə alaraq, müştərinin ekranındakı "Kuryer 8 dəqiqəyə çatacaq" taymerini anlıq olaraq saniyələr daxilində yeniləyir.
- **Kuryer Dönüş Trayektoriyası (Bearing Control):** Siyahıdan gələn xüsusi bucaq dəyəri sayəsində xəritədəki kuryer (motosiklet/maşın) ikonu kuryerin real hərəkət istiqamətinə uyğun olaraq sağa və ya sola dönür. Bu da vizual olaraq mükəmməl bir istifadəçi təcrübəsi (UX) yaradır.

## 3. Texniki Arxitektura və Data Axını

### Standart HTTP və WebSocket Fərqi:

Müştərinin tətbiqi kuryerin yerini öyrənmək üçün hər 2 saniyədən bir serverə standart HTTP sorğusu (Polling) göndərsəydi, bu həm serveri çökdürərdi, həm də telefonun bateriyasını dərhal bitirərdi. WebSocket sayəsində kanal **1 dəfə açılır**, server kuryerdən yeni koordinat alan kimi (11-ci API-dən gələn data) onu dərhal boru xətti (pipeline) vasitəsilə müştəriyə "itələyir" (Push event).

### Daxili Proses (Step-by-Step):

1. **Handshake (Bağlantı Qurulması):** Frontend `ws://` protokolu ilə serverə müraciət edir. API Gateway istifadəçinin tokenini yoxlayır, əgər bu sifariş həqiqətən həmin müştəriyə məxsusdursa, HTTP bağlantısını WebSocket bağlantısına yüksəldir (Upgrade).
2. **Pub/Sub Mechanism (Abunəlik):** Müştəri daxildə həmin sifarişin mövzu kanalına (Topic: `order:ord-99102:tracking`) abunə yazılır.
3. **Data Broadcast (Yayım):** Kuryer hər 5 saniyədən bir öz koordinatlarını yenilədikcə, mərkəzi `Notification Service` bu kanala qoşulmuş aktiv müştərini tapır və yeni koordinat paketini asinxron olaraq müştərinin tətbiqinə ötürür.

## 4. Endpoint URL Strukturlaşdırılması

- **Protokol:** `ws://` (və ya canlı mühitdə təhlükəsizlik üçün `wss://`)
- **URL:** `ws://api.foodexpress.az/api/v1/orders/{id}/tracking-stream`
- **Path Parameter:** `{id}` — İzlənilən aktiv sifarişin bazadakı unikal identifikatoru (Məs: `ord-99102`).

## 5. Responses (Davamlı Axın Mesaj Paketləri)

Bağlantı açıq qaldığı müddətdə, serverdən frontend tərəfə mütəmadi olaraq aşağıdakı JSON paketləri daxil olur:

JSON

```json
{
  "event": "COURIER_LOCATION_CHANGED",
  "payload": {
    "orderId": "ord-99102",
    "courierId": "cur-7721",
    "coordinates": {
      "latitude": 40.412033,
      "longitude": 49.871544
    },
    "bearing": 120.4,
    "estimatedArrivalSeconds": 480
  }
}
```

## 6. Expected Behavior (Frontend üçün gözlənilən loqika)

- **Xəritə Animasiyası (Smooth Marker Movement):** Frontend gələn yeni `latitude` və `longitude` dəyərlərini birbaşa xəritə markerinə set edir. İkonun xəritədə tullanaraq (abrupt) deyil, rəvan (smooth interpolation/animation) hərəkət etməsi üçün daxili xəritə kitabxanalarından (Google Maps SDK) istifadə olunmalıdır.
- **Marker Rotation:** `bearing` dəyərinə (məsələn: 120.4 dərəcə) əsasən xəritədəki motosiklet ikonu real hərəkət istiqamətinə doğru fırladılır.
- **Yekun Qapanma:** Sifariş statusu 8-ci API-da `DELIVERED` (Təslim edildi) və ya `CANCELLED` olduqda, backend avtomatik olaraq bu WebSocket kanalını bağlayır (Disconnect) və frontend resursları təmizləyir (cleanup memory leaks).

## 7. Code Example (Frontend WebSocket İnteqrasiyası)

JavaScript

```jsx
// Müştəri tətbiqində sifariş izləmə ekranı açılanda işə düşən funksiya
function startLiveTracking(orderId){
  const token = "eyJhbGciOiJIUzI1Ni..."; // Müştərinin mərkəzi JWT tokeni

  // WebSocket bağlantısının mərkəzi URL-ə açılması
  const wsUrl = `ws://api.foodexpress.az/api/v1/orders/${orderId}/tracking-stream?token=${token}`;
  const socket = new WebSocket(wsUrl);

  // Bağlantı uğurla açıldıqda
  socket.onopen = () => {
    console.log("Canlı izləmə kanalı uğurla aktivləşdirildi.");
  };

  // Serverdən hər yeni GPS paketi gəldikdə bu funksiya tetiklənir
  socket.onmessage = (event) => {
    const responseData = JSON.parse(event.data);

    if (responseData.event === "COURIER_LOCATION_CHANGED") {
      const { coordinates, bearing, estimatedArrivalSeconds } = responseData.payload;

      // Xəritədə kuryer markerini və qalan dəqiqəni yeniləyən frontend funksiyaları
      updateMapMarker(coordinates.latitude, coordinates.longitude, bearing);
      updateEstimatedTimeText(estimatedArrivalSeconds);

      console.log("Kuryerin yeni mövqeyi xəritəyə işlənildi:", coordinates);
    }
  };

  // Bağlantı bağlandıqda və ya xəta baş verdikdə
  socket.onclose = () => {
    console.log("Sifariş çatdırıldı və ya ləğv olundu. Kanal qapadıldı.");
  };

  socket.onerror = (error) => {
    console.error("WebSocket axınında daxili kəsinti yyarandı:", error);
  };
}
```
