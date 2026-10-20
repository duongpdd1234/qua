# Technical Specification & Implementation Plan
## Vũ Trụ Ký Ức Đa Giác Quan — Thiệp tương tác 20/10

Tài liệu này mô tả kiến trúc, trạng thái dữ liệu, bốn chương tương tác, hệ thống Studio, quy trình nhúng ảnh Base64 và checklist kiểm thử cho một trang thiệp 20/10 chạy trong **một file `index.html` duy nhất**.

---

## 1. Kiến trúc tổng thể và Tech Stack

Mục tiêu là tạo trải nghiệm mượt trên trình duyệt máy tính và điện thoại, đặc biệt là Mobile Safari và Chrome, hạn chế phụ thuộc và dễ chỉnh sửa.

| Thành phần | Công nghệ / kỹ thuật | Mục đích |
|---|---|---|
| Core Layout | HTML5 + Tailwind CSS qua CDN | Khóa khung nhìn `100dvh`, hạn chế cuộn trang và xử lý bố cục mobile-first |
| Typography | Google Fonts: Dancing Script, Playfair Display, Be Vietnam Pro | Kết hợp phong cách lãng mạn, sang trọng và hiển thị tiếng Việt |
| 3D & chuyển ảnh | CSS3 3D Transforms, `perspective: 1200px`, `preserve-3d`, keyframe animations | Xếp chồng thẻ ảnh 3D, hiệu ứng bay/lật thẻ, hologram glare và quỹ đạo ảnh |
| Canvas 2D | Hai lớp canvas riêng | Lớp sao/bụi phép thuật và lớp sương gương có thể xóa bằng thao tác chạm |
| Audio & Haptics | Web Audio API, `navigator.vibrate` | Tạo nhạc hộp, nhịp tim, tiếng lật ảnh, tiếng tích tắc và rung phản hồi |
| Lưu dữ liệu | FileReader, Canvas downscaling, `localStorage` | Tải ảnh, nén ảnh và lưu cấu hình trên thiết bị |
| JavaScript | Vanilla JavaScript | Không cần framework hoặc bước build |

> **Lưu ý triển khai:** trình duyệt có thể chặn âm thanh cho đến khi người dùng tương tác; `navigator.vibrate` không được hỗ trợ trên mọi thiết bị, đặc biệt là iOS. `localStorage` có giới hạn dung lượng và có thể không phù hợp để lưu bốn ảnh lớn. Cần bắt lỗi `QuotaExceededError`, báo lỗi rõ ràng và giảm kích thước/chất lượng ảnh khi cần. Nếu cần lưu ảnh ổn định với dung lượng lớn, cân nhắc IndexedDB.

---

## 2. Cấu trúc dữ liệu và trạng thái ứng dụng

Dùng `appState` để quản lý chương hiện tại, tiến độ quét, ảnh đang xem, trạng thái phát tự động, âm thanh, lớp sương và đếm ngược. Dùng `configData` để lưu nội dung có thể tùy chỉnh.

Khóa lưu cấu hình:

```js
const STORAGE_KEY = "valentine_2010_universe_v2";

const defaultConfig = {
  herName: "Em yêu",
  secretLock: false,
  photos: [
    {
      url: "https://images.unsplash.com/photo-1518199266791-5375a83190b7?w=800&auto=format&fit=crop&q=80",
      date: "Kỷ niệm #1",
      title: "Nụ cười tỏa nắng",
      caption: "Từ khoảnh khắc đầu tiên nhìn thấy nụ cười của em, thế giới của anh bỗng nhiên dịu dàng hơn hẳn."
    },
    {
      url: "https://images.unsplash.com/photo-1516589178581-6cd7833ae3b2?w=800&auto=format&fit=crop&q=80",
      date: "Kỷ niệm #2",
      title: "Khoảnh khắc ngọt ngào",
      caption: "Mỗi phút giây ở bên em đều trở thành một ký ức mà anh muốn giữ thật lâu."
    },
    {
      url: "https://images.unsplash.com/photo-1511895426328-dc8714191300?w=800&auto=format&fit=crop&q=80",
      date: "Kỷ niệm #3",
      title: "Bên nhau bình yên",
      caption: "Cảm ơn em đã bước vào thế giới của anh và mang theo thật nhiều ánh sáng."
    },
    {
      url: "https://images.unsplash.com/photo-1511988617509-a57c8a288659?w=800&auto=format&fit=crop&q=80",
      date: "Kỷ niệm #4",
      title: "Mãi yêu em",
      caption: "Chỉ cần nhìn thấy em hạnh phúc, mọi mệt mỏi ngoài kia đều trở nên nhẹ nhàng."
    }
  ],
  mirrorQuote: "Phía sau lớp sương mù luôn là hình bóng người con gái anh thương nhất...",
  finalRevealTitle: "Em quay sang nhìn anh đi!",
  finalRevealMessage: "Hoa trên màn hình dù lung linh đến mấy... cũng không sánh bằng bó hoa thật anh đang cầm trên tay dành riêng cho em! Chúc mừng 20/10 người yêu của anh ❤️"
};

const appState = {
  chapter: 1,
  scanProgress: 0,
  activePhotoIndex: 0,
  isScanning: false,
  isAutoPlaying: false,
  isSoundEnabled: true,
  isMirrorCleared: false,
  isCountdownRunning: false,
  isFinalRevealVisible: false
};
```

