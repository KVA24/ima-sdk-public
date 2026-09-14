# `sample-welcome.html`: Welcome ad khi bật TV

Sample tương ứng: [`../sample/sample-welcome.html`](../sample/sample-welcome.html)

Quảng cáo chào mừng toàn màn hình (VAST video hoặc ảnh) phát ngay khi người dùng bật TV / mở ứng dụng, trước khi vào màn hình chính. Sample mô phỏng nút nguồn TV: bật TV → khởi tạo SDK → phát welcome → hết quảng cáo thì vào nội dung.

---

## 1. Nhúng

```html
<script src="https://ima-sdk-public.pages.dev/vtv-go/1.0.0/wiinvent-sdk.js"></script>
```

```js
if (!window.WI || !WI.WelcomeSdk) {
  // Không có SDK: vào thẳng màn hình chính, không chặn người dùng
}
```

Chạy trên TV đời cũ nên toàn bộ trang tích hợp phải là ES5: `var`, `function`, không arrow function, không optional chaining.

## 2. DOM

SDK tự tạo container của nó: một `<div id="wii_Welcome">` `position: fixed`, phủ kín `100vw × 100vh`, `z-index: 9999`, gắn vào `document.body` khi có quảng cáo và gỡ đi khi `destroy()`. Ứng dụng không phải khai báo DOM nào cho creative.

```html
<div id="wiinvent_ads_container_id"></div>   <!-- container của riêng sample, không phải của SDK -->
```

Sample vẫn tạo container riêng và tự hiện / ẩn theo event (`START` → hiện, kết thúc → ẩn) để đặt phần UI của chính nó. `config.domId` truyền vào không được module welcome đọc.

## 3. Thứ tự bắt buộc: listener trước, SDK sau

SDK có thể phát `START` rất sớm. Sample đăng ký `window.addEventListener('message', ...)` trước khi gọi `new WI.WelcomeSdk(...)`. Đăng ký sau thì bỏ lỡ event đầu tiên, màn hình kẹt ở trạng thái chờ.

## 4. Khởi tạo

```js
var wiiSdkWelcome = new WI.WelcomeSdk({
  env: WI.Environment.SANDBOX,       // PRODUCTION khi phát hành
  deviceType: WI.DeviceType.TV,
  tenantId: 2,
  adId: '<AD_ID>',
  transId: '<MÃ PHIÊN>',
  age: '20',
  gender: WI.Gender.MALE,
  userId: '<USER_ID>',
  adPendingTime: '',
  userImpressionLimit: '',
  model: '',
  segments: '1,2,3,11,22,33',
  vastLoadTimeout: 5,                // giây
  bufferingVideoTimeout: 10,         // giây
  bitrate: 1024,
  muted: false,
  isUsePartnerSkipButton: true
});

wiiSdkWelcome.start();
```

Sample gọi `handleAds()` một lần duy nhất (cờ `initAds`) khi bật TV, tránh tạo nhiều instance chồng nhau.

Sample còn truyền một loạt key mà module welcome không đọc (mục 8 liệt kê đủ), giữ lại để tương thích với cấu hình cũ của đối tác. Đếm ngược nút Skip trong sample đọc `wiiSdkWelcome.partnerSkipOffset`, tức chính giá trị trang tự truyền vào.

## 5. Xử lý event

```js
window.addEventListener('message', function (e) {
  var data = e && e.data ? e.data : null;
  var type = data && data.type ? data.type : null;
  if (!type) return;

  if (type === 'START') {
    showAdsContainer();      // hiện container
    renderSkipButton();      // nút Skip do ứng dụng render
    return;
  }
  if (type === 'END' || type === 'COMPLETE' || type === 'All_ADS_COMPLETE' || type === 'NO_ADS') {
    contentPlayBack();       // ẩn container, gỡ nút Skip
    wiiSdkWelcome.destroy();
    return;
  }
  if (type === 'PLAYER_ERROR' || type === 'ERROR') {
    contentPlayBack();       // luôn trả người dùng về màn hình chính
    return;
  }
});
```

Sample còn gọi `wiiSdkWelcome.setSource()` khi nhận `GET_ADS`, nhưng luôn bọc trong `typeof ... === 'function'`. Instance `WelcomeSdk` không expose method này, SDK tự nạp creative bên trong. Ứng dụng không cần xử lý `GET_ADS`.

Các `type` sample có ghi log: `REQUEST`, `LOADED`, `START`, `PAUSED`, `RESUMED`, `ERROR`, `PLAYER_ERROR`, `CLICK`, `IMPRESSION`, `SKIPPED`, `COMPLETE`, `DESTROY`, `FULLSCREEN`, `END`, `All_ADS_COMPLETE`, `NO_ADS`, `BEGIN`, `GET_ADS`, `AD_PAUSED`, `AD_RESUMED`.

Hết quảng cáo, không có quảng cáo, lỗi player: mọi nhánh kết thúc đều phải gọi cùng một hàm đưa người dùng vào màn hình chính. Quên một nhánh là người dùng kẹt màn hình đen khi bật TV.

## 6. Nút Skip do ứng dụng render

TV điều khiển bằng remote nên nút Skip phải thuộc cây focus của ứng dụng. Sample:

1. Nhận `START` → tạo nút, ẩn/khoá, đếm ngược từ `partnerSkipOffset`.
2. Đang đếm: `disabled = true`, chữ "Skip sau {n}s".
3. Hết đếm: `disabled = false`, chữ "Skip ads", bấm được bằng chuột hoặc phím Enter.
4. Bấm → `wiiSdkWelcome.skip()` rồi tự đưa người dùng vào màn hình chính.
5. Kết thúc quảng cáo → gỡ nút và `clearInterval` bộ đếm.

## 7. API dùng trong sample

| Gọi | Khi nào |
|---|---|
| `start()` | Sau khi tạo instance và đã đăng ký listener. |
| `skip()` | Từ nút Skip của ứng dụng, khi đã hết đếm ngược. |
| `pauseAd(cb)` / `resumeAd(cb)` | Người dùng bấm Pause trên remote hoặc ứng dụng mở menu đè lên quảng cáo. `cb(status)` = `'SUCCESS'` / `'ERROR'`. |
| `submitReport(reasons, cb)` | Người dùng báo cáo quảng cáo. `cb(status)` = `'SUCCESS'` / `'ERROR'`. |
| `destroy()` | Khi quảng cáo kết thúc, trước khi vào màn hình chính. Sau đó phải tạo instance mới. |

---

## 8. Parameter

| Key | Description | Type |
|:---|:---|---:|
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
| `vastLoadTimeout` | Timeout tải VAST, giây (mặc định 5) | integer |
| `bufferingVideoTimeout` | Timeout buffering video, giây (mặc định 5) | integer |
| `bitrate` | Bitrate ưu tiên khi chọn media file (mặc định 1000) | number |
| `muted` | Phát quảng cáo ở trạng thái tắt tiếng | boolean |
| `isUsePartnerSkipButton` | Creative dạng banner: SDK render nút báo cáo | boolean |

Request welcome chỉ gửi các tham số trên. `streamId`, `channelId`, `contentType`, `title`, `category`, `keyword`, `partnerSkipOffset`, `alwaysCustomSkip`, `isAutoRequestFocus`, `mediaLoadTimeout`, `playerType`, `thirdPartyToken` không được module welcome đọc, truyền vào cũng bị bỏ qua.

`domId` cũng vậy: creative VAST và creative banner đều render vào overlay `wii_Welcome` do SDK tự tạo (mục 2).

---

## 9. Constant

| Key | Giá trị |
|:---|:---|
| `deviceType` | `WI.DeviceType.TV` · `WI.DeviceType.PC` · `WI.DeviceType.PHONE` · `WI.DeviceType.WEB` |
| `env` | `WI.Environment.SANDBOX` · `WI.Environment.PRODUCTION` · `WI.Environment.VIETTEL_PRODUCTION` · `WI.Environment.WIINVENT_PRODUCTION` |
| `gender` | `WI.Gender.MALE` · `FEMALE` · `OTHER` · `NONE` |
| `contentType` | `WI.ContentType.VOD` · `LIVE_STREAM` · `LIVE` · `TV` · `FILM` · `VIDEO` · `SHORT_VOD` |

---

## 10. Ads Callback

Nhận qua `window.addEventListener('message', ...)`, payload dạng `{ type, ...extra }`.

| Type | Mô tả |
|:---|:---|
| `HAVE_ADS` | Có welcome ad |
| `NO_ADS` | Không có welcome ad — vào thẳng màn hình chính |
| `REQUEST` | Bắt đầu request quảng cáo |
| `GET_ADS` | Đã lấy được link quảng cáo |
| `LOADED` | Creative đã tải xong |
| `START` | Quảng cáo bắt đầu phát |
| `PLAY` | Video quảng cáo phát |
| `IMPRESSION` | Đã ghi nhận impression |
| `BUFFERING` | Quảng cáo đang buffer |
| `PAUSED` / `RESUMED` | Quảng cáo dừng / chạy tiếp |
| `AD_PAUSED` / `AD_RESUMED` | Kết quả của `pauseAd()` / `resumeAd()` |
| `AD_PROGRESS` | Tiến độ quảng cáo |
| `CLICK` | Người dùng bấm quảng cáo |
| `SKIPPED` | Người dùng bỏ qua |
| `USER_CLOSE` | Người dùng đóng quảng cáo |
| `COMPLETE` | Một creative kết thúc |
| `ALL_ADS_COMPLETE` | Hết toàn bộ quảng cáo |
| `END` | Kết thúc phiên welcome |
| `FULLSCREEN` | Trạng thái fullscreen thay đổi |
| `PLAYER_ERROR` | Lỗi player |
| `ERROR` | Lỗi request hoặc phát quảng cáo |
| `DESTROY` | Quảng cáo đã bị hủy |
| `REPORT_SUBMITTED` / `REPORT_FAILED` | Kết quả `submitReport()` |

Mọi nhánh kết thúc (`NO_ADS`, `END`, `ALL_ADS_COMPLETE`, `COMPLETE`, `ERROR`, `PLAYER_ERROR`) đều phải đưa người dùng vào màn hình chính.

---

## 11. Clear

`destroy()` gỡ overlay `#wii_Welcome` khỏi `document.body`, cùng listener `message` và `visibilitychange` của SDK. Nút Skip, bộ đếm và container riêng của trang thuộc về ứng dụng, tự dọn.

```js
wiiSdkWelcome.destroy();
wiiSdkWelcome = null;
clearInterval(skipTimer);          // dừng đếm ngược
skipButton.parentNode.removeChild(skipButton);
displayEle(adsContainer, false);   // ẩn container riêng của trang
```

| Tình huống | Làm gì |
|:---|:---|
| `END`, `COMPLETE`, `All_ADS_COMPLETE` | Dọn UI rồi `destroy()`, vào màn hình chính |
| `NO_ADS` | Vào thẳng màn hình chính (chưa có gì để dọn) |
| `ERROR`, `PLAYER_ERROR` | Dọn UI rồi vào màn hình chính — không để người dùng kẹt |
| Người dùng bấm Skip | `skip()`, rồi dọn UI như khi kết thúc |
| Tắt TV / thoát app giữa chừng | `destroy()` |

Sample gom phần dọn UI vào một hàm `contentPlayBack()` rồi gọi ở mọi nhánh kết thúc, nên không sót đường nào.

Sau `destroy()` phải tạo instance mới, không gọi lại `start()` trên instance đã hủy.
