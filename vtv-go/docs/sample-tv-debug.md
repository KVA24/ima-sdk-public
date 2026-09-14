# `sample-tv-debug.html`: Instream trên HTML TV / STB

Sample tương ứng: [`../sample/sample-tv-debug.html`](../sample/sample-tv-debug.html)

Quảng cáo pre/mid/post-roll chèn vào player video trên HTML TV (Smart TV, STB). Engine này gọi endpoint `/v1/adserving/instr/campaign/2.0/html-tv`, khác endpoint `vmap` của bản web. SDK không thay player của đối tác. Nó giữ lịch quảng cáo, tải VAST/VMAP, render vào một container đè lên player, và phát tín hiệu để ứng dụng tạm dừng rồi phát tiếp nội dung.

---

## 1. Nhúng

```html
<script src="https://ima-sdk-public.pages.dev/vtv-go/1.0.0/wiinvent-sdk.js"></script>
```

Bundle expose global `WI`. Kiểm tra trước khi dùng. Không có SDK thì ứng dụng vẫn phải phát nội dung bình thường:

```js
if (typeof WI === 'undefined') { /* phát nội dung, không chặn người dùng */ }
```

Bundle đã transpile ES5 cho Chrome 38 / WebOS 3.x. Trang tích hợp cũng phải viết ES5: `var`, `function`, không arrow function, không optional chaining, không template literal.

## 2. DOM

Sample đặt container quảng cáo đè lên `<video>` nội dung:

```html
<div id="player-wrap">
  <video id="content-video" src="..."></video>
  <div id="ads-container"></div>   <!-- SDK render quảng cáo vào đây -->
  <div id="ad-overlay-badge">Quảng cáo</div>
  <button id="skip-btn">Bỏ qua</button>   <!-- ứng dụng tự render, xem mục 6 -->
</div>
```

`#ads-container` phải phủ đúng vùng video và nằm trên `<video>` (`position:absolute; inset:0; z-index` cao hơn video).

## 3. Khởi tạo

Sample khởi tạo khi người dùng bấm Play lần đầu (`video.addEventListener('play', ...)`), tránh tạo SDK trước khi biết chắc có phiên xem.

```js
var wiiSdk = new WI.InstreamSdk({
  domId: 'ads-container',
  env: WI.Environment.SANDBOX,       // PRODUCTION khi phát hành
  platform: WI.Platform.TV,          // bắt buộc: chọn engine TV
  deviceType: WI.DeviceType.TV,
  tenantId: 2,
  channelId: '<CHANNEL_ID>',
  streamId: '<STREAM_ID>',
  adId: '',
  contentType: WI.ContentType.VIDEO,
  liveTv: false,
  isUseLegacy: false,                // true cho TV đời rất cũ, theo hướng dẫn của Wiinvent
  title: 'TV Debug',
  transId: 'tv-debug-001',
  category: '1',
  keyword: 'tv,debug',
  segments: '',
  age: '20',
  gender: WI.Gender.MALE,
  userId: '<USER_ID>',
  adPendingTime: '',
  userImpressionLimit: '',
  partnerSkipOffset: 5,              // giây, dùng cho nút Skip do ứng dụng render
  isUsePartnerSkipButton: true,
  alwaysCustomSkip: true
});

wiiSdk.setDuration(video.duration);  // khi đã có loadedmetadata
wiiSdk.start().then(function () { /* đã tải lịch quảng cáo */ });
```

`platform` quyết định engine. Luôn truyền rõ, đừng để SDK đoán theo user-agent.

## 4. Bơm thời gian nội dung

Mid-roll và post-roll chỉ kích hoạt khi ứng dụng báo tiến độ:

```js
video.addEventListener('timeupdate', function () {
  if (!isAdPlaying && !video.paused && wiiSdk) {
    wiiSdk.updateTime(video.currentTime, video.duration);
  }
});
```

Không bơm thời gian trong lúc quảng cáo đang phát.

## 5. Vòng đời một ad break

```
SDK ──postMessage PAUSE_CONTENT──► app: video.pause() → wiiSdk.playAd()
SDK ──START──► app: isAdPlaying = true, render nút Skip
SDK ──AD_PROGRESS──► app: cập nhật đếm ngược
SDK ──ALL_ADS_COMPLETED / ERROR──► app: ẩn nút Skip, video.play()
```

```js
window.addEventListener('message', function (ev) {
  if (!ev.data || !ev.data.type) return;
  var type = ev.data.type;
  if (type.indexOf('WII_') === 0) return;      // kênh nội bộ của SDK, bỏ qua

  if (type === 'PAUSE_CONTENT') {
    if (!video.paused) video.pause();
    wiiSdk.playAd();
    return;
  }
  if (type === 'START') {
    isAdPlaying = true;
    showSkipButton(ev.data.skipOffset);         // skipOffset có thể có trong payload
    return;
  }
  if (type === 'ALL_ADS_COMPLETED' || type === 'ERROR') {
    isAdPlaying = false;
    clearSkipButton();
    if (video.paused) video.play();
    return;
  }
});
```

`COMPLETE` chỉ kết thúc một creative trong pod. Chỉ phát tiếp nội dung khi nhận `ALL_ADS_COMPLETED` hoặc `ERROR`. Bỏ sót thì người dùng kẹt ở màn hình đen.

Các `type` khác sample có xử lý: `REQUEST`, `LOADED`, `IMPRESSION`, `CLICK`, `PAUSED`, `RESUMED`, `AD_PAUSED`, `AD_RESUMED`, `SKIPPED`, `AD_PROGRESS`, `ADS_EMPTY`.

## 6. Nút Skip do ứng dụng render

Trên TV, SDK không render nút Skip. Người dùng điều khiển bằng remote nên nút phải nằm trong cây focus của ứng dụng. Sample làm:

1. Nhận `START` → hiện nút ở trạng thái khoá, đếm ngược từ `skipOffset` (hoặc `partnerSkipOffset`).
2. Hết đếm ngược → cho nút `tabIndex = 0`, gọi `focus()` để remote tới được.
3. Người dùng bấm → gọi `wiiSdk.skip()`.
4. Nhận `SKIPPED` / `ALL_ADS_COMPLETED` → ẩn nút, trả focus về player.

Phím trong sample: `S` skip, `I` init lại SDK, `D` destroy, `P` phát ad ngay, `M` seek giữa video, mũi tên tua / chỉnh âm lượng.

## 7. API dùng trong sample

| Gọi | Khi nào |
|---|---|
| `start()` | Sau khi tạo instance. Trả `Promise`. |
| `setDuration(seconds)` | Khi biết thời lượng nội dung (cần cho break theo phần trăm và post-roll). |
| `updateTime(currentTime, duration)` | Mỗi `timeupdate` của nội dung. |
| `playAd()` | Sau khi nhận `PAUSE_CONTENT` **và** đã pause nội dung. |
| `skip()` | Từ nút Skip của ứng dụng. |
| `pauseAd(cb)` / `resumeAd(cb)` | Người dùng bấm Pause trên remote, hoặc ứng dụng mở menu đè lên player. `cb(status)` = `'SUCCESS'` / `'ERROR'`. |
| `submitReport(reasons, cb)` | Người dùng báo cáo quảng cáo. `cb(status)` = `'SUCCESS'` / `'ERROR'`. |
| `destroy()` | Rời màn hình player / đổi nội dung. Sau đó phải tạo instance mới. |

---

## 8. Parameter

| Key | Description | Type |
|:---|:---|---:|
| `domId` | Id container SDK render quảng cáo vào | string |
| `streamId` | Id nội dung | string |
| `channelId` | Id kênh | string |
| `contentType` | Loại nội dung | constant |
| `title` | Tiêu đề nội dung | string |
| `category` | Danh sách category của nội dung, cách nhau bằng `,` | string |
| `keyword` | Từ khoá của nội dung (nếu có) | string |
| `liveTv` | Nội dung đang phát trực tiếp | boolean |

| `tenantId` | Mã tenant Wiinvent cấp | integer |
| `env` | Môi trường ad server | constant |
| `deviceType` | Loại thiết bị | constant |
| `adId` | Lọc theo mã quảng cáo (nếu có) | string |
| `transId` | Mã giao dịch do server đối tác sinh, dùng đối soát | string |
| `age` | Tuổi người dùng (nếu có) | string \| number |
| `gender` | Giới tính (nếu có) | constant |
| `userId` | ID người dùng phía đối tác, gửi lên tham số `uid` | string |
| `adPendingTime` | Gửi lên tham số `apt`. Chỉ nhận string hoặc số hữu hạn | string \| number |
| `userImpressionLimit` | Giới hạn impression theo user, gửi lên tham số `uil` | string \| number |
| `model` | Model thiết bị. Hãng và OS do SDK tự phát hiện | string |
| `segments` | Danh sách segment id của user, cách nhau bằng `,` | string |
| `isUseLegacy` | Chế độ tương thích TV đời cũ | boolean |
| `vastLoadTimeout` | Timeout tải VAST, giây (mặc định 5) — chỉ dùng ở chế độ legacy | integer |
| `bufferingVideoTimeout` | Timeout buffering video, giây (mặc định 5) — chỉ dùng ở chế độ legacy | integer |
| `bitrate` | Bitrate ưu tiên khi chọn media file (mặc định 1000) — chỉ dùng ở chế độ legacy | number |

