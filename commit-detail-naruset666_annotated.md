# โค้ดทั้งหมดที่ naruset666 ทำ พร้อมคำอธิบายทีละบรรทัด

โปรเจกต์: **How to Survive This Term** (ระบบจัดการการแข่งขัน E-Sport, Java 21 + Spring Boot + PostgreSQL)

**วิธีอ่านเอกสารนี้**

1. อ่านหมวด **0. พื้นฐานที่ต้องรู้ก่อน** (สั้น ๆ แต่ช่วยให้โค้ดที่เหลืออ่านรู้เรื่อง)
2. ไล่อ่านตามลำดับหมวด 1 → 7 ทุกหมวดมี "ไฟล์นี้ทำอะไร" ตามด้วยโค้ดเต็ม และคอมเมนต์ภาษาไทยที่ขึ้นต้นด้วย `// 👉` คือส่วนที่ผมเพิ่มเพื่ออธิบาย (คอมเมนต์เดิมของคุณคงไว้)
3. โค้ดทุกบรรทัดคือโค้ดจริงจาก repo (snapshot ที่ commit `2bcd0ea`) ผมเพิ่มแค่คอมเมนต์ ไม่ได้แก้ตรรกะ

> **หมายเหตุเรื่องความแม่นยำ:** ผมไม่มี diff รายคอมมิต จึงจับคู่ "ชื่อ commit → ไฟล์" ตามชื่อ ถ้าไฟล์ไหนในนี้ไม่ใช่ของคุณจริง ให้ตัดออกได้ (ดูหมวด 8)

---

## สารบัญ

0. พื้นฐานที่ต้องรู้ก่อน
1. ชั้นข้อมูลของ Tournament (enum, repository, DTO) — commit 7 ต.ค.
2. Service: ตรรกะหลักของ Tournament — commit 7–8 ต.ค.
3. Controller: REST API ของ Tournament + ค้นหา/กรอง — commit 8–9 ต.ค.
4. ตรวจสถานะแมตช์ก่อนบันทึกผล — commit 8 ต.ค.
5. ตัวกรองผู้เล่น (ชื่อ / ตำแหน่ง) — commit 8 ต.ค.
6. กฎการลงทะเบียนทีม (Chain of Responsibility) — commit 9 ต.ค.
7. Unit Test ทั้งหมด
8. ส่วนที่ยืนยันไม่ได้ + data dictionary

---

# 0. พื้นฐานที่ต้องรู้ก่อน

## 0.1 โปรแกรมนี้แบ่งเป็น "ชั้น" (Layer)

เปรียบเทียบกับร้านอาหาร

| ชั้น | หน้าที่ | เปรียบเหมือน | ตัวอย่างในโปรเจกต์ |
|---|---|---|---|
| **Controller** | รับคำขอจากเว็บ/แอป (HTTP) แล้วส่งต่อ | พนักงานรับออเดอร์ | `TournamentController` |
| **Service** | ตรรกะและกฎของระบบ | พ่อครัว | `TournamentServiceImpl` |
| **Repository** | คุยกับฐานข้อมูล | ตู้เก็บวัตถุดิบ | `TournamentRepository` |
| **Entity** | ตัวแทนของ "ตาราง" ในฐานข้อมูล | ใบรายการวัตถุดิบ | `Tournament` |
| **DTO** | กล่องข้อมูลที่รับเข้า/ส่งออกทาง API | ถาดเสิร์ฟ | `TournamentRequest`, `TournamentResponse` |

เส้นทางของคำขอหนึ่งครั้ง:

```
ผู้ใช้ส่ง HTTP  →  Controller  →  Service  →  Repository  →  ฐานข้อมูล
ผู้ใช้ได้ JSON ←  Controller  ←  Service  ←  Repository  ←  ฐานข้อมูล
```

## 0.2 Annotation (ตัวที่ขึ้นต้นด้วย @) ที่จะเจอบ่อย

Annotation คือ "ป้ายติดบนโค้ด" บอก Spring ว่าให้จัดการอย่างไร

| Annotation | ความหมาย |
|---|---|
| `@RestController` | คลาสนี้รับคำขอ HTTP และตอบเป็น JSON |
| `@RequestMapping("/api/v1/tournaments")` | คำขอที่ขึ้นต้นด้วย path นี้มาที่คลาสนี้ |
| `@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` | เมธอดนี้รับคำขอแบบ GET (ดู) / POST (สร้าง) / PUT (แก้) / DELETE (ลบ) |
| `@PathVariable Long id` | ดึงค่าจาก path เช่น `/tournaments/5` → `id = 5` |
| `@RequestParam` | ดึงค่าจาก query string เช่น `?name=ffws` |
| `@RequestBody` | แปลง JSON ที่ส่งมาให้เป็น object Java |
| `@Valid` | สั่งให้ตรวจข้อมูลตามกฎที่กำหนดใน DTO |
| `@Service` | คลาสนี้เป็นชั้น Service ให้ Spring สร้างและจัดการให้ |
| `@Component` | คลาสทั่วไปที่ให้ Spring จัดการ |
| `@Transactional` | ทุกอย่างในเมธอดเป็น "ธุรกรรมเดียว" ถ้ามี error จะย้อนกลับทั้งหมด (`readOnly = true` = อ่านอย่างเดียว) |
| `@Override` | เมธอดนี้เขียนทับ/ทำตามเมธอดที่ประกาศใน interface |
| `@Test` | เมธอดนี้คือเทสต์ |

**Dependency Injection (DI):** คลาสไม่สร้างของที่ต้องใช้เอง แต่ "ขอ" ผ่าน constructor แล้ว Spring หยิบมาให้ เช่น `TournamentServiceImpl` ขอ `TournamentRepository` ใน constructor

## 0.3 Repository ไม่ต้องเขียน SQL

`interface TournamentRepository extends JpaRepository<Tournament, Long>` ได้เมธอดพื้นฐานมาเลย (`findAll`, `findById`, `save`, `delete`, `existsById`...) และถ้าตั้งชื่อเมธอดตามรูปแบบ Spring จะสร้าง SQL ให้เอง

| ชื่อเมธอด | SQL ที่ได้โดยประมาณ |
|---|---|
| `findByStatus(s)` | `... WHERE status = s` |
| `existsByName(n)` | `... WHERE name = n` แล้วตอบ true/false |
| `findByNameContainingIgnoreCase(n)` | `... WHERE LOWER(name) LIKE LOWER('%n%')` |
| `...OrderByPlacementAsc` | `... ORDER BY placement ASC` |
| `countByTeamId(t)` | `SELECT COUNT(*) ... WHERE team_id = t` |

## 0.4 Exception → รหัสสถานะ HTTP

เมื่อกฎไม่ผ่าน โค้ดจะ `throw` exception แล้ว `GlobalExceptionHandler` แปลงเป็นรหัส HTTP

| Exception | HTTP | ใช้เมื่อ |
|---|---|---|
| `ResourceNotFoundException` | **404** Not Found | หาข้อมูลไม่เจอ |
| `BusinessException` | **409** Conflict | ขัดกับกฎธุรกิจ/สถานะปัจจุบัน (เช่น ชื่อซ้ำ, ทีมเต็ม) |
| `ValidationException` | **400** Bad Request | ข้อมูลที่ส่งมาผิดรูปแบบ/ผิดกฎ |

## 0.5 ศัพท์ Java ที่จะเจอ

- **`Optional<T>`** = กล่องที่อาจมีหรือไม่มีค่า `findById(id)` คืน Optional เพราะอาจหาไม่เจอ
  `.orElseThrow(() -> new X())` แปลว่า "ถ้าไม่มีค่า ให้ throw X"
- **lambda `() -> ...` / `x -> ...`** = ฟังก์ชันย่อสั้น ๆ
- **`::` (method reference)** เช่น `this::toResponse` = อ้างถึงเมธอด `toResponse` (ย่อจาก `x -> this.toResponse(x)`)
- **Stream** `list.stream().map(f).toList()` = เอาลิสต์มาไล่ทีละตัว แปลงด้วย `f` แล้วเก็บเป็นลิสต์ใหม่
- **`record`** = คลาสเก็บข้อมูลแบบย่อ ประกาศฟิลด์ในวงเล็บ Java สร้าง constructor/getter ให้อัตโนมัติ (getter ชื่อเดียวกับฟิลด์ เช่น `response.id()`)
- **`Long` / `Short`** = ตัวเลขแบบ object (ใส่ `null` ได้ ต่างจาก `long` / `short`)
- **`final`** = ตัวแปรที่กำหนดค่าได้ครั้งเดียว

## 0.6 Unit Test และ Mockito (ใช้ในหมวด 7)

- **Unit test** = โค้ดที่ทดสอบโค้ดอีกชิ้นแบบอัตโนมัติ
- **Mock** = ของปลอมที่แทน Repository จริง เพื่อไม่ต้องต่อฐานข้อมูล
- `when(repo.existsByName("A")).thenReturn(true);` = "ถ้ามีคนเรียก `existsByName("A")` ให้ตอบ true"
- `verify(repo).saveAndFlush(any())` = ตรวจว่า `saveAndFlush` ถูกเรียกจริง
- `assertEquals(expected, actual)` = ค่าต้องเท่ากัน
- `assertThrows(X.class, () -> ...)` = โค้ดข้างในต้อง throw X
- `assertDoesNotThrow(...)` = โค้ดข้างในต้องไม่ throw
- รูปแบบเทสต์: **Arrange** (เตรียมข้อมูล) → **Act** (เรียกของจริง) → **Assert** (ตรวจผล)

---

# 1. ชั้นข้อมูลของ Tournament (commit 7 ต.ค.)

Commit ที่เกี่ยวข้อง: `feat: add tournament status`, `add tournament repository`, `add tournament request DTO`, `add tournament response DTO`

## 1.1 `TournamentStatus.java` — สถานะของรายการแข่ง

**ทำอะไร:** enum = ชุดค่าคงที่ที่กำหนดไว้ล่วงหน้า รายการแข่งหนึ่งรายการมีสถานะได้แค่ 3 แบบนี้

```java
package com.example.tournament.domain.enums;

// 👉 ประกาศ enum เพื่อจำกัดค่าให้เป็นชุดสถานะที่กำหนดไว้
public enum TournamentStatus {

    UPCOMING,    // 👉 ยังไม่เริ่ม (รายการใหม่เริ่มที่ค่านี้เสมอ) ลบ/แก้ทีมได้
    ONGOING,     // 👉 กำลังแข่งอยู่
    COMPLETED    // 👉 แข่งจบแล้ว
}```

**ทำไมใช้ enum:** ป้องกันการพิมพ์สถานะผิด (เช่น "upcomming") เพราะคอมไพเลอร์จะแจ้งทันที

---

## 1.2 `TournamentRepository.java` — ค้นหารายการแข่งจากฐานข้อมูล

**ทำอะไร:** ประกาศเมธอดค้นหา Spring จะสร้างโค้ด SQL ให้เองจากชื่อเมธอด

```java
package com.example.tournament.repository;

import java.util.List;
import java.util.Optional;

import org.springframework.data.jpa.repository.JpaRepository;

import com.example.tournament.domain.entity.Tournament;
import com.example.tournament.domain.enums.TournamentStatus;

// 👉 extends JpaRepository<Tournament, Long> = จัดการตารางของ Tournament, คีย์หลัก (id) เป็นชนิด Long
//    ได้เมธอดพื้นฐานฟรี: findAll, findById, save, delete, existsById ...
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public interface TournamentRepository extends JpaRepository<Tournament, Long> {

    // ค้นหาทัวร์นาเมนต์จากชื่อบางส่วน โดยไม่สนใจตัวพิมพ์ใหญ่หรือเล็ก
    // 👉 Containing = LIKE '%คำค้น%',  IgnoreCase = ไม่สนตัวพิมพ์ เช่น "fire" เจอ "Free Fire Cup"
    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    List<Tournament> findByNameContainingIgnoreCase(String name);

    // ค้นหาทัวร์นาเมนต์ตามสถานะ
    // 👉 WHERE status = ?
    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    List<Tournament> findByStatus(TournamentStatus status);

    // ค้นหาทัวร์นาเมนต์จากชื่อบางส่วนและสถานะพร้อมกัน
    // 👉 WHERE name LIKE ... AND status = ?
    List<Tournament> findByNameContainingIgnoreCaseAndStatus(
            String name,
            TournamentStatus status);

    // 👉 หาด้วยชื่อตรงตัว คืนเป็น Optional (อาจไม่เจอ)
    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    Optional<Tournament> findByName(String name);

    // 👉 เช็คว่ามีชื่อนี้อยู่แล้วหรือไม่ (true/false) ใช้กันชื่อรายการซ้ำตอนสร้าง
    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    boolean existsByName(String name);
}```

---

## 1.3 `TournamentRequest.java` — กล่องข้อมูล "ขาเข้า" (สร้าง/แก้รายการ)

**ทำอะไร:** รับ JSON จากผู้ใช้แล้วแปลงมาเก็บในฟิลด์เหล่านี้ (Spring ใช้ setter เติมค่าให้)

ตัวอย่าง JSON ที่ผู้ใช้ส่งมา:

```json
{
  "name": "FFWS Qualifier",
  "description": "รอบคัดเลือก",
  "gameId": 1,
  "format": "POINTS",
  "totalGames": 6,
  "pointsPerKill": 1,
  "logoUrl": null,
  "startDate": "2026-11-01",
  "endDate": "2026-11-03"
}
```

```java
package com.example.tournament.dto.request;

import java.time.LocalDate;

import com.example.tournament.domain.enums.TournamentFormat;

// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TournamentRequest {

    private Long gameId;                 // 👉 รหัสเกมที่แข่ง (ต้องมีอยู่ในตาราง games)
    private TournamentFormat format;     // 👉 รูปแบบ: POINTS (สะสมคะแนน) หรือ SINGLE_ELIMINATION (แพ้คัดออก)
    private Short totalGames;            // 👉 จำนวนเกมทั้งหมด (ใช้เฉพาะแบบ POINTS)
    private Short pointsPerKill;         // 👉 คะแนนต่อ 1 kill (ใช้กับ Free Fire)
    private String logoUrl;              // 👉 ที่อยู่รูปโลโก้ของรายการ
    private LocalDate startDate;         // 👉 วันเริ่มแข่ง
    private LocalDate endDate;           // 👉 วันจบ

    private String name;                 // 👉 ชื่อรายการ
    private String description;          // 👉 คำอธิบาย

    // 👉 ต่อไปนี้คือ getter/setter ตามแบบมาตรฐานของ Java Bean:
    //    getXxx() = อ่านค่า, setXxx(v) = ตั้งค่า (Spring ใช้ตอนแปลง JSON → object)
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public Long getGameId() {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return gameId;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setGameId(Long gameId) {
        this.gameId = gameId;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TournamentFormat getFormat() {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return format;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setFormat(TournamentFormat format) {
        this.format = format;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public Short getTotalGames() {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return totalGames;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setTotalGames(Short totalGames) {
        this.totalGames = totalGames;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public Short getPointsPerKill() {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return pointsPerKill;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setPointsPerKill(Short pointsPerKill) {
        this.pointsPerKill = pointsPerKill;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public String getLogoUrl() {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return logoUrl;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setLogoUrl(String logoUrl) {
        this.logoUrl = logoUrl;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public LocalDate getStartDate() {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return startDate;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setStartDate(LocalDate startDate) {
        this.startDate = startDate;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public LocalDate getEndDate() {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return endDate;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setEndDate(LocalDate endDate) {
        this.endDate = endDate;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public String getName() {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return name;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setName(String name) {
        this.name = name;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public String getDescription() {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return description;
    }

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setDescription(String description) {
        this.description = description;
    }

}```

> **ข้อสังเกต:** คลาสนี้ไม่มี annotation ตรวจข้อมูล (เช่น `@NotBlank`) ดังนั้น `@Valid` ใน Controller ยังไม่ได้ตรวจอะไร ถ้า `name` เป็น `null` โค้ด `request.getName().trim()` ใน Service จะเกิด `NullPointerException`

---

## 1.4 `TournamentResponse.java` — กล่องข้อมูล "ขาออก"

**ทำอะไร:** รูปแบบข้อมูลที่ส่งกลับให้ผู้ใช้ ใช้ `record` (ประกาศสั้น ๆ ได้ constructor + getter อัตโนมัติ)

```java
package com.example.tournament.dto.response;

import java.time.LocalDate;

import com.example.tournament.domain.enums.TournamentFormat;
import com.example.tournament.domain.enums.TournamentStatus;

// 👉 ทุกค่าในวงเล็บคือฟิลด์ของ record  ใช้อ่านค่าด้วย response.id(), response.name() ...
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public record TournamentResponse(
        Long id,                       // 👉 รหัสรายการที่ฐานข้อมูลสร้างให้
        String name,
        String description,
        Long gameId,                   // 👉 ส่งแค่ id ของเกม ไม่ส่งทั้ง object Game
        TournamentFormat format,
        Short totalGames,
        Short pointsPerKill,
        String logoUrl,
        LocalDate startDate,
        LocalDate endDate,
        TournamentStatus status) {     // 👉 สถานะปัจจุบัน (UPCOMING/ONGOING/COMPLETED)
}```

**ทำไมแยก Request/Response ออกจาก Entity:** Request ไม่ต้องมี `id`/`status` (ระบบกำหนดเอง) ส่วน Response เลือกส่งเฉพาะที่ผู้ใช้ควรเห็น และแก้โครงสร้างฐานข้อมูลได้โดยไม่กระทบรูป JSON

---

## 1.5 `PlacementPointResponse.java` — คะแนนของอันดับหนึ่ง ๆ

```java
package com.example.tournament.dto.response;

// 👉 1 แถว = อันดับที่ (placement) ได้กี่คะแนน (points)  เช่น (1, 12) = อันดับ 1 ได้ 12 คะแนน
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public record PlacementPointResponse(
        Short placement,
        Short points) {
}```

---

## 1.6 `TournamentPlacementPointRepository.java` — เข้าถึงตารางคะแนนอันดับ

```java
package com.example.tournament.repository;

import java.util.List;

import org.springframework.data.jpa.repository.JpaRepository;

import com.example.tournament.domain.entity.TournamentPlacementPoint;
import com.example.tournament.domain.entity.TournamentPlacementPointId;

// 👉 คีย์หลักของตารางนี้เป็น "คีย์คู่" (tournament_id + placement) จึงใช้คลาส TournamentPlacementPointId เป็นชนิดคีย์
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public interface TournamentPlacementPointRepository
        extends JpaRepository<TournamentPlacementPoint, TournamentPlacementPointId> {

    // ใช้ใน FreeFireResultServiceImpl (คำนวณคะแนน)
    // 👉 ดึงคะแนนทุกอันดับของรายการหนึ่ง (ไม่เรียงลำดับ) งานของสมาชิกอีกคนใช้
    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    List<TournamentPlacementPoint> findByTournamentId(Long tournamentId);

    // ใช้ในงาน Tournament ของ Naruset
    // 👉 ดึงคะแนนของรายการหนึ่ง เรียงอันดับ 1 → 12 (ASC = น้อยไปมาก)
    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    List<TournamentPlacementPoint> findByTournamentIdOrderByPlacementAsc(Long tournamentId);
}```

---

## 1.7 `TournamentService.java` — "สัญญา" ของ Service

**ทำอะไร:** interface บอกว่า Service ต้องทำอะไรได้บ้าง (ยังไม่มีเนื้อ) ตัวทำจริงอยู่ใน `TournamentServiceImpl` (หมวด 2) Controller จะพึ่งพา interface นี้ ไม่ผูกกับตัวทำจริง

```java
package com.example.tournament.service;

import java.util.List;

import com.example.tournament.dto.request.TournamentRequest;
import com.example.tournament.dto.response.TournamentResponse;
import com.example.tournament.dto.response.PlacementPointResponse;
import com.example.tournament.domain.enums.TournamentStatus;

// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public interface TournamentService {

    TournamentResponse create(TournamentRequest request);          // 👉 สร้างรายการ

    List<TournamentResponse> getAll();                             // 👉 ดึงทั้งหมด
    List<TournamentResponse> searchByName(String name);            // 👉 ค้นหาตามชื่อ

    TournamentResponse getById(Long id);                           // 👉 ดึงรายการเดียว

    TournamentResponse update(Long id, TournamentRequest request); // 👉 แก้ไข

    void delete(Long id);                                          // 👉 ลบ

    List<PlacementPointResponse> getPlacementPoints(Long tournamentId); // 👉 ดูคะแนนตามอันดับ

    List<TournamentResponse> getByStatus(TournamentStatus status); // 👉 กรองตามสถานะ

    List<TournamentResponse> searchByNameAndStatus(                // 👉 ค้นหาชื่อ + สถานะพร้อมกัน
            String name,
            TournamentStatus status);
}```

---

# 2. Service: ตรรกะหลักของ Tournament (commit 7–8 ต.ค.)

Commit ที่เกี่ยวข้อง: `feat: add tournament service`, `feat: add FFWS placement points`

## 2.1 `TournamentServiceImpl.java` (โค้ดเต็ม)

**ทำอะไร:** นี่คือ "สมอง" ของโมดูล Tournament ตรวจกฎทุกข้อก่อนบันทึก ข้อมูลถึงฐานข้อมูลได้ก็ต่อเมื่อผ่านกฎ

**ภาพรวมเมธอด**

| เมธอด | ทำอะไร |
|---|---|
| `create` | สร้างรายการใหม่ + สร้างคะแนน FFWS ถ้าเป็นแบบ POINTS |
| `getAll` / `getById` | ดึงข้อมูล |
| `searchByName` / `getByStatus` / `searchByNameAndStatus` | ค้นหา/กรอง |
| `update` | แก้ไขรายการ |
| `delete` | ลบ (ได้เฉพาะ UPCOMING) |
| `getPlacementPoints` | ดูคะแนนตามอันดับ |
| `createPlacementPoints` (private) | ใส่คะแนน FFWS 12 อันดับ |
| `validateTournamentRequest` (private) | ตรวจกฎข้อมูล |
| `findTournament` (private) | หารายการ ถ้าไม่เจอ throw 404 |
| `toResponse` (private) | แปลง Entity → Response |

```java
package com.example.tournament.service.impl;

import java.time.LocalDateTime;
import java.util.List;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.example.tournament.domain.entity.Game;
import com.example.tournament.domain.entity.Tournament;
import com.example.tournament.domain.entity.TournamentPlacementPoint;
import com.example.tournament.domain.enums.TournamentFormat;
import com.example.tournament.domain.enums.TournamentStatus;
import com.example.tournament.dto.request.TournamentRequest;
import com.example.tournament.dto.response.PlacementPointResponse;
import com.example.tournament.dto.response.TournamentResponse;
import com.example.tournament.exception.BusinessException;
import com.example.tournament.exception.ResourceNotFoundException;
import com.example.tournament.exception.ValidationException;
import com.example.tournament.repository.GameRepository;
import com.example.tournament.repository.TournamentPlacementPointRepository;
import com.example.tournament.repository.TournamentRepository;
import com.example.tournament.service.TournamentService;

@Service          // 👉 บอก Spring ว่านี่คือ Service ให้สร้างและจัดการให้
@Transactional    // 👉 ทุกเมธอดเป็นธุรกรรม: ถ้า throw exception กลางทาง ข้อมูลที่บันทึกไปแล้วในเมธอดนั้นจะถูกย้อนกลับ
public class TournamentServiceImpl implements TournamentService {   // 👉 implements = ทำตามสัญญาใน TournamentService

    // 👉 ของที่ต้องใช้ 3 อย่าง (final = ตั้งค่าครั้งเดียวใน constructor)
    private final TournamentRepository tournamentRepository;                       // ตารางรายการแข่ง
    private final GameRepository gameRepository;                                   // ตารางเกม
    private final TournamentPlacementPointRepository tournamentPlacementPointRepository; // ตารางคะแนนอันดับ

    // 👉 Constructor: Spring หยิบ Repository 3 ตัวมาให้เอง (Dependency Injection)
    public TournamentServiceImpl(
            TournamentRepository tournamentRepository,
            GameRepository gameRepository,
            TournamentPlacementPointRepository tournamentPlacementPointRepository) {

        this.tournamentRepository = tournamentRepository;
        this.gameRepository = gameRepository;
        this.tournamentPlacementPointRepository = tournamentPlacementPointRepository;
    }

    // ================= สร้างรายการ =================
    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TournamentResponse create(TournamentRequest request) {
        // 👉 ขั้น 1: ตัดช่องว่างหัวท้ายของชื่อ  "  ROV Cup " → "ROV Cup"
        // 👉 ตัดช่องว่างหัวท้ายของข้อความ
        String name = request.getName().trim();

        // 👉 ขั้น 2: ชื่อซ้ำหรือไม่? ถ้าซ้ำ → BusinessException → HTTP 409
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (tournamentRepository.existsByName(name)) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new BusinessException("Tournament name already exists");
        }

        // 👉 ขั้น 3: หาเกมจาก gameId  ถ้าไม่เจอ → ResourceNotFoundException → HTTP 404
        // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
        Game game = gameRepository.findById(request.getGameId())
                // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
                .orElseThrow(() -> new ResourceNotFoundException("Game not found"));

        // 👉 ขั้น 4: ตรวจกฎข้อมูล (วันที่, format ฯลฯ) ถ้าผิด → ValidationException → HTTP 400
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        validateTournamentRequest(request, game);

        // 👉 ขั้น 5: สร้าง Entity แล้วคัดลอกค่าจาก request ใส่
        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        Tournament tournament = new Tournament();

        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setName(name);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setDescription(request.getDescription());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setGame(game);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setFormat(request.getFormat());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setTotalGames(request.getTotalGames());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setPointsPerKill(request.getPointsPerKill());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setLogoUrl(request.getLogoUrl());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setStartDate(request.getStartDate());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setEndDate(request.getEndDate());
        tournament.setStatus(TournamentStatus.UPCOMING);   // 👉 รายการใหม่เริ่มที่ UPCOMING เสมอ ผู้ใช้กำหนดเองไม่ได้
        tournament.setCreatedAt(LocalDateTime.now());      // 👉 เวลาที่สร้าง = ตอนนี้

        // 👉 ขั้น 6: บันทึกลงฐานข้อมูล  saveAndFlush = บันทึกและส่งคำสั่งไปฐานข้อมูลทันที
        //    และคืน object ที่มี id ซึ่งฐานข้อมูลเพิ่งสร้างให้
        // 👉 บันทึกข้อมูลและสั่งให้ JPA ส่งการเปลี่ยนแปลงไปฐานข้อมูลทันที
        Tournament savedTournament = tournamentRepository.saveAndFlush(tournament);

        /*
         * สำหรับทัวร์นาเมนต์ Free Fire รูปแบบ POINTS
         * ระบบจะสร้างคะแนนตามอันดับให้อัตโนมัติ
         */
        // 👉 ขั้น 7: ถ้าเป็นแบบ POINTS ให้ใส่คะแนนอันดับตั้งต้น 12 แถว (ดู createPlacementPoints ด้านล่าง)
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (savedTournament.getFormat() == TournamentFormat.POINTS) {
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            createPlacementPoints(savedTournament);
        }

        // 👉 ขั้น 8: แปลงเป็น Response แล้วส่งกลับ
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return toResponse(savedTournament);
    }

    // ================= ดึงข้อมูล =================
    @Override
    @Transactional(readOnly = true)   // 👉 เมธอดนี้แค่อ่าน ไม่แก้ข้อมูล (เร็วกว่าเล็กน้อย)
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public List<TournamentResponse> getAll() {
        return tournamentRepository.findAll()   // 👉 ดึงทุกแถวจากตาราง tournaments
                .stream()                       // 👉 เปลี่ยนลิสต์เป็น stream เพื่อไล่ทีละตัว
                .map(this::toResponse)          // 👉 แปลง Entity แต่ละตัวเป็น TournamentResponse
                .toList();                      // 👉 เก็บกลับเป็นลิสต์
    }

    // ================= ค้นหาตามชื่อ =================
    @Override
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public List<TournamentResponse> searchByName(String name) {
        // ตัดช่องว่างส่วนเกินก่อนค้นหาชื่อ
        String keyword = name.trim();   // 👉 "  ROV  " → "ROV"

        // ค้นหาจากชื่อบางส่วนโดยไม่สนใจตัวพิมพ์ใหญ่หรือเล็ก
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return tournamentRepository
                .findByNameContainingIgnoreCase(keyword)
                // 👉 เปลี่ยนรายการให้เป็น Stream เพื่อประมวลผลทีละรายการ
                .stream()
                // 👉 แปลงข้อมูลแต่ละรายการให้เป็นรูปแบบที่ต้องการ
                .map(this::toResponse)
                // 👉 รวบรวมผลลัพธ์กลับเป็น List
                .toList();
    }

    // ================= กรองตามสถานะ =================
    @Override
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public List<TournamentResponse> getByStatus(TournamentStatus status) {
        // ค้นหาทัวร์นาเมนต์ตามสถานะที่ระบุ
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return tournamentRepository.findByStatus(status)
                // 👉 เปลี่ยนรายการให้เป็น Stream เพื่อประมวลผลทีละรายการ
                .stream()
                // 👉 แปลงข้อมูลแต่ละรายการให้เป็นรูปแบบที่ต้องการ
                .map(this::toResponse)
                // 👉 รวบรวมผลลัพธ์กลับเป็น List
                .toList();
    }

    // ================= ค้นหาชื่อ + สถานะ =================
    @Override
    @Transactional(readOnly = true)
    public List<TournamentResponse> searchByNameAndStatus(
            String name,
            TournamentStatus status) {

        // ตัดช่องว่างส่วนเกินก่อนค้นหาชื่อ
        // 👉 ตัดช่องว่างหัวท้ายของข้อความ
        String keyword = name.trim();

        // ค้นหาจากชื่อบางส่วนและสถานะพร้อมกัน
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return tournamentRepository
                .findByNameContainingIgnoreCaseAndStatus(keyword, status)
                // 👉 เปลี่ยนรายการให้เป็น Stream เพื่อประมวลผลทีละรายการ
                .stream()
                // 👉 แปลงข้อมูลแต่ละรายการให้เป็นรูปแบบที่ต้องการ
                .map(this::toResponse)
                // 👉 รวบรวมผลลัพธ์กลับเป็น List
                .toList();
    }

    // ================= ดึงรายการเดียว =================
    @Override
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TournamentResponse getById(Long id) {
        return toResponse(findTournament(id));   // 👉 findTournament throw 404 ให้เองถ้าไม่เจอ
    }

    // ================= แก้ไขรายการ =================
    @Override
    public TournamentResponse update(
            Long id,
            TournamentRequest request) {

        // 👉 หารายการเดิม (ไม่เจอ → 404)
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = findTournament(id);

        // 👉 ตัดช่องว่างหัวท้ายของข้อความ
        String name = request.getName().trim();

        // 👉 เช็คชื่อซ้ำ "เฉพาะเมื่อชื่อเปลี่ยนจริง"
        //    ถ้าส่งชื่อเดิมมา (ไม่สนตัวพิมพ์) ไม่ถือว่าซ้ำกับตัวเอง
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (!name.equalsIgnoreCase(tournament.getName())
                && tournamentRepository.existsByName(name)) {

            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new BusinessException("Tournament name already exists");
        }

        // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
        Game game = gameRepository.findById(request.getGameId())
                // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
                .orElseThrow(() -> new ResourceNotFoundException("Game not found"));

        validateTournamentRequest(request, game);   // 👉 ใช้กฎชุดเดียวกับตอนสร้าง

        // 👉 เขียนค่าใหม่ทับ Entity เดิม (ไม่แตะ status/createdAt)
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setName(name);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setDescription(request.getDescription());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setGame(game);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setFormat(request.getFormat());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setTotalGames(request.getTotalGames());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setPointsPerKill(request.getPointsPerKill());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setLogoUrl(request.getLogoUrl());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setStartDate(request.getStartDate());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setEndDate(request.getEndDate());

        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return toResponse(tournamentRepository.saveAndFlush(tournament));
    }

    // ================= ลบรายการ =================
    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void delete(Long id) {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = findTournament(id);

        // 👉 ลบได้เฉพาะรายการที่ยังไม่เริ่ม ถ้าเริ่มหรือจบแล้ว → 409
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (tournament.getStatus() != TournamentStatus.UPCOMING) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new BusinessException(
                    "Tournament can only be deleted when it is UPCOMING");
        }

        // 👉 ลบข้อมูลที่ระบุออกจากฐานข้อมูล
        tournamentRepository.delete(tournament);
        // 👉 สั่งให้ JPA ส่งคำสั่งค้างไปยังฐานข้อมูล
        tournamentRepository.flush();
    }

    // ================= ดูคะแนนตามอันดับ (API) =================
    @Override
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public List<PlacementPointResponse> getPlacementPoints(Long tournamentId) {

        // ตรวจสอบก่อนว่าทัวร์นาเมนต์มีอยู่จริง
        findTournament(tournamentId);   // 👉 ไม่มี → 404 (ค่าที่คืนไม่ได้ใช้ ใช้แค่เช็คว่ามีจริง)

        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return tournamentPlacementPointRepository
                .findByTournamentIdOrderByPlacementAsc(tournamentId)   // 👉 ดึงคะแนนเรียงอันดับ 1→12
                // 👉 เปลี่ยนรายการให้เป็น Stream เพื่อประมวลผลทีละรายการ
                .stream()
                .map(point -> new PlacementPointResponse(              // 👉 แปลงแต่ละแถวเป็น (อันดับ, คะแนน)
                        point.getPlacement(),
                        point.getPoints()))
                // 👉 รวบรวมผลลัพธ์กลับเป็น List
                .toList();
    }

    /**
     * สร้างคะแนนตามอันดับสำหรับทัวร์นาเมนต์รูปแบบ POINTS
     */
    private void createPlacementPoints(Tournament tournament) {

        // 👉 อาร์เรย์คะแนนของอันดับ 1..12  (ช่อง 0 = อันดับ 1, ช่อง 1 = อันดับ 2, ...)
        //    อันดับ: 1  2  3  4  5  6  7  8  9  10 11 12
        //    คะแนน: 12  9  8  7  6  5  4  3  2  1  0  0   (รูปแบบ FFWS)
        short[] points = {
                12, 9, 8, 7, 6, 5,
                4, 3, 2, 1, 0, 0
        };

        // 👉 วนอันดับ 1 ถึง 12 สร้างแถวใหม่ทีละอันดับ
        // 👉 วนทำงานกับข้อมูลทีละรายการตามเงื่อนไขที่กำหนด
        for (short placement = 1; placement <= 12; placement++) {
            // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
            TournamentPlacementPoint placementPoint = new TournamentPlacementPoint();

            placementPoint.setTournament(tournament);           // 👉 ของรายการไหน
            placementPoint.setPlacement(placement);             // 👉 อันดับที่
            placementPoint.setPoints(points[placement - 1]);    // 👉 ได้กี่คะแนน (ลบ 1 เพราะอาร์เรย์เริ่มที่ช่อง 0)

            tournamentPlacementPointRepository.save(placementPoint);   // 👉 บันทึกลงตาราง tournament_placement_points
        }
    }

    // ================= กฎตรวจสอบข้อมูล =================
    private void validateTournamentRequest(
            TournamentRequest request,
            Game game) {

        // 👉 กฎ 1: ต้องมีทั้งวันเริ่มและวันจบ
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (request.getStartDate() == null || request.getEndDate() == null) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException("Start date and end date are required");
        }

        // 👉 กฎ 2: วันจบต้องไม่มาก่อนวันเริ่ม (วันเดียวกันได้)
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (request.getEndDate().isBefore(request.getStartDate())) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException("End date must be on or after start date");
        }

        // 👉 กฎ 3: ต้องระบุรูปแบบการแข่ง
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (request.getFormat() == null) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException("Tournament format is required");
        }

        // 👉 เช็คว่าเกมนี้คือ Free Fire หรือไม่ จากรหัสเกม (ไม่สนตัวพิมพ์)
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        boolean isFreeFire = "FREE_FIRE".equalsIgnoreCase(game.getCode());

        // 👉 กฎ 4: Free Fire ต้องใช้ POINTS เท่านั้น (เพราะเป็นเกมแบบแข่งสะสมคะแนน ไม่ใช่ 1v1)
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (isFreeFire && request.getFormat() != TournamentFormat.POINTS) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException(
                    "Free Fire tournaments must use POINTS format");
        }

        // 👉 กฎ 5: เกมอื่นห้ามใช้ POINTS
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (!isFreeFire && request.getFormat() == TournamentFormat.POINTS) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException(
                    "Only Free Fire tournaments can use POINTS format");
        }

        // 👉 กฎ 6: แบบ POINTS ต้องบอกจำนวนเกมทั้งหมด
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (request.getFormat() == TournamentFormat.POINTS
                && request.getTotalGames() == null) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException("POINTS tournaments require totalGames");
        }

        // 👉 กฎ 7: แบบแพ้คัดออกห้ามมี totalGames (เพราะจำนวนแมตช์คำนวณจากจำนวนทีม)
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (request.getFormat() == TournamentFormat.SINGLE_ELIMINATION
                && request.getTotalGames() != null) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException(
                    "SINGLE_ELIMINATION tournaments must not have totalGames");
        }
    }

    // 👉 เมธอดช่วย: หารายการจาก id ถ้าไม่เจอให้ throw 404 (ใช้ซ้ำหลายที่)
    private Tournament findTournament(Long id) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return tournamentRepository.findById(id)
                // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
                .orElseThrow(() -> new ResourceNotFoundException(
                        "Tournament not found: " + id));
    }

    // 👉 เมธอดช่วย: แปลง Entity เป็น Response (เลือกส่งเฉพาะฟิลด์ที่ผู้ใช้ควรเห็น)
    private TournamentResponse toResponse(Tournament tournament) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return new TournamentResponse(
                tournament.getId(),
                tournament.getName(),
                tournament.getDescription(),
                tournament.getGame().getId(),    // 👉 ส่งแค่ id ของเกม
                tournament.getFormat(),
                tournament.getTotalGames(),
                tournament.getPointsPerKill(),
                tournament.getLogoUrl(),
                tournament.getStartDate(),
                tournament.getEndDate(),
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                tournament.getStatus());
    }
}```

## 2.2 ตัวอย่างการทำงาน: สร้างรายการ Free Fire

**คำขอ:** `POST /api/v1/tournaments`

```json
{ "name": "FFWS Qualifier", "gameId": 1, "format": "POINTS",
  "totalGames": 6, "startDate": "2026-11-01", "endDate": "2026-11-03" }
```

ลำดับใน `create`:

1. ตัดช่องว่างชื่อ → `"FFWS Qualifier"`
2. ชื่อซ้ำไหม? ไม่ซ้ำ ผ่าน
3. หาเกม id=1 เจอ (Free Fire) ผ่าน
4. `validateTournamentRequest`: วันที่ถูกต้อง, Free Fire + POINTS ✔, มี `totalGames` ✔ ผ่าน
5. สร้าง Entity ตั้งสถานะ `UPCOMING` บันทึก ได้ `id`
6. format = POINTS → สร้างคะแนนอันดับ 12 แถว
7. ตอบ **201 Created** พร้อม JSON ของรายการ

**ถ้าผิดกฎ ผู้ใช้จะเจอ:**

| ส่งอะไรมา | ได้อะไร |
|---|---|
| ชื่อซ้ำ | 409 `Tournament name already exists` |
| `gameId` ที่ไม่มี | 404 `Game not found` |
| `endDate` ก่อน `startDate` | 400 `End date must be on or after start date` |
| Free Fire + `SINGLE_ELIMINATION` | 400 `Free Fire tournaments must use POINTS format` |
| ROV + `POINTS` | 400 `Only Free Fire tournaments can use POINTS format` |
| `POINTS` ไม่ใส่ `totalGames` | 400 `POINTS tournaments require totalGames` |

---

# 3. Controller: REST API ของ Tournament (commit 8–9 ต.ค.)

Commit ที่เกี่ยวข้อง: `feat: add tournament crud api`, `add placement points api`, `add tournament name search`, `add tournament status filter`

## 3.1 `TournamentController.java` (โค้ดเต็ม)

**ทำอะไร:** เป็นชั้นรับ HTTP แล้วส่งต่อให้ Service ไม่มีกฎธุรกิจอยู่ในนี้เลย (ตามหลักแบ่งชั้น)

**ตาราง API**

| Method | Path | ทำอะไร | ตอบสำเร็จ |
|---|---|---|---|
| GET | `/api/v1/tournaments` | ดึงทั้งหมด หรือค้นหา/กรอง | 200 |
| GET | `/api/v1/tournaments/{id}` | ดึงรายการเดียว | 200 |
| GET | `/api/v1/tournaments/{id}/placement-points` | ดูคะแนนตามอันดับ | 200 |
| POST | `/api/v1/tournaments` | สร้างรายการ | 201 |
| PUT | `/api/v1/tournaments/{id}` | แก้ไข | 200 |
| DELETE | `/api/v1/tournaments/{id}` | ลบ | 204 (ไม่มีเนื้อหา) |

```java
package com.example.tournament.controller.api;

import java.net.URI;
import java.util.List;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.support.ServletUriComponentsBuilder;

import com.example.tournament.domain.enums.TournamentStatus;
import com.example.tournament.dto.request.TournamentRequest;
import com.example.tournament.dto.response.PlacementPointResponse;
import com.example.tournament.dto.response.TournamentResponse;
import com.example.tournament.service.TournamentService;

import jakarta.validation.Valid;

@RestController                              // 👉 คลาสนี้รับ HTTP และตอบเป็น JSON
@RequestMapping("/api/v1/tournaments")       // 👉 ทุกเมธอดในคลาสนี้ขึ้นต้น path ด้วย /api/v1/tournaments
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TournamentController {

    private final TournamentService tournaments;   // 👉 พึ่งพา interface ไม่ใช่ตัวทำจริง

    public TournamentController(TournamentService tournaments) {   // 👉 Spring ฉีด Service ให้
        this.tournaments = tournaments;
    }

    // ================= ดึงทั้งหมด / ค้นหา / กรอง =================
    @GetMapping   // 👉 GET /api/v1/tournaments
    public List<TournamentResponse> list(
            @RequestParam(required = false) String name,              // 👉 ?name=...   (ไม่ส่งก็ได้)
            @RequestParam(required = false) TournamentStatus status) { // 👉 ?status=... (ไม่ส่งก็ได้)

        // ตรวจสอบว่าผู้ใช้ระบุคำค้นหาชื่อหรือไม่
        boolean hasName = name != null && !name.isBlank();   // 👉 ไม่ใช่ null และไม่ใช่ข้อความว่าง/ช่องว่างล้วน

        // ถ้าไม่ได้ระบุชื่อและสถานะ ให้แสดงทัวร์นาเมนต์ทั้งหมด
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (!hasName && status == null) {
            // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
            return tournaments.getAll();
        }

        // ถ้าระบุเฉพาะชื่อ ให้ค้นหาจากชื่อ
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (status == null) {
            // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
            return tournaments.searchByName(name);
        }

        // ถ้าระบุเฉพาะสถานะ ให้ค้นหาจากสถานะ
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (!hasName) {
            // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
            return tournaments.getByStatus(status);
        }

        // ถ้าระบุทั้งชื่อและสถานะ ให้ค้นหาด้วยเงื่อนไขทั้งสอง
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return tournaments.searchByNameAndStatus(name, status);
    }

    // ================= ดึงรายการเดียว =================
    @GetMapping("/{id}")   // 👉 GET /api/v1/tournaments/5  → id = 5
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TournamentResponse get(@PathVariable Long id) {
        return tournaments.getById(id);   // 👉 ไม่พบ → Service throw 404 → ผู้ใช้ได้ 404
    }

    // ================= ดูคะแนนตามอันดับ =================
    @GetMapping("/{id}/placement-points")   // 👉 GET /api/v1/tournaments/5/placement-points
    public List<PlacementPointResponse> getPlacementPoints(
            @PathVariable Long id) {

        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return tournaments.getPlacementPoints(id);
    }

    // ================= สร้างรายการ =================
    @PostMapping   // 👉 POST /api/v1/tournaments
    public ResponseEntity<TournamentResponse> create(
            @Valid @RequestBody TournamentRequest request) {   // 👉 แปลง JSON ในตัวคำขอ → TournamentRequest

        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        TournamentResponse tournament = tournaments.create(request);

        // 👉 สร้าง URL ของรายการใหม่ = URL ปัจจุบัน + "/{id}"  เช่น http://host/api/v1/tournaments/7
        URI location = ServletUriComponentsBuilder
                .fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(tournament.id())
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                .toUri();

        // 👉 ตอบ 201 Created พร้อม header "Location" และ body เป็น JSON ของรายการที่สร้าง
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return ResponseEntity
                .created(location)
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                .body(tournament);
    }

    // ================= แก้ไข =================
    @PutMapping("/{id}")   // 👉 PUT /api/v1/tournaments/5
    public TournamentResponse update(
            @PathVariable Long id,
            @Valid @RequestBody TournamentRequest request) {

        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return tournaments.update(id, request);
    }

    // ================= ลบ =================
    @DeleteMapping("/{id}")   // 👉 DELETE /api/v1/tournaments/5
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        // 👉 ลบข้อมูลที่ระบุออกจากฐานข้อมูล
        tournaments.delete(id);
        return ResponseEntity.noContent().build();   // 👉 ตอบ 204 No Content (สำเร็จ ไม่มี body)
    }
}```

## 3.2 ตรรกะเลือกค้นหาใน `list` (ตารางสรุป)

| ส่ง `name` | ส่ง `status` | เรียกเมธอด | ตัวอย่าง URL |
|---|---|---|---|
| ไม่ | ไม่ | `getAll()` | `/api/v1/tournaments` |
| มี | ไม่ | `searchByName(name)` | `/api/v1/tournaments?name=ffws` |
| ไม่ | มี | `getByStatus(status)` | `/api/v1/tournaments?status=UPCOMING` |
| มี | มี | `searchByNameAndStatus(name, status)` | `/api/v1/tournaments?name=ffws&status=ONGOING` |

**จุดที่ควรรู้:** `status` ต้องเป็นค่าใน enum พอดี (`UPCOMING`, `ONGOING`, `COMPLETED`) ถ้าพิมพ์ผิด Spring จะตอบ 400 ให้เอง

## 3.3 ตัวอย่างการตอบของ API คะแนนอันดับ

`GET /api/v1/tournaments/1/placement-points`

```json
[
  { "placement": 1,  "points": 12 },
  { "placement": 2,  "points": 9  },
  { "placement": 3,  "points": 8  },
  { "placement": 4,  "points": 7  },
  { "placement": 5,  "points": 6  },
  { "placement": 6,  "points": 5  },
  { "placement": 7,  "points": 4  },
  { "placement": 8,  "points": 3  },
  { "placement": 9,  "points": 2  },
  { "placement": 10, "points": 1  },
  { "placement": 11, "points": 0  },
  { "placement": 12, "points": 0  }
]
```

---

# 4. ตรวจสถานะแมตช์ก่อนบันทึกผล (commit 8 ต.ค.)

Commit: `feat: validate match status before recording result`

## 4.1 `MatchStatus.java` — สถานะของแมตช์

```java
package com.example.tournament.domain.enums;

// 👉 ประกาศ enum เพื่อจำกัดค่าให้เป็นชุดสถานะที่กำหนดไว้
public enum MatchStatus {
    PENDING,     // 👉 รอ: ยังไม่รู้ทีมครบ (เช่น รอผู้ชนะจากรอบก่อน)
    SCHEDULED,   // 👉 กำหนดแล้ว: ทีมครบ พร้อมแข่ง/พร้อมบันทึกผล
    COMPLETED    // 👉 จบแล้ว: มีผลการแข่งขันแล้ว
}```

## 4.2 `MatchResultServiceImpl.java` (โค้ดเต็ม)

**ทำอะไร:** บันทึกผลของแมตช์แบบแพ้คัดออก ที่คุณเพิ่มคือ **ข้อ 4** (บังคับให้แมตช์เป็น `SCHEDULED` ก่อน) ส่วนข้ออื่นเป็นของเดิมในไฟล์เดียวกัน (ผมอธิบายครบเพื่อให้เห็นภาพรวม)

```java
package com.example.tournament.service.impl;

import java.time.LocalDateTime;

import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.example.tournament.domain.entity.Match;
import com.example.tournament.domain.entity.MatchResult;
import com.example.tournament.domain.entity.Team;
import com.example.tournament.domain.enums.MatchStatus;
import com.example.tournament.dto.request.CreateMatchResultRequest;
import com.example.tournament.dto.response.MatchResultResponse;
import com.example.tournament.event.MatchResultRecordedEvent;
import com.example.tournament.exception.BusinessException;
import com.example.tournament.exception.ResourceNotFoundException;
import com.example.tournament.exception.ValidationException;
import com.example.tournament.mapper.MatchResultMapper;
import com.example.tournament.repository.MatchRepository;
import com.example.tournament.repository.MatchResultRepository;
import com.example.tournament.service.MatchResultService;

@Service
@Transactional
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class MatchResultServiceImpl implements MatchResultService {

    private final MatchRepository matches;               // 👉 ตารางแมตช์
    private final MatchResultRepository results;         // 👉 ตารางผลการแข่ง
    private final MatchResultMapper mapper;              // 👉 ตัวแปลง Entity → Response
    private final ApplicationEventPublisher events;      // 👉 ตัวประกาศ "เหตุการณ์" (Observer Pattern)

    public MatchResultServiceImpl(MatchRepository matches, MatchResultRepository results,
            MatchResultMapper mapper, ApplicationEventPublisher events) {
        this.matches = matches;
        this.results = results;
        this.mapper = mapper;
        this.events = events;
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public MatchResultResponse record(Long matchId, CreateMatchResultRequest request) {
        // 1. แมตช์ต้องมีอยู่จริง → 404
        // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
        Match match = matches.findById(matchId)
                // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
                .orElseThrow(() -> new ResourceNotFoundException("Match not found: " + matchId));

        // 2. แมตช์ต้องยังไม่มีผล → 409
        // 👉 กันบันทึกผลซ้ำ
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (results.existsByMatchId(matchId)) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new BusinessException("Match already has a result");
        }

        // 3. ต้องมีทีมครบทั้งสองฝั่ง → 409
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team teamA = match.getTeamA();
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team teamB = match.getTeamB();
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (teamA == null || teamB == null) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new BusinessException("Both teams must be set before recording a result");
        }

        // 4. Match ต้องอยู่ในสถานะ SCHEDULED ก่อนบันทึกผล
        // 👉 ★ ส่วนที่เพิ่มใน commit ของคุณ ★
        //    match.getStatus() เป็น String ("PENDING"/"SCHEDULED"/"COMPLETED")
        //    จึงเทียบกับ MatchStatus.SCHEDULED.name() (แปลง enum เป็นข้อความ "SCHEDULED")
        //    ถ้าไม่ใช่ SCHEDULED (เช่น ยัง PENDING หรือ COMPLETED แล้ว) → 409
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (!MatchStatus.SCHEDULED.name().equals(match.getStatus())) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new BusinessException(
                    "Match must be SCHEDULED before recording a result");
        }

        // 5. คะแนนไม่ติดลบ ตรวจแล้วใน DTO ด้วย @Min(0) → 400
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        int scoreA = request.teamAScore();
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        int scoreB = request.teamBScore();

        // 6. แพ้คัดออกห้ามเสมอ → 400
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (scoreA == scoreB) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException("Single elimination match cannot end in a draw");
        }

        // 7. ผู้ชนะต้องเป็นทีมในแมตช์ → 400
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Long winnerId = request.winnerTeamId();
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (!winnerId.equals(teamA.getId()) && !winnerId.equals(teamB.getId())) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException("Winner must be one of the teams in this match");
        }

        // 8. ผู้ชนะต้องมีคะแนนมากกว่า → 400
        // 👉 คำนวณผู้ชนะจากคะแนนเอง แล้วเทียบกับ winnerTeamId ที่ผู้ใช้ส่งมา ต้องตรงกัน
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team winner = scoreA > scoreB ? teamA : teamB;
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (!winner.getId().equals(winnerId)) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException("Winner must have the higher score");
        }

        // 👉 ผ่านทุกข้อ → สร้างผลการแข่งและบันทึก
        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        MatchResult result = new MatchResult();
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        result.setMatch(match);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        result.setTeamAScore(scoreA);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        result.setTeamBScore(scoreB);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        result.setWinnerTeam(winner);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        result.setCreatedAt(LocalDateTime.now());
        // 👉 บันทึกข้อมูลลงฐานข้อมูล
        MatchResult saved = results.save(result);

        // 👉 แมตช์นี้จบแล้ว เปลี่ยนสถานะเป็น COMPLETED
        // 👉 กำหนดสถานะใหม่ให้กับข้อมูล
        match.setStatus(MatchStatus.COMPLETED.name());

        // Observer: ประกาศว่ามีผลใหม่ ให้ Listener ส่งผู้ชนะไปแมตช์ถัดไป
        // 👉 ประกาศเหตุการณ์ "บันทึกผลแล้ว" Listener (อีกคลาส) จะรับไปเลื่อนผู้ชนะเข้ารอบถัดไป
        // 👉 ประกาศ Event เพื่อให้ Listener ที่เกี่ยวข้องทำงานต่อ
        events.publishEvent(new MatchResultRecordedEvent(match.getId(), winner.getId()));

        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return mapper.toResponse(saved);
    }

    @Override
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public MatchResultResponse get(Long matchId) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return results.findByMatchId(matchId)
                // 👉 แปลงข้อมูลแต่ละรายการให้เป็นรูปแบบที่ต้องการ
                .map(mapper::toResponse)
                // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
                .orElseThrow(() -> new ResourceNotFoundException("Result not found for match: " + matchId));
    }
}```