Các URL ảnh mẫu chỉ là ảnh minh họa. Thay bằng ảnh thật qua Studio hoặc nhúng Base64 theo hướng dẫn ở phần 7.

---

## 3. Đặc tả bốn chương tương tác

### Chương 1 — Quét vân tay đồng bộ nhịp tim

**Giao diện**
- Vòng tròn sinh trắc học với SVG progress ring.
- Bên trong là bốn ảnh chuyển cảnh bằng cross-fade.
- Tia laser quét dọc và hiệu ứng ripple.

**Tương tác**
1. Bắt đầu khi người dùng nhấn giữ bằng Pointer Events: `pointerdown`.
2. Khi nhả tay, hủy tiến trình bằng `pointerup`, `pointercancel` hoặc `lostpointercapture`.
3. Tăng `scanProgress` từ 0 đến 100 trong khoảng 3,2 giây.
4. Chuyển ảnh theo tiến độ:
   - 0–25%: Ảnh 1.
   - 25–50%: Ảnh 2.
   - 50–75%: Ảnh 3.
   - 75–100%: Ảnh 4.
5. Ảnh mờ dần từ `blur(6px)` về `blur(0px)`.
6. Khoảng mỗi 600 ms, phát nhịp tim trầm, tạo ripple và gọi `navigator.vibrate(40)` nếu được hỗ trợ.
7. Khi đạt 100%, tự động chuyển mượt sang Chương 2.

**Yêu cầu kỹ thuật**
- Dùng `performance.now()` để tính tiến độ, tránh cộng dồn sai số bằng `setInterval`.
- Hủy animation frame và timeout khi người dùng nhả tay hoặc rời chương.
- Chỉ chuyển chương một lần khi hoàn tất.

### Chương 2 — Lau mặt gương sương mù

**Giao diện**
- Khung gương cổ điển dạng vòm.
- Ảnh phía dưới có hiệu ứng Ken Burns.
- Canvas `#fog-canvas` phủ lớp sương mờ và dòng hướng dẫn: “Dùng ngón tay lau nhẹ mặt gương...”.

**Thuật toán**
1. Vẽ lớp sương phủ canvas bằng gradient trắng mờ, ví dụ `rgba(235, 240, 248, 0.92)`.
2. Khi có `pointerdown`/`pointermove`, đổi `ctx.globalCompositeOperation = "destination-out"`.
3. Dùng radial gradient mềm, bán kính khoảng 38 px, để xóa sương tại vị trí con trỏ.
4. Sinh 3–5 hạt sáng vàng gần vị trí thao tác.
5. Mỗi khoảng 15 frame, lấy mẫu pixel alpha để ước lượng phần canvas đã xóa.
6. Khi vùng trong suốt vượt 40–42%, làm tan phần sương còn lại, làm sáng ảnh và hiển thị nút sang Chương 3.
7. Có thể thêm nút bỏ qua sau 3 giây để tránh người dùng bị mắc kẹt.

**Lưu ý hiệu năng**
- `getImageData()` trên toàn bộ canvas liên tục khá tốn tài nguyên. Hãy lấy mẫu trên canvas đã thu nhỏ hoặc chỉ kiểm tra thưa; tránh đọc toàn bộ pixel ở mỗi frame.
- Cần tính tọa độ theo kích thước thực của canvas và `getBoundingClientRect()` để thao tác cảm ứng chính xác trên màn hình có mật độ điểm ảnh cao.
- Đặt `touch-action: none` riêng cho vùng canvas để thao tác lau không cuộn trang.

### Chương 3 — Bộ bài ký ức xếp chồng 3D và chuyển ảnh hologram

Đây là chương trọng tâm. **Hiệu ứng chuyển ảnh phải nhìn thấy rõ, mượt và có đủ trạng thái bay ra – thẻ phía sau tiến lên.**

#### Bố cục và CSS 3D

- Render đồng thời cả bốn `.memory-card` bên trong `.deck-stage`.
- Đặt `perspective: 1200px` cho vùng chứa và `transform-style: preserve-3d`.
- Tính vị trí tương đối bằng:

```js
const offset = (cardIndex - activeIndex + 4) % 4;
```

| Offset | Trạng thái | Transform gợi ý | Opacity | Z-index |
|---|---|---|---:|---:|
| 0 | Thẻ hiện tại | `translate3d(0, 0, 0) rotateZ(0deg) scale(1)` | 1 | 30 |
| 1 | Thẻ kế tiếp, phía sau bên phải | `translate3d(18px, 14px, -60px) rotateZ(5deg) scale(0.93)` | 0.75 | 20 |
| 2 | Thẻ thứ ba, phía sau bên trái | `translate3d(-18px, 24px, -120px) rotateZ(-5deg) scale(0.86)` | 0.45 | 10 |
| 3 | Thẻ cuối | `translate3d(0, 32px, -180px) scale(0.8)` | 0 | 5 |

Thẻ hiện tại cần có:
- Ken Burns zoom/pan chậm trên ảnh.
- Lớp hologram dùng gradient sáng quét ngang.
- Hiệu ứng tilt nhẹ theo vị trí chạm/con trỏ, nhưng phải giới hạn góc để không gây chóng mặt.

#### Cơ chế chuyển ảnh

Kích hoạt bằng nút “Lật ảnh tiếp theo”, mũi tên trái/phải, pagination hoặc thao tác vuốt ngang.

1. Thẻ đang xem nhận class `.card-fly-out`, bay khỏi màn hình theo hướng tương ứng. Ví dụ:

```css
.card-fly-out-left {
  transform: translate3d(-135%, -20px, 80px)
    rotateY(-35deg) rotateZ(-15deg);
  opacity: 0;
}
```

2. Bắn khoảng 15 hạt sáng ở viền thẻ.
3. Thẻ phía sau chuyển lên vị trí trung tâm bằng transition:

```css
transition:
  transform 0.65s cubic-bezier(0.22, 1, 0.36, 1),
  opacity 0.5s ease;
```

4. Phát tiếng whoosh nếu âm thanh đang bật.
5. Chạy lại hiệu ứng typewriter cho tiêu đề, ngày kỷ niệm và lời nhắn của ảnh mới.

**Điều khiển**
- Nút trái/phải.
- Chấm pagination có thể bấm.
- Vuốt ngang trên vùng bộ ảnh.
- Nút `▶️ Tự chuyển ảnh`, tự động chuyển mỗi 4–4,5 giây; dừng khi rời chương hoặc vào Chương 4.
- Khi tới ảnh thứ 4, hiển thị nút nổi bật: **“🌱 Gieo hạt giống hạnh phúc (Nhận quà 20/10)”**.

Cần tránh việc người dùng bấm liên tục gây chồng animation. Có thể khóa điều khiển trong khoảng thời gian bay ra, sau đó mở lại. Nếu muốn nút nhận quà chỉ hiện ở ảnh thứ tư, hãy đảm bảo trạng thái pagination và trạng thái ảnh luôn đồng bộ.

### Chương 4 — Hoa hồng pha lê, vòng xoay ảnh 3D và đếm ngược

Khi người dùng bấm nút nhận quà:

1. Đổi nền sang tông đỏ-hoàng kim:

```css
background:
  radial-gradient(circle, #4a0e2e 0%, #120309 100%);
```

