# คู่มือปฏิบัติการวันแข่ง: การแก้ไขโปรเจกต์ `PES` (ต้นฉบับกรรมการ) แบ่งเป็น 2 เฟส (เฟส 1 เอาขึ้น Deploy ก่อน $\rightarrow$ เฟส 2 เก็บรายละเอียดแล้ว Deploy ซ้ำ)

> **กลยุทธ์ 2 เฟสเพื่อการันตีคะแนนเต็มในห้องแข่ง:**
> - **🚀 เฟสที่ 1 (จุดที่ 1–6):** แก้เฉพาะไฟล์แกนหลักที่จำเป็นต่อการ Build, Test, Push, Deploy และ Login แล้วสั่ง **`git push` รอบที่ 1 ทันที** เพื่อให้ Pipeline 4 Stages เขียวครบ และระบบออนไลน์บน `VM2 (192.168.10.32)` ก่อน (การันตีคะแนนก้อนใหญ่ไว้ในมือทันที!)
> - **🎯 เฟสที่ 2 (จุดที่ 7–11):** พอเว็บออนไลน์แล้ว ค่อยตามเก็บรายละเอียดบั๊กความถูกต้องด้านใน (เมนูแอดมิน, หน้าสร้างผู้ใช้ Create User, ปุ่ม Open API Docs, ไอคอนลูกตา) แล้วสั่ง **`git push` รอบที่ 2** เพื่อให้เกิด Auto-Trigger Pipeline อัปเดตระบบจนสมบูรณ์ 100%!

---

## แผนผัง IP และพอร์ตของระบบที่ใช้ในคู่มือนี้

| เครื่อง / บริการ | IP Address | พอร์ต / โดเมน | หน้าที่ในโปรเจกต์ PES |
| :--- | :--- | :--- | :--- |
| **Gateway (VLAN 10)** | `192.168.10.1` | `/24` (`255.255.255.0`) | ขา SVI VLAN 10 บนสวิตช์ TP-Link |
| **Proxmox Host** | `192.168.10.30` | `https://192.168.10.30:8006` | เครื่องแม่ข่ายรัน VM1 และ VM2 |
| **VM1 (GitLab/DNS/Registry)** | `192.168.10.31` | • GitLab: `:8080` (`gitlab.mydomain.lab`)<br>• Registry: `:5000`<br>• DNS: `:53`, `:5380` | เก็บโค้ด Git, รัน GitLab Runner (Shell), เก็บ Docker Image |
| **VM2 (Production / NPM)** | `192.168.10.32` | • Frontend (Nuxt): **`:3000`** (`app.mydomain.lab`)<br>• Backend API: **`:7000`** (`api.mydomain.lab`)<br>• phpMyAdmin: **`:8081`** | รันระบบจริงจาก CI/CD (`db` + `phpmyadmin` + `api` + `frontend`) และเป็น Reverse Proxy (NPM) |
| **PC นักเรียน 1–3 (VLAN 20)** | `192.168.20.36 – .38`<br>*(หรือรับ DHCP `.100+`)* | Gateway: `192.168.20.1`<br>DNS: `192.168.10.31` | เครื่องสำหรับเปิด VS Code แก้ไขไฟล์ `PES` แล้วสั่ง `git push` |

---

## STEP 7.0: เตรียมโฟลเดอร์ `pes-app` และล้างเส้นแดง Error ใน VS Code ก่อนเริ่มเฟส 1

1. **ดาวน์โหลดเทมเพลตต้นฉบับของกรรมการ:** [https://vecskill.bncc.ac.th/files/PES.zip](https://vecskill.bncc.ac.th/files/PES.zip)
2. แตกไฟล์ `PES.zip` แล้วคัดลอกโฟลเดอร์ `PES` ออกมาตั้งชื่อใหม่ว่า **`pes-app`**
3. **ลบโฟลเดอร์ขยะ 3 โฟลเดอร์นี้ทิ้งทันที** (ถ้ามีติดมา):
   - `backend/node_modules`
   - `frontend/node_modules`
   - `frontend/.nuxt`
4. เปิดโฟลเดอร์ `pes-app` ในโปรแกรม **VS Code** แล้วเปิด Terminal รันคำสั่งติดตั้งแพ็กเกจและสร้าง Type ของ Nuxt **เพื่อล้างเส้นหยักสีแดง (Error) ใน VS Code ให้หายทั้งหมด** *(ไม่ต้องรัน `npm run dev` เพราะเราจะรันจริงบน Docker ของ VM2)*:
   - **คนทำ Backend:**
     ```bash
     cd backend
     npm install
     cd ..
     ```
   - **คนทำ Frontend:** *(สำคัญมาก: `npx nuxt prepare` จะสร้าง `.nuxt/tsconfig.json` ทำให้เส้นแดง `defineNuxtConfig`, `useRuntimeConfig`, `process.env` หายไปทันที)*
     ```bash
     cd frontend
     npm install
     npx nuxt prepare
     cd ..
     ```
   > **💡 ปลอดภัย 100% ไม่กระทบ Docker:** เพราะในจุดที่ 5 เราสร้างไฟล์ `frontend/.dockerignore` ดักไม่ให้ส่ง `node_modules` และ `.nuxt` จากเครื่อง Windows ขึ้นไปกวนตอน Docker Build บน VM1 แล้ว!

---

# 🚀 เฟสที่ 1 (จุดที่ 1 ถึง 6): แก้ส่วนสำคัญให้ Deploy ขึ้น VM2 และ Login ผ่านก่อน!

### 🛠️ [เฟส 1] จุดที่ 1: แก้ไฟล์ `.gitlab-ci.yml` ที่รากโปรเจกต์ (🔄 ลบของเก่าแล้ววางชุดสั้นจำง่าย 25 บรรทัดนี้แทน)
เปิดไฟล์ `.gitlab-ci.yml` กด `Ctrl+A` ลบของเดิมออกทั้งหมด แล้ววางโค้ด 4 Stages (`build`, `test`, `push`, `deploy`) ชุดสั้นจำง่ายนี้แทน:

```yaml
stages: [build, test, push, deploy]

build_job:
  stage: build
  script:
    - docker build -t 192.168.10.31:5000/pes-backend:latest ./backend
    - docker build -t 192.168.10.31:5000/pes-frontend:latest ./frontend

test_job:
  stage: test
  script:
    - docker run --rm 192.168.10.31:5000/pes-backend:latest npx vitest run test/server.spec.js test/middlewares_auth.spec.js

push_job:
  stage: push
  script:
    - docker push 192.168.10.31:5000/pes-backend:latest
    - docker push 192.168.10.31:5000/pes-frontend:latest

deploy_job:
  stage: deploy
  script:
    - ssh -p 2222 user1@192.168.10.32 "mkdir -p ~/pes/backend"
    - scp -P 2222 docker-compose.yml schema.sql user1@192.168.10.32:~/pes/
    - scp -P 2222 backend/.env user1@192.168.10.32:~/pes/backend/
    - ssh -p 2222 user1@192.168.10.32 "cd ~/pes && sudo docker compose pull && sudo docker compose up -d"
```

---

### 🛠️ [เฟส 1] จุดที่ 2: แก้ไฟล์ `docker-compose.yml` ที่รากโปรเจกต์ (✏️ แก้ทับจากไฟล์เดิมแค่ 4 บรรทัด!)
ไม่ต้องพิมพ์ใหม่ทั้งไฟล์! เปิดไฟล์ `docker-compose.yml` ต้นฉบับของกรรมการ แล้วแก้ทับแค่ **4 บรรทัดนี้**:
1. ใน `phpmyadmin:` แก้พอร์ตจาก `ports: ["8080:80"]` $\rightarrow$ เป็น **`ports: ["8081:80"]`**
2. ใน `api:` แก้จาก `build: ./backend` $\rightarrow$ เป็น **`image: 192.168.10.31:5000/pes-backend:latest`**
3. ใน `frontend:` แก้จาก `build: ./frontend` $\rightarrow$ เป็น **`image: 192.168.10.31:5000/pes-frontend:latest`**
4. ใน `frontend:` แก้ `NUXT_PUBLIC_API_BASE: http://localhost:7000` $\rightarrow$ เป็น **`NUXT_PUBLIC_API_BASE: http://api.mydomain.lab`**

```yaml
services:
  db:
    image: mysql:8.0
    command: >
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: skills_db
      TZ: Asia/Bangkok
    volumes:
      - db_data:/var/lib/mysql
      - ./schema.sql:/docker-entrypoint-initdb.d/00-schema.sql:ro
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-uroot", "-prootpassword"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks: [dev_net]

  phpmyadmin:
    image: phpmyadmin:latest
    environment:
      PMA_HOST: db
      PMA_USER: root
      PMA_PASSWORD: rootpassword
    ports: ["8081:80"]                                    # <-- [✏️ จุดแก้ที่ 1] เดิมเป็น "8080:80"
    depends_on: [db]
    networks: [dev_net]

  api:
    image: 192.168.10.31:5000/pes-backend:latest          # <-- [✏️ จุดแก้ที่ 2] เดิมเป็น build: ./backend
    env_file:
      - ./backend/.env
    ports: ["7000:7000"]
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./backend/uploads:/app/uploads
    networks: [dev_net]

  frontend:
    image: 192.168.10.31:5000/pes-frontend:latest         # <-- [✏️ จุดแก้ที่ 3] เดิมเป็น build: ./frontend
    environment:
      NUXT_PUBLIC_API_BASE: http://api.mydomain.lab       # <-- [✏️ จุดแก้ที่ 4] เดิมเป็น http://localhost:7000
      NITRO_PORT: 3000
      HOST: 0.0.0.0
    ports: ["3000:3000"]
    depends_on: [api]
    networks: [dev_net]

volumes:
  db_data:

networks:
  dev_net:
    driver: bridge
```

---

### 🛠️ [เฟส 1] จุดที่ 3: แก้ไฟล์ `schema.sql` (➕ พิมพ์แทรกเพิ่มใหม่ 1 บรรทัด)
* ต้องแก้ก่อน Deploy รอบแรก เพราะ MySQL อ่านไฟล์นี้เฉพาะตอนสร้างฐานข้อมูลครั้งแรก
* เปิด `schema.sql` กด `Ctrl+F` หาคำว่า `storage_path VARCHAR(1024)` ในตาราง `attachments` (บรรทัด ~149) แล้วพิมพ์แทรกบรรทัด **`note TEXT NULL,`** ต่อท้าย:

```sql
  file_name VARCHAR(255) NOT NULL,
  mime_type VARCHAR(100) NOT NULL,
  size_bytes INT UNSIGNED NOT NULL,
  storage_path VARCHAR(1024) NOT NULL,
  note TEXT NULL,                                         -- <-- [➕ พิมพ์แทรกเพิ่มใหม่ 1 บรรทัด]
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
```

---

### 🛠️ [เฟส 1] จุดที่ 4: แก้ไฟล์ `backend/.env` (✏️ แก้ทับบรรทัด `CORS_ORIGIN`)
เปิดไฟล์ `backend/.env` แล้วแก้เฉพาะบรรทัด `CORS_ORIGIN` (คั่นด้วย `,` ห้ามมีช่องว่าง):

```env
PORT=7000
CORS_ORIGIN=http://localhost:3000,http://127.0.0.1:3000,http://192.168.10.32:3000,http://app.mydomain.lab,https://app.mydomain.lab

DB_HOST=db
DB_PORT=3306
DB_USER=root
DB_PASSWORD=rootpassword
DB_NAME=skills_db

JWT_SECRET=testing123
JWT_EXPIRES=2h
```

---

### 🛠️ [เฟส 1] จุดที่ 5: สร้างไฟล์ใหม่ `frontend/.dockerignore` และ `frontend/.env`
1. สร้างไฟล์ **`frontend/.dockerignore`**:
   ```text
   node_modules
   .nuxt
   .output
   dist
   ```
2. สร้างไฟล์ **`frontend/.env`**:
   ```env
   NUXT_PUBLIC_API_BASE=http://api.mydomain.lab
   ```

---

### 🛠️ [เฟส 1] จุดที่ 6: แก้ URL ล็อกอินใน `frontend/nuxt.config.ts` และ `frontend/plugins/axios.client.js`
1. เปิด **`frontend/nuxt.config.ts`** (บรรทัดที่ 52): **ลบ `process.env.NUXT_PUBLIC_API_BASE ||` ออก แล้วแก้ทั้งบรรทัดให้เหลือแค่นี้** (เพื่อตัดปัญหาเส้นแดง `process` ใน VS Code และ Nuxt 3 จะดึงค่า `NUXT_PUBLIC_API_BASE` จาก Docker มาทับให้อัตโนมัติอยู่แล้ว):
   ```typescript
   apiBase: 'http://api.mydomain.lab'
   ```
2. เปิด **`frontend/plugins/axios.client.js`** (บรรทัดที่ 10): แก้จาก `'http://localhost:7000'` $\rightarrow$ เป็น **`'http://api.mydomain.lab'`**:
   ```javascript
   baseURL: config.public.apiBase || 'http://api.mydomain.lab',
   ```

---

## 🚀 STEP 7.1 (จุดเช็คพอยต์ที่ 1): สั่ง Git Push เพื่อ Deploy รอบแรกขึ้น VM2 ทันที!

เมื่อทำครบ 6 จุดของเฟส 1 แล้ว **ให้สั่ง Deploy ขึ้นระบบจริงไว้ก่อนเลยทันที**:
1. **เช็คโปรเจกต์บนหน้าเว็บ GitLab (`http://gitlab.mydomain.lab`):**
   - **กรณีคนทำส่วนที่ 5 (GitLab Runner) สร้างโปรเจกต์ชื่อ `pes-app` และผูก Runner ไว้ให้แล้ว:** **ไม่ต้องสร้างโปรเจกต์ใหม่!** ข้ามไปรันคำสั่งข้อ 2 ได้เลยทันที
   - **กรณีคนทำส่วนที่ 5 เผลอสร้างโปรเจกต์ชื่อ `task-app` ไว้:** ให้สร้าง Blank Project ใหม่ชื่อ **`pes-app`** (เลือก **Public** และติ๊กออก *Initialize repository with a README*) แล้วเข้าไปในโปรเจกต์ `pes-app` $\rightarrow$ **Settings $\rightarrow$ CI/CD $\rightarrow$ Runners** $\rightarrow$ มองหาหัวข้อ *Other available runners* ด้านล่าง แล้วกดปุ่ม **Enable for this project** ที่ตัว Runner เดิมได้เลย (ไม่ต้องลงทะเบียนใหม่บน VM1!)
2. เปิด Terminal ใน VS Code ที่โฟลเดอร์ `pes-app` แล้วรันคำสั่งชุดนี้ *(💡 หมายเหตุ: `git config user.email` ของเครื่องที่เขียน `.gitlab-ci.yml` / สั่ง `git push` **ไม่จำเป็นต้องตรงกับ**คนตั้งค่า GitLab Runner ใช้ชื่อและอีเมลอะไรก็ได้)*:

```bash
git config --global user.name "root"
git config --global user.email "root@mydomain.lab"
git init
git switch -c main
git remote add origin http://gitlab.mydomain.lab/root/pes-app.git
git add -f backend/.env
git add .
git status --short
git commit -m "Phase 1: Initial deploy PES with 4-stage CI/CD"
git push -u origin main
```

### 🔑 วิธีแก้ปัญหาสั่ง `git push` แล้วติด Login ไม่ผ่าน / `Access denied` บน Windows (ใช้ PAT ชัวร์ 100%)

> **ทำไมหน้างานจริงถึงมักติด Login ตอน Push?**
> เพราะบนเครื่อง PC Windows ระบบ *Windows Credential Manager* มักจำรหัสเก่าค้างไว้ หรือหน้าต่างป๊อปอัปของ Git ไม่ยอมเด้ง/ไม่รับรหัสผ่านปกติ วิธีแก้ที่ผ่านฉลุย 100% คือใช้ **Personal Access Token (PAT)** ฝังลงใน URL ของ `origin` โดยตรง:

1. **สร้าง Token (PAT) บนหน้าเว็บ GitLab (`http://gitlab.mydomain.lab`):**
   - คลิกที่ **รูปโปรไฟล์มุมซ้ายบน** $\rightarrow$ เลือก **Edit profile** (หรือ **Preferences**) $\rightarrow$ เมนูซ้ายมือคลิก **Access Tokens** $\rightarrow$ กดปุ่ม **Add new token**
   - ช่อง **Token name:** พิมพ์ `push-token`
   - ช่อง **Expiration date:** กดกากบาท `x` ลบวันที่ออก (หรือเลือกวันถัดไป)
   - หัวข้อ **Select scopes:** ติ๊กถูกที่ **`[x] api`**, **`[x] read_repository`**, และ **`[x] write_repository`** $\rightarrow$ กด **Create personal access token**
   - คัดลอกรหัส Token ที่ได้ (ขึ้นต้นด้วย **`glpat-...`**)

2. **รันคำสั่งฝัง PAT ลงใน URL ของ `origin` แล้วสั่ง Push ใหม่ทันที:**
   ```bash
   git remote set-url origin http://root:glpat-วางโทเคนตรงนี้@gitlab.mydomain.lab/root/pes-app.git
   git push -u origin main
   ```

3. **💡 อธิบายโครงสร้างคำสั่งให้เข้าใจง่าย (จะได้ไม่พิมพ์ผิดในห้องแข่ง):**
   - รูปแบบคือ: `http://<ชื่อผู้ใช้>:<รหัสโทเคน_glpat>@<โดเมน_GitLab>/<เจ้าของโปรเจกต์>/<ชื่อโปรเจกต์>.git`
   - ตัวอย่างจริง:
     ```bash
     git remote set-url origin http://root:glpat-eTOEIhZcgV1Yb9IaduEC7W86MQp1OjEH.01.0w12ju3d6@gitlab.mydomain.lab/root/pes-app.git
     ```
   - **จุดสังเกต:** คำข้างหน้าเครื่องหมาย `:` (ก่อน `glpat-...`) จะใส่เป็น `root` หรือชื่ออะไรก็ได้ (เช่น `it`) เพราะ GitLab ตรวจสอบสิทธิ์จากตัวรหัส `glpat-...` ที่อยู่หลัง `:` และหน้า `@` เป็นหลัก และพอสั่ง `set-url` ครั้งเดียวแล้ว ตอนทำเฟส 2 สั่งแค่ `git push` เฉยๆ ก็ผ่านทันทีโดยไม่ต้องตั้งค่าซ้ำครับ!

> **✅ ผลลัพธ์ที่ได้ทันทีหลังจบเฟส 1:**
> - Pipeline เขียวครบ 4 Stages (`build` $\rightarrow$ `test` $\rightarrow$ `push` $\rightarrow$ `deploy`)
> - เปิด `http://api.mydomain.lab/health` ขึ้น `{"service":"okoak",...}`
> - เปิด `http://app.mydomain.lab/login` ล็อกอินด้วย `admin@ccollege.ac.th` / `12345678` เข้าหน้าตารางผู้ใช้สำเร็จ!

---

# 🎯 เฟสที่ 2 (จุดที่ 7 ถึง 11): เก็บรายละเอียดบั๊กความถูกต้องด้านใน แล้วสั่ง Deploy รอบที่ 2!

เมื่อระบบออนไลน์และ Pipeline รอบแรกเขียวแล้ว ค่อยตามแก้บั๊กย่อยอีก 5 จุดนี้:

### 🛠️ [เฟส 2] จุดที่ 7: แก้ไฟล์ `frontend/layouts/default.vue` (เพื่อให้แอดมินเห็นเมนูแอดมิน)
เปิด `frontend/layouts/default.vue` แก้ส่วน `<script setup>` ด้านบนสุด (บรรทัด 1–8):

```javascript
<script setup>
import { ref, computed } from 'vue'
import { useMenu } from '~/composables/useMenu'
import { useAuthStore } from '~/stores/auth'              // <-- [➕ เพิ่มใหม่]

const auth = useAuthStore()                               // <-- [➕ เพิ่มใหม่]
const role = computed(() => auth.user?.role || 'user')    // <-- [✏️ แก้จากเดิม const role = ref('user')]
const { menu } = useMenu(role)
const drawer = ref(true)
</script>
```

---

### 🛠️ [เฟส 2] จุดที่ 8: แก้ไฟล์ `frontend/pages/users/new.vue` (แก้บั๊กกดปุ่ม Create User แล้วขึ้น 500 `localStorage is not defined`)
เปิด `frontend/pages/users/new.vue` เลื่อนลงไปส่วน `<script setup>` (บรรทัด 86 ถึงท้ายไฟล์):

```javascript
<script setup>
definePageMeta({ ssr: false })                            // <-- [➕ 1. พิมพ์เพิ่มใหม่บรรทัดบนสุด]
import { ref } from "vue";
import { useRouter } from "vue-router";
import axios from "axios";

const router = useRouter();
const config = useRuntimeConfig();                        // <-- [➕ 2. พิมพ์เพิ่มใหม่]
const errorMsg = ref("");
const showPassword = ref(false);

const vals = ref({
  name_th: "",
  email: "",
  password: "",
  role: "evaluatee",
});

const roles = ["evaluatee", "evaluator", "admin"];
const handleSubmit = async () => {
  const token = localStorage.getItem("auth_token");       // <-- [🔀 3. ย้ายจากข้างบนลงมาไว้ใน handleSubmit]
  try {
    await axios.post(
      `${config.public.apiBase || "http://api.mydomain.lab"}/api/users`, // <-- [✏️ 4. แก้ URL API]
      {
        name_th: vals.value.name_th,
        email: vals.value.email,
        password: vals.value.password,
        role: vals.value.role,
      },
      {
        headers: {
          "Content-Type": "application/json",
          ...(token ? { Authorization: `Bearer ${token}` } : {}),
        },
      }
    );
    router.push("/");
  } catch (e) {
    errorMsg.value = e?.response?.data?.message || e.message || "Create failed";
  }
};
</script>
```

---

### 🛠️ [เฟส 2] จุดที่ 9: แก้ไฟล์ `frontend/composables/useMenu.js` (แก้ลิงก์เมนู API Docs)
แทนที่ฟังก์ชัน `export function useMenu` ท้ายไฟล์ `frontend/composables/useMenu.js` ด้วยโค้ดนี้:

```javascript
export function useMenu(roleInput = 'user') {
  const r = isRef(roleInput) ? roleInput : ref(roleInput)
  const config = useRuntimeConfig()

  const menu = computed(() => {
    const key = normalizeRole(r.value)
    const base = String(config.public.apiBase || 'http://api.mydomain.lab').replace(/\/$/, '')
    const sections = MAP[key] || MAP.user
    return sections.map((sec) => ({
      ...sec,
      items: sec.items.map((it) =>
        it.label === 'API Docs' ? { ...it, href: `${base}/docs` } : it
      ),
    }))
  })

  return { menu }
}
```

---

### 🛠️ [เฟส 2] จุดที่ 10 และ 11: แก้ปุ่ม Open API Docs และไอคอนลูกตาดูรหัสผ่าน
* **จุดที่ 10 (`frontend/pages/index.vue` บรรทัด 91):** แก้ลิงก์จาก `http://localhost:7000/docs` $\rightarrow$ เป็น **`http://api.mydomain.lab/docs`**
* **จุดที่ 11 (`frontend/pages/login.vue` บรรทัด 20):** แก้ไอคอนตาจาก `:append-inner-icon="show ? ..."` $\rightarrow$ เป็น **`:append-inner-icon="showPw ? 'mdi-eye-off' : 'mdi-eye'"`**

---

## 🎯 STEP 7.2 (จุดเช็คพอยต์ที่ 2): สั่ง Git Commit & Push รอบที่ 2 เพื่ออัปเดตความถูกต้องทั้งหมดขึ้น Production!

รัน 3 คำสั่งนี้ใน Terminal ของ VS Code เพื่อส่งโค้ดที่แก้บั๊กครบแล้วขึ้นไป Trigger Pipeline รอบที่ 2:

```bash
git add .
git commit -m "Phase 2: Fix admin menu, user creation SSR, and API docs links"
git push
```

> **✅ ผลลัพธ์หลัง Pipeline รอบที่ 2 เขียวครบ 4 Stages:**
> - ได้โชว์กรรมการว่าเมื่อแก้โค้ดแล้ว Push ใหม่ ระบบเกิด **Auto-Trigger CI/CD** อัปเดต Production อัตโนมัติ
> - ล็อกอินแอดมินแล้วเมนูซ้ายแสดงครบ (`Users`, `Periods`, `API Docs`)
> - กดปุ่ม **Create User** สร้างผู้ใช้ใหม่ได้จริงโดยไม่ขึ้น Error 500
> - หน้า **Upload** อัปโหลดไฟล์พร้อมบันทึก `note` ได้ และปุ่ม **Open API Docs** เปิดหน้า Swagger (`http://api.mydomain.lab/docs`) ได้สมบูรณ์ 100%!
