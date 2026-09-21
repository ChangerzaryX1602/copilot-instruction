# กฎ → สิ่งที่บังคับ → ช่องโหว่ที่เหลือ

ตารางนี้คือ **ตัวตัดสินว่าใครบังคับอะไร** อ่านคู่กับ `README.md`
สถานะ: ✅ = บังคับด้วยเครื่อง (analyzer/CI/hook) · ~ = บังคับบางส่วน · ✖ = convention + คนรีวิวเท่านั้น

| # | กฎ | สถานะ | บังคับด้วย |
|---|----|-------|-----------|
| 1 | File structure / layering | ✖ | `architecture.instructions.md` + PR checklist · lint ช่วยแค่ `always_use_package_imports`, `directives_ordering` (ไม่รู้จัก layer) |
| 2 | Minimal 3rd-party deps | ~ | `sort_pub_dependencies`, `depend_on_referenced_packages` + CI `flutter pub outdated` · allowlist ใน `dependencies.instructions.md` ยังต้องใช้คนตัดสิน |
| 3 | Null safety & error handling | ✅ | analyzer strict mode `strict-casts`/`strict-inference`/`strict-raw-types` + 15 lint (`avoid_dynamic_calls`, `unawaited_futures`, `discarded_futures`, `only_throw_errors`, `avoid_empty_else`, `avoid_catches_without_on_clauses`, `throw_in_finally`, …) |
| 3b | **ห้ามใช้ `!` (bang)** | ✖ | **ไม่มี lint ทางการตัวไหนห้าม `!`** (เช็คกับชุด linter rules 213 ตัว) ⇒ analyzer จับได้เฉพาะกรณีที่ `!` ไม่จำเป็น · ที่เหลือพึ่ง `dart-review` prompt + คนรีวิว |
| 4 | `const` + extract widget | ✅/~ | ✅ 6 lint (`prefer_const_constructors`, `prefer_const_constructors_in_immutables`, `prefer_const_declarations`, `prefer_const_literals_to_create_immutables`, `avoid_unnecessary_containers`, `sized_box_for_whitespace`) · ~ ความลึกของ widget tree / `_buildX()` แยกออกมาเป็น class ⇒ review |
| 5 | Logic ≠ UI | ~ | `no_logic_in_create_state`, `use_build_context_synchronously` · การเรียก API ใน `build()` ⇒ review + prompt |
| 6 | Naming conventions | ✅ | `constant_identifier_names`, `camel_case_types`, `non_constant_identifier_names`, `file_names`, `library_names`, `package_names`, `library_private_types_in_public_api`, `no_leading_underscores_for_local_identifiers` |
| 7 | Dispose / memory | ✅ | `cancel_subscriptions`, `close_sinks`, `avoid_slow_async_io` · ลำดับการ dispose + `mounted` guard ⇒ review |
| 8 | SOLID & DRY | ✖ | ไม่มี lint ที่จับได้เลย · `dart-review` checklist ข้อ 8 + rule of three |
| 9 | Testable design (constructor DI) | ~ | ✖ lint · ~ CI บังคับเรื่อง **ผลลัพธ์** (coverage floor + test ต้องรันผ่าน) ไม่ได้บังคับวิธีออกแบบ |
| 10 | Documentation `///` | ~ | `slash_for_doc_comments`, `comment_references`, `flutter_style_todos` · "อธิบาย why ไม่ใช่ what" ⇒ review · ถ้าเปิด `public_member_api_docs: true` จะบังคับว่ามี doc จริง (แต่ไม่ได้บอกว่าดี) |
| 11 | State mgmt เดียว | ✖ | ไม่มี lint · บังคับได้ด้วยกฎ review: 1 ไฟล์ state ต่อ feature + pinned ในชั้น 2 |
| 12 | No hardcoded strings | ✖ | ไม่มี lint มาตรฐาน · ถ้าอยากได้จริงต้องใช้ `custom_lint` / `dart_code_metrics`-style rule (เพิ่ม dependency) หรือ grep `Text('...')` ใน CI (เปราะ) ⇒ ตอนนี้พึ่ง `dart-review` + l10n layer |
| 13 | Responsive / adaptive | ✖ | ไม่มี lint · บังคับด้วย widget test 2 ขนาด (320 dp + ~800 dp) + `textScaler` 2.0 ⇒ ถ้าเพิ่ม test นี้ = ยกระดับเป็น ~ ได้ |
| 14 | Secrets / env vars | ✅ | secret scan ใน CI (gitleaks) + `.githooks/pre-commit` (มี grep fallback เสมอแม้ไม่มี gitleaks) + `.gitignore` |
| 15 | Trailing commas | ✅ | `require_trailing_commas` + `dart format --set-exit-if-changed` ใน CI · `formatter.trailing_commas: preserve` |
| + | Formatting รวม | ✅ | `dart format` (CI: `--output=none --set-exit-if-changed`), `prefer_single_quotes`, `directives_ordering` |
| + | Analyzer สะอาด | ✅ | `flutter analyze --fatal-infos` — infos นับเป็น failure |
| + | Build ไม่พัง | ✅ | `flutter build apk --debug` ใน CI (จับ plugin/platform config แตก) |

## สรุปช่องโหว่ที่เหลือ (ข้อที่บอกลูกค้า/หัวหน้าได้ตรง ๆ)

- **ข้อ 1, 8, 11, 12, 13 บังคับด้วยเครื่องไม่ได้ในชุดนี้** — ถ้าต้องการบังคับจริง ต้องเพิ่ม
  custom lint pipeline (มีค่าใช้จ่าย: dependency + maintenance) หรือยอมรับว่ามันคือ convention
- **ข้อ 3b (`!`) ไม่มี lint มาตรฐาน** — กฎนี้จะถอยกลับได้ถ้าไม่มีคนรีวิว PR
- **ข้อ 9 บังคับได้แค่ผลลัพธ์** (coverage/test ผ่าน) ไม่ได้บังคับวิธีออกแบบ
- ทุกอย่างในตารางนี้เป็น **client-side** เท่านั้น — authorization/validation ที่ปลอดภัยจริงต้องอยู่ฝั่ง server

## วิธีตรวจว่าตารางนี้ยังจริง

```bash
# 1) analyzer + lint ทั้งหมดต้องสะอาด (ถ้า lint ชื่อผิด analyzer จะฟ้อง unknown rule)
flutter analyze --fatal-infos

# 2) ตรวจว่า lint rule ที่เขียนไว้มีอยู่จริงในเวอร์ชัน Dart ที่ใช้
curl -s https://raw.githubusercontent.com/dart-lang/linter/main/example/all.yaml \
  | grep -E '^\s*-\s*[a-z_]+$' | tr -d ' -' | sort > /tmp/all_rules.txt
grep -oE '^\s{4}[a-z_]+:' analysis_options.yaml | tr -d ' :' | sort -u > /tmp/used_rules.txt
comm -23 /tmp/used_rules.txt /tmp/all_rules.txt    # ต้องว่าง — ถ้าไม่ว่าง = ชื่อ lint ผิด/เก่าเกินไป

# 3) gate ในเครื่องทำงาน
git config core.hooksPath .githooks && chmod +x .githooks/pre-commit
```