2. Hiển thị lồng kính trong suốt ở trung tâm.
3. Vẽ hoa hồng pha lê bằng SVG nhiều lớp; cánh hoa nở dần trong khoảng 3,5 giây.
4. Hiển thị bốn ảnh tròn nhỏ, khoảng 56 × 56 px, bay quanh lồng kính theo quỹ đạo elip 3D.
5. Đếm ngược `3... 2... 1...`; mỗi nhịp có tiếng tick nếu âm thanh bật.
6. Khi về 0, bắn confetti và hiển thị thông điệp cuối cùng để đồng bộ với khoảnh khắc tặng hoa thật.

Quỹ đạo ảnh có thể dùng:

\[
x = R_x \cos(\theta_i), \qquad
y = R_y \sin(\theta_i), \qquad
z = R_z \sin(\theta_i)
\]

Khi `sin(theta) > 0`, ảnh ở phía trước: tăng scale và z-index. Khi `sin(theta) < 0`, ảnh ở phía sau: giảm scale, giảm opacity và đặt dưới lồng kính.

**Thông điệp cuối mặc định**

> **Em quay sang nhìn anh đi!**  
> Hoa trên màn hình dù lung linh đến mấy cũng không sánh bằng bó hoa thật anh đang cầm trên tay dành riêng cho em! Chúc mừng 20/10 người yêu của anh ❤️

**Lưu ý đồng bộ thời gian:** bộ đếm là tín hiệu trực quan để người tặng chuẩn bị đưa hoa ra. Trình duyệt không thể đảm bảo đồng bộ tuyệt đối với thời gian ngoài đời nếu tab bị treo, thiết bị khóa hoặc trình duyệt giảm tốc animation nền. Nên dùng thời điểm đích dựa trên timestamp thay vì chỉ trừ số nguyên theo từng `setInterval`.

---

## 4. Âm thanh và phản hồi xúc giác

Viết class `SoundEngine` sử dụng Web Audio API, không phụ thuộc file MP3 ngoài.

Các âm thanh cần có:
- Nhạc nền music box nhẹ nhàng.
- Nhịp tim trầm, khoảng 60 Hz.
- Tiếng lấp lánh khi lau gương.
- Tiếng whoosh khi lật thẻ.
- Tiếng tick trong đếm ngược.
- Hợp âm chiến thắng khi hoa nở.

Yêu cầu:
- Khởi tạo hoặc `resume()` `AudioContext` sau tương tác đầu tiên của người dùng.
- Có nút bật/tắt âm thanh.
- Dừng hoặc giảm âm lượng nhạc khi chuyển chương nếu phù hợp.
- Dọn oscillator, gain node, timer và animation không còn dùng.
- Bắt lỗi khi Web Audio API không khả dụng.
- Chỉ gọi `navigator.vibrate()` khi hàm tồn tại; không coi rung là tính năng bắt buộc.

---

## 5. Hệ thống Studio và khóa bí mật

Header có tiến trình bốn chương, nút bật/tắt âm thanh và nút `⚙️ Studio`.

Modal Studio cho phép:

1. Tải lên bốn ảnh từ máy tính/điện thoại.
2. Nén ảnh qua FileReader và Canvas:
   - Giới hạn cạnh dài tối đa 900 px.
   - Xuất JPEG chất lượng khoảng `0.8`.
   - Giữ tỷ lệ ảnh.
3. Chỉnh tên bạn gái.
4. Chỉnh câu trích dẫn ở gương.
5. Chỉnh tiêu đề, ngày kỷ niệm và lời nhắn của từng ảnh.
6. Chỉnh tiêu đề và thông điệp cuối.
7. Bật tùy chọn **“🔒 Ẩn nút Cài đặt”**.

Khi bật `secretLock` và lưu, nút Studio bị ẩn. Để mở lại, chạm liên tiếp ba lần vào góc trên bên trái màn hình.

**Lưu ý:** cơ chế chạm ba lần chỉ là cách ẩn giao diện, không phải bảo mật thực sự. Người dùng có thể xóa dữ liệu trình duyệt hoặc sửa mã nguồn để mở lại.

### Lưu cấu hình

- Đọc cấu hình từ `localStorage` khi khởi động.
- Nếu dữ liệu thiếu hoặc lỗi JSON, quay về cấu hình mặc định.
- Khi lưu, bắt lỗi `QuotaExceededError` và thông báo cho người dùng.
- Vì Base64 làm dữ liệu tăng đáng kể, không nên mặc định rằng bốn ảnh chất lượng cao luôn vừa giới hạn `localStorage`. Có thể giảm chất lượng/kích thước hoặc chuyển ảnh sang IndexedDB.
- Nếu ảnh được nhúng thẳng trong `index.html`, không cần lưu lại chuỗi Base64 vào `localStorage` trừ khi người dùng thay ảnh qua Studio.

---

## 6. Checklist kiểm thử

### Kiểm thử ảnh
- [ ] Tải bốn ảnh từ máy tính/điện thoại.
- [ ] Ảnh được nén và hiển thị đúng tỷ lệ.
- [ ] Lưu, tải lại trang và kiểm tra ảnh vẫn còn.
- [ ] Nếu dung lượng vượt quota, giao diện báo lỗi dễ hiểu và không mất toàn bộ cấu hình.
- [ ] Kiểm tra ảnh mẫu từ URL và ảnh Base64.

### Kiểm thử chuyển ảnh Chương 3
- [ ] Bấm nút chuyển ảnh.
- [ ] Bấm mũi tên trái/phải.
- [ ] Bấm pagination.
- [ ] Vuốt ngang trên điện thoại.
- [ ] Thẻ trước bay ra, thẻ sau trượt lên mượt.
- [ ] Hologram, hạt sáng và typewriter chạy đúng.
- [ ] Auto-play dừng khi chuyển chương.
- [ ] Nút nhận quà xuất hiện đúng điều kiện đã chọn.

### Kiểm thử khóa Studio
- [ ] Bật “Ẩn nút Cài đặt” và lưu.
- [ ] Nút Studio biến mất.
- [ ] Chạm ba lần vào góc trên bên trái để mở lại.
- [ ] Tải lại trang và xác nhận trạng thái khóa được giữ.

### Kiểm thử mobile và âm thanh
- [ ] Không có cuộn trang ngoài ý muốn ở từng chương.
- [ ] Kiểm tra trên Mobile Safari và Chrome Android.
- [ ] Thử thao tác bằng một tay.
- [ ] Âm thanh chỉ phát sau tương tác người dùng.
- [ ] Tắt âm thanh không làm dừng các hiệu ứng hình ảnh.
- [ ] Rung không được hỗ trợ thì ứng dụng vẫn hoạt động.
- [ ] Đếm ngược và thông điệp cuối vẫn hoạt động khi tab bị chuyển nền rồi quay lại.

---

## 7. Công cụ đóng gói ảnh Base64

Có thể tạo file trung gian `convert.html`, mở bằng Chrome/Edge, chọn bốn ảnh và sinh JSON chứa ảnh Base64 đã nén. Sau đó sao chép JSON vào cấu hình `photos` của `index.html`.

### Mã nguồn `convert.html`

Lưu toàn bộ đoạn dưới đây thành file `convert.html`:

```html
<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Công cụ đóng gói ảnh 20/10</title>
  <style>
    body {
      font-family: sans-serif;
      background: #fff0f3;
      padding: 24px;
      text-align: center;
      color: #333;
    }
    .card {
      background: white;
      max-width: 600px;
      margin: 0 auto;
      padding: 24px;
      border-radius: 20px;
      box-shadow: 0 10px 30px rgba(0, 0, 0, 0.08);
    }
    input[type="file"] { margin: 15px 0; max-width: 100%; }
    textarea {
      box-sizing: border-box;
      width: 100%;
      height: 220px;
      margin-top: 15px;
      font-family: monospace;
      font-size: 12px;
      padding: 10px;
      border-radius: 8px;
      border: 1px solid #ddd;
    }
    button {
      background: #e63946;
      color: white;
      border: none;
      padding: 12px 25px;
      border-radius: 25px;
      font-size: 16px;
      cursor: pointer;
      font-weight: bold;
      margin-top: 10px;
    }
    button:disabled { opacity: 0.6; cursor: wait; }
  </style>
</head>
<body>
  <div class="card">
    <h2>🌹 Công cụ đóng gói ảnh 20/10</h2>
    <p>Chọn tối đa bốn bức ảnh kỷ niệm từ máy tính hoặc điện thoại:</p>
    <input type="file" id="imageInput" multiple accept="image/*"><br>
    <button id="convertButton" type="button" onclick="processImages()">
      🚀 Chuyển đổi & Nhúng vào Code
    </button>
    <p id="status" role="status" aria-live="polite"></p>
    <textarea id="resultOutput" placeholder="Mã JSON chứa ảnh Base64 sẽ xuất hiện ở đây..."></textarea>
  </div>

  <script>
    const titles = [
      "Nụ cười tỏa nắng",
      "Khoảnh khắc ngọt ngào",
      "Bên nhau bình yên",
      "Mãi yêu em"
    ];

    const captions = [
      "Từ khoảnh khắc đầu tiên nhìn thấy nụ cười của em, thế giới của anh bỗng dịu dàng hơn hẳn.",
      "Mỗi phút giây ở bên em đều trở thành một ký ức mà anh muốn giữ thật lâu.",
      "Cảm ơn em đã bước vào thế giới của anh và mang theo muôn vàn ánh sáng ấm áp.",
      "Chỉ cần nhìn thấy em hạnh phúc, mọi mệt mỏi ngoài kia đều tan biến hết."
    ];

    async function processImages() {
      const input = document.getElementById("imageInput");
      const status = document.getElementById("status");
      const output = document.getElementById("resultOutput");
      const button = document.getElementById("convertButton");
      const files = Array.from(input.files).slice(0, 4);

      if (files.length === 0) {
        status.textContent = "Vui lòng chọn ít nhất một ảnh.";
        return;
      }

      button.disabled = true;
      status.textContent = "Đang xử lý và nén ảnh...";
      output.value = "";

      try {
        const photoArray = [];

        for (let i = 0; i < files.length; i++) {
          if (!files[i].type.startsWith("image/")) {
            throw new Error(`Tệp thứ ${i + 1} không phải ảnh hợp lệ.`);
          }

          const base64 = await resizeAndCompress(files[i]);
          photoArray.push({
            url: base64,
            date: `Kỷ niệm #${i + 1}`,
            title: titles[i],
            caption: captions[i]
          });
        }

        output.value = JSON.stringify(photoArray, null, 2);
        status.textContent =
          `✨ Hoàn tất ${photoArray.length} ảnh. Hãy sao chép JSON vào cấu hình photos.`;
      } catch (error) {
        status.textContent = `Không thể xử lý ảnh: ${error.message}`;
      } finally {
        button.disabled = false;
      }
    }

    function resizeAndCompress(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();

        reader.onerror = () => reject(new Error("Không đọc được tệp ảnh."));
        reader.onload = () => {
          const img = new Image();

          img.onerror = () => reject(new Error("Tệp không phải ảnh hợp lệ hoặc ảnh bị lỗi."));
          img.onload = () => {
            const maxSide = 900;
            const ratio = Math.min(1, maxSide / Math.max(img.width, img.height));
            const width = Math.max(1, Math.round(img.width * ratio));
            const height = Math.max(1, Math.round(img.height * ratio));
            const canvas = document.createElement("canvas");

            canvas.width = width;
            canvas.height = height;

            const ctx = canvas.getContext("2d");
            if (!ctx) {
              reject(new Error("Trình duyệt không hỗ trợ Canvas 2D."));
              return;
            }

            ctx.drawImage(img, 0, 0, width, height);

            try {
              resolve(canvas.toDataURL("image/jpeg", 0.8));
            } catch (error) {
              reject(new Error("Không thể xuất ảnh đã nén."));
            }
          };

          img.src = reader.result;
        };

        reader.readAsDataURL(file);
      });
    }
  </script>