Engine TV không đọc `partnerSkipOffset`, `alwaysCustomSkip`, `isAutoRequestFocus`, `isUsePartnerSkipButton`. Nút Skip do ứng dụng render (mục 6) nên sample tự đọc giá trị đếm ngược từ cấu hình của chính trang. `platform` đọc ở `WI.InstreamSdk` để chọn engine.

---

## 9. Constant

| Key | Giá trị |
|:---|:---|
| `platform` | `WI.Platform.TV` · `WI.Platform.WEB` · `WI.Platform.IOS` · `WI.Platform.ANDROID` |
| `deviceType` | `WI.DeviceType.TV` · `WI.DeviceType.PC` · `WI.DeviceType.PHONE` · `WI.DeviceType.WEB` |
| `env` | `WI.Environment.SANDBOX` · `WI.Environment.PRODUCTION` · `WI.Environment.VIETTEL_PRODUCTION` · `WI.Environment.WIINVENT_PRODUCTION` |
| `contentType` | `WI.ContentType.VOD` · `LIVE_STREAM` · `LIVE` · `TV` · `FILM` · `VIDEO` · `SHORT_VOD` |
| `gender` | `WI.Gender.MALE` · `FEMALE` · `OTHER` · `NONE` |

`WI.PlayerType` (`VIDEO_JS`, `SHAKA`, `HLS`, `AKAMAI`) và `WI.EventType` cũng được expose, nhưng hai engine instream trong bundle này không đọc `config.playerType`.

---

## 10. Ads Callback

Nhận qua `window.addEventListener('message', ...)`, payload dạng `{ type, ...extra }`.

| Type | Mô tả |
|:---|:---|
| `PAUSE_CONTENT` | Tới ad break và đã có VAST hợp lệ — pause nội dung rồi gọi `playAd()` |
| `REQUEST` | Bắt đầu request quảng cáo |
| `LOADED` | Creative đã tải xong |
| `START` | Quảng cáo bắt đầu phát, payload có thể kèm `skipOffset` |
| `IMPRESSION` | Đã ghi nhận impression |
| `AD_PROGRESS` | Tiến độ quảng cáo: `currentTime`, `duration` |
| `CLICK` | Người dùng bấm quảng cáo |
| `PAUSED` / `RESUMED` | Quảng cáo dừng / chạy tiếp |
| `AD_PAUSED` / `AD_RESUMED` | Kết quả của `pauseAd()` / `resumeAd()` |
| `AD_PAUSE_FAILED` / `AD_RESUME_FAILED` | Pause / resume thất bại |
| `SKIPPED` | Người dùng bỏ qua |
| `COMPLETE` | Một creative kết thúc — pod có thể còn creative khác |
| `ALL_ADS_COMPLETED` | Hết toàn bộ ad break — phát tiếp nội dung |
| `ADS_EMPTY` | Không còn ad break chưa phát |
| `ERROR` | Lỗi request hoặc phát quảng cáo (`errorCode`, `errorMessage`) |
| `REPORT_SUBMITTED` / `REPORT_FAILED` | Kết quả `submitReport()` |
| `WII_*` | Kênh nội bộ của SDK — bỏ qua |

---

## 11. Clear

`destroy()` gỡ listener `message` của SDK và hủy ad manager. Container `#ads-container`, nút Skip và trạng thái UI thuộc về ứng dụng: SDK không đụng tới, ứng dụng tự dọn.

```js
wiiSdk.destroy();
wiiSdk = null;
clearSkipButton();                 // gỡ nút Skip + clearInterval đếm ngược
adBadge.style.display = 'none';    // ẩn badge "Quảng cáo"
isAdPlaying = false;
```

| Tình huống | Làm gì |
|:---|:---|
| Hết ad break (`ALL_ADS_COMPLETED`) | Ẩn nút Skip và badge, `video.play()`. **Không** `destroy()` — instance còn phải phục vụ break sau |
| Lỗi quảng cáo (`ERROR`) | Như trên: ẩn overlay, trả người dùng về nội dung |
| Đổi nội dung đang phát | `destroy()` rồi khởi tạo instance mới cho nội dung mới |
| Rời màn hình player | `destroy()` |

Sau `destroy()` phải tạo instance mới, không gọi lại `start()` trên instance đã hủy. Sample gọi `destroy()` trước khi load video khác, và ở nút Destroy / phím `D`.
