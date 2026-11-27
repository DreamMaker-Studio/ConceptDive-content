---
id: android-release-r8-signing
title: R8, ký app, lên Play
domain: android
tags: [android, r8, proguard, app-signing, play-store]
collections: []
related: [android-gradle-build, android-crashlytics-and-fcm, kotlin-inline-function]
created: 2026-11-27
sources:
  - Shrink, obfuscate, and optimize your app — Android Developers | https://developer.android.com/build/shrink-code
  - Sign your app — Android Developers | https://developer.android.com/studio/publish/app-signing
  - About Android App Bundles — Android Developers | https://developer.android.com/guide/app-bundle
diagrams: [Cơ chế|r8-signing-play-delivery.png|Luồng R8 cắt mã thừa và rút gọn tên lớp ra tệp DEX kèm mapping.txt, ký bằng Upload Key trên gói AAB để Google Play sinh Split APKs ký bằng App Signing Key]
---

## Bản 30 giây
Bản cài biên dịch thô kéo theo hàng nghìn hàm thư viện thừa và giữ nguyên tên lớp nên ai cũng dịch ngược đọc được. Trước khi phát hành, `R8` cắt bỏ mã thừa, đổi tên lớp thành ký tự ngắn và xuất gói ứng dụng ký khóa mật mã. Cuối cùng, bạn đẩy gói `.aab` lên Google Play để chợ ứng dụng tự cắt riêng bản cài vừa khít cho từng dòng máy.

## Cần biết trước
- android-gradle-build

## Vấn đề buộc nó ra đời
Khi nhúng nhiều thư viện, tệp thực thi `classes.dex` nhanh chóng chạm trần 65.536 tham chiếu hàm (giới hạn chỉ số 16-bit của `DEX`). Trước đây, quy trình phát hành phải chạy `ProGuard` cắt mã thừa rồi mới biên dịch sang `DEX`, làm thời gian đóng gói rất lâu. Tệ hơn, nếu lập trình viên làm mất tệp khóa `.jks` cá nhân, họ vĩnh viễn không thể cập nhật ứng dụng trên Google Play.

Để giải quyết, Google ra mắt định dạng `Android App Bundle` (`.aab`) tại Google I/O 2018 (bắt buộc cho ứng dụng mới từ tháng 8 năm 2021). Đi kèm là cơ chế `Play App Signing` tách riêng khóa tải lên và khóa phân phối chính thức. Đến năm 2019, từ bản `Android Gradle Plugin` 3.4.0, Google bật mặc định `R8`. Trình biên dịch này gộp việc cắt mã, làm rối tên và xuất `DEX` vào một bước.

## Nó thực sự là cái gì
`R8` không phải là lớp mã hóa chống dịch ngược tuyệt đối. Tệp `.aab` cũng không phải là bộ cài chạy trực tiếp được trên điện thoại. Thực chất, `R8` là trình biên dịch biến bytecode Java thành `DEX` kết hợp tỉa cây tham chiếu tĩnh (*Tree-Shaking*). Còn `.aab` là gói nguyên liệu được bạn ký bằng `Upload Key` để Google Play tách thành các `Split APKs` và ký lại bằng `App Signing Key`.

## Cơ chế
Khi bật `isMinifyEnabled = true`, `R8` đi từ các điểm neo trong `AndroidManifest.xml` và các luật `-keep` trong `proguard-rules.pro` để lần theo đồ thị gọi hàm. Lớp không nối tới điểm neo bị xóa, còn tên lớp sống sót bị đổi thành `a.b.a` và ghi sổ đối chiếu vào `mapping.txt`. Nếu bật thêm `isShrinkResources = true`, công cụ xóa tiếp ảnh và XML thừa trước khi bạn đóng gói tệp `.aab` ký bằng `Upload Key`. Từ gói `.aab` tải lên Play Console, Google Play sinh các `Split APKs` vừa khít cấu hình máy người dùng và đóng dấu bằng `App Signing Key`.

## Đánh đổi
Ứng dụng giảm mạnh dung lượng, tránh chạm trần 65.536 hàm và cho phép xin cấp lại `Upload Key` nếu bạn lỡ mất tệp `.jks`. Đổi lại, stack trace khi sập sẽ toàn tên `a.b.a` nếu thiếu `mapping.txt`. Các lớp gọi ngầm lúc chạy cũng sẽ bị xóa nhầm nếu thiếu luật `-keep`.

## Nguyên lý gốc
Chỉ đóng gói đúng những nhánh mã và tài nguyên mà thiết bị đích thực sự chạm tới. Đồng thời, luôn tách khóa người gửi khỏi khóa phân phối.

## Key insight
> `R8` không biết lớp nào được gọi qua phản chiếu (*reflection*) lúc chạy; khi thiếu đường nối tĩnh và luật `-keep`, nó xóa lớp đó.

## Khuôn mẫu chuẩn
```kotlin
android {
    buildTypes {
        getByName("release") {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

## Bẫy & ranh giới
Nếu lớp JSON chỉ đọc qua tên trường lúc chạy (*reflection*) mà thiếu `@Keep` hoặc luật `-keep`, `R8` sẽ đổi tên trường thành `a`, `b`. Triệu chứng là bản `debug` chạy đúng nhưng bản `release` trả về toàn dữ liệu `null` hoặc sập `ClassNotFoundException`. Ngược lại, đừng chép bừa luật `-keep class com.example.** { *; }` vì nó làm mất tác dụng cắt mã. Cũng nhớ đăng ký `SHA-1` của `App Signing Key` lên Firebase, nếu không đăng nhập Google sẽ lỗi trên bản tải từ Play.

## Câu hỏi thực chiến
**Hỏi:** Vì sao tính năng gọi API chạy mượt ở bản debug nhưng vừa xuất bản release lên máy là dữ liệu bị null hoặc sập `ClassNotFoundException`?
Vì ở bản `release`, `R8` đã xóa hoặc đổi tên các lớp dữ liệu không được gọi trực tiếp bằng câu lệnh thông thường. Khi thư viện phân tích JSON tìm trường theo tên gốc lúc chạy, nó sẽ không tìm thấy nữa. Bạn cần gắn chú thích `@Keep` lên lớp dữ liệu đó hoặc dùng bộ sinh mã lúc biên dịch.

**Hỏi:** Có thể chép trực tiếp tệp `.aab` vào điện thoại để bấm cài đặt như tệp `.apk` được không?
Không. Tệp `.aab` chỉ là gói chứa mã và tài nguyên thô dành cho máy chủ Google Play chứ không phải gói cài đặt hoàn chỉnh. Muốn cài thử từ tệp `.aab` lên máy thật ở dưới máy tính, bạn phải dùng công cụ `bundletool` để trích xuất ra bộ `Split APKs`.

**Hỏi:** Nếu làm mất tệp keystore `.jks` trên máy tính khi đã dùng Play App Signing thì ứng dụng có bị mất khả năng cập nhật không?
Không. Vì khóa chính thức `App Signing Key` dùng để ký bản cài trên máy người dùng đã nằm an toàn trên hạ tầng của Google. Bạn chỉ cần tạo một tệp `.jks` mới làm `Upload Key` rồi gửi yêu cầu đặt lại khóa tải lên trong Google Play Console.

## Ý bắt buộc khi giải thích lại
- [ ] Giải thích được 3 việc của R8 khi bật `isMinifyEnabled = true`: cắt mã thừa từ điểm neo và luật `-keep`, rút gọn tên lớp, và ghi sổ đối chiếu `mapping.txt`.
- [ ] Phân biệt được vai trò của `Upload Key` (bạn giữ để tải `.aab` lên) và `App Signing Key` (Google giữ để ký `Split APKs` giao cho máy người dùng).
- [ ] Chỉ ra được nguyên nhân lỗi mất dữ liệu JSON do `R8` đổi tên lớp dùng reflection và lỗi quên đăng ký `SHA-1` của `App Signing Key`.
- [ ] Viết lại được khuôn mẫu chuẩn từ trí nhớ.

## Tôi đã gặp nó ở đâu
