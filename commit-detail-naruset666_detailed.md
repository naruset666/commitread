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

// 👉 ประกาศ enum เพื่อกำหนดชุดสถานะที่ระบบยอมรับได้ ผู้เรียกต้องเลือกหนึ่งในค่าคงที่ด้านล่าง แทนการส่งข้อความอะไรก็ได้
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
// 👉 ประกาศสัญญา `TournamentRepository` ระบุว่าโค้ดที่นำไป implement ต้องมีเมธอดตามที่ประกาศไว้ แต่ยังไม่บอกวิธีทำงานจริง
public interface TournamentRepository extends JpaRepository<Tournament, Long> {

    // ค้นหาทัวร์นาเมนต์จากชื่อบางส่วน โดยไม่สนใจตัวพิมพ์ใหญ่หรือเล็ก
    // 👉 Containing = LIKE '%คำค้น%',  IgnoreCase = ไม่สนตัวพิมพ์ เช่น "fire" เจอ "Free Fire Cup"
    // 👉 ประกาศตัวแปร/ชนิดข้อมูล `List<Tournament> findByNameContainingIgnoreCase(String name)` เพื่อระบุว่าค่าที่เก็บเป็นรายการหรือค่าที่อาจไม่มีข้อมูล
    List<Tournament> findByNameContainingIgnoreCase(String name);

    // ค้นหาทัวร์นาเมนต์ตามสถานะ
    // 👉 WHERE status = ?
    // 👉 ประกาศตัวแปร/ชนิดข้อมูล `List<Tournament> findByStatus(TournamentStatus status)` เพื่อระบุว่าค่าที่เก็บเป็นรายการหรือค่าที่อาจไม่มีข้อมูล
    List<Tournament> findByStatus(TournamentStatus status);

    // ค้นหาทัวร์นาเมนต์จากชื่อบางส่วนและสถานะพร้อมกัน
    // 👉 WHERE name LIKE ... AND status = ?
    List<Tournament> findByNameContainingIgnoreCaseAndStatus(
            // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
            String name,
            // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
            TournamentStatus status);

    // 👉 หาด้วยชื่อตรงตัว คืนเป็น Optional (อาจไม่เจอ)
    // 👉 ประกาศตัวแปร/ชนิดข้อมูล `Optional<Tournament> findByName(String name)` เพื่อระบุว่าค่าที่เก็บเป็นรายการหรือค่าที่อาจไม่มีข้อมูล
    Optional<Tournament> findByName(String name);

    // 👉 เช็คว่ามีชื่อนี้อยู่แล้วหรือไม่ (true/false) ใช้กันชื่อรายการซ้ำตอนสร้าง
    // 👉 บรรทัด `boolean existsByName(String name);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
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

// 👉 ประกาศคลาส `TournamentRequest` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
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
    // 👉 ประกาศเมธอด `getGameId()` เพื่ออ่านค่าฟิลด์ `gameId` แล้วคืนค่าให้โค้ดที่เรียกใช้; ชนิดค่าที่คืนคือ `Long`
    public Long getGameId() {
        // 👉 คืนค่า `gameId` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return gameId;
    }

    // 👉 ประกาศเมธอด `setGameId(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `gameId`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setGameId(Long gameId) {
        // 👉 เก็บ dependency/ค่าจาก `gameId` ลงฟิลด์ `gameId` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.gameId = gameId;
    }

    // 👉 ประกาศเมธอด `getFormat()` เพื่ออ่านค่าฟิลด์ `format` แล้วคืนค่าให้โค้ดที่เรียกใช้; ชนิดค่าที่คืนคือ `TournamentFormat`
    public TournamentFormat getFormat() {
        // 👉 คืนค่า `format` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return format;
    }

    // 👉 ประกาศเมธอด `setFormat(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `format`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setFormat(TournamentFormat format) {
        // 👉 เก็บ dependency/ค่าจาก `format` ลงฟิลด์ `format` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.format = format;
    }

    // 👉 ประกาศเมธอด `getTotalGames()` เพื่ออ่านค่าฟิลด์ `totalGames` แล้วคืนค่าให้โค้ดที่เรียกใช้; ชนิดค่าที่คืนคือ `Short`
    public Short getTotalGames() {
        // 👉 คืนค่า `totalGames` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return totalGames;
    }

    // 👉 ประกาศเมธอด `setTotalGames(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `totalGames`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setTotalGames(Short totalGames) {
        // 👉 เก็บ dependency/ค่าจาก `totalGames` ลงฟิลด์ `totalGames` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.totalGames = totalGames;
    }

    // 👉 ประกาศเมธอด `getPointsPerKill()` เพื่ออ่านค่าฟิลด์ `pointsPerKill` แล้วคืนค่าให้โค้ดที่เรียกใช้; ชนิดค่าที่คืนคือ `Short`
    public Short getPointsPerKill() {
        // 👉 คืนค่า `pointsPerKill` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return pointsPerKill;
    }

    // 👉 ประกาศเมธอด `setPointsPerKill(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `pointsPerKill`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setPointsPerKill(Short pointsPerKill) {
        // 👉 เก็บ dependency/ค่าจาก `pointsPerKill` ลงฟิลด์ `pointsPerKill` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.pointsPerKill = pointsPerKill;
    }

    // 👉 ประกาศเมธอด `getLogoUrl()` เพื่ออ่านค่าฟิลด์ `logoUrl` แล้วคืนค่าให้โค้ดที่เรียกใช้; ชนิดค่าที่คืนคือ `String`
    public String getLogoUrl() {
        // 👉 คืนค่า `logoUrl` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return logoUrl;
    }

    // 👉 ประกาศเมธอด `setLogoUrl(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `logoUrl`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setLogoUrl(String logoUrl) {
        // 👉 เก็บ dependency/ค่าจาก `logoUrl` ลงฟิลด์ `logoUrl` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.logoUrl = logoUrl;
    }

    // 👉 ประกาศเมธอด `getStartDate()` เพื่ออ่านค่าฟิลด์ `startDate` แล้วคืนค่าให้โค้ดที่เรียกใช้; ชนิดค่าที่คืนคือ `LocalDate`
    public LocalDate getStartDate() {
        // 👉 คืนค่า `startDate` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return startDate;
    }

    // 👉 ประกาศเมธอด `setStartDate(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `startDate`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setStartDate(LocalDate startDate) {
        // 👉 เก็บ dependency/ค่าจาก `startDate` ลงฟิลด์ `startDate` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.startDate = startDate;
    }

    // 👉 ประกาศเมธอด `getEndDate()` เพื่ออ่านค่าฟิลด์ `endDate` แล้วคืนค่าให้โค้ดที่เรียกใช้; ชนิดค่าที่คืนคือ `LocalDate`
    public LocalDate getEndDate() {
        // 👉 คืนค่า `endDate` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return endDate;
    }

    // 👉 ประกาศเมธอด `setEndDate(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `endDate`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setEndDate(LocalDate endDate) {
        // 👉 เก็บ dependency/ค่าจาก `endDate` ลงฟิลด์ `endDate` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.endDate = endDate;
    }

    // 👉 ประกาศเมธอด `getName()` เพื่ออ่านค่าฟิลด์ `name` แล้วคืนค่าให้โค้ดที่เรียกใช้; ชนิดค่าที่คืนคือ `String`
    public String getName() {
        // 👉 คืนค่า `name` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return name;
    }

    // 👉 ประกาศเมธอด `setName(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `name`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setName(String name) {
        // 👉 เก็บ dependency/ค่าจาก `name` ลงฟิลด์ `name` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.name = name;
    }

    // 👉 ประกาศเมธอด `getDescription()` เพื่ออ่านค่าฟิลด์ `description` แล้วคืนค่าให้โค้ดที่เรียกใช้; ชนิดค่าที่คืนคือ `String`
    public String getDescription() {
        // 👉 คืนค่า `description` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return description;
    }

    // 👉 ประกาศเมธอด `setDescription(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `description`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setDescription(String description) {
        // 👉 เก็บ dependency/ค่าจาก `description` ลงฟิลด์ `description` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
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
// 👉 ประกาศ `record` สำหรับเก็บข้อมูลหลายฟิลด์เป็นชุด Java สร้าง constructor และเมธอดอ่านค่าตามชื่อฟิลด์ให้อัตโนมัติ
public record TournamentResponse(
        Long id,                       // 👉 รหัสรายการที่ฐานข้อมูลสร้างให้
        // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
        String name,
        // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
        String description,
        Long gameId,                   // 👉 ส่งแค่ id ของเกม ไม่ส่งทั้ง object Game
        // 👉 พารามิเตอร์นี้เก็บรูปแบบการแข่งขัน เช่น `POINTS` หรือ `SINGLE_ELIMINATION`
        TournamentFormat format,
        // 👉 พารามิเตอร์นี้เก็บจำนวนเกมทั้งหมดของรายการแบบสะสมคะแนน
        Short totalGames,
        // 👉 พารามิเตอร์นี้เก็บจำนวนคะแนนที่ให้ต่อหนึ่ง kill
        Short pointsPerKill,
        // 👉 พารามิเตอร์นี้เก็บ URL ที่ชี้ไปยังรูปโลโก้ของรายการ
        String logoUrl,
        // 👉 พารามิเตอร์นี้เก็บวันที่เริ่มแข่งขัน โดย `LocalDate` ไม่มีส่วนเวลา
        LocalDate startDate,
        // 👉 พารามิเตอร์นี้เก็บวันที่สิ้นสุดแข่งขัน โดย `LocalDate` ไม่มีส่วนเวลา
        LocalDate endDate,
        TournamentStatus status) {     // 👉 สถานะปัจจุบัน (UPCOMING/ONGOING/COMPLETED)
}```

**ทำไมแยก Request/Response ออกจาก Entity:** Request ไม่ต้องมี `id`/`status` (ระบบกำหนดเอง) ส่วน Response เลือกส่งเฉพาะที่ผู้ใช้ควรเห็น และแก้โครงสร้างฐานข้อมูลได้โดยไม่กระทบรูป JSON

---

## 1.5 `PlacementPointResponse.java` — คะแนนของอันดับหนึ่ง ๆ

```java
package com.example.tournament.dto.response;