**ทำไมต้องเพิ่มข้อ 4:** ข้อ 3 เช็คแค่ว่ามีทีมครบ แต่แมตช์ที่ทีมครบแล้วอาจมีสถานะ `COMPLETED` อยู่ก่อน ข้อ 4 ช่วยให้บันทึกผลได้เฉพาะแมตช์ที่ถึงคิวแข่งจริง (`SCHEDULED`)

**เทสต์ที่ตรวจข้อ 4** (ใน `MatchResultServiceImplTest`): `rejectsWhenMatchIsNotScheduled`, `rejectsWhenMatchIsAlreadyCompleted`

---

# 5. ตัวกรองผู้เล่น (commit 8 ต.ค.)

Commit: `feat: add player name filter`, `feat: add player role filter`

มี 2 จุดที่เกี่ยวข้อง
- **กรองชื่อ** ที่รายการผู้เล่นทั้งระบบ: `GET /api/v1/players?name=...`
- **กรองชื่อ + ตำแหน่ง (role)** ที่ผู้เล่นในทีม: `GET /api/v1/teams/{teamId}/players?name=...&role=...`

## 5.1 `PlayerController.java` (กรองชื่อ)

```java
package com.example.tournament.controller.api;

import java.net.URI;

import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.support.ServletUriComponentsBuilder;

import com.example.tournament.dto.request.PlayerRequest;
import com.example.tournament.dto.response.PageResponse;
import com.example.tournament.dto.response.PlayerResponse;
import com.example.tournament.service.PlayerService;

import jakarta.validation.Valid;

@RestController
@RequestMapping("/api/v1/players")
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class PlayerController {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final PlayerService players;

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public PlayerController(PlayerService players) {
        this.players = players;
    }

    // 👉 ★ ตัวกรองชื่อ ★  ?name=... (ไม่ส่งก็ได้)
    //    Pageable = ตัวบอกว่าจะขอหน้าที่เท่าไร ขนาดหน้าเท่าไร
    //    @PageableDefault(size = 20, sort = "name") = ถ้าไม่ระบุ ให้หน้าละ 20 รายการ เรียงตามชื่อ
    //    ผู้ใช้เปลี่ยนได้ เช่น ?page=1&size=10
    @GetMapping
    public PageResponse<PlayerResponse> list(@RequestParam(required = false) String name,
            @PageableDefault(size = 20, sort = "name") Pageable pageable) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return PageResponse.from(players.list(name, pageable));
    }

    @GetMapping("/{id}")
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public PlayerResponse get(@PathVariable Long id) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return players.get(id);
    }

    @PostMapping
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public ResponseEntity<PlayerResponse> create(@Valid @RequestBody PlayerRequest request) {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        PlayerResponse player = players.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest().path("/{id}")
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                .buildAndExpand(player.id()).toUri();
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return ResponseEntity.created(location).body(player);
    }

    @PutMapping("/{id}")
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public PlayerResponse update(@PathVariable Long id, @Valid @RequestBody PlayerRequest request) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return players.update(id, request);
    }

    @DeleteMapping("/{id}")
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        // 👉 ลบข้อมูลที่ระบุออกจากฐานข้อมูล
        players.delete(id);
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return ResponseEntity.noContent().build();
    }
}```

## 5.2 `PlayerServiceImpl.java` (ส่วนที่เกี่ยวกับตัวกรองชื่อ — โค้ดเต็มของไฟล์)

```java
package com.example.tournament.service.impl;

import java.time.LocalDateTime;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.example.tournament.domain.entity.Player;
import com.example.tournament.domain.entity.Team;
import com.example.tournament.dto.request.PlayerRequest;
import com.example.tournament.dto.response.PlayerResponse;
import com.example.tournament.exception.ResourceNotFoundException;
import com.example.tournament.mapper.PlayerMapper;
import com.example.tournament.repository.PlayerRepository;
import com.example.tournament.repository.TeamRepository;
import com.example.tournament.service.PlayerService;

@Service
@Transactional
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class PlayerServiceImpl implements PlayerService {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final PlayerRepository players;
    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final TeamRepository teams;
    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final PlayerMapper mapper;

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public PlayerServiceImpl(PlayerRepository players, TeamRepository teams, PlayerMapper mapper) {
        this.players = players;
        this.teams = teams;
        this.mapper = mapper;
    }

    // 👉 ★ ตัวกรองชื่อ ★
    @Override
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public Page<PlayerResponse> list(String name, Pageable pageable) {
        // 👉 ถ้าไม่ได้ส่งชื่อ (null หรือว่าง) → ดึงทุกคน (แบ่งหน้า)
        //    ถ้าส่งชื่อ → ค้นชื่อที่มีคำนั้น ไม่สนตัวพิมพ์ (ตัดช่องว่างหัวท้ายก่อน)
        //    รูปแบบ  เงื่อนไข ? ค่าถ้าจริง : ค่าถ้าเท็จ  (ternary operator)
        Page<Player> result = name == null || name.isBlank()
                ? players.findAll(pageable)
                // 👉 ตัดช่องว่างหัวท้ายของข้อความ
                : players.findByNameContainingIgnoreCase(name.trim(), pageable);
        return result.map(mapper::toResponse);   // 👉 แปลงทุกคนในหน้านั้นเป็น Response
    }

    @Override
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public PlayerResponse get(Long id) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return mapper.toResponse(findPlayer(id));
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public PlayerResponse create(PlayerRequest request) {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = findOptionalTeam(request.teamId());
        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        Player player = new Player();
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        player.setName(request.name().trim());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        player.setRole(request.role().trim());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        player.setDescription(request.description());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        player.setTeam(team);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        player.setCreatedAt(LocalDateTime.now());
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return mapper.toResponse(players.saveAndFlush(player));
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public PlayerResponse update(Long id, PlayerRequest request) {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Player player = findPlayer(id);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = findOptionalTeam(request.teamId());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        player.setName(request.name().trim());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        player.setRole(request.role().trim());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        player.setDescription(request.description());
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        player.setTeam(team);
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return mapper.toResponse(players.saveAndFlush(player));
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void delete(Long id) {
        // 👉 ลบข้อมูลที่ระบุออกจากฐานข้อมูล
        players.delete(findPlayer(id));
        // 👉 สั่งให้ JPA ส่งคำสั่งค้างไปยังฐานข้อมูล
        players.flush();
    }

    private Player findPlayer(Long id) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return players.findById(id)
                // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
                .orElseThrow(() -> new ResourceNotFoundException("Player not found: " + id));
    }

    // 👉 ผู้เล่นไม่จำเป็นต้องมีทีม: ถ้า teamId เป็น null คืน null ถ้ามีแต่หาไม่เจอ → 404
    private Team findOptionalTeam(Long teamId) {
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (teamId == null) {
            // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
            return null;
        }
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return teams.findById(teamId)
                // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
                .orElseThrow(() -> new ResourceNotFoundException("Team not found: " + teamId));
    }
}```

## 5.3 `PlayerRepository.java` — เมธอดค้นหาที่รองรับตัวกรอง

```java
package com.example.tournament.repository;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

import com.example.tournament.domain.entity.Player;

// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public interface PlayerRepository extends JpaRepository<Player, Long> {

    // ค้นหา Player ทั้งหมดใน Team
    // 👉 WHERE team_id = ?   (Page<> = ผลลัพธ์แบบแบ่งหน้า)
    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    Page<Player> findByTeamId(Long teamId, Pageable pageable);

    // ค้นหา Player ใน Team ตามชื่อ
    // 👉 WHERE team_id = ? AND name LIKE %?%  (ไม่สนตัวพิมพ์)
    Page<Player> findByTeamIdAndNameContainingIgnoreCase(
            Long teamId,
            String name,
            Pageable pageable);

    // ค้นหา Player ใน Team ตาม Role
    // 👉 WHERE team_id = ? AND role = ?  (ตรงทั้งคำ แต่ไม่สนตัวพิมพ์ เช่น "support" = "Support")
    Page<Player> findByTeamIdAndRoleIgnoreCase(
            Long teamId,
            String role,
            Pageable pageable);

    // ค้นหา Player ใน Team ตามทั้งชื่อและ Role
    // 👉 ใช้เมื่อผู้ใช้ส่งทั้ง name และ role
    Page<Player> findByTeamIdAndNameContainingIgnoreCaseAndRoleIgnoreCase(
            Long teamId,
            String name,
            String role,
            Pageable pageable);

    // นับจำนวน Player ใน Team
    // 👉 ใช้ในกฎ "ทีมต้องมีผู้เล่นอย่างน้อย N คน" (หมวด 6)
    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    long countByTeamId(Long teamId);

    // ค้นหา Player จากชื่อ โดยไม่จำกัด Team
    // 👉 ใช้กับตัวกรองชื่อที่ /api/v1/players
    Page<Player> findByNameContainingIgnoreCase(
            String name,
            Pageable pageable);
}```

## 5.4 `TeamMembershipController.java` + `TeamMembershipServiceImpl.java` (กรองชื่อ + ตำแหน่ง)

```java
// ===== TeamMembershipController.java =====
package com.example.tournament.controller.api;

import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import com.example.tournament.dto.response.PageResponse;
import com.example.tournament.dto.response.TeamPlayerResponse;
import com.example.tournament.service.TeamMembershipService;

@RestController
@RequestMapping("/api/v1/teams/{teamId}/players")   // 👉 {teamId} อยู่ใน path เช่น /api/v1/teams/3/players
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TeamMembershipController {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final TeamMembershipService memberships;

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TeamMembershipController(TeamMembershipService memberships) {
        this.memberships = memberships;
    }

    // 👉 ★ ตัวกรองชื่อ + ตำแหน่ง ★  ส่งแค่ตัวใดตัวหนึ่ง หรือทั้งคู่ หรือไม่ส่งเลยก็ได้
    @GetMapping
    public PageResponse<TeamPlayerResponse> list(@PathVariable Long teamId,
            @RequestParam(required = false) String name,
            @RequestParam(required = false) String role,
            @PageableDefault(size = 20, sort = "name") Pageable pageable) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return PageResponse.from(memberships.listPlayers(teamId, name, role, pageable));
    }

    @PutMapping("/{playerId}")
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TeamPlayerResponse add(@PathVariable Long teamId, @PathVariable Long playerId) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return memberships.addPlayer(teamId, playerId);
    }

    @DeleteMapping("/{playerId}")
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public ResponseEntity<Void> remove(@PathVariable Long teamId, @PathVariable Long playerId) {
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        memberships.removePlayer(teamId, playerId);
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return ResponseEntity.noContent().build();
    }
}```

```java
// ===== TeamMembershipServiceImpl.java =====
package com.example.tournament.service.impl;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.example.tournament.domain.entity.Player;
import com.example.tournament.domain.entity.Team;
import com.example.tournament.dto.response.TeamPlayerResponse;
import com.example.tournament.exception.ResourceNotFoundException;
import com.example.tournament.mapper.TeamMembershipMapper;
import com.example.tournament.repository.PlayerRepository;
import com.example.tournament.repository.TeamRepository;
import com.example.tournament.service.TeamMembershipService;

@Service
@Transactional
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TeamMembershipServiceImpl implements TeamMembershipService {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final TeamRepository teams;
    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final PlayerRepository players;
    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final TeamMembershipMapper mapper;

    public TeamMembershipServiceImpl(TeamRepository teams, PlayerRepository players,
            TeamMembershipMapper mapper) {
        this.teams = teams;
        this.players = players;
        this.mapper = mapper;
    }

    @Override
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public Page<TeamPlayerResponse> listPlayers(Long teamId, String name, String role, Pageable pageable) {
        findTeam(teamId);   // 👉 ทีมไม่มีอยู่จริง → 404

        // 👉 เช็คว่าผู้ใช้ส่ง name / role มาจริงหรือไม่ (ไม่ใช่ null และไม่ว่าง)
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        boolean hasName = name != null && !name.isBlank();
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        boolean hasRole = role != null && !role.isBlank();

        // 👉 เลือกเมธอดค้นหาตาม 4 กรณี (ชื่อ?/ตำแหน่ง?) ตารางสรุปด้านล่าง
        Page<Player> result;
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (hasName && hasRole) {
            result = players.findByTeamIdAndNameContainingIgnoreCaseAndRoleIgnoreCase(
                    // 👉 ตัดช่องว่างหัวท้ายของข้อความ
                    teamId, name.trim(), role.trim(), pageable);
        } else if (hasName) {
            // 👉 ตัดช่องว่างหัวท้ายของข้อความ
            result = players.findByTeamIdAndNameContainingIgnoreCase(teamId, name.trim(), pageable);
        } else if (hasRole) {
            // 👉 ตัดช่องว่างหัวท้ายของข้อความ
            result = players.findByTeamIdAndRoleIgnoreCase(teamId, role.trim(), pageable);
        } else {
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            result = players.findByTeamId(teamId, pageable);
        }
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return result.map(mapper::toResponse);
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TeamPlayerResponse addPlayer(Long teamId, Long playerId) {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = findTeam(teamId);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Player player = findPlayer(playerId);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        player.setTeam(team);
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return mapper.toResponse(players.save(player));
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void removePlayer(Long teamId, Long playerId) {
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        findTeam(teamId);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Player player = findPlayer(playerId);
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (player.getTeam() == null || !teamId.equals(player.getTeam().getId())) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ResourceNotFoundException("Player is not in this team: " + playerId);
        }
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        player.setTeam(null);
        // 👉 บันทึกข้อมูลลงฐานข้อมูล
        players.save(player);
    }

    private Team findTeam(Long id) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return teams.findById(id)
                // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
                .orElseThrow(() -> new ResourceNotFoundException("Team not found: " + id));
    }

    private Player findPlayer(Long id) {
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return players.findById(id)
                // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
                .orElseThrow(() -> new ResourceNotFoundException("Player not found: " + id));
    }
}```

| ผู้ใช้ส่ง | เมธอด Repository ที่ถูกเรียก | ตัวอย่าง URL |
|---|---|---|
| ไม่ส่งอะไร | `findByTeamId` | `/api/v1/teams/3/players` |
| ชื่ออย่างเดียว | `findByTeamIdAndNameContainingIgnoreCase` | `/api/v1/teams/3/players?name=jo` |
| ตำแหน่งอย่างเดียว | `findByTeamIdAndRoleIgnoreCase` | `/api/v1/teams/3/players?role=support` |
| ทั้งสอง | `findByTeamIdAndNameContainingIgnoreCaseAndRoleIgnoreCase` | `/api/v1/teams/3/players?name=jo&role=support` |

> **หมายเหตุ:** `role` ต้องตรงทั้งคำ (เช่น `support`) ไม่ใช่ค้นบางส่วน ต่างจาก `name` ที่ค้นบางส่วนได้

---

# 6. กฎการลงทะเบียนทีม — Chain of Responsibility (commit 9 ต.ค.)

Commit: `feat: add tournament team registration rules` (และเทสต์ในหมวด 7)

## 6.0 ภาพรวม: ปัญหาและวิธีแก้

**โจทย์:** เมื่อ Admin เพิ่มทีมเข้ารายการ ต้องตรวจ 8 เรื่อง (ทีมมีจริงไหม, รายการเริ่มแล้วหรือยัง, ทีมซ้ำไหม, เกมตรงกันไหม, ผู้เล่นพอไหม, ทีมเต็มไหม, วันแข่งชนไหม ...)

**วิธีที่ไม่ดี:** เขียน `if` ยาว ๆ 8 ข้อในเมธอดเดียว → อ่านยาก แก้ยาก เทสต์ยาก

**วิธีที่ใช้ (Chain of Responsibility):** แยกแต่ละข้อเป็น "ด่านตรวจ" 1 คลาส เรียงต่อกันเป็นโซ่

```
ทีม+รายการ → [1 ทีมมีจริง] → [2 รายการมีจริง] → [3 รายการ UPCOMING] → [4 ทีมยังไม่สมัคร]
          → [5 เกมตรงกัน] → [6 ผู้เล่นพอ] → [7 ไม่เกิน 12 ทีม] → [8 วันแข่งไม่ชน] → ผ่าน
             ด่านใดไม่ผ่านจะ throw ทันที และด่านถัดไปจะไม่ถูกเรียก
```

