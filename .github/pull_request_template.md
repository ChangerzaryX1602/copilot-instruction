<!-- PR checklist — ส่วนหนึ่งของชั้น 5 (บังคับด้วยวินัย + review, CI บังคับได้แค่บางข้อ) -->

## What changed

<!-- 1-3 บรรทัด: เปลี่ยนอะไร ทำไม ไม่ต้องเล่า process -->

## Which rules does this touch

<!-- เช่น rule 1 (structure), rule 3 (null safety), rule 13 (responsive) -->

## Verification (ต้องมีของจริง ไม่ใช่คำพูด)

- [ ] `dart format .` แล้ว (ไม่มี diff)
- [ ] `flutter analyze --fatal-infos` สะอาด — ผลจริง: <!-- paste 1 บรรทัด -->
- [ ] `flutter test` ผ่าน — ผลจริง: <!-- paste 1 บรรทัด -->
- [ ] ทดสอบบนอุปกรณ์/emulator จริง: <!-- รุ่น + OS + ขนาดจอ -->

## Rules that need a human (lint จับไม่ได้ — ดู docs/rule-to-lint.md)

- [ ] **rule 1** — ไฟล์อยู่ใน layer ที่ถูก และ dependency direction ไม่ย้อน (`presentation → domain ← data`)
- [ ] **rule 5** — ไม่มี I/O หรือ business logic ใน widget / `build()`
- [ ] **rule 8** — ไม่มี abstraction ที่สร้างมาครั้งเดียว และไม่มีการซ้ำที่ถึง 3 แล้วยังไม่แยก
- [ ] **rule 9** — dependency เข้าทาง constructor ทั้งหมด (ไม่มี `new`/locator ในคลาส)
- [ ] **rule 11** — ใช้ state management เดียวตามที่ pin ไว้ ไม่ผสม
- [ ] **rule 12** — ไม่มี string ที่ผู้ใช้เห็นฝังใน widget (รวม semantics label / ข้อความ error)
- [ ] **rule 13** — ผ่านที่ 320 dp, จอใหญ่/landscape และ `textScaler` 2.0 (แนบหลักฐานถ้าอ้างว่าผ่าน)
- [ ] **rule 3b** — ไม่มี `!` ที่ null ได้จริง และไม่มี `catch` ที่กลืน error
- [ ] **rule 14** — ไม่มี secret/config จริงใน diff (และไม่ได้ลบ config ออกจาก `.gitignore`)

## Risk / rollback

<!-- ความเสี่ยง, feature flag, วิธีย้อนกลับ -->