// 👉 1 แถว = อันดับที่ (placement) ได้กี่คะแนน (points)  เช่น (1, 12) = อันดับ 1 ได้ 12 คะแนน
// 👉 ประกาศ `record` สำหรับเก็บข้อมูลหลายฟิลด์เป็นชุด Java สร้าง constructor และเมธอดอ่านค่าตามชื่อฟิลด์ให้อัตโนมัติ
public record PlacementPointResponse(
        // 👉 ฟิลด์/พารามิเตอร์นี้ระบุค่าตัวเลขของข้อมูลที่ record หรือเมธอดกำลังเก็บ
        Short placement,
        // 👉 ฟิลด์/พารามิเตอร์นี้ระบุค่าตัวเลขของข้อมูลที่ record หรือเมธอดกำลังเก็บ
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
// 👉 ประกาศสัญญา `TournamentPlacementPointRepository` ระบุว่าโค้ดที่นำไป implement ต้องมีเมธอดตามที่ประกาศไว้ แต่ยังไม่บอกวิธีทำงานจริง
public interface TournamentPlacementPointRepository
        // 👉 สืบทอดความสามารถ CRUD จาก Spring Data JPA โดย generic ตัวแรกคือ Entity และตัวที่สองคือชนิดของ primary key
        extends JpaRepository<TournamentPlacementPoint, TournamentPlacementPointId> {

    // ใช้ใน FreeFireResultServiceImpl (คำนวณคะแนน)
    // 👉 ดึงคะแนนทุกอันดับของรายการหนึ่ง (ไม่เรียงลำดับ) งานของสมาชิกอีกคนใช้
    // 👉 ประกาศตัวแปร/ชนิดข้อมูล `List<TournamentPlacementPoint> findByTournamentId(Long tournamentId)` เพื่อระบุว่าค่าที่เก็บเป็นรายการหรือค่าที่อาจไม่มีข้อมูล
    List<TournamentPlacementPoint> findByTournamentId(Long tournamentId);

    // ใช้ในงาน Tournament ของ Naruset
    // 👉 ดึงคะแนนของรายการหนึ่ง เรียงอันดับ 1 → 12 (ASC = น้อยไปมาก)
    // 👉 ประกาศตัวแปร/ชนิดข้อมูล `List<TournamentPlacementPoint> findByTournamentIdOrderByPlacementAsc(Long tournamentId)` เพื่อระบุว่าค่าที่เก็บเป็นรายการหรือค่าที่อาจไม่มีข้อมูล
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

// 👉 ประกาศสัญญา `TournamentService` ระบุว่าโค้ดที่นำไป implement ต้องมีเมธอดตามที่ประกาศไว้ แต่ยังไม่บอกวิธีทำงานจริง
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
            // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
            String name,
            // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
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
            // 👉 พารามิเตอร์ constructor นี้เป็น Repository ที่ Service ต้องใช้ติดต่อฐานข้อมูล
            TournamentRepository tournamentRepository,
            // 👉 พารามิเตอร์ constructor นี้เป็น Repository ที่ Service ต้องใช้ติดต่อฐานข้อมูล
            GameRepository gameRepository,
            // 👉 พารามิเตอร์ constructor นี้เป็น Repository ที่ Service ต้องใช้ติดต่อฐานข้อมูล
            TournamentPlacementPointRepository tournamentPlacementPointRepository) {

        // 👉 เก็บ dependency/ค่าจาก `tournamentRepository` ลงฟิลด์ `tournamentRepository` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.tournamentRepository = tournamentRepository;
        // 👉 เก็บ dependency/ค่าจาก `gameRepository` ลงฟิลด์ `gameRepository` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.gameRepository = gameRepository;
        // 👉 เก็บ dependency/ค่าจาก `tournamentPlacementPointRepository` ลงฟิลด์ `tournamentPlacementPointRepository` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.tournamentPlacementPointRepository = tournamentPlacementPointRepository;
    }

    // ================= สร้างรายการ =================
    @Override
    // 👉 ประกาศเมธอด `create()` โดยรับพารามิเตอร์ `TournamentRequest request`; ชนิดผลลัพธ์ `TournamentResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public TournamentResponse create(TournamentRequest request) {
        // 👉 ขั้น 1: ตัดช่องว่างหัวท้ายของชื่อ  "  ROV Cup " → "ROV Cup"
        // 👉 สร้างตัวแปรข้อความ `name` โดยอ่านค่าจาก `request.getName().trim()`; `trim()` ตัดช่องว่างหัวท้ายก่อนนำชื่อไปตรวจสอบหรือบันทึก
        String name = request.getName().trim();

        // 👉 ขั้น 2: ชื่อซ้ำหรือไม่? ถ้าซ้ำ → BusinessException → HTTP 409
        // 👉 ตรวจว่ามีทัวร์นาเมนต์ใช้ชื่อนี้อยู่แล้วหรือไม่; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (tournamentRepository.existsByName(name)) {`
        if (tournamentRepository.existsByName(name)) {
            // 👉 โยน `BusinessException` พร้อมข้อความ `"Tournament name already exists"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new BusinessException("Tournament name already exists");
        }

        // 👉 ขั้น 3: หาเกมจาก gameId  ถ้าไม่เจอ → ResourceNotFoundException → HTTP 404
        // 👉 ประกาศตัวแปร `game` แล้วค้นหาข้อมูลด้วย `gameRepository.findById(request.getGameId())`; Repository คืน `Optional` เพราะรหัสนั้นอาจมีหรือไม่มีในฐานข้อมูล
        Game game = gameRepository.findById(request.getGameId())
                // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
                .orElseThrow(() -> new ResourceNotFoundException("Game not found"));

        // 👉 ขั้น 4: ตรวจกฎข้อมูล (วันที่, format ฯลฯ) ถ้าผิด → ValidationException → HTTP 400
        // 👉 เรียกเมธอดตรวจสอบกฎข้อมูลก่อนบันทึก/แก้ไข เช่น วันที่ รูปแบบเกม และจำนวนเกม; ถ้าผิดกฎเมธอดนี้จะ throw exception ทำให้ไม่ไปขั้นตอนบันทึก
        validateTournamentRequest(request, game);

        // 👉 ขั้น 5: สร้าง Entity แล้วคัดลอกค่าจาก request ใส่
        // 👉 ประกาศตัวแปร `tournament` เพื่ออ้างถึง object `Tournament` ที่เพิ่งสร้าง หรือข้อมูลที่ได้จากนิพจน์ `new Tournament()`
        Tournament tournament = new Tournament();

        // 👉 เรียก `tournament.setName(name)` เพื่อกำหนดค่า ชื่อรายการที่ผ่านการตัดช่องว่างแล้ว; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setName(name);
        // 👉 เรียก `tournament.setDescription(request.getDescription())` เพื่อกำหนดค่า คำอธิบายรายการจาก request; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setDescription(request.getDescription());
        // 👉 เรียก `tournament.setGame(game)` เพื่อกำหนดค่า object เกมที่ค้นพบจาก gameId; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setGame(game);
        // 👉 เรียก `tournament.setFormat(request.getFormat())` เพื่อกำหนดค่า รูปแบบการแข่งขันที่ผู้ใช้เลือก; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setFormat(request.getFormat());
        // 👉 เรียก `tournament.setTotalGames(request.getTotalGames())` เพื่อกำหนดค่า จำนวนเกมทั้งหมด (สำคัญกับ format POINTS); setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setTotalGames(request.getTotalGames());
        // 👉 เรียก `tournament.setPointsPerKill(request.getPointsPerKill())` เพื่อกำหนดค่า คะแนนต่อการกำจัดหนึ่งครั้ง; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setPointsPerKill(request.getPointsPerKill());
        // 👉 เรียก `tournament.setLogoUrl(request.getLogoUrl())` เพื่อกำหนดค่า URL ของโลโก้รายการ; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setLogoUrl(request.getLogoUrl());
        // 👉 เรียก `tournament.setStartDate(request.getStartDate())` เพื่อกำหนดค่า วันที่เริ่มแข่งขัน; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setStartDate(request.getStartDate());
        // 👉 เรียก `tournament.setEndDate(request.getEndDate())` เพื่อกำหนดค่า วันที่สิ้นสุดการแข่งขัน; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setEndDate(request.getEndDate());
        tournament.setStatus(TournamentStatus.UPCOMING);   // 👉 รายการใหม่เริ่มที่ UPCOMING เสมอ ผู้ใช้กำหนดเองไม่ได้
        tournament.setCreatedAt(LocalDateTime.now());      // 👉 เวลาที่สร้าง = ตอนนี้

        // 👉 ขั้น 6: บันทึกลงฐานข้อมูล  saveAndFlush = บันทึกและส่งคำสั่งไปฐานข้อมูลทันที
        //    และคืน object ที่มี id ซึ่งฐานข้อมูลเพิ่งสร้างให้
        // 👉 ประกาศตัวแปร `savedTournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `tournamentRepository.saveAndFlush(tournament)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament savedTournament = tournamentRepository.saveAndFlush(tournament);

        /*
         * สำหรับทัวร์นาเมนต์ Free Fire รูปแบบ POINTS
         * ระบบจะสร้างคะแนนตามอันดับให้อัตโนมัติ
         */
        // 👉 ขั้น 7: ถ้าเป็นแบบ POINTS ให้ใส่คะแนนอันดับตั้งต้น 12 แถว (ดู createPlacementPoints ด้านล่าง)
        // 👉 ตรวจค่าของเงื่อนไข `savedTournament.getFormat() == TournamentFormat.POINTS`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (savedTournament.getFormat() == TournamentFormat.POINTS) {`
        if (savedTournament.getFormat() == TournamentFormat.POINTS) {
            // 👉 บรรทัด `createPlacementPoints(savedTournament);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            createPlacementPoints(savedTournament);
        }

        // 👉 ขั้น 8: แปลงเป็น Response แล้วส่งกลับ
        // 👉 แปลง entity ด้วย `toResponse(savedTournament)` แล้วคืน DTO ให้ Controller; DTO คือรูปข้อมูลที่ API ส่งกลับไปยังผู้ใช้
        return toResponse(savedTournament);
    }

    // ================= ดึงข้อมูล =================
    @Override
    @Transactional(readOnly = true)   // 👉 เมธอดนี้แค่อ่าน ไม่แก้ข้อมูล (เร็วกว่าเล็กน้อย)
    // 👉 ประกาศเมธอด `getAll()` เพื่ออ่านค่าฟิลด์ `all` แล้วคืนค่าให้โค้ดที่เรียกใช้; ชนิดค่าที่คืนคือ `List<TournamentResponse>`
    public List<TournamentResponse> getAll() {
        return tournamentRepository.findAll()   // 👉 ดึงทุกแถวจากตาราง tournaments
                .stream()                       // 👉 เปลี่ยนลิสต์เป็น stream เพื่อไล่ทีละตัว
                .map(this::toResponse)          // 👉 แปลง Entity แต่ละตัวเป็น TournamentResponse
                .toList();                      // 👉 เก็บกลับเป็นลิสต์
    }

    // ================= ค้นหาตามชื่อ =================
    @Override
    // 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด `searchByName()` โดยรับพารามิเตอร์ `String name`; ชนิดผลลัพธ์ `List<TournamentResponse>` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public List<TournamentResponse> searchByName(String name) {
        // ตัดช่องว่างส่วนเกินก่อนค้นหาชื่อ
        String keyword = name.trim();   // 👉 "  ROV  " → "ROV"

        // ค้นหาจากชื่อบางส่วนโดยไม่สนใจตัวพิมพ์ใหญ่หรือเล็ก
        // 👉 คืนค่า `tournamentRepository` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return tournamentRepository
                // 👉 เรียก query method `.findByNameContainingIgnoreCase(keyword)` ที่ Repository; Spring Data แปลชื่อเมธอดเป็นเงื่อนไขค้นหาในฐานข้อมูล
                .findByNameContainingIgnoreCase(keyword)
                // 👉 บรรทัด `.stream()` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .stream()
                // 👉 บรรทัด `.map(this::toResponse)` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .map(this::toResponse)
                // 👉 บรรทัด `.toList();` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .toList();
    }

    // ================= กรองตามสถานะ =================
    @Override
    // 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด `getByStatus()` โดยรับพารามิเตอร์ `TournamentStatus status`; ชนิดผลลัพธ์ `List<TournamentResponse>` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public List<TournamentResponse> getByStatus(TournamentStatus status) {
        // ค้นหาทัวร์นาเมนต์ตามสถานะที่ระบุ
        // 👉 คืนผลการค้นหาจาก Repository ให้ผู้เรียก; รายการที่ได้มาจากฐานข้อมูลตามเงื่อนไข `tournamentRepository.findByStatus(status)`
        return tournamentRepository.findByStatus(status)
                // 👉 บรรทัด `.stream()` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .stream()
                // 👉 บรรทัด `.map(this::toResponse)` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .map(this::toResponse)
                // 👉 บรรทัด `.toList();` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .toList();
    }

    // ================= ค้นหาชื่อ + สถานะ =================
    @Override
    // 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
    @Transactional(readOnly = true)
    // 👉 ประกาศสมาชิกของคลาส: `public List<TournamentResponse> searchByNameAndStatus(`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public List<TournamentResponse> searchByNameAndStatus(
            // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
            String name,
            // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
            TournamentStatus status) {

        // ตัดช่องว่างส่วนเกินก่อนค้นหาชื่อ
        // 👉 สร้างตัวแปรข้อความ `keyword` โดยอ่านค่าจาก `name.trim()`; `trim()` ตัดช่องว่างหัวท้ายก่อนนำชื่อไปตรวจสอบหรือบันทึก
        String keyword = name.trim();

        // ค้นหาจากชื่อบางส่วนและสถานะพร้อมกัน
        // 👉 คืนค่า `tournamentRepository` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return tournamentRepository
                // 👉 เรียก query method `.findByNameContainingIgnoreCaseAndStatus(keyword, status)` ที่ Repository; Spring Data แปลชื่อเมธอดเป็นเงื่อนไขค้นหาในฐานข้อมูล
                .findByNameContainingIgnoreCaseAndStatus(keyword, status)
                // 👉 บรรทัด `.stream()` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .stream()
                // 👉 บรรทัด `.map(this::toResponse)` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .map(this::toResponse)
                // 👉 บรรทัด `.toList();` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .toList();
    }

    // ================= ดึงรายการเดียว =================
    @Override
    // 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด `getById()` โดยรับพารามิเตอร์ `Long id`; ชนิดผลลัพธ์ `TournamentResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public TournamentResponse getById(Long id) {
        return toResponse(findTournament(id));   // 👉 findTournament throw 404 ให้เองถ้าไม่เจอ
    }

    // ================= แก้ไขรายการ =================
    @Override
    // 👉 ประกาศสมาชิกของคลาส: `public TournamentResponse update(`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public TournamentResponse update(
            // 👉 พารามิเตอร์นี้ส่งข้อมูลจากผู้เรียกเข้ามาให้เมธอดใช้; ชื่อและชนิดข้อมูลช่วยบอกว่าต้องส่งอะไร
            Long id,
            // 👉 พารามิเตอร์นี้ส่งข้อมูลจากผู้เรียกเข้ามาให้เมธอดใช้; ชื่อและชนิดข้อมูลช่วยบอกว่าต้องส่งอะไร
            TournamentRequest request) {

        // 👉 หารายการเดิม (ไม่เจอ → 404)
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `findTournament(id)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = findTournament(id);

        // 👉 สร้างตัวแปรข้อความ `name` โดยอ่านค่าจาก `request.getName().trim()`; `trim()` ตัดช่องว่างหัวท้ายก่อนนำชื่อไปตรวจสอบหรือบันทึก
        String name = request.getName().trim();

        // 👉 เช็คชื่อซ้ำ "เฉพาะเมื่อชื่อเปลี่ยนจริง"
        //    ถ้าส่งชื่อเดิมมา (ไม่สนตัวพิมพ์) ไม่ถือว่าซ้ำกับตัวเอง
        // 👉 ตรวจค่าของเงื่อนไข `!name.equalsIgnoreCase(tournament.getName()`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (!name.equalsIgnoreCase(tournament.getName())`
        if (!name.equalsIgnoreCase(tournament.getName())
                // 👉 เป็นเงื่อนไขต่อจากบรรทัด `if` ก่อนหน้า โดย `&&` หมายถึงทั้งเงื่อนไขก่อนหน้าและเงื่อนไขนี้ต้องเป็นจริง
                && tournamentRepository.existsByName(name)) {

            // 👉 โยน `BusinessException` พร้อมข้อความ `"Tournament name already exists"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new BusinessException("Tournament name already exists");
        }

        // 👉 ประกาศตัวแปร `game` แล้วค้นหาข้อมูลด้วย `gameRepository.findById(request.getGameId())`; Repository คืน `Optional` เพราะรหัสนั้นอาจมีหรือไม่มีในฐานข้อมูล
        Game game = gameRepository.findById(request.getGameId())
                // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
                .orElseThrow(() -> new ResourceNotFoundException("Game not found"));

        validateTournamentRequest(request, game);   // 👉 ใช้กฎชุดเดียวกับตอนสร้าง

        // 👉 เขียนค่าใหม่ทับ Entity เดิม (ไม่แตะ status/createdAt)
        // 👉 เรียก `tournament.setName(name)` เพื่อกำหนดค่า ชื่อรายการที่ผ่านการตัดช่องว่างแล้ว; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setName(name);
        // 👉 เรียก `tournament.setDescription(request.getDescription())` เพื่อกำหนดค่า คำอธิบายรายการจาก request; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setDescription(request.getDescription());
        // 👉 เรียก `tournament.setGame(game)` เพื่อกำหนดค่า object เกมที่ค้นพบจาก gameId; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setGame(game);
        // 👉 เรียก `tournament.setFormat(request.getFormat())` เพื่อกำหนดค่า รูปแบบการแข่งขันที่ผู้ใช้เลือก; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setFormat(request.getFormat());
        // 👉 เรียก `tournament.setTotalGames(request.getTotalGames())` เพื่อกำหนดค่า จำนวนเกมทั้งหมด (สำคัญกับ format POINTS); setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setTotalGames(request.getTotalGames());
        // 👉 เรียก `tournament.setPointsPerKill(request.getPointsPerKill())` เพื่อกำหนดค่า คะแนนต่อการกำจัดหนึ่งครั้ง; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setPointsPerKill(request.getPointsPerKill());
        // 👉 เรียก `tournament.setLogoUrl(request.getLogoUrl())` เพื่อกำหนดค่า URL ของโลโก้รายการ; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setLogoUrl(request.getLogoUrl());
        // 👉 เรียก `tournament.setStartDate(request.getStartDate())` เพื่อกำหนดค่า วันที่เริ่มแข่งขัน; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setStartDate(request.getStartDate());
        // 👉 เรียก `tournament.setEndDate(request.getEndDate())` เพื่อกำหนดค่า วันที่สิ้นสุดการแข่งขัน; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setEndDate(request.getEndDate());

        // 👉 แปลง entity ด้วย `toResponse(tournamentRepository.saveAndFlush(tournament))` แล้วคืน DTO ให้ Controller; DTO คือรูปข้อมูลที่ API ส่งกลับไปยังผู้ใช้
        return toResponse(tournamentRepository.saveAndFlush(tournament));
    }

    // ================= ลบรายการ =================
    @Override
    // 👉 ประกาศเมธอด `delete()` โดยรับพารามิเตอร์ `Long id`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void delete(Long id) {
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `findTournament(id)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = findTournament(id);

        // 👉 ลบได้เฉพาะรายการที่ยังไม่เริ่ม ถ้าเริ่มหรือจบแล้ว → 409
        // 👉 ตรวจว่าสถานะไม่ใช่ UPCOMING หรือไม่ เพราะห้ามลบรายการที่เริ่มแล้ว; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (tournament.getStatus() != TournamentStatus.UPCOMING) {`
        if (tournament.getStatus() != TournamentStatus.UPCOMING) {
            // 👉 บรรทัด `throw new BusinessException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new BusinessException(
                    // 👉 ข้อความนี้เป็นรายละเอียดของข้อผิดพลาด ช่วยบอก client ว่าข้อมูลผิดกฎข้อไหน
                    "Tournament can only be deleted when it is UPCOMING");
        }

        // 👉 เรียก `tournamentRepository.delete(tournament)` เพื่อสั่ง Repository ลบ entity ที่ระบุออกจากฐานข้อมูล
        tournamentRepository.delete(tournament);
        // 👉 สั่งให้ JPA ส่งคำสั่งค้างไปยังฐานข้อมูล
        tournamentRepository.flush();
    }

    // ================= ดูคะแนนตามอันดับ (API) =================
    @Override
    // 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด `getPlacementPoints()` โดยรับพารามิเตอร์ `Long tournamentId`; ชนิดผลลัพธ์ `List<PlacementPointResponse>` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public List<PlacementPointResponse> getPlacementPoints(Long tournamentId) {

        // ตรวจสอบก่อนว่าทัวร์นาเมนต์มีอยู่จริง
        findTournament(tournamentId);   // 👉 ไม่มี → 404 (ค่าที่คืนไม่ได้ใช้ ใช้แค่เช็คว่ามีจริง)

        // 👉 คืนค่า `tournamentPlacementPointRepository` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return tournamentPlacementPointRepository
                .findByTournamentIdOrderByPlacementAsc(tournamentId)   // 👉 ดึงคะแนนเรียงอันดับ 1→12
                // 👉 บรรทัด `.stream()` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .stream()
                .map(point -> new PlacementPointResponse(              // 👉 แปลงแต่ละแถวเป็น (อันดับ, คะแนน)
                        // 👉 ส่งเลขอันดับของแถวคะแนนนี้เป็น argument แรกของ `PlacementPointResponse`
                        point.getPlacement(),
                        // 👉 ส่งคะแนนของอันดับนี้เป็น argument ถัดไป เพื่อสร้าง response ที่จับคู่เลขอันดับกับคะแนน
                        point.getPoints()))
                // 👉 บรรทัด `.toList();` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .toList();
    }

    /**
     * สร้างคะแนนตามอันดับสำหรับทัวร์นาเมนต์รูปแบบ POINTS
     */
    // 👉 ประกาศสมาชิกของคลาส: `private void createPlacementPoints(Tournament tournament)`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    private void createPlacementPoints(Tournament tournament) {

        // 👉 อาร์เรย์คะแนนของอันดับ 1..12  (ช่อง 0 = อันดับ 1, ช่อง 1 = อันดับ 2, ...)
        //    อันดับ: 1  2  3  4  5  6  7  8  9  10 11 12
        //    คะแนน: 12  9  8  7  6  5  4  3  2  1  0  0   (รูปแบบ FFWS)
        short[] points = {
                // 👉 ตัวเลขในอาร์เรย์เป็นคะแนนตามอันดับ เรียงจากอันดับ 1 ไป 12; ตำแหน่งในอาร์เรย์เริ่มนับจาก 0
                12, 9, 8, 7, 6, 5,
                // 👉 ตัวเลขในอาร์เรย์เป็นคะแนนตามอันดับ เรียงจากอันดับ 1 ไป 12; ตำแหน่งในอาร์เรย์เริ่มนับจาก 0
                4, 3, 2, 1, 0, 0
        };

        // 👉 วนอันดับ 1 ถึง 12 สร้างแถวใหม่ทีละอันดับ
        // 👉 เริ่มวนอันดับตั้งแต่ 1 ถึง 12; แต่ละรอบจะสร้างคะแนนให้หนึ่งอันดับ โดยเพิ่ม placement ทีละ 1
        for (short placement = 1; placement <= 12; placement++) {
            // 👉 ประกาศตัวแปร `placementPoint` เพื่ออ้างถึง object `TournamentPlacementPoint` ที่เพิ่งสร้าง หรือข้อมูลที่ได้จากนิพจน์ `new TournamentPlacementPoint()`
            TournamentPlacementPoint placementPoint = new TournamentPlacementPoint();

            placementPoint.setTournament(tournament);           // 👉 ของรายการไหน
            placementPoint.setPlacement(placement);             // 👉 อันดับที่
            placementPoint.setPoints(points[placement - 1]);    // 👉 ได้กี่คะแนน (ลบ 1 เพราะอาร์เรย์เริ่มที่ช่อง 0)

            tournamentPlacementPointRepository.save(placementPoint);   // 👉 บันทึกลงตาราง tournament_placement_points
        }
    }

    // ================= กฎตรวจสอบข้อมูล =================
    private void validateTournamentRequest(
            // 👉 พารามิเตอร์นี้ส่งข้อมูลจากผู้เรียกเข้ามาให้เมธอดใช้; ชื่อและชนิดข้อมูลช่วยบอกว่าต้องส่งอะไร
            TournamentRequest request,
            // 👉 พารามิเตอร์นี้ส่งข้อมูลจากผู้เรียกเข้ามาให้เมธอดใช้; ชื่อและชนิดข้อมูลช่วยบอกว่าต้องส่งอะไร
            Game game) {

        // 👉 กฎ 1: ต้องมีทั้งวันเริ่มและวันจบ
        // 👉 ตรวจว่าขาดวันเริ่มหรือวันจบอย่างน้อยหนึ่งค่า; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (request.getStartDate() == null || request.getEndDate() == null) {`
        if (request.getStartDate() == null || request.getEndDate() == null) {
            // 👉 โยน `ValidationException` พร้อมข้อความ `"Start date and end date are required"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ValidationException("Start date and end date are required");
        }

        // 👉 กฎ 2: วันจบต้องไม่มาก่อนวันเริ่ม (วันเดียวกันได้)
        // 👉 ตรวจว่าวันจบอยู่ก่อนวันเริ่มหรือไม่; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (request.getEndDate().isBefore(request.getStartDate())) {`
        if (request.getEndDate().isBefore(request.getStartDate())) {
            // 👉 โยน `ValidationException` พร้อมข้อความ `"End date must be on or after start date"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ValidationException("End date must be on or after start date");
        }

        // 👉 กฎ 3: ต้องระบุรูปแบบการแข่ง
        // 👉 ตรวจว่าผู้ใช้ไม่ได้ส่งรูปแบบการแข่งขันมา; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (request.getFormat() == null) {`
        if (request.getFormat() == null) {
            // 👉 โยน `ValidationException` พร้อมข้อความ `"Tournament format is required"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ValidationException("Tournament format is required");
        }

        // 👉 เช็คว่าเกมนี้คือ Free Fire หรือไม่ จากรหัสเกม (ไม่สนตัวพิมพ์)
        // 👉 สร้างตัวแปร boolean `isFreeFire` เพื่อเก็บผล true/false จากการเทียบข้อความโดยไม่สนตัวพิมพ์ใหญ่เล็ก
        boolean isFreeFire = "FREE_FIRE".equalsIgnoreCase(game.getCode());

        // 👉 กฎ 4: Free Fire ต้องใช้ POINTS เท่านั้น (เพราะเป็นเกมแบบแข่งสะสมคะแนน ไม่ใช่ 1v1)
        // 👉 ตรวจว่าเป็น Free Fire แต่เลือก format ไม่ใช่ POINTS หรือไม่; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (isFreeFire && request.getFormat() != TournamentFormat.POINTS) {`
        if (isFreeFire && request.getFormat() != TournamentFormat.POINTS) {
            // 👉 บรรทัด `throw new ValidationException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new ValidationException(
                    // 👉 ข้อความนี้เป็นรายละเอียดของข้อผิดพลาด ช่วยบอก client ว่าข้อมูลผิดกฎข้อไหน
                    "Free Fire tournaments must use POINTS format");
        }

        // 👉 กฎ 5: เกมอื่นห้ามใช้ POINTS
        // 👉 ตรวจว่าเป็นเกมอื่นที่ไม่ใช่ Free Fire แต่เลือก POINTS หรือไม่; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (!isFreeFire && request.getFormat() == TournamentFormat.POINTS) {`
        if (!isFreeFire && request.getFormat() == TournamentFormat.POINTS) {
            // 👉 บรรทัด `throw new ValidationException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new ValidationException(
                    // 👉 ข้อความนี้เป็นรายละเอียดของข้อผิดพลาด ช่วยบอก client ว่าข้อมูลผิดกฎข้อไหน
                    "Only Free Fire tournaments can use POINTS format");
        }

        // 👉 กฎ 6: แบบ POINTS ต้องบอกจำนวนเกมทั้งหมด
        // 👉 ตรวจค่าของเงื่อนไข `request.getFormat(`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (request.getFormat() == TournamentFormat.POINTS`
        if (request.getFormat() == TournamentFormat.POINTS
                // 👉 เป็นเงื่อนไขต่อจากบรรทัด `if` ก่อนหน้า โดย `&&` หมายถึงทั้งเงื่อนไขก่อนหน้าและเงื่อนไขนี้ต้องเป็นจริง
                && request.getTotalGames() == null) {
            // 👉 โยน `ValidationException` พร้อมข้อความ `"POINTS tournaments require totalGames"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ValidationException("POINTS tournaments require totalGames");
        }

        // 👉 กฎ 7: แบบแพ้คัดออกห้ามมี totalGames (เพราะจำนวนแมตช์คำนวณจากจำนวนทีม)
        // 👉 ตรวจค่าของเงื่อนไข `request.getFormat(`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (request.getFormat() == TournamentFormat.SINGLE_ELIMINATION`
        if (request.getFormat() == TournamentFormat.SINGLE_ELIMINATION
                // 👉 เป็นเงื่อนไขต่อจากบรรทัด `if` ก่อนหน้า โดย `&&` หมายถึงทั้งเงื่อนไขก่อนหน้าและเงื่อนไขนี้ต้องเป็นจริง
                && request.getTotalGames() != null) {
            // 👉 บรรทัด `throw new ValidationException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new ValidationException(
                    // 👉 ข้อความนี้เป็นรายละเอียดของข้อผิดพลาด ช่วยบอก client ว่าข้อมูลผิดกฎข้อไหน
                    "SINGLE_ELIMINATION tournaments must not have totalGames");
        }
    }

    // 👉 เมธอดช่วย: หารายการจาก id ถ้าไม่เจอให้ throw 404 (ใช้ซ้ำหลายที่)
    private Tournament findTournament(Long id) {
        // 👉 คืนผลการค้นหาจาก Repository ให้ผู้เรียก; รายการที่ได้มาจากฐานข้อมูลตามเงื่อนไข `tournamentRepository.findById(id)`
        return tournamentRepository.findById(id)
                // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
                .orElseThrow(() -> new ResourceNotFoundException(
                        // 👉 บรรทัดนี้ `"Tournament not found: " + id));` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                        "Tournament not found: " + id));
    }

    // 👉 เมธอดช่วย: แปลง Entity เป็น Response (เลือกส่งเฉพาะฟิลด์ที่ผู้ใช้ควรเห็น)
    private TournamentResponse toResponse(Tournament tournament) {
        // 👉 คืนค่า `new TournamentResponse(` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return new TournamentResponse(
                // 👉 ส่ง id ของ entity เข้าไปเป็นฟิลด์แรกของ `TournamentResponse` เพื่อให้ client อ้างถึงรายการนี้ได้
                tournament.getId(),
                // 👉 ส่งชื่อทัวร์นาเมนต์จาก entity ไปยัง DTO ที่จะกลายเป็น JSON response
                tournament.getName(),
                // 👉 ส่งคำอธิบายของทัวร์นาเมนต์ไปยัง response
                tournament.getDescription(),
                tournament.getGame().getId(),    // 👉 ส่งแค่ id ของเกม
                // 👉 ส่งรูปแบบการแข่งขันไปยัง response เพื่อให้ client รู้ว่ารายการใช้กติกาแบบใด
                tournament.getFormat(),
                // 👉 ส่งจำนวนเกมทั้งหมดไปยัง response; ค่าอาจเป็น null หากรูปแบบการแข่งขันไม่ใช้จำนวนเกม
                tournament.getTotalGames(),
                // 👉 ส่งคะแนนต่อ kill ไปยัง response
                tournament.getPointsPerKill(),
                // 👉 ส่ง URL โลโก้ไปยัง response เพื่อให้ client นำไปแสดงรูป
                tournament.getLogoUrl(),
                // 👉 ส่งวันเริ่มแข่งขันไปยัง response
                tournament.getStartDate(),
                // 👉 ส่งวันสิ้นสุดแข่งขันไปยัง response
                tournament.getEndDate(),
                // 👉 เรียกเมธอด `tournament.getStatus())`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
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
// 👉 ประกาศคลาส `TournamentController` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TournamentController {

    private final TournamentService tournaments;   // 👉 พึ่งพา interface ไม่ใช่ตัวทำจริง

    public TournamentController(TournamentService tournaments) {   // 👉 Spring ฉีด Service ให้
        // 👉 เก็บ dependency/ค่าจาก `tournaments` ลงฟิลด์ `tournaments` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.tournaments = tournaments;
    }

    // ================= ดึงทั้งหมด / ค้นหา / กรอง =================
    @GetMapping   // 👉 GET /api/v1/tournaments
    // 👉 ประกาศสมาชิกของคลาส: `public List<TournamentResponse> list(`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public List<TournamentResponse> list(
            @RequestParam(required = false) String name,              // 👉 ?name=...   (ไม่ส่งก็ได้)
            @RequestParam(required = false) TournamentStatus status) { // 👉 ?status=... (ไม่ส่งก็ได้)

        // ตรวจสอบว่าผู้ใช้ระบุคำค้นหาชื่อหรือไม่
        boolean hasName = name != null && !name.isBlank();   // 👉 ไม่ใช่ null และไม่ใช่ข้อความว่าง/ช่องว่างล้วน

        // ถ้าไม่ได้ระบุชื่อและสถานะ ให้แสดงทัวร์นาเมนต์ทั้งหมด
        // 👉 ตรวจค่าของเงื่อนไข `!hasName && status == null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (!hasName && status == null) {`
        if (!hasName && status == null) {
            // 👉 คืนค่า `tournaments.getAll()` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
            return tournaments.getAll();
        }

        // ถ้าระบุเฉพาะชื่อ ให้ค้นหาจากชื่อ
        // 👉 ตรวจค่าของเงื่อนไข `status == null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (status == null) {`
        if (status == null) {
            // 👉 คืนค่า `tournaments.searchByName(name)` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
            return tournaments.searchByName(name);
        }

        // ถ้าระบุเฉพาะสถานะ ให้ค้นหาจากสถานะ
        // 👉 ตรวจค่าของเงื่อนไข `!hasName`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (!hasName) {`
        if (!hasName) {
            // 👉 คืนค่า `tournaments.getByStatus(status)` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
            return tournaments.getByStatus(status);
        }

        // ถ้าระบุทั้งชื่อและสถานะ ให้ค้นหาด้วยเงื่อนไขทั้งสอง
        // 👉 คืนค่า `tournaments.searchByNameAndStatus(name, status)` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return tournaments.searchByNameAndStatus(name, status);
    }

    // ================= ดึงรายการเดียว =================
    @GetMapping("/{id}")   // 👉 GET /api/v1/tournaments/5  → id = 5
    // 👉 ประกาศเมธอด `get()` โดยรับพารามิเตอร์ `@PathVariable Long id`; ชนิดผลลัพธ์ `TournamentResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public TournamentResponse get(@PathVariable Long id) {
        return tournaments.getById(id);   // 👉 ไม่พบ → Service throw 404 → ผู้ใช้ได้ 404
    }

    // ================= ดูคะแนนตามอันดับ =================
    @GetMapping("/{id}/placement-points")   // 👉 GET /api/v1/tournaments/5/placement-points
    // 👉 ประกาศสมาชิกของคลาส: `public List<PlacementPointResponse> getPlacementPoints(`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public List<PlacementPointResponse> getPlacementPoints(
            // 👉 รับค่าจากส่วน `{id}` ใน URL แล้วแปลงเป็นตัวแปร Java `id`
            @PathVariable Long id) {

        // 👉 คืนค่า `tournaments.getPlacementPoints(id)` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return tournaments.getPlacementPoints(id);
    }

    // ================= สร้างรายการ =================
    @PostMapping   // 👉 POST /api/v1/tournaments
    // 👉 ประกาศสมาชิกของคลาส: `public ResponseEntity<TournamentResponse> create(`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public ResponseEntity<TournamentResponse> create(
            @Valid @RequestBody TournamentRequest request) {   // 👉 แปลง JSON ในตัวคำขอ → TournamentRequest

        // 👉 กำหนดค่าจาก `tournaments.create(request)` ให้ `tournament`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        TournamentResponse tournament = tournaments.create(request);

        // 👉 สร้าง URL ของรายการใหม่ = URL ปัจจุบัน + "/{id}"  เช่น http://host/api/v1/tournaments/7
        URI location = ServletUriComponentsBuilder
                // 👉 เริ่มประกอบ URI โดยใช้ URL ของ HTTP request ปัจจุบันเป็นฐาน
                .fromCurrentRequest()
                // 👉 ต่อ path เพิ่มท้าย URI; `/{id}` เป็นช่องว่างที่จะแทนด้วย id ของรายการที่เพิ่งสร้าง
                .path("/{id}")
                // 👉 แทน placeholder ใน path ด้วยค่าที่ส่งเข้าไป เช่น `{id}` ถูกแทนด้วย `tournament.id()`
                .buildAndExpand(tournament.id())
                // 👉 บรรทัด `.toUri();` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .toUri();

        // 👉 ตอบ 201 Created พร้อม header "Location" และ body เป็น JSON ของรายการที่สร้าง
        // 👉 คืนค่า `ResponseEntity` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return ResponseEntity
                // 👉 กำหนด HTTP status เป็น 201 Created และเตรียมใส่ Location header ที่ชี้ไปยัง resource ใหม่
                .created(location)
                // 👉 บรรทัด `.body(tournament);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .body(tournament);
    }

    // ================= แก้ไข =================
    @PutMapping("/{id}")   // 👉 PUT /api/v1/tournaments/5
    // 👉 ประกาศสมาชิกของคลาส: `public TournamentResponse update(`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public TournamentResponse update(
            // 👉 รับค่าจากส่วน `{id}` ใน URL แล้วแปลงเป็นตัวแปร Java `id`
            @PathVariable Long id,
            // 👉 ขอให้ Spring ตรวจ validation ของ object ที่รับเข้ามาก่อนเรียกเมธอด; จะมีผลเมื่อ DTO มี validation constraints
            @Valid @RequestBody TournamentRequest request) {

        // 👉 คืนค่า `tournaments.update(id, request)` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return tournaments.update(id, request);
    }

    // ================= ลบ =================
    @DeleteMapping("/{id}")   // 👉 DELETE /api/v1/tournaments/5
    // 👉 ประกาศเมธอด `delete()` โดยรับพารามิเตอร์ `@PathVariable Long id`; ชนิดผลลัพธ์ `ResponseEntity<Void>` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        // 👉 เรียก `tournaments.delete(id)` เพื่อสั่ง Repository ลบ entity ที่ระบุออกจากฐานข้อมูล
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

// 👉 ประกาศ enum เพื่อกำหนดชุดสถานะที่ระบบยอมรับได้ ผู้เรียกต้องเลือกหนึ่งในค่าคงที่ด้านล่าง แทนการส่งข้อความอะไรก็ได้
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

// 👉 บรรทัดนี้ `@Service` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@Service
// 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
@Transactional
// 👉 ประกาศคลาส `MatchResultServiceImpl` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class MatchResultServiceImpl implements MatchResultService {

    private final MatchRepository matches;               // 👉 ตารางแมตช์
    private final MatchResultRepository results;         // 👉 ตารางผลการแข่ง
    private final MatchResultMapper mapper;              // 👉 ตัวแปลง Entity → Response
    private final ApplicationEventPublisher events;      // 👉 ตัวประกาศ "เหตุการณ์" (Observer Pattern)

    // 👉 ประกาศสมาชิกของคลาส: `public MatchResultServiceImpl(MatchRepository matches, MatchResultRepository results,`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public MatchResultServiceImpl(MatchRepository matches, MatchResultRepository results,
            // 👉 บรรทัดนี้ `MatchResultMapper mapper, ApplicationEventPublisher events) {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            MatchResultMapper mapper, ApplicationEventPublisher events) {
        // 👉 เก็บ dependency/ค่าจาก `matches` ลงฟิลด์ `matches` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.matches = matches;
        // 👉 เก็บ dependency/ค่าจาก `results` ลงฟิลด์ `results` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.results = results;
        // 👉 เก็บ dependency/ค่าจาก `mapper` ลงฟิลด์ `mapper` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.mapper = mapper;
        // 👉 เก็บ dependency/ค่าจาก `events` ลงฟิลด์ `events` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.events = events;
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `record()` โดยรับพารามิเตอร์ `Long matchId, CreateMatchResultRequest request`; ชนิดผลลัพธ์ `MatchResultResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public MatchResultResponse record(Long matchId, CreateMatchResultRequest request) {
        // 1. แมตช์ต้องมีอยู่จริง → 404
        // 👉 ประกาศตัวแปร `match` แล้วค้นหาข้อมูลด้วย `matches.findById(matchId)`; Repository คืน `Optional` เพราะรหัสนั้นอาจมีหรือไม่มีในฐานข้อมูล
        Match match = matches.findById(matchId)
                // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
                .orElseThrow(() -> new ResourceNotFoundException("Match not found: " + matchId));

        // 2. แมตช์ต้องยังไม่มีผล → 409
        // 👉 กันบันทึกผลซ้ำ
        // 👉 ตรวจว่ามีผลการแข่งขันของแมตช์นี้บันทึกไว้แล้วหรือไม่; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (results.existsByMatchId(matchId)) {`
        if (results.existsByMatchId(matchId)) {
            // 👉 โยน `BusinessException` พร้อมข้อความ `"Match already has a result"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new BusinessException("Match already has a result");
        }

        // 3. ต้องมีทีมครบทั้งสองฝั่ง → 409
        // 👉 ประกาศตัวแปร `teamA` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `match.getTeamA()` เพื่อใช้ในขั้นตอนถัดไป
        Team teamA = match.getTeamA();
        // 👉 ประกาศตัวแปร `teamB` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `match.getTeamB()` เพื่อใช้ในขั้นตอนถัดไป
        Team teamB = match.getTeamB();
        // 👉 ตรวจว่าทีม A หรือทีม B ยังว่างอยู่หรือไม่; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (teamA == null || teamB == null) {`
        if (teamA == null || teamB == null) {
            // 👉 โยน `BusinessException` พร้อมข้อความ `"Both teams must be set before recording a result"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new BusinessException("Both teams must be set before recording a result");
        }

        // 4. Match ต้องอยู่ในสถานะ SCHEDULED ก่อนบันทึกผล
        // 👉 ★ ส่วนที่เพิ่มใน commit ของคุณ ★
        //    match.getStatus() เป็น String ("PENDING"/"SCHEDULED"/"COMPLETED")
        //    จึงเทียบกับ MatchStatus.SCHEDULED.name() (แปลง enum เป็นข้อความ "SCHEDULED")
        //    ถ้าไม่ใช่ SCHEDULED (เช่น ยัง PENDING หรือ COMPLETED แล้ว) → 409
        // 👉 ตรวจค่าของเงื่อนไข `!MatchStatus.SCHEDULED.name().equals(match.getStatus())`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (!MatchStatus.SCHEDULED.name().equals(match.getStatus())) {`
        if (!MatchStatus.SCHEDULED.name().equals(match.getStatus())) {
            // 👉 บรรทัด `throw new BusinessException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new BusinessException(
                    // 👉 บรรทัดนี้ `"Match must be SCHEDULED before recording a result");` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                    "Match must be SCHEDULED before recording a result");
        }

        // 5. คะแนนไม่ติดลบ ตรวจแล้วใน DTO ด้วย @Min(0) → 400
        // 👉 กำหนดค่าจาก `request.teamAScore()` ให้ `scoreA`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        int scoreA = request.teamAScore();
        // 👉 กำหนดค่าจาก `request.teamBScore()` ให้ `scoreB`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        int scoreB = request.teamBScore();

        // 6. แพ้คัดออกห้ามเสมอ → 400
        // 👉 ตรวจค่าของเงื่อนไข `scoreA == scoreB`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (scoreA == scoreB) {`
        if (scoreA == scoreB) {
            // 👉 โยน `ValidationException` พร้อมข้อความ `"Single elimination match cannot end in a draw"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ValidationException("Single elimination match cannot end in a draw");
        }

        // 7. ผู้ชนะต้องเป็นทีมในแมตช์ → 400
        // 👉 กำหนดค่าจาก `request.winnerTeamId()` ให้ `winnerId`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        Long winnerId = request.winnerTeamId();
        // 👉 ตรวจค่าของเงื่อนไข `!winnerId.equals(teamA.getId()) && !winnerId.equals(teamB.getId())`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (!winnerId.equals(teamA.getId()) && !winnerId.equals(teamB.getId())) {`
        if (!winnerId.equals(teamA.getId()) && !winnerId.equals(teamB.getId())) {
            // 👉 โยน `ValidationException` พร้อมข้อความ `"Winner must be one of the teams in this match"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ValidationException("Winner must be one of the teams in this match");
        }

        // 8. ผู้ชนะต้องมีคะแนนมากกว่า → 400
        // 👉 คำนวณผู้ชนะจากคะแนนเอง แล้วเทียบกับ winnerTeamId ที่ผู้ใช้ส่งมา ต้องตรงกัน
        // 👉 ประกาศตัวแปร `winner` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `scoreA > scoreB ? teamA : teamB` เพื่อใช้ในขั้นตอนถัดไป
        Team winner = scoreA > scoreB ? teamA : teamB;
        // 👉 ตรวจค่าของเงื่อนไข `!winner.getId().equals(winnerId)`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (!winner.getId().equals(winnerId)) {`
        if (!winner.getId().equals(winnerId)) {
            // 👉 โยน `ValidationException` พร้อมข้อความ `"Winner must have the higher score"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ValidationException("Winner must have the higher score");
        }

        // 👉 ผ่านทุกข้อ → สร้างผลการแข่งและบันทึก
        // 👉 ประกาศตัวแปร `result` เพื่ออ้างถึง object `MatchResult` ที่เพิ่งสร้าง หรือข้อมูลที่ได้จากนิพจน์ `new MatchResult()`
        MatchResult result = new MatchResult();
        // 👉 เรียก `result.setMatch(match)` เพื่อกำหนดค่า ฟิลด์ match; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        result.setMatch(match);
        // 👉 เรียก `result.setTeamAScore(scoreA)` เพื่อกำหนดค่า ฟิลด์ teamAScore; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        result.setTeamAScore(scoreA);
        // 👉 เรียก `result.setTeamBScore(scoreB)` เพื่อกำหนดค่า ฟิลด์ teamBScore; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        result.setTeamBScore(scoreB);
        // 👉 เรียก `result.setWinnerTeam(winner)` เพื่อกำหนดค่า ฟิลด์ winnerTeam; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        result.setWinnerTeam(winner);
        // 👉 เรียก `result.setCreatedAt(LocalDateTime.now())` เพื่อกำหนดค่า เวลาที่สร้างรายการ; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        result.setCreatedAt(LocalDateTime.now());
        // 👉 ประกาศตัวแปร `saved` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `results.save(result)` เพื่อใช้ในขั้นตอนถัดไป
        MatchResult saved = results.save(result);

        // 👉 แมตช์นี้จบแล้ว เปลี่ยนสถานะเป็น COMPLETED
        // 👉 กำหนดสถานะใหม่ให้กับข้อมูล
        match.setStatus(MatchStatus.COMPLETED.name());

        // Observer: ประกาศว่ามีผลใหม่ ให้ Listener ส่งผู้ชนะไปแมตช์ถัดไป
        // 👉 ประกาศเหตุการณ์ "บันทึกผลแล้ว" Listener (อีกคลาส) จะรับไปเลื่อนผู้ชนะเข้ารอบถัดไป
        // 👉 ประกาศ Event เพื่อให้ Listener ที่เกี่ยวข้องทำงานต่อ
        events.publishEvent(new MatchResultRecordedEvent(match.getId(), winner.getId()));

        // 👉 แปลง entity ด้วย `mapper.toResponse(saved)` แล้วคืน DTO ให้ Controller; DTO คือรูปข้อมูลที่ API ส่งกลับไปยังผู้ใช้
        return mapper.toResponse(saved);
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด `get()` โดยรับพารามิเตอร์ `Long matchId`; ชนิดผลลัพธ์ `MatchResultResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public MatchResultResponse get(Long matchId) {
        // 👉 คืนผลการค้นหาจาก Repository ให้ผู้เรียก; รายการที่ได้มาจากฐานข้อมูลตามเงื่อนไข `results.findByMatchId(matchId)`
        return results.findByMatchId(matchId)
                // 👉 บรรทัด `.map(mapper::toResponse)` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .map(mapper::toResponse)
                // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
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

// 👉 บรรทัดนี้ `@RestController` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@RestController
// 👉 บรรทัด `@RequestMapping("/api/v1/players")` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
@RequestMapping("/api/v1/players")
// 👉 ประกาศคลาส `PlayerController` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class PlayerController {

    // 👉 บรรทัด `private final PlayerService players;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final PlayerService players;

    // 👉 บรรทัด `public PlayerController(PlayerService players) {` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    public PlayerController(PlayerService players) {
        // 👉 เก็บ dependency/ค่าจาก `players` ลงฟิลด์ `players` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.players = players;
    }

    // 👉 ★ ตัวกรองชื่อ ★  ?name=... (ไม่ส่งก็ได้)
    //    Pageable = ตัวบอกว่าจะขอหน้าที่เท่าไร ขนาดหน้าเท่าไร
    //    @PageableDefault(size = 20, sort = "name") = ถ้าไม่ระบุ ให้หน้าละ 20 รายการ เรียงตามชื่อ
    //    ผู้ใช้เปลี่ยนได้ เช่น ?page=1&size=10
    @GetMapping
    // 👉 ประกาศสมาชิกของคลาส: `public PageResponse<PlayerResponse> list(@RequestParam(required = false) String name,`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public PageResponse<PlayerResponse> list(@RequestParam(required = false) String name,
            // 👉 บรรทัดนี้ `@PageableDefault(size = 20, sort = "name") Pageable pageable) {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            @PageableDefault(size = 20, sort = "name") Pageable pageable) {
        // 👉 คืนค่า `PageResponse.from(players.list(name, pageable))` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return PageResponse.from(players.list(name, pageable));
    }

    // 👉 บรรทัด `@GetMapping("/{id}")` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
    @GetMapping("/{id}")
    // 👉 ประกาศเมธอด `get()` โดยรับพารามิเตอร์ `@PathVariable Long id`; ชนิดผลลัพธ์ `PlayerResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public PlayerResponse get(@PathVariable Long id) {
        // 👉 คืนค่า `players.get(id)` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return players.get(id);
    }

    // 👉 บรรทัดนี้ `@PostMapping` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    @PostMapping
    // 👉 ประกาศเมธอด `create()` โดยรับพารามิเตอร์ `@Valid @RequestBody PlayerRequest request`; ชนิดผลลัพธ์ `ResponseEntity<PlayerResponse>` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public ResponseEntity<PlayerResponse> create(@Valid @RequestBody PlayerRequest request) {
        // 👉 กำหนดค่าจาก `players.create(request)` ให้ `player`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        PlayerResponse player = players.create(request);
        // 👉 บรรทัด `URI location = ServletUriComponentsBuilder.fromCurrentRequest().path("/{id}")` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
        URI location = ServletUriComponentsBuilder.fromCurrentRequest().path("/{id}")
                // 👉 บรรทัด `.buildAndExpand(player.id()).toUri();` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                .buildAndExpand(player.id()).toUri();
        // 👉 คืนค่า `ResponseEntity.created(location).body(player)` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return ResponseEntity.created(location).body(player);
    }

    // 👉 บรรทัด `@PutMapping("/{id}")` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
    @PutMapping("/{id}")
    // 👉 ประกาศเมธอด `update()` โดยรับพารามิเตอร์ `@PathVariable Long id, @Valid @RequestBody PlayerRequest request`; ชนิดผลลัพธ์ `PlayerResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public PlayerResponse update(@PathVariable Long id, @Valid @RequestBody PlayerRequest request) {
        // 👉 คืนค่า `players.update(id, request)` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return players.update(id, request);
    }

    // 👉 บรรทัด `@DeleteMapping("/{id}")` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
    @DeleteMapping("/{id}")
    // 👉 ประกาศเมธอด `delete()` โดยรับพารามิเตอร์ `@PathVariable Long id`; ชนิดผลลัพธ์ `ResponseEntity<Void>` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        // 👉 เรียก `players.delete(id)` เพื่อสั่ง Repository ลบ entity ที่ระบุออกจากฐานข้อมูล
        players.delete(id);
        // 👉 คืนค่า `ResponseEntity.noContent().build()` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
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

// 👉 บรรทัดนี้ `@Service` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@Service
// 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
@Transactional
// 👉 ประกาศคลาส `PlayerServiceImpl` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class PlayerServiceImpl implements PlayerService {

    // 👉 บรรทัด `private final PlayerRepository players;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final PlayerRepository players;
    // 👉 บรรทัด `private final TeamRepository teams;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final TeamRepository teams;
    // 👉 บรรทัด `private final PlayerMapper mapper;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final PlayerMapper mapper;

    // 👉 บรรทัด `public PlayerServiceImpl(PlayerRepository players, TeamRepository teams, PlayerMapper mapper) {` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    public PlayerServiceImpl(PlayerRepository players, TeamRepository teams, PlayerMapper mapper) {
        // 👉 เก็บ dependency/ค่าจาก `players` ลงฟิลด์ `players` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.players = players;
        // 👉 เก็บ dependency/ค่าจาก `teams` ลงฟิลด์ `teams` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.teams = teams;
        // 👉 เก็บ dependency/ค่าจาก `mapper` ลงฟิลด์ `mapper` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.mapper = mapper;
    }

    // 👉 ★ ตัวกรองชื่อ ★
    @Override
    // 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด `list()` โดยรับพารามิเตอร์ `String name, Pageable pageable`; ชนิดผลลัพธ์ `Page<PlayerResponse>` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public Page<PlayerResponse> list(String name, Pageable pageable) {
        // 👉 ถ้าไม่ได้ส่งชื่อ (null หรือว่าง) → ดึงทุกคน (แบ่งหน้า)
        //    ถ้าส่งชื่อ → ค้นชื่อที่มีคำนั้น ไม่สนตัวพิมพ์ (ตัดช่องว่างหัวท้ายก่อน)
        //    รูปแบบ  เงื่อนไข ? ค่าถ้าจริง : ค่าถ้าเท็จ  (ternary operator)
        Page<Player> result = name == null || name.isBlank()
                // 👉 บรรทัด `? players.findAll(pageable)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
                ? players.findAll(pageable)
                // 👉 บรรทัด `: players.findByNameContainingIgnoreCase(name.trim(), pageable);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                : players.findByNameContainingIgnoreCase(name.trim(), pageable);
        return result.map(mapper::toResponse);   // 👉 แปลงทุกคนในหน้านั้นเป็น Response
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด `get()` โดยรับพารามิเตอร์ `Long id`; ชนิดผลลัพธ์ `PlayerResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public PlayerResponse get(Long id) {
        // 👉 แปลง entity ด้วย `mapper.toResponse(findPlayer(id))` แล้วคืน DTO ให้ Controller; DTO คือรูปข้อมูลที่ API ส่งกลับไปยังผู้ใช้
        return mapper.toResponse(findPlayer(id));
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `create()` โดยรับพารามิเตอร์ `PlayerRequest request`; ชนิดผลลัพธ์ `PlayerResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public PlayerResponse create(PlayerRequest request) {
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `findOptionalTeam(request.teamId())` เพื่อใช้ในขั้นตอนถัดไป
        Team team = findOptionalTeam(request.teamId());
        // 👉 สร้าง object ชนิด `Player` ใหม่ แล้วเก็บไว้ในตัวแปร `player`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        Player player = new Player();
        // 👉 เรียก `player.setName(request.name().trim())` เพื่อกำหนดค่า ชื่อรายการที่ผ่านการตัดช่องว่างแล้ว; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        player.setName(request.name().trim());
        // 👉 เรียก `player.setRole(request.role().trim())` เพื่อกำหนดค่า ฟิลด์ role; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        player.setRole(request.role().trim());
        // 👉 เรียก `player.setDescription(request.description())` เพื่อกำหนดค่า คำอธิบายรายการจาก request; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        player.setDescription(request.description());
        // 👉 เรียก `player.setTeam(team)` เพื่อกำหนดค่า ฟิลด์ team; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        player.setTeam(team);
        // 👉 เรียก `player.setCreatedAt(LocalDateTime.now())` เพื่อกำหนดค่า เวลาที่สร้างรายการ; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        player.setCreatedAt(LocalDateTime.now());
        // 👉 แปลง entity ด้วย `mapper.toResponse(players.saveAndFlush(player))` แล้วคืน DTO ให้ Controller; DTO คือรูปข้อมูลที่ API ส่งกลับไปยังผู้ใช้
        return mapper.toResponse(players.saveAndFlush(player));
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `update()` โดยรับพารามิเตอร์ `Long id, PlayerRequest request`; ชนิดผลลัพธ์ `PlayerResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public PlayerResponse update(Long id, PlayerRequest request) {
        // 👉 กำหนดค่าจาก `findPlayer(id)` ให้ `player`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        Player player = findPlayer(id);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `findOptionalTeam(request.teamId())` เพื่อใช้ในขั้นตอนถัดไป
        Team team = findOptionalTeam(request.teamId());
        // 👉 เรียก `player.setName(request.name().trim())` เพื่อกำหนดค่า ชื่อรายการที่ผ่านการตัดช่องว่างแล้ว; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        player.setName(request.name().trim());
        // 👉 เรียก `player.setRole(request.role().trim())` เพื่อกำหนดค่า ฟิลด์ role; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        player.setRole(request.role().trim());
        // 👉 เรียก `player.setDescription(request.description())` เพื่อกำหนดค่า คำอธิบายรายการจาก request; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        player.setDescription(request.description());
        // 👉 เรียก `player.setTeam(team)` เพื่อกำหนดค่า ฟิลด์ team; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        player.setTeam(team);
        // 👉 แปลง entity ด้วย `mapper.toResponse(players.saveAndFlush(player))` แล้วคืน DTO ให้ Controller; DTO คือรูปข้อมูลที่ API ส่งกลับไปยังผู้ใช้
        return mapper.toResponse(players.saveAndFlush(player));
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `delete()` โดยรับพารามิเตอร์ `Long id`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void delete(Long id) {
        // 👉 เรียก `players.delete(findPlayer(id))` เพื่อสั่ง Repository ลบ entity ที่ระบุออกจากฐานข้อมูล
        players.delete(findPlayer(id));
        // 👉 สั่งให้ JPA ส่งคำสั่งค้างไปยังฐานข้อมูล
        players.flush();
    }

    // 👉 ประกาศสมาชิกของคลาส: `private Player findPlayer(Long id)`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    private Player findPlayer(Long id) {
        // 👉 คืนผลการค้นหาจาก Repository ให้ผู้เรียก; รายการที่ได้มาจากฐานข้อมูลตามเงื่อนไข `players.findById(id)`
        return players.findById(id)
                // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
                .orElseThrow(() -> new ResourceNotFoundException("Player not found: " + id));
    }

    // 👉 ผู้เล่นไม่จำเป็นต้องมีทีม: ถ้า teamId เป็น null คืน null ถ้ามีแต่หาไม่เจอ → 404
    private Team findOptionalTeam(Long teamId) {
        // 👉 ตรวจค่าของเงื่อนไข `teamId == null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (teamId == null) {`
        if (teamId == null) {
            // 👉 คืนค่า `null` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
            return null;
        }
        // 👉 คืนผลการค้นหาจาก Repository ให้ผู้เรียก; รายการที่ได้มาจากฐานข้อมูลตามเงื่อนไข `teams.findById(teamId)`
        return teams.findById(teamId)
                // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
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

// 👉 ประกาศสัญญา `PlayerRepository` ระบุว่าโค้ดที่นำไป implement ต้องมีเมธอดตามที่ประกาศไว้ แต่ยังไม่บอกวิธีทำงานจริง
public interface PlayerRepository extends JpaRepository<Player, Long> {

    // ค้นหา Player ทั้งหมดใน Team
    // 👉 WHERE team_id = ?   (Page<> = ผลลัพธ์แบบแบ่งหน้า)
    // 👉 บรรทัด `Page<Player> findByTeamId(Long teamId, Pageable pageable);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    Page<Player> findByTeamId(Long teamId, Pageable pageable);

    // ค้นหา Player ใน Team ตามชื่อ
    // 👉 WHERE team_id = ? AND name LIKE %?%  (ไม่สนตัวพิมพ์)
    Page<Player> findByTeamIdAndNameContainingIgnoreCase(
            // 👉 เป็นค่าหรือพารามิเตอร์ `Long teamId` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            Long teamId,
            // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
            String name,
            // 👉 บรรทัดนี้ `Pageable pageable);` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            Pageable pageable);

    // ค้นหา Player ใน Team ตาม Role
    // 👉 WHERE team_id = ? AND role = ?  (ตรงทั้งคำ แต่ไม่สนตัวพิมพ์ เช่น "support" = "Support")
    Page<Player> findByTeamIdAndRoleIgnoreCase(
            // 👉 เป็นค่าหรือพารามิเตอร์ `Long teamId` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            Long teamId,
            // 👉 เป็นค่าหรือพารามิเตอร์ `String role` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            String role,
            // 👉 บรรทัดนี้ `Pageable pageable);` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            Pageable pageable);

    // ค้นหา Player ใน Team ตามทั้งชื่อและ Role
    // 👉 ใช้เมื่อผู้ใช้ส่งทั้ง name และ role
    Page<Player> findByTeamIdAndNameContainingIgnoreCaseAndRoleIgnoreCase(
            // 👉 เป็นค่าหรือพารามิเตอร์ `Long teamId` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            Long teamId,
            // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
            String name,
            // 👉 เป็นค่าหรือพารามิเตอร์ `String role` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            String role,
            // 👉 บรรทัดนี้ `Pageable pageable);` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            Pageable pageable);

    // นับจำนวน Player ใน Team
    // 👉 ใช้ในกฎ "ทีมต้องมีผู้เล่นอย่างน้อย N คน" (หมวด 6)
    // 👉 บรรทัด `long countByTeamId(Long teamId);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    long countByTeamId(Long teamId);

    // ค้นหา Player จากชื่อ โดยไม่จำกัด Team
    // 👉 ใช้กับตัวกรองชื่อที่ /api/v1/players
    Page<Player> findByNameContainingIgnoreCase(
            // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
            String name,
            // 👉 บรรทัดนี้ `Pageable pageable);` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
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

// 👉 บรรทัดนี้ `@RestController` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@RestController
@RequestMapping("/api/v1/teams/{teamId}/players")   // 👉 {teamId} อยู่ใน path เช่น /api/v1/teams/3/players
// 👉 ประกาศคลาส `TeamMembershipController` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TeamMembershipController {

    // 👉 บรรทัด `private final TeamMembershipService memberships;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final TeamMembershipService memberships;

    // 👉 บรรทัด `public TeamMembershipController(TeamMembershipService memberships) {` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    public TeamMembershipController(TeamMembershipService memberships) {
        // 👉 เก็บ dependency/ค่าจาก `memberships` ลงฟิลด์ `memberships` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.memberships = memberships;
    }

    // 👉 ★ ตัวกรองชื่อ + ตำแหน่ง ★  ส่งแค่ตัวใดตัวหนึ่ง หรือทั้งคู่ หรือไม่ส่งเลยก็ได้
    @GetMapping
    // 👉 ประกาศสมาชิกของคลาส: `public PageResponse<TeamPlayerResponse> list(@PathVariable Long teamId,`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public PageResponse<TeamPlayerResponse> list(@PathVariable Long teamId,
            // 👉 เป็นค่าหรือพารามิเตอร์ `@RequestParam(required = false) String name` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            @RequestParam(required = false) String name,
            // 👉 เป็นค่าหรือพารามิเตอร์ `@RequestParam(required = false) String role` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            @RequestParam(required = false) String role,
            // 👉 บรรทัดนี้ `@PageableDefault(size = 20, sort = "name") Pageable pageable) {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            @PageableDefault(size = 20, sort = "name") Pageable pageable) {
        // 👉 คืนค่า `PageResponse.from(memberships.listPlayers(teamId, name, role, pageable))` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return PageResponse.from(memberships.listPlayers(teamId, name, role, pageable));
    }

    // 👉 บรรทัด `@PutMapping("/{playerId}")` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
    @PutMapping("/{playerId}")
    // 👉 ประกาศเมธอด `add()` โดยรับพารามิเตอร์ `@PathVariable Long teamId, @PathVariable Long playerId`; ชนิดผลลัพธ์ `TeamPlayerResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public TeamPlayerResponse add(@PathVariable Long teamId, @PathVariable Long playerId) {
        // 👉 คืนค่า `memberships.addPlayer(teamId, playerId)` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return memberships.addPlayer(teamId, playerId);
    }

    // 👉 บรรทัด `@DeleteMapping("/{playerId}")` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
    @DeleteMapping("/{playerId}")
    // 👉 ประกาศเมธอด `remove()` โดยรับพารามิเตอร์ `@PathVariable Long teamId, @PathVariable Long playerId`; ชนิดผลลัพธ์ `ResponseEntity<Void>` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public ResponseEntity<Void> remove(@PathVariable Long teamId, @PathVariable Long playerId) {
        // 👉 เรียกเมธอด `memberships.removePlayer(teamId, playerId)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
        memberships.removePlayer(teamId, playerId);
        // 👉 คืนค่า `ResponseEntity.noContent().build()` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
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

// 👉 บรรทัดนี้ `@Service` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@Service
// 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
@Transactional
// 👉 ประกาศคลาส `TeamMembershipServiceImpl` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TeamMembershipServiceImpl implements TeamMembershipService {

    // 👉 บรรทัด `private final TeamRepository teams;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final TeamRepository teams;
    // 👉 บรรทัด `private final PlayerRepository players;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final PlayerRepository players;
    // 👉 บรรทัด `private final TeamMembershipMapper mapper;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final TeamMembershipMapper mapper;

    // 👉 ประกาศสมาชิกของคลาส: `public TeamMembershipServiceImpl(TeamRepository teams, PlayerRepository players,`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public TeamMembershipServiceImpl(TeamRepository teams, PlayerRepository players,
            // 👉 บรรทัดนี้ `TeamMembershipMapper mapper) {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            TeamMembershipMapper mapper) {
        // 👉 เก็บ dependency/ค่าจาก `teams` ลงฟิลด์ `teams` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.teams = teams;
        // 👉 เก็บ dependency/ค่าจาก `players` ลงฟิลด์ `players` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.players = players;
        // 👉 เก็บ dependency/ค่าจาก `mapper` ลงฟิลด์ `mapper` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.mapper = mapper;
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 กำหนดขอบเขต transaction ให้เมธอดที่ตามมา; `readOnly = true` บอกว่าเมธอดนี้เน้นอ่านข้อมูล ไม่ได้ตั้งใจแก้ฐานข้อมูล
    @Transactional(readOnly = true)
    // 👉 ประกาศเมธอด `listPlayers()` โดยรับพารามิเตอร์ `Long teamId, String name, String role, Pageable pageable`; ชนิดผลลัพธ์ `Page<TeamPlayerResponse>` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public Page<TeamPlayerResponse> listPlayers(Long teamId, String name, String role, Pageable pageable) {
        findTeam(teamId);   // 👉 ทีมไม่มีอยู่จริง → 404

        // 👉 เช็คว่าผู้ใช้ส่ง name / role มาจริงหรือไม่ (ไม่ใช่ null และไม่ว่าง)
        // 👉 สร้างตัวแปร `hasName` เพื่อเก็บผลการตรวจสอบ `name != null && !name.isBlank()`; true หมายถึงเงื่อนไขเป็นจริง และ false หมายถึงไม่จริง
        boolean hasName = name != null && !name.isBlank();
        // 👉 สร้างตัวแปร `hasRole` เพื่อเก็บผลการตรวจสอบ `role != null && !role.isBlank()`; true หมายถึงเงื่อนไขเป็นจริง และ false หมายถึงไม่จริง
        boolean hasRole = role != null && !role.isBlank();

        // 👉 เลือกเมธอดค้นหาตาม 4 กรณี (ชื่อ?/ตำแหน่ง?) ตารางสรุปด้านล่าง
        Page<Player> result;
        // 👉 ตรวจค่าของเงื่อนไข `hasName && hasRole`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (hasName && hasRole) {`
        if (hasName && hasRole) {
            // 👉 บรรทัดนี้ `result = players.findByTeamIdAndNameContainingIgnoreCaseAndRoleIgnoreCase(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            result = players.findByTeamIdAndNameContainingIgnoreCaseAndRoleIgnoreCase(
                    // 👉 บรรทัด `teamId, name.trim(), role.trim(), pageable);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                    teamId, name.trim(), role.trim(), pageable);
        } else if (hasName) {
            // 👉 กำหนดค่าจาก `players.findByTeamIdAndNameContainingIgnoreCase(teamId, name.trim(), pageable)` ให้ `result`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
            result = players.findByTeamIdAndNameContainingIgnoreCase(teamId, name.trim(), pageable);
        } else if (hasRole) {
            // 👉 กำหนดค่าจาก `players.findByTeamIdAndRoleIgnoreCase(teamId, role.trim(), pageable)` ให้ `result`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
            result = players.findByTeamIdAndRoleIgnoreCase(teamId, role.trim(), pageable);
        } else {
            // 👉 กำหนดค่าจาก `players.findByTeamId(teamId, pageable)` ให้ `result`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
            result = players.findByTeamId(teamId, pageable);
        }
        // 👉 คืนค่า `result.map(mapper::toResponse)` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return result.map(mapper::toResponse);
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `addPlayer()` โดยรับพารามิเตอร์ `Long teamId, Long playerId`; ชนิดผลลัพธ์ `TeamPlayerResponse` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public TeamPlayerResponse addPlayer(Long teamId, Long playerId) {
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `findTeam(teamId)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = findTeam(teamId);
        // 👉 กำหนดค่าจาก `findPlayer(playerId)` ให้ `player`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        Player player = findPlayer(playerId);
        // 👉 เรียก `player.setTeam(team)` เพื่อกำหนดค่า ฟิลด์ team; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        player.setTeam(team);
        // 👉 แปลง entity ด้วย `mapper.toResponse(players.save(player))` แล้วคืน DTO ให้ Controller; DTO คือรูปข้อมูลที่ API ส่งกลับไปยังผู้ใช้
        return mapper.toResponse(players.save(player));
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `removePlayer()` โดยรับพารามิเตอร์ `Long teamId, Long playerId`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void removePlayer(Long teamId, Long playerId) {
        // 👉 บรรทัด `findTeam(teamId);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        findTeam(teamId);
        // 👉 กำหนดค่าจาก `findPlayer(playerId)` ให้ `player`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        Player player = findPlayer(playerId);
        // 👉 ตรวจค่าของเงื่อนไข `player.getTeam() == null || !teamId.equals(player.getTeam().getId())`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (player.getTeam() == null || !teamId.equals(player.getTeam().getId())) {`
        if (player.getTeam() == null || !teamId.equals(player.getTeam().getId())) {
            // 👉 โยน `ResourceNotFoundException` พร้อมข้อความ `"Player is not in this team: " + playerId` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ResourceNotFoundException("Player is not in this team: " + playerId);
        }
        // 👉 เรียก `player.setTeam(null)` เพื่อกำหนดค่า ฟิลด์ team; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        player.setTeam(null);
        // 👉 เรียก `players.save(player)` เพื่อให้ Repository บันทึก entity; ถ้าเป็นข้อมูลใหม่จะ INSERT และถ้าเป็น entity ที่มีอยู่แล้วอาจ UPDATE
        players.save(player);
    }

    // 👉 ประกาศสมาชิกของคลาส: `private Team findTeam(Long id)`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    private Team findTeam(Long id) {
        // 👉 คืนผลการค้นหาจาก Repository ให้ผู้เรียก; รายการที่ได้มาจากฐานข้อมูลตามเงื่อนไข `teams.findById(id)`
        return teams.findById(id)
                // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
                .orElseThrow(() -> new ResourceNotFoundException("Team not found: " + id));
    }

    // 👉 ประกาศสมาชิกของคลาส: `private Player findPlayer(Long id)`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    private Player findPlayer(Long id) {
        // 👉 คืนผลการค้นหาจาก Repository ให้ผู้เรียก; รายการที่ได้มาจากฐานข้อมูลตามเงื่อนไข `players.findById(id)`
        return players.findById(id)
                // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
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

// 👉 ประกาศสัญญา `TeamJoinRule` ระบุว่าโค้ดที่นำไป implement ต้องมีเมธอดตามที่ประกาศไว้ แต่ยังไม่บอกวิธีทำงานจริง
public interface TeamJoinRule {

    // 👉 ตรวจกฎ: ผ่าน = ไม่ทำอะไร / ไม่ผ่าน = throw exception
    // 👉 บรรทัด `void validate(Team team, Tournament tournament);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    void validate(Team team, Tournament tournament);

    // 👉 กำหนด "ด่านถัดไป" ที่จะส่งต่อให้เมื่อด่านนี้ผ่าน
    // 👉 บรรทัด `void setNext(TeamJoinRule nextRule);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
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
// 👉 ประกาศคลาส `TeamExistsRule` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TeamExistsRule implements TeamJoinRule {

    // 👉 บรรทัด `private final TeamRepository teamRepository;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final TeamRepository teamRepository;
    private TeamJoinRule nextRule;   // 👉 ด่านถัดไป (ตอนแรกยังไม่มี = null จนกว่า Chain จะมาต่อให้)

    // 👉 บรรทัด `public TeamExistsRule(TeamRepository teamRepository) {` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    public TeamExistsRule(TeamRepository teamRepository) {
        // 👉 เก็บ dependency/ค่าจาก `teamRepository` ลงฟิลด์ `teamRepository` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.teamRepository = teamRepository;
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `validate()` โดยรับพารามิเตอร์ `Team team, Tournament tournament`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void validate(Team team, Tournament tournament) {
        // 👉 existsById = ถามฐานข้อมูลว่ามีทีม id นี้ไหม  ไม่มี → 404
        // 👉 ตรวจค่าของเงื่อนไข `!teamRepository.existsById(team.getId())`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (!teamRepository.existsById(team.getId())) {`
        if (!teamRepository.existsById(team.getId())) {
            // 👉 บรรทัด `throw new ResourceNotFoundException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new ResourceNotFoundException(
                // 👉 บรรทัด `"Team not found: " + team.getId()` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
                "Team not found: " + team.getId()
            );
        }

        // 👉 ผ่านแล้ว ส่งต่อให้ด่านถัดไป (ถ้ามี)
        // 👉 ตรวจค่าของเงื่อนไข `nextRule != null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (nextRule != null) {`
        if (nextRule != null) {
            // 👉 เรียกเมธอด `nextRule.validate(team, tournament)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
            nextRule.validate(team, tournament);
        }
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `setNext(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `next`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setNext(TeamJoinRule nextRule) {
        // 👉 เก็บ dependency/ค่าจาก `nextRule` ลงฟิลด์ `nextRule` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
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

// 👉 บรรทัดนี้ `@Component` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@Component
// 👉 ประกาศคลาส `TournamentExistsRule` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TournamentExistsRule implements TeamJoinRule {

    // 👉 บรรทัด `private final TournamentRepository tournamentRepository;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final TournamentRepository tournamentRepository;
    // 👉 บรรทัด `private TeamJoinRule nextRule;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private TeamJoinRule nextRule;

    // 👉 บรรทัด `public TournamentExistsRule(TournamentRepository tournamentRepository) {` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    public TournamentExistsRule(TournamentRepository tournamentRepository) {
        // 👉 เก็บ dependency/ค่าจาก `tournamentRepository` ลงฟิลด์ `tournamentRepository` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.tournamentRepository = tournamentRepository;
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `validate()` โดยรับพารามิเตอร์ `Team team, Tournament tournament`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void validate(Team team, Tournament tournament) {
        // 👉 โครงเดียวกับด่าน 1 แต่เช็คตารางรายการแข่ง ไม่มี → 404
        // 👉 ตรวจค่าของเงื่อนไข `!tournamentRepository.existsById(tournament.getId())`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (!tournamentRepository.existsById(tournament.getId())) {`
        if (!tournamentRepository.existsById(tournament.getId())) {
            // 👉 บรรทัด `throw new ResourceNotFoundException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new ResourceNotFoundException(
                // 👉 บรรทัด `"Tournament not found: " + tournament.getId()` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
                "Tournament not found: " + tournament.getId()
            );
        }

        // 👉 ตรวจค่าของเงื่อนไข `nextRule != null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (nextRule != null) {`
        if (nextRule != null) {
            // 👉 เรียกเมธอด `nextRule.validate(team, tournament)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
            nextRule.validate(team, tournament);
        }
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `setNext(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `next`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setNext(TeamJoinRule nextRule) {
        // 👉 เก็บ dependency/ค่าจาก `nextRule` ลงฟิลด์ `nextRule` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
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

// 👉 บรรทัดนี้ `@Component` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@Component
// 👉 ประกาศคลาส `TournamentStatusRule` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TournamentStatusRule implements TeamJoinRule {

    // 👉 บรรทัด `private final MatchRepository matchRepository;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final MatchRepository matchRepository;
    // 👉 บรรทัด `private TeamJoinRule nextRule;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private TeamJoinRule nextRule;

    // 👉 บรรทัด `public TournamentStatusRule(MatchRepository matchRepository) {` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    public TournamentStatusRule(MatchRepository matchRepository) {
        // 👉 เก็บ dependency/ค่าจาก `matchRepository` ลงฟิลด์ `matchRepository` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.matchRepository = matchRepository;
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `validate()` โดยรับพารามิเตอร์ `Team team, Tournament tournament`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void validate(Team team, Tournament tournament) {
        // 👉 เงื่อนไข A: รายการต้องยังไม่เริ่ม (UPCOMING) ไม่งั้น → 409
        // 👉 ตรวจว่าสถานะไม่ใช่ UPCOMING หรือไม่ เพราะห้ามลบรายการที่เริ่มแล้ว; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (tournament.getStatus() != TournamentStatus.UPCOMING) {`
        if (tournament.getStatus() != TournamentStatus.UPCOMING) {
            // 👉 บรรทัด `throw new BusinessException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new BusinessException(
                // 👉 บรรทัดนี้ `"Teams can only join an upcoming tournament"` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                "Teams can only join an upcoming tournament"
            );
        }

        // 👉 เงื่อนไข B: ต้องยังไม่มีแมตช์ในรายการนี้ (ตารางแข่ง/สายถูกสร้างแล้ว = ปิดรับสมัคร)
        //    existsByTournamentId = มีแมตช์ของรายการนี้อย่างน้อย 1 แมตช์หรือไม่
        // 👉 ตรวจค่าของเงื่อนไข `matchRepository.existsByTournamentId(tournament.getId())`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (matchRepository.existsByTournamentId(tournament.getId())) {`
        if (matchRepository.existsByTournamentId(tournament.getId())) {
            // 👉 บรรทัด `throw new BusinessException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new BusinessException(
                // 👉 บรรทัดนี้ `"Cannot join a tournament after its schedule or bracket has been created"` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                "Cannot join a tournament after its schedule or bracket has been created"
            );
        }

        // 👉 ตรวจค่าของเงื่อนไข `nextRule != null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (nextRule != null) {`
        if (nextRule != null) {
            // 👉 เรียกเมธอด `nextRule.validate(team, tournament)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
            nextRule.validate(team, tournament);
        }
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `setNext(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `next`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setNext(TeamJoinRule nextRule) {
        // 👉 เก็บ dependency/ค่าจาก `nextRule` ลงฟิลด์ `nextRule` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
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

// 👉 บรรทัดนี้ `@Component` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@Component
// 👉 ประกาศคลาส `TeamNotAlreadyJoinedRule` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TeamNotAlreadyJoinedRule implements TeamJoinRule {

    // 👉 บรรทัด `private final TournamentTeamRepository tournamentTeamRepository;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final TournamentTeamRepository tournamentTeamRepository;
    // 👉 บรรทัด `private TeamJoinRule nextRule;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private TeamJoinRule nextRule;

    // 👉 ประกาศสมาชิกของคลาส: `public TeamNotAlreadyJoinedRule(`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public TeamNotAlreadyJoinedRule(
            // 👉 บรรทัดนี้ `TournamentTeamRepository tournamentTeamRepository) {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            TournamentTeamRepository tournamentTeamRepository) {
        // 👉 เก็บ dependency/ค่าจาก `tournamentTeamRepository` ลงฟิลด์ `tournamentTeamRepository` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.tournamentTeamRepository = tournamentTeamRepository;
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `validate()` โดยรับพารามิเตอร์ `Team team, Tournament tournament`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void validate(Team team, Tournament tournament) {
        // 👉 ตาราง tournament_teams ใช้ "คีย์คู่" (tournament_id, team_id)
        //    จึงสร้าง TournamentTeamId(รายการ, ทีม) แล้วถามว่ามีแถวนี้อยู่แล้วไหม
        boolean alreadyJoined =
                // 👉 บรรทัดนี้ `tournamentTeamRepository.existsById(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                tournamentTeamRepository.existsById(
                        // 👉 บรรทัดนี้ `new com.example.tournament.domain.entity.TournamentTeamId(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                        new com.example.tournament.domain.entity.TournamentTeamId(
                                // 👉 เรียกเมธอด `tournament.getId(), team.getId()))`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                                tournament.getId(), team.getId()));

        // 👉 สมัครไว้แล้ว → 409
        // 👉 ตรวจค่าของเงื่อนไข `alreadyJoined`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (alreadyJoined) {`
        if (alreadyJoined) {
            // 👉 บรรทัด `throw new BusinessException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new BusinessException(
                    // 👉 บรรทัดนี้ `"Team has already joined this tournament");` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                    "Team has already joined this tournament");
        }

        // 👉 ตรวจค่าของเงื่อนไข `nextRule != null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (nextRule != null) {`
        if (nextRule != null) {
            // 👉 เรียกเมธอด `nextRule.validate(team, tournament)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
            nextRule.validate(team, tournament);
        }
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `setNext(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `next`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setNext(TeamJoinRule nextRule) {
        // 👉 เก็บ dependency/ค่าจาก `nextRule` ลงฟิลด์ `nextRule` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
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

// 👉 บรรทัดนี้ `@Component` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@Component
// 👉 ประกาศคลาส `TeamGameMatchRule` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TeamGameMatchRule implements TeamJoinRule {

    private TeamJoinRule nextRule;   // 👉 ด่านนี้ไม่ต้องใช้ Repository จึงไม่มี constructor

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `validate()` โดยรับพารามิเตอร์ `Team team, Tournament tournament`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void validate(Team team, Tournament tournament) {
        // 👉 ไม่ผ่านถ้า: ทีมไม่มีเกม  หรือ รายการไม่มีเกม  หรือ id เกมไม่เท่ากัน
        //    (|| = "หรือ"  ถ้าข้อซ้ายจริง จะไม่ประเมินข้อขวาต่อ จึงไม่เกิด NullPointer)
        // 👉 ตรวจค่าของเงื่อนไข `team.getGame() == null || tournament.getGame(`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (team.getGame() == null || tournament.getGame() == null`
        if (team.getGame() == null || tournament.getGame() == null
                // 👉 เป็นเงื่อนไขต่อจากบรรทัด `if` ก่อนหน้า โดย `||` หมายถึงมีอย่างน้อยหนึ่งเงื่อนไขเป็นจริงก็เข้าเงื่อนไขได้
                || !team.getGame().getId().equals(tournament.getGame().getId())) {
            // 👉 บรรทัด `throw new BusinessException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new BusinessException(
                    // 👉 บรรทัดนี้ `"Team's game must match the tournament's game");` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                    "Team's game must match the tournament's game");
        }

        // 👉 ตรวจค่าของเงื่อนไข `nextRule != null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (nextRule != null) {`
        if (nextRule != null) {
            // 👉 เรียกเมธอด `nextRule.validate(team, tournament)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
            nextRule.validate(team, tournament);
        }
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `setNext(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `next`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setNext(TeamJoinRule nextRule) {
        // 👉 เก็บ dependency/ค่าจาก `nextRule` ลงฟิลด์ `nextRule` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
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

// 👉 บรรทัดนี้ `@Component` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@Component
// 👉 ประกาศคลาส `TeamMinimumPlayersRule` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TeamMinimumPlayersRule implements TeamJoinRule {

    // 👉 บรรทัด `private final PlayerRepository playerRepository;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final PlayerRepository playerRepository;
    // 👉 บรรทัด `private TeamJoinRule nextRule;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private TeamJoinRule nextRule;

    // 👉 บรรทัด `public TeamMinimumPlayersRule(PlayerRepository playerRepository) {` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    public TeamMinimumPlayersRule(PlayerRepository playerRepository) {
        // 👉 เก็บ dependency/ค่าจาก `playerRepository` ลงฟิลด์ `playerRepository` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.playerRepository = playerRepository;
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `validate()` โดยรับพารามิเตอร์ `Team team, Tournament tournament`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void validate(Team team, Tournament tournament) {
        // 👉 จำนวนผู้เล่นขั้นต่ำของเกมนั้น (ตาราง games คอลัมน์ min_players) อาจเป็น null ถ้าไม่ได้กำหนด
        // 👉 กำหนดค่าจาก `tournament.getGame().getMinPlayers()` ให้ `minimumPlayers`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        Integer minimumPlayers = tournament.getGame().getMinPlayers();

        // 👉 ถ้าเกมไม่ได้กำหนดขั้นต่ำ ข้ามการตรวจนี้ไปเลย
        // 👉 ตรวจค่าของเงื่อนไข `minimumPlayers != null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (minimumPlayers != null) {`
        if (minimumPlayers != null) {
            // 👉 นับผู้เล่นจริงในทีม (SELECT COUNT(*) ... WHERE team_id = ?)
            // 👉 กำหนดค่าจาก `playerRepository.countByTeamId(team.getId())` ให้ `actualPlayers`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
            long actualPlayers = playerRepository.countByTeamId(team.getId());

            // 👉 น้อยกว่าขั้นต่ำ → 409 เช่น ROV ต้อง 5 คน ทีมมี 4 คน
            // 👉 ตรวจค่าของเงื่อนไข `actualPlayers < minimumPlayers`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (actualPlayers < minimumPlayers) {`
            if (actualPlayers < minimumPlayers) {
                // 👉 บรรทัด `throw new BusinessException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                throw new BusinessException(
                        // 👉 บรรทัดนี้ `"Team must have at least " + minimumPlayers` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                        "Team must have at least " + minimumPlayers
                                // 👉 บรรทัดนี้ `+ " players to join this tournament");` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                                + " players to join this tournament");
            }
        }

        // 👉 ตรวจค่าของเงื่อนไข `nextRule != null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (nextRule != null) {`
        if (nextRule != null) {
            // 👉 เรียกเมธอด `nextRule.validate(team, tournament)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
            nextRule.validate(team, tournament);
        }
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `setNext(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `next`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setNext(TeamJoinRule nextRule) {
        // 👉 เก็บ dependency/ค่าจาก `nextRule` ลงฟิลด์ `nextRule` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
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

// 👉 บรรทัดนี้ `@Component` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@Component
// 👉 ประกาศคลาส `TeamPointsLimitRule` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TeamPointsLimitRule implements TeamJoinRule {

    // 👉 บรรทัด `private final TournamentTeamRepository tournamentTeamRepository;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final TournamentTeamRepository tournamentTeamRepository;
    // 👉 บรรทัด `private TeamJoinRule nextRule;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private TeamJoinRule nextRule;

    // 👉 ประกาศสมาชิกของคลาส: `public TeamPointsLimitRule(`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public TeamPointsLimitRule(
            // 👉 บรรทัดนี้ `TournamentTeamRepository tournamentTeamRepository) {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            TournamentTeamRepository tournamentTeamRepository) {
        // 👉 เก็บ dependency/ค่าจาก `tournamentTeamRepository` ลงฟิลด์ `tournamentTeamRepository` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.tournamentTeamRepository = tournamentTeamRepository;
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `validate()` โดยรับพารามิเตอร์ `Team team, Tournament tournament`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void validate(Team team, Tournament tournament) {
        // 👉 ใช้เฉพาะรายการแบบ POINTS (Free Fire) เท่านั้น แบบแพ้คัดออกไม่จำกัดจำนวนทีมด้วยกฎนี้
        // 👉 ตรวจค่าของเงื่อนไข `tournament.getFormat() == TournamentFormat.POINTS`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (tournament.getFormat() == TournamentFormat.POINTS) {`
        if (tournament.getFormat() == TournamentFormat.POINTS) {
            // 👉 นับทีมที่สมัครอยู่ตอนนี้
            long currentTeams =
                    // 👉 บรรทัดนี้ `tournamentTeamRepository.countByTournamentId(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                    tournamentTeamRepository.countByTournamentId(
                            // 👉 เรียกเมธอด `tournament.getId())`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                            tournament.getId());

            // 👉 ถ้ามี 12 ทีมแล้ว (เต็ม) ทีมที่ 13 สมัครไม่ได้ → 409
            //    (12 มาจากตารางคะแนนอันดับที่มี 12 อันดับ)
            // 👉 ตรวจค่าของเงื่อนไข `currentTeams >= 12`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (currentTeams >= 12) {`
            if (currentTeams >= 12) {
                // 👉 บรรทัด `throw new BusinessException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                throw new BusinessException(
                        // 👉 บรรทัดนี้ `"POINTS tournaments cannot have more than 12 teams");` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                        "POINTS tournaments cannot have more than 12 teams");
            }
        }

        // 👉 ตรวจค่าของเงื่อนไข `nextRule != null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (nextRule != null) {`
        if (nextRule != null) {
            // 👉 เรียกเมธอด `nextRule.validate(team, tournament)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
            nextRule.validate(team, tournament);
        }
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `setNext(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `next`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setNext(TeamJoinRule nextRule) {
        // 👉 เก็บ dependency/ค่าจาก `nextRule` ลงฟิลด์ `nextRule` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
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

// 👉 บรรทัดนี้ `@Component` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@Component
// 👉 ประกาศคลาส `TeamTournamentDateRule` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TeamTournamentDateRule implements TeamJoinRule {

    // 👉 บรรทัด `private final TournamentTeamRepository tournamentTeamRepository;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final TournamentTeamRepository tournamentTeamRepository;
    // 👉 บรรทัด `private TeamJoinRule nextRule;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private TeamJoinRule nextRule;

    // 👉 ประกาศสมาชิกของคลาส: `public TeamTournamentDateRule(`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public TeamTournamentDateRule(
            // 👉 บรรทัดนี้ `TournamentTeamRepository tournamentTeamRepository) {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            TournamentTeamRepository tournamentTeamRepository) {
        // 👉 เก็บ dependency/ค่าจาก `tournamentTeamRepository` ลงฟิลด์ `tournamentTeamRepository` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.tournamentTeamRepository = tournamentTeamRepository;
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `validate()` โดยรับพารามิเตอร์ `Team team, Tournament tournament`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void validate(Team team, Tournament tournament) {
        // 👉 ถามฐานข้อมูลว่า "ทีมนี้อยู่ในรายการอื่นกี่รายการ ที่ช่วงวันแข่งชนกับรายการนี้"
        long overlappingTournaments =
                // 👉 บรรทัดนี้ `tournamentTeamRepository.countOverlappingTournaments(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                tournamentTeamRepository.countOverlappingTournaments(
                        // 👉 เป็นค่าหรือพารามิเตอร์ `team.getId()` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                        team.getId(),
                        // 👉 ส่ง id ของ entity เข้าไปเป็นฟิลด์แรกของ `TournamentResponse` เพื่อให้ client อ้างถึงรายการนี้ได้
                        tournament.getId(),
                        // 👉 ส่งวันเริ่มแข่งขันไปยัง response
                        tournament.getStartDate(),
                        // 👉 เรียกเมธอด `tournament.getEndDate())`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                        tournament.getEndDate());

        // 👉 มีอย่างน้อย 1 รายการที่ชน → 409
        // 👉 ตรวจค่าของเงื่อนไข `overlappingTournaments > 0`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (overlappingTournaments > 0) {`
        if (overlappingTournaments > 0) {
            // 👉 บรรทัด `throw new BusinessException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
            throw new BusinessException(
                    // 👉 บรรทัดนี้ `"Team already joined another tournament with overlapping dates");` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                    "Team already joined another tournament with overlapping dates");
        }

        // 👉 ตรวจค่าของเงื่อนไข `nextRule != null`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (nextRule != null) {`
        if (nextRule != null) {
            // 👉 เรียกเมธอด `nextRule.validate(team, tournament)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
            nextRule.validate(team, tournament);
        }
    }

    // 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
    @Override
    // 👉 ประกาศเมธอด `setNext(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `next`; `void` หมายถึงไม่ส่งค่ากลับ
    public void setNext(TeamJoinRule nextRule) {
        // 👉 เก็บ dependency/ค่าจาก `nextRule` ลงฟิลด์ `nextRule` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.nextRule = nextRule;
    }
}```

### 6.9.1 query นับรายการที่วันทับกัน (`TournamentTeamRepository.java`)

```java
// 👉 ประกาศสัญญา `TournamentTeamRepository` ระบุว่าโค้ดที่นำไป implement ต้องมีเมธอดตามที่ประกาศไว้ แต่ยังไม่บอกวิธีทำงานจริง
public interface TournamentTeamRepository extends JpaRepository<TournamentTeam, TournamentTeamId> {

    // 👉 ประกาศตัวแปร/ชนิดข้อมูล `List<TournamentTeam> findByTournamentId(Long tournamentId)` เพื่อระบุว่าค่าที่เก็บเป็นรายการหรือค่าที่อาจไม่มีข้อมูล
    List<TournamentTeam> findByTournamentId(Long tournamentId);

    // 👉 ประกาศตัวแปร/ชนิดข้อมูล `List<TournamentTeam> findByTournamentIdOrderByJoinedAtAsc(Long tournamentId)` เพื่อระบุว่าค่าที่เก็บเป็นรายการหรือค่าที่อาจไม่มีข้อมูล
    List<TournamentTeam> findByTournamentIdOrderByJoinedAtAsc(Long tournamentId);

    long countByTournamentId(Long tournamentId);   // 👉 นับทีมในรายการ (ใช้ในด่าน 7)

    // 👉 บรรทัด `boolean existsByTeam_Id(Long teamId);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    boolean existsByTeam_Id(Long teamId);

    // 👉 @Query = เขียนคำสั่งค้นหาเอง (ภาษา JPQL คล้าย SQL แต่ใช้ชื่อคลาส/ฟิลด์ Java)
    //    คำอธิบายแต่ละบรรทัดของ query อยู่ในตารางใต้บล็อกโค้ดนี้
    @Query("""
            // 👉 บรรทัด `SELECT COUNT(tt)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            SELECT COUNT(tt)
            // 👉 บรรทัดนี้ `FROM TournamentTeam tt` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            FROM TournamentTeam tt
            // 👉 บรรทัดนี้ `WHERE tt.team.id = :teamId` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            WHERE tt.team.id = :teamId
              // 👉 บรรทัดนี้ `AND tt.tournament.id <> :tournamentId` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
              AND tt.tournament.id <> :tournamentId
              // 👉 บรรทัดนี้ `AND tt.tournament.startDate <= :endDate` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
              AND tt.tournament.startDate <= :endDate
              // 👉 บรรทัดนี้ `AND tt.tournament.endDate >= :startDate` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
              AND tt.tournament.endDate >= :startDate
            // 👉 บรรทัด `""")` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            """)
    // 👉 บรรทัดนี้ `long countOverlappingTournaments(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    long countOverlappingTournaments(
            // 👉 เป็นค่าหรือพารามิเตอร์ `@Param("teamId") Long teamId` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            @Param("teamId") Long teamId,
            // 👉 เป็นค่าหรือพารามิเตอร์ `@Param("tournamentId") Long tournamentId` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            @Param("tournamentId") Long tournamentId,
            // 👉 เป็นค่าหรือพารามิเตอร์ `@Param("startDate") LocalDate startDate` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            @Param("startDate") LocalDate startDate,
            // 👉 บรรทัด `@Param("endDate") LocalDate endDate);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
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

// 👉 บรรทัดนี้ `@Component` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
@Component
// 👉 ประกาศคลาส `TeamJoinRuleChain` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TeamJoinRuleChain {

    private final TeamExistsRule teamExistsRule;   // 👉 เก็บ "ด่านแรก" ไว้ ตอนใช้งานเริ่มจากด่านนี้

    // 👉 Spring ฉีดทั้ง 8 ด่านเข้ามาทาง constructor
    public TeamJoinRuleChain(
            // 👉 เป็นค่าหรือพารามิเตอร์ `TeamExistsRule teamExistsRule` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            TeamExistsRule teamExistsRule,
            // 👉 เป็นค่าหรือพารามิเตอร์ `TournamentExistsRule tournamentExistsRule` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            TournamentExistsRule tournamentExistsRule,
            // 👉 เป็นค่าหรือพารามิเตอร์ `TournamentStatusRule tournamentStatusRule` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            TournamentStatusRule tournamentStatusRule,
            // 👉 เป็นค่าหรือพารามิเตอร์ `TeamNotAlreadyJoinedRule teamNotAlreadyJoinedRule` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            TeamNotAlreadyJoinedRule teamNotAlreadyJoinedRule,
            // 👉 เป็นค่าหรือพารามิเตอร์ `TeamGameMatchRule teamGameMatchRule` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            TeamGameMatchRule teamGameMatchRule,
            // 👉 เป็นค่าหรือพารามิเตอร์ `TeamMinimumPlayersRule teamMinimumPlayersRule` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            TeamMinimumPlayersRule teamMinimumPlayersRule,
            // 👉 เป็นค่าหรือพารามิเตอร์ `TeamPointsLimitRule teamPointsLimitRule` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            TeamPointsLimitRule teamPointsLimitRule,
            // 👉 บรรทัดนี้ `TeamTournamentDateRule teamTournamentDateRule) {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            TeamTournamentDateRule teamTournamentDateRule) {

        // 👉 เก็บ dependency/ค่าจาก `teamExistsRule` ลงฟิลด์ `teamExistsRule` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
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
    // 👉 ประกาศเมธอด `validate()` โดยรับพารามิเตอร์ `Team team, Tournament tournament`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public void validate(Team team, Tournament tournament) {
        // 👉 เรียกเมธอด `teamExistsRule.validate(team, tournament)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
        teamExistsRule.validate(team, tournament);
    }
}```

## 6.11 ที่ที่เรียกใช้โซ่: `TournamentTeamServiceImpl.addTeam` และ Controller

```java
// ===== ส่วนของ TournamentTeamServiceImpl.java =====
@Override
// 👉 ประกาศเมธอด `addTeam()` โดยรับพารามิเตอร์ `Long tournamentId, Long teamId`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
public void addTeam(Long tournamentId, Long teamId) {
    // ตรวจสอบทีมก่อน เพื่อรักษาลำดับการตรวจสอบของกฎเดิม
    // 👉 โหลดทีมและรายการจากฐานข้อมูล (ไม่เจอ → 404)
    // 👉 ประกาศตัวแปร `team` แล้วค้นหาข้อมูลด้วย `teamRepository.findById(teamId)`; Repository คืน `Optional` เพราะรหัสนั้นอาจมีหรือไม่มีในฐานข้อมูล
    Team team = teamRepository.findById(teamId)
            // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
            .orElseThrow(() ->
                    // 👉 บรรทัด `new ResourceNotFoundException("Team not found"));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                    new ResourceNotFoundException("Team not found"));

    // 👉 ประกาศตัวแปร `tournament` แล้วค้นหาข้อมูลด้วย `tournamentRepository.findById(tournamentId)`; Repository คืน `Optional` เพราะรหัสนั้นอาจมีหรือไม่มีในฐานข้อมูล
    Tournament tournament = tournamentRepository.findById(tournamentId)
            // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
            .orElseThrow(() ->
                    // 👉 บรรทัด `new ResourceNotFoundException("Tournament not found"));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                    new ResourceNotFoundException("Tournament not found"));

    // ตรวจสอบเงื่อนไขการสมัครทีมทั้งหมด
    // 👉 ★ จุดที่เรียกโซ่ 8 ด่าน ★ ไม่ผ่านด่านไหนจะ throw ออกมาตรงนี้ และโค้ดข้างล่างจะไม่ทำงาน
    // 👉 เรียกเมธอด `teamJoinRuleChain.validate(team, tournament)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
    teamJoinRuleChain.validate(team, tournament);

    // บันทึกข้อมูลทีม ณ วันที่สมัคร และสร้าง snapshot ของผู้เล่น
    // 👉 ผ่านทุกด่านแล้วจึงบันทึกการสมัคร พร้อมเก็บ "ชื่อ/คำอธิบาย/โลโก้ของทีม ณ ตอนนี้" ไว้
    //    (ถ้าทีมเปลี่ยนชื่อภายหลัง ประวัติของรายการนี้ยังแสดงชื่อเดิม)
    // 👉 สร้าง object ชนิด `TournamentTeam` ใหม่ แล้วเก็บไว้ในตัวแปร `tournamentTeam`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
    TournamentTeam tournamentTeam = new TournamentTeam();
    // 👉 เรียก `tournamentTeam.setId(new TournamentTeamId(tournamentId, teamId))` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
    tournamentTeam.setId(new TournamentTeamId(tournamentId, teamId));
    // 👉 เรียก `tournamentTeam.setTournament(tournament)` เพื่อกำหนดค่า ทัวร์นาเมนต์ที่คะแนนอันดับนี้สังกัด; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
    tournamentTeam.setTournament(tournament);
    // 👉 เรียก `tournamentTeam.setTeam(team)` เพื่อกำหนดค่า ฟิลด์ team; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
    tournamentTeam.setTeam(team);
    // 👉 เรียก `tournamentTeam.setJoinedAt(LocalDateTime.now())` เพื่อกำหนดค่า ฟิลด์ joinedAt; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
    tournamentTeam.setJoinedAt(LocalDateTime.now());
    // 👉 เรียก `tournamentTeam.setTeamName(team.getName())` เพื่อกำหนดค่า ฟิลด์ teamName; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
    tournamentTeam.setTeamName(team.getName());
    // 👉 เรียก `tournamentTeam.setTeamDescription(team.getDescription())` เพื่อกำหนดค่า ฟิลด์ teamDescription; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
    tournamentTeam.setTeamDescription(team.getDescription());
    // 👉 เรียก `tournamentTeam.setTeamLogoUrl(team.getLogoUrl())` เพื่อกำหนดค่า ฟิลด์ teamLogoUrl; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
    tournamentTeam.setTeamLogoUrl(team.getLogoUrl());

    // 👉 เรียก `tournamentTeamRepository.saveAndFlush(tournamentTeam)` เพื่อบันทึก entity และบังคับให้ JPA ส่งคำสั่ง SQL ไปฐานข้อมูลทันที ทำให้มักได้ id ที่สร้างแล้วกลับมาใช้ต่อ
    tournamentTeamRepository.saveAndFlush(tournamentTeam);
    rosters.capturePlayers(tournamentId, teamId);   // 👉 บันทึกรายชื่อผู้เล่นของทีม ณ วันที่สมัคร (snapshot)
}

// 👉 ยืนยันว่าเมธอดถัดไปกำลัง implement/override เมธอดที่ประกาศไว้ใน interface หรือคลาสแม่ ถ้าชื่อหรือพารามิเตอร์ไม่ตรง compiler จะแจ้ง
@Override
// 👉 ประกาศเมธอด `removeTeam()` โดยรับพารามิเตอร์ `Long tournamentId, Long teamId`; ชนิดผลลัพธ์ `void` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
public void removeTeam(Long tournamentId, Long teamId) {
    // 👉 สร้าง object ชนิด `TournamentTeamId` ใหม่ แล้วเก็บไว้ในตัวแปร `id`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
    TournamentTeamId id = new TournamentTeamId(tournamentId, teamId);

    // ค้นหารายการสมัครทีมก่อน แล้วตรวจสอบสถานะจากรายการนั้น
    // 👉 บรรทัด `TournamentTeam tournamentTeam = tournamentTeamRepository.findById(id)` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    TournamentTeam tournamentTeam = tournamentTeamRepository.findById(id)
            // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
            .orElseThrow(() ->
                    // 👉 บรรทัด `new ResourceNotFoundException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                    new ResourceNotFoundException(
                            // 👉 บรรทัดนี้ `"Team is not registered in this tournament"));` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                            "Team is not registered in this tournament"));

    // 👉 ถอนทีมได้เฉพาะตอนรายการยังไม่เริ่ม
    // 👉 ตรวจค่าของเงื่อนไข `tournamentTeam.getTournament().getStatus(`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (tournamentTeam.getTournament().getStatus()`
    if (tournamentTeam.getTournament().getStatus()
            // 👉 บรรทัดนี้ `!= TournamentStatus.UPCOMING) {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            != TournamentStatus.UPCOMING) {
        // 👉 บรรทัด `throw new BusinessException(` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        throw new BusinessException(
                // 👉 บรรทัดนี้ `"Cannot remove a team after the tournament has started");` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                "Cannot remove a team after the tournament has started");
    }

    // 👉 เรียก `tournamentTeamRepository.delete(tournamentTeam)` เพื่อสั่ง Repository ลบ entity ที่ระบุออกจากฐานข้อมูล
    tournamentTeamRepository.delete(tournamentTeam);
}```

```java
// ===== TournamentTeamController.java =====
@RestController
// 👉 บรรทัด `@RequestMapping("/api/v1/tournaments/{tournamentId}/teams")` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
@RequestMapping("/api/v1/tournaments/{tournamentId}/teams")
// 👉 ประกาศคลาส `TournamentTeamController` เป็นที่รวมข้อมูลและพฤติกรรมของส่วนงานนี้; คำสั่งในวงเล็บปีกกาจะเป็นสมาชิกของคลาส
public class TournamentTeamController {

    // 👉 บรรทัด `private final TournamentTeamService tournamentTeams;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private final TournamentTeamService tournamentTeams;

    // 👉 บรรทัด `public TournamentTeamController(TournamentTeamService tournamentTeams) {` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    public TournamentTeamController(TournamentTeamService tournamentTeams) {
        // 👉 เก็บ dependency/ค่าจาก `tournamentTeams` ลงฟิลด์ `tournamentTeams` ของ object ปัจจุบัน เพื่อให้เมธอดอื่นในคลาสเรียกใช้ได้
        this.tournamentTeams = tournamentTeams;
    }

    // ผ่านกฎ 8 ข้อ (Chain of Responsibility) แล้วบันทึก → 204, ไม่ผ่าน → 400 / 404 / 409
    // 👉 POST /api/v1/tournaments/5/teams   body: { "teamId": 3 }
    @PostMapping
    // 👉 ประกาศสมาชิกของคลาส: `public ResponseEntity<Void> add(@PathVariable Long tournamentId,`; ชนิดข้อมูล/ชนิดคืนค่าและพารามิเตอร์กำหนดวิธีเรียกใช้งาน
    public ResponseEntity<Void> add(@PathVariable Long tournamentId,
            // 👉 ขอให้ Spring ตรวจ validation ของ object ที่รับเข้ามาก่อนเรียกเมธอด; จะมีผลเมื่อ DTO มี validation constraints
            @Valid @RequestBody AddTournamentTeamRequest request) {
        // 👉 เรียกเมธอด `tournamentTeams.addTeam(tournamentId, request.teamId())`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
        tournamentTeams.addTeam(tournamentId, request.teamId());
        // 👉 คืนค่า `ResponseEntity.noContent().build()` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
        return ResponseEntity.noContent().build();
    }

    // ถอนได้เฉพาะรายการที่ยังไม่เริ่ม (UPCOMING) → 204, ไม่พบ → 404, เริ่มแล้ว → 409
    // 👉 DELETE /api/v1/tournaments/5/teams/3
    @DeleteMapping("/{teamId}")
    // 👉 ประกาศเมธอด `remove()` โดยรับพารามิเตอร์ `@PathVariable Long tournamentId, @PathVariable Long teamId`; ชนิดผลลัพธ์ `ResponseEntity<Void>` บอกว่าผู้เรียกจะได้รับข้อมูลชนิดใด
    public ResponseEntity<Void> remove(@PathVariable Long tournamentId, @PathVariable Long teamId) {
        // 👉 เรียกเมธอด `tournamentTeams.removeTeam(tournamentId, teamId)`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
        tournamentTeams.removeTeam(tournamentId, teamId);
        // 👉 คืนค่า `ResponseEntity.noContent().build()` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
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
// 👉 บรรทัดนี้ `class TournamentServiceImplTest {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
class TournamentServiceImplTest {

        // 👉 @Mock = สร้าง Repository "ปลอม" (ไม่ต่อฐานข้อมูล ตอบตามที่เราสั่งด้วย when(...))
        @Mock
        // 👉 พารามิเตอร์ constructor นี้เป็น Repository ที่ Service ต้องใช้ติดต่อฐานข้อมูล
        TournamentRepository tournaments;

        // 👉 บรรทัดนี้ `@Mock` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        @Mock
        // 👉 พารามิเตอร์ constructor นี้เป็น Repository ที่ Service ต้องใช้ติดต่อฐานข้อมูล
        GameRepository games;

        // 👉 บรรทัดนี้ `@Mock` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        @Mock
        // 👉 พารามิเตอร์ constructor นี้เป็น Repository ที่ Service ต้องใช้ติดต่อฐานข้อมูล
        TournamentPlacementPointRepository tournamentPlacementPoints;

        TournamentServiceImpl service;   // 👉 ตัวที่ถูกทดสอบ (ของจริง แต่ใช้ Repository ปลอม)

        Game freeFire;   // 👉 เกมจำลอง: Free Fire (รหัส FREE_FIRE)
        Game rov;        // 👉 เกมจำลอง: ROV

        // 👉 @BeforeEach = รันก่อน "ทุกเทสต์" เพื่อสร้างของใหม่ทุกครั้ง เทสต์จะได้ไม่กวนกัน
        @BeforeEach
        // 👉 บรรทัดนี้ `void setUp() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void setUp() {
                // 👉 สร้าง object ชนิด `TournamentServiceImpl` ใหม่ แล้วเก็บไว้ในตัวแปร `service`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
                service = new TournamentServiceImpl(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `tournaments` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                tournaments,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `games` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                games,
                                // 👉 บรรทัดนี้ `tournamentPlacementPoints);` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                                tournamentPlacementPoints);

                freeFire = game(1L, "FREE_FIRE", 4);   // 👉 id=1, ขั้นต่ำ 4 คน
                rov = game(2L, "ROV", 5);              // 👉 id=2, ขั้นต่ำ 5 คน
        }

        // ---------- กลุ่ม 1: เคส "ต้องปฏิเสธ" (คาดว่าต้อง throw) ----------

        // 👉 เคส: วันจบก่อนวันเริ่ม (19 ต.ค. < 20 ต.ค.) ต้อง throw ValidationException
        @Test
        // 👉 บรรทัดนี้ `void rejectsWhenEndDateIsBeforeStartDate() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void rejectsWhenEndDateIsBeforeStartDate() {
                // Arrange: สร้างคำขอที่ผิดกฎ (เริ่ม 20 ต.ค. แต่จบ 19 ต.ค.)
                TournamentRequest request = request(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `"Test Tournament"` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                "Test Tournament",
                                // 👉 เป็นค่าหรือพารามิเตอร์ `2L` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                2L,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `TournamentFormat.SINGLE_ELIMINATION` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                TournamentFormat.SINGLE_ELIMINATION,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `null` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                null,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `LocalDate.of(2026, 10, 20)` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกเมธอด `LocalDate.of(2026, 10, 19))`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                                LocalDate.of(2026, 10, 19));

                // 👉 สั่ง mock: ชื่อนี้ยังไม่ซ้ำ / เกม id 2 มีอยู่ (คือ ROV) เพื่อให้โค้ดเดินไปถึงขั้นตรวจวันที่
                // 👉 บรรทัด `when(tournaments.existsByName("Test Tournament")).thenReturn(false);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.existsByName("Test Tournament")).thenReturn(false);
                // 👉 บรรทัด `when(games.findById(2L)).thenReturn(Optional.of(rov));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(games.findById(2L)).thenReturn(Optional.of(rov));

                // Act + Assert: เรียก create แล้วต้อง throw ValidationException
                assertThrows(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `ValidationException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                ValidationException.class,
                                // 👉 บรรทัด `() -> service.create(request));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                () -> service.create(request));
        }

        // 👉 เคส: ชื่อซ้ำ ต้อง throw BusinessException (409)
        @Test
        // 👉 บรรทัดนี้ `void rejectsDuplicateTournamentName() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void rejectsDuplicateTournamentName() {
                // 👉 บรรทัดนี้ `TournamentRequest request = request(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                TournamentRequest request = request(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `"Duplicate Tournament"` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                "Duplicate Tournament",
                                // 👉 เป็นค่าหรือพารามิเตอร์ `2L` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                2L,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `TournamentFormat.SINGLE_ELIMINATION` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                TournamentFormat.SINGLE_ELIMINATION,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `null` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                null,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `LocalDate.of(2026, 10, 20)` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกเมธอด `LocalDate.of(2026, 10, 21))`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                                LocalDate.of(2026, 10, 21));

                // 👉 สั่ง mock ให้ตอบว่า "ชื่อนี้มีอยู่แล้ว" (true)
                // 👉 บรรทัด `when(tournaments.existsByName("Duplicate Tournament")).thenReturn(true);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.existsByName("Duplicate Tournament")).thenReturn(true);

                // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                assertThrows(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `BusinessException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                BusinessException.class,
                                // 👉 บรรทัด `() -> service.create(request));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                () -> service.create(request));
        }

        // 👉 เคส: Free Fire แต่เลือก SINGLE_ELIMINATION ต้องปฏิเสธ (กฎ 4)
        @Test
        // 👉 บรรทัดนี้ `void rejectsFreeFireWithSingleElimination() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void rejectsFreeFireWithSingleElimination() {
                // 👉 บรรทัดนี้ `TournamentRequest request = request(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                TournamentRequest request = request(
                                // 👉 ข้อความนี้เป็นรายละเอียดของข้อผิดพลาด ช่วยบอก client ว่าข้อมูลผิดกฎข้อไหน
                                "Free Fire Tournament",
                                // 👉 เป็นค่าหรือพารามิเตอร์ `1L` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                1L,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `TournamentFormat.SINGLE_ELIMINATION` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                TournamentFormat.SINGLE_ELIMINATION,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `null` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                null,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `LocalDate.of(2026, 10, 20)` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกเมธอด `LocalDate.of(2026, 10, 21))`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                                LocalDate.of(2026, 10, 21));

                // 👉 บรรทัด `when(tournaments.existsByName("Free Fire Tournament")).thenReturn(false);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.existsByName("Free Fire Tournament")).thenReturn(false);
                // 👉 บรรทัด `when(games.findById(1L)).thenReturn(Optional.of(freeFire));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(games.findById(1L)).thenReturn(Optional.of(freeFire));

                // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                assertThrows(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `ValidationException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                ValidationException.class,
                                // 👉 บรรทัด `() -> service.create(request));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                () -> service.create(request));
        }

        // 👉 เคส: เกมที่ไม่ใช่ Free Fire (ROV) แต่เลือก POINTS ต้องปฏิเสธ (กฎ 5)
        @Test
        // 👉 บรรทัดนี้ `void rejectsNonFreeFireWithPointsFormat() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void rejectsNonFreeFireWithPointsFormat() {
                // 👉 บรรทัดนี้ `TournamentRequest request = request(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                TournamentRequest request = request(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `"ROV Points Tournament"` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                "ROV Points Tournament",
                                // 👉 เป็นค่าหรือพารามิเตอร์ `2L` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                2L,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `TournamentFormat.POINTS` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                TournamentFormat.POINTS,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `(short) 10` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                (short) 10,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `LocalDate.of(2026, 10, 20)` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกเมธอด `LocalDate.of(2026, 10, 21))`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                                LocalDate.of(2026, 10, 21));

                // 👉 บรรทัด `when(tournaments.existsByName("ROV Points Tournament")).thenReturn(false);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.existsByName("ROV Points Tournament")).thenReturn(false);
                // 👉 บรรทัด `when(games.findById(2L)).thenReturn(Optional.of(rov));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(games.findById(2L)).thenReturn(Optional.of(rov));

                // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                assertThrows(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `ValidationException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                ValidationException.class,
                                // 👉 บรรทัด `() -> service.create(request));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                () -> service.create(request));
        }

        // 👉 เคส: POINTS แต่ไม่ใส่ totalGames (null) ต้องปฏิเสธ (กฎ 6)
        @Test
        // 👉 บรรทัดนี้ `void rejectsPointsWithoutTotalGames() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void rejectsPointsWithoutTotalGames() {
                // 👉 บรรทัดนี้ `TournamentRequest request = request(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                TournamentRequest request = request(
                                // 👉 ข้อความนี้เป็นรายละเอียดของข้อผิดพลาด ช่วยบอก client ว่าข้อมูลผิดกฎข้อไหน
                                "Free Fire Tournament",
                                // 👉 เป็นค่าหรือพารามิเตอร์ `1L` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                1L,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `TournamentFormat.POINTS` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                TournamentFormat.POINTS,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `null` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                null,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `LocalDate.of(2026, 10, 20)` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกเมธอด `LocalDate.of(2026, 10, 21))`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                                LocalDate.of(2026, 10, 21));

                // 👉 บรรทัด `when(tournaments.existsByName("Free Fire Tournament")).thenReturn(false);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.existsByName("Free Fire Tournament")).thenReturn(false);
                // 👉 บรรทัด `when(games.findById(1L)).thenReturn(Optional.of(freeFire));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(games.findById(1L)).thenReturn(Optional.of(freeFire));

                // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                assertThrows(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `ValidationException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                ValidationException.class,
                                // 👉 บรรทัด `() -> service.create(request));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                () -> service.create(request));
        }

        // 👉 เคส: แพ้คัดออกแต่ใส่ totalGames = 10 ต้องปฏิเสธ (กฎ 7)
        @Test
        // 👉 บรรทัดนี้ `void rejectsEliminationWithTotalGames() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void rejectsEliminationWithTotalGames() {
                // 👉 บรรทัดนี้ `TournamentRequest request = request(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                TournamentRequest request = request(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `"ROV Tournament"` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                "ROV Tournament",
                                // 👉 เป็นค่าหรือพารามิเตอร์ `2L` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                2L,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `TournamentFormat.SINGLE_ELIMINATION` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                TournamentFormat.SINGLE_ELIMINATION,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `(short) 10` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                (short) 10,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `LocalDate.of(2026, 10, 20)` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกเมธอด `LocalDate.of(2026, 10, 21))`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                                LocalDate.of(2026, 10, 21));

                // 👉 บรรทัด `when(tournaments.existsByName("ROV Tournament")).thenReturn(false);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.existsByName("ROV Tournament")).thenReturn(false);
                // 👉 บรรทัด `when(games.findById(2L)).thenReturn(Optional.of(rov));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(games.findById(2L)).thenReturn(Optional.of(rov));

                // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                assertThrows(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `ValidationException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                ValidationException.class,
                                // 👉 บรรทัด `() -> service.create(request));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                () -> service.create(request));
        }

        // ---------- กลุ่ม 2: เคส "ต้องสำเร็จ" ----------

        // 👉 เคส: สร้างรายการ Free Fire แบบ POINTS ถูกต้อง ต้องสำเร็จ
        @Test
        // 👉 บรรทัดนี้ `void createsFreeFirePointsTournamentSuccessfully() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void createsFreeFirePointsTournamentSuccessfully() {
                // 👉 บรรทัดนี้ `TournamentRequest request = request(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                TournamentRequest request = request(
                                // 👉 ข้อความนี้เป็นรายละเอียดของข้อผิดพลาด ช่วยบอก client ว่าข้อมูลผิดกฎข้อไหน
                                "Free Fire Tournament",
                                // 👉 เป็นค่าหรือพารามิเตอร์ `1L` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                1L,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `TournamentFormat.POINTS` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                TournamentFormat.POINTS,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `(short) 10` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                (short) 10,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `LocalDate.of(2026, 10, 20)` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกเมธอด `LocalDate.of(2026, 10, 21))`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                                LocalDate.of(2026, 10, 21));

                // 👉 บรรทัด `when(tournaments.existsByName("Free Fire Tournament")).thenReturn(false);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.existsByName("Free Fire Tournament")).thenReturn(false);
                // 👉 บรรทัด `when(games.findById(1L)).thenReturn(Optional.of(freeFire));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(games.findById(1L)).thenReturn(Optional.of(freeFire));

                // 👉 สั่ง mock ของ saveAndFlush: ตอนบันทึก ให้ "ใส่ id = 1 ให้ object แล้วคืนกลับ"
                //    (จำลองสิ่งที่ฐานข้อมูลจริงทำ คือสร้าง id ให้)
                //    thenAnswer = ตอบด้วยโค้ดที่เราเขียนเอง  invocation.getArgument(0) = อาร์กิวเมนต์ตัวแรกที่ถูกส่งเข้ามา
                // 👉 บรรทัด `when(tournaments.saveAndFlush(any(Tournament.class)))` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.saveAndFlush(any(Tournament.class)))
                                // 👉 บรรทัดนี้ `.thenAnswer(invocation -> {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                                .thenAnswer(invocation -> {
                                        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `invocation.getArgument(0)` เพื่อใช้ในขั้นตอนถัดไป
                                        Tournament tournament = invocation.getArgument(0);
                                        // 👉 เรียก `tournament.setId(1L)` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                                        tournament.setId(1L);
                                        // 👉 คืนค่า `tournament` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
                                        return tournament;
                                });

                var response = service.create(request);   // 👉 Act: สร้างจริง (var = ให้ Java เดาชนิดเอง)

                // 👉 Assert: ตรวจค่าที่ตอบกลับ
                // 👉 บรรทัด `assertEquals(1L, response.id());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals(1L, response.id());
                // 👉 บรรทัด `assertEquals("Free Fire Tournament", response.name());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals("Free Fire Tournament", response.name());
                // 👉 บรรทัด `assertEquals(TournamentFormat.POINTS, response.format());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals(TournamentFormat.POINTS, response.format());
                // 👉 บรรทัด `assertEquals((short) 10, response.totalGames());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals((short) 10, response.totalGames());
                // 👉 บรรทัด `assertEquals("FREE_FIRE", freeFire.getCode());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals("FREE_FIRE", freeFire.getCode());

                // 👉 ตรวจว่า saveAndFlush ถูกเรียกจริง 1 ครั้ง (any(...) = ด้วย Tournament ตัวไหนก็ได้)
                // 👉 บรรทัด `verify(tournaments).saveAndFlush(any(Tournament.class));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                verify(tournaments).saveAndFlush(any(Tournament.class));
        }

        // 👉 เคส: สร้างรายการแพ้คัดออก (ROV) ถูกต้อง ต้องสำเร็จ และ totalGames ต้องเป็น null
        @Test
        // 👉 บรรทัดนี้ `void createsEliminationTournamentSuccessfully() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void createsEliminationTournamentSuccessfully() {
                // 👉 บรรทัดนี้ `TournamentRequest request = request(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                TournamentRequest request = request(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `"ROV Tournament"` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                "ROV Tournament",
                                // 👉 เป็นค่าหรือพารามิเตอร์ `2L` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                2L,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `TournamentFormat.SINGLE_ELIMINATION` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                TournamentFormat.SINGLE_ELIMINATION,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `null` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                null,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `LocalDate.of(2026, 10, 20)` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกเมธอด `LocalDate.of(2026, 10, 21))`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                                LocalDate.of(2026, 10, 21));

                // 👉 บรรทัด `when(tournaments.existsByName("ROV Tournament")).thenReturn(false);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.existsByName("ROV Tournament")).thenReturn(false);
                // 👉 บรรทัด `when(games.findById(2L)).thenReturn(Optional.of(rov));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(games.findById(2L)).thenReturn(Optional.of(rov));

                // 👉 บรรทัด `when(tournaments.saveAndFlush(any(Tournament.class)))` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.saveAndFlush(any(Tournament.class)))
                                // 👉 บรรทัดนี้ `.thenAnswer(invocation -> {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                                .thenAnswer(invocation -> {
                                        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `invocation.getArgument(0)` เพื่อใช้ในขั้นตอนถัดไป
                                        Tournament tournament = invocation.getArgument(0);
                                        // 👉 เรียก `tournament.setId(2L)` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                                        tournament.setId(2L);
                                        // 👉 คืนค่า `tournament` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
                                        return tournament;
                                });

                // 👉 กำหนดค่าจาก `service.create(request)` ให้ `response`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
                var response = service.create(request);

                // 👉 บรรทัด `assertEquals(2L, response.id());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals(2L, response.id());
                // 👉 บรรทัด `assertEquals("ROV Tournament", response.name());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals("ROV Tournament", response.name());
                // 👉 บรรทัด `assertEquals(TournamentFormat.SINGLE_ELIMINATION, response.format());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals(TournamentFormat.SINGLE_ELIMINATION, response.format());
                // 👉 บรรทัด `assertEquals(null, response.totalGames());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals(null, response.totalGames());

                // 👉 บรรทัด `verify(tournaments).saveAndFlush(any(Tournament.class));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                verify(tournaments).saveAndFlush(any(Tournament.class));
        }

        // ---------- กลุ่ม 3: คะแนนอันดับ FFWS ----------

        // 👉 เคส: สร้างรายการ POINTS แล้วระบบต้อง "บันทึกคะแนนอันดับ 12 แถว" ให้เอง
        @Test
        // 👉 บรรทัดนี้ `void createsFfwsPlacementPointsAutomatically() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void createsFfwsPlacementPointsAutomatically() {
                // 👉 บรรทัดนี้ `TournamentRequest request = request(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                TournamentRequest request = request(
                                // 👉 ข้อความนี้เป็นรายละเอียดของข้อผิดพลาด ช่วยบอก client ว่าข้อมูลผิดกฎข้อไหน
                                "Free Fire FFWS Tournament",
                                // 👉 เป็นค่าหรือพารามิเตอร์ `1L` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                1L,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `TournamentFormat.POINTS` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                TournamentFormat.POINTS,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `(short) 10` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                (short) 10,
                                // 👉 เป็นค่าหรือพารามิเตอร์ `LocalDate.of(2026, 10, 20)` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                LocalDate.of(2026, 10, 20),
                                // 👉 เรียกเมธอด `LocalDate.of(2026, 10, 21))`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
                                LocalDate.of(2026, 10, 21));

                // 👉 บรรทัด `when(tournaments.existsByName("Free Fire FFWS Tournament"))` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
                when(tournaments.existsByName("Free Fire FFWS Tournament"))
                                // 👉 บรรทัด `.thenReturn(false);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                .thenReturn(false);

                // 👉 บรรทัด `when(games.findById(1L))` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(games.findById(1L))
                                // 👉 บรรทัด `.thenReturn(Optional.of(freeFire));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                .thenReturn(Optional.of(freeFire));

                // 👉 บรรทัด `when(tournaments.saveAndFlush(any(Tournament.class)))` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.saveAndFlush(any(Tournament.class)))
                                // 👉 บรรทัดนี้ `.thenAnswer(invocation -> {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                                .thenAnswer(invocation -> {
                                        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `invocation.getArgument(0)` เพื่อใช้ในขั้นตอนถัดไป
                                        Tournament tournament = invocation.getArgument(0);
                                        // 👉 เรียก `tournament.setId(1L)` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                                        tournament.setId(1L);
                                        // 👉 คืนค่า `tournament` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
                                        return tournament;
                                });

                // 👉 กำหนดค่าจาก `service.create(request)` ให้ `response`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
                var response = service.create(request);

                // 👉 บรรทัด `assertEquals(1L, response.id());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals(1L, response.id());
                // 👉 บรรทัด `assertEquals(TournamentFormat.POINTS, response.format());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals(TournamentFormat.POINTS, response.format());

                // 👉 ตรวจว่า save() ของ Repository คะแนนอันดับถูกเรียก "พอดี 12 ครั้ง" (อันดับ 1–12)
                verify(tournamentPlacementPoints, times(12))
                                // 👉 บรรทัด `.save(any(TournamentPlacementPoint.class));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                .save(any(TournamentPlacementPoint.class));
        }

        // 👉 เคส: API ดูคะแนน ต้องคืนเรียงอันดับ 1,2,3 และค่าถูกต้อง (12, 9, 8)
        @Test
        // 👉 บรรทัดนี้ `void returnsFfwsPlacementPointsInAscendingOrder() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void returnsFfwsPlacementPointsInAscendingOrder() {

                // 👉 ประกาศตัวแปร `tournament` เพื่ออ้างถึง object `Tournament` ที่เพิ่งสร้าง หรือข้อมูลที่ได้จากนิพจน์ `new Tournament()`
                Tournament tournament = new Tournament();
                // 👉 เรียก `tournament.setId(1L)` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                tournament.setId(1L);

                // 👉 เตรียมข้อมูลปลอม 3 แถว (อันดับ 1=12, 2=9, 3=8) โดยใช้เมธอดช่วย placementPoint(...) ด้านล่าง
                // 👉 ประกาศตัวแปร `first` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `placementPoint(tournament, (short) 1, (short) 12)` เพื่อใช้ในขั้นตอนถัดไป
                TournamentPlacementPoint first = placementPoint(tournament, (short) 1, (short) 12);

                // 👉 ประกาศตัวแปร `second` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `placementPoint(tournament, (short) 2, (short) 9)` เพื่อใช้ในขั้นตอนถัดไป
                TournamentPlacementPoint second = placementPoint(tournament, (short) 2, (short) 9);

                // 👉 ประกาศตัวแปร `third` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `placementPoint(tournament, (short) 3, (short) 8)` เพื่อใช้ในขั้นตอนถัดไป
                TournamentPlacementPoint third = placementPoint(tournament, (short) 3, (short) 8);

                // 👉 สั่ง mock: รายการ id 1 มีอยู่จริง และ Repository คืน 3 แถวนี้
                // 👉 บรรทัด `when(tournaments.findById(1L))` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.findById(1L))
                                // 👉 บรรทัด `.thenReturn(Optional.of(tournament));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                .thenReturn(Optional.of(tournament));

                // 👉 บรรทัดนี้ `when(tournamentPlacementPoints` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                when(tournamentPlacementPoints
                                // 👉 เรียก query method `.findByTournamentIdOrderByPlacementAsc(1L))` ที่ Repository; Spring Data แปลชื่อเมธอดเป็นเงื่อนไขค้นหาในฐานข้อมูล
                                .findByTournamentIdOrderByPlacementAsc(1L))
                                // 👉 บรรทัด `.thenReturn(List.of(first, second, third));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                .thenReturn(List.of(first, second, third));

                // 👉 กำหนดค่าจาก `service.getPlacementPoints(1L)` ให้ `result`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
                List<PlacementPointResponse> result = service.getPlacementPoints(1L);

                // 👉 ตรวจ: มี 3 แถว และเรียงตามลำดับ พร้อมคะแนนถูกต้อง
                // 👉 บรรทัด `assertEquals(3, result.size());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals(3, result.size());

                // 👉 บรรทัด `assertEquals((short) 1, result.get(0).placement());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals((short) 1, result.get(0).placement());
                // 👉 บรรทัด `assertEquals((short) 12, result.get(0).points());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals((short) 12, result.get(0).points());

                // 👉 บรรทัด `assertEquals((short) 2, result.get(1).placement());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals((short) 2, result.get(1).placement());
                // 👉 บรรทัด `assertEquals((short) 9, result.get(1).points());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals((short) 9, result.get(1).points());

                // 👉 บรรทัด `assertEquals((short) 3, result.get(2).placement());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals((short) 3, result.get(2).placement());
                // 👉 บรรทัด `assertEquals((short) 8, result.get(2).points());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals((short) 8, result.get(2).points());

                // 👉 ตรวจว่า Service เรียกเมธอดที่ "เรียงลำดับ" จริง (ไม่ใช่เมธอดอื่น)
                verify(tournamentPlacementPoints)
                                // 👉 บรรทัด `.findByTournamentIdOrderByPlacementAsc(1L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                .findByTournamentIdOrderByPlacementAsc(1L);
        }

        // 👉 เคส: ขอดูคะแนนของรายการที่ไม่มีอยู่ (id 999) ต้อง throw 404
        @Test
        // 👉 บรรทัดนี้ `void rejectsPlacementPointsWhenTournamentDoesNotExist() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void rejectsPlacementPointsWhenTournamentDoesNotExist() {

                // 👉 บรรทัด `when(tournaments.findById(999L))` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                when(tournaments.findById(999L))
                                .thenReturn(Optional.empty());   // 👉 Optional.empty() = "ไม่เจอ"

                // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                assertThrows(
                                // 👉 เป็นค่าหรือพารามิเตอร์ `ResourceNotFoundException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                                ResourceNotFoundException.class,
                                // 👉 บรรทัด `() -> service.getPlacementPoints(999L));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                () -> service.getPlacementPoints(999L));
        }

        // ---------- กลุ่ม 4: ค้นหาตามชื่อ ----------

        // 👉 เคส: ค้นด้วย "fire" ต้องได้ "Free Fire Championship"
        @Test
        // 👉 บรรทัดนี้ `void searchesTournamentsByPartialNameIgnoringCase() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void searchesTournamentsByPartialNameIgnoringCase() {
                // 👉 ประกาศตัวแปร `freeFireTournament` เพื่ออ้างถึง object `Tournament` ที่เพิ่งสร้าง หรือข้อมูลที่ได้จากนิพจน์ `new Tournament()`
                Tournament freeFireTournament = new Tournament();
                // 👉 เรียก `freeFireTournament.setId(1L)` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                freeFireTournament.setId(1L);
                // 👉 เรียก `freeFireTournament.setName("Free Fire Championship")` เพื่อกำหนดค่า ชื่อรายการที่ผ่านการตัดช่องว่างแล้ว; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                freeFireTournament.setName("Free Fire Championship");

                // Set the game because the response includes the game ID.
                // 👉 ต้องตั้งเกมให้ Entity เพราะ toResponse เรียก getGame().getId()
                // 👉 เรียก `freeFireTournament.setGame(freeFire)` เพื่อกำหนดค่า object เกมที่ค้นพบจาก gameId; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                freeFireTournament.setGame(freeFire);

                // 👉 บรรทัด `when(tournaments.findByNameContainingIgnoreCase("fire"))` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
                when(tournaments.findByNameContainingIgnoreCase("fire"))
                                // 👉 บรรทัด `.thenReturn(List.of(freeFireTournament));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                .thenReturn(List.of(freeFireTournament));

                // 👉 กำหนดค่าจาก `service.searchByName("fire")` ให้ `result`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
                List<TournamentResponse> result = service.searchByName("fire");

                // 👉 บรรทัด `assertEquals(1, result.size());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals(1, result.size());
                // 👉 บรรทัด `assertEquals("Free Fire Championship", result.get(0).name());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals("Free Fire Championship", result.get(0).name());

                // 👉 บรรทัด `verify(tournaments).findByNameContainingIgnoreCase("fire");` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                verify(tournaments).findByNameContainingIgnoreCase("fire");
        }

        // 👉 เคส: ค้นด้วย "  ROV  " (มีช่องว่าง) Service ต้องตัดช่องว่างก่อนส่งให้ Repository ("ROV")
        @Test
        // 👉 บรรทัดนี้ `void trimsSearchKeywordBeforeSearching() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        void trimsSearchKeywordBeforeSearching() {
                // 👉 บรรทัด `when(tournaments.findByNameContainingIgnoreCase("ROV"))` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
                when(tournaments.findByNameContainingIgnoreCase("ROV"))
                                // 👉 บรรทัด `.thenReturn(List.of());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                                .thenReturn(List.of());

                // 👉 กำหนดค่าจาก `service.searchByName("  ROV  ")` ให้ `result`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
                List<TournamentResponse> result = service.searchByName("  ROV  ");

                // 👉 บรรทัด `assertEquals(0, result.size());` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                assertEquals(0, result.size());

                // 👉 ตรวจว่า Repository ถูกเรียกด้วย "ROV" (ไม่มีช่องว่าง) = พิสูจน์ว่า trim ทำงาน
                // 👉 บรรทัด `verify(tournaments).findByNameContainingIgnoreCase("ROV");` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
                verify(tournaments).findByNameContainingIgnoreCase("ROV");
        }

        // ---------- เมธอดช่วยสร้างข้อมูลทดสอบ (ไม่ใช่เทสต์) ----------

        // 👉 สร้างแถวคะแนนอันดับ 1 แถว
        private static TournamentPlacementPoint placementPoint(
                        // 👉 เป็นค่าหรือพารามิเตอร์ `Tournament tournament` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                        Tournament tournament,
                        // 👉 ฟิลด์/พารามิเตอร์นี้ระบุค่าตัวเลขของข้อมูลที่ record หรือเมธอดกำลังเก็บ
                        Short placement,
                        // 👉 ฟิลด์/พารามิเตอร์นี้ระบุค่าตัวเลขของข้อมูลที่ record หรือเมธอดกำลังเก็บ
                        Short points) {

                // 👉 ประกาศตัวแปร `placementPoint` เพื่ออ้างถึง object `TournamentPlacementPoint` ที่เพิ่งสร้าง หรือข้อมูลที่ได้จากนิพจน์ `new TournamentPlacementPoint()`
                TournamentPlacementPoint placementPoint = new TournamentPlacementPoint();

                // 👉 เรียก `placementPoint.setTournament(tournament)` เพื่อกำหนดค่า ทัวร์นาเมนต์ที่คะแนนอันดับนี้สังกัด; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                placementPoint.setTournament(tournament);
                // 👉 เรียก `placementPoint.setPlacement(placement)` เพื่อกำหนดค่า อันดับของทีม; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                placementPoint.setPlacement(placement);
                // 👉 เรียก `placementPoint.setPoints(points)` เพื่อกำหนดค่า คะแนนที่กำหนดให้แก่อันดับนั้น; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                placementPoint.setPoints(points);

                // 👉 คืนค่า `placementPoint` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
                return placementPoint;
        }

        // 👉 สร้างคำขอสร้างรายการ (ลดการเขียนซ้ำในทุกเทสต์)
        private static TournamentRequest request(
                        // 👉 พารามิเตอร์นี้เป็นข้อมูลที่เมธอดรับเข้ามา; ชนิดก่อนชื่อตัวแปรกำหนดว่าผู้เรียกต้องส่งข้อมูลชนิดใด
                        String name,
                        // 👉 เป็นค่าหรือพารามิเตอร์ `Long gameId` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
                        Long gameId,
                        // 👉 พารามิเตอร์นี้เก็บรูปแบบการแข่งขัน เช่น `POINTS` หรือ `SINGLE_ELIMINATION`
                        TournamentFormat format,
                        // 👉 พารามิเตอร์นี้เก็บจำนวนเกมทั้งหมดของรายการแบบสะสมคะแนน
                        Short totalGames,
                        // 👉 พารามิเตอร์นี้เก็บวันที่เริ่มแข่งขัน โดย `LocalDate` ไม่มีส่วนเวลา
                        LocalDate startDate,
                        // 👉 บรรทัดนี้ `LocalDate endDate) {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
                        LocalDate endDate) {

                // 👉 สร้าง object ชนิด `TournamentRequest` ใหม่ แล้วเก็บไว้ในตัวแปร `request`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
                TournamentRequest request = new TournamentRequest();

                // 👉 เรียก `request.setName(name)` เพื่อกำหนดค่า ชื่อรายการที่ผ่านการตัดช่องว่างแล้ว; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                request.setName(name);
                // 👉 เรียก `request.setGameId(gameId)` เพื่อกำหนดค่า ฟิลด์ gameId; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                request.setGameId(gameId);
                // 👉 เรียก `request.setFormat(format)` เพื่อกำหนดค่า รูปแบบการแข่งขันที่ผู้ใช้เลือก; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                request.setFormat(format);
                // 👉 เรียก `request.setTotalGames(totalGames)` เพื่อกำหนดค่า จำนวนเกมทั้งหมด (สำคัญกับ format POINTS); setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                request.setTotalGames(totalGames);
                // 👉 เรียก `request.setStartDate(startDate)` เพื่อกำหนดค่า วันที่เริ่มแข่งขัน; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                request.setStartDate(startDate);
                // 👉 เรียก `request.setEndDate(endDate)` เพื่อกำหนดค่า วันที่สิ้นสุดการแข่งขัน; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                request.setEndDate(endDate);

                // 👉 คืนค่า `request` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
                return request;
        }

        // 👉 สร้างเกมจำลอง
        private static Game game(Long id, String code, int minPlayers) {
                // 👉 ประกาศตัวแปร `game` เพื่ออ้างถึง object `Game` ที่เพิ่งสร้าง หรือข้อมูลที่ได้จากนิพจน์ `new Game()`
                Game game = new Game();

                // 👉 เรียก `game.setId(id)` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                game.setId(id);
                // 👉 เรียก `game.setCode(code)` เพื่อกำหนดค่า ฟิลด์ code; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                game.setCode(code);
                // 👉 เรียก `game.setMinPlayers(minPlayers)` เพื่อกำหนดค่า ฟิลด์ minPlayers; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                game.setMinPlayers(minPlayers);
                // 👉 เรียก `game.setIsActive(true)` เพื่อกำหนดค่า ฟิลด์ isActive; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
                game.setIsActive(true);

                // 👉 คืนค่า `game` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
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
// 👉 บรรทัดนี้ `class TeamExistsRuleTest {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
class TeamExistsRuleTest {

    // 👉 เคส: ทีมมีอยู่ → ผ่าน
    @Test
    // 👉 บรรทัดนี้ `void shouldPassWhenTeamExists() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldPassWhenTeamExists() {
        TeamRepository repo = mock(TeamRepository.class);   // 👉 Repository ปลอม
        Team team = mock(Team.class);                       // 👉 ทีมปลอม
        Tournament tournament = mock(Tournament.class);     // 👉 รายการปลอม

        when(team.getId()).thenReturn(10L);        // 👉 ทีมปลอมนี้มี id = 10
        when(repo.existsById(10L)).thenReturn(true);   // 👉 ฐานข้อมูลปลอมตอบว่า "มีทีม id 10"

        // 👉 สร้าง object ชนิด `TeamExistsRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamExistsRule rule = new TeamExistsRule(repo);

        assertDoesNotThrow(() -> rule.validate(team, tournament));   // 👉 ต้องไม่ throw
    }

    // 👉 เคส: ทีมไม่มี → throw 404
    @Test
    // 👉 บรรทัดนี้ `void shouldThrowWhenTeamDoesNotExist() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldThrowWhenTeamDoesNotExist() {
        // 👉 กำหนดค่าจาก `mock(TeamRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        TeamRepository repo = mock(TeamRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 บรรทัด `when(team.getId()).thenReturn(10L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(team.getId()).thenReturn(10L);
        when(repo.existsById(10L)).thenReturn(false);   // 👉 ตอบว่า "ไม่มี"

        // 👉 สร้าง object ชนิด `TeamExistsRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamExistsRule rule = new TeamExistsRule(repo);

        // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        assertThrows(
            // 👉 เป็นค่าหรือพารามิเตอร์ `ResourceNotFoundException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            ResourceNotFoundException.class,
            // 👉 บรรทัด `() -> rule.validate(team, tournament)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            () -> rule.validate(team, tournament)
        );
    }
}```

### `TournamentExistsRuleTest` (ด่าน 2)

```java
// 👉 บรรทัดนี้ `class TournamentExistsRuleTest {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
class TournamentExistsRuleTest {

