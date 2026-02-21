---
layout: post
title: "34. CloudFront"
title2: "CloudFront"
date: 2026-02-20
permalink: /2026/02/20/cloudfront
categories: [CloudFront]
tags: [CloudFront]
img: /assets/34_cloudfront/cloudfront.png
summary: "Hẳn bạn đọc đã từng nghe khái niệm CDN (Content Delivery Network), một mạng lưới máy chủ đặt tại nhiều vị trí địa lý khác nhau, giúp lưu trữ bản sao dữ liệu (hình ảnh, video, tài liệu, v.v.), giúp giảm độ trễ khi người dùng truy xuất dữ liệu, khi chỉ cần phân phối từ máy chủ gần nhất. AWS cung cấp dịch vụ CDN, tên gọi CloudFront."
---

Hẳn bạn đọc đã từng nghe khái niệm CDN (Content Delivery Network), một mạng lưới máy chủ đặt tại nhiều vị trí địa lý khác nhau, giúp lưu trữ bản sao dữ liệu (hình ảnh, video, tài liệu, v.v.), giúp giảm độ trễ khi người dùng truy xuất dữ liệu, khi chỉ cần phân phối từ máy chủ gần nhất. AWS cung cấp dịch vụ CDN, tên gọi CloudFront.


## Trong bài này:

- [1. Cơ chế Hoạt động](#overview)
- [2. Các Khái niệm Chính](#key-concepts)
    - [2.1. Distribution](#distribution)
    - [2.2. Origin](#origin)
    - [2.3. Cache Behavior](#cache-behavior)
    - [2.4. Point of Presence](#pop)
- [3. Chủ động Xoá Bộ đệm](#invalidation)
- [4. Edge Function](#edge-function)
    - [4.1. Lambda@Edge](#lambda-edge)
    - [4.2. CloudFront Function](#cloudfront-function)
- [Tài liệu tham khảo](#reference)


<a name="overview"></a>

## 1. Cơ chế Hoạt động

Khi tích hợp với ứng dụng, CloudFront hoạt động như hai tầng bộ nhớ đệm (cache) giữa client và máy chủ gốc (_**origin** server_). Khi client yêu cầu tài nguyên (tài liệu, hình ảnh, video, v.v.):

<p>
<image src="/assets/34_cloudfront/cloudfront.png" alt="CloudFront" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>


1. DNS gửi yêu cầu đến **Edge Location** có độ trễ tới client thấp nhất (thường là gần nhất). Đây là tầng bộ nhớ đệm đầu tiên. Nếu tài nguyên có tại Edge Location, CloudFront trả ngay về cho client.
2. Nếu tài nguyên không có tại Edge Location, yêu cầu được chuyển đến **Regional Edge Cache** (tầng bộ nhớ đệm thứ hai). Tương tự, ếu tài nguyên có tại đây (*cache hit*), CloudFront trả về Edge Location và lưu lại tại đây, sau đó trả về cho client. Các yêu cầu sau cho cùng tài nguyên sẽ được phục vụ ngay từ Edge Location (nếu tài nguyên vẫn chưa hết hạn).
3. Nếu tài nguyên không có tại Regional Edge Cache, yêu cầu được chuyển đến máy chủ gốc. Tài nguyên được lấy từ origin server, trả về và lưu lại tại Regional Edge Cache và Edge Location, cuối cùng trả về cho client.



<a name = "key-concepts"></a>

## 2. Các Khái niệm Chính

<a name = "distribution"></a>

### 2.1. Distribution

Distribution là một cấu hình cụ thể người dùng tạo trong CloudFront, quy định cách phân phối nội dung:

- Dữ liệu gốc ở đâu ([origin](#origin)).
- Thiết lập bộ đệm ([cache behavior](#cache-behavior)).
- Vị trí [Edge Location](#edge-location).
- Các thiết lập bảo mật (sẽ được đề cập trong bài sau).
- v.v.

Khi tạo Distribution, CloudFront tự động cài đặt một số cấu hình cho Distribution, tuỳ theo loại Origin. Chi tiết [tại đây](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/template-preconfigured-origin-settings.html), bạn đọc có thể tham khảo để hiểu rõ hơn. Ví dụ, Origin là S3 Bucket sẽ có một số thiết lập sẵn như sau:

```yml
- Thiết lập Origin:
    - Origin Access Control: Yes
    - Add custom header: None
    - Enable Origin Shield: No
    - Connection attempts: 3

- Thiết lập bộ đệm:
    - Compress objects automatically: Yes
    - Viewer protocol policy: Redirect to HTTPS
    - Allowed HTTP method: GET, HEAD
    - Restrict viewer access: No
    - Cache policy: CachingOptimized
    - Origin request policy: None
    - Response header policy: None
    - Smooth Streaming: No
    - Field level encryption: No
    - Enable real-time access logs: No
    - Functions: No
```



<a name = "origin"></a>

### 2.2. Origin

Từ này mang nghĩa *nguồn gốc*. Trong CloudFront, đây là nơi lưu trữ dữ liệu gốc, có thể là:
- [S3 Bucket](/2025/11/30/s3-fundamental): đây là origin phổ biến nhất, do CloudFront hay CDN nói chung thường dùng để phân phối dữ liệu tĩnh (hình ảnh, video, tài liệu, v.v.). Dữ liệu kiểu này rất phù hợp lưu trữ trên S3.
- Máy chủ web: chạy trên EC2 hoặc on-premise.
- [Elastic Load Balancer](/2026/02/05/load-balancer)
- [API Gateway](/2026/01/28/api-gateway)
- AWS Elemental MediaPackage

**Origin Group**: là một nhóm Origin trong một Distribution, dùng để thiết lập dự phòng (*failover*). Khi tạo Origin Group, người dung chỉ định một Origin **chính** và một hoặc nhiều Origin **phụ**, cùng với **điều kiện chuyển đổi**, là các mã lỗi HTTP: 502, 503, 504, hoặc 404.

Nếu bạn đang ôn thi chứng chỉ AWS, lưu ý phần này. Để chuyển đổi dự phòng chỉ cần dùng Origin Group trong cùng Distribution, không cần tạo Distribution khác.



<a name = "cache-behavior"></a>

### 2.3. Cache Behavior

Đây là tập hợp các quy tắc xác định cách CloudFront xử lý yêu cầu từ client. 


#### **Path Pattern**

Là khuôn mẫu để xác định sẽ áp dụng Cache Behavior nào. Ví dụ, giả sử ta có 3 quy tắc Cache Behavior với 3 Path Pattern sau:
- `images/*.jpg`: áp dụng cho tất cả yêu cầu đến các tệp có đuôi `jpg` trong thư mục `images`.
- `images/*`: áp dụng cho tất cả yêu cầu đến thư mục `images`.
- `*`: áp dụng cho tất cả yêu cầu (đây là khuôn mẫu mặc định).

Client yêu cầu tài nguyên `images/photo.png`. CloudFront sẽ kiểm tra lần lượt các Path Pattern:
- `images/*.jpg`: không khớp (tệp không có đuôi `jpg`).
- `images/*`: khớp (tệp nằm trong thư mục `images`). Áp dụng Cache Behavior tương ứng (ví dụ như lấy từ Origin được chỉ định). 
- `*`: không kiểm tra nữa, vì đã khớp trước đó.

Lưu ý, khi khớp Path Pattern, CloudFront xử lý theo trình tự từ trên xuống dưới. Khi một yêu cầu khớp với một Path Pattern, CloudFront áp dụng Cache Behavior tương ứng và **dừng kiểm tra các quy tắc tiếp theo**. Cần lưu ý điều này để đặt thứ tự Path Pattern sao cho phù hợp.


#### **Origin và Origin Group**

Chỉ định Origin hoặc Origin Group mà CloudFront sẽ tương tác khi có yêu cầu khớp với Path Pattern.

#### **Viewer Protocol Policy**

Kiểm tra giao thức của yêu cầu, và quyết định có cho phép yêu cầu đó hay không. Có ba lựa chọn:
- *HTTP and HTTPS*: cho phép cả hai giao thức.
- *Redirect HTTP to HTTPS*: nếu yêu cầu dùng HTTP, CloudFront sẽ tự động chuyển hướng sang HTTPS.
- *HTTPS Only*: chỉ cho phép yêu cầu dùng HTTPS.


#### **Allowed HTTP Methods**

Giới hạn các hành động mà người dùng có thể thực hiện:

- `GET`, `HEAD`: Chỉ đọc dữ liệu, thường dùng cho các nội dung tĩnh như hình ảnh, video, tài liệu trong S3.

- `GET`, `HEAD`, `OPTIONS`, `PUT`, `POST`, `PATCH`, `DELETE`: Cho phép cả đọc, cập nhật và xoá dữ liệu, thường dùng cho Origin API Gateway.


####  **Restrict Viewer Access**

Nếu bật tính năng này, client phải có quyền truy cập hợp lệ (qua [Signed URL hoặc Signed Cookie](/2026/02/25/cloudfront-security#signed-url-cookie)) mới có thể lấy dữ liệu từ CloudFront. Phần này sẽ được đề cập kỹ hơn trong bài sau.


#### **Time to Live (TTL)**

Xác định thời gian dữ liệu được lưu trong bộ nhớ đệm tại [Point of Presence](#pop) và [Regional Edge Cache](#regional-edge-cache), trước khi hết hạn và cần lấy lại từ Origin. Có ba tham số:
- *Minimum TTL*: thời gian tối thiểu.
- *Maximum TTL*: thời gian tối đa (mặc định 31536000 giây tức 1 năm). Áp dụng khi Origin thêm HTTP Header `Cache-Control max-age`, `Cache-Control s-maxage`, hoặc `Expires` cho dữ liệu.
- *Default TTL*: thời gian mặc định (24 giờ). Áp dụng khi Origin **không thêm** 3 HTTP Header trên cho dữ liệu.




<a name = "pop"></a>

### 2.4. Point of Presence

Là mạng lưới máy chủ đặt tại các thành phố lớn trên toàn cầu. Đây chính là hai tầng bộ đệm đã đề cập ở phần [Cơ chế Hoạt động](#overview):
- **Edge Location**: tương tác trực tiếp với client.
- **Regional Edge Cache**: tầng bộ nhớ đệm thứ hai, nằm giữa Edge Location và Origin. Bản chất là một Edge Location có dung lượng lớn hơn. Nếu dữ liệu không có tại Edge Location, CloudFront sẽ kiểm tra tại Regional Edge Cache trước khi truy xuất từ Origin.

Một cách ví von dễ hiểu: 
- Origin như kho hàng tổng của doanh nghiệp.
- Regional Edge Cache như các đại lý bán buôn tại các khu vực.
- Edge Location như hệ thống cửa hàng bán lẻ khắp các đường phố.




<a name = "invalidation"></a>

## 3. Chủ động Xoá Bộ đệm

Thực tế, rất nhiều lúc khi Origin cập nhật dữ liệu mới, nhưng CloudFront vẫn dùng dữ liệu cũ để phục vụ client, do chưa hết hạn theo TTL. Để giải quyết vấn đề này, CloudFront cung cấp tính năng **Invalidation** (chủ động xoá bộ đệm). 

Khi tạo yêu cầu xoá bộ đệm, người dùng chỉ định các **Path** cần xoá khỏi bộ nhớ đệm. CloudFront sẽ xoá dữ liệu tương ứng tại tất cả Edge Location và Regional Edge Cache. Các yêu cầu tiếp theo cho các Path này sẽ được lấy từ Origin.

Người dùng có thể:
- Chỉ định chính xác tệp nào cần xoá, ví dụ: `/images/photo.jpg`
- Dùng wildcard `*` để xoá nhiều tệp cùng lúc, ví dụ: `/images/*` để xoá tất cả tệp trong thư mục `images`.
- Dùng `/*` để **xoá toàn bộ** bộ đệm trong Distribution. Cần cân nhắc khi sử dụng, vì không còn bộ đệm tức mọi yêu cầu sẽ được lấy trực tiếp từ Origin, tải lên Origin sẽ tăng đột biến.


Một giải pháp khác để giải quyết vấn đề dữ liệu cũ là thêm phiên bản (**version**) vào tên tài nguyên tại Origin. Ví dụ, thay vì cập nhật tệp `style.css`, ta có thể tạo tệp mới với tên `style_v2.css`, và cập nhật tham chiếu trong website. CloudFront sẽ coi đây là tài nguyên hoàn toàn mới, và lấy từ Origin mà không cần xoá bộ đệm.


<a name = "edge-function"></a>

## 4. Edge Function 

CloudFront hỗ trợ chạy mã nguồn để tuỳ chỉnh cách thức xử lý yêu cầu và phản hồi ngay tại Edge Location, giúp giảm độ trễ và tải cho Origin, khi một số xử lý có thể thực hiện ngay tại Edge Location. Mã nguồn này được gọi chung là **Edge Function**, gồm hai loại: Lambda@Edge và CloudFront Function.


<a name = "lambda-edge"></a>

### 4.1. Lambda@Edge

Đây là phiên bản mở rộng của [Lambda](/2026/01/18/lambda-fundamental), chạy NodeJS và Python, được [kích hoạt](/2026/01/18/lambda-fundamental#trigger) bởi 4 sự kiện sau:

- **Viewer Request**: sau khi CloudFront nhận yêu cầu từ client.
- **Origin Request**: trước khi CloudFront chuyển tiếp yêu cầu đến Origin (khi cache miss).
- **Origin Response**: sau khi CloudFront nhận được phản hồi từ Origin.
- **Viewer Response**: trước khi CloudFront trả phản hồi về cho client.

<p>
<image src="/assets/34_cloudfront/lambda-edge.png" alt="Lambda@Edge" style="max-width:90%;height:auto;display:block;margin:0 auto;"/>
</p>

Có thể dùng Lambda@Edge để:
- Thêm/cập nhật HTTP Header trong yêu cầu và phản hồi.
- Xác thực ngay tại Edge Location.
- Chuyển hướng yêu cầu.

Khác với Lambda Function thông thường, Lambda@Edge chỉ có thời gian thực thi tối đa 30 giây

<a name = "cloudfront-function"></a>

### 4.2. CloudFront Function

Là dạng Edge Function nằm trong CloudFront, khác với Lambda@Edge cần tích hợp với Lambda. CloudFront Function chỉ hỗ trợ JavaScript, và chỉ được kích hoạt bởi hai sự kiện:

- **Viewer Request**: sau khi CloudFront nhận yêu cầu từ client.
- **Viewer Response**: trước khi CloudFront trả phản hồi về cho client.

Cũng có thể dùng CloudFront Function để thêm/cập nhật HTTP Header, xác thực, chuyển hướng yêu cầu, tương tự Lambda@Edge. Tuy nhiên, CloudFront Function chỉ có gian thực thi tối đa dưới 1 mili giây, bộ nhớ và dung lượng lưu trữ cũng nhỏ hơn nhiều, nên chỉ phù hợp cho các tác vụ nhẹ.


Bảng dưới đây so sánh hai loại Edge Function tại thời điểm viết bài:

|  | CloudFront Function | Lambda@Edge |
|:---|:---|:----|
| **Ngôn ngữ** | JavaScript | Node.js và Python |
| **Sự kiện** | Viewer request<br>Viewer response | Viewer request<br>Viewer response<br>Origin request<br>Origin response |
| **Quy mô tối đa** | Hàng triệu request mỗi giây | 10000 request mỗi giây cho mỗi Region |
| **Thời gian thực thi** | Dưới 1 mili giây | 30 giây |
| **Bộ nhớ tối đa** | 2 MB | 128 MB cho viewer request/response<br>10 GB cho origin request/response |
| **Kích thước code tối đa (bao gồm thư viện)** | 10 KB | 50 MB |
| **Truy cập mạng** | Không | Có |
| **Truy cập File system** | Không | Có |



<a name = "reference"></a>

## Tài liệu tham khảo

1. [Cơ chế Hoạt động](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/HowCloudFrontWorks.html)
2. [Thiết lập Distribution](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-working-with.html)
3. [Một số thiết lập Distribution có sẵn](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/template-preconfigured-origin-settings.html)
4. [Cache Behavior](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesCacheBehavior.html#DownloadDistValuesTargetOriginId)
5. [Hệ thống Edge Location trên toàn cầu](https://aws.amazon.com/cloudfront/features/#Global_Edge_Network)
6. [Invalidation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/invalidation-specifying-objects.html)
7. [Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-edge-ways-to-use.html)
8. [CloudFront Function](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html)

Trong bài tiếp theo, ta sẽ tìm hiểu về bảo mật trong CloudFront.