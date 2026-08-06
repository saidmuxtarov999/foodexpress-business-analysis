# **Risks & Mitigation**

Bu bölmə layihə ərzində yarana biləcək riskləri və onların qarşısının alınması üçün tətbiq edilən mitigation strategiyalarını təsvir edir.

---

## **1 Technical Risks**

### 🔴 Risk: System overload (High traffic spike)

- **Description:** Peak saatlarda (lunch/dinner) sistemə həddindən artıq yük düşə bilər.
- **Impact:** Slow response time, request failure, degraded UX

**Mitigation:**

- Auto-scaling infrastructure
- Load balancing (API Gateway level)
- Caching layer (Redis)
- Queue-based request processing

---

### 🔴 Risk: Payment gateway failure

- **Description:** External payment provider unavailable ola bilər.
- **Impact:** Order completion block

**Mitigation:**

- Retry mechanism (exponential backoff)
- Multiple payment provider support (failover)
- Payment status reconciliation job

---

### 🔴 Risk: Microservice communication failure

- **Description:** Service-to-service communication breakdown
- **Impact:** Order flow interruption

**Mitigation:**

- Circuit breaker pattern
- Event-driven architecture (Kafka/RabbitMQ)
- Graceful degradation strategy

---

## **2 Operational Risks**

### 🔴 Risk: Courier shortage in peak hours

- **Description:** Yetərli kuryer olmaya bilər
- **Impact:** Delayed deliveries, cancelled orders

**Mitigation:**

- Dynamic courier allocation
- Surge pricing mechanism
- Queue-based courier assignment system
- Geo-based optimization

---

### 🔴 Risk: Restaurant rejection rate increase

- **Description:** Restoran sifarişi qəbul etməyə bilər
- **Impact:** Order cancellation rate artar

**Mitigation:**

- Restaurant performance scoring system
- Auto-routing to alternative restaurants
- Notification throttling + retry logic

---

## **3 Data & Security Risks**

### 🔴 Risk: Data breach (PII exposure)

- **Description:** User məlumatlarının sızması
- **Impact:** Legal + reputational damage

**Mitigation:**

- AES-256 encryption
- JWT secure authentication
- Role-based access control (RBAC)
- Regular penetration testing

---

### 🔴 Risk: Unauthorized API access

- **Description:** API-lərin icazəsiz istifadəsi
- **Impact:** System abuse, data leakage

**Mitigation:**

- API Gateway security layer
- Rate limiting & throttling
- OAuth2 / JWT validation
- IP filtering for sensitive endpoints

---

## **4 Business Risks**

### 🔴 Risk: Low user adoption

- **Description:** Platform gözlənilən istifadəçi sayına çatmaya bilər
- **Impact:** Revenue reduction

**Mitigation:**

- UX optimization
- Promo campaigns & discounts
- Referral system
- Continuous feedback loop

---

### 🔴 Risk: High cancellation rate

- **Description:** Users orders cancel edə bilər
- **Impact:** Operational inefficiency

**Mitigation:**

- Real-time courier tracking
- Faster delivery SLA
- Notification improvements
- Predictive demand allocation

# **Assumptions**

Bu bölmə layihənin dizaynı və analizində qəbul edilən əsas fərziyyələri (assumptions) təsvir edir. Bu fərziyyələr sistem tələblərinin hazırlanması və arxitektura qərarlarının əsasını təşkil edir.

---

## **1 Business Assumptions**

- Sistem əsasən **on-demand food delivery platform** kimi fəaliyyət göstərəcək
- Restoranlar platformaya inteqrasiya olunmuş partnyorlar kimi çıxış edir
- Bütün sifarişlər mobil tətbiq üzərindən həyata keçirilir
- Kuryer sistemi dinamik (real-time) şəkildə idarə olunur
- Ödənişlər tamamilə onlayn (cashless-first model) qəbul edilir

---

## **2 User Behavior Assumptions**

- İstifadəçilər mobil tətbiqi aktiv internet bağlantısı ilə istifadə edirlər
- İstifadəçilər real-time tracking və notification xidmətlərinə güvənir
- Ortalama istifadəçi sifarişini **10–20 dəqiqə ərzində tamamlayır**
- Promo code və endirimlər istifadə davranışına təsir edir

---

## **3 Technical Assumptions**

- Sistem cloud-based infrastruktura əsaslanır (AWS / Azure / GCP)
- Microservices architecture tətbiq olunur
- Event-driven communication (Kafka / RabbitMQ) istifadə edilir
- API Gateway bütün client request-ləri idarə edir
- Database replication və backup mexanizmi aktivdir

---

## **4 Integration Assumptions**

- Payment Gateway üçün üçüncü tərəf xidmət (Stripe / local bank API) istifadə olunur
- Maps API real-time courier tracking üçün mövcuddur
- SMS / Push notification servisləri external provider üzərindən təmin edilir
- Restoran sistemləri platformaya API ilə inteqrasiya olunmuşdur

---

## **5 Operational Assumptions**

- Sistem 24/7 aktiv işləyir
- Minimum 99.9% availability hədəflənir
- Peak hours (12:00–14:00 və 18:00–21:00) nəzərə alınır
- Monitoring və alerting sistemi aktiv şəkildə işləyir
- Incident management process mövcuddur (L1–L3 support model)

---

## **6 Constraints & Limitations (Assumption-based)**

- Offline order placement dəstəklənmir
- Cash payment sistemi ilkin mərhələdə daxil edilmir
- Kuryer availability region-based məhdud ola bilər
- Real-time tracking yalnız internet bağlantısı olduqda işləyir