    // 👉 รายการมีอยู่ → ผ่าน
    @Test
    // 👉 บรรทัดนี้ `void shouldPassWhenTournamentExists() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldPassWhenTournamentExists() {
        // 👉 กำหนดค่าจาก `mock(TournamentRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        TournamentRepository repo = mock(TournamentRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 บรรทัด `when(tournament.getId()).thenReturn(20L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getId()).thenReturn(20L);
        // 👉 บรรทัด `when(repo.existsById(20L)).thenReturn(true);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(repo.existsById(20L)).thenReturn(true);

        // 👉 สร้าง object ชนิด `TournamentExistsRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TournamentExistsRule rule = new TournamentExistsRule(repo);

        // 👉 บรรทัด `assertDoesNotThrow(() -> rule.validate(team, tournament));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 รายการไม่มี → throw 404
    @Test
    // 👉 บรรทัดนี้ `void shouldThrowWhenTournamentDoesNotExist() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldThrowWhenTournamentDoesNotExist() {
        // 👉 กำหนดค่าจาก `mock(TournamentRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        TournamentRepository repo = mock(TournamentRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 บรรทัด `when(tournament.getId()).thenReturn(20L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getId()).thenReturn(20L);
        // 👉 บรรทัด `when(repo.existsById(20L)).thenReturn(false);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(repo.existsById(20L)).thenReturn(false);

        // 👉 สร้าง object ชนิด `TournamentExistsRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TournamentExistsRule rule = new TournamentExistsRule(repo);

        // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        assertThrows(
            // 👉 เป็นค่าหรือพารามิเตอร์ `ResourceNotFoundException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            ResourceNotFoundException.class,
            // 👉 บรรทัด `() -> rule.validate(team, tournament)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            () -> rule.validate(team, tournament)
        );
    }
}```

### `TournamentStatusRuleTest` (ด่าน 3)

```java
// 👉 บรรทัดนี้ `class TournamentStatusRuleTest {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
class TournamentStatusRuleTest {

    // 👉 UPCOMING และยังไม่มีแมตช์ → ผ่าน
    @Test
    // 👉 บรรทัดนี้ `void shouldPassWhenUpcomingAndNoMatchesExist() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldPassWhenUpcomingAndNoMatchesExist() {
        // 👉 กำหนดค่าจาก `mock(MatchRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        MatchRepository repo = mock(MatchRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 บรรทัด `when(tournament.getId()).thenReturn(20L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getId()).thenReturn(20L);
        // 👉 บรรทัด `when(tournament.getStatus()).thenReturn(TournamentStatus.UPCOMING);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getStatus()).thenReturn(TournamentStatus.UPCOMING);
        when(repo.existsByTournamentId(20L)).thenReturn(false);   // 👉 ยังไม่มีแมตช์

