---
layout: post
title: "45. Storage Gateway"
title2: "Storage Gateway"
date: 2026-04-25
permalink: /2026/04/25/storage-gateway
categories: [Storage, Hybrid]
tags: [Storage, Hybrid]
img: /assets/45_storage_gateway/storage-gateway-volume-stored.png
summary: "Storage Gateway là một dịch vụ lưu trữ linh động giữa hạ tầng nội bộ và AWS (hybrid cloud storage), giúp sao lưu, di chuyển dữ liệu lên AWS, hoặc tận dụng AWS để mở rộng lưu trữ nội bộ. Bài này giới thiệu 3 loại Storage Gateway chính: Volume Gateway cho dữ liệu dạng khối, S3 File Gateway cho dữ liệu dạng tệp, và Tape Gateway cho dữ liệu băng từ. "
---

Storage Gateway là một dịch vụ lưu trữ linh động giữa hạ tầng nội bộ và AWS (*hybrid storage*), giúp sao lưu, di chuyển dữ liệu lên AWS, hoặc tận dụng AWS để mở rộng lưu trữ nội bộ. Bài này giới thiệu 3 loại Storage Gateway chính: Volume Gateway cho dữ liệu dạng khối, S3 File Gateway cho dữ liệu dạng tệp, và Tape Gateway cho dữ liệu băng từ. 



## Trong bài này:

