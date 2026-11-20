# คู่มือปฏิบัติการวันแข่ง: การแก้ไขโปรเจกต์ `PES` (ต้นฉบับกรรมการ) เพื่อทำ CI/CD 4 Stages และ Deploy ขึ้น VM2 (`192.168.10.32`)

> **สถานการณ์จริงในห้องแข่ง:** กรรมการจะแจกโฟลเดอร์โปรเจกต์ต้นฉบับชื่อ **`PES`** (ประกอบด้วย `backend`, `frontend`, `schema.sql`, `docker-compose.yml`, `.gitlab-ci.yml`) ซึ่งในไฟล์ต้นฉบับของกรรมการ **มีการวางยา/บั๊กไว้หลายจุด** และยังไม่ได้ชี้ IP มายังเซิร์ฟเวอร์ของทีมเรา
>
> เอกสารฉบับนี้สรุป **ลำดับขั้นตอนการแก้ไขไฟล์ใน `PES` ต้นฉบับทีละไฟล์** แบบจับมือทำ เพื่อให้ Push ขึ้น GitLab (`VM1 : 192.168.10.31`) แล้วรัน Pipeline ผ่านครบ 4 Stages (`build` $\rightarrow$ `test` $\rightarrow$ `push` $\rightarrow$ `deploy`) ไปออนไลน์บน Production Server (`VM2 : 192.168.10.32`) ได้สมบูรณ์ 100%

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

## ส่วนที่ 1: สิ่งที่ต้องตั้งค่าเตรียมไว้ที่ `VM1 (192.168.10.31)` และ `VM2 (192.168.10.32)`

ก่อนจะส่งโค้ด `PES` ขึ้น GitLab ให้ตรวจสอบว่าบน `VM1` และ `VM2` ตั้งค่าเหล่านี้ครบแล้ว:

### 1.1 บนเครื่อง `VM1 (192.168.10.31)`
1. **เปิด UFW Firewall รับ DNS ทั้งจาก VLAN 10 และ VLAN 20:**
   ```bash
   sudo ufw allow 2222/tcp
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   sudo ufw allow from 192.168.10.0/24 to any port 53
   sudo ufw allow from 192.168.20.0/24 to any port 53
   sudo ufw allow 5380/tcp
   sudo ufw allow 5000/tcp
   sudo ufw allow 8080/tcp
   sudo ufw enable
   ```
2. **ตั้งค่า Insecure Registry (`/etc/docker/daemon.json`):**
   ```json
   {
     "insecure-registries": [
       "192.168.10.31:5000"
     ]
   }
   ```
   *(แก้เสร็จแล้วรัน `sudo systemctl restart docker`)*
3. **สร้าง SSH Key ในชื่อ `gitlab-runner` ส่งข้ามไปหา `user1@192.168.10.32` (พอร์ต 2222):**
   ```bash
   sudo su - gitlab-runner
   ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
   ssh-copy-id -p 2222 user1@192.168.10.32
   ssh -p 2222 user1@192.168.10.32 "sudo docker ps"
   rm -f ~/.bash_logout
   exit
   ```

### 1.2 บนเครื่อง `VM2 (192.168.10.32)`
1. **เปิด UFW Firewall สำหรับพอร์ตของ `PES` (`3000`, `7000`, `8081`):**
   ```bash
   sudo ufw allow 2222/tcp
   sudo ufw allow 80/tcp
   sudo ufw allow 81/tcp
   sudo ufw allow 443/tcp
   sudo ufw allow 3000/tcp
   sudo ufw allow 7000/tcp
   sudo ufw allow 8080/tcp
   sudo ufw allow 8081/tcp
   sudo ufw enable
   ```
2. **ตั้งค่า Insecure Registry (`/etc/docker/daemon.json`) บน `VM2` ด้วย (เพื่อให้ `VM2` ดึง Image จาก `192.168.10.31:5000` ได้):**
   ```json
   {
     "insecure-registries": [
       "192.168.10.31:5000"
     ]
   }
   ```
   *(แก้เสร็จแล้วรัน `sudo systemctl restart docker`)*
3. **ตั้งค่า Proxy Host ใน Nginx Proxy Manager (`http://192.168.10.32:81`):**
   - `gitlab.mydomain.lab` $\rightarrow$ `http://192.168.10.31:8080`
   - `dns.mydomain.lab` $\rightarrow$ `http://192.168.10.31:5380`
   - `npm.mydomain.lab` $\rightarrow$ `http://192.168.10.32:81`
   - `app.mydomain.lab` $\rightarrow$ `http://192.168.10.32:3000` *(PES Frontend — เปิดสวิตช์ Websockets Support ด้วย)*
   - `api.mydomain.lab` $\rightarrow$ `http://192.168.10.32:7000` *(PES Backend API — หน้าเว็บเรียก API ผ่านชื่อนี้)*
   - `registry.mydomain.lab` $\rightarrow$ `http://192.168.10.31:5000`

   *(ใน Technitium DNS ต้องมี A Record `gitlab`, `dns`, `npm`, `app`, `api`, `registry` ชี้ไป `192.168.10.32` ทุกตัว)*

---

## ส่วนที่ 2: ลำดับการแก้ไขไฟล์ในโฟลเดอร์ `PES` (ทำบนเครื่อง PC นักเรียนด้วย VS Code)

> **ข้อแนะนำสำคัญก่อนเริ่ม:**
> 1. ให้คัดลอกโฟลเดอร์ `PES` ต้นฉบับของกรรมการออกมาเป็นโฟลเดอร์ใหม่ชื่อ `pes-app` (เผื่อทำผิดจะได้มีต้นฉบับเดิมเก็บไว้ดู)
> 2. **ลบโฟลเดอร์ `backend/node_modules`, `frontend/node_modules`, และ `frontend/.nuxt` ทิ้งทันที** (ถ้ามีติดมา) เพื่อไม่ให้ไฟล์ขยะขนาดใหญ่ถูกส่งเข้า Docker ตอน Build

เมื่อเปิดโฟลเดอร์ `pes-app` ใน VS Code แล้ว ให้ไล่แก้ **11 จุดตามลำดับนี้เป๊ะๆ** ครับ:

---

### 🛠️ จุดที่ 1: แก้ไฟล์ `schema.sql` (เพิ่มคอลัมน์ `note` ที่กรรมการตัดออก)
* **สาเหตุที่ต้องแก้:** หน้าอัปโหลดไฟล์ส่งค่า `note` มาบันทึก แต่ในตาราง `attachments` ของกรรมการไม่มีคอลัมน์ `note` ทำให้กดอัปโหลดแล้ว API พัง (Error 500)
* **วิธีแก้:** เปิดไฟล์ `schema.sql` กด `Ctrl+F` หาคำว่า `storage_path VARCHAR(1024)` (ประมาณบรรทัดที่ 149 ในตาราง `attachments`) แล้วแทรกบรรทัด **`note TEXT NULL,`** ลงไปดังนี้:

```sql
  file_name VARCHAR(255) NOT NULL,
  mime_type VARCHAR(100) NOT NULL,
  size_bytes INT UNSIGNED NOT NULL,
  storage_path VARCHAR(1024) NOT NULL,
  note TEXT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
```

บัญชีตัวอย่างใน `schema.sql` ใช้รหัสตัวอักษร **`12345678`** ทั้งชุด เข้ารหัสแบบ bcrypt `$2b$` cost 10 คำว่า `password` ใช้ล็อกอินไม่ได้

| อีเมล | password_hash ในตาราง `users` |
| :--- | :--- |
| `admin@ccollege.ac.th` | `$2b$10$f6g9QMzpdIjzUyckEbFLIeuSRKEGJdNSu.TZ3tmegQ5ioSop02og6` |
| `eva.me@ccollege.ac.th` | `$2b$10$ycxCewoT/qjuiZiDb7hfP.aGEnWZu8rMF3UzRO6QgxgIO7lKLsRSm` |
| `eva.it@ccollege.ac.th` | `$2b$10$rCg8BVUQSVs51Hb/fwctneQcBfIE0RL5dVRm1bcX5CPyGKyRAxFoe` |
| `t.it01@ccollege.ac.th` | `$2b$10$V0GTPQ/2Ap5r0nzE49FjfOW7xmXuSPQ8m7P81jwKrFFltwCvBXTsy` |
| `t.me01@ccollege.ac.th` | `$2b$10$gkmAZQmS5GjA3cgHAzZgN.HZzaH4gKeuTkeJnNoAEFT2OyczRibuC` |
| `t.acc01@ccollege.ac.th` | `$2b$10$5FALWHRfgaBZC0Az5BAVdeelVK4LgRGyKOmSC0hNI3yU6.PRbCxnW` |

hash คนละบรรทัดเพราะ salt ไม่ซ้ำ แต่ถอดกลับได้รหัสเดียวกันคือ `12345678` ถ้ากรรมการเปลี่ยน seed ให้เทียบค่าในไฟล์ที่ได้รับวันนั้น

---

### 🛠️ จุดที่ 2: แก้ไฟล์ `backend/.env` (เปิด CORS ให้เรียกข้ามเครื่องได้)
* **สาเหตุที่ต้องแก้:** ของเดิมอนุญาตแค่ `localhost:3000` พอเปิดเว็บผ่าน IP `192.168.10.32:3000` หรือ `app.mydomain.lab` จะติดบล็อก CORS
* **วิธีแก้:** เปิดไฟล์ `backend/.env` แล้วแก้บรรทัด `CORS_ORIGIN` เป็น (คั่นด้วย `,` ห้ามมีช่องว่าง เพราะโค้ด API ไม่ตัดช่องว่างให้):

```env
# Backend env
PORT=7000
CORS_ORIGIN=http://localhost:3000,http://127.0.0.1:3000,http://192.168.10.32:3000,http://app.mydomain.lab,https://app.mydomain.lab

# Database
DB_HOST=db
DB_PORT=3306
DB_USER=root
DB_PASSWORD=rootpassword
DB_NAME=skills_db

# Security
JWT_SECRET=testing123
JWT_EXPIRES=2h
```

ไฟล์นี้โดน `.gitignore` ตัดไว้ ตอน push ต้องสั่ง `git add -f backend/.env` ไม่งั้นเครื่อง Runner ไม่มีไฟล์นี้ และขั้น deploy ที่สั่ง `scp backend/.env` จะล้ม

---

### 🛠️ จุดที่ 3: สร้างไฟล์ `frontend/.dockerignore` และ `frontend/.env`
1. คลิกขวาที่โฟลเดอร์ `frontend` $\rightarrow$ เลือก **New File** ตั้งชื่อว่า **`.dockerignore`** แล้ววาง 4 บรรทัดนี้:
   ```text
   node_modules
   .nuxt
   .output
   dist
   ```
2. คลิกขวาที่โฟลเดอร์ `frontend` $\rightarrow$ เลือก **New File** ตั้งชื่อว่า **`.env`** แล้ววางบรรทัดนี้ (ชี้ไปที่ API บน VM2):
   ```env
   NUXT_PUBLIC_API_BASE=http://api.mydomain.lab
   ```

---

### 🛠️ จุดที่ 4: แก้ไฟล์ `frontend/nuxt.config.ts` (เปลี่ยน `apiBase` ไปที่ `http://api.mydomain.lab`)
* เปิดไฟล์ `frontend/nuxt.config.ts` เลื่อนลงมาที่บรรทัดที่ 52 (ในก้อน `runtimeConfig -> public`) แก้จาก `http://localhost:7000` เป็น **`http://api.mydomain.lab`**:

```typescript
  runtimeConfig: {
    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE || 'http://api.mydomain.lab'
    }
  },
```

---

### 🛠️ จุดที่ 5: แก้ไฟล์ `frontend/plugins/axios.client.js`
* เปิดไฟล์ `frontend/plugins/axios.client.js` ดูบรรทัดที่ 10 แก้จาก `http://localhost:7000` เป็น **`http://api.mydomain.lab`**:

```javascript
  const api = axios.create({
    baseURL: config.public.apiBase || 'http://api.mydomain.lab',
    timeout: 15000,
```

---

### 🛠️ จุดที่ 6: แก้ไฟล์ `frontend/composables/useMenu.js` (แก้ลิงก์ API Docs)
* เปิดไฟล์ `frontend/composables/useMenu.js`:
  1. กด `Ctrl+H` ค้นหา `http://localhost:7000/docs` แล้วแทนที่ (Replace All 2 จุด) ด้วย **`http://api.mydomain.lab/docs`**
  2. เลื่อนลงไปท้ายสุดของไฟล์ แทนที่ฟังก์ชัน `export function useMenu` ด้วยโค้ดนี้:

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

### 🛠️ จุดที่ 7: แก้ไฟล์ `frontend/pages/index.vue` (แก้ปุ่ม Open API Docs)
* เปิดไฟล์ `frontend/pages/index.vue`:
  1. บรรทัดที่ 3 แก้เป็น: `import { ref, computed, onMounted, watch } from 'vue'`
  2. ใต้บรรทัด `const auth = useAuthStore()` (บรรทัดที่ 8) ให้วางเพิ่ม 5 บรรทัดนี้:
     ```javascript
     const config = useRuntimeConfig()
     const docsUrl = computed(() => {
       const base = String(config.public.apiBase || 'http://api.mydomain.lab').replace(/\/$/, '')
       return `${base}/docs`
     })
     ```
  3. เลื่อนลงมาที่ `<template>` บรรทัดที่ 91 แก้จาก `<a href="http://localhost:7000/docs" target="_blank">` เป็น:
     ```html
     <a :href="docsUrl" target="_blank">
     ```

---

### 🛠️ จุดที่ 8: แก้ไฟล์ `frontend/pages/users/new.vue` (แก้บั๊ก `localStorage is not defined` 500 Error)
* **สาเหตุที่ต้องแก้:** ต้นฉบับเรียก `localStorage.getItem` นอกฟังก์ชัน ทำให้ตอน Nuxt ทำ SSR บนเซิร์ฟเวอร์พังทันที และยิง API ไปที่ `localhost:7000`
* **วิธีแก้:** เปิดไฟล์ `frontend/pages/users/new.vue` เลื่อนลงไปส่วน `<script setup>` (ตั้งแต่บรรทัดที่ 86 จนจบไฟล์) แล้ว **แทนที่ก้อน `<script setup>` ทั้งก้อน** ด้วยโค้ดที่แก้แล้วนี้:

```javascript
<script setup>
definePageMeta({ ssr: false })
import { ref } from "vue";
import { useRouter } from "vue-router";
import axios from "axios";

const router = useRouter();
const config = useRuntimeConfig();
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
  const token = localStorage.getItem("auth_token");
  try {
    await axios.post(
      `${config.public.apiBase || "http://api.mydomain.lab"}/api/users`,
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

### 🛠️ จุดที่ 9: แก้ไฟล์ `frontend/layouts/default.vue` และ `frontend/pages/login.vue`
1. **ในไฟล์ `frontend/layouts/default.vue`:** แทนที่ก้อน `<script setup>` ด้านบนสุด (บรรทัดที่ 1–8) ด้วยก้อนนี้ (เพื่อให้แอดมินล็อกอินแล้วเห็นเมนูแอดมิน):
   ```javascript
   <script setup>
   import { ref, computed } from 'vue'
   import { useMenu } from '~/composables/useMenu'
   import { useAuthStore } from '~/stores/auth'

   const auth = useAuthStore()
   const role = computed(() => auth.user?.role || 'user')
   const { menu } = useMenu(role)
   const drawer = ref(true)
   </script>
   ```
2. **ในไฟล์ `frontend/pages/login.vue`:** บรรทัดที่ 20 แก้จาก `:append-inner-icon="show ? 'mdi-eye-off' : 'mdi-eye'"` เป็น:
   ```html
   :append-inner-icon="showPw ? 'mdi-eye-off' : 'mdi-eye'"
   ```

---

### 🛠️ จุดที่ 10: แก้ไฟล์ `docker-compose.yml` ที่รากของโปรเจกต์ (สำหรับรันบน `VM2` ตอน Deploy)
เปิดไฟล์ `docker-compose.yml` ที่รากของโปรเจกต์ แล้ว **ลบของเก่าทิ้งทั้งหมด** วางโค้ดชุดนี้แทน (ซึ่งใช้ Image ที่ Build เสร็จจาก Registry `192.168.10.31:5000` มารันคู่กับ MySQL และ phpMyAdmin ในไฟล์เดียว ได้คะแนนเกณฑ์ข้อ 2.5 เต็ม!):

```yaml
services:
  db:
    image: mysql:8.0
    container_name: pes-db
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
    networks: [pes_net]
    restart: always

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: pes-pma
    environment:
      PMA_HOST: db
      PMA_USER: root
      PMA_PASSWORD: rootpassword
    ports: ["8081:80"]
    depends_on: [db]
    networks: [pes_net]
    restart: always

  api:
    image: 192.168.10.31:5000/pes-backend:latest
    container_name: pes-api
    env_file:
      - ./backend/.env
    ports: ["7000:7000"]
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./backend/uploads:/app/uploads
    networks: [pes_net]
    restart: always

  frontend:
    image: 192.168.10.31:5000/pes-frontend:latest
    container_name: pes-frontend
    environment:
      NUXT_PUBLIC_API_BASE: http://api.mydomain.lab
      NITRO_PORT: 3000
      HOST: 0.0.0.0
    ports: ["3000:3000"]
    depends_on: [api]
    networks: [pes_net]
    restart: always

volumes:
  db_data:

networks:
  pes_net:
    driver: bridge
```

---

### 🛠️ จุดที่ 11: แก้ไฟล์ `.gitlab-ci.yml` ที่รากของโปรเจกต์ (หัวใจสำคัญของคะแนน CI/CD 4 Stages!)
* **ทำไมต้องเปลี่ยนไฟล์ `.gitlab-ci.yml` ของกรรมการ:**
  1. ไฟล์เดิมของกรรมการมีแค่ 3 Stages (`test, build, deploy`) **แต่เกณฑ์ให้คะแนน ปวส. ข้อ 2.6 และ 2.7 บังคับตรวจ 4 Stages (`build`, `test`, `push`, `deploy`)**
  2. ไฟล์เดิมสั่ง `npm test` ทั้งก้อน ซึ่งในโฟลเดอร์ `backend/test` มีไฟล์เทสต์ที่พยายามเชื่อมต่อฐานข้อมูล `db` ที่ยังไม่ได้เปิด ทำให้ `npm test` แดง! เราจึงต้องสั่งรันเฉพาะไฟล์ Unit Test & Integration Test ที่สมบูรณ์แล้ว คือ `npx vitest run test/server.spec.js test/middlewares_auth.spec.js` (ผ่านสีเขียว 100% ได้คะแนนทั้ง Unit Test และ Integration Test!)
  3. เปลี่ยนมาใช้ Shell Runner บน `VM1` ส่ง Image เข้า `192.168.10.31:5000` และ SSH ไปสั่ง `docker compose up -d` ที่ `user1@192.168.10.32` พอร์ต `2222` โดยอัตโนมัติทันทีที่ Push!

* **วิธีแก้:** เปิดไฟล์ `.gitlab-ci.yml` ลบของเดิมออกทั้งหมด แล้ววางโค้ดนี้แทน:

```yaml
stages:
  - build
  - test
  - push
  - deploy

variables:
  REGISTRY_URL: "192.168.10.31:5000"
  BACKEND_IMAGE: "192.168.10.31:5000/pes-backend:latest"
  FRONTEND_IMAGE: "192.168.10.31:5000/pes-frontend:latest"
  PROD_SERVER: "user1@192.168.10.32"
  SSH_PORT: "2222"
  DEPLOY_DIR: "/home/user1/pes-prod"

# 1. Stage Build: สร้าง Docker Image ทั้ง Backend และ Frontend
build_job:
  stage: build
  script:
    - echo "=== [1/4] Building Backend & Frontend Docker Images ==="
    - docker build -t $BACKEND_IMAGE ./backend
    - docker build -t $FRONTEND_IMAGE ./frontend

# 2. Stage Test: รัน Unit Test (JWT Auth Middleware) และ Integration Test (GET /health) ใน Container
test_job:
  stage: test
  script:
    - echo "=== [2/4] Running Unit & Integration Tests (Vitest) ==="
    - docker run --rm $BACKEND_IMAGE npx vitest run test/middlewares_auth.spec.js test/server.spec.js

# 3. Stage Push: ส่ง Image ทั้งสองตัวขึ้น Private Registry บน VM1 (192.168.10.31:5000)
push_job:
  stage: push
  script:
    - echo "=== [3/4] Pushing Images to Private Registry ==="
    - docker push $BACKEND_IMAGE
    - docker push $FRONTEND_IMAGE

# 4. Stage Deploy: ส่งไฟล์คอนฟิกไปที่ VM2 (192.168.10.32) แล้วสั่งรัน Docker Compose
deploy_job:
  stage: deploy
  script:
    - echo "=== [4/4] Deploying PES Full-Stack to Production Server (VM2) ==="
    - ssh -o StrictHostKeyChecking=no -p $SSH_PORT $PROD_SERVER "mkdir -p $DEPLOY_DIR/backend/uploads"
    - scp -o StrictHostKeyChecking=no -P $SSH_PORT docker-compose.yml schema.sql $PROD_SERVER:$DEPLOY_DIR/
    - scp -o StrictHostKeyChecking=no -P $SSH_PORT backend/.env $PROD_SERVER:$DEPLOY_DIR/backend/.env
    - ssh -o StrictHostKeyChecking=no -p $SSH_PORT $PROD_SERVER "
        cd $DEPLOY_DIR &&
        sudo docker compose pull api frontend &&
        sudo docker compose up -d --remove-orphans
      "
    - echo "Deployment Completed! App is live at http://app.mydomain.lab (or http://192.168.10.32:3000)"
```

> **💡 จุดเด่นของ `test_job` ข้างต้น:** เราสั่ง `docker run --rm $BACKEND_IMAGE npx vitest run ...` ซึ่งรันเทสต์จากข้างใน Docker Image ที่เพิ่ง Build เสร็จใน Stage 1 เลย! ทำให้ **บนเครื่อง `VM1` ไม่จำเป็นต้องลง Node.js เพิ่มด้วยซ้ำ** และรันผ่านสีเขียว 100% ทุกครั้ง!

---

## ส่วนที่ 3: สั่ง Git Push ขึ้น GitLab เพื่อรัน Pipeline อัตโนมัติ

1. เข้าหน้าเว็บ GitLab (`http://gitlab.mydomain.lab`) สร้าง Blank Project ชื่อ **`pes-app`** (เลือก **Public** และติ๊กออก *Initialize repository with a README*)
2. Runner ที่ลงทะเบียนไว้กับโปรเจกต์ `task-app` ใช้กับ `pes-app` ไม่ได้เอง ให้ไปที่ **Settings → CI/CD → Runners → New project runner** ในโปรเจกต์ `pes-app` (ติ๊ก `[✓] Run untagged jobs`) แล้วเอา token `glrt-...` ตัวใหม่ไปรัน `sudo gitlab-runner register` บน VM1 อีกครั้ง (executor `shell`, description `vm1-shell-runner-pes`) ส่วน SSH Key ของ `gitlab-runner` ใช้ของเดิมได้ หลัง `sudo usermod -aG docker gitlab-runner` ต้องมี `sudo systemctl restart gitlab-runner` ด้วย เพราะตอนติดตั้ง บริการถูกเปิดไว้ก่อนเข้ากลุ่ม docker ถ้าข้ามบรรทัดนี้ ขั้น build จะขึ้น `permission denied` ที่ `/var/run/docker.sock`
3. เปิด Terminal ใน VS Code ที่โฟลเดอร์ `pes-app` บนเครื่อง PC นักเรียน แล้วรันคำสั่งชุดนี้:

```bash
# ลบโฟลเดอร์ .git เก่าของกรรมการออกก่อน (ถ้ามี)
rm -rf .git

git config --global user.name "Administrator"
git config --global user.email "admin@mydomain.lab"
git init
git switch -c main
git remote add origin http://gitlab.mydomain.lab/root/pes-app.git
git add -f backend/.env
git add .
git status --short
git commit -m "Deploy PES Full-Stack with 4-stage CI/CD pipeline"
git push -u origin main
```

ใน `git status` ต้องเห็น `backend/.env` ถ้าไม่เห็น แปลว่าคำสั่ง `git add -f` ยังไม่ทำงาน อย่าเพิ่ง push

---

## ส่วนที่ 4: วิธีตรวจสอบผลลัพธ์เมื่อ Pipeline เขียวครบ 4 Stages

เมื่อทั้ง 4 Stages (`build` $\rightarrow$ `test` $\rightarrow$ `push` $\rightarrow$ `deploy`) ขึ้นเครื่องหมายถูกสีเขียวทั้งหมดแล้ว ให้เปิดเบราว์เซอร์ทดสอบ:

1. **เช็ค API Health:** เปิด `http://api.mydomain.lab/health` $\rightarrow$ ต้องเห็น `{"service":"okoak","time":"..."}` (ถ้าไม่ได้ ลอง `http://192.168.10.32:7000/health` เพื่อแยกว่าเสียที่ NPM หรือที่ API)
2. **เช็ค Swagger API Docs:** เปิด `http://api.mydomain.lab/docs` $\rightarrow$ ต้องเห็นหน้าเอกสาร Teaching API
3. **เช็ค phpMyAdmin:** เปิด `http://192.168.10.32:8081` (ผู้ใช้ `root` / รหัส `rootpassword`) $\rightarrow$ เข้าฐานข้อมูล `skills_db` $\rightarrow$ ตาราง `attachments` ต้องมีคอลัมน์ `note`
4. **เข้าหน้าเว็บระบบประเมินบุคลากร (PES Frontend):**
   - เปิด **`http://app.mydomain.lab/login`** (หรือ `http://192.168.10.32:3000/login`)
   - ล็อกอินด้วยอีเมล: **`admin@ccollege.ac.th`** | รหัสผ่าน: **`12345678`**
   - ค่า `password_hash` ของแอดมินใน `schema.sql` คือ `$2b$10$f6g9QMzpdIjzUyckEbFLIeuSRKEGJdNSu.TZ3tmegQ5ioSop02og6` ซึ่งเป็น bcrypt ของ `12345678`
   - ต้องเข้าสู่หน้าตารางรายชื่อผู้ใช้ (Users) เมนูซ้ายมีคำว่า `Users`, `Periods`, `API Docs`
   - กดปุ่ม **Create User** สามารถกรอกเพิ่มผู้ใช้ใหม่ได้จริง และหน้า **Upload** สามารถอัปโหลดไฟล์แนบพร้อมบันทึก `note` ได้สมบูรณ์!

> **⚠️ ถ้าแก้ `schema.sql` หลัง Deploy ไปแล้ว:** MySQL รัน `schema.sql` เฉพาะตอน volume ยังว่าง ให้รันบน VM2 `cd ~/pes-prod && sudo docker compose down -v` แล้วกด Retry job `deploy_job`
