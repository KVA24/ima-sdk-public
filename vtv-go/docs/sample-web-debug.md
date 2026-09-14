# `sample-web-debug.html`: Instream trên web / WebView

Sample tương ứng: [`../sample/sample-web-debug.html`](../sample/sample-web-debug.html)

Cùng module `WI.InstreamSdk` như bản TV nhưng chạy engine Web: nút Skip do SDK render, có xử lý fullscreen, không có `pauseAd()` / `resumeAd()`.

---

## 1. Nhúng

```html
<script src="https://ima-sdk-public.pages.dev/vtv-go/1.0.0/wiinvent-sdk.js"></script>
```

```js
if (!window.WI) { /* phát nội dung bình thường, không chặn người dùng */ }
```

## 2. DOM

```html
<div class="video-wrap">
  <video id="content-video" src="..."></video>
  <div id="ads-container"></div>   <!-- SDK render quảng cáo vào đây -->
  <div id="ad-badge">Quảng cáo</div>
</div>
```

Sample chỉ bật `pointer-events` cho `#ads-container` khi quảng cáo đang phát (class `active`), để lúc bình thường người dùng vẫn bấm được vào player bên dưới.

## 3. Khởi tạo

Sample khởi tạo ở lần `play` đầu tiên của nội dung:

```js
var wiiSdk = new WI.InstreamSdk({
  domId: 'ads-container',
  env: WI.Environment.SANDBOX,       // PRODUCTION khi phát hành
  platform: WI.Platform.WEB,         // bắt buộc: chọn engine Web
  deviceType: WI.DeviceType.PC,      // hoặc PHONE
  tenantId: 2,
  channelId: '<CHANNEL_ID>',
  streamId: '<STREAM_ID>',
  adId: '',
  contentType: WI.ContentType.VIDEO,
  liveTv: false,
  title: '<TIÊU ĐỀ NỘI DUNG>',
  transId: '<MÃ PHIÊN>',
  category: '1, 2',
  keyword: '1, 2',
  segments: '',
  age: '20',
  gender: WI.Gender.MALE,
  userId: '<USER_ID>',
  adPendingTime: '',
  userImpressionLimit: '',
  manufacturer: '', model: '', osName: '', osVersion: '',
  partnerSkipOffset: 7,
  vastLoadTimeout: 5,
  mediaLoadTimeout: 10,
  bufferingVideoTimeout: 10,
  bitrate: 1024,
  alwaysCustomSkip: true,
  isAutoRequestFocus: false,
  isUsePartnerSkipButton: true,
  skipText: 'Bỏ qua sau {0}s',
  skippableText: 'Bỏ qua quảng cáo',
  nonSkippableText: 'Quảng cáo kết thúc sau {0}s'
});

wiiSdk.start().then(function () { /* đã tải lịch quảng cáo */ });
wiiSdk.setDuration(video.duration);
```

## 4. Bơm thời gian nội dung

```js
video.addEventListener('timeupdate', function () {
  if (!isAdPlaying && !video.paused && wiiSdk) {
    wiiSdk.updateTime(video.currentTime, video.duration);
  }
});
```

## 5. Vòng đời một ad break

```js
window.addEventListener('message', function (e) {
  if (!e.data || !e.data.type) return;
  var type = e.data.type;
  if (type.indexOf('WII_') === 0) return;      // kênh nội bộ của SDK, bỏ qua

  if (type === 'PAUSE_CONTENT') {
    if (!video.paused) video.pause();
    adsContainer.classList.add('active');      // bật pointer-events
    wiiSdk.playAd(video.muted);                // truyền trạng thái tiếng của player
    return;
  }
  if (type === 'START') { isAdPlaying = true; return; }
  if (type === 'ALL_ADS_COMPLETED' || type === 'ERROR') {
    isAdPlaying = false;
    adsContainer.classList.remove('active');
    if (video.paused) video.play();
    return;
  }
});
```

Khác bản TV: `playAd(video.muted)` nhận tham số muted để quảng cáo không bật tiếng ngược với player (và để autoplay không bị trình duyệt chặn).

Các `type` khác sample có xử lý: `IMPRESSION`, `CLICK`, `COMPLETE`, `SKIPPED`, `AD_PROGRESS`, `ADS_EMPTY`.

`COMPLETE` chỉ kết thúc một creative trong pod. Chỉ phát tiếp nội dung ở `ALL_ADS_COMPLETED` hoặc `ERROR`.

## 6. Fullscreen

Sample dùng fullscreen của phần tử bọc player (`requestFullscreen` / `webkitRequestFullscreen`), nên quảng cáo trong `#ads-container` vào fullscreen cùng nội dung. Nếu muốn hỗ trợ creative tự yêu cầu fullscreen, xử lý thêm `REQUEST_ENTER_AD_FULLSCREEN` / `REQUEST_EXIT_FULLSCREEN`.

## 7. API dùng trong sample

| Gọi | Khi nào |
|---|---|
| `start()` | Sau khi tạo instance. Trả `Promise`. |
| `setDuration(seconds)` | Khi biết thời lượng nội dung. |
| `updateTime(currentTime, duration)` | Mỗi `timeupdate` của nội dung. |
| `playAd(muted)` | Sau khi nhận `PAUSE_CONTENT` và đã pause nội dung. |
| `skip()` | Nút skip phụ của ứng dụng (bản Web SDK đã tự render nút skip). |
| `destroy()` | Đổi nội dung hoặc rời trang. Sample gọi trước khi load video mới rồi khởi tạo lại. |

Engine Web không có `pauseAd()` / `resumeAd()` / `submitReport()` trên instance. Báo cáo quảng cáo đi qua UI do SDK render.

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
| `partnerSkipOffset` | Số giây trước khi cho bỏ qua, chỉ nhận 2–50 | integer |
| `isUsePartnerSkipButton` | Tôn trọng cờ skippable của VAST thay vì luôn cho skip | boolean |

Engine Web không đọc `alwaysCustomSkip`, `isAutoRequestFocus`, `playerType`, `bitrate`, `thirdPartyToken`, `mediaLoadTimeout`, `bufferingVideoTimeout`; truyền vào cũng bị bỏ qua. `vastLoadTimeout` cố định 5s trong engine. `platform` đọc ở `WI.InstreamSdk` để chọn engine.

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
| `PAUSE_CONTENT` | Tới ad break — pause nội dung rồi gọi `playAd(muted)` |
| `LOADED` | Creative đã tải xong |
| `START` | Quảng cáo bắt đầu phát |
| `IMPRESSION` | Đã ghi nhận impression |
| `AD_PROGRESS` | Tiến độ quảng cáo: `currentTime`, `duration` |
| `CLICK` | Người dùng bấm quảng cáo |
| `PAUSED` | Quảng cáo tạm dừng |
| `SKIPPED` | Người dùng bỏ qua |
| `NEXT` | Chuyển sang creative kế tiếp trong pod |
| `ALL_ADS_COMPLETED` | Hết toàn bộ ad break — phát tiếp nội dung |
| `ERROR` | Lỗi request hoặc phát quảng cáo |
| `RESIZE` / `FULLSCREEN_CHANGE` | Kích thước / trạng thái fullscreen thay đổi |
| `REQUEST_ENTER_AD_FULLSCREEN` / `REQUEST_EXIT_FULLSCREEN` | Creative yêu cầu vào / ra fullscreen |
| `AD_PAUSED_FOR_REPORT` / `AD_RESUMED_AFTER_REPORT` | SDK dừng quảng cáo khi mở form báo cáo và phát lại sau đó |
| `REPORT_SUBMITTED` / `REPORT_FAILED` | Kết quả gửi báo cáo |
| `WII_*` | Kênh nội bộ của SDK — bỏ qua |

---

## 11. Clear

`destroy()` gỡ listener `message` của SDK và hủy ad manager. Container `#ads-container`, badge và trạng thái UI thuộc về ứng dụng: SDK không đụng tới, ứng dụng tự dọn.

```js
wiiSdk.destroy();
wiiSdk = null;
adsContainer.classList.remove('active');   // tắt pointer-events
adBadge.classList.remove('show');
isAdPlaying = false;
```

| Tình huống | Làm gì |
|:---|:---|
| Hết ad break (`ALL_ADS_COMPLETED`) | Bỏ class `active`, ẩn badge, `video.play()`. **Không** `destroy()` — instance còn phải phục vụ break sau |
| Lỗi quảng cáo (`ERROR`) | Như trên: tắt overlay, trả người dùng về nội dung |
| Load video khác | `destroy()` rồi khởi tạo instance mới — sample làm đúng thứ tự này ở nút Load |
| Rời trang / unmount player | `destroy()` |

Sau `destroy()` phải tạo instance mới, không gọi lại `start()` trên instance đã hủy.