- [1. Tổng quan](#overview)
- [2. Volume Gateway](#volume-gateway)
    - [2.1. Stored Volume](#stored-volume)
    - [2.2. Cached Volume](#cached-volume)
- [3. S3 File Gateway](#file-gateway)
- [4. Tape Gateway](#tape-gateway)
- [Tài liệu tham khảo](#reference)


<a name = "overview"></a>

## 1. Tổng quan

Storage Gateway kết nối hạ tầng nội bộ với các dịch vụ lưu trữ trên AWS, thường dùng với 3 mục đích:
- **Migration**: Di chuyển dữ liệu từ hạ tầng nội bộ lên AWS
- **Backup**: Sao lưu dữ liệu từ hạ tầng nội bộ lên AWS, dự phòng khi có sự cố với hạ tầng nội bộ (*disaster recovery*)
- **Extension**: Tận dụng AWS để mở rộng lưu trữ nội bộ


Storage Gateway có thể được triển khai theo 3 cách chính:
- **Virtual Machine**: triển khai trên một máy ảo ở hạ tầng nội bộ. Cách này **phổ biến nhất**, dễ triển khai, chạy được trên nhiều nền tảng ảo hóa (*hypervisor*) khác nhau.
- **Hardware Appliance**: AWS cung cấp phần cứng tích hợp Storage Gateway, thường dùng cho các doanh nghiệp có nhu cầu lưu trữ lớn và muốn giải pháp phần cứng chuyên dụng.
- Triển khai trên một **EC2 Instance**.


Dưới đây, mình sẽ trình bày chi tiết 3 loại Storage Gateway chính: Volume Gateway, S3 File Gateway, và Tape Gateway.



<a name = "volume-gateway"></a>

## 2. Volume Gateway

"Volume" trong tên gọi ám chỉ "ổ đĩa" (*disk*), bản chất là [block storage](/2025/11/30/s3-fundamental#storage-comparison). Volume Gateway là một ổ đĩa ảo **trên AWS**, có thể được gắn (*mount*) vào máy chủ nội bộ qua giao thức [**iSCSI**](https://en.wikipedia.org/wiki/ISCSI). 


Volume Gateway có 2 chế độ hoạt động chính: **Stored Volume** và **Cached Volume**, tuỳ thuộc vào dữ liệu ở hạ tầng nội bộ được lưu trữ **toàn bộ** hay **một phần**. 



<a name = "stored-volume"></a>

### 2.1. Stored Volume

Chế độ này lưu trữ **toàn bộ dữ liệu** ở hạ tầng nội bộ, đồng thời **sao lưu lên AWS**. Đây bản chất là một phương án sao lưu dữ liệu, còn việc truy cập dữ liệu vẫn trên hạ tầng nội bộ.

Dữ liệu được sao lưu **bất đồng bộ** lên AWS dưới dạng [EBS snapshot](/2025/12/20/ebs#ebs-snapshot), dùng để phục hồi dữ liệu khi có sự cố với hạ tầng nội bộ.

<p>
<image src="/assets/45_storage_gateway/storage-gateway-volume-stored.png" alt="Storage Gateway: Volume Stored" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Hình trên minh hoạ cách thức hoạt động của Stored Volume Gateway. Ở phía hạ tầng nội bộ:
- Người dùng cài đặt Storage Gateway trên một máy ảo, rồi tạo một hoặc nhiều ổ đĩa ảo (**Volume Storage**) trên đó.
- Ổ đĩa ảo này được đồng bộ với hệ thống lưu trữ nội bộ, có thể là:
    - **Direct Attached Storage (DAS)**: ổ đĩa gắn trực tiếp vào máy chủ, không qua mạng.
    - **Network Attached Storage (NAS)**: thiết lưu trữ được truy cập qua mạng nội bộ (LAN), tương tự như [EFS](/2025/12/24/efs).
    - **Storage Area Network (SAN)**: là mạng lưu trữ chuyên dụng, hiệu năng rất cao. dùng trong các doanh nghiệp lớn
- Sau khi đồng bộ, ổ đĩa ảo này được gắn vào máy chủ nội bộ qua giao thức **iSCSI**, cho phép đọc/ghi dữ liệu như một ổ đĩa thông thường. 
- **Upload Buffer**: là một vùng đệm để chuẩn bị dữ liệu trước khi tải lên AWS. 

Ở phía AWS:

- Gateway VM kết nối đến AWS qua một Endpoint, tải dữ liệu từ Upload Buffer lên AWS, mã hoá rồi lưu trên [S3](/2025/11/30/s3-fundamental). 
- Hỗ trợ sao lưu lũy kế (*incremental backup*) dưới dạng [EBS snapshot](/2025/12/20/ebs#ebs-snapshot). Khi tạo snapshot mới ở Gateway VM, chỉ lưu lại những thay đổi kể từ snapshot trước đó.
- Snapshot có thể dùng để tạo ổ EBS mới trên AWS, hoặc cũng có thể phục hồi ổ đĩa ảo trên hạ tầng nội bộ nếu cần.

Mỗi Gateway VM hỗ trợ tối đa **32** ổ đĩa ảo, mỗi ổ có dung lượng từ 1 GiB đến **16 TiB**, tức tổng dung lượng tối đa **512 TiB**.


<a name = "cached-volume"></a>

### 2.2. Cached Volume

Như tên gọi, chế độ này chỉ lưu trữ **một phần dữ liệu được truy cập thường xuyên** ở hạ tầng nội bộ (coi như **cache**), còn lại chuyển lên [S3](/2025/11/30/s3-fundamental) là nơi lưu trữ chính. Cách thức này phù hợp khi không muốn mở rộng hạ tầng lưu trữ nội bộ, nhưng vẫn muốn tối ưu độ trễ khi truy cập dữ liệu thường xuyên.



<p>
<image src="/assets/45_storage_gateway/storage-gateway-volume-cached.png" alt="Storage Gateway: Volume Cached" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Cơ chế hoạt động của Cached Volume Gateway cũng gần giống như Stored Volume, nhưng đơn giản hơn:
Ở phía hạ tầng nội bộ:

- Vẫn cài đặt Storage Gateway trên một máy ảo, tạo ổ đĩa ảo trên đó, rồi gắn vào máy chủ nội bộ qua **iSCSI**.
- Không có hạ tầng lưu trữ nội bộ (DAS, NAS, SAN) nào, do nơi lưu trữ chính là S3 trên AWS.
- **Cache Storage**: là một vùng lưu trữ cục bộ để lưu trữ dữ liệu được truy cập thường xuyên, giảm độ trễ khi truy cập dữ liệu.
- **Upload Buffer**: vẫn có để chuẩn bị dữ liệu mới trước khi tải lên AWS.

Ở phía AWS:
- Dữ liệu được lưu trữ trên S3, có mã hoá khi lưu trữ ([SSE](/2025/12/06/s3-security#sse)). Nhưng không thể thấy những dữ liệu này trên giao diện S3 hay API.
- Cũng hỗ trợ sao lưu lũy kế dưới dạng EBS snapshot và phục hồi dữ liệu tương tự như Stored Volume Gateway. 


Giống Stored Volume, Cached Volume Gateway cũng hỗ trợ tối đa **32** ổ đĩa ảo, mỗi ổ có dung lượng từ 1 GiB đến **16 TiB**, tức tổng dung lượng tối đa **512 TiB** mỗi Gateway.


<a name = "file-gateway"></a>

## 3. S3 File Gateway

Nếu cần làm việc với dữ liệu dạng tệp (*file*) trên hạ tầng nội bộ, thì S3 File Gateway là lựa chọn phù hợp. Cơ chế hoạt động như sau:



<p>
<image src="/assets/45_storage_gateway/storage-gateway-file.png" alt="Storage Gateway: S3 File" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

- Người dùng cài đặt S3 File Gateway trên một máy ảo ở hạ tầng nội bộ, rồi tạo một hoặc nhiều **File Share** (là một thư mục lớn chứa các thư mục con và tệp tin) trên đó. **Mỗi File Share tương ứng với một S3 Bucket**.
- File Share được truy cập từ máy chủ nội bộ thông qua giao thức [NFS](https://en.wikipedia.org/wiki/Network_File_System) cho Linux hoặc [SMB](https://en.wikipedia.org/wiki/Server_Message_Block) cho Windows. 
- Tệp tin ghi vào File Share sẽ được lưu dưới dạng [object](/2025/11/30/s3-fundamental#s3-object) trên S3, với object key là đường dẫn tệp tin trong File Share. Ví dụ, tệp tin `/folder1/awscoban.md` sẽ được lưu dưới dạng object với key là `folder1/awscoban.md`. **Mỗi tệp tương ứng với một object**. Khi tệp tin được cập nhật trên File Share, một [phiên bản](/2025/11/30/s3-fundamental#s3-versioning) mới của object sẽ được tạo trên S3.
- Ngoài ra, dữ liệu thường xuyên được truy cập sẽ được lưu trong **cache** tại máy ảo.

S3 File Gateway hỗ trợ các [lớp lưu trữ trong S3](/2025/12/09/s3-storage-classes). Khi tạo File Share, người dùng có thể chọn lớp lưu trữ mặc định cho các tệp tin, rồi tạo [Life Cycle policy](/2025/12/09/s3-storage-classes#s3-lifecycle) để tự động chuyển đổi lớp lưu trữ theo tần suất truy cập nhằm tối ưu chi phí.



<a name = "tape-gateway"></a>

## 4. Tape Gateway

Dữ liệu băng từ (*tape*) là một phương pháp lưu trữ truyền thống, thường dùng cho dữ liệu cũ (**archived data**).
Tape Gateway là giải pháp sao lưu dữ liệu lên AWS, thay thế thư viện băng từ vật lý (*Physical Tape Library*) bằng băng từ ảo (*Virtual Tape Library*) lưu trên [S3 Glacier](/2025/12/09/s3-storage-classes#s3-glacier) hoặc [Glacier Deep Archive](/2025/12/09/s3-storage-classes#s3-glacier-deep-archive).


<p>
<image src="/assets/45_storage_gateway/storage-gateway-tape.png" alt="Storage Gateway: Tape" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

- Người dùng vẫn cài đặt Tape Gateway trên một máy ảo ở hạ tầng nội bộ. Rồi tạo **Virtual Tape Library (VTL)** trên Gateway. VTL là tổ hợp nhiều **Virtual Tape** (băng từ ảo), mỗi băng có dung lượng từ 100 GiB đến 15 TiB. 1 Tape Gateway ứng với 1 VTL, chứa tối đa 1500 băng từ ảo hoặc tổng 1 PiB dữ liệu trên toàn bộ các băng từ ảo.
- **Tape Drive**: một thiết bị ảo để đọc/ghi dữ liệu vào băng từ ảo, tương ứng với ổ đĩa để đọc/ghi băng từ vật lý.
- **Media Changer**: một thiết bị ảo để quản lý việc di chuyển băng từ ảo giữa Tape Drive và Storage. Tương ứng với cánh tay robot trong thư viện băng từ vật lý.
- Máy chủ nội bộ truy cập Tape Gateway qua giao thức **iSCSI**, dùng phần mềm sao lưu để ghi dữ liệu vào Tape Gateway như một thiết bị băng từ vật lý thông thường. Tape Gateway tải dữ liệu lên S3.

Ở phía AWS:

- **Archive**: trong băng từ vật lý, một băng từ sau khi ghi đầy sẽ được chuyển vào kho lưu trữ lâu dài (*archive*). Tương tự, khi một Virtual Tape được ghi đầy, nó sẽ được "đóng" và chuyển vào một **Virtual Tape Shelf** (hiểu như một "giá sách" ảo) trên AWS, rồi lưu vào Glacier hoặc Glacier Deep Archive để tiết kiệm chi phí.
- **Retrieve**: khi cần truy cập dữ liệu đã lưu trữ, người dùng có thể yêu cầu Tape Gateway lấy băng từ ảo từ Virtual Tape Shelf về Tape Gateway. Không thể đọc trực tiếp từ Virtual Tape Shelf.



<a name = "reference"></a>

## Tài liệu tham khảo

1. [Storage Gateway: Stored và Cached Volume](https://docs.aws.amazon.com/storagegateway/latest/vgw/StorageGatewayConcepts.html)
2. [iSCSI](https://en.wikipedia.org/wiki/ISCSI)
3. [File Gateway](https://docs.aws.amazon.com/filegateway/latest/files3/file-gateway-concepts.html)
4. [NFS](https://en.wikipedia.org/wiki/Network_File_System)
5. [SMB](https://en.wikipedia.org/wiki/Server_Message_Block)
6. [Storage Class cho S3 File Gateway](https://docs.aws.amazon.com/filegateway/latest/files3/storage-classes.html)
6. [Tape Gateway](https://docs.aws.amazon.com/storagegateway/latest/tgw/StorageGatewayConcepts.html)


Đây là những kiến thức cần biết về Storage Gateway cần thiết cho kỳ thi SAA. Bạn đọc cần nắm vững cách sử dụng, **giao thức** nào được đề cập trong câu hỏi (iSCSI, NFS, hay SMB), và ngữ cảnh liên quan để trả lời đúng. Tiếp theo, hãy tiếp tục tìm hiểu cách dịch chuyển dữ liệu từ hạ tầng nội bộ lên AWS với các dịch vụ khác như DataSync và Snowball Edge.
