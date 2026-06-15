# AGENT PLAYBOOK — Thiết kế UI rời & Connect vào dự án khách

> Tài liệu cho **các AI agent** trong hệ thống Long Sang. Mục tiêu: làm UI điện ảnh
> (cinematic landing) **một lần**, rồi **gắn vào bất kỳ dự án khách nào trên GitHub**
> nhanh nhất, ổn định nhất, **tốn ít token nhất**.
> Đọc hết phần TL;DR trước khi làm. Không refactor khi chưa cần.

---

## 0. TRIẾT LÝ (đọc 1 lần, nhớ mãi)

1. **UI là package rời, self-contained.** 1 thư mục = 1 trang chạy được độc lập
   (`index.html` + assets local). Không phụ thuộc framework của khách.
2. **Connect ≠ Rewrite.** Mặc định **mount tĩnh + iframe**, KHÔNG port sang React/Vue
   trừ khi khách yêu cầu tương tác sâu. Port lại = tốn 5–10× token, dễ vỡ hiệu ứng.
3. **Nội dung tách khỏi hiệu ứng.** Chữ/ảnh nằm ở `data-key` + 1 nguồn data (JSON/API).
   Sửa nội dung không đụng tới GSAP/Three.js.
4. **Offline-ready là mặc định khi giao khách.** Vendor lib/font/ảnh về local.

---

## 1. TL;DR — QUY TRÌNH 5 BƯỚC

```
1. DESIGN   →  /cinematic-landing tạo UI self-contained trong  ui/<tên>/index.html
2. VENDOR   →  tải GSAP/Three/font/ảnh về  ui/_assets/  (offline, không CDN)
3. DATAKEY  →  gắn data-key vào chỗ chữ/ảnh cần động (xem §4)
4. CONNECT  →  copy vào public/ của repo khách + link/iframe  (xem §3)
5. VERIFY   →  serve tĩnh, kiểm DOM bằng preview_eval (KHÔNG screenshot canvas)
```

Thời gian chuẩn: thiết kế tách riêng, connect chỉ ~10–15 phút/dự án.

---

## 2. CẤU TRÚC CHUẨN CỦA 1 "UI PACKAGE"

```
website/                      ← package bàn giao (zip nguyên cụm này)
├── index.html                ← gallery/menu (tuỳ chọn)
├── _assets/                  ← DÙNG CHUNG, offline
│   ├── gsap.min.js  ScrollTrigger.min.js  lenis.min.js
│   ├── three.min.js          ← chỉ khi có 3D (xem §6: dùng r137 classic cho offline)
│   ├── three-pp/             ← post-processing classic (đúng thứ tự, xem §6)
│   └── fonts/                ← woff2 + fonts.css (KHÔNG link Google Fonts)
├── <tên-1>/index.html        ← mỗi UI 1 thư mục
├── <tên-2>/index.html
└── <tên-3>/index.html + img/ ← ảnh riêng của trang đặt trong img/
```

**Quy ước đường dẫn**: trong từng `index.html`, asset chung trỏ `../_assets/...`,
ảnh riêng trỏ `img/...` (relative) → chạy được cả `file://` lẫn server.

---

## 3. CONNECT VÀO DỰ ÁN KHÁCH (chọn theo stack)

### 3A. Mount tĩnh — CÁCH MẶC ĐỊNH, nhanh nhất, zero code

| Stack khách | Lệnh | Truy cập |
|---|---|---|
| **Vite / React** | `cp -r website <repo>/public/showreel` | `/showreel/<tên>/index.html` |
| **Next.js** | `cp -r website <repo>/public/showreel` | `/showreel/<tên>/index.html` |
| **Express / PM2** | `app.use('/showreel', express.static('website'))` | `/showreel/...` |
| **Tĩnh thuần / Vercel** | đặt vào thư mục root output | `/website/<tên>/index.html` |

> `public/` được copy nguyên trạng khi build → asset relative vẫn đúng. **Không sửa code app.**

Link mở: `<a href="/showreel/<tên>/index.html" target="_blank">Xem demo</a>`

### 3B. Nhúng trong 1 route của SPA (không mở tab mới)

```jsx
<iframe src="/showreel/<tên>/index.html"
        style={{width:'100%', height:'100vh', border:0}}
        loading="lazy" title="demo" />
```
> An toàn nhất: JS của UI (GSAP/Three) **không đụng** React lifecycle → không leak, không conflict.

### 3C. Port sang component (CHỈ khi bắt buộc)

Dùng khi cần truyền props/state/router chung. Chi phí cao:
- Chuyển init vào `useEffect(()=>{ ...; return ()=>cleanup() },[])`.
- `ScrollTrigger.kill()`, `renderer.dispose()`, huỷ `requestAnimationFrame` khi unmount.
- Import lib qua npm thay vì script tag.
→ **Mặc định KHÔNG làm.** Chỉ làm khi khách yêu cầu rõ.

---

## 4. BƠM NỘI DUNG ĐỘNG (thay placeholder, không vỡ hiệu ứng)

Trong HTML, đánh dấu chỗ động:
```html
<h2 class="headline" data-key="hero_title">Tiêu đề mặc định</h2>
<div class="bg-inner" data-img="hero_bg" style="background-image:url('img/01.png')"></div>
```
Một đoạn JS connect (đặt cuối `index.html` hoặc file `content.js`):
```js
fetch('/api/landing/<slug>').then(r=>r.json()).then(d=>{
  document.querySelectorAll('[data-key]').forEach(el=>{
    if (d[el.dataset.key]) el.textContent = d[el.dataset.key];
  });
  document.querySelectorAll('[data-img]').forEach(el=>{
    if (d[el.dataset.img]) el.style.backgroundImage = `url('${d[el.dataset.img]}')`;
  });
}).catch(()=>{/* giữ nội dung mặc định nếu offline/API lỗi */});
```
Nguồn data: API/Supabase/CMS của khách. **Sửa nội dung 1 lần, dùng mãi.**
Luôn để **fallback nội dung mặc định** trong HTML để trang không trắng khi API fail.

---

## 5. TOKEN-SAVING RULES (bắt buộc cho mọi agent)

1. **KHÔNG screenshot canvas/WebGL** — preview tab thường `hidden` → rAF freeze →
   screenshot timeout, đốt token. Verify bằng `preview_eval` đọc DOM/biến/`renderer.info`.
2. **KHÔNG đọc lại file vừa Edit/Write** — harness đã xác nhận; chỉ đọc khi cần khớp chuỗi.
3. **Tải lib/ảnh bằng 1 lệnh Bash gộp** (`&&`), không từng cái một.
4. **Generate ảnh: preflight `get_cost:true`**, generate batch song song trong 1 block,
   poll `job_display` sau khi chờ đủ (~60–90s), curl tải gộp.
5. **Mặc định mount tĩnh (§3A/3B)** — đừng port sang component nếu không được yêu cầu.
6. **Tái dùng package có sẵn** — copy `_assets/` chung, không tải lại lib cho mỗi trang.
7. **Sửa khu trú** — dùng Edit theo chuỗi chính xác, không Write lại cả file lớn.

---

## 6. GHI CHÚ 3D (Three.js) — TRÁNH BẪY ĐÃ GẶP

- **Online/dev**: dùng ESM importmap (`three@0.160` + `three/addons/`). Gọn, mới.
- **Offline/giao khách** (`file://` chặn ES module): chuyển sang **Three.js r137 classic UMD**:
  - Thứ tự script BẮT BUỘC (sai → `THREE.Pass`/`ShaderPass` undefined):
    `three.min.js → EffectComposer.js → CopyShader.js → ShaderPass.js → MaskPass.js → RenderPass.js → LuminosityHighPassShader.js → UnrealBloomPass.js`
  - `EffectComposer.js` định nghĩa base `THREE.Pass` ⇒ phải nạp trước các Pass khác.
  - `UnrealBloomPass` cần `LuminosityHighPassShader` (KHÔNG bundle) ⇒ nạp trước nó.
  - r137 **không có** `CapsuleGeometry`, `OutputPass`; `outputColorSpace` →
    dùng `renderer.outputEncoding = THREE.sRGBEncoding`.
- Luôn render **1 frame đồng bộ trong init** + safety-net `setTimeout` để trang không trắng.

---

## 7. CHECKLIST BÀN GIAO KHÁCH

- [ ] Mở `index.html` bằng Chrome/Edge chạy được, **không cần internet**.
- [ ] Không còn link CDN/Google Fonts nào (grep `https://` trong HTML/CSS).
- [ ] Ảnh, font, lib đều nằm trong package (relative path).
- [ ] Có `HUONG-DAN.txt` (cách mở + fallback `npx serve .`).
- [ ] Nội dung placeholder đã đánh `data-key` (nếu cần động) hoặc đã điền thật.
- [ ] `prefers-reduced-motion` + mobile breakpoint OK.
- [ ] Đã verify bằng `preview_eval` (DOM/0 console error), KHÔNG bằng screenshot.

---

## 8. VÍ DỤ THỰC TẾ (cụm này)

`product/website/` gồm 3 UI tham chiếu, build theo đúng playbook:
- `cinematic-gemini/`  — GSAP, slow-mo HUD, toggle 2 chế độ.
- `cinematic-windland/`— **2 bản**: `index.html` (full Three 0.160 + Bokeh DOF) · `index.offline.html` (r137 classic, zip khách).
- `cinematic-sougen/`  — 6 ảnh AI + crossfade/Ken Burns, particle, grain.
- `_source/` — snapshot gốc Claude UI (CDN thuần), dùng khi cần diff/khôi phục.

### ⚠️ Đừng ghi đè bản full bằng bản offline

Quá trình "full-offline" **chỉ** được phép tạo `index.offline.html` hoặc thư mục `offline/`.
**KHÔNG** thay `index.html` mặc định nếu đã có hiệu ứng Three 0.160 (BokehPass, OutputPass, CapsuleGeometry).

| Trang | Mất gì khi offline-only? | Gemini/Sougen |
|---|---|---|
| Windland | DOF bokeh (Day), OutputPass, Capsule→Cylinder | Không mất UI — chỉ swap CDN→`_assets/` |
| Hub flagship showreel | Sai font nếu dùng nhầm `fonts.css` của cinematic (Sora) thay Space Grotesk | Dùng `_SHOWREEL/website/_assets/fonts/` (có Space Grotesk) |

---

_Cập nhật: 2026-06. Khi đổi quy ước, sửa file này — đây là nguồn sự thật cho mọi agent._
