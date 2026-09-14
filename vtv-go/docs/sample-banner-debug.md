# `sample-banner-debug.html`: Banner (DISPLAY / OVERLAY / WELCOME)

Sample tương ứng: [`../sample/sample-banner-debug.html`](../sample/sample-banner-debug.html)

Banner dạng ảnh, HTML hoặc VAST render vào một slot có sẵn trong trang. Sample gồm bốn phần: banner trong trang, hai banner chạy song song để thử báo cáo, banner pause trên video, và welcome overlay.

---

## 1. Nhúng

```html
<script src="https://ima-sdk-public.pages.dev/vtv-go/1.0.0/wiinvent-sdk.js"></script>
```

Module dùng ở đây là `WI.BannerSdk`, có bộ hằng số riêng: `WI.BannerSdk.ENV`, `TYPE`, `GENDER`, `AD_SIZE`, `BANNER_TYPE`.

## 2. Slot

Slot phải có trong DOM trước khi gọi `start()`, và phải có kích thước lớn hơn 0:

```html
<div id="adPreview" style="position:relative;width:100%;height:300px"></div>
```

SDK tạo `<div class="ad-sdk-wrapper">` bên trong slot, scale creative giữ nguyên tỷ lệ gốc rồi canh giữa. Wrapper co đúng bằng khung creative sau khi scale, nên slot chỉ là vùng giới hạn tối đa: slot lệch tỷ lệ vẫn hiển thị đúng, phần dư là khoảng trống trong suốt. Nút đóng, nút báo cáo và nhãn "Quảng cáo" bám mép creative.

Slot đang là `static` sẽ được SDK đặt `position: relative` và trả lại khi `dismiss()`; slot đã có `position` riêng thì giữ nguyên.

## 3. Khởi tạo

```js
var sdk = new WI.BannerSdk({
  env: WI.BannerSdk.ENV.SANDBOX,        // PRODUCTION khi phát hành
  type: WI.BannerSdk.TYPE.DISPLAY,
  tenantId: 2,
  streamId: '<STREAM_ID>',
  channelId: '<CHANNEL_ID>',
  title: 'Banner Debug',
  transId: 'debug-' + Date.now(),
  category: '1, 2',
  keyword: 'test',
  age: '25',
  gender: WI.BannerSdk.GENDER.MALE,
  userId: '<USER_ID>',
  adPendingTime: '',
  userImpressionLimit: '',
  isUsePartnerSkipButton: true,          // SDK render nút báo cáo + nút đóng
  debug: true                            // tắt ở production
});
```

Một instance quản lý nhiều slot: mỗi `domId` có state, timer và callback riêng.

## 4. Banner trong trang (`DISPLAY`)

```js
sdk.start('adPreview', 'DISPLAY', bannerType, positionId, function (result) {
  // result.status: 'success' | 'fallback' | 'error' | 'delay'
});
sdk.dismiss('adPreview');
```

`bannerType` ở tham số 2 là cách hiển thị (`DISPLAY` / `OVERLAY`), `adSize` ở tham số 3 là loại vị trí (`HOMEPAGE_LARGE_BANNER`, `SUBPAGE_BANNER`, `PAUSE_BANNER`...). Sample lấy `adSize` từ ô select "Banner Type".

Gọi lại `start()` trên cùng slot sẽ hủy kết quả lần trước (cơ chế token): response về muộn của request cũ bị bỏ qua.

## 5. Hai banner song song và báo cáo theo slot

Sample chạy `adPreviewA` và `adPreviewB` cùng lúc với hai `positionId` khác nhau, rồi gửi báo cáo riêng từng slot:

```js
sdk.start('adPreviewA', 'DISPLAY', bannerType, positionIdA, cb);
sdk.start('adPreviewB', 'DISPLAY', bannerType, positionIdB, cb);

sdk.submitReport('adPreviewA', ['MISLEADING'], function (status) {
  // 'SUCCESS' | 'ERROR'
});
```

Event `report_submitted` trả về `{ domId, adId, reasons }`; sample so `adId` này với `adId` của creative đang hiển thị trong slot để chắc chắn báo cáo đúng quảng cáo.

## 6. Banner pause trên video (`OVERLAY`)

Slot nằm đè lên player, mặc định ẩn (`opacity: 0`, `pointer-events: none`), chỉ bật khi có creative:

```html
<div class="video-container">
  <video id="videoDemoMain" controls></video>
  <div class="video-ad-slot" id="videoAdSlot"></div>
</div>
```

```js
video.addEventListener('pause', function () {
  if (isSeeking) return;                      // pause do tua thì bỏ qua
  clearTimeout(pauseTimeout);
  pauseTimeout = setTimeout(function () {     // debounce 150ms
    if (!video.paused || isSeeking) return;
    sdk.start('videoAdSlot', 'OVERLAY', 'PAUSE_BANNER', null, function (result) { ... });
  }, 150);
});

video.addEventListener('play',  function () { sdk.dismiss('videoAdSlot'); });
video.addEventListener('ended', function () { sdk.dismiss('videoAdSlot'); });
```

Với `OVERLAY`, ad server có thể trả `delayOffSet`: số giây tối thiểu giữa hai request cho cùng slot. Trong khoảng đó `start()` không gửi request, callback trả `status: 'delay'`, và SDK phát event `inDelay`.

`skipOffSet: 0` trong response nghĩa là cho đóng ngay: nút đóng hiện lập tức. Không có `skipOffSet` nghĩa là không cho đóng.

## 7. Welcome overlay qua BannerSdk

Sample tạo instance thứ hai với `type: WI.BannerSdk.TYPE.WELCOME` và gọi `start()` không cần `domId`. SDK tự tạo overlay toàn màn hình:

```js
welcomeSdk.start(null, null, null, function (result) { ... });
```

Khung welcome rộng bằng một nửa viewport, chiều cao suy ra từ `ratioWidth`/`ratioHeight` của creative (mặc định 300×250 khi ad server không trả tỷ lệ). Creative dọc cao quá màn hình thì SDK giữ tỷ lệ và thu hẹp khung. Kích thước tính lại khi cửa sổ đổi kích thước.

Welcome dạng video VAST toàn màn hình trên TV dùng module khác, xem [`sample-welcome.md`](sample-welcome.md).

## 8. Event

Đăng ký trước khi gọi `start()`:

```js
['start','loaded','rendered','skip','click','dismiss','error','report_submitted','report_failed']
  .forEach(function (ev) { sdk.on(ev, function (data) { ... }); });
```

| Event | Payload | Khi phát |
|---|---|---|
| `start` | `{ domId }` | Bắt đầu request cho slot. |
| `loaded` | `{ domId, data }` | Đã nhận response. |
| `rendered` | `{ domId, ad }` | Creative đã hiển thị. |
| `click` | `{ domId, ad }` | Người dùng bấm quảng cáo. |
| `skip` | `{ domId, ad }` | Người dùng bấm nút đóng của SDK. |
| `dismiss` | `{ domId }` | Slot đã được dọn. |
| `error` | `{ domId, err }` | Lỗi request hoặc render. |
| `inDelay` | `{ domId, remainingSeconds, delayOffSet }` | `OVERLAY` còn trong thời gian chờ. |
| `report_submitted` / `report_failed` | `{ domId, adId, reasons }` / `{ domId, error }` | Kết quả báo cáo. |

`loaded` chỉ xác nhận có response; dùng `rendered` hoặc callback `status === 'success'` để biết creative đã hiển thị.

## 9. Parameter

