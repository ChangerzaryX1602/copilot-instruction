# copilot-instruction — Flutter/Dart AI coding kit

ชุดไฟล์ที่ทำให้ **GitHub Copilot ใน VS Code** เขียนโค้ด Flutter/Dart ตามมาตรฐานของเรา
โดยไม่ต้องเตือนซ้ำทุกครั้ง — คัดลอกเข้า repo แล้วจบ

> **หลักคิด:** บอก AI ในแชท = ได้ผล 1 เทิร์นแล้วหายไป
> ทำให้สิ่งที่ถูกต้องเป็น **ค่าเริ่มต้น** (กฎอยู่ที่ที่ AI เลี่ยงอ่านไม่ได้)
> และเป็น **สิ่งที่ถูกบังคับ** (CI ที่ merge ไม่ผ่านถ้าฝ่าฝืน)

ชุดนี้ **ไม่มี application code** — เป็นกฎ, prompt, skill, lint config และ CI gate เท่านั้น

---

## 5 ชั้น — อะไรอยู่ชั้นไหน และทำไม

| # | ชั้น | ไฟล์ | Copilot อ่านเมื่อไหร่ | ราคา context |
|---|------|------|------------------------|--------------|
| 1 | คุยในแชท | — | เทิร์นนั้นเทิร์นเดียว | ฟรี แต่หาย |
| 2 | รัฐธรรมนูญของ repo | `.github/copilot-instructions.md` | **ทุก request อัตโนมัติ** | always-on → ต้องไม่เกิน 1 หน้า |
| 3 | กฎตามชนิดไฟล์ | `.github/instructions/*.instructions.md` | ทุก request ที่แตะไฟล์ตรง `applyTo` | จ่ายเฉพาะไฟล์ที่ตรง |
| 4 | Agent skill | `.github/skills/*/SKILL.md` | เมื่อ prompt ตรงกับ `description` | ~ศูนย์จนกว่าจะใช้ |
| 5 | CI + git hook | `.github/workflows/quality.yml`, `.githooks/pre-commit` | ตอน commit / push | ไม่กิน context เลย |

**กฎการวาง:** สั้น+ใช้ทุกที่ → ชั้น 2 · เฉพาะภาษา/เฟรมเวิร์ก → ชั้น 3 · ยาวหรือใช้นาน ๆ ครั้ง → ชั้น 4 · ห้ามถอยหลัง → ชั้น 5 ด้วย

> หมายเหตุ VS Code 2026: settings-based codegen instructions **deprecated ตั้งแต่ VS Code 1.102**
> ⇒ ใช้ไฟล์เท่านั้น · และต้องเปิด `chat.includeApplyingInstructions` / `chat.useAgentsMdFile`
> (มีให้แล้วใน `.vscode/settings.json` ของชุดนี้)

---

## 15 กฎของคุณ → วางชั้นไหน → บังคับด้วยอะไร

| # | กฎ | อยู่ที่ชั้น | ชั้น 5 บังคับได้ไหม |
|---|----|-----------|---------------------|
| 1 | File Structure (feature-first) | 3 `architecture` | ✖ convention + review checklist |
| 2 | Minimal 3rd-party deps | 3 `dependencies` | ~ `sort_pub_dependencies` + CI audit |
| 3 | Null safety & error handling | 3 `dart` | ✅ strict mode + 8 lint |
| 4 | `const` + extract widget | 3 `flutter-ui` | ✅ 6 lint (`const`, containers) |
| 5 | Logic ≠ UI | 3 `flutter-ui` | ~ lint บางส่วน + review |
| 6 | Dart naming conventions | 3 `dart` | ✅ 4 lint |
| 7 | Dispose / memory | 3 `dart` | ✅ `cancel_subscriptions`, `close_sinks` |
| 8 | SOLID & DRY (rule of three) | 4 skill | ✖ review |
| 9 | Testable design (constructor DI) | 3 `tests` | ~ review + coverage gate |
| 10 | Dart docstrings `///` | 3 `dart` | ~ `slash_for_doc_comments`, `comment_references` |
| 11 | State mgmt เดียว | 2 constitution | ✖ review (pinned ไว้ชั้น 2) |
| 12 | No hardcoded strings | 3 `flutter-ui` | ✖ review + prompt |
| 13 | Responsive / adaptive UI | 3 `flutter-ui` | ✖ review + widget test หลายขนาด |
| 14 | Secrets / env vars | 2 + 3 `security` | ✅ secret scan ใน CI + hook |
| 15 | Trailing commas | 3 `dart` | ✅ `require_trailing_commas` |

รายละเอียดระดับ "กฎนี้ map กับ lint ตัวไหน / อะไรยังไม่มี lint" อยู่ที่ 👉 [`docs/rule-to-lint.md`](docs/rule-to-lint.md)

**คำเตือนที่ต้องรู้:** กฎบางข้อบังคับด้วย lint ไม่ได้เลย (ข้อ 1, 8, 11, 12, 13)
⇒ มันถูกเขียนเป็น **convention** ไม่ใช่ gate อย่าเข้าใจว่ามี CI กันไว้แล้ว

---

## สิ่งที่ 15 กฎเดิมยังไม่ครอบ แต่ชุดนี้เพิ่มให้

| หัวข้อ | อยู่ที่ |
|--------|---------|
| Pinned version ของ Flutter/Dart (training data ของ AI เก่ากว่า release) | ชั้น 2 |
| Error/observability + ห้าม log PII, token, เบอร์โทร | ชั้น 3 `security` |
| Navigation / deep link / route guard | ชั้น 3 `architecture` |
| Offline & cache policy | ชั้น 3 `architecture` |
| Accessibility, text scale, contrast, touch target 48dp | ชั้น 3 `flutter-ui` |
| Permission (mic/camera/location) — ขอตอนใช้ ไม่ขอตอนเปิดแอป | ชั้น 3 `security` |
| Testing pyramid + coverage ขั้นต่ำ + test 401/403 ไม่ใช่แค่ success | ชั้น 3 `tests` |
| เกณฑ์อนุมัติ dependency ใหม่ (pub points, publish ล่าสุด, license, transitive) | ชั้น 3 `dependencies` |
| Git/PR workflow — Conventional Commits, PR checklist | ชั้น 2 |
| Plan ก่อนเขียนโค้ด | ชั้น 4 prompt `plan-first` |
| Release build — flavors, obfuscation, ตัด debug logging | ชั้น 3 `security` |

---

## ค่าที่ตั้งไว้ให้แล้วใน `copilot-instructions.md` §1

เติมให้เรียบร้อยแล้วด้วย stable ล่าสุด ณ 18 ก.ย. 2026 (ตรวจจาก `releases_linux.json` ทางการ +
`pub.dev/api`) — **4 บรรทัดนี้คือ "การตัดสินใจของทีม" ไม่ใช่ความจริงสากล** ถ้าโปรเจกต์มีอยู่แล้ว
ให้แก้ 4 บรรทัดนี้ให้ตรงกับของจริง:

| หัวข้อ | ค่าที่ตั้งให้ | เปลี่ยนได้เป็น |
|--------|---------------|----------------|
| Flutter / Dart | 3.47.5 / 3.13.4 | version ที่เครื่องทีมใช้ (`flutter --version`) |
| State management | flutter_riverpod `^3.4.3` | provider / bloc — **ตัวเดียวทั้งโปรเจกต์** |
| Model codegen | freezed `^4.0.2` + json_serializable `^6.14.1` + build_runner `^2.16.1` | built_value / เขียน model มือ |
| Routing | go_router `^18.0.1` | auto_route / Navigator 2.0 |
| HTTP | dio `^5.11.1` | `http` |
| Storage | flutter_secure_storage `^11.2.0` + shared_preferences `^2.5.5` | hive / sqflite |
| Lints / Tests | flutter_lints `^6.0.0` · flutter_test + mocktail `^1.0.5` | very_good_analysis |
| Minimum OS | Android 8.0 (API 26) · iOS 14.0 | ตามที่ธุรกิจต้องการ |

ทำไมต้อง pin: training data ของ AI เก่ากว่า release ⇒ ถ้าไม่บอกเวอร์ชัน มันจะเขียน idiom ของ
major เก่า (เช่น syntax freezed 2 ที่ไม่คอมไพล์บน freezed 4) — บรรทัดพวกนี้คือ lever แรงสุดในไฟล์

---

## ติดตั้ง (บนเครื่องที่ใช้ทำงาน)

```bash
# 1) คัดลอกเข้า repo ของคุณ (จาก root ของ repo นั้น)
cp -r /path/to/copilot-instruction/.github      ./
cp -r /path/to/copilot-instruction/.githooks    ./
cp -r /path/to/copilot-instruction/.vscode      ./
cp    /path/to/copilot-instruction/analysis_options.yaml  ./        # ถ้าแอปอยู่ root
cp    /path/to/copilot-instruction/l10n.yaml              ./        # ถ้าใช้ gen-l10n

# 2) เปิด gate ในเครื่อง (ทำครั้งเดียวต่อ clone)
git config core.hooksPath .githooks
chmod +x .githooks/pre-commit

# 3) เช็คว่า pin ใน §1 ตรงกับเครื่องจริง (ค่า default เติมไว้แล้ว ไม่มี marker ค้าง)
flutter --version
grep -n "Flutter \*\*" .github/copilot-instructions.md
```

## ตรวจว่า Copilot อ่านจริง (ห้ามข้าม)

1. Copilot Chat → **gear / Configure Chat** → ต้องเห็นไฟล์ instruction ที่มันเจอ
2. พิมพ์ `/` → ต้องขึ้น `plan-first`, `new-feature`, `fix-analyzer`, `dart-review`, `security-review`
3. ถาม Copilot: *"repo นี้ pin เวอร์ชันอะไรไว้ และอ่านจากไฟล์ไหน"* → ถ้าตอบถูกและอ้าง
   `.github/copilot-instructions.md` = ชั้น 2 ทำงาน
4. ทำให้พังแล้วดูว่า gate จับได้: ใส่ `print('x')` แล้ว commit → pre-commit ต้องเตือน

---

## Growth loop (ส่วนที่ทำให้มันอยู่ได้นาน)

**อย่าเขียนกฎให้ครบตั้งแต่แรก** — เริ่มจากหน้าเดียว แล้วทุกครั้งที่ AI ทำผิด:
แก้โค้ด + เพิ่ม **1 บรรทัด** ลงชั้นที่ถูก

| ผิดแบบ | ไปเพิ่มที่ |
|--------|-----------|
| กฎทั่วไปที่ใช้ทุกไฟล์ | ชั้น 2 `copilot-instructions.md` |
| เฉพาะ Dart / เฉพาะ UI | ชั้น 3 ไฟล์ที่ตรง `applyTo` |
| เป็นขั้นตอนทำงานซ้ำ ๆ | ชั้น 4 prompt |
| ความรู้ยาว/ใช้นาน ๆ ครั้ง | ชั้น 4 skill |
| เรื่องที่ห้ามพลาดเด็ดขาด | ชั้น 5 CI |

แล้วตัดชั้น 2 ให้สั้นลงอีก — ไฟล์ always-on ยิ่งสั้น แต่ละบรรทัดยิ่งมีน้ำหนัก

---

## ชุดนี้ **ไม่** ครอบอะไร (พูดตรง ๆ)

- ไม่ได้ทำให้คุณเก่ง Flutter/Dart แทนได้ — มันยกพื้นขั้นต่ำขึ้นเท่านั้น
- ข้อ 1, 8, 11, 12, 13 ไม่มี lint บังคับ ⇒ ยังต้องมีคนรีวิว
- Security ในนี้เป็นด่านแรก ไม่ใช่ audit
- NestJS/Go ไม่ได้อยู่ในชุดนี้ (คนละชุด)

---

## โครงสร้างไฟล์

```
.github/
├── copilot-instructions.md          # ชั้น 2 — 1 หน้า ห้ามยาวกว่านี้
├── instructions/                    # ชั้น 3 — กฎตามชนิดไฟล์ (applyTo)
│   ├── architecture.instructions.md
│   ├── dart.instructions.md
│   ├── flutter-ui.instructions.md
│   ├── dependencies.instructions.md
│   ├── security.instructions.md
│   └── tests.instructions.md
├── prompts/                         # ชั้น 4 — slash command
│   ├── plan-first.prompt.md
│   ├── new-feature.prompt.md
│   ├── fix-analyzer.prompt.md
│   ├── dart-review.prompt.md
│   └── security-review.prompt.md
├── skills/                          # ชั้น 4 — ความรู้ยาว โหลดเมื่อตรงเท่านั้น
│   ├── flutter-dart-house-style/
│   │   ├── SKILL.md
│   │   └── reference/the-15-rules.md
│   └── flutter-testing/
│       ├── SKILL.md
│       └── reference/patterns.md
├── agents/
│   └── flutter-reviewer.agent.md    # reviewer แบบอ่านอย่างเดียว
└── workflows/quality.yml            # ชั้น 5 — CI gate
.githooks/pre-commit                 # ชั้น 5 — gate ในเครื่อง (fallback เมื่อไม่มี gitleaks)
.vscode/settings.json                # เปิด instruction file + format on save
docs/rule-to-lint.md                 # ตาราง กฎ → lint/CI/review (ตัวตัดสินว่าใครบังคับอะไร)
analysis_options.yaml                # หัวใจของข้อ 3,4,6,7,10,15
l10n.yaml                            # ตัวอย่าง gen-l10n (ข้อ 12)
```
