# ระบบลงทะเบียนเรียนรายวิชา

เว็บเดโมสำหรับสมัครสมาชิก เข้าสู่ระบบ ดูรายวิชา ลงทะเบียนเรียน และรับข้อความยืนยัน โดยแยกระบบเป็น 5 Microservices แต่ละบริการมี source code, tests และ Docker image ของตัวเอง ส่วน repository นี้รวมบริการ เอกสาร Docker Compose และ GitHub Actions เพื่อทดสอบและเผยแพร่ระบบ

## ภาพรวมสถาปัตยกรรม

```text
Browser
  └── API Gateway / Web
      ├── User / Auth Service
      ├── Course Service
      └── Enrollment Service
            └── Notification Service
```

ผู้ใช้เข้าเว็บผ่าน Gateway ที่ `http://localhost:8080` ส่วนบริการภายในติดต่อกันด้วยชื่อ service บนเครือข่าย Docker Compose

## บริการและหน้าที่

| บริการ | บทบาทในกลุ่ม | หน้าที่หลัก |
|---|---|---|
| API Gateway / Web | PM | แสดงหน้าเข้าสู่ระบบและหน้ารายวิชา รวมถึงส่งต่อคำขอไปยังบริการอื่น |
| User / Auth Service | Tester | สมัครบัญชีและเข้าสู่ระบบในฐานะนักศึกษา |
| Course Service | Developer | แสดงรายการและรายละเอียดรายวิชา พร้อม Endpoint สำหรับเพิ่มรายวิชา ปัจจุบันยังไม่มี Endpoint แก้ไขหรือลบ |
| Enrollment Service | Tester | ลงทะเบียนเรียนและแสดงรายการลงทะเบียนของนักศึกษา |
| Notification Service | Developer | สร้างข้อความยืนยันการลงทะเบียนเพื่อแสดงบนเว็บ โดยไม่เชื่อมต่อผู้ให้บริการอีเมลหรือ SMS |

## เทคโนโลยีและเครื่องมือ

- Node.js, TypeScript และ Express
- Docker และ Docker Compose
- GitHub สำหรับเก็บ source code และทำงานร่วมกัน
- GitHub Actions สำหรับ Build, Test และเผยแพร่ Docker image
- Docker Hub สำหรับเก็บ image แยกตามบริการ

## เริ่มระบบในเครื่อง

สิ่งที่ต้องติดตั้งก่อน: Git, Node.js 24 และ Docker Desktop ที่เปิดใช้งานอยู่

อัปเดตโค้ดจาก `main` แล้วเปิดระบบจากโฟลเดอร์หลักของ repository:

```powershell
git switch main
git pull origin main
docker compose up --build -d
docker compose ps
```

รอให้บริการขึ้นสถานะ `Healthy` แล้วเปิดเว็บ:

```powershell
Start-Process "http://localhost:8080"
```

ตรวจสุขภาพ Gateway ได้ด้วย:

```powershell
Invoke-RestMethod http://localhost:8080/health
```

ปิดระบบเมื่อใช้งานเสร็จ:

```powershell
docker compose down
```

## ขั้นตอนสาธิต

1. สมัครบัญชี หรือเข้าสู่ระบบด้วยบัญชีที่มีอยู่
2. กด **โหลดรายวิชา** เพื่อดูรายการวิชา
3. กรอกรหัสนักศึกษาและรหัสวิชา แล้วกด **ลงทะเบียนเรียน**
4. ตรวจข้อความยืนยันจาก Notification ในป๊อปอัปและส่วน **ผลการทำงาน**

ข้อมูลของบริการเป็นข้อมูลเดโมที่เก็บในหน่วยความจำ เมื่อ container เริ่มใหม่ ข้อมูลที่สมัครหรือลงทะเบียนไว้อาจหายไป

## การทดสอบ

แต่ละบริการเก็บ tests ไว้ใน `services/<service>/tests/` โดยมี Unit Test สำหรับทดสอบส่วนย่อย และ Integration Test สำหรับตรวจ API หรือการเชื่อมต่อกับบริการอื่น ให้เข้าโฟลเดอร์บริการที่ต้องการทดสอบแล้วรัน:

```powershell
npm ci
npm test
```

ตัวอย่างทดสอบ Gateway/Web:

```powershell
Set-Location services/gateway-web
npm ci
npm test
```

Test Specification ของกลุ่มอยู่ที่ [docs/test-specifications.md](docs/test-specifications.md) ครอบคลุมเกณฑ์ทดสอบทั้ง 5 บริการและ flow รวม ส่วนเอกสาร API อยู่ที่ [docs/api-contract.md](docs/api-contract.md)

## GitHub Actions และ Docker Hub

แต่ละบริการมี Workflow CI ของตัวเองใน `.github/workflows/ci-*.yml` เมื่อเปิด Pull Request หรือ push การเปลี่ยนแปลงเข้า `main` Workflow จะติดตั้ง dependencies, Build, Test และตรวจว่า Docker image สร้างได้

หลัง merge การเปลี่ยนแปลงของบริการเข้า `main` Workflow ของบริการนั้นจะเผยแพร่ image ไป Docker Hub โดยใช้ tag `latest` และ commit SHA เมื่อ push tag รุ่นที่ขึ้นต้นด้วย `v` เช่น `v1.0.0` Workflow `release-images.yml` จะ Build, Test และเผยแพร่ image ของทั้ง 5 บริการด้วย tag รุ่นและ `latest`

ชื่อ Docker Hub repository ของแต่ละ image:

- `registration-gateway`
- `registration-auth`
- `registration-course`
- `registration-enrollment`
- `registration-notification`

ผู้ดูแล repository ต้องตั้งค่า GitHub Actions Secrets ที่ **Settings → Secrets and variables → Actions**:

- `DOCKERHUB_USERNAME` คือชื่อผู้ใช้หรือ namespace บน Docker Hub
- `DOCKERHUB_TOKEN` คือ Docker Hub access token ที่มีสิทธิ์ push image

ห้ามใส่ token หรือข้อมูลลับใน source code, ไฟล์ workflow หรือ commit ลง Git ตรวจผลการทำงานจริงในแท็บ **Actions** และตรวจ tag ของ image ใน Docker Hub

## เปิดระบบจาก Docker Hub

สร้างไฟล์ `.env` จาก `.env.example` แล้วกำหนด namespace และ tag ของ image ที่ต้องการใช้ จากนั้นรันที่โฟลเดอร์หลัก:

```powershell
Copy-Item .env.example .env
# แก้ DOCKERHUB_USERNAME และ IMAGE_TAG ภายในไฟล์ .env
docker compose --env-file .env -f deploy/docker-compose.release.yml pull
docker compose --env-file .env -f deploy/docker-compose.release.yml up -d
docker compose --env-file .env -f deploy/docker-compose.release.yml ps
```

เปิดเว็บที่ `http://localhost:8080` และหยุดระบบด้วย:

```powershell
docker compose --env-file .env -f deploy/docker-compose.release.yml down
```

ห้าม commit ไฟล์ `.env` ดูรายละเอียดเพิ่มเติมที่ [docs/deployment.md](docs/deployment.md)

## โครงสร้างสำคัญ

```text
.github/workflows/          CI ของแต่ละบริการและ Release workflow
deploy/                     Docker Compose สำหรับ image ที่เผยแพร่แล้ว
docs/                       API Contract, Test Specification และ Deploy guide
services/gateway-web/       เว็บและ API Gateway
services/user-auth/         สมัครบัญชีและเข้าสู่ระบบ
services/course/            ข้อมูลรายวิชา
services/enrollment/        ลงทะเบียนเรียน
services/notification/      ข้อความยืนยัน
```

ไฟล์งานรายบุคคลอยู่ในโฟลเดอร์ของแต่ละบริการ เช่น `services/gateway-web/windsurf_work.md` และ `services/notification/safe_work.md`
