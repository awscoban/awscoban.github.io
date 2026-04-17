---
layout: post
title: "44. Direct Connect"
title2: "Direct Connect"
date: 2026-04-18
permalink: /2026/04/18/direct-connect
categories: [Networking, Hybrid Environment]
tags: [Networking, Hybrid Environment]
img: /assets/44_dx/dx.png
summary: "Cho đến bài viết này, khi thảo luận về các dịch vụ AWS, ta luôn mặc định sử dụng kết nối qua Internet để truy cập vào các dịch vụ đó. Tuy nhiên, đối với một số tổ chức, việc sử dụng Internet để kết nối đến AWS có thể không đáp ứng được yêu cầu hiệu suất, độ trễ, hay bảo mật. Với những trường hợp này, giải pháp là Direct Connect (DX), dịch vụ giúp thiết lập kết nối mạng chuyên dụng từ hạ tầng nội bộ trực tiếp đến AWS, mà không đi qua Internet. Cùng tìm hiểu chi tiết hơn dưới đây."
---


Cho đến bài viết này, khi thảo luận về các dịch vụ AWS, ta luôn mặc định sử dụng kết nối qua Internet để truy cập vào các dịch vụ đó. Tuy nhiên, đối với một số tổ chức, việc sử dụng Internet để kết nối đến AWS có thể không đáp ứng được yêu cầu hiệu suất, độ trễ, hay bảo mật. Với những trường hợp này, giải pháp là Direct Connect (DX), dịch vụ giúp thiết lập kết nối mạng chuyên dụng từ hạ tầng nội bộ trực tiếp đến AWS, mà **không đi qua Internet**. Cùng tìm hiểu chi tiết hơn dưới đây.


## Trong bài này:

- [1. Kiến trúc](#architecture)
- [2. Thiết kế HA cho Direct Connect](#ha)
- [3. Kết hợp Site-to-Site VPN và Direct Connect](#dx-with-vpn)
    - [3.1. Chuyển đổi Dự phòng](#failover)
    - [3.2. Mã hoá](#encryption)
- [4. Một vài Lưu ý](#notes)
- [5. Câu hỏi Ôn tập](#quiz)
- [Tài liệu tham khảo](#reference)


<a name = "architecture"></a>

## 1. Kiến trúc

Hình dưới đây mô tả kiến trúc đơn giản nhất của Direct Connect, cách kết nối từ hạ tầng nội bộ đến AWS. Về cơ bản, ta **kết nối từ hạ tầng nội bộ đến một _DX Location_, AWS sẽ lo việc kết nối từ DX Location đến hạ tầng của họ**.

<p>
<image src="/assets/44_dx/dx.png" alt="Direct Connect Architecture" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>


Trọng tâm trong kiến trúc này tất nhiên là **DX Location**. Đây bản chất là một trung tâm dữ liệu mà AWS thuê để đặt thiết bị mạng của họ. AWS hiện có hơn 100 DX Location trên toàn cầu, đặt ở các thành phố lớn và trong các trung tâm dữ liệu lớn. Danh sách chi tiết có thể tham khảo [tại đây](https://aws.amazon.com/directconnect/locations/).

- Mạng nội bộ phải được cài đặt ở DX Location, có thể tự triển khai hoặc thuê qua bên thứ ba. Cần ít nhất một **router** đặt trong một **cage** (lồng chứa máy chủ, đơn vị cho thuê trong trung tâm dữ liệu). 
- AWS cũng thuê một cage khác trong cùng DX Location để đặt router (gọi là **DX Endpoint**). Router này sẽ kết nối đến hạ tầng AWS thông qua mạng riêng của họ, không qua Internet.
- Người dùng yêu cầu một LOA-CFA (*Letter of Authorization - Connecting Facility Assignment*) từ AWS, gửi đến trung tâm dữ liệu để **nối cáp quang từ Customer Router đến DX Endpoint**, gọi là **Cross Connect**.

Đến đây việc triển khai ở phía người dùng đã hoàn tất. 

Ở phía AWS, DX Endpoint kết nối đến hạ tầng cloud bằng **Virtual Interface (VIF)**. Có 3 loại VIF:

- **Private VIF**: 1 private VIF kết nối trực tiếp đến 1 VPC (nằm trong [private network](/2025/11/13/vpc#private-network)), thông qua Virtual Private Gateway giống như [Site-to-Site VPN](/2026/04/12/vpn). Giao tiếp với tài nguyên trong VPC qua private IP.
- **Public VIF**: 1 public VIF kết nối đến tất cả dịch vụ AWS có endpoint công khai (nằm trong [public network](/2025/11/13/vpc#public-network)), như [S3](/2025/11/30/s3-fundamental), [DynamoDB](/2026/03/18/dynamodb), v.v. Nhắc lại, vẫn không đi qua Internet.
- **Transit VIF**: 1 transit VIF kết nối đến một [Transit Gateway](/2026/02/08/vpc-advanced#transit-gateway), để giao tiếp với tất cả VPC gắn vào Transit Gateway.


<a name = "ha"></a>

## 2. Thiết kế HA cho Direct Connect

Kiến trúc trình bày ở phần trên là kiến trúc tối thiểu nhất để thiết lập Direct Connect, nhưng nó không có tính sẵn sàng (*High Availability - HA*) cao, vì có quá nhiều *single point of failure*:

- Mạng nội bộ chỉ có một kết nối duy nhất đến DX Location. Nếu kết nối này gặp sự cố, toàn bộ kết nối đến AWS sẽ bị gián đoạn.
- Trong DX Location, *Cross Connect* bản chất chỉ là một sợi cáp quang, hoàn toàn có thể gặp sự cố vật lý.

Ở đây ta không cần quan tâm đến phía AWS, họ sẽ đảm bảo kết nối từ DX Endpoint đến hạ tầng cloud có tính HA cao. 

Kể cả khi khắc phục 2 vấn đề trên, nhìn rộng hơn một chút, ta sẽ thấy 2 điểm yếu khác:
- DX Location cũng có thể gặp sự cố (dù xác suất nhỏ hơn nhiều), ví dụ như mất điện, hỏa hoạn, v.v.
- Hạ tầng nội bộ nếu chỉ đặt ở một ví trí địa lý duy nhất cũng là single point of failure.

Tổng kết lại, kiến trúc HA tốt nhất cho Direct Connect sẽ như sau: 
- Ít nhất 2 địa điểm đặt hạ tầng nội bộ.
- Ít nhất 2 DX Location. 
- Trong mỗi địa điểm đặt hạ tầng nội bộ, ít nhất 2 mạng nội bộ kết nối đến DX Location tương ứng.
- Trong mỗi DX Location, ít nhất 2 Cross Connect tương ứng với 2 mạng nội bộ.


<p>
<image src="/assets/44_dx/dx-ha.png" alt="Direct Connect High Availability" style="max-width:60%;height:auto;display:block;margin:0 auto;"/>
</p>

Tất nhiên chi phí để triển khai kiến trúc HA này cao hơn nhiều so với kiến trúc tối thiểu, do AWS tính phí Direct Connect theo giờ (dù có dùng hay không) và theo lượng dữ liệu truyền ra khỏi AWS (*outbound data transfer*). Do đó, tùy yêu cầu về HA và ngân sách, có thể lựa chọn triển khai một phần hoặc toàn bộ kiến trúc HA này.


<a name = "dx-with-vpn"></a>

## 3. Kết hợp Site-to-Site VPN và Direct Connect

Có thể kết hợp VPN vào Direct Connect, cho 2 mục đích chính: **chuyển đổi dự phòng** (**failover**) và **mã hoá dữ liệu**.


<a name = "failover"></a>

### 3.1. Chuyển đổi Dự phòng


<p>
<image src="/assets/44_dx/dx-vpn.png" alt="Direct Connect Failover with VPN" style="max-width:90%;height:auto;display:block;margin:0 auto;"/>
</p>

Nếu không có nhu cầu triển khai DX với kiến trúc HA cao như trình bày ở phần trên, có thể sử dụng Site-to-Site VPN làm giải pháp dự phòng. DX là đường truyền chính để tận dụng tốc độ và băng thông cao, còn VPN sẽ là giải pháp dự phòng khi DX gặp sự cố, đảm bảo vẫn có thể kết nối đến AWS cho tới khi khắc phục xong.

Ngoài ra VPN cũng có thể dùng khi đang chờ thiết lập Direct Connect, quá trình có thể mất vài tuần để hoàn thành.


<a name = "encryption"></a>

### 3.2. Mã hoá 

Cả 3 loại VIF đều không mã hoá dữ liệu, vì vậy trong các trường hợp có yêu cầu bảo mật cao, có thể thiết lập VPN trên đường truyền Direct Connect để mã hoá dữ liệu **truyền từ DX Location tới AWS**. 



<p>
<image src="/assets/44_dx/dx-encrypt.png" alt="Direct Connect Encryption with VPN" style="max-width:90%;height:auto;display:block;margin:0 auto;"/>
</p>

Thiết kế này thường dùng cho các tổ chức tài chính hoặc chính phủ, đáp ứng các tiêu chuẩn bảo mật khắt khe yêu cầu mọi dữ liệu rời khỏi trung tâm dữ liệu phải được mã hóa.


<a name = "notes"></a>

## 4. Một vài Lưu ý

Khi ôn tập thi chứng chỉ AWS, bạn đọc cần lưu ý những điểm sau khi cân nhắc giữa Site-to-Site VPN và Direct Connect để trả lời các câu hỏi về giải pháp kết nối hạ tầng nội bộ và AWS:

- DX hỗ trợ băng thông 1, 10, hoặc 100 Gbps. Cao hơn nhiều so với giới hạn 1.25 Gbps của Site-to-Site VPN. Tốc độ cao, độ trễ thấp, và ổn định là những lợi ích chính của DX so với VPN.
- **Thiết lập DX mất nhiều thời gian**, thường là vài tuần, do cần làm việc với trung tâm dữ liệu để thiết lập Cross Connect. 
- Mặc định, **DX không mã hoá dữ liệu**.
- Nhắc lại, **DX không đi qua Internet**. 




<a name = "quiz"></a>

## 5. Câu hỏi Ôn tập

<iframe src="https://www.facebook.com/plugins/post.php?href=https%3A%2F%2Fwww.facebook.com%2Fawscoban%2Fposts%2Fpfbid02L799Ws656BifBRB6okShKuP6DRzLFxGn3iZSXVWqnaJsQaRF1kUQirGnbZFprS2xl&show_text=true&width=500" width="500" height="704" style="border:none;overflow:hidden" scrolling="no" frameborder="0" allowfullscreen="true" allow="autoplay; clipboard-write; encrypted-media; picture-in-picture; web-share"></iframe>


<a name = "reference"></a>

## Tài liệu tham khảo

1. [Giới thiệu về AWS Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html)
2. [Danh sách các DX Location](https://aws.amazon.com/directconnect/locations/)
3. [Kết hợp Site-to-Site VPN và Direct Connect](https://docs.aws.amazon.com/vpn/latest/s2svpn/Examples.html#vpn-direct-connect)


Qua 2 bài vừa rồi, mình đã trình bày 2 giải pháp kết nối hạ tầng nội bộ đến AWS phổ biến nhất: Site-to-Site VPN và Direct Connect. Tiếp theo, hãy chuyển sang bài toán lưu trữ dữ liệu trong môi trường hybrid, với AWS **Storage Gateway**.
