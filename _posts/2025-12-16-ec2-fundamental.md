---
layout: post
title: "13. EC2 Cơ Bản"
title2: "EC2 Cơ Bản"
date: 2025-12-16
permalink: /2025/12/16/ec2-fundamental
categories: [EC2, Storage, EBS, EFS]
tags: [EC2, Storage, EBS, EFS]
img: /assets/13_ec2/ec2-storage.png
summary: "EC2 là dịch vụ cốt lõi bậc nhất trên AWS, cung cấp máy chủ ảo (gọi là EC2 Instance), giúp người dùng \"thuê\" năng lực tính toán (CPU, bộ nhớ, lưu trữ, mạng, v.v.) để chạy ứng dụng mà không cần mua phần cứng và tự cài đặt. Về bản chất, mỗi EC2 Instance là một máy ảo chạy trên một phần cứng vật lý (gọi là Host), theo công nghệ Virtualization."
---

Hy vọng qua loạt bài vừa rồi, bạn đọc đã nắm được các khái niệm, tính năng và cách sử dụng S3 hợp lý. Giờ hãy chuyển sang một dịch vụ AWS khác quan trọng và phổ biến không kém: **Elastic Compute Cloud** hay EC2.


## Trong bài này:

- [1. Virtualization](#virtualization)
- [2. Các Trạng thái của EC2 Instance](#ec2-state)
- [3. Lưu trữ trong EC2](#ec2-storage)
- [4. Các loại Instance](#instance-type)
- [5. Giao diện Mạng trong EC2](#network-interface)
- [Tài liệu tham khảo](#reference)



EC2 là dịch vụ cốt lõi bậc nhất trên AWS, cung cấp máy chủ ảo (gọi là *EC2 Instance*), giúp người dùng "thuê" năng lực tính toán (CPU, bộ nhớ, lưu trữ, mạng, v.v.) để chạy ứng dụng mà không cần mua phần cứng và tự cài đặt. 

<a name="virtualization"></a>

## 1. Virtualization 

Về bản chất, mỗi EC2 Instance là một máy ảo chạy trên một phần cứng vật lý (gọi là **Host**), theo công nghệ [**Virtualization**](https://www.ibm.com/think/topics/virtualization).
Công nghệ này giúp chia một host đơn lẻ thành nhiều máy ảo. Mỗi máy ảo chạy hệ điều hành riêng, hoạt động như thể một máy có phần cứng riêng biệt. Việc này được quản lý bởi một lớp phần mềm đặc biệt gọi là [**Hypervisor**](https://aws.amazon.com/what-is/hypervisor/), chạy trên host, "ảo hoá" tài nguyên và phân phối cho các máy ảo.


<p>
<image src="/assets/13_ec2/virtualization.png" alt="Virtualization" style="max-width:50%;height:auto;display:block;margin:0 auto;"/>
</p>

Virtualization giúp khai thác tối đa tài nguyên trên các máy chủ vật lý, quản lý và mở rộng dễ dàng hơn so với việc sử dụng phần cứng riêng biệt cho mỗi EC2 Instance.
Lưu ý, cần phân biệt **virtualization** với **containerization** (Docker, Kubernetes). Virtualization ảo hoá **phần cứng** để chia cho các máy ảo (chạy cả hệ điều hành và các ứng dụng). Còn containerization chỉ ảo hoá **hệ điều hành** để chạy ứng dụng, quy mô và độ phức tạp nhỏ hơn. 
Ta có thể tạo máy ảo, rồi chạy một Docker container trên máy ảo đó. 

Mỗi EC2 Host được đặt tại 1 AZ, tất nhiên chỉ có khả năng phục hồi [AZ Resilience](/2025/11/12/aws_infrastructure/#resilience). Nếu AZ xảy ra vấn đề, EC2 Host sẽ ngừng hoạt động, cùng với tất cả EC2 Instance chạy trên nó. Một Host thường được dùng chung bởi nhiều người dùng (mỗi người dùng chạy Instance riêng), hoặc nếu cần thiết, một người dùng có thể trả tiền và sử dụng toàn bộ Host (Dedicated Host). 
<!-- TODO: include link to Dedicated Host -->

Một EC2 Instance chạy trên một EC2 Host, nếu bạn **reboot** Instance, nó vẫn chạy trên cùng Host. 
Nhưng nếu bạn **stop** rồi lại **start** Instance đó, có khả năng nó sẽ được **chuyển qua Host khác**. 
Ngoài ra, nếu một Host cần tạm ngưng để bảo trì, các Instance trên đó cũng được chuyển qua Host khác.


<a name = "ec2-state"></a>

## 2. Các Trạng thái của EC2 Instance:

- **Pending**: Instance đang khởi tạo, thường chỉ trong 1-2 phút. Lúc này, AWS đang cấp phát tài nguyên và cài đặt hệ điều hành. Đây là trạng thái sau khi tạo Instance mới, **start** một Instance đang ở trạng thái `stopped`, hoặc **reboot** một Instance đang ở trạng thái `running`.
- **Running**: Instance hoạt động bình thường và có thể sử dụng và truy cập.
- **Stopping**: Instance đang dừng, thường chỉ trong 1-2 phút. Đây là trạng thái sau khi **stop** một Instance đang ở trạng thái `running`. 
- **Stopped**: Instance đã dừng hoàn toàn. Instance không chạy và không tính chi phí tính toán, chỉ tính phí lưu dữ liệu liên quan.
- **Terminated**: Instance bị xóa vĩnh viễn. Đây là trạng thái sau khi **delete** Instance.


<a name = "ec2-storage"></a>

## 3. Lưu trữ trong EC2

<p>
<image src="/assets/13_ec2/ec2-storage.png" alt="EC2 Storage" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>


Về lưu trữ, EC2 hỗ trợ tất cả [các loại lưu trữ trên AWS](/2025/11/30/s3-fundamental#storage-comparison). Cụ thể:

- **Block Storage**: Instance Store và Elastic Block Store (EBS). Dưới đây mình tóm tắt các ý chính, bạn đọc có thể tìm hiểu kỹ hơn trong bài sau.
    <!-- TODO: include link to bài sau -->
    - Trên EC2 Host có sẵn block storage để lưu dữ liệu **tạm thời**, gọi là **Instance Store**, được chia nhỏ thành nhiều *volume*. Mỗi EC2 Instance chạy trên Host sẽ được cấp một (vài) volume tuỳ vào loại và kích thước Instance. Khi dừng, **ngủ đông (hibernate)**, hay **chấm dứt (terminate)** Instance đó, dữ liệu lưu trong Instance Store Volume tương ứng sẽ bị xoá đi.
    - Do Instance Store chỉ lưu dữ liệu tạm thời, loại Block Storage thường dùng hơn cho EC2 là **Elastic Block Store (EBS)**, là một dạng lưu trữ ngoài ổn định. Mỗi EBS được chia nhỏ thành *volume* để kết nối với EC2 Instance cụ thể, và quan trọng nhất,  **độc lập với Instance**, tức dữ liệu lưu trên volume mặc định sẽ không mất đi khi ngắt kết nối với Instance (stop, hibernate, hay terminate). Lưu ý, EBS phải ở **cùng AZ** với EC2 Instance (hay Host). Không thể kết nối Instance đến một EBS volume khác AZ.
    <!-- TODO: include link to EBS -->

- **File Storage**: hỗ trợ **Elastic File System** (**EFS**) cho các Instance chạy Linux, và **FSx** (một loại File Storage hiệu năng cao). Với EFS, có thể "**mount**" Instance tới một *file system* và đọc/ghi dữ liệu lên đó. Khác với EBS, nhiều Instance có thể mount cùng một EFS file system, và do kết nối bằng mạng máy tính, file system không cần phải ở cùng AZ với với EC2 Instance (hay Host). Bạn đọc có thể tìm hiểu kỹ hơn trong bài EFS.
    <!-- TODO: include link to EFS -->
    <!-- TODO: include link to FSx -->
    <!-- TODO: include link to bài EFS -->


- **Object Storage**: có thể kết nối EC2 Instance để đọc/ghi dữ liệu lên S3, chỉ cần cấu hình [Bucket Policy](/2025/12/06/s3-security#bucket-policy) đúng. Ngoài ra, các bản sao lưu của EBS (gọi là **EBS Snapshot**) cũng được lưu trên S3.



<a name="instance-type"></a>

## 4. Các loại Instance


EC2 có thể được sử dụng cho hầu hết các tác vụ. AWS chia EC2 thành các nhóm nhỏ tối ưu cho từng loại tác vụ như sau:

- **General Purpose**: tổng thể tài nguyên cân bằng, dùng cho các ứng dụng phổ biến. 
- **Compute Optimized**: tăng CPU, dùng cho các tác vụ nặng tính toán, xử lý dữ liệu theo batch (*batch processing*), máy chủ hiệu năng cao, v.v.
- **Memory Optimized**: tăng RAM, dùng cho các tác vụ cần nhiều bộ nhớ, như in-memory database, phân tích dữ liệu lớn, v.v.
- **Storage Optimized**: tăng bộ nhớ ngoài và tốc độ giao tiếp I/O, dùng cho các tác vụ cần đọc/ghi bộ nhớ ngoài nhiều, cơ sở dữ liệu, kho dữ liệu (*data warehouse*), v.v.
- **Accelerated Computing**: tăng sức mạnh xử lý đồ hoạ (GPU) hoặc phần cứng chuyên dụng (FPGA), dùng cho các tác vụ học máy, xử lý đồ hoạ, mô phỏng khoa học, v.v.


#### Cách Đọc Ký hiệu Instance

```
[Family][Generation][Suffix (optional)].[Size]
```


Ví dụ, với Instance có mã `c6gd.4xlarge`:

- Chữ cái đầu tiên `c` cho biết đây là Compute Optimized Instance. 
- Chữ số tiếp theo `6` cho biết đây là thế hệ thứ 6 được phát triển của loại này.
- 0-2 chữ cái tiếp theo (tuỳ chọn) cho biết các đặc tính bổ sung: `a` (bộ xử lý AMD), `g` (bộ xử lý AWS Graviton), `d` (có Instance store volume)
- Sau dấu `.` là kích thước vCPU và RAM: `nano`, `micro`, `small`, `medium`, `large`, `xlarge`, `2xlarge`, `4xlarge`, v.v.

Chi tiết cách đọc ký hiệu Instance [tại đây](https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-type-names.html).

Các loại Instance, kích thước, bộ nhớ, kiến trúc, hay các lựa chọn thanh toán có thể xem [tại đây](https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-types.html).


<a name="network-interface"></a>

## 5. Giao diện Mạng trong EC2

Giao diện mạng (**Network Interface**) có nhiệm vụ kết nối thiết bị với mạng bên ngoài (từ mạng nội bộ đến Internet). 
Có thể là **phần cứng** (như cổng Ethernet, WIFI adapter trên máy tính cá nhân) hay **phần mềm** (như loopback interface trên bộ định tuyến, giúp tạo giao diện mạng ảo hoạt động như giao diện mạng vật lý).

<a name="eni-ena"></a>

Trên EC2, mỗi Instance có một hoặc hai giao diện mạng gọi là [*Elastic Network Interface* - *ENI*](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html). ENI thực chất được ảo hoá nhờ [Virtualization](#virtualization) đề cập ở phần trên, từ một giao diện mạng phần cứng rất mạnh được AWS thiết kế, gọi là **Elastic Network Adapter** (**ENA**). 

Mỗi ENI có thể chứa các thuộc tính sau:
- Địa chỉ MAC 
- Địa chỉ IPv4 riêng tư chính (*primary private IPv4*) thuộc [CIDR](/2025/11/13/vpc#cidr) của [subnet](/2025/11/13/vpc#subnet) mà EC2 Instance đang chạy
- Địa chỉ IPv6 chính thuộc CIDR của subnet (nếu subnet có IPv6)
- 0 hoặc nhiều địa chỉ IPv4 riêng tư phụ (*secondary private IPv4*) thuộc CIDR của subnet
- 0 hoặc nhiều địa chỉ IPv6 phụ (*secondary IPv6*) thuộc CIDR của subnet
- Một địa chỉ IPv4 **tĩnh** (**Elastic IP**) cho mỗi IPv4 riêng tư (chính hoặc phụ). Đọc thêm [tại đây](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html). Đây là dịch vụ tính phí nếu sử dụng. **Không thay đổi** khi **stop rồi start** Instance.
- Một địa chỉ IPv4 công cộng. **Thay đổi** khi **stop rồi start** Instance. Nếu dùng Elastic IP, địa chỉ IPv4 công cộng này sẽ bị thay thế.
- [Security Group](/2025/11/27/sg-nacl#sg)
- Cờ kiểm tra nguồn/đích (*source/destination check*)

Bản chất khi nói đến IP của EC2 Instance, ví dụ như trong Security Group, thực chất đó là IP của ENI được gắn vào Instance. 
Mỗi Instance có một ENI chính (*primary ENI*) mặc định được gắn vào khi khởi tạo, không thể tách ra. Người dùng có thể tạo và gắn thêm ENI phụ (*secondary ENI*) để dùng cho tường lửa, hoặc nếu cần tách luồng mạng riêng.





<a name = "reference"></a>

## Tài liệu tham khảo

1. [Virtualization: Bài Viết của IBM](https://www.ibm.com/think/topics/virtualization)
2. [AWS Hypervisor](https://aws.amazon.com/what-is/hypervisor/)
3. [Các loại Lưu trữ có thể dùng cho EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Storage.html)
4. [So sánh các loại EC2 Instance](https://Instances.vantage.sh/)
5. [Cách Đọc Ký hiệu Instance](https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-type-names.html)
6. [EC2 ENI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html)
7. [Elastic IP Address](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)

Tiếp theo, hãy tìm hiểu sâu hơn về Instance Store và EBS.