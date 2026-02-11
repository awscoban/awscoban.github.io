---
layout: post
title: "5. Thiết kế VPC"
title2: "Thiết kế VPC"
date: 2025-11-20
permalink: /2025/11/20/design-vpc
categories: [VPC, Networking]
tags: [VPC, Networking]
img: /assets/5_design_vpc/multi-tier.png
summary: "Khi tạo VPC, ta cần chỉ rõ IPv4 CIDR. 
IPv4 chỉ gồm 32 bit, tức tổng cộng \\(2^{32}\\) ~ hơn 4 tỷ địa chỉ IPv4, rõ ràng là không đủ nếu tất cả các thiết bị kết nối trên toàn cầu đều cần có IP duy nhất. 
Thực tế, không cần cấp phát IP duy nhất cho tất cả thiết bị. 
Trong một tổ chức, rất nhiều thiết bị chỉ cần kết nối nội bộ, không cần kết nối với thiết bị trong tổ chức khác hoặc với Internet, nên chỉ cần IP là duy nhất trong phạm vi tổ chức là đủ."
---

VPC Mặc định được AWS cấu hình sẵn để người dùng nhanh chóng bắt đầu triển khai ứng dụng, tuy nhiên các subnet trong đó đều là public, không phù hợp với nhiều ứng dụng phân tầng cần bảo mật dữ liệu và logic tính toán. Trong các trường hợp này, tự thiết kế VPC là yêu cầu bắt buộc.

## Trong bài này:

- [1. Chọn CIDR](#ip-design)
- [2. Thiết kế VPC cho ứng dụng đa tầng](#multi-tier-design)
    - [2.1. Cấu trúc đa tầng](#multi-tier)
    - [2.2. Cấu trúc đa vùng, đa giai đoạn](#multi-region-acc)
    - [2.3. Ví dụ](#example)
- [Tài liệu tham khảo](#reference)






<a name = "ip-design"></a>

## 1. Chọn CIDR

Khi tạo VPC, ta cần chỉ rõ IPv4 CIDR. 
IPv4 chỉ gồm 32 bit, tức tổng cộng \\(2^{32}\\) ~ hơn 4 tỷ địa chỉ IPv4, rõ ràng là không đủ nếu tất cả các thiết bị kết nối trên toàn cầu đều cần có IP duy nhất. 
Thực tế, không cần cấp phát IP duy nhất cho tất cả thiết bị. 
Trong một tổ chức, rất nhiều thiết bị chỉ cần kết nối nội bộ, **không cần kết nối với thiết bị trong tổ chức khác hoặc với Internet**, nên chỉ cần IP là duy nhất **trong phạm vi tổ chức** là đủ.
Do đó, các tổ chức có thể sử dụng chung IP, miễn là không kết nối với nhau. 
Nghe rất quen, phải không? 
VPC chính là một ví dụ khác (xem lại [bài trước](/2025/11/13/vpc#private-network)): 2 VPC cô lập (không kết nối với nhau) có thể có CIDR trùng nhau, tức chung nhiều IP. 


Ý tưởng này được đề xuất trong [RFC 1918](http://www.faqs.org/rfcs/rfc1918.html) để giải quyết vấn đề cạn kiệt địa chỉ IPv4.
Họ đề xuất dành 3 khối IP cho nhu cầu kết nối nội bộ (*private address space*), không dùng làm IP công cộng:

- `10.0.0.0/8`: từ `10.0.0.0` đến `10.255.255.255`
- `172.16.0.0/12`: từ `172.16.0.0` đến `172.31.255.255`
- `192.168.0.0/16`: từ `192.168.0.0` đến `192.168.255.255`

Tổng cộng, \\(2^{24} + 2^{20} + 2^{16}\\) tức gần 18 triệu địa chỉ IP được dành riêng cho kết nối nội bộ. 
AWS cũng khuyến nghị chọn CIDR nằm trong 3 dải địa chỉ này khi tạo VPC, cũng như một vài dải địa chỉ cần tránh, xem thêm [tại đây](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html).  


Kích thước VPC (và subnet) tối thiểu là `/28` (16 IP), tối đa là `/16` (65,536 IP), sau đó trừ đi tổng số IP không thể sử dụng trong mỗi CIDR (mỗi subnet trong VPC có 5 IP không thể sử dụng, xem lại [bài trước](/2025/11/13/vpc#vpc-subnet)). 

Dưới đây là bảng tổng hợp kích thước VPC và subnet phổ biến:

| Kích thước VPC | Số subnet trong VPC | Kích thước Subnet | Số IP khả dụng mỗi subnet | Tổng số IP khả dụng trong VPC |
|:---------------:|:---------------:|:---------------:|:---------------:|:---------------:|
| /24            | 8                   | /27               | 27                | 216                 |
| /21            | 8                   | /24               | 251               | 2,008                |
| /19            | 8                   | /22               | 1,019              | 8,152                |
| /18            | 8                   | /21               | 2,043              | 16,344               |
| /16            | 16                  | /20               | 4,091              | 65,456               |


**Diễn giải**: giả sử ứng dụng có quy mô nhỏ, chỉ cần tổng cộng dưới 200 IP để cấp phát (cho các EC2 instance, DB instance, Lambda function, v.v.), có thể chọn kích thước VPC `/24` ở dòng đầu tiên trong bảng. 
Có thể chia thành 8 subnet kích thước `/27` (vì \\(2^{32-24} / 2^{32-27} = 8\\)), mỗi subnet sẽ có \\(2^{32-27} - 5 = 27\\) IP khả dụng, như vậy tổng số IP khả dụng trong VPC là \\( 27 * 8 = 216 \\), đủ dùng cho ứng dụng. 

Tất nhiên, đây chỉ là thiết kế thô, chưa cân nhắc đến vấn đề mở rộng hay [đáp ứng HA, FT](/2025/11/12/aws_infrastructure#ha-ft-dr).
Và do việc tạo VPC là miễn phí, bất kể kích thước (bạn chỉ bị tính phí cho các tài nguyên chạy trên VPC), nên phương án tốt nhất là chọn kích thước VPC tối đa (`/16`), tránh các vấn đề về sau khi cần mở rộng.



<a name = "multi-tier-design"></a>

## 2. Thiết kế VPC cho ứng dụng đa tầng


Hãy đi vào một bài toán cụ thể, ta cần thiết kế VPC cho một ứng dụng đa tầng (*multi-tier*) trên AWS, hay được sử dụng trong thực tế cũng như trong các câu hỏi thi chứng chỉ AWS, gồm 3 tầng:

- **Web Tier** (hay còn gọi Presentation Tier): giao tiếp với người dùng, hiển thị dữ liệu, thu thập và kiểm tra đầu vào.
- **Application Tier** (hay Logic Tier): xử lý logic chính của ứng dụng.
- **Database Tier**: lưu trữ dữ liệu.


<a name="multi-tier"></a>

### 2.1. Cấu trúc đa tầng

Khi thiết kế VPC cho ứng dụng này, đầu tiên ta cân nhắc sẽ triển khai trên bao nhiêu AZ (để [đáp ứng HA, FT](/2025/11/12/aws_infrastructure#ha-ft-dr)).
Một VPC ứng với một Region, một Region có ít nhất 3 AZ. Và 3 AZ là đủ để ứng dụng đạt [Region Resilience](/2025/11/12/aws_infrastructure#resilience). 

Trong một AZ, mỗi tầng ứng dụng nên được đặt trong một subnet, cùng với một subnet dự phòng, tổng cộng ta cần 4 subnet trong mỗi AZ, tức 12 subnet trong 3 AZ.

Như vậy, ta có thiết kế như sau:

<image src="/assets/5_design_vpc/multi-tier.png" alt="Multi-Tier VPC design" style="max-width:90%;height:auto;display:block;margin:0 auto;"/>


<p></p>

Giả sử VPC có kích thước `/16`, ta cần 12 subnet. Để tiện tính toán, ta có thể chia VPC thành 16 subnet kích thước `/20` (luỹ thừa của 2 gần 12 nhất làm tròn lên là 16 = \\(2^4\\)), dùng 12 subnet chia cho các tầng ứng dụng ở 3 AZ.

Ở đây, tầng Web sẽ đặt trong *public subnet* (màu xanh lá trong hình), do tính chất cần tương tác với người dùng. Tầng Application và Database nên đặt trong *private subnet* (màu thiên thanh trong hình) để bảo mật, do chúng thường không cần kết nối bên ngoài. 
Các subnet trong cùng VPC mặc định có thể giao tiếp với nhau, nên giao tiếp giữa các tầng ứng dụng vẫn được đảm bảo. Bạn đọc có thể xem lại định nghĩa public và private subnet [trong bài trước](/2025/11/13/vpc#vpc-subnet).

<a name="multi-region-acc"></a>

### 2.2. Cấu trúc đa vùng, đa giai đoạn


Nhưng, vẫn chưa xong! Thiết kế như vậy chỉ giúp ứng dụng đạt khả năng phục hồi [Region Resilience](/2025/11/12/aws_infrastructure#resilience).
Việc này, dù trong đa số trường hợp là đủ đáp ứng yêu cầu kỹ thuật, nhưng với các ứng dụng lớn có quy mô toàn cầu thì cần [Global Resilience](/2025/11/12/aws_infrastructure#resilience). 
Khi đó, ta cần triển khai ứng dụng trên ít nhất một Region khác, tức là cần tạo một VPC với **CIDR không trùng lặp** với các VPC ở Region khác của ứng dụng, hay tốt hơn hết là không trùng với CIDR nào của doanh nghiệp.
Dễ hiểu, vì theo nguyên tắc ta cần IP duy nhất trong phạm vi nội bộ tổ chức. 

Thêm nữa, trong dự án thường sẽ theo quy trình Development, Staging, Production. 
Ta có thể thiết kế tiếp VPC cho từng giai đoạn, đơn giản là tạo các tài khoản AWS tương ứng rồi "nhân bản có điều chỉnh" khối VPC đa vùng ở trên cho các tài khoản. 
Việc "điều chỉnh" ở đây là cho CIDR của từng VPC.
Một lần nữa, nguyên tắc cần ghi nhớ là **CIDR của các VPC không trùng lặp**! 
Việc này sẽ hạn chế rất nhiều vấn đề phát sinh về sau nếu cùng IP được cấp phát cho nhiều thiết bị trong cùng dự án hay cùng tổ chức.

Hình dưới đây tổng quát hoá việc thiết kế VPC cho ứng dụng đa tầng, đa vùng, và đa giai đoạn (vì lặp lại nên mình chỉ minh hoạ Development và Production). 

<image src="/assets/5_design_vpc/multi-acc-region.png" alt="Multi-Tier-Account-Region VPC design" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>

<p></p>
Đương nhiên, thực tế chỉ môi trường Production là cần mức chịu lỗi cao nhất, nên nếu cần tiết kiệm chi phí, chỉ cần thiết kế VPC đa vùng cho môi trường này. Development và Staging có thể cân nhắc sử dụng VPC đơn vùng (thậm chí đơn AZ).

<a name="example"></a>

### 2.3. Ví dụ

Dưới đây là ví dụ CIDR cụ thể cho các VPC của ứng dụng, giả sử triển khai 3 giai đoạn, 2 Region, 3 AZ, 3 tầng + 1 dự phòng. 
Ở đây mình chọn dải IP `10.x.x.x`.

| Giai đoạn   | Region   | VPC CIDR       |
|-------------|----------|----------------|
| Development | Region 1 | 10.0.0.0/16    |
| Development | Region 2 | 10.1.0.0/16    |
| Staging     | Region 1 | 10.2.0.0/16    |
| Staging     | Region 2 | 10.3.0.0/16    |
| Production  | Region 1 | 10.4.0.0/16    |
| Production  | Region 2 | 10.5.0.0/16    |

<p></p>

Trong VPC `10.0.0.0/16`, ta chia thành 12 subnet kích thước `/20` theo AZ và tầng như sau:


**AZ 1:**

| Web AZ 1 | App AZ 1 | DB AZ 1 | Spare AZ 1 |
|----------|----------|---------|------------|
| 10.0.0.0/20 | 10.0.16.0/20 | 10.0.32.0/20 | 10.0.48.0/20 |

<p></p>

**AZ 2:**

| Web AZ 2 | App AZ 2 | DB AZ 2 | Spare AZ 2 |
|----------|----------|---------|------------|
| 10.0.64.0/20 | 10.0.80.0/20 | 10.0.96.0/20 | 10.0.112.0/20 |

<p></p>

**AZ 3:**

| Web AZ 3 | App AZ 3 | DB AZ 3 | Spare AZ 3 |
|----------|----------|---------|------------|
| 10.0.128.0/20 | 10.0.144.0/20 | 10.0.160.0/20 | 10.0.176.0/20 |

<p></p>
Và làm tương tự cho các VPC khác.





<a name = "reference"></a>

## Tài liệu tham khảo

1. [Cân nhắc VPC CIDR](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)
2. [RFC 1918](http://www.faqs.org/rfcs/rfc1918.html)
3. [Thiết kế AWS VPC](https://aws.amazon.com/answers/networking/aws-single-vpc-design/)


Vậy là ta đã biết cách tự thiết kế VPC cho ứng dụng cụ thể. Tiếp theo, hãy tìm hiểu về định tuyến trong VPC, cùng hai dịch vụ quan trọng: Internet Gateway và NAT Gateway.