| Key | Description | Type |
|:---|:---|---:|
| `tenantId` | Mã tenant Wiinvent cấp | integer |
| `env` | Môi trường ad server | constant |
| `type` | Loại instance: `DISPLAY`, `OUTSTREAM`, `WELCOME` | constant |
| `platform` | Nền tảng, mặc định `WEB` | constant |
| `deviceType` | Loại thiết bị | constant |
| `streamId` | Id nội dung | string |
| `channelId` | Id kênh | string |
| `positionId` | Mã vị trí mặc định; giá trị thật lấy từ `start()` | string |
| `adId` | Lọc theo mã quảng cáo (nếu có) | string |
| `title` | Tiêu đề nội dung | string |
| `transId` | Mã giao dịch do server đối tác sinh | string |
| `category` | Danh sách category của nội dung, cách nhau bằng `,` | string |
| `keyword` | Từ khoá của nội dung (nếu có) | string |
| `age` | Tuổi người dùng (nếu có) | string \| number |
| `gender` | Giới tính (nếu có) | constant |
| `userId` | ID người dùng phía đối tác, gửi lên tham số `uid` | string |
| `adPendingTime` | Gửi lên tham số `apt` | string \| number |
| `userImpressionLimit` | Giới hạn impression theo user, gửi lên `uil` | string \| number |
| `segments` | Danh sách segment id của user, cách nhau bằng `,` | string |
| `token` | Token bổ sung nếu chiến dịch yêu cầu | string |
| `width` / `height` | Ép kích thước wrapper (px). Bỏ trống → theo slot | number |
| `isUsePartnerSkipButton` | SDK render nút báo cáo và nút đóng | boolean |
| `debug` | Bật log `[AdSDK]` trên console | boolean |

`adSize` và `bannerType` truyền ở `start()` chứ không ở constructor. Module banner không đọc `partnerSkipOffset`, `vastLoadTimeout`, `bitrate`, `playerType`, `thirdPartyToken`.

---

## 10. Constant

Module banner có bộ hằng số riêng trên `WI.BannerSdk`:

| Key | Giá trị |
|:---|:---|
| `env` | `WI.BannerSdk.ENV.SANDBOX` · `PRODUCTION` |
| `type` | `WI.BannerSdk.TYPE.DISPLAY` · `OUTSTREAM` · `WELCOME` |
| `bannerType` | `WI.BannerSdk.BANNER_TYPE.DISPLAY` · `OVERLAY` |
| `adSize` | `WI.BannerSdk.AD_SIZE.MINI_BANNER` · `SUBPAGE_BANNER` · `HOMEPAGE_LARGE_BANNER` · `PAUSE_BANNER` |
| `platform` | `WI.BannerSdk.PLATFORM.TV` · `WEB` · `ANDROID` · `IOS` |
| `contentType` | `WI.BannerSdk.CONTENT_TYPE.VOD` · `LIVE` · `FILM` · `VIDEO` |
| `gender` | `WI.BannerSdk.GENDER.MALE` · `FEMALE` · `OTHER` · `NONE` |

---

## 11. Clear slot

Nút đóng của SDK chỉ ẩn chính nó, creative vẫn nằm trong slot. Riêng `WELCOME` thì SDK tự gỡ overlay. Gọi `dismiss()` để trả slot về trạng thái trống:

```js
sdk.on('skip', function (payload) {
  sdk.dismiss(payload.domId);      // người dùng bấm nút đóng
});

sdk.dismiss('videoAdSlot');        // một slot: gỡ wrapper, timer, listener, callback của slot đó
sdk.dismiss();                     // mọi slot của instance
sdk.destroy();                     // kết thúc instance: dọn slot + gỡ listener + hủy report service
```

| Tình huống | Gọi |
|:---|:---|
| Người dùng bấm nút đóng (`skip`) | `dismiss(domId)` |
| Video phát tiếp / kết thúc, không cần pause banner nữa | `dismiss(domId)` |
| Đổi nội dung trang, muốn slot trống trong lúc chờ creative mới | `dismiss(domId)` trước khi `start()` lại |
| Rời màn hình, không dùng instance nữa | `destroy()` |

Sau `dismiss()` instance vẫn dùng tiếp được. Sau `destroy()` phải tạo instance mới, không gọi lại `start()` trên instance đã hủy.
