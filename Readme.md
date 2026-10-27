# Mini Discord

Spring Boot ilə qurulan sadə qrup çatı (group chat). İstifadəçilər qeydiyyatdan keçir, qrup yaradır, digər istifadəçiləri **nickname** ilə qrupa əlavə edir və qrup üzvləri ilə real-vaxtda yazışır.

> **Status:** 🚧 İnkişaf mərhələsindədir. Aşağıdakı [Yol xəritəsi](#yol-xəritəsi) cari vəziyyəti göstərir.

## Əsas imkanlar

- Qeydiyyat və JWT ilə login
- Qrup yaratmaq
- Nickname ilə istifadəçi axtarmaq və qrupa əlavə etmək
- Qrup daxilində real-vaxt mesajlaşma (WebSocket)
- Mesaj tarixçəsi (səhifələnmiş)
- Online/offline status *(son mərhələ)*

## Texnologiyalar

| Qat | Seçim |
|---|---|
| Dil / Framework | Java 21, Spring Boot 3.x |
| Təhlükəsizlik | Spring Security, JWT |
| Real-time | Spring WebSocket (STOMP) |
| Verilənlər bazası | PostgreSQL |
| Migrasiya | Flyway |
| Test | JUnit 5, Mockito, Testcontainers |
| DevOps | Docker, docker-compose |

## Arxitektura

### Verilənlər bazası

**User**
- `id`, `nickname` (unique), `email` (unique), `passwordHash`, `avatarUrl`, `createdAt`

**ChatGroup**
- `id`, `name`, `owner` (→ User), `createdAt`

**GroupMember** *(User ↔ ChatGroup Many-to-Many ara cədvəli)*
- `id`, `group` (→ ChatGroup), `user` (→ User), `role` (OWNER / MEMBER), `joinedAt`
- `(group_id, user_id)` üzərində unique constraint

**Message**
- `id`, `content`, `sender` (→ User), `group` (→ ChatGroup), `sentAt`
- `(group_id, sent_at)` üzərində index

```
User 1 ──< GroupMember >── 1 ChatGroup
ChatGroup 1 ──< Message
User 1 ──< Message
```

> Qeyd: `group` SQL-də rezerv sözdür, cədvəl adı üçün `chat_groups` kimi alternativ seçilməlidir.

### REST API

**Auth**

| Method | Path | Təsvir |
|---|---|---|
| POST | `/api/auth/register` | Qeydiyyat |
| POST | `/api/auth/login` | Login, JWT qaytarır |
| GET | `/api/users/me` | Cari istifadəçi |
| GET | `/api/users?nickname=` | Nickname ilə axtarış |

**Qruplar**

| Method | Path | Təsvir |
|---|---|---|
| POST | `/api/groups` | Qrup yarat (yaradan OWNER olur) |
| GET | `/api/groups` | Üzvü olduğum qruplar |
| GET | `/api/groups/{id}` | Qrup detalı və üzvlər |
| DELETE | `/api/groups/{id}` | Qrupu sil (yalnız OWNER) |
| POST | `/api/groups/{id}/members` | Nickname ilə üzv əlavə et (yalnız OWNER) |
| DELETE | `/api/groups/{id}/members/{userId}` | Üzvü çıxar (yalnız OWNER) |
| DELETE | `/api/groups/{id}/leave` | Qrupdan çıx |

**Mesajlar**

| Method | Path | Təsvir |
|---|---|---|
| GET | `/api/groups/{id}/messages?page=&size=` | Mesaj tarixçəsi (yalnız üzvlər) |

### Real-time mesajlaşma

Qrup üçün ayrıca topic-lərə abunəlik (subscribe) mexanizmi **yoxdur**. Əvəzində server hər mesajı qrup üzvlərinin şəxsi növbəsinə özü göndərir.

| İstiqamət | Ünvan | Məqsəd |
|---|---|---|
| Client → Server | `/app/chat.sendMessage` | Mesaj göndərmə (`groupId`, `content`) |
| Server → Client | `/user/queue/messages` | İstifadəçinin öz şəxsi mesaj növbəsi |

**Axın:**

```
Client ──(CONNECT + JWT)──> ChannelInterceptor (token yoxlanılır)
Client ──SEND /app/chat.sendMessage──> @MessageMapping
   1. göndərən üzvdürmü?
   2. mesajı DB-yə yaz
   3. qrupun hər üzvünə /user/queue/messages vasitəsilə göndər
```

Göndərənin kimliyi payload-dan yox, autentifikasiya olunmuş `Principal`-dan götürülür.

## Yol xəritəsi

- [ ] **Mərhələ 0:** Layihə strukturu, docker-compose (PostgreSQL), Flyway ilə ilk migrasiya
- [ ] **Mərhələ 1:** REST API və təhlükəsizlik (qeydiyyat, login, JWT, qrup və üzv endpointləri)
- [ ] **Mərhələ 2:** WebSocket konfiqurasiyası və JWT yoxlaması (`ChannelInterceptor`)
- [ ] **Mərhələ 3:** Canlı mesajlaşma və mesaj tarixçəsi
- [ ] **Mərhələ 4:** Online/offline status (presence)
- [ ] **Bonus:** Typing indicator, mesaj redaktə/silmə, rate limiting, integration testlər, GitHub Actions CI

## İşə salmaq

> Layihə inkişaf etdikcə bu bölmə yenilənəcək.

```bash
git clone https://github.com/revandev11/mini-discord.git
cd mini-discord
docker compose up -d      # PostgreSQL
./mvnw spring-boot:run
```

## Testlər

```bash
./mvnw test
```

## Müəllif

**Revan Abushov**: Backend Developer (Java, Spring Boot)
GitHub: [@revandev11](https://github.com/revandev11)
