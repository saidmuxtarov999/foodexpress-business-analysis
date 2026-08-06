# **Acceptance Criteria**

Bu bölmə sistemin “tam olaraq işlək və qəbul edilə bilən” sayılması üçün lazım olan ölçülə bilən və test oluna bilən kriteriyaları müəyyən edir.

---

## **1 General Acceptance Principles**

Sistem aşağıdakı şərtləri ödədikdə “Accepted” statusu alır:

- Bütün əsas user flows (login → order → payment → delivery) problemsiz işləyir
- API-lər standart response formatına uyğundur
- Error handling düzgün işləyir və sistem crash etmir
- Performance və security tələbləri qarşılanır
- Real-time tracking və notification işləkdir

---

## **2 Functional Acceptance Criteria**

### 🟢 User Authentication

- User uğurla register və login edə bilir
- Invalid credentials zamanı sistem düzgün error qaytarır
- JWT token düzgün yaradılır və expire olur

---

### 🟢 Restaurant & Menu Flow

- User restoranları problemsiz görə bilir
- Menu data düzgün və tam şəkildə yüklənir
- Search funksiyası düzgün nəticə qaytarır (relevance-based)

---

### 🟢 Order Creation

- User səbət (cart) yarada və dəyişdirə bilir
- Order API uğurla draft order yaradır
- Duplicate order yaranmır

---

### 🟢 Payment Flow

- Uğurlu payment → order confirmed statusuna keçir
- Uğursuz payment → order yaradılmır və ya cancel edilir
- Payment gateway timeout → retry/failover işləyir

---

### 🟢 Restaurant Acceptance

- Restaurant order qəbul və ya reject edə bilir
- Reject halında user dərhal notification alır
- Accept halında order “Preparing” statusuna keçir

---

### 🟢 Courier Flow

- Kuryer tapılır və order ona assign olunur
- Kuryer reject edərsə sistem avtomatik başqa kuryer tapır
- Delivery tracking real-time işləyir (1–3 sec delay max)

---

### 🟢 Order Completion

- Delivery completed statusu düzgün update olunur
- User review yaza bilir
- Notification success/failure düzgün göstərilir

---

## **3 Non-Functional Acceptance Criteria**

### Performance

- API response time ≤ 300ms (p95)
- Checkout flow ≤ 2 seconds
- System 10,000+ concurrent users dəstəkləyir

---

### Security

- Bütün API-lər JWT ilə qorunur
- Sensitive data encrypted saxlanılır
- Unauthorized access mümkün deyil

---

### Availability

- System uptime ≥ 99.9%
- Critical failure zamanı rollback işləyir
- Sistem partial failure zamanı da işləməyə davam edir

---

## **4 Integration Acceptance Criteria**

- Payment gateway uğurla inteqrasiya olunub
- Maps API real-time tracking təmin edir
- Notification service push bildiriş göndərə bilir
- Event bus (queue) message loss olmadan işləyir

---

## **5 End-to-End Acceptance Scenario**

Sistem aşağıdakı flow 100% uğurla işlədikdə qəbul edilir:

1. User login olur
2. Restoran seçir
3. Order yaradır
4. Payment edir
5. Restaurant qəbul edir
6. Kuryer tapılır
7. Delivery həyata keçirilir
8. User order tamamlanmasını görür