**หลักการของทุกด่าน (เหมือนกันหมด)**

1. ตรวจเงื่อนไขของตัวเอง → ไม่ผ่านก็ `throw`
2. ถ้าผ่านและมีด่านถัดไป (`nextRule != null`) → เรียก `nextRule.validate(...)`

| ลำดับ | คลาส | ตรวจอะไร | ถ้าไม่ผ่าน |
|---|---|---|---|
| 1 | `TeamExistsRule` | ทีมมีอยู่จริง | 404 |
| 2 | `TournamentExistsRule` | รายการมีอยู่จริง | 404 |
| 3 | `TournamentStatusRule` | รายการ `UPCOMING` และยังไม่สร้างตารางแข่ง | 409 |
| 4 | `TeamNotAlreadyJoinedRule` | ทีมยังไม่อยู่ในรายการนี้ | 409 |
| 5 | `TeamGameMatchRule` | เกมของทีม = เกมของรายการ | 409 |
| 6 | `TeamMinimumPlayersRule` | ผู้เล่นในทีม ≥ ขั้นต่ำของเกม | 409 |
| 7 | `TeamPointsLimitRule` | รายการ POINTS ไม่เกิน 12 ทีม | 409 |
| 8 | `TeamTournamentDateRule` | วันแข่งไม่ทับรายการอื่นของทีม | 409 |

## 6.1 `TeamJoinRule.java` — interface กลางของทุกด่าน

```java
package com.example.tournament.service.rule;

import com.example.tournament.domain.entity.Team;
import com.example.tournament.domain.entity.Tournament;

// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public interface TeamJoinRule {

    // 👉 ตรวจกฎ: ผ่าน = ไม่ทำอะไร / ไม่ผ่าน = throw exception
    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    void validate(Team team, Tournament tournament);

    // 👉 กำหนด "ด่านถัดไป" ที่จะส่งต่อให้เมื่อด่านนี้ผ่าน
    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    void setNext(TeamJoinRule nextRule);
}```

## 6.2 `TeamExistsRule.java` — ด่าน 1: ทีมมีอยู่จริง

```java
package com.example.tournament.service.rule;

import org.springframework.stereotype.Component;

import com.example.tournament.domain.entity.Team;
import com.example.tournament.domain.entity.Tournament;
import com.example.tournament.exception.ResourceNotFoundException;
import com.example.tournament.repository.TeamRepository;

@Component   // 👉 ให้ Spring สร้างอ็อบเจกต์นี้ให้ จะได้ถูกฉีดเข้า TeamJoinRuleChain
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TeamExistsRule implements TeamJoinRule {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final TeamRepository teamRepository;
    private TeamJoinRule nextRule;   // 👉 ด่านถัดไป (ตอนแรกยังไม่มี = null จนกว่า Chain จะมาต่อให้)

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TeamExistsRule(TeamRepository teamRepository) {
        this.teamRepository = teamRepository;
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void validate(Team team, Tournament tournament) {
        // 👉 existsById = ถามฐานข้อมูลว่ามีทีม id นี้ไหม  ไม่มี → 404
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (!teamRepository.existsById(team.getId())) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ResourceNotFoundException(
                "Team not found: " + team.getId()
            );
        }

        // 👉 ผ่านแล้ว ส่งต่อให้ด่านถัดไป (ถ้ามี)
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (nextRule != null) {
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            nextRule.validate(team, tournament);
        }
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setNext(TeamJoinRule nextRule) {
        this.nextRule = nextRule;
    }
}```

## 6.3 `TournamentExistsRule.java` — ด่าน 2: รายการมีอยู่จริง

```java
package com.example.tournament.service.rule;

import org.springframework.stereotype.Component;

import com.example.tournament.domain.entity.Team;
import com.example.tournament.domain.entity.Tournament;
import com.example.tournament.exception.ResourceNotFoundException;
import com.example.tournament.repository.TournamentRepository;

@Component
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TournamentExistsRule implements TeamJoinRule {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final TournamentRepository tournamentRepository;
    // 👉 ประกาศฟิลด์ที่คลาสใช้เก็บข้อมูลหรือ Dependency
    private TeamJoinRule nextRule;

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TournamentExistsRule(TournamentRepository tournamentRepository) {
        this.tournamentRepository = tournamentRepository;
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void validate(Team team, Tournament tournament) {
        // 👉 โครงเดียวกับด่าน 1 แต่เช็คตารางรายการแข่ง ไม่มี → 404
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (!tournamentRepository.existsById(tournament.getId())) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ResourceNotFoundException(
                "Tournament not found: " + tournament.getId()
            );
        }

        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (nextRule != null) {
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            nextRule.validate(team, tournament);
        }
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setNext(TeamJoinRule nextRule) {
        this.nextRule = nextRule;
    }
}```

## 6.4 `TournamentStatusRule.java` — ด่าน 3: ต้อง UPCOMING และยังไม่สร้างตารางแข่ง

```java
package com.example.tournament.service.rule;

import org.springframework.stereotype.Component;

import com.example.tournament.domain.entity.Team;
import com.example.tournament.domain.entity.Tournament;
import com.example.tournament.domain.enums.TournamentStatus;
import com.example.tournament.exception.BusinessException;
import com.example.tournament.repository.MatchRepository;

@Component
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TournamentStatusRule implements TeamJoinRule {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final MatchRepository matchRepository;
    // 👉 ประกาศฟิลด์ที่คลาสใช้เก็บข้อมูลหรือ Dependency
    private TeamJoinRule nextRule;

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TournamentStatusRule(MatchRepository matchRepository) {
        this.matchRepository = matchRepository;
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void validate(Team team, Tournament tournament) {
        // 👉 เงื่อนไข A: รายการต้องยังไม่เริ่ม (UPCOMING) ไม่งั้น → 409
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (tournament.getStatus() != TournamentStatus.UPCOMING) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new BusinessException(
                "Teams can only join an upcoming tournament"
            );
        }

        // 👉 เงื่อนไข B: ต้องยังไม่มีแมตช์ในรายการนี้ (ตารางแข่ง/สายถูกสร้างแล้ว = ปิดรับสมัคร)
        //    existsByTournamentId = มีแมตช์ของรายการนี้อย่างน้อย 1 แมตช์หรือไม่
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (matchRepository.existsByTournamentId(tournament.getId())) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new BusinessException(
                "Cannot join a tournament after its schedule or bracket has been created"
            );
        }

        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (nextRule != null) {
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            nextRule.validate(team, tournament);
        }
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setNext(TeamJoinRule nextRule) {
        this.nextRule = nextRule;
    }
}```

## 6.5 `TeamNotAlreadyJoinedRule.java` — ด่าน 4: ทีมยังไม่สมัครซ้ำ

```java
package com.example.tournament.service.rule;

import org.springframework.stereotype.Component;

import com.example.tournament.domain.entity.Team;
import com.example.tournament.domain.entity.Tournament;
import com.example.tournament.exception.BusinessException;
import com.example.tournament.repository.TournamentTeamRepository;

@Component
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TeamNotAlreadyJoinedRule implements TeamJoinRule {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final TournamentTeamRepository tournamentTeamRepository;
    // 👉 ประกาศฟิลด์ที่คลาสใช้เก็บข้อมูลหรือ Dependency
    private TeamJoinRule nextRule;

    public TeamNotAlreadyJoinedRule(
            TournamentTeamRepository tournamentTeamRepository) {
        this.tournamentTeamRepository = tournamentTeamRepository;
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void validate(Team team, Tournament tournament) {
        // 👉 ตาราง tournament_teams ใช้ "คีย์คู่" (tournament_id, team_id)
        //    จึงสร้าง TournamentTeamId(รายการ, ทีม) แล้วถามว่ามีแถวนี้อยู่แล้วไหม
        boolean alreadyJoined =
                tournamentTeamRepository.existsById(
                        new com.example.tournament.domain.entity.TournamentTeamId(
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                tournament.getId(), team.getId()));

        // 👉 สมัครไว้แล้ว → 409
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (alreadyJoined) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new BusinessException(
                    "Team has already joined this tournament");
        }

        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (nextRule != null) {
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            nextRule.validate(team, tournament);
        }
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setNext(TeamJoinRule nextRule) {
        this.nextRule = nextRule;
    }
}```

## 6.6 `TeamGameMatchRule.java` — ด่าน 5: เกมของทีมต้องตรงกับเกมของรายการ

```java
package com.example.tournament.service.rule;

import org.springframework.stereotype.Component;

import com.example.tournament.domain.entity.Team;
import com.example.tournament.domain.entity.Tournament;
import com.example.tournament.exception.BusinessException;

@Component
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TeamGameMatchRule implements TeamJoinRule {

    private TeamJoinRule nextRule;   // 👉 ด่านนี้ไม่ต้องใช้ Repository จึงไม่มี constructor

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void validate(Team team, Tournament tournament) {
        // 👉 ไม่ผ่านถ้า: ทีมไม่มีเกม  หรือ รายการไม่มีเกม  หรือ id เกมไม่เท่ากัน
        //    (|| = "หรือ"  ถ้าข้อซ้ายจริง จะไม่ประเมินข้อขวาต่อ จึงไม่เกิด NullPointer)
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (team.getGame() == null || tournament.getGame() == null
                || !team.getGame().getId().equals(tournament.getGame().getId())) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new BusinessException(
                    "Team's game must match the tournament's game");
        }

        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (nextRule != null) {
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            nextRule.validate(team, tournament);
        }
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setNext(TeamJoinRule nextRule) {
        this.nextRule = nextRule;
    }
}```

## 6.7 `TeamMinimumPlayersRule.java` — ด่าน 6: ผู้เล่นต้องพอ

```java
package com.example.tournament.service.rule;

import org.springframework.stereotype.Component;

import com.example.tournament.domain.entity.Team;
import com.example.tournament.domain.entity.Tournament;
import com.example.tournament.exception.BusinessException;
import com.example.tournament.repository.PlayerRepository;

@Component
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TeamMinimumPlayersRule implements TeamJoinRule {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final PlayerRepository playerRepository;
    // 👉 ประกาศฟิลด์ที่คลาสใช้เก็บข้อมูลหรือ Dependency
    private TeamJoinRule nextRule;

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TeamMinimumPlayersRule(PlayerRepository playerRepository) {
        this.playerRepository = playerRepository;
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void validate(Team team, Tournament tournament) {
        // 👉 จำนวนผู้เล่นขั้นต่ำของเกมนั้น (ตาราง games คอลัมน์ min_players) อาจเป็น null ถ้าไม่ได้กำหนด
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Integer minimumPlayers = tournament.getGame().getMinPlayers();

        // 👉 ถ้าเกมไม่ได้กำหนดขั้นต่ำ ข้ามการตรวจนี้ไปเลย
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (minimumPlayers != null) {
            // 👉 นับผู้เล่นจริงในทีม (SELECT COUNT(*) ... WHERE team_id = ?)
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            long actualPlayers = playerRepository.countByTeamId(team.getId());

            // 👉 น้อยกว่าขั้นต่ำ → 409 เช่น ROV ต้อง 5 คน ทีมมี 4 คน
            // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
            if (actualPlayers < minimumPlayers) {
                // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
                throw new BusinessException(
                        "Team must have at least " + minimumPlayers
                                + " players to join this tournament");
            }
        }

        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (nextRule != null) {
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            nextRule.validate(team, tournament);
        }
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setNext(TeamJoinRule nextRule) {
        this.nextRule = nextRule;
    }
}```

## 6.8 `TeamPointsLimitRule.java` — ด่าน 7: รายการ POINTS ไม่เกิน 12 ทีม

```java
package com.example.tournament.service.rule;

import org.springframework.stereotype.Component;

import com.example.tournament.domain.entity.Team;
import com.example.tournament.domain.entity.Tournament;
import com.example.tournament.domain.enums.TournamentFormat;
import com.example.tournament.exception.BusinessException;
import com.example.tournament.repository.TournamentTeamRepository;

@Component
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TeamPointsLimitRule implements TeamJoinRule {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final TournamentTeamRepository tournamentTeamRepository;
    // 👉 ประกาศฟิลด์ที่คลาสใช้เก็บข้อมูลหรือ Dependency
    private TeamJoinRule nextRule;

    public TeamPointsLimitRule(
            TournamentTeamRepository tournamentTeamRepository) {
        this.tournamentTeamRepository = tournamentTeamRepository;
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void validate(Team team, Tournament tournament) {
        // 👉 ใช้เฉพาะรายการแบบ POINTS (Free Fire) เท่านั้น แบบแพ้คัดออกไม่จำกัดจำนวนทีมด้วยกฎนี้
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (tournament.getFormat() == TournamentFormat.POINTS) {
            // 👉 นับทีมที่สมัครอยู่ตอนนี้
            long currentTeams =
                    tournamentTeamRepository.countByTournamentId(
                            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                            tournament.getId());

            // 👉 ถ้ามี 12 ทีมแล้ว (เต็ม) ทีมที่ 13 สมัครไม่ได้ → 409
            //    (12 มาจากตารางคะแนนอันดับที่มี 12 อันดับ)
            // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
            if (currentTeams >= 12) {
                // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
                throw new BusinessException(
                        "POINTS tournaments cannot have more than 12 teams");
            }
        }

        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (nextRule != null) {
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            nextRule.validate(team, tournament);
        }
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setNext(TeamJoinRule nextRule) {
        this.nextRule = nextRule;
    }
}```

## 6.9 `TeamTournamentDateRule.java` — ด่าน 8: วันแข่งต้องไม่ทับกัน

```java
package com.example.tournament.service.rule;

import org.springframework.stereotype.Component;

import com.example.tournament.domain.entity.Team;
import com.example.tournament.domain.entity.Tournament;
import com.example.tournament.exception.BusinessException;
import com.example.tournament.repository.TournamentTeamRepository;

@Component
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TeamTournamentDateRule implements TeamJoinRule {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final TournamentTeamRepository tournamentTeamRepository;
    // 👉 ประกาศฟิลด์ที่คลาสใช้เก็บข้อมูลหรือ Dependency
    private TeamJoinRule nextRule;

    public TeamTournamentDateRule(
            TournamentTeamRepository tournamentTeamRepository) {
        this.tournamentTeamRepository = tournamentTeamRepository;
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void validate(Team team, Tournament tournament) {
        // 👉 ถามฐานข้อมูลว่า "ทีมนี้อยู่ในรายการอื่นกี่รายการ ที่ช่วงวันแข่งชนกับรายการนี้"
        long overlappingTournaments =
                tournamentTeamRepository.countOverlappingTournaments(
                        team.getId(),
                        tournament.getId(),
                        tournament.getStartDate(),
                        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                        tournament.getEndDate());

        // 👉 มีอย่างน้อย 1 รายการที่ชน → 409
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (overlappingTournaments > 0) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new BusinessException(
                    "Team already joined another tournament with overlapping dates");
        }

        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (nextRule != null) {
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            nextRule.validate(team, tournament);
        }
    }

    @Override
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void setNext(TeamJoinRule nextRule) {
        this.nextRule = nextRule;
    }
}```

### 6.9.1 query นับรายการที่วันทับกัน (`TournamentTeamRepository.java`)

```java
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public interface TournamentTeamRepository extends JpaRepository<TournamentTeam, TournamentTeamId> {

    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    List<TournamentTeam> findByTournamentId(Long tournamentId);

    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    List<TournamentTeam> findByTournamentIdOrderByJoinedAtAsc(Long tournamentId);

    long countByTournamentId(Long tournamentId);   // 👉 นับทีมในรายการ (ใช้ในด่าน 7)

    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    boolean existsByTeam_Id(Long teamId);

    // 👉 @Query = เขียนคำสั่งค้นหาเอง (ภาษา JPQL คล้าย SQL แต่ใช้ชื่อคลาส/ฟิลด์ Java)
    //    คำอธิบายแต่ละบรรทัดของ query อยู่ในตารางใต้บล็อกโค้ดนี้
    @Query("""
            SELECT COUNT(tt)
            FROM TournamentTeam tt
            WHERE tt.team.id = :teamId
              AND tt.tournament.id <> :tournamentId
              AND tt.tournament.startDate <= :endDate
              AND tt.tournament.endDate >= :startDate
            """)
    long countOverlappingTournaments(
            @Param("teamId") Long teamId,
            @Param("tournamentId") Long tournamentId,
            @Param("startDate") LocalDate startDate,
            // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
            @Param("endDate") LocalDate endDate);
}```

อธิบาย query ทีละบรรทัด

| บรรทัด | ความหมาย |
|---|---|
| `SELECT COUNT(tt) FROM TournamentTeam tt` | นับจำนวนแถวการสมัครทีม |
| `tt.team.id = :teamId` | เฉพาะของทีมนี้ |
| `tt.tournament.id <> :tournamentId` | ที่ไม่ใช่รายการที่กำลังจะสมัคร (`<>` = ไม่เท่ากับ) |
| `tt.tournament.startDate <= :endDate` | รายการเดิมเริ่มก่อนหรือวันเดียวกับวันที่รายการใหม่จบ |
| `tt.tournament.endDate >= :startDate` | รายการเดิมจบหลังหรือวันเดียวกับวันที่รายการใหม่เริ่ม |

**ทำไมสูตร `เดิมเริ่ม <= ใหม่จบ AND เดิมจบ >= ใหม่เริ่ม` ถึงบอกว่าช่วงวันทับกัน:** ช่วงสองช่วงจะ "ไม่ทับ" ก็ต่อเมื่อช่วงหนึ่งจบก่อนอีกช่วงเริ่ม ปฏิเสธข้อความนั้นก็ได้สูตรข้างต้น ตัวอย่าง

| รายการเดิม | รายการใหม่ (1–10 พ.ย.) | ทับไหม |
|---|---|---|
| 5–8 พ.ย. | 1–10 พ.ย. | ทับ |
| 10–15 พ.ย. | 1–10 พ.ย. | ทับ (วันที่ 10 ซ้ำกัน) |
| 11–15 พ.ย. | 1–10 พ.ย. | ไม่ทับ |
| 20–25 ต.ค. | 1–10 พ.ย. | ไม่ทับ |

## 6.10 `TeamJoinRuleChain.java` — ต่อด่านทั้งหมดเป็นโซ่

```java
package com.example.tournament.service.rule;

import org.springframework.stereotype.Component;

import com.example.tournament.domain.entity.Team;
import com.example.tournament.domain.entity.Tournament;

@Component
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TeamJoinRuleChain {

    private final TeamExistsRule teamExistsRule;   // 👉 เก็บ "ด่านแรก" ไว้ ตอนใช้งานเริ่มจากด่านนี้

    // 👉 Spring ฉีดทั้ง 8 ด่านเข้ามาทาง constructor
    public TeamJoinRuleChain(
            TeamExistsRule teamExistsRule,
            TournamentExistsRule tournamentExistsRule,
            TournamentStatusRule tournamentStatusRule,
            TeamNotAlreadyJoinedRule teamNotAlreadyJoinedRule,
            TeamGameMatchRule teamGameMatchRule,
            TeamMinimumPlayersRule teamMinimumPlayersRule,
            TeamPointsLimitRule teamPointsLimitRule,
            TeamTournamentDateRule teamTournamentDateRule) {

        this.teamExistsRule = teamExistsRule;

        // 👉 ต่อโซ่: ด่านหน้า.setNext(ด่านหลัง)  ลำดับตรงนี้คือ "ลำดับการตรวจ"
        teamExistsRule.setNext(tournamentExistsRule);                 // 1 → 2
        tournamentExistsRule.setNext(tournamentStatusRule);           // 2 → 3
        tournamentStatusRule.setNext(teamNotAlreadyJoinedRule);       // 3 → 4
        teamNotAlreadyJoinedRule.setNext(teamGameMatchRule);          // 4 → 5
        teamGameMatchRule.setNext(teamMinimumPlayersRule);            // 5 → 6
        teamMinimumPlayersRule.setNext(teamPointsLimitRule);          // 6 → 7
        teamPointsLimitRule.setNext(teamTournamentDateRule);          // 7 → 8 (ด่านสุดท้าย nextRule = null)
    }

    // 👉 ผู้ใช้โซ่เรียกเมธอดเดียวนี้ ระบบจะวิ่งผ่านทุกด่านเอง
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public void validate(Team team, Tournament tournament) {
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        teamExistsRule.validate(team, tournament);
    }
}```

## 6.11 ที่ที่เรียกใช้โซ่: `TournamentTeamServiceImpl.addTeam` และ Controller

```java
// ===== ส่วนของ TournamentTeamServiceImpl.java =====
@Override
// 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
public void addTeam(Long tournamentId, Long teamId) {
    // ตรวจสอบทีมก่อน เพื่อรักษาลำดับการตรวจสอบของกฎเดิม
    // 👉 โหลดทีมและรายการจากฐานข้อมูล (ไม่เจอ → 404)
    // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
    Team team = teamRepository.findById(teamId)
            // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
            .orElseThrow(() ->
                    // 👉 แจ้งว่าหาข้อมูลที่ต้องการไม่พบ
                    new ResourceNotFoundException("Team not found"));

    // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
    Tournament tournament = tournamentRepository.findById(tournamentId)
            // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
            .orElseThrow(() ->
                    // 👉 แจ้งว่าหาข้อมูลที่ต้องการไม่พบ
                    new ResourceNotFoundException("Tournament not found"));

    // ตรวจสอบเงื่อนไขการสมัครทีมทั้งหมด
    // 👉 ★ จุดที่เรียกโซ่ 8 ด่าน ★ ไม่ผ่านด่านไหนจะ throw ออกมาตรงนี้ และโค้ดข้างล่างจะไม่ทำงาน
    // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
    teamJoinRuleChain.validate(team, tournament);

    // บันทึกข้อมูลทีม ณ วันที่สมัคร และสร้าง snapshot ของผู้เล่น
    // 👉 ผ่านทุกด่านแล้วจึงบันทึกการสมัคร พร้อมเก็บ "ชื่อ/คำอธิบาย/โลโก้ของทีม ณ ตอนนี้" ไว้
    //    (ถ้าทีมเปลี่ยนชื่อภายหลัง ประวัติของรายการนี้ยังแสดงชื่อเดิม)
    // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
    TournamentTeam tournamentTeam = new TournamentTeam();
    // 👉 กำหนดค่าฟิลด์ของ object นี้
    tournamentTeam.setId(new TournamentTeamId(tournamentId, teamId));
    // 👉 กำหนดค่าฟิลด์ของ object นี้
    tournamentTeam.setTournament(tournament);
    // 👉 กำหนดค่าฟิลด์ของ object นี้
    tournamentTeam.setTeam(team);
    // 👉 กำหนดค่าฟิลด์ของ object นี้
    tournamentTeam.setJoinedAt(LocalDateTime.now());
    // 👉 กำหนดค่าฟิลด์ของ object นี้
    tournamentTeam.setTeamName(team.getName());
    // 👉 กำหนดค่าฟิลด์ของ object นี้
    tournamentTeam.setTeamDescription(team.getDescription());
    // 👉 กำหนดค่าฟิลด์ของ object นี้
    tournamentTeam.setTeamLogoUrl(team.getLogoUrl());

    // 👉 บันทึกข้อมูลและสั่งให้ JPA ส่งการเปลี่ยนแปลงไปฐานข้อมูลทันที
    tournamentTeamRepository.saveAndFlush(tournamentTeam);
    rosters.capturePlayers(tournamentId, teamId);   // 👉 บันทึกรายชื่อผู้เล่นของทีม ณ วันที่สมัคร (snapshot)
}

@Override
// 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
public void removeTeam(Long tournamentId, Long teamId) {
    // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
    TournamentTeamId id = new TournamentTeamId(tournamentId, teamId);

    // ค้นหารายการสมัครทีมก่อน แล้วตรวจสอบสถานะจากรายการนั้น
    // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
    TournamentTeam tournamentTeam = tournamentTeamRepository.findById(id)
            // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
            .orElseThrow(() ->
                    // 👉 แจ้งว่าหาข้อมูลที่ต้องการไม่พบ
                    new ResourceNotFoundException(
                            "Team is not registered in this tournament"));

    // 👉 ถอนทีมได้เฉพาะตอนรายการยังไม่เริ่ม
    // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
    if (tournamentTeam.getTournament().getStatus()
            != TournamentStatus.UPCOMING) {
        // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
        throw new BusinessException(
                "Cannot remove a team after the tournament has started");
    }

    // 👉 ลบข้อมูลที่ระบุออกจากฐานข้อมูล
    tournamentTeamRepository.delete(tournamentTeam);
}```

```java
// ===== TournamentTeamController.java =====
@RestController
@RequestMapping("/api/v1/tournaments/{tournamentId}/teams")
// 👉 ประกาศชนิดข้อมูลหลักของไฟล์นี้
public class TournamentTeamController {

    // 👉 เก็บ Dependency หรือข้อมูลของคลาส โดย final ทำให้กำหนดค่าได้ครั้งเดียว
    private final TournamentTeamService tournamentTeams;

    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public TournamentTeamController(TournamentTeamService tournamentTeams) {
        this.tournamentTeams = tournamentTeams;
    }

    // ผ่านกฎ 8 ข้อ (Chain of Responsibility) แล้วบันทึก → 204, ไม่ผ่าน → 400 / 404 / 409
    // 👉 POST /api/v1/tournaments/5/teams   body: { "teamId": 3 }
    @PostMapping
    public ResponseEntity<Void> add(@PathVariable Long tournamentId,
            @Valid @RequestBody AddTournamentTeamRequest request) {
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        tournamentTeams.addTeam(tournamentId, request.teamId());
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return ResponseEntity.noContent().build();
    }

    // ถอนได้เฉพาะรายการที่ยังไม่เริ่ม (UPCOMING) → 204, ไม่พบ → 404, เริ่มแล้ว → 409
    // 👉 DELETE /api/v1/tournaments/5/teams/3
    @DeleteMapping("/{teamId}")
    // 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
    public ResponseEntity<Void> remove(@PathVariable Long tournamentId, @PathVariable Long teamId) {
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        tournamentTeams.removeTeam(tournamentId, teamId);
        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
        return ResponseEntity.noContent().build();
    }
}```

## 6.12 ตัวอย่างการเดินของโซ่

**กรณี A: สมัครสำเร็จ** ทีม 3 (ROV, ผู้เล่น 5 คน) สมัครรายการ 5 (ROV, UPCOMING, แบบแพ้คัดออก, 1–10 พ.ย.) ผ่านด่าน 1→8 ครบ → บันทึก → ตอบ 204

**กรณี B: ผู้เล่นไม่พอ** ทีมมีผู้เล่น 4 คน แต่ ROV ต้อง 5 → ผ่านด่าน 1–5 → **ตกด่าน 6** → 409 `Team must have at least 5 players...` (ด่าน 7, 8 ไม่ถูกเรียก)

**กรณี C: วันชน** ทีมอยู่รายการอื่น 8–12 พ.ย. แล้วสมัครรายการ 1–10 พ.ย. → ผ่านด่าน 1–7 → **ตกด่าน 8** → 409 `Team already joined another tournament with overlapping dates`

**ข้อดี:** อยากเพิ่มกฎใหม่ (เช่น ห้ามทีมที่ถูกแบน) แค่สร้างคลาสใหม่ที่ `implements TeamJoinRule` แล้วเพิ่มบรรทัดต่อโซ่ใน `TeamJoinRuleChain` โดยไม่ต้องแก้ด่านเดิมเลย

---

# 7. Unit Test ทั้งหมด

Commit: `test: add tournament validation tests`, `test: add team game match rule tests`, `test: cover remaining team join rules`

> เพื่อให้อ่านง่าย ผมตัดบรรทัด `import` ของไฟล์เทสต์ออก (ใช้ชื่อคลาสตรงกับที่ใช้ในโค้ด) ถ้าต้องการให้ครบทุกบรรทัด ดูในไฟล์จริงของ repo

## 7.0 อ่านเทสต์ยังไง

เทสต์ทุกอันมี 3 ขั้น

1. **Arrange (เตรียม):** สร้างข้อมูลทดสอบ + ตั้งค่า mock (ของปลอม) ให้ตอบตามที่ต้องการ
2. **Act (ลงมือ):** เรียกโค้ดจริงที่ต้องการทดสอบ
3. **Assert (ตรวจ):** ตรวจว่าผลตรงตามคาด (`assertEquals`, `assertThrows`, `verify` ...)

---

## 7.1 `TournamentServiceImplTest.java` — ทดสอบ `TournamentServiceImpl` (โค้ดเต็ม)

```java
// 👉 @ExtendWith(MockitoExtension.class) = ให้ JUnit ใช้ Mockito ช่วยสร้าง mock ที่ติด @Mock
@ExtendWith(MockitoExtension.class)
class TournamentServiceImplTest {

        // 👉 @Mock = สร้าง Repository "ปลอม" (ไม่ต่อฐานข้อมูล ตอบตามที่เราสั่งด้วย when(...))
        @Mock
        TournamentRepository tournaments;

        @Mock
        GameRepository games;

        @Mock
        TournamentPlacementPointRepository tournamentPlacementPoints;

        TournamentServiceImpl service;   // 👉 ตัวที่ถูกทดสอบ (ของจริง แต่ใช้ Repository ปลอม)

        Game freeFire;   // 👉 เกมจำลอง: Free Fire (รหัส FREE_FIRE)
        Game rov;        // 👉 เกมจำลอง: ROV

        // 👉 @BeforeEach = รันก่อน "ทุกเทสต์" เพื่อสร้างของใหม่ทุกครั้ง เทสต์จะได้ไม่กวนกัน
        @BeforeEach
        void setUp() {
                // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
                service = new TournamentServiceImpl(
                                tournaments,
                                games,
                                tournamentPlacementPoints);

                freeFire = game(1L, "FREE_FIRE", 4);   // 👉 id=1, ขั้นต่ำ 4 คน
                rov = game(2L, "ROV", 5);              // 👉 id=2, ขั้นต่ำ 5 คน
        }

        // ---------- กลุ่ม 1: เคส "ต้องปฏิเสธ" (คาดว่าต้อง throw) ----------

        // 👉 เคส: วันจบก่อนวันเริ่ม (19 ต.ค. < 20 ต.ค.) ต้อง throw ValidationException
        @Test
        void rejectsWhenEndDateIsBeforeStartDate() {
                // Arrange: สร้างคำขอที่ผิดกฎ (เริ่ม 20 ต.ค. แต่จบ 19 ต.ค.)
                TournamentRequest request = request(
                                "Test Tournament",
                                2L,
                                TournamentFormat.SINGLE_ELIMINATION,
                                null,
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                LocalDate.of(2026, 10, 19));

                // 👉 สั่ง mock: ชื่อนี้ยังไม่ซ้ำ / เกม id 2 มีอยู่ (คือ ROV) เพื่อให้โค้ดเดินไปถึงขั้นตรวจวันที่
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                when(tournaments.existsByName("Test Tournament")).thenReturn(false);
                // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
                when(games.findById(2L)).thenReturn(Optional.of(rov));

                // Act + Assert: เรียก create แล้วต้อง throw ValidationException
                assertThrows(
                                ValidationException.class,
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                () -> service.create(request));
        }

        // 👉 เคส: ชื่อซ้ำ ต้อง throw BusinessException (409)
        @Test
        void rejectsDuplicateTournamentName() {
                TournamentRequest request = request(
                                "Duplicate Tournament",
                                2L,
                                TournamentFormat.SINGLE_ELIMINATION,
                                null,
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                LocalDate.of(2026, 10, 21));

                // 👉 สั่ง mock ให้ตอบว่า "ชื่อนี้มีอยู่แล้ว" (true)
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                when(tournaments.existsByName("Duplicate Tournament")).thenReturn(true);

                assertThrows(
                                BusinessException.class,
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                () -> service.create(request));
        }

        // 👉 เคส: Free Fire แต่เลือก SINGLE_ELIMINATION ต้องปฏิเสธ (กฎ 4)
        @Test
        void rejectsFreeFireWithSingleElimination() {
                TournamentRequest request = request(
                                "Free Fire Tournament",
                                1L,
                                TournamentFormat.SINGLE_ELIMINATION,
                                null,
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                LocalDate.of(2026, 10, 21));

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                when(tournaments.existsByName("Free Fire Tournament")).thenReturn(false);
                // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
                when(games.findById(1L)).thenReturn(Optional.of(freeFire));

                assertThrows(
                                ValidationException.class,
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                () -> service.create(request));
        }

        // 👉 เคส: เกมที่ไม่ใช่ Free Fire (ROV) แต่เลือก POINTS ต้องปฏิเสธ (กฎ 5)
        @Test
        void rejectsNonFreeFireWithPointsFormat() {
                TournamentRequest request = request(
                                "ROV Points Tournament",
                                2L,
                                TournamentFormat.POINTS,
                                (short) 10,
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                LocalDate.of(2026, 10, 21));

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                when(tournaments.existsByName("ROV Points Tournament")).thenReturn(false);
                // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
                when(games.findById(2L)).thenReturn(Optional.of(rov));

                assertThrows(
                                ValidationException.class,
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                () -> service.create(request));
        }

        // 👉 เคส: POINTS แต่ไม่ใส่ totalGames (null) ต้องปฏิเสธ (กฎ 6)
        @Test
        void rejectsPointsWithoutTotalGames() {
                TournamentRequest request = request(
                                "Free Fire Tournament",
                                1L,
                                TournamentFormat.POINTS,
                                null,
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                LocalDate.of(2026, 10, 21));

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                when(tournaments.existsByName("Free Fire Tournament")).thenReturn(false);
                // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
                when(games.findById(1L)).thenReturn(Optional.of(freeFire));

                assertThrows(
                                ValidationException.class,
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                () -> service.create(request));
        }

        // 👉 เคส: แพ้คัดออกแต่ใส่ totalGames = 10 ต้องปฏิเสธ (กฎ 7)
        @Test
        void rejectsEliminationWithTotalGames() {
                TournamentRequest request = request(
                                "ROV Tournament",
                                2L,
                                TournamentFormat.SINGLE_ELIMINATION,
                                (short) 10,
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                LocalDate.of(2026, 10, 21));

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                when(tournaments.existsByName("ROV Tournament")).thenReturn(false);
                // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
                when(games.findById(2L)).thenReturn(Optional.of(rov));

                assertThrows(
                                ValidationException.class,
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                () -> service.create(request));
        }

        // ---------- กลุ่ม 2: เคส "ต้องสำเร็จ" ----------

        // 👉 เคส: สร้างรายการ Free Fire แบบ POINTS ถูกต้อง ต้องสำเร็จ
        @Test
        void createsFreeFirePointsTournamentSuccessfully() {
                TournamentRequest request = request(
                                "Free Fire Tournament",
                                1L,
                                TournamentFormat.POINTS,
                                (short) 10,
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                LocalDate.of(2026, 10, 21));

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                when(tournaments.existsByName("Free Fire Tournament")).thenReturn(false);
                // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
                when(games.findById(1L)).thenReturn(Optional.of(freeFire));

                // 👉 สั่ง mock ของ saveAndFlush: ตอนบันทึก ให้ "ใส่ id = 1 ให้ object แล้วคืนกลับ"
                //    (จำลองสิ่งที่ฐานข้อมูลจริงทำ คือสร้าง id ให้)
                //    thenAnswer = ตอบด้วยโค้ดที่เราเขียนเอง  invocation.getArgument(0) = อาร์กิวเมนต์ตัวแรกที่ถูกส่งเข้ามา
                // 👉 บันทึกข้อมูลและสั่งให้ JPA ส่งการเปลี่ยนแปลงไปฐานข้อมูลทันที
                when(tournaments.saveAndFlush(any(Tournament.class)))
                                .thenAnswer(invocation -> {
                                        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
                                        Tournament tournament = invocation.getArgument(0);
                                        // 👉 กำหนดค่าฟิลด์ของ object นี้
                                        tournament.setId(1L);
                                        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
                                        return tournament;
                                });

                var response = service.create(request);   // 👉 Act: สร้างจริง (var = ให้ Java เดาชนิดเอง)

                // 👉 Assert: ตรวจค่าที่ตอบกลับ
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals(1L, response.id());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals("Free Fire Tournament", response.name());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals(TournamentFormat.POINTS, response.format());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals((short) 10, response.totalGames());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals("FREE_FIRE", freeFire.getCode());

                // 👉 ตรวจว่า saveAndFlush ถูกเรียกจริง 1 ครั้ง (any(...) = ด้วย Tournament ตัวไหนก็ได้)
                // 👉 บันทึกข้อมูลและสั่งให้ JPA ส่งการเปลี่ยนแปลงไปฐานข้อมูลทันที
                verify(tournaments).saveAndFlush(any(Tournament.class));
        }

        // 👉 เคส: สร้างรายการแพ้คัดออก (ROV) ถูกต้อง ต้องสำเร็จ และ totalGames ต้องเป็น null
        @Test
        void createsEliminationTournamentSuccessfully() {
                TournamentRequest request = request(
                                "ROV Tournament",
                                2L,
                                TournamentFormat.SINGLE_ELIMINATION,
                                null,
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                LocalDate.of(2026, 10, 21));

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                when(tournaments.existsByName("ROV Tournament")).thenReturn(false);
                // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
                when(games.findById(2L)).thenReturn(Optional.of(rov));

                // 👉 บันทึกข้อมูลและสั่งให้ JPA ส่งการเปลี่ยนแปลงไปฐานข้อมูลทันที
                when(tournaments.saveAndFlush(any(Tournament.class)))
                                .thenAnswer(invocation -> {
                                        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
                                        Tournament tournament = invocation.getArgument(0);
                                        // 👉 กำหนดค่าฟิลด์ของ object นี้
                                        tournament.setId(2L);
                                        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
                                        return tournament;
                                });

                // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
                var response = service.create(request);

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals(2L, response.id());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals("ROV Tournament", response.name());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals(TournamentFormat.SINGLE_ELIMINATION, response.format());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals(null, response.totalGames());

                // 👉 บันทึกข้อมูลและสั่งให้ JPA ส่งการเปลี่ยนแปลงไปฐานข้อมูลทันที
                verify(tournaments).saveAndFlush(any(Tournament.class));
        }

        // ---------- กลุ่ม 3: คะแนนอันดับ FFWS ----------

        // 👉 เคส: สร้างรายการ POINTS แล้วระบบต้อง "บันทึกคะแนนอันดับ 12 แถว" ให้เอง
        @Test
        void createsFfwsPlacementPointsAutomatically() {
                TournamentRequest request = request(
                                "Free Fire FFWS Tournament",
                                1L,
                                TournamentFormat.POINTS,
                                (short) 10,
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                LocalDate.of(2026, 10, 21));

                when(tournaments.existsByName("Free Fire FFWS Tournament"))
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                .thenReturn(false);

                // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
                when(games.findById(1L))
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                .thenReturn(Optional.of(freeFire));

                // 👉 บันทึกข้อมูลและสั่งให้ JPA ส่งการเปลี่ยนแปลงไปฐานข้อมูลทันที
                when(tournaments.saveAndFlush(any(Tournament.class)))
                                .thenAnswer(invocation -> {
                                        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
                                        Tournament tournament = invocation.getArgument(0);
                                        // 👉 กำหนดค่าฟิลด์ของ object นี้
                                        tournament.setId(1L);
                                        // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
                                        return tournament;
                                });

                // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
                var response = service.create(request);

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals(1L, response.id());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals(TournamentFormat.POINTS, response.format());

                // 👉 ตรวจว่า save() ของ Repository คะแนนอันดับถูกเรียก "พอดี 12 ครั้ง" (อันดับ 1–12)
                verify(tournamentPlacementPoints, times(12))
                                // 👉 บันทึกข้อมูลลงฐานข้อมูล
                                .save(any(TournamentPlacementPoint.class));
        }

        // 👉 เคส: API ดูคะแนน ต้องคืนเรียงอันดับ 1,2,3 และค่าถูกต้อง (12, 9, 8)
        @Test
        void returnsFfwsPlacementPointsInAscendingOrder() {

                // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
                Tournament tournament = new Tournament();
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                tournament.setId(1L);

                // 👉 เตรียมข้อมูลปลอม 3 แถว (อันดับ 1=12, 2=9, 3=8) โดยใช้เมธอดช่วย placementPoint(...) ด้านล่าง
                // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
                TournamentPlacementPoint first = placementPoint(tournament, (short) 1, (short) 12);

                // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
                TournamentPlacementPoint second = placementPoint(tournament, (short) 2, (short) 9);

                // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
                TournamentPlacementPoint third = placementPoint(tournament, (short) 3, (short) 8);

                // 👉 สั่ง mock: รายการ id 1 มีอยู่จริง และ Repository คืน 3 แถวนี้
                // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
                when(tournaments.findById(1L))
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                .thenReturn(Optional.of(tournament));

                when(tournamentPlacementPoints
                                .findByTournamentIdOrderByPlacementAsc(1L))
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                .thenReturn(List.of(first, second, third));

                // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
                List<PlacementPointResponse> result = service.getPlacementPoints(1L);

                // 👉 ตรวจ: มี 3 แถว และเรียงตามลำดับ พร้อมคะแนนถูกต้อง
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals(3, result.size());

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals((short) 1, result.get(0).placement());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals((short) 12, result.get(0).points());

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals((short) 2, result.get(1).placement());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals((short) 9, result.get(1).points());

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals((short) 3, result.get(2).placement());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals((short) 8, result.get(2).points());

                // 👉 ตรวจว่า Service เรียกเมธอดที่ "เรียงลำดับ" จริง (ไม่ใช่เมธอดอื่น)
                verify(tournamentPlacementPoints)
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                .findByTournamentIdOrderByPlacementAsc(1L);
        }

        // 👉 เคส: ขอดูคะแนนของรายการที่ไม่มีอยู่ (id 999) ต้อง throw 404
        @Test
        void rejectsPlacementPointsWhenTournamentDoesNotExist() {

                // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
                when(tournaments.findById(999L))
                                .thenReturn(Optional.empty());   // 👉 Optional.empty() = "ไม่เจอ"

                assertThrows(
                                ResourceNotFoundException.class,
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                () -> service.getPlacementPoints(999L));
        }

        // ---------- กลุ่ม 4: ค้นหาตามชื่อ ----------

        // 👉 เคส: ค้นด้วย "fire" ต้องได้ "Free Fire Championship"
        @Test
        void searchesTournamentsByPartialNameIgnoringCase() {
                // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
                Tournament freeFireTournament = new Tournament();
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                freeFireTournament.setId(1L);
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                freeFireTournament.setName("Free Fire Championship");

                // Set the game because the response includes the game ID.
                // 👉 ต้องตั้งเกมให้ Entity เพราะ toResponse เรียก getGame().getId()
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                freeFireTournament.setGame(freeFire);

                when(tournaments.findByNameContainingIgnoreCase("fire"))
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                .thenReturn(List.of(freeFireTournament));

                // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
                List<TournamentResponse> result = service.searchByName("fire");

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals(1, result.size());
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals("Free Fire Championship", result.get(0).name());

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                verify(tournaments).findByNameContainingIgnoreCase("fire");
        }

        // 👉 เคส: ค้นด้วย "  ROV  " (มีช่องว่าง) Service ต้องตัดช่องว่างก่อนส่งให้ Repository ("ROV")
        @Test
        void trimsSearchKeywordBeforeSearching() {
                when(tournaments.findByNameContainingIgnoreCase("ROV"))
                                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                                .thenReturn(List.of());

                // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
                List<TournamentResponse> result = service.searchByName("  ROV  ");

                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                assertEquals(0, result.size());

                // 👉 ตรวจว่า Repository ถูกเรียกด้วย "ROV" (ไม่มีช่องว่าง) = พิสูจน์ว่า trim ทำงาน
                // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
                verify(tournaments).findByNameContainingIgnoreCase("ROV");
        }

        // ---------- เมธอดช่วยสร้างข้อมูลทดสอบ (ไม่ใช่เทสต์) ----------

        // 👉 สร้างแถวคะแนนอันดับ 1 แถว
        private static TournamentPlacementPoint placementPoint(
                        Tournament tournament,
                        Short placement,
                        Short points) {

                // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
                TournamentPlacementPoint placementPoint = new TournamentPlacementPoint();

                // 👉 กำหนดค่าฟิลด์ของ object นี้
                placementPoint.setTournament(tournament);
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                placementPoint.setPlacement(placement);
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                placementPoint.setPoints(points);

                // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
                return placementPoint;
        }

        // 👉 สร้างคำขอสร้างรายการ (ลดการเขียนซ้ำในทุกเทสต์)
        private static TournamentRequest request(
                        String name,
                        Long gameId,
                        TournamentFormat format,
                        Short totalGames,
                        LocalDate startDate,
                        LocalDate endDate) {

                // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
                TournamentRequest request = new TournamentRequest();

                // 👉 กำหนดค่าฟิลด์ของ object นี้
                request.setName(name);
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                request.setGameId(gameId);
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                request.setFormat(format);
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                request.setTotalGames(totalGames);
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                request.setStartDate(startDate);
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                request.setEndDate(endDate);

                // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
                return request;
        }

        // 👉 สร้างเกมจำลอง
        private static Game game(Long id, String code, int minPlayers) {
                // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
                Game game = new Game();

                // 👉 กำหนดค่าฟิลด์ของ object นี้
                game.setId(id);
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                game.setCode(code);
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                game.setMinPlayers(minPlayers);
                // 👉 กำหนดค่าฟิลด์ของ object นี้
                game.setIsActive(true);

                // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
                return game;
        }
}```

**สรุปว่าเทสต์นี้ครอบคลุมอะไร**

| เทสต์ | ตรวจกฎ/ฟีเจอร์ |
|---|---|
| `rejectsWhenEndDateIsBeforeStartDate` | วันจบก่อนวันเริ่ม → 400 |
| `rejectsDuplicateTournamentName` | ชื่อซ้ำ → 409 |
| `rejectsFreeFireWithSingleElimination` | Free Fire ต้อง POINTS |
| `rejectsNonFreeFireWithPointsFormat` | เกมอื่นห้าม POINTS |
| `rejectsPointsWithoutTotalGames` | POINTS ต้องมี totalGames |
| `rejectsEliminationWithTotalGames` | แพ้คัดออกห้ามมี totalGames |
| `createsFreeFirePointsTournamentSuccessfully` | สร้างแบบ POINTS สำเร็จ |
| `createsEliminationTournamentSuccessfully` | สร้างแบบแพ้คัดออกสำเร็จ |
| `createsFfwsPlacementPointsAutomatically` | สร้างคะแนน 12 แถวอัตโนมัติ |
| `returnsFfwsPlacementPointsInAscendingOrder` | API คะแนนเรียงถูกต้อง |
| `rejectsPlacementPointsWhenTournamentDoesNotExist` | ไม่พบรายการ → 404 |
| `searchesTournamentsByPartialNameIgnoringCase` | ค้นชื่อบางส่วน |
| `trimsSearchKeywordBeforeSearching` | ตัดช่องว่างก่อนค้น |

> **ที่ยังไม่มีเทสต์ในไฟล์นี้:** `update`, `delete`, `getByStatus`, `searchByNameAndStatus` (ถ้าอยากเพิ่ม ใช้รูปแบบเดียวกับเทสต์ข้างบน)

---

## 7.2 เทสต์ของกฎลงทะเบียนทีม (`service/rule/*Test.java`)

เทสต์กลุ่มนี้ใช้ `mock(...)` โดยตรง (ไม่ใช้ `@Mock`) รูปแบบเหมือนกันทุกไฟล์: เตรียม mock → สร้างกฎ → ตรวจว่า "ผ่าน" (`assertDoesNotThrow`) หรือ "ไม่ผ่าน" (`assertThrows`)

### `TeamExistsRuleTest` (ด่าน 1)

```java
class TeamExistsRuleTest {

    // 👉 เคส: ทีมมีอยู่ → ผ่าน
    @Test
    void shouldPassWhenTeamExists() {
        TeamRepository repo = mock(TeamRepository.class);   // 👉 Repository ปลอม
        Team team = mock(Team.class);                       // 👉 ทีมปลอม
        Tournament tournament = mock(Tournament.class);     // 👉 รายการปลอม

        when(team.getId()).thenReturn(10L);        // 👉 ทีมปลอมนี้มี id = 10
        when(repo.existsById(10L)).thenReturn(true);   // 👉 ฐานข้อมูลปลอมตอบว่า "มีทีม id 10"

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamExistsRule rule = new TeamExistsRule(repo);

        assertDoesNotThrow(() -> rule.validate(team, tournament));   // 👉 ต้องไม่ throw
    }

    // 👉 เคส: ทีมไม่มี → throw 404
    @Test
    void shouldThrowWhenTeamDoesNotExist() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        TeamRepository repo = mock(TeamRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(team.getId()).thenReturn(10L);
        when(repo.existsById(10L)).thenReturn(false);   // 👉 ตอบว่า "ไม่มี"

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamExistsRule rule = new TeamExistsRule(repo);

        assertThrows(
            ResourceNotFoundException.class,
            () -> rule.validate(team, tournament)
        );
    }
}```

### `TournamentExistsRuleTest` (ด่าน 2)

```java
class TournamentExistsRuleTest {

    // 👉 รายการมีอยู่ → ผ่าน
    @Test
    void shouldPassWhenTournamentExists() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        TournamentRepository repo = mock(TournamentRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getId()).thenReturn(20L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(repo.existsById(20L)).thenReturn(true);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TournamentExistsRule rule = new TournamentExistsRule(repo);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 รายการไม่มี → throw 404
    @Test
    void shouldThrowWhenTournamentDoesNotExist() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        TournamentRepository repo = mock(TournamentRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getId()).thenReturn(20L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(repo.existsById(20L)).thenReturn(false);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TournamentExistsRule rule = new TournamentExistsRule(repo);

        assertThrows(
            ResourceNotFoundException.class,
            () -> rule.validate(team, tournament)
        );
    }
}```

### `TournamentStatusRuleTest` (ด่าน 3)

```java
class TournamentStatusRuleTest {

    // 👉 UPCOMING และยังไม่มีแมตช์ → ผ่าน
    @Test
    void shouldPassWhenUpcomingAndNoMatchesExist() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        MatchRepository repo = mock(MatchRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getId()).thenReturn(20L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getStatus()).thenReturn(TournamentStatus.UPCOMING);
        when(repo.existsByTournamentId(20L)).thenReturn(false);   // 👉 ยังไม่มีแมตช์

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TournamentStatusRule rule = new TournamentStatusRule(repo);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 สถานะไม่ใช่ UPCOMING (ที่นี่จำลองด้วย null) → throw และต้องไม่แตะ repo เลย (ล้มก่อนถึงขั้นเช็คแมตช์)
    @Test
    void shouldThrowWhenTournamentIsNotUpcoming() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        MatchRepository repo = mock(MatchRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getStatus()).thenReturn(null);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TournamentStatusRule rule = new TournamentStatusRule(repo);

        assertThrows(
            BusinessException.class,
            () -> rule.validate(team, tournament)
        );

        verifyNoInteractions(repo);   // 👉 ตรวจว่า repo ไม่ถูกเรียกเลยสักครั้ง
    }

    // 👉 UPCOMING แต่มีแมตช์แล้ว (สร้างตารางแข่งแล้ว) → throw
    @Test
    void shouldThrowWhenMatchesAlreadyExist() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        MatchRepository repo = mock(MatchRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getId()).thenReturn(20L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getStatus()).thenReturn(TournamentStatus.UPCOMING);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(repo.existsByTournamentId(20L)).thenReturn(true);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TournamentStatusRule rule = new TournamentStatusRule(repo);

        assertThrows(
            BusinessException.class,
            () -> rule.validate(team, tournament)
        );
    }
}```

### `TeamNotAlreadyJoinedRuleTest` (ด่าน 4)

```java
class TeamNotAlreadyJoinedRuleTest {

    // 👉 ยังไม่เคยสมัคร → ผ่าน
    @Test
    void shouldPassWhenTeamHasNotJoined() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(team.getId()).thenReturn(10L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getId()).thenReturn(20L);
        // 👉 คีย์คู่ (รายการ 20, ทีม 10) ไม่มีอยู่ในฐานข้อมูล
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(repo.existsById(new TournamentTeamId(20L, 10L))).thenReturn(false);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamNotAlreadyJoinedRule rule = new TeamNotAlreadyJoinedRule(repo);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 สมัครไว้แล้ว → throw 409
    @Test
    void shouldThrowWhenTeamAlreadyJoined() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(team.getId()).thenReturn(10L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getId()).thenReturn(20L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(repo.existsById(new TournamentTeamId(20L, 10L))).thenReturn(true);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamNotAlreadyJoinedRule rule = new TeamNotAlreadyJoinedRule(repo);

        assertThrows(
            BusinessException.class,
            () -> rule.validate(team, tournament)
        );
    }
}```

### `TeamGameMatchRuleTest` (ด่าน 5) — commit `test: add team game match rule tests`

```java
class TeamGameMatchRuleTest {

    // 👉 ประกาศฟิลด์ที่คลาสใช้เก็บข้อมูลหรือ Dependency
    private TeamGameMatchRule rule;
    // 👉 ประกาศฟิลด์ที่คลาสใช้เก็บข้อมูลหรือ Dependency
    private Team team;
    // 👉 ประกาศฟิลด์ที่คลาสใช้เก็บข้อมูลหรือ Dependency
    private Tournament tournament;
    // 👉 ประกาศฟิลด์ที่คลาสใช้เก็บข้อมูลหรือ Dependency
    private Game teamGame;
    // 👉 ประกาศฟิลด์ที่คลาสใช้เก็บข้อมูลหรือ Dependency
    private Game tournamentGame;

    // 👉 ก่อนทุกเทสต์: ตั้งให้ทีมและรายการเล่น "เกม id 1" เหมือนกัน (สถานะตั้งต้น = ผ่าน)
    @BeforeEach
    void setUp() {
        rule = new TeamGameMatchRule();   // 👉 ด่านนี้ไม่ใช้ Repository จึง new ตรง ๆ ได้ ไม่ต้อง mock

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        teamGame = new Game();
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        teamGame.setId(1L);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        tournamentGame = new Game();
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournamentGame.setId(1L);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        team = new Team();
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        team.setId(10L);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        team.setGame(teamGame);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        tournament = new Tournament();
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setId(20L);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setGame(tournamentGame);
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setFormat(TournamentFormat.SINGLE_ELIMINATION);
    }

    // 👉 เกมตรงกัน → ผ่าน
    @Test
    void acceptsWhenTeamGameMatchesTournamentGame() {
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 เปลี่ยนเกมของรายการเป็น id 2 (ไม่ตรง) → ปฏิเสธ
    @Test
    void rejectsWhenTeamGameDoesNotMatchTournamentGame() {
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournamentGame.setId(2L);

        assertThrows(
            BusinessException.class,
            () -> rule.validate(team, tournament)
        );
    }

    // 👉 ทีมไม่มีเกม (null) → ปฏิเสธ
    @Test
    void rejectsWhenTeamHasNoGame() {
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        team.setGame(null);

        assertThrows(
            BusinessException.class,
            () -> rule.validate(team, tournament)
        );
    }

    // 👉 รายการไม่มีเกม (null) → ปฏิเสธ
    @Test
    void rejectsWhenTournamentHasNoGame() {
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        tournament.setGame(null);

        assertThrows(
            BusinessException.class,
            () -> rule.validate(team, tournament)
        );
    }
}```

### `TeamMinimumPlayersRuleTest` (ด่าน 6)

```java
class TeamMinimumPlayersRuleTest {

    // 👉 ผู้เล่น 5 คน ขั้นต่ำ 5 → ผ่าน (เท่ากับขั้นต่ำถือว่าพอ เพราะเงื่อนไขคือ "น้อยกว่า" จึงปฏิเสธ)
    @Test
    void shouldPassWhenTeamHasEnoughPlayers() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        PlayerRepository repo = mock(PlayerRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Game game = mock(Game.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(team.getId()).thenReturn(10L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getGame()).thenReturn(game);
        when(game.getMinPlayers()).thenReturn(5);         // 👉 เกมต้องการอย่างน้อย 5 คน
        when(repo.countByTeamId(10L)).thenReturn(5L);     // 👉 ทีมมีจริง 5 คน

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamMinimumPlayersRule rule = new TeamMinimumPlayersRule(repo);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 ผู้เล่น 4 คน ขั้นต่ำ 5 → ปฏิเสธ
    @Test
    void shouldThrowWhenTeamHasTooFewPlayers() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        PlayerRepository repo = mock(PlayerRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Game game = mock(Game.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(team.getId()).thenReturn(10L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getGame()).thenReturn(game);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(game.getMinPlayers()).thenReturn(5);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(repo.countByTeamId(10L)).thenReturn(4L);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamMinimumPlayersRule rule = new TeamMinimumPlayersRule(repo);

        assertThrows(
            BusinessException.class,
            () -> rule.validate(team, tournament)
        );
    }

    // 👉 เกมไม่ได้กำหนดขั้นต่ำ (null) → ข้ามการตรวจ ผ่านเลย และต้องไม่ไปนับผู้เล่นด้วย
    @Test
    void shouldPassWhenMinimumPlayersIsNotConfigured() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        PlayerRepository repo = mock(PlayerRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Game game = mock(Game.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getGame()).thenReturn(game);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(game.getMinPlayers()).thenReturn(null);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamMinimumPlayersRule rule = new TeamMinimumPlayersRule(repo);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        assertDoesNotThrow(() -> rule.validate(team, tournament));
        verifyNoInteractions(repo);   // 👉 ยืนยันว่าไม่ได้เรียก repo เลย
    }
}```

### `TeamPointsLimitRuleTest` (ด่าน 7)

```java
class TeamPointsLimitRuleTest {

    // 👉 POINTS มี 11 ทีม → ยังรับได้ (ทีมที่ 12) → ผ่าน
    @Test
    void shouldPassWhenPointsTournamentHasFewerThanTwelveTeams() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getId()).thenReturn(20L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getFormat()).thenReturn(TournamentFormat.POINTS);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(repo.countByTournamentId(20L)).thenReturn(11L);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamPointsLimitRule rule = new TeamPointsLimitRule(repo);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 POINTS มี 12 ทีมแล้ว → เต็ม → ปฏิเสธ
    @Test
    void shouldThrowWhenPointsTournamentAlreadyHasTwelveTeams() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getId()).thenReturn(20L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getFormat()).thenReturn(TournamentFormat.POINTS);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(repo.countByTournamentId(20L)).thenReturn(12L);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamPointsLimitRule rule = new TeamPointsLimitRule(repo);

        assertThrows(
            BusinessException.class,
            () -> rule.validate(team, tournament)
        );
    }

    // 👉 รายการแพ้คัดออก → กฎนี้ไม่เกี่ยว ข้ามไป และต้องไม่แตะ repo
    @Test
    void shouldSkipTeamLimitForNonPointsTournament() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        when(tournament.getFormat()).thenReturn(
            TournamentFormat.SINGLE_ELIMINATION
        );

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamPointsLimitRule rule = new TeamPointsLimitRule(repo);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        assertDoesNotThrow(() -> rule.validate(team, tournament));
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        verifyNoInteractions(repo);
    }
}```

### `TeamTournamentDateRuleTest` (ด่าน 8) — เคส "วันทับ → ปฏิเสธ" และ "วันไม่ทับ → ผ่าน" ตามที่บรีฟกำหนด

```java
class TeamTournamentDateRuleTest {

    // 👉 เคส: วันไม่ทับ → ผ่าน
    @Test
    void shouldPassWhenThereAreNoOverlappingTournaments() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        LocalDate start = LocalDate.of(2026, 11, 1);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        LocalDate end = LocalDate.of(2026, 11, 10);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(team.getId()).thenReturn(10L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getId()).thenReturn(20L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getStartDate()).thenReturn(start);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getEndDate()).thenReturn(end);

        // 👉 ฐานข้อมูลปลอมตอบว่า "มี 0 รายการที่ชน"
        when(repo.countOverlappingTournaments(
            10L, 20L, start, end
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        )).thenReturn(0L);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamTournamentDateRule rule = new TeamTournamentDateRule(repo);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 เคส: วันทับ → ปฏิเสธ
    @Test
    void shouldThrowWhenTournamentDatesOverlap() {
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Team team = mock(Team.class);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        Tournament tournament = mock(Tournament.class);

        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        LocalDate start = LocalDate.of(2026, 11, 1);
        // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
        LocalDate end = LocalDate.of(2026, 11, 10);

        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(team.getId()).thenReturn(10L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getId()).thenReturn(20L);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getStartDate()).thenReturn(start);
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        when(tournament.getEndDate()).thenReturn(end);

        // 👉 ตอบว่า "มี 1 รายการที่ชน"
        when(repo.countOverlappingTournaments(
            10L, 20L, start, end
        // 👉 เรียกใช้เมธอดเพื่อดำเนินการตามหน้าที่ของมัน
        )).thenReturn(1L);

        // 👉 สร้าง object ใหม่เพื่อใช้เก็บหรือจัดการข้อมูล
        TeamTournamentDateRule rule = new TeamTournamentDateRule(repo);

        assertThrows(
            BusinessException.class,
            () -> rule.validate(team, tournament)
        );
    }
}```

> **หมายเหตุ:** เทสต์นี้ mock ผลของ query ไม่ได้ทดสอบตัวสูตร SQL ว่าคำนวณวันทับถูกต้องหรือไม่ (ต้องมี integration test กับฐานข้อมูลจริงถึงจะตรวจสูตรได้)

---

# 8. ส่วนที่ยืนยันไม่ได้ + เอกสาร data dictionary

## 8.1 Commit ที่ยังยืนยันเนื้อหาไม่ได้

| Commit | สถานะ |
|---|---|
| `คนที่ 3` (2 อัน) | ชื่อ commit ไม่บอกเนื้อหา ไม่ทราบว่าแก้ไฟล์ไหน |
| `fix: resolve tournament test failures` | ไม่ทราบว่าแก้เทสต์ตัวไหน |
| `merge: ... with tournament seed` | ฟีเจอร์ seed (จัดอันดับทีมใน bracket) อยู่ในโมดูลเดียวกับงานคุณ แต่ยืนยันไม่ได้ว่าใครเขียน ดูโค้ดด้านล่าง |
| `merge main/develop ...` | เป็นการรวม branch ไม่มีโค้ดของตัวเองให้สรุป |

โค้ดฟีเจอร์ seed ใน snapshot (ถ้าเป็นงานคุณ ให้ย้ายไปไว้หมวด 6):

```java
// ===== TournamentTeamServiceImpl.setSeeds =====
@Override
// 👉 ประกาศเมธอด/constructor ที่ส่วนอื่นเรียกใช้ได้
public List<TournamentTeamResponse> setSeeds(Long tournamentId, SeedAssignmentRequest request) {

    // 👉 ค้นหาข้อมูลจากฐานข้อมูลด้วยรหัสที่ระบุ
    Tournament tournament = tournamentRepository.findById(tournamentId)
            // 👉 ถ้าไม่พบข้อมูล ให้โยน Exception เพื่อแจ้งว่าไม่พบ
            .orElseThrow(() -> new ResourceNotFoundException("Tournament not found: " + tournamentId));

    // 👉 จัด seed ได้เฉพาะรายการที่ยังไม่เริ่ม
    // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
    if (tournament.getStatus() != TournamentStatus.UPCOMING) {
        // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
        throw new BusinessException("Seeds can only be changed for upcoming tournaments");
    }

    // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
    if (request == null || request.assignments() == null || request.assignments().isEmpty()) {
        // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
        throw new ValidationException("Seed assignments must not be empty");
    }

    // 👉 คำนวณหรือรับค่า แล้วเก็บไว้ในตัวแปรสำหรับใช้ต่อ
    List<TournamentTeam> entries = tournamentTeamRepository.findByTournamentId(tournamentId);

    // 👉 ทำ Map: teamId → แถวการสมัครของทีมนั้น เพื่อค้นหาเร็ว
    // 👉 เปลี่ยนรายการให้เป็น Stream เพื่อประมวลผลทีละรายการ
    Map<Long, TournamentTeam> entriesByTeamId = entries.stream()
            // 👉 รวบรวมข้อมูลจาก Stream เป็นโครงสร้างข้อมูลปลายทาง
            .collect(Collectors.toMap(entry -> entry.getTeam().getId(), Function.identity()));

    Set<Long> submittedTeamIds = new HashSet<>();   // 👉 เก็บ teamId ที่เจอแล้ว ไว้จับซ้ำ
    Set<Integer> submittedSeeds = new HashSet<>();   // 👉 เก็บ seed ที่เจอแล้ว ไว้จับซ้ำ

    // ตรวจสอบข้อมูล Seed ให้ครบก่อนแก้ไขข้อมูลในฐานข้อมูล
    // 👉 วนทำงานกับข้อมูลทีละรายการตามเงื่อนไขที่กำหนด
    for (SeedAssignmentRequest.SeedItem item : request.assignments()) {
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (item == null || item.teamId() == null || item.seed() == null
                || item.teamId() <= 0 || item.seed() <= 0) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException("Team ID and seed must be positive");
        }
        if (!submittedTeamIds.add(item.teamId())) {     // 👉 add คืน false ถ้ามีค่านี้อยู่แล้ว = ซ้ำ
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException("Duplicate team ID: " + item.teamId());
        }
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (!submittedSeeds.add(item.seed())) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException("Duplicate seed: " + item.seed());
        }
        // 👉 ตรวจสอบเงื่อนไขก่อนเลือกว่าจะทำงานในบล็อกนี้หรือไม่
        if (!entriesByTeamId.containsKey(item.teamId())) {
            // 👉 หยุดการทำงานและแจ้งข้อผิดพลาดตามเงื่อนไขนี้
            throw new ValidationException("Team " + item.teamId() + " is not registered in this tournament");
        }
    }

    // บันทึก Seed หลังจากข้อมูลทั้งหมดผ่านการตรวจสอบแล้ว
    // 👉 ตรวจครบก่อนแล้วค่อยแก้ ทำให้ถ้ามีข้อผิดพลาดจะไม่มี seed ที่ถูกแก้ไปครึ่งเดียว
    // 👉 วนทำงานกับข้อมูลทีละรายการตามเงื่อนไขที่กำหนด
    for (SeedAssignmentRequest.SeedItem item : request.assignments()) {
        // 👉 กำหนดค่าฟิลด์ของ object นี้
        entriesByTeamId.get(item.teamId()).setSeed(item.seed());
    }

    // 👉 บันทึกข้อมูลหลายรายการในครั้งเดียว
    tournamentTeamRepository.saveAll(entries);

    // 👉 ส่งค่าผลลัพธ์กลับไปยังผู้เรียกเมธอด
    return entries.stream().map(this::toResponse).toList();
}```

คอลัมน์ในฐานข้อมูล (`V12__add_tournament_team_seed.sql`):

```sql
ALTER TABLE tournament_teams
    ADD COLUMN seed INTEGER CHECK (seed IS NULL OR seed > 0);   -- 👉 seed ต้องว่าง (ยังไม่จัด) หรือเป็นเลขบวก
```

## 8.2 ตาราง `tournament_placement_points` (เกี่ยวกับคะแนนอันดับ)

```sql
CREATE TABLE tournament_placement_points (
    tournament_id BIGINT NOT NULL REFERENCES tournaments (id) ON DELETE CASCADE,   -- 👉 ถ้ารายการถูกลบ คะแนนของรายการนั้นลบตาม
    placement SMALLINT NOT NULL CHECK (placement >= 1),     -- 👉 อันดับต้อง ≥ 1
    points SMALLINT NOT NULL CHECK (points >= 0),           -- 👉 คะแนนต้อง ≥ 0
    PRIMARY KEY (tournament_id, placement)                  -- 👉 คีย์คู่: รายการเดียวกันมีอันดับเดียวกันซ้ำไม่ได้
);
```

## 8.3 `doc/data-dictionary.md` (commit `Add data dictionary`)

เอกสารอธิบายตาราง 11 ตารางของระบบ แต่ละตารางมีคอลัมน์ ชนิดข้อมูล คีย์ (PK/FK) และคำอธิบาย + หัวข้อสรุปความสัมพันธ์ระหว่างตาราง ตารางที่อธิบาย ได้แก่ `users`, `players`, `teams`, `games`, `tournaments`, `tournament_teams`, `matches`, `match_results`, `tournament_placement_points`, `free_fire_games`, `free_fire_game_results`

(เนื้อหาเต็มดูที่ไฟล์ `doc/data-dictionary.md` ใน repo)

---

## ข้อสังเกตจากการอ่านโค้ด (ไม่ใช่ส่วนของ commit ใด)

1. **`TournamentRequest` ไม่มี annotation ตรวจข้อมูล** ถ้าส่ง `name` หรือ `gameId` เป็น `null` จะเกิด `NullPointerException` (ผู้ใช้เห็นเป็น 500) แทนที่จะเป็น 400 วิธีแก้คือเพิ่ม `@NotBlank` ที่ `name` และ `@NotNull` ที่ `gameId`
2. **เลข `12` ใน `TeamPointsLimitRule` และ `createPlacementPoints` เป็น "ตัวเลขมหัศจรรย์"** ถ้าอนาคตอยากเปลี่ยนจำนวนทีมสูงสุด ต้องไล่แก้หลายที่ ควรแยกเป็นค่าคงที่
3. **เทสต์ของ `update`, `delete`, `getByStatus`, `searchByNameAndStatus` ยังไม่มี** ใน `TournamentServiceImplTest`