        // 👉 สร้าง object ชนิด `TournamentStatusRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TournamentStatusRule rule = new TournamentStatusRule(repo);

        // 👉 บรรทัด `assertDoesNotThrow(() -> rule.validate(team, tournament));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 สถานะไม่ใช่ UPCOMING (ที่นี่จำลองด้วย null) → throw และต้องไม่แตะ repo เลย (ล้มก่อนถึงขั้นเช็คแมตช์)
    @Test
    // 👉 บรรทัดนี้ `void shouldThrowWhenTournamentIsNotUpcoming() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldThrowWhenTournamentIsNotUpcoming() {
        // 👉 กำหนดค่าจาก `mock(MatchRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        MatchRepository repo = mock(MatchRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 บรรทัด `when(tournament.getStatus()).thenReturn(null);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getStatus()).thenReturn(null);

        // 👉 สร้าง object ชนิด `TournamentStatusRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TournamentStatusRule rule = new TournamentStatusRule(repo);

        // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        assertThrows(
            // 👉 เป็นค่าหรือพารามิเตอร์ `BusinessException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            BusinessException.class,
            // 👉 บรรทัด `() -> rule.validate(team, tournament)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            () -> rule.validate(team, tournament)
        );

        verifyNoInteractions(repo);   // 👉 ตรวจว่า repo ไม่ถูกเรียกเลยสักครั้ง
    }

    // 👉 UPCOMING แต่มีแมตช์แล้ว (สร้างตารางแข่งแล้ว) → throw
    @Test
    // 👉 บรรทัดนี้ `void shouldThrowWhenMatchesAlreadyExist() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldThrowWhenMatchesAlreadyExist() {
        // 👉 กำหนดค่าจาก `mock(MatchRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        MatchRepository repo = mock(MatchRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 บรรทัด `when(tournament.getId()).thenReturn(20L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getId()).thenReturn(20L);
        // 👉 บรรทัด `when(tournament.getStatus()).thenReturn(TournamentStatus.UPCOMING);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getStatus()).thenReturn(TournamentStatus.UPCOMING);
        // 👉 บรรทัด `when(repo.existsByTournamentId(20L)).thenReturn(true);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(repo.existsByTournamentId(20L)).thenReturn(true);

        // 👉 สร้าง object ชนิด `TournamentStatusRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TournamentStatusRule rule = new TournamentStatusRule(repo);

        // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        assertThrows(
            // 👉 เป็นค่าหรือพารามิเตอร์ `BusinessException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            BusinessException.class,
            // 👉 บรรทัด `() -> rule.validate(team, tournament)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            () -> rule.validate(team, tournament)
        );
    }
}```

### `TeamNotAlreadyJoinedRuleTest` (ด่าน 4)

```java
// 👉 บรรทัดนี้ `class TeamNotAlreadyJoinedRuleTest {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
class TeamNotAlreadyJoinedRuleTest {

    // 👉 ยังไม่เคยสมัคร → ผ่าน
    @Test
    // 👉 บรรทัดนี้ `void shouldPassWhenTeamHasNotJoined() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldPassWhenTeamHasNotJoined() {
        // 👉 กำหนดค่าจาก `mock(TournamentTeamRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 บรรทัด `when(team.getId()).thenReturn(10L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(team.getId()).thenReturn(10L);
        // 👉 บรรทัด `when(tournament.getId()).thenReturn(20L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getId()).thenReturn(20L);
        // 👉 คีย์คู่ (รายการ 20, ทีม 10) ไม่มีอยู่ในฐานข้อมูล
        // 👉 บรรทัด `when(repo.existsById(new TournamentTeamId(20L, 10L))).thenReturn(false);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(repo.existsById(new TournamentTeamId(20L, 10L))).thenReturn(false);

        // 👉 สร้าง object ชนิด `TeamNotAlreadyJoinedRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamNotAlreadyJoinedRule rule = new TeamNotAlreadyJoinedRule(repo);

        // 👉 บรรทัด `assertDoesNotThrow(() -> rule.validate(team, tournament));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 สมัครไว้แล้ว → throw 409
    @Test
    // 👉 บรรทัดนี้ `void shouldThrowWhenTeamAlreadyJoined() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldThrowWhenTeamAlreadyJoined() {
        // 👉 กำหนดค่าจาก `mock(TournamentTeamRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 บรรทัด `when(team.getId()).thenReturn(10L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(team.getId()).thenReturn(10L);
        // 👉 บรรทัด `when(tournament.getId()).thenReturn(20L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getId()).thenReturn(20L);
        // 👉 บรรทัด `when(repo.existsById(new TournamentTeamId(20L, 10L))).thenReturn(true);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(repo.existsById(new TournamentTeamId(20L, 10L))).thenReturn(true);

        // 👉 สร้าง object ชนิด `TeamNotAlreadyJoinedRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamNotAlreadyJoinedRule rule = new TeamNotAlreadyJoinedRule(repo);

        // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        assertThrows(
            // 👉 เป็นค่าหรือพารามิเตอร์ `BusinessException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            BusinessException.class,
            // 👉 บรรทัด `() -> rule.validate(team, tournament)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            () -> rule.validate(team, tournament)
        );
    }
}```

### `TeamGameMatchRuleTest` (ด่าน 5) — commit `test: add team game match rule tests`

```java
// 👉 บรรทัดนี้ `class TeamGameMatchRuleTest {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
class TeamGameMatchRuleTest {

    // 👉 บรรทัด `private TeamGameMatchRule rule;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private TeamGameMatchRule rule;
    // 👉 บรรทัด `private Team team;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private Team team;
    // 👉 บรรทัด `private Tournament tournament;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private Tournament tournament;
    // 👉 บรรทัด `private Game teamGame;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private Game teamGame;
    // 👉 บรรทัด `private Game tournamentGame;` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    private Game tournamentGame;

    // 👉 ก่อนทุกเทสต์: ตั้งให้ทีมและรายการเล่น "เกม id 1" เหมือนกัน (สถานะตั้งต้น = ผ่าน)
    @BeforeEach
    // 👉 บรรทัดนี้ `void setUp() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void setUp() {
        rule = new TeamGameMatchRule();   // 👉 ด่านนี้ไม่ใช้ Repository จึง new ตรง ๆ ได้ ไม่ต้อง mock

        // 👉 สร้าง object ชนิด `Game` ใหม่ แล้วเก็บไว้ในตัวแปร `teamGame`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        teamGame = new Game();
        // 👉 เรียก `teamGame.setId(1L)` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        teamGame.setId(1L);

        // 👉 สร้าง object ชนิด `Game` ใหม่ แล้วเก็บไว้ในตัวแปร `tournamentGame`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        tournamentGame = new Game();
        // 👉 เรียก `tournamentGame.setId(1L)` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournamentGame.setId(1L);

        // 👉 สร้าง object ชนิด `Team` ใหม่ แล้วเก็บไว้ในตัวแปร `team`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        team = new Team();
        // 👉 เรียก `team.setId(10L)` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        team.setId(10L);
        // 👉 เรียก `team.setGame(teamGame)` เพื่อกำหนดค่า object เกมที่ค้นพบจาก gameId; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        team.setGame(teamGame);

