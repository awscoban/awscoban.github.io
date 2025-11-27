---
layout: post
title: "8. S3 Cơ bản"
title2: "S3 Cơ bản"
date: 2025-11-30
permalink: /2025/11/30/s3-fundamental
categories: [S3, Storage]
tags: [S3, Storage]
img: assets/8_s3/s3-versioning.png
summary: "Dịch vụ lưu trữ trên AWS có thể chia thành 3 loại dựa theo cấu trúc lưu trữ: Object Storage, Block Storage, và File Storage. Object Storage lưu trữ và quản lý dữ liệu theo cách coi như từng đối tượng riêng biệt. Mỗi đối tượng gồm dữ liệu gốc (văn bản, hình ảnh, tệp tin, v.v.) cùng với  các thông tin thêm về đối tượng có thể truy xuất và sử dụng, như ID, tên, ngày tạo."
---

Trong 4 bài tiếp theo, hãy cùng tìm hiểu về dịch vụ lưu trữ phổ biến nhất trên AWS: Simple Storage Service, hay S3. Bài này sẽ giới thiệu tổng quan bức tranh lớn và các khái niệm quan trọng trong S3.

## Trong bài này:

- [1. Các dịch vụ lưu trữ trên AWS](#storage-comparison)
- [2. S3](#s3)
    - [2.1. S3 Object](#s3-object)
    - [2.2. S3 Bucket](#s3-bucket)
- [3. Versioning](#s3-versioning)
    - [3.1. PUT](#put-version)
    - [3.2. Delete](#delete-version)
- [4. Tăng tốc S3](#optimize-s3)
    - [4.1. Multi-part Upload](#multi-part-upload)
    - [4.2. Transfer Acceleration](#transfer-acceleration)
- [Tài liệu tham khảo](#reference)



<a name="storage-comparison"></a>

## 1. Các dịch vụ lưu trữ trên AWS

Dựa theo cấu trúc lưu trữ, có thể chia các dịch vụ lưu trữ trên AWS thành 3 loại:

- **Object Storage**: lưu trữ và quản lý dữ liệu theo cách coi như từng đối tượng (*object*) riêng biệt. Mỗi đối tượng gồm dữ liệu gốc (văn bản, hình ảnh, tệp tin, v.v.) cùng với *metadata*, là các thông tin thêm về đối tượng có thể truy xuất và sử dụng, như ID, tên, ngày tạo, v.v. Object Storage lưu trữ phẳng, tức không có cấu trúc hình cây thư mục, tất cả đối tượng đều nằm cùng nơi chứa gốc (ở S3 gọi là **bucket**, sẽ được đề cập trong bài này). Các đối tượng được đánh địa chỉ để truy vấn bằng ID duy nhất, tạo ra bằng cách dùng hàm băm (*hash*) lên nội dung dữ liệu. Phù hợp cho đa số tác vụ lưu trữ, sao lưu dữ liệu. S3 là dịch vụ tiêu biểu, sẽ được đề cập trong bài này.


- **Block Storage**: chia dữ liệu thành các **khối** (**block**) kích thước bằng nhau, từ vài KB đến MB. Hệ điều hành sẽ gán địa chỉ duy nhất cho từng khối, lưu lại để tra cứu khi cần truy vấn dữ liệu. Block Storage cho phép truy xuất trực tiếp đến một khối dữ liệu cụ thể của dữ liệu mà không cần động vào toàn bộ các khối của dữ liệu đó. Phù hợp cho các tác vụ cần hiệu suất cao, độ trễ thấp. Block Storage trên AWS là Elastic Block Store (EBS), sẽ được thảo luận trong các bài sau. 
<!-- TODO: include EBS post link -->

- **File Storage**: đây là loại lưu trữ theo tệp tin và thư mục trên mà ta thường sử dụng. Phù hợp khi cần chia sẻ dữ liệu giữa nhiều máy với nhau. AWS cung cấp các giải pháp File Storage như Elastic File System (EFS), hay Amazon FSx.
<!-- TODO: include EFS, FSx post link -->

Bạn đọc có thể đọc thêm so sánh [tại đây](https://aws.amazon.com/compare/the-difference-between-block-file-object-storage/).


<a name="s3"></a>

## 2. S3

Như đã giới thiệu, S3 (*Simple Storage Service*) là object storage trên AWS, được khai thác theo Region, tức người dùng sẽ chọn Region khi tải dữ liệu lên S3. S3 mặc định sẽ sao lưu dữ liệu trên các AZ trong Region, nên có khả năng phục hồi [Region Resilience](/2025/11/12/aws_infrastructure/#resilience). Cũng có thể cấu hình sao lưu đa vùng (*cross-region replication*) để tăng lên [Global Resilience](/2025/11/12/aws_infrastructure/#resilience).
<!-- TODO: add link to CRR -->

S3 hỗ trợ lưu trữ tất cả các dạng dữ liệu, từ văn bản, hình ảnh, âm thanh, đến các tập dữ liệu lớn dùng cho các ứng dụng AI. 
Kích thước dữ liệu có thể lưu trữ trên S3 là gần như **không giới hạn** (tất nhiên vẫn phải trả phí theo kích thước và thời gian lưu trữ). Đây cũng là dịch vụ lưu trữ dữ liệu với chi phí thấp nhất trên AWS.

Có hai khái niệm quan trọng trong S3: **object** (dữ liệu được lưu) và **bucket** (nơi chứa object).


<a name="s3-object"></a>

### 2.1. S3 Object

S3 là object storage, nên dữ liệu trong S3 được lưu dưới dạng object. Một object có hai thành phần chính:

- **Key**: là định danh của object trong bucket. Dạng chuỗi ký tự gần giống như đường dẫn đến tệp tin. Một object có thể được truy cập khi biết key và bucket chứa nó (giả sử có quyền truy cập). Key phải đảm bảo là duy nhất trong bucket (để xác định đúng object).
- **Value**: là dữ liệu được lưu trong object, tối đa **5 TB**.

Ví dụ, trong hình dưới đây ta có một bucket tên `awscoban`, trong đó chứa một object là một tệp tin PDF tên là `sample_object.pdf`, và hai "thư mục" là `folder1` và `folder2`. Trong `folder2` có một "thư mục con" là `folder3`. Trong `folder1` có object `sample2.pdf`, trong `folder2/folder3` có object `sample3.pdf`.

Về mặt kỹ thuật, "thư mục" thực chất chỉ là **tiền tố** (**prefix**) của object, **không phải thư mục** như trong File Storage, giao diện hiển thị giống vậy để người dùng dễ tương tác, ta sẽ thảo luận tại [phần sau](#s3-bucket).

<p>
<image src="/assets/8_s3/sample_object.png" alt="S3 Object" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>


Mình chạy lệnh CLI sau để liệt kê tất cả object trong bucket `awscoban` này (bỏ qua các "thư mục"):

```sh
aws s3 ls --recursive s3://awscoban | awk '$3 != 0 {print $4}'
```

Kết quả là:

```sh
folder1/sample2.pdf
folder2/folder3/sample3.pdf
sample_object.pdf
```

Đây cũng chính là **key** của các object. Trong đó, dễ thấy các **tiền tố** là `folder1/`, `folder2/folder3/`. Định dạng key của object này khá giống cách tổ chức tệp và thư mục, nên AWS hiển thị trên giao diện giống như vậy, bạn đọc cần lưu ý để phân biệt.



Ngoài ra, object còn có một số thông tin phụ như [metadata](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingMetadata.html), permission, hay version (sẽ được đề cập ở [phần sau](#s3-versioning)).




<a name="s3-bucket"></a>

### 2.2. S3 Bucket

Có thể hiểu bucket là một "xô" chứa object (cũng là nghĩa của từ này trong tiếng Anh). 
Tên của bucket phải là **duy nhất trên toàn cầu**, bất kể ở Region hay tài khoản nào. 
Cũng dễ hiểu, vì trong phần trên ta đã nhấn mạnh một object có thể được xác định khi biết key và bucket chứa nó. Key là *duy nhất trong phạm vi bucket*, nên để hoàn toàn xác định object (có thể thuộc bất cứ tài khoản nào), tên của bucket phải là duy nhất trong phạm vi S3 trên toàn cầu. 

Một vài quy tắc đặt tên bucket:

- Tên bucket phải có độ dài từ 3 đến 63 ký tự.
- Tên bucket chỉ có thể bao gồm chữ cái thường, số, dấu chấm (`.`), và dấu gạch ngang (`-`).
- Tên bucket phải bắt đầu và kết thúc bằng một chữ cái hoặc chữ số.
- Tên bucket không được chứa hai dấu chấm liền kề.
- Tên bucket không được định dạng như một địa chỉ IP (ví dụ: 192.168.5.4).

Bạn đọc có thể xem tất cả quy tắc [tại đây](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html).



Một bucket có thể chứa **không giới hạn** số lượng object (mỗi object giới hạn kích thước từ 0 B đến 5 TB). Tại thời điểm viết bài, mặc định mỗi tài khoản có thể tạo tối đa 10,000 bucket, có thể đề nghị tăng thêm.


Bucket có cấu trúc **phẳng**, không phân tầng hình cây phức tạp, tức không có các "bucket con" hay "thư mục", các object đều đặt ở cùng cấp ngay trong bucket. Tuy giao diện web của S3 hiển thị bucket giống tổ chức tệp và thư mục (thậm chí còn có nút `Create folder`), xem [phần trên](s3-object), nhưng thực chất việc "tạo thư mục" này là tạo tiền tố cho key của object, mình nhấn mạnh điều này một lần nữa.

<p>
<image src="/assets/8_s3/bucket.png" alt="S3 Bucket" style="max-width:70%;height:auto;display:block;margin:0 auto;"/>
</p>



<a name="s3-versioning"></a>

## 3. Versioning

Trong một bucket, có thể bật chế độ quản lý phiên bản (**versioning**) để giữ các phiên bản của object trong bucket đó.
Khi được bật, các hành động làm thay đổi object sẽ tạo phiên bản mới cho object thay vì ghi đè.
Chức năng này giúp giảm thiểu rủi ro mất dữ liệu khi vô tình xoá hoặc ghi đè object.

Mặc định, khi tạo bucket, chức năng versioning ở trạng thái **disabled**. Ta có thể dễ dàng chuyển sang **enabled**, tuy nhiên một khi đã bật chức năng này, **không thể chuyển về disabled được nữa**, chỉ có thể dừng lại (chuyển sang **suspended**). Ở trạng thái này, việc tạo phiên bản mới cho object sẽ dừng lại, nhưng các phiên bản cũ vẫn được giữ lại. Có thể chuyển đổi qua lại giữa **suspended** và **enabled** tuỳ ý, chỉ không thể quay lại **disabled**. Một vài câu hỏi thi chứng chỉ sẽ kiểm tra kiến thức này, bạn đọc hãy lưu ý. Ngoài ra, chi phí lưu trữ một object sẽ được tính cho tất cả phiên bản của object đó.


Khi bật versioning, AWS tự động tạo *version ID* duy nhất cho mỗi phiên bản object. Tất cả các phiên bản có cùng Key, chỉ khác version ID. Hình dưới đây mô tả diễn biến khi thay đổi (`PUT`) và xoá (`DELETE`) object khi có versioning.


<p>
<image src="/assets/8_s3/s3-versioning.png" alt="S3 Versioning" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

<a name='put-version'>

### 3.1. PUT

Giả sử trong bucket có sẵn object có key `sample.pdf`, version ID `1111`. Khi tải lên một phiên bản mới (cùng key), S3 sẽ tạo ra version ID mới (giả sử `2222`) cho phiên bản này, và đánh dấu ID `2222` này là phiên bản hiện tại của object. Khi truy xuất object này (ví dụ, bằng `s3:GetObject`) nếu không chỉ rõ version ID, mặc định phiên bản được đánh dấu là *Current* này sẽ là đầu ra. Phiên bản cũ vẫn được giữ lại.

<a name='delete-version'>

### 3.2. Delete

Khi xoá một object có versioning, có hai trường hợp:
<a name='delete-marker'>

- Nếu **không chỉ rõ** Version ID khi xoá (hình phía trên): S3 sẽ tạo một **Delete Marker** đánh dấu object này không thể truy xuất, nhưng các phiên bản cũ và dữ liệu thực tế vẫn không mất. Delete Marker là một phiên bản đặc biệt của object, cũng có cùng key, cũng có Version ID riêng, nhưng sẽ không trả về dữ liệu của object. Trong trường hợp này, Delete Marker sẽ được đánh dấu là phiên bản hiện tại, và nếu gọi `s3:GetObject` **mà không chỉ rõ version ID** (tức sẽ trả về phiên bản hiện tại là Delete Marker), sẽ gặp lỗi `404 Not Found`. Nhưng nếu **chỉ rõ version ID** khi gọi `s3:GetObject`, phiên bản cụ thể đó vẫn được trả về. Có thể "undo" thao tác này bằng cách xoá Delete Marker (khi xoá chỉ rõ version ID của nó như trình bày ngay dưới đây). Khi đó, phiên bản ngay phía dưới sẽ trở lại là "Current". Lúc này, có thể đơn giản gọi `s3:GetObject` mà không chỉ rõ version ID, sẽ không gặp lỗi 404.

- Nếu **chỉ rõ** Version ID khi xoá (hình phía dưới): phiên bản đó sẽ bị **xoá vĩnh viễn**, Delete Marker sẽ **không được chèn vào**. Nếu đó là phiên bản hiện tại, nhãn "Current" sẽ được chuyển sang phiên bản gần nhất (nếu có). Như trên hình, khi xoá phiên bản `2222`, phiên bản `1111` sẽ trở thành "Current".



<!-- <a name='mfa-delete'>

### 3.3. MFA Delete

Đây là chế độ xác thực đa yếu tố khi xoá object, chỉ có thể được bật khi bật versioning. Chế độ này sẽ yêu cầu xác thực đa yếu tố khi xoá một phiên bản object, và thay đổi trạng thái của versioning (*enabled* qua *suspended* và ngược lại). Tìm hiểu thêm [tại đây](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingMFADelete.html). -->


<a name="optimize-s3"></a>

## 4. Tăng tốc S3

Trong phần này, hãy tìm hiểu cách tối ưu tốc độ tải dữ liệu lên S3.


<a name="multi-part-upload"></a>

### 4.1. Multi-part Upload


Khi tải một object lên S3 (trên giao diện, dùng SDK, API, hay CLI), mặc định S3 sẽ **tải toàn bộ** object, trong một [PUT request](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods/PUT). Vấn đề là nếu kích thước object lớn, nếu gặp sự cố giữa chừng (như khi kết nối mạng không ổn định), quá trình tải lên sẽ thất bại và phải bắt đầu lại từ đầu, mất thời gian và ảnh hưởng đến trải nghiệm người dùng. 




<p>
<image src="/assets/8_s3/s3-multi-part-upload.png" alt="S3 Multi-part Upload" style="max-width:50%;height:auto;display:block;margin:0 auto;"/>
</p>

[Multi-part Upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html) có thể giải quyết vấn đề này. Object kích thước lớn (AWS hỗ trợ multi-part upload cho object từ 100 MB trở lên) sẽ được chia nhỏ thành từng phần, mỗi phần có kích thước từ 5 MB đến 5 GB, trừ phần cuối cùng có thể nhỏ hơn 5 MB. Khi có sự cố, những phần đã tải lên thành công sẽ được giữ lại, chỉ thực hiện lại những phần chưa xong. Hơn nữa, việc tải lên từng phần có thể được thực hiện **song song**, giúp tận dụng đường truyền và tăng tốc độ tải lên đáng kể.

Trên AWS CLI, có thể dùng lệnh `s3 cp` để sao chép một tệp tin lớn lên S3 bucket như sau, mặc định multi-part upload sẽ được sử dụng:

```sh
aws s3 cp large_file.zip s3:://<BUCKET-NAME>/your-prefix/large_file.zip
```

Nếu cần kiểm soát chặt chẽ hơn, hãy dùng API tầng dưới là `s3api create-multipart-upload`, đọc thêm [tại đây](https://docs.aws.amazon.com/cli/latest/reference/s3api/create-multipart-upload.html#synopsis).


<a name="transfer-acceleration"></a>

### 4.2. Transfer Acceleration

Như đã giới thiệu, S3 là dịch vụ theo Region, tức dữ liệu thực tế được đặt tại một trung tâm dữ liệu thuộc một [Region](/2025/11/12/aws_infrastructure#region) cụ thể. Việc tải dữ liệu lên S3 theo cách thông thường sẽ được thực hiện qua Internet, nên bị ảnh hưởng bởi khoảng cách địa lý và tốc độ đường truyền. Trong những trường hợp đặc thù cần giảm độ trễ, giảm xác suất mất mát dữ liệu dẫn đến phải tải lại, hay cần tải các tập dữ liệu lớn từ các văn phòng trên toàn cầu lên một S3 bucket tập trung, có thể cân nhắc sử dụng Transfer Acceleration.

<p>
<image src="/assets/8_s3/s3-transfer-acc.png" alt="S3 Transfer Acceleration" style="max-width:70%;height:auto;display:block;margin:0 auto;"/>
</p>


Đây là một tính năng giúp tăng tốc độ truyền dữ liệu dù khoảng cách xa, bằng cách tải dữ liệu đến [Edge Location](/2025/11/12/aws_infrastructure#edge-location) gần nhất thay vì trực tiếp tới bucket, rồi sử dụng hạ tầng mạng riêng của AWS đưa dữ liệu từ Edge Location tới bucket ở Region chính. Theo cách này, tốc độ tải lên sẽ nhanh và ổn định hơn nhiều (50-500%) so với việc sử dụng hạ tầng Internet công cộng.

Tính năng này có thể được bật thông qua giao diện web trên từng bucket, và sử dụng endpoint `s3-accelerate.amazonaws.com` ở phía ứng dụng khi dùng AWS API, SDK hay CLI.



<a name = "reference"></a>

## Tài liệu tham khảo

1. [So sánh Block vs File vs Object storage](https://aws.amazon.com/compare/the-difference-between-block-file-object-storage/)
2. [Quy định đặt tên bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html)
3. [Bucket quotas](https://docs.aws.amazon.com/AmazonS3/latest/userguide/BucketRestrictions.html)
4. [Versioning Workflow](https://docs.aws.amazon.com/AmazonS3/latest/userguide/versioning-workflows.html)
5. [Multi-part Upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
6. [Transfer Acceleration](https://aws.amazon.com/s3/transfer-acceleration/)


Chủ đề tiếp theo mình muốn giới thiệu là bảo mật trong S3, liên quan đến mã hoá và giải mã dữ liệu. Nên trước đó, hãy tìm hiểu về Key Management Service (KMS).
