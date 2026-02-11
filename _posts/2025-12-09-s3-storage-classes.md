---
layout: post
title: "11. Các Lớp Lưu trữ trong S3"
title2: "Các Lớp Lưu trữ trong S3"
date: 2025-12-09
permalink: /2025/12/09/s3-storage-classes
categories: [S3]
tags: [S3]
img: /assets/11_s3_classes/lifecycle-transitions.png
summary: "Mỗi object lưu trên S3 được phân vào một lớp lưu trữ, với chi phí, độ trễ truy cập, và tính khả dụng khác nhau.
Tất cả các lớp lưu trữ của S3 đều có độ bền dữ liệu 11 số 9 (99.999999999%), tức về mặt xác suất, nếu lưu 1 tỷ object, trong 100 năm ước tính chỉ mất mát 1 object. 
Việc lựa chọn lớp lưu trữ nào tuỳ thuộc vào tần suất truy cập dữ liệu, các yêu cầu tối ưu chi phí hay độ trễ. Cùng tìm hiểu kỹ hơn dưới đây."
---

Mỗi object lưu trên S3 được phân vào một lớp lưu trữ, với chi phí, độ trễ truy cập, và tính khả dụng khác nhau.
Tất cả các lớp lưu trữ của S3 đều có [độ bền dữ liệu (*durability*)](https://docs.aws.amazon.com/whitepapers/latest/real-time-communication-on-aws/data-durability-and-ha-with-persistent-storage.html) 11 số 9 (99.999999999%), tức về mặt xác suất, nếu lưu 1 tỷ object, trong 100 năm ước tính chỉ mất mát 1 object. 
Việc lựa chọn lớp lưu trữ nào tuỳ thuộc vào tần suất truy cập dữ liệu, các yêu cầu tối ưu chi phí hay độ trễ. Cùng tìm hiểu kỹ hơn dưới đây. 
Trong bài có nhiều danh từ riêng, mình sẽ hạn chế dịch các từ này để bạn đọc quen mặt chữ và tiện tra cứu tài liệu.



## Trong bài này:

- [1. Với Dữ liệu Truy cập Thường xuyên](#frequent-access)
    - [1.1. S3 Standard](#s3-standard)
    - [1.2. S3 Express One Zone](#s3-express-one-zone)
- [2. Với Dữ liệu Truy cập Không Thường xuyên](#infrequent-access)
    - [2.1. S3 Standard-IA](#s3-standard-ia)
    - [2.2. S3 One Zone-IA](#s3-one-zone-ia)
- [3. Với Dữ liệu Cất trữ](#archive-data)
    - [3.1. S3 Glacier](#s3-glacier)
        - [3.1.1. Glacier Instant Retrieval](#glacier-instant)
        - [3.1.2. Glacier Flexible Retrieval](#glacier-flexible)
    - [3.2. S3 Glacier Deep Archive](#s3-glacier-deep-archive)
- [4. S3 Intelligence-Tiering](#s3-intelligent-tiering)
- [5. So sánh Các Lớp Lưu trữ S3](#s3-storage-classes-comparison)
- [6. S3 Lifecycle](#s3-lifecycle)
- [Tài liệu tham khảo](#reference)



<a name = "frequent-access"></a>

## 1. Với Dữ liệu Truy cập Thường xuyên

Nếu truy cập dữ liệu thường xuyên (nhiều lần trong tháng), hay với ứng dụng cần độ trễ thấp, có thể chọn hai lớp lưu trữ sau:

<a name = "s3-standard"></a>

### 1.1. S3 Standard

Đây là lớp lưu trữ **mặc định** khi tải dữ liệu lên S3. 
Object được sao lưu trên tất cả AZ của Region (ít nhất 3 AZ), nên có khả năng phục hồi [Region Resilience](/2025/11/12/aws_infrastructure#resilience).

S3 Standard tính phí lưu trữ theo GB mỗi tháng, phí di chuyển dữ liệu **RA** khỏi S3 (theo GB), cỡ vài cent/GB tuỳ Region, kể cả ra ngoài Internet hoặc ra AWS Region khác. Việc tải dữ liệu **vào** S3 là miễn phí.
Ngoài ra còn tính phí theo mỗi 1000 yêu cầu. 

Ví dụ, tại thời điểm viết bài, ở Region `us-east-1`:
- Chi phí lưu trữ trung bình **$0.022** mỗi GB mỗi tháng.
- Mỗi 1000 yêu cầu PUT, COPY, POST, hoặc LIST tính phí **$0.005**. GET, SELECT là **$0.0004** mỗi 1000 yêu cầu.

S3 Standard không yêu cầu thời gian lưu trữ tối thiểu, và không có kích thước object tối thiểu.
Độ trễ truy cập dữ liệu cỡ 50 - 200 ms.


Ngoài việc phù hợp với dữ liệu truy cập thường xuyên, nên sử dụng S3 Standard cho **dữ liệu quan trọng**, không thể tái tạo. 


<a name = "s3-express-one-zone"></a>

### 1.2. S3 Express One Zone

Lớp này phù hợp cho ứng dụng cần độ trễ siêu thấp (dưới 10 ms), như AI, Data Streaming, HPC (High Performance Computing).
S3 Express One Zone có tốc độ truy xuất dữ liệu nhanh hơn S3 Standard cỡ 10 lần. 
Hạn chế là dữ liệu chỉ được lưu trên 1 AZ tuỳ chọn, nên chỉ có khả năng phục hồi [AZ Resilience](/2025/11/12/aws_infrastructure#resilience).

Về chi phí ở `us-east-1`, tại thời điểm viết bài:
- Chi phí lưu trữ là **$0.11** mỗi GB mỗi tháng, gấp gần 5 lần S3 Standard. S3 Express One Zone cũng tính phí tải lên dữ liệu, **$0.0032** mỗi GB, thay vì miễn phí.
- Bù lại, mỗi 1000 yêu cầu PUT, COPY, POST, hoặc LIST chỉ tính phí **$0.00113**, rẻ hơn ~4.5 lần . GET, SELECT là **$0.00003** mỗi 1000 yêu cầu, rẻ hơn ~13 lần. Đây là ưu điểm khi làm việc với các ứng dụng trên.
- Ngoài ra còn có phí *truy xuất* dữ liệu (*data retrieval*) khi cần truy cập, **$0.0006** mỗi GB.


<a name = "infrequent-access"></a>

## 2. Với Dữ liệu Truy cập Không Thường xuyên


<a name = "s3-standard-ia"></a>

### 2.1. S3 Standard-IA

Viết tắt của S3 Standard-Infrequent Access. Các tính chất lưu trữ hầu hết đều giống [S3 Standard](#s3-standard), cũng có độ bền dữ liệu 11 số 9, dữ liệu cũng được sao lưu trên tất cả AZ nên có khả năng phục hồi [Region Resilience](/2025/11/12/aws_infrastructure#resilience).

Điểm khác biệt là ở **chi phí**, ví dụ cũng tại `us-east-1`:

- Chi phí lưu trữ là **$0.0125** mỗi GB mỗi tháng, thấp hơn cỡ 45% so với S3 Standard.
- Mỗi 1000 yêu cầu PUT, COPY, POST, hoặc LIST tính phí **$0.01**, gấp đôi S3 Standard. GET, SELECT là **$0.001** mỗi 1000 yêu cầu, cao hơn 2.5 lần.
- Phí truy xuất dữ liệu **$0.01** mỗi GB.

Rõ ràng, như tên gọi, lớp lưu trữ này được thiết kế cho dữ liệu truy cập không thường xuyên, sẽ tiết kiệm được nhiều hơn so với S3 Standard với dữ liệu lớn. Nhưng nếu cần truy cập nhiều, chi phí phát sinh thêm có thể vượt quá phần tiết kiệm được này.


Hai điểm cần lưu ý nữa là S3 Standard-IA tính phí lưu trữ tối thiểu 30 ngày, kích thước tối thiểu 128 KB. Nếu object lưu trên S3 Standard-IA dưới 30 ngày rồi bị xoá, ghi đè hoặc chuyển sang lưu trữ ở lớp khác sẽ vẫn bị tính phí như 30 ngày, và object dưới 128 KB vẫn bị tính phí như object 128 KB.

Kết luận, hãy dùng S3 Standard-IA nếu cần **lưu trữ lâu**, **quan trọng** (cần sao lưu trên tất cả AZ) nhưng **không truy cập nhiều**.

<a name = "s3-one-zone-ia"></a>

### 2.2. S3 One Zone-IA

Đây là một lựa chọn **rẻ** hơn S3 Standard-IA, cũng dùng cho dữ liệu cần **lưu trữ lâu**, **không truy cập nhiều**, nhưng **không quan trọng** và có thể dễ dàng tái tạo hoặc thay thế. Do lớp lưu trữ này chỉ lưu dữ liệu trên 1 AZ, nên chỉ có khả năng phục hồi [AZ Resilience](/2025/11/12/aws_infrastructure#resilience).

Tại `us-east-1` ở thời điểm viết bài, chi phí lưu trữ trên S3 One Zone-IA là **$0.01** mỗi GB mỗi tháng, rẻ hơn 20% so với S3 Standard-IA.

Còn lại các chi phí khác như PUT, GET, hay phí truy xuất dữ liệu đều bằng S3 Standard-IA. Phí lưu trữ tối thiểu cũng được tính là cho 30 ngày, và kích thước tính phí tối thiểu cũng là 128 KB.




<a name = "archive-data"></a>

## 3. Với Dữ liệu Cất trữ

Thuật ngữ là **Archived Data**, đây là loại dữ liệu gần như không được sử dụng, nhưng vẫn cần phải lưu lại khi có yêu cầu đặc biệt. 
Ví dụ, doanh nghiệp cần lưu trữ hồ sơ, báo cáo tài chính, các tài liệu pháp lý trong quá khứ. 
Dữ liệu kiểu này ưu tiên chi phí lưu trữ thấp hơn là tốc độ truy xuất, do đó thường được *lưu trữ lạnh* (trong *cold storage*). 
S3 cung cấp 2 lớp lưu trữ lạnh: **Glacier** và **Glacier Deep Archive**.


<a name = "s3-glacier"></a>

### 3.1. S3 Glacier




<a name = "glacier-instant"></a>

#### 3.1.1. Glacier Instant Retrieval

Lớp lưu trữ này khá giống S3 Standard-IA, dữ liệu cũng được sao lưu trên 3 AZ, vẫn có độ trễ truy xuất cỡ chục-trăm milisecond (do đó trong tên có "Instant").
Đây có thể hiểu là một lớp đệm giữa các lớp lưu trữ nóng và lạnh, phù hợp với dữ liệu truy cập ít (cỡ 1 lần/quý), nhưng khi cần vẫn có thể sẵn sàng ngay.


Chi phí so với S3 Standard-IA, ở `us-east-1` thời điểm viết bài:
- Chi phí lưu trữ là **$0.004** mỗi GB mỗi tháng, rẻ hơn S3 Standard-IA hơn 3 lần.
- Mỗi 1000 yêu cầu PUT, COPY, POST, hoặc LIST tính phí **$0.02**, cao gấp đôi S3 Standard-IA. GET, SELECT là **$0.01** mỗi 1000 yêu cầu, gấp 10 lần.
- Phí truy xuất dữ liệu **$0.03** mỗi GB, gấp 3 lần.


S3 Glacier Instant Retrieval tính phí lưu trữ tối thiểu 90 ngày, kích thước tối thiểu 128 KB.


<a name = "glacier-flexible"></a>

#### 3.1.2. Glacier Flexible Retrieval

Trước đây S3 chỉ có Glacier, không có Glacier Instant, cuối năm 2021 khi thêm vào thì Glacier được đổi tên thành Glacier Flexible.
Tiếp tục xu hướng càng ít truy cập thì phí lưu trữ càng rẻ, nhưng phí truy xuất càng đắt, chi phí của Glacier Flexible tại thời điểm viết bài ở `us-east-1` như sau:

- Phí lưu trữ là **$0.0036** mỗi GB mỗi tháng, rẻ hơn 10% so với Glacier Instant.
- Mỗi 1000 yêu cầu PUT, COPY, POST, hoặc LIST tính phí **$0.03**, cao hơn 50%. GET, SELECT là **$0.0004** mỗi 1000 yêu cầu, rẻ hơn 25 lần. Việc Glacier Instant đắt hơn rất nhiều ở điểm này vì nó hỗ trợ truy xuất dữ liệu gần như **ngay lập tức** (*instant*).

Phí lưu trữ tính tối thiểu cũng cho 90 ngày. Glacier Flexible không yêu cầu kích thước tính phí tối thiểu, nhưng AWS tính thêm phí lưu 40 KB metadata cho mỗi object.

So với Glacier Instant, đây mới thật sự là lớp lưu trữ **lạnh**. Dữ liệu không thể sẵn sàng ngay khi được yêu cầu, mà sẽ tốn thời gian truy xuất. Có 3 lựa chọn truy xuất như sau:
- *Expedited*: 1-5 phút. Phí truy xuất **$0.03** mỗi GB.
- *Standard*: 3-5 giờ. Phí truy xuất **$0.01** mỗi GB.
- *Bulk*: truy xuất theo lô, 5-12 giờ. **Miễn phí** truy xuất.

Dữ liệu khi truy xuất sẽ được tạm thời lưu ở S3 Standard-IA.

Lớp này phù hợp cho dữ liệu cất trữ lâu, hiếm khi cần truy cập (cỡ 1 lần/năm hoặc ít hơn).


<a name = "s3-glacier-deep-archive"></a>

### 3.2. S3 Glacier Deep Archive



Đây là lựa chọn lưu trữ **rẻ nhất**. Tại thời điểm viết bài ở `us-east-1`:

- Phí lưu trữ là **$0.00099** mỗi GB mỗi tháng, rẻ hơn ~3.6 lần so với Glacier Flexible.
- Mỗi 1000 yêu cầu PUT, COPY, POST, hoặc LIST tính phí **$0.05**, cao hơn 66%. GET, SELECT là **$0.0004** mỗi 1000 yêu cầu, bằng Glacier Flexible.

Phí lưu trữ tính tối thiểu cho 180 ngày. Glacier Deep Archive không yêu cầu kích thước tính phí tối thiểu, nhưng AWS tính thêm phí lưu 40 KB metadata cho mỗi object.

Có 2 lựa chọn truy xuất dữ liệu như sau:
- *Standard*: 12 giờ. Phí truy xuất **$0.02** mỗi GB.
- *Bulk*: 48 giờ. Phí truy xuất **$0.0025** mỗi GB.



S3 Glacier Deep Archive rẻ nhất, nhưng thời gian truy xuất cũng chậm nhất, do đó tần suất truy cập cũng nên thấp nhất (dưới 1 lần/năm).


<a name = "s3-intelligent-tiering"></a>

## 4. S3 Intelligence-Tiering

Với dữ liệu có tần suất truy cập không rõ, thay đổi liên tục hoặc không dự đoán được, AWS cung cấp lớp lưu trữ này, tự động phân tích hành vi và lưu object vào lớp phù hợp để tối ưu chi phí. Có 5 tier:

- *Frequent Access Tier*: phí lưu trữ và hiệu năng giống S3 Standard. 
- *Infrequent Access Tier*: giống S3 Standard-IA.
- *Archive Instant Access Tier*: giống S3 Glacier Instant.
- *Archive Access Tier*: giống S3 Glacier Flexible.
- *Deep Archive Access Tier*: giống S3 Glacier Deep Archive.

Khi tải object lên S3 vào lớp này, object sẽ được đặt vào Frequent Access Tier (S3 Standard). S3 Intelligence-Tiering tự động theo dõi tần suất truy cập từng object. Object không được truy cập trong 30 ngày liên tục sẽ được chuyển vào Infrequent Access Tier (S3 Standard-IA) để giảm chi phí. 

Nếu tiếp tục không được truy cập trong 60 ngày liên tục tiếp theo (tổng 90 ngày), object sẽ được chuyển vào Archive Instant Access Tier (Glacier Instant), chi phí tiếp tục rẻ hơn nhưng vẫn có thể truy xuất object ngay lập tức. Còn nếu được truy cập trong 90 ngày này, object sẽ được chuyển lại về Frequent Access Tier.

Hai tier cuối là tuỳ chọn, với dữ liệu không yêu cầu phải sẵn sàng ngay khi truy xuất. Nếu object không được truy cập trong 90 ngày, nó có thể được đưa vào Archive Access Tier (Glacier Flexible). 
Sau 180 ngày không được truy cập, object có thể được đưa vào Deep Archive Access Tier (Glacier Deep Archive).

Chi phí lưu trữ ở từng tier giống chi phí ở các lớp tương ứng. Ngoài ra, S3 Intelligence-Tiering tính thêm phí theo dõi và tự động hoá, **$0.0025** mỗi 1000 object, tại `us-east-1` ở thời điểm viết bài.




<a name = "s3-storage-classes-comparison"></a>

## 5. So sánh Các Lớp Lưu trữ S3

| Lớp | Thiết kế cho | Phí lưu trữ (/GB/tháng) | Tính khả dụng (thiết kế) | Số&nbsp;AZ&nbsp; | Thời gian tính phí tối thiểu | Kích thước tính phí tối thiểu | Khác |
|:--|:--|:--|:--|:--|:--|:--|:--|
| **S3 Standard** | Dữ liệu truy cập thường xuyên (nhiều lần/tháng), độ trễ millisecond | $0.022 | 99.99% | >= 3 | Không | Không | Không |
| **S3 Express One Zone** | Ứng dụng cần độ trễ cực thấp, cỡ single-digit millisecond | $0.11 | 99.95% | 1 | Không | Không | Không |
| **S3 Standard-IA** | Dữ liệu lưu trữ lâu, truy cập ít (1 lần/tháng), độ trễ millisecond | $0.0125 | 99.9% | >= 3 | 30 ngày | 128 KB | Có phí truy xuất object theo GB |
| **S3 One Zone-IA** | Dữ liệu có thể tái tạo, truy cập ít (1 lần/tháng), độ trễ millisecond | $0.01 | 99.5% | 1 | 30 ngày | 128 KB | Có phí truy xuất object theo GB. |
| **S3 Glacier Instant Retrieval** | Dữ liệu cất trữ lâu, truy cập cỡ 1 lần/quý, độ trễ millisecond | $0.004 | 99.9% | >= 3 | 90 ngày | 128 KB | Có phí truy xuất object theo GB |
| **S3 Glacier Flexible Retrieval** | Dữ liệu cất trữ lâu, truy cập 1 lần/năm | $0.0036 | 99.99% (sau khi truy xuất) | >= 3 | 90 ngày | Không, nhưng thêm 40 KB cho mỗi object | Có phí truy xuất object theo GB. Thời gian truy xuất cỡ phút đến giờ |
| **S3 Glacier Deep Archive** | Dữ liệu cất trữ lâu, truy cập <1 lần/năm | $0.00099 | 99.99% (sau khi truy xuất) | >= 3 | 180 ngày | Không, nhưng thêm 40 KB cho mỗi object | Có phí truy xuất object theo GB. Thời gian truy xuất hàng giờ |
| **S3 Intelligent-Tiering** | Dữ liệu có tần suất truy cập không rõ, thay đổi liên tục hoặc không dự đoán được | Tùy tier (tham khảo giá S3 Standard / Standard‑IA / Glacier) | 99.9% | >= 3 | Không | Không | Có phí theo dõi và tự động hoá cho mỗi object. Không có phí truy xuất object. Object <128KB luôn ở S3 Standard |



<a name = "s3-lifecycle"></a>

## 6. S3 Lifecycle

Tính năng này giúp quản lý chi phí lưu trữ object hiệu quả, thông qua:
- **Transition**: giúp tự động chuyển object sang các lớp lưu trữ khác nhau khi điều kiện thoả mãn. Ví dụ, có thể đặt luật chuyển object từ S3 Standard qua S3 Standard-IA 30 ngày kể từ khi object được tạo, hoặc chuyển hẳn đến S3 Glacier Flexible Retrieval sau 1 năm kể từ khi tạo. 

- **Expiration**: xoá các object quá hạn. Ví dụ, nếu cần tuân thủ quy định lưu trữ giữ liệu tối thiểu 3 năm, bạn có thể đặt luật xoá đi sau thời gian đó.

Có thể cấu hình Lifecycle cho **bucket** hoặc các **nhóm object**.
Dưới đây là mô hình chuyển đổi giữa các lớp lưu trữ mà người dùng có thể cân nhắc khi cấu hình Lifecycle. 

<p>
<image src="/assets/11_s3_classes/lifecycle-transitions.png" alt="Chuyển đổi giữa các lớp lưu trữ dùng Lifecycle" style="max-width:90%;height:auto;display:block;margin:0 auto;"/>
</p>

*Nguồn ảnh: [AWS](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-transition-general-considerations.html)*

Một số lưu ý khi cấu hình chuyển đổi:
- Object ở S3 Standard phải được lưu **tối thiểu 30 ngày** trước khi chuyển qua S3 Standard-IA hoặc S3 One Zone-IA.
- Object nhỏ hơn 128 KB mặc định không thể chuyển qua lớp lưu trữ nào. Người dùng có thể thêm [bộ lọc kích thước](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html#intro-lifecycle-rules-filter) vào luật chuyển đổi nếu muốn, nhưng AWS không khuyến khích việc này, vì có phí khi yêu cầu chuyển đổi mỗi object, và với các object nhỏ, phí này có thể vượt quá phần có thể tiết kiệm được khi chuyển đổi.
- Từ S3 Glacier Deep Archive không thể chuyển qua các lớp khác nếu dùng Lifecycle (trong hình thể hiện rõ đây là điểm cuối). Người dùng cần thực hiện truy vấn thủ công rồi sao chép qua các lớp khác.

Bạn đọc có thể tìm hiểu thêm [tại đây](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-transition-general-considerations.html).


<a name = "reference"></a>

## Tài liệu tham khảo

1. [Các lớp lưu trữ S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)
2. [S3 Durability](https://docs.aws.amazon.com/whitepapers/latest/real-time-communication-on-aws/data-durability-and-ha-with-persistent-storage.html)
3. [Chi tiết phí lưu trữ trên S3](https://aws.amazon.com/s3/pricing/)
4. [Object Lifecycle](http://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
5. [Chuyển vị object dùng Lifecycle](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-transition-general-considerations.html)
