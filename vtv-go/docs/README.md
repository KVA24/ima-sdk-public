# Wiinvent SDK cho VTV Go

```html
<script src="https://ima-sdk-public.pages.dev/vtv-go/1.0.0/wiinvent-sdk.js"></script>
```

Một file, một global `WI`, ba module.

| Cần tích hợp | Module | Đọc | Sample |
|---|---|---|---|
| Quảng cáo trong video, trên HTML TV / STB | `WI.InstreamSdk` | [sample-tv-debug.md](sample-tv-debug.md) | [sample-tv-debug.html](../sample/sample-tv-debug.html) |
| Quảng cáo trong video, trên web / WebView | `WI.InstreamSdk` | [sample-web-debug.md](sample-web-debug.md) | [sample-web-debug.html](../sample/sample-web-debug.html) |
| Banner trong trang, banner khi pause video | `WI.BannerSdk` | [sample-banner-debug.md](sample-banner-debug.md) | [sample-banner-debug.html](../sample/sample-banner-debug.html) |
| Quảng cáo chào mừng khi bật TV | `WI.WelcomeSdk` | [sample-welcome.md](sample-welcome.md) | [sample-welcome.html](../sample/sample-welcome.html) |

Chạy sample qua HTTP, không mở bằng `file://`:

```bash
python3 -m http.server 8000
# mở http://localhost:8000/sdk/vtv-go/sample/sample-tv-debug.html
```