        // 👉 สร้าง object ชนิด `Tournament` ใหม่ แล้วเก็บไว้ในตัวแปร `tournament`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        tournament = new Tournament();
        // 👉 เรียก `tournament.setId(20L)` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setId(20L);
        // 👉 เรียก `tournament.setGame(tournamentGame)` เพื่อกำหนดค่า object เกมที่ค้นพบจาก gameId; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setGame(tournamentGame);
        // 👉 เรียก `tournament.setFormat(TournamentFormat.SINGLE_ELIMINATION)` เพื่อกำหนดค่า รูปแบบการแข่งขันที่ผู้ใช้เลือก; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setFormat(TournamentFormat.SINGLE_ELIMINATION);
    }

    // 👉 เกมตรงกัน → ผ่าน
    @Test
    // 👉 บรรทัดนี้ `void acceptsWhenTeamGameMatchesTournamentGame() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void acceptsWhenTeamGameMatchesTournamentGame() {
        // 👉 บรรทัด `assertDoesNotThrow(() -> rule.validate(team, tournament));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 เปลี่ยนเกมของรายการเป็น id 2 (ไม่ตรง) → ปฏิเสธ
    @Test
    // 👉 บรรทัดนี้ `void rejectsWhenTeamGameDoesNotMatchTournamentGame() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void rejectsWhenTeamGameDoesNotMatchTournamentGame() {
        // 👉 เรียก `tournamentGame.setId(2L)` เพื่อกำหนดค่า ฟิลด์ id; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournamentGame.setId(2L);

        // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        assertThrows(
            // 👉 เป็นค่าหรือพารามิเตอร์ `BusinessException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            BusinessException.class,
            // 👉 บรรทัด `() -> rule.validate(team, tournament)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            () -> rule.validate(team, tournament)
        );
    }

    // 👉 ทีมไม่มีเกม (null) → ปฏิเสธ
    @Test
    // 👉 บรรทัดนี้ `void rejectsWhenTeamHasNoGame() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void rejectsWhenTeamHasNoGame() {
        // 👉 เรียก `team.setGame(null)` เพื่อกำหนดค่า object เกมที่ค้นพบจาก gameId; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        team.setGame(null);

        // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        assertThrows(
            // 👉 เป็นค่าหรือพารามิเตอร์ `BusinessException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            BusinessException.class,
            // 👉 บรรทัด `() -> rule.validate(team, tournament)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            () -> rule.validate(team, tournament)
        );
    }

    // 👉 รายการไม่มีเกม (null) → ปฏิเสธ
    @Test
    // 👉 บรรทัดนี้ `void rejectsWhenTournamentHasNoGame() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void rejectsWhenTournamentHasNoGame() {
        // 👉 เรียก `tournament.setGame(null)` เพื่อกำหนดค่า object เกมที่ค้นพบจาก gameId; setter นำค่า argument ไปเก็บใน object ไม่ได้บันทึกฐานข้อมูลเองโดยอัตโนมัติ
        tournament.setGame(null);

        // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        assertThrows(
            // 👉 เป็นค่าหรือพารามิเตอร์ `BusinessException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            BusinessException.class,
            // 👉 บรรทัด `() -> rule.validate(team, tournament)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            () -> rule.validate(team, tournament)
        );
    }
}```

### `TeamMinimumPlayersRuleTest` (ด่าน 6)

```java
// 👉 บรรทัดนี้ `class TeamMinimumPlayersRuleTest {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
class TeamMinimumPlayersRuleTest {

    // 👉 ผู้เล่น 5 คน ขั้นต่ำ 5 → ผ่าน (เท่ากับขั้นต่ำถือว่าพอ เพราะเงื่อนไขคือ "น้อยกว่า" จึงปฏิเสธ)
    @Test
    // 👉 บรรทัดนี้ `void shouldPassWhenTeamHasEnoughPlayers() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldPassWhenTeamHasEnoughPlayers() {
        // 👉 กำหนดค่าจาก `mock(PlayerRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        PlayerRepository repo = mock(PlayerRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);
        // 👉 ประกาศตัวแปร `game` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Game.class)` เพื่อใช้ในขั้นตอนถัดไป
        Game game = mock(Game.class);

        // 👉 บรรทัด `when(team.getId()).thenReturn(10L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(team.getId()).thenReturn(10L);
        // 👉 บรรทัด `when(tournament.getGame()).thenReturn(game);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getGame()).thenReturn(game);
        when(game.getMinPlayers()).thenReturn(5);         // 👉 เกมต้องการอย่างน้อย 5 คน
        when(repo.countByTeamId(10L)).thenReturn(5L);     // 👉 ทีมมีจริง 5 คน

        // 👉 สร้าง object ชนิด `TeamMinimumPlayersRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamMinimumPlayersRule rule = new TeamMinimumPlayersRule(repo);

        // 👉 บรรทัด `assertDoesNotThrow(() -> rule.validate(team, tournament));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 ผู้เล่น 4 คน ขั้นต่ำ 5 → ปฏิเสธ
    @Test
    // 👉 บรรทัดนี้ `void shouldThrowWhenTeamHasTooFewPlayers() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldThrowWhenTeamHasTooFewPlayers() {
        // 👉 กำหนดค่าจาก `mock(PlayerRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        PlayerRepository repo = mock(PlayerRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);
        // 👉 ประกาศตัวแปร `game` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Game.class)` เพื่อใช้ในขั้นตอนถัดไป
        Game game = mock(Game.class);

        // 👉 บรรทัด `when(team.getId()).thenReturn(10L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(team.getId()).thenReturn(10L);
        // 👉 บรรทัด `when(tournament.getGame()).thenReturn(game);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getGame()).thenReturn(game);
        // 👉 บรรทัด `when(game.getMinPlayers()).thenReturn(5);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(game.getMinPlayers()).thenReturn(5);
        // 👉 บรรทัด `when(repo.countByTeamId(10L)).thenReturn(4L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(repo.countByTeamId(10L)).thenReturn(4L);

        // 👉 สร้าง object ชนิด `TeamMinimumPlayersRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamMinimumPlayersRule rule = new TeamMinimumPlayersRule(repo);

        // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        assertThrows(
            // 👉 เป็นค่าหรือพารามิเตอร์ `BusinessException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            BusinessException.class,
            // 👉 บรรทัด `() -> rule.validate(team, tournament)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            () -> rule.validate(team, tournament)
        );
    }

    // 👉 เกมไม่ได้กำหนดขั้นต่ำ (null) → ข้ามการตรวจ ผ่านเลย และต้องไม่ไปนับผู้เล่นด้วย
    @Test
    // 👉 บรรทัดนี้ `void shouldPassWhenMinimumPlayersIsNotConfigured() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldPassWhenMinimumPlayersIsNotConfigured() {
        // 👉 กำหนดค่าจาก `mock(PlayerRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        PlayerRepository repo = mock(PlayerRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);
        // 👉 ประกาศตัวแปร `game` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Game.class)` เพื่อใช้ในขั้นตอนถัดไป
        Game game = mock(Game.class);

        // 👉 บรรทัด `when(tournament.getGame()).thenReturn(game);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getGame()).thenReturn(game);
        // 👉 บรรทัด `when(game.getMinPlayers()).thenReturn(null);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(game.getMinPlayers()).thenReturn(null);

        // 👉 สร้าง object ชนิด `TeamMinimumPlayersRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamMinimumPlayersRule rule = new TeamMinimumPlayersRule(repo);

        // 👉 บรรทัด `assertDoesNotThrow(() -> rule.validate(team, tournament));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        assertDoesNotThrow(() -> rule.validate(team, tournament));
        verifyNoInteractions(repo);   // 👉 ยืนยันว่าไม่ได้เรียก repo เลย
    }
}```

### `TeamPointsLimitRuleTest` (ด่าน 7)

```java
// 👉 บรรทัดนี้ `class TeamPointsLimitRuleTest {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
class TeamPointsLimitRuleTest {

    // 👉 POINTS มี 11 ทีม → ยังรับได้ (ทีมที่ 12) → ผ่าน
    @Test
    // 👉 บรรทัดนี้ `void shouldPassWhenPointsTournamentHasFewerThanTwelveTeams() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldPassWhenPointsTournamentHasFewerThanTwelveTeams() {
        // 👉 กำหนดค่าจาก `mock(TournamentTeamRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 บรรทัด `when(tournament.getId()).thenReturn(20L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getId()).thenReturn(20L);
        // 👉 บรรทัด `when(tournament.getFormat()).thenReturn(TournamentFormat.POINTS);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getFormat()).thenReturn(TournamentFormat.POINTS);
        // 👉 บรรทัด `when(repo.countByTournamentId(20L)).thenReturn(11L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(repo.countByTournamentId(20L)).thenReturn(11L);

        // 👉 สร้าง object ชนิด `TeamPointsLimitRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamPointsLimitRule rule = new TeamPointsLimitRule(repo);

        // 👉 บรรทัด `assertDoesNotThrow(() -> rule.validate(team, tournament));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 POINTS มี 12 ทีมแล้ว → เต็ม → ปฏิเสธ
    @Test
    // 👉 บรรทัดนี้ `void shouldThrowWhenPointsTournamentAlreadyHasTwelveTeams() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldThrowWhenPointsTournamentAlreadyHasTwelveTeams() {
        // 👉 กำหนดค่าจาก `mock(TournamentTeamRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 บรรทัด `when(tournament.getId()).thenReturn(20L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getId()).thenReturn(20L);
        // 👉 บรรทัด `when(tournament.getFormat()).thenReturn(TournamentFormat.POINTS);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getFormat()).thenReturn(TournamentFormat.POINTS);
        // 👉 บรรทัด `when(repo.countByTournamentId(20L)).thenReturn(12L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(repo.countByTournamentId(20L)).thenReturn(12L);

        // 👉 สร้าง object ชนิด `TeamPointsLimitRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamPointsLimitRule rule = new TeamPointsLimitRule(repo);

        // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        assertThrows(
            // 👉 เป็นค่าหรือพารามิเตอร์ `BusinessException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            BusinessException.class,
            // 👉 บรรทัด `() -> rule.validate(team, tournament)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
            () -> rule.validate(team, tournament)
        );
    }

    // 👉 รายการแพ้คัดออก → กฎนี้ไม่เกี่ยว ข้ามไป และต้องไม่แตะ repo
    @Test
    // 👉 บรรทัดนี้ `void shouldSkipTeamLimitForNonPointsTournament() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldSkipTeamLimitForNonPointsTournament() {
        // 👉 กำหนดค่าจาก `mock(TournamentTeamRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 บรรทัดนี้ `when(tournament.getFormat()).thenReturn(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        when(tournament.getFormat()).thenReturn(
            // 👉 บรรทัดนี้ `TournamentFormat.SINGLE_ELIMINATION` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            TournamentFormat.SINGLE_ELIMINATION
        );

        // 👉 สร้าง object ชนิด `TeamPointsLimitRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamPointsLimitRule rule = new TeamPointsLimitRule(repo);

        // 👉 บรรทัด `assertDoesNotThrow(() -> rule.validate(team, tournament));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        assertDoesNotThrow(() -> rule.validate(team, tournament));
        // 👉 บรรทัด `verifyNoInteractions(repo);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        verifyNoInteractions(repo);
    }
}```

### `TeamTournamentDateRuleTest` (ด่าน 8) — เคส "วันทับ → ปฏิเสธ" และ "วันไม่ทับ → ผ่าน" ตามที่บรีฟกำหนด

```java
// 👉 บรรทัดนี้ `class TeamTournamentDateRuleTest {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
class TeamTournamentDateRuleTest {

    // 👉 เคส: วันไม่ทับ → ผ่าน
    @Test
    // 👉 บรรทัดนี้ `void shouldPassWhenThereAreNoOverlappingTournaments() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldPassWhenThereAreNoOverlappingTournaments() {
        // 👉 กำหนดค่าจาก `mock(TournamentTeamRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 กำหนดค่าจาก `LocalDate.of(2026, 11, 1)` ให้ `start`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        LocalDate start = LocalDate.of(2026, 11, 1);
        // 👉 กำหนดค่าจาก `LocalDate.of(2026, 11, 10)` ให้ `end`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        LocalDate end = LocalDate.of(2026, 11, 10);

        // 👉 บรรทัด `when(team.getId()).thenReturn(10L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(team.getId()).thenReturn(10L);
        // 👉 บรรทัด `when(tournament.getId()).thenReturn(20L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getId()).thenReturn(20L);
        // 👉 บรรทัด `when(tournament.getStartDate()).thenReturn(start);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getStartDate()).thenReturn(start);
        // 👉 บรรทัด `when(tournament.getEndDate()).thenReturn(end);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getEndDate()).thenReturn(end);

        // 👉 ฐานข้อมูลปลอมตอบว่า "มี 0 รายการที่ชน"
        when(repo.countOverlappingTournaments(
            // 👉 บรรทัดนี้ `10L, 20L, start, end` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            10L, 20L, start, end
        // 👉 บรรทัด `)).thenReturn(0L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        )).thenReturn(0L);

        // 👉 สร้าง object ชนิด `TeamTournamentDateRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamTournamentDateRule rule = new TeamTournamentDateRule(repo);

        // 👉 บรรทัด `assertDoesNotThrow(() -> rule.validate(team, tournament));` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        assertDoesNotThrow(() -> rule.validate(team, tournament));
    }

    // 👉 เคส: วันทับ → ปฏิเสธ
    @Test
    // 👉 บรรทัดนี้ `void shouldThrowWhenTournamentDatesOverlap() {` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
    void shouldThrowWhenTournamentDatesOverlap() {
        // 👉 กำหนดค่าจาก `mock(TournamentTeamRepository.class)` ให้ `repo`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        TournamentTeamRepository repo = mock(TournamentTeamRepository.class);
        // 👉 ประกาศตัวแปร `team` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Team.class)` เพื่อใช้ในขั้นตอนถัดไป
        Team team = mock(Team.class);
        // 👉 ประกาศตัวแปร `tournament` ชนิดข้อมูลตามที่ระบุ และเก็บผลจาก `mock(Tournament.class)` เพื่อใช้ในขั้นตอนถัดไป
        Tournament tournament = mock(Tournament.class);

        // 👉 กำหนดค่าจาก `LocalDate.of(2026, 11, 1)` ให้ `start`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        LocalDate start = LocalDate.of(2026, 11, 1);
        // 👉 กำหนดค่าจาก `LocalDate.of(2026, 11, 10)` ให้ `end`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
        LocalDate end = LocalDate.of(2026, 11, 10);

        // 👉 บรรทัด `when(team.getId()).thenReturn(10L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(team.getId()).thenReturn(10L);
        // 👉 บรรทัด `when(tournament.getId()).thenReturn(20L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getId()).thenReturn(20L);
        // 👉 บรรทัด `when(tournament.getStartDate()).thenReturn(start);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getStartDate()).thenReturn(start);
        // 👉 บรรทัด `when(tournament.getEndDate()).thenReturn(end);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        when(tournament.getEndDate()).thenReturn(end);

        // 👉 ตอบว่า "มี 1 รายการที่ชน"
        when(repo.countOverlappingTournaments(
            // 👉 บรรทัดนี้ `10L, 20L, start, end` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
            10L, 20L, start, end
        // 👉 บรรทัด `)).thenReturn(1L);` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
        )).thenReturn(1L);

        // 👉 สร้าง object ชนิด `TeamTournamentDateRule` ใหม่ แล้วเก็บไว้ในตัวแปร `rule`; ตอนสร้างยังเป็นค่าเริ่มต้นจนกว่าจะเรียก setter หรือ constructor กำหนดข้อมูล
        TeamTournamentDateRule rule = new TeamTournamentDateRule(repo);

        // 👉 บรรทัดนี้ `assertThrows(` เป็นส่วนของตรรกะที่กำลังทำงาน; อ่านร่วมกับคำสั่งก่อนหน้าเพื่อดูว่าค่าหรือผลลัพธ์นี้ถูกนำไปใช้ต่ออย่างไร
        assertThrows(
            // 👉 เป็นค่าหรือพารามิเตอร์ `BusinessException.class` ที่ส่งต่อไปยัง constructor/เมธอดซึ่งเริ่มจากบรรทัดก่อนหน้า
            BusinessException.class,
            // 👉 บรรทัด `() -> rule.validate(team, tournament)` เป็นส่วนปิดของการเรียก/ประกาศที่เริ่มไว้ก่อนหน้า; ค่าที่ส่งเข้ามาจะถูกใช้ตามลำดับพารามิเตอร์
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
// 👉 ประกาศเมธอด `setSeeds(...)` เพื่อรับค่าเข้ามาแล้วกำหนดให้ฟิลด์ `seeds`; `void` หมายถึงไม่ส่งค่ากลับ
public List<TournamentTeamResponse> setSeeds(Long tournamentId, SeedAssignmentRequest request) {

    // 👉 ประกาศตัวแปร `tournament` แล้วค้นหาข้อมูลด้วย `tournamentRepository.findById(tournamentId)`; Repository คืน `Optional` เพราะรหัสนั้นอาจมีหรือไม่มีในฐานข้อมูล
    Tournament tournament = tournamentRepository.findById(tournamentId)
            // 👉 เรียก `orElseThrow(...)` กับ `Optional`: ถ้ามีข้อมูลจะใช้ค่าที่อยู่ข้างใน แต่ถ้าไม่มีจะสร้างและโยน exception ที่กำหนดใน lambda
            .orElseThrow(() -> new ResourceNotFoundException("Tournament not found: " + tournamentId));

    // 👉 จัด seed ได้เฉพาะรายการที่ยังไม่เริ่ม
    // 👉 ตรวจว่าสถานะไม่ใช่ UPCOMING หรือไม่ เพราะห้ามลบรายการที่เริ่มแล้ว; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (tournament.getStatus() != TournamentStatus.UPCOMING) {`
    if (tournament.getStatus() != TournamentStatus.UPCOMING) {
        // 👉 โยน `BusinessException` พร้อมข้อความ `"Seeds can only be changed for upcoming tournaments"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
        throw new BusinessException("Seeds can only be changed for upcoming tournaments");
    }

    // 👉 ตรวจค่าของเงื่อนไข `request == null || request.assignments() == null || request.assignments().isEmpty()`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (request == null || request.assignments() == null || request.assignments().isEmpty()) {`
    if (request == null || request.assignments() == null || request.assignments().isEmpty()) {
        // 👉 โยน `ValidationException` พร้อมข้อความ `"Seed assignments must not be empty"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
        throw new ValidationException("Seed assignments must not be empty");
    }

    // 👉 กำหนดค่าจาก `tournamentTeamRepository.findByTournamentId(tournamentId)` ให้ `entries`; เครื่องหมาย `=` เป็นการกำหนดค่า ไม่ใช่การเปรียบเทียบ
    List<TournamentTeam> entries = tournamentTeamRepository.findByTournamentId(tournamentId);

    // 👉 ทำ Map: teamId → แถวการสมัครของทีมนั้น เพื่อค้นหาเร็ว
    // 👉 บรรทัด `Map<Long, TournamentTeam> entriesByTeamId = entries.stream()` เป็นส่วนหนึ่งของคำสั่งในเมธอดนี้; ความหมายต้องอ่านร่วมกับค่าที่รับเข้าและเงื่อนไขรอบข้าง
    Map<Long, TournamentTeam> entriesByTeamId = entries.stream()
            // 👉 รวบรวมข้อมูลจาก Stream เป็นโครงสร้างข้อมูลปลายทาง
            .collect(Collectors.toMap(entry -> entry.getTeam().getId(), Function.identity()));

    Set<Long> submittedTeamIds = new HashSet<>();   // 👉 เก็บ teamId ที่เจอแล้ว ไว้จับซ้ำ
    Set<Integer> submittedSeeds = new HashSet<>();   // 👉 เก็บ seed ที่เจอแล้ว ไว้จับซ้ำ

    // ตรวจสอบข้อมูล Seed ให้ครบก่อนแก้ไขข้อมูลในฐานข้อมูล
    // 👉 เริ่มลูป `for (SeedAssignmentRequest.SeedItem item : request.assignments())` เพื่อทำคำสั่งในบล็อกซ้ำตามค่าเริ่มต้น เงื่อนไข และการเปลี่ยนค่าของตัวนับ
    for (SeedAssignmentRequest.SeedItem item : request.assignments()) {
        // 👉 ตรวจค่าของเงื่อนไข `item == null || item.teamId() == null || item.seed(`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (item == null || item.teamId() == null || item.seed() == null`
        if (item == null || item.teamId() == null || item.seed() == null
                // 👉 เป็นเงื่อนไขต่อจากบรรทัด `if` ก่อนหน้า โดย `||` หมายถึงมีอย่างน้อยหนึ่งเงื่อนไขเป็นจริงก็เข้าเงื่อนไขได้
                || item.teamId() <= 0 || item.seed() <= 0) {
            // 👉 โยน `ValidationException` พร้อมข้อความ `"Team ID and seed must be positive"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ValidationException("Team ID and seed must be positive");
        }
        if (!submittedTeamIds.add(item.teamId())) {     // 👉 add คืน false ถ้ามีค่านี้อยู่แล้ว = ซ้ำ
            // 👉 โยน `ValidationException` พร้อมข้อความ `"Duplicate team ID: " + item.teamId()` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ValidationException("Duplicate team ID: " + item.teamId());
        }
        // 👉 ตรวจค่าของเงื่อนไข `!submittedSeeds.add(item.seed())`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (!submittedSeeds.add(item.seed())) {`
        if (!submittedSeeds.add(item.seed())) {
            // 👉 โยน `ValidationException` พร้อมข้อความ `"Duplicate seed: " + item.seed()` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ValidationException("Duplicate seed: " + item.seed());
        }
        // 👉 ตรวจค่าของเงื่อนไข `!entriesByTeamId.containsKey(item.teamId())`; ถ้าเงื่อนไขเป็น true จะทำคำสั่งภายในบล็อก `if (!entriesByTeamId.containsKey(item.teamId())) {`
        if (!entriesByTeamId.containsKey(item.teamId())) {
            // 👉 โยน `ValidationException` พร้อมข้อความ `"Team " + item.teamId() + " is not registered in this tournament"` เพื่อหยุด flow ปกติ; ชั้นจัดการ exception จะนำชนิดข้อผิดพลาดไปแปลงเป็น HTTP status ที่เหมาะสม
            throw new ValidationException("Team " + item.teamId() + " is not registered in this tournament");
        }
    }

    // บันทึก Seed หลังจากข้อมูลทั้งหมดผ่านการตรวจสอบแล้ว
    // 👉 ตรวจครบก่อนแล้วค่อยแก้ ทำให้ถ้ามีข้อผิดพลาดจะไม่มี seed ที่ถูกแก้ไปครึ่งเดียว
    // 👉 เริ่มลูป `for (SeedAssignmentRequest.SeedItem item : request.assignments())` เพื่อทำคำสั่งในบล็อกซ้ำตามค่าเริ่มต้น เงื่อนไข และการเปลี่ยนค่าของตัวนับ
    for (SeedAssignmentRequest.SeedItem item : request.assignments()) {
        // 👉 เรียกเมธอด `entriesByTeamId.get(item.teamId()).setSeed(item.seed())`; ค่าภายในวงเล็บเป็น argument ที่ส่งให้เมธอดใช้ทำงาน
        entriesByTeamId.get(item.teamId()).setSeed(item.seed());
    }

    // 👉 บันทึกข้อมูลหลายรายการในครั้งเดียว
    tournamentTeamRepository.saveAll(entries);

    // 👉 คืนค่า `entries.stream().map(this::toResponse).toList()` ให้เมธอดที่เรียกอยู่; หลัง `return` เมธอดนี้จะจบทันที
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