</body>
</html>
```

### Cách sử dụng

1. Tạo file `convert.html` và dán mã nguồn ở trên.
2. Mở file bằng Chrome hoặc Edge.
3. Chọn tối đa bốn ảnh.
4. Bấm **“Chuyển đổi & Nhúng vào Code”**.
5. Sao chép JSON được tạo.
6. Thay mảng `photos` mặc định trong `index.html` bằng JSON đó.

Có thể gửi JSON cho Coding Agent kèm yêu cầu:

> Hãy thay thế trực tiếp mảng `photos` mặc định trong `index.html` bằng JSON ảnh Base64 dưới đây. Giữ nguyên cấu trúc dữ liệu `url`, `date`, `title`, `caption`; đảm bảo mọi chương đều đọc ảnh từ cùng một cấu hình và không cần tải lại ảnh từ URL ngoài.

**Lưu ý:** nhúng Base64 trực tiếp vào HTML làm file lớn hơn đáng kể. Nếu cần triển khai nhanh, nên kiểm tra kích thước file cuối và thời gian tải trên điện thoại. Công cụ chỉ xử lý tối đa bốn ảnh; nếu chọn ít hơn bốn, hãy bổ sung ảnh còn thiếu hoặc để ứng dụng xử lý số lượng ảnh linh hoạt thay vì giả định luôn có đủ bốn ảnh.

---

## 8. Prompt hoàn chỉnh dành cho Coding Agent

Sao chép prompt dưới đây để yêu cầu Agent tạo ứng dụng. Đính kèm tài liệu này và JSON Base64 (nếu đã tạo) khi cần.

```text
Bạn là Senior Creative Frontend Developer. Hãy tạo một ứng dụng web thiệp tương tác 20/10 hoàn chỉnh trong đúng một file index.html, sử dụng HTML5, Tailwind CSS qua CDN, Google Fonts và Vanilla JavaScript. Không dùng framework, không để TODO, không dùng placeholder cho logic cốt lõi và không bỏ qua hiệu ứng chuyển ảnh.

MỤC TIÊU
Tạo trải nghiệm “Vũ Trụ Ký Ức Đa Giác Quan” gồm bốn chương, dẫn dắt người nhận từ quét nhịp tim, lau gương sương mù, xem bộ ảnh kỷ niệm 3D đến khoảnh khắc đếm ngược tặng hoa thật.

YÊU CẦU CHUNG
- Mobile-first, bố cục vừa màn hình với 100dvh; hạn chế cuộn trang nhưng vẫn bảo đảm nội dung có thể dùng trên màn hình thấp.
- Nền sao và bụi phép thuật bằng canvas.
- Header có tiến trình 4 chương, nút bật/tắt âm thanh và nút Studio.
- Dùng Dancing Script, Playfair Display và Be Vietnam Pro.
- Dùng Web Audio API để tạo nhạc hộp, nhịp tim, tiếng lấp lánh, whoosh, tick và hợp âm chiến thắng; không dùng file âm thanh ngoài.
- Hỗ trợ pointer events, touch, chuột và bàn phím ở các thao tác chính.
- Có fallback nếu Web Audio API hoặc navigator.vibrate không khả dụng.
- Mọi timeout, interval và requestAnimationFrame phải được dọn khi chuyển chương hoặc kết thúc hiệu ứng.

CHƯƠNG 1 — BIOMETRIC HEART SCANNER
- SVG progress ring, laser scan và bốn ảnh cross-fade.
- Nhấn giữ khoảng 3,2 giây để tiến độ đạt 100%; nhả tay trước khi hoàn tất thì hủy hoặc đặt lại tiến trình một cách nhất quán.
- Đổi ảnh theo bốn mốc tiến độ, làm ảnh nét dần, tạo nhịp tim khoảng 600 ms, ripple và rung nếu thiết bị hỗ trợ.
- Hoàn tất thì chuyển sang Chương 2 đúng một lần.

CHƯƠNG 2 — INTERACTIVE FOGGY MIRROR
- Ảnh có Ken Burns phía sau canvas phủ sương.
- Dùng destination-out và radial gradient mềm để xóa sương theo vị trí chạm.
- Sinh hạt sáng, kiểm tra phần canvas đã xóa một cách tiết kiệm hiệu năng.
- Khi xóa khoảng 40% lớp sương, làm tan phần còn lại và mở nút sang Chương 3.
- Có nút bỏ qua để người dùng không bị mắc kẹt.

CHƯƠNG 3 — 3D STACKED PHOTO DECK
- Render đồng thời bốn thẻ ảnh với perspective: 1200px và preserve-3d.
- Tính offset bằng (cardIndex - activeIndex + 4) % 4.
- Thẻ hiện tại ở giữa; các thẻ còn lại xếp lớp phía sau với transform, opacity và z-index khác nhau.
- Khi chuyển ảnh, thẻ hiện tại phải bay khỏi màn hình bằng animation 3D; thẻ kế tiếp trượt lên vị trí trung tâm trong khoảng 650 ms với cubic-bezier(0.22, 1, 0.36, 1).
- Có hologram glare, Ken Burns, hạt sáng và typewriter cho nội dung từng ảnh.
- Hỗ trợ nút trước/sau, pagination, swipe ngang và auto-play 4–4,5 giây.
- Chặn thao tác liên tục gây xung đột animation; dừng auto-play khi rời chương.
- Khi tới ảnh thứ tư, hiển thị nút “🌱 Gieo hạt giống hạnh phúc (Nhận quà 20/10)”.

CHƯƠNG 4 — ENCHANTED ROSE DOME
- Khi nhấn nhận quà, đổi nền sang đỏ-hoàng kim để tạo tín hiệu trực quan.
- Tạo lồng kính và hoa hồng pha lê SVG với animation nở trong khoảng 3,5 giây.
- Bốn ảnh tròn quay theo quỹ đạo elip 3D; ảnh phía trước lớn/rõ hơn, ảnh phía sau nhỏ/mờ hơn.
- Hiển thị đếm ngược 3, 2, 1, 0 bằng timestamp để giảm sai số khi tab bị chậm.
- Khi về 0, phát confetti và hiện thông điệp:
  “Em quay sang nhìn anh đi! Hoa trên màn hình dù lung linh đến mấy cũng không sánh bằng bó hoa thật anh đang cầm trên tay dành riêng cho em! ❤️”
- Có nút xem lại từ đầu.

STUDIO
- Modal chỉnh tên người nhận, câu trích dẫn ở gương, tiêu đề/ngày/lời nhắn của bốn ảnh và thông điệp cuối.
- Tải tối đa bốn ảnh; nén ảnh bằng FileReader + Canvas, giới hạn cạnh dài 900 px, JPEG quality 0.8.
- Lưu cấu hình bằng localStorage dưới key valentine_2010_universe_v2.
- Bắt lỗi JSON và QuotaExceededError; nếu bốn ảnh không vừa localStorage, báo lỗi rõ ràng và đề xuất giảm dung lượng hoặc dùng IndexedDB.
- Có tùy chọn ẩn nút Studio. Khi đã khóa, chạm ba lần góc trên bên trái để mở lại; ghi rõ đây chỉ là khóa giao diện, không phải bảo mật thực sự.
- Nếu có JSON Base64 do người dùng cung cấp, dùng trực tiếp làm dữ liệu ảnh mặc định và bảo đảm cả bốn chương cùng sử dụng một nguồn dữ liệu.

CHẤT LƯỢNG VÀ KIỂM THỬ
- Không để lỗi JavaScript khi thiếu ảnh hoặc khi người dùng tải ít hơn bốn ảnh.
- Không để animation hoặc âm thanh chạy nền vô hạn sau khi chuyển chương.
- Tương thích thao tác cảm ứng, xử lý pointercancel và resize canvas đúng kích thước.
- Giao diện hiển thị tốt trên màn hình điện thoại thấp và máy tính.
- Hoàn thiện code trong index.html duy nhất; chỉ dùng convert.html như công cụ phụ nếu người dùng muốn chuyển ảnh Base64.
- Sau khi viết code, rà soát logic chuyển chương, chuyển ảnh, auto-play, lưu cấu hình, khóa Studio và đếm ngược; không tuyên bố đã kiểm thử trên thiết bị thật nếu chưa chạy thử.
```

---

## 9. Tiêu chí nghiệm thu

Chỉ coi sản phẩm hoàn thành khi:

- [ ] Có đủ bốn chương hoạt động.
- [ ] Chương 1 hỗ trợ nhấn giữ và tiến trình tự hoàn tất.
- [ ] Chương 2 xóa sương bằng thao tác vuốt/chạm.
- [ ] Chương 3 có bốn thẻ 3D và animation bay ra/chuyển thẻ rõ rệt.
- [ ] Auto-play, swipe, pagination và typewriter hoạt động đồng bộ.
- [ ] Chương 4 có hoa nở, quỹ đạo ảnh, đếm ngược, confetti và thông điệp cuối.
- [ ] Studio chỉnh sửa được nội dung và ảnh.
- [ ] Dữ liệu tồn tại sau khi tải lại trang hoặc có thông báo lỗi rõ ràng khi vượt dung lượng.
- [ ] Khóa Studio và thao tác mở lại hoạt động.
- [ ] Không có lỗi nghiêm trọng trong console khi thiếu ảnh, tắt âm thanh hoặc chuyển chương nhanh.
- [ ] Giao diện đã được kiểm tra trên trình duyệt mục tiêu trước khi dùng trong khoảnh khắc tặng hoa thật.
