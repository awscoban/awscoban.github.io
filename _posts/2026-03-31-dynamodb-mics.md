---
layout: post
title: "41. DynamoDB: Các Tính năng Hữu ích Khác"
title2: "DynamoDB: Các Tính năng Hữu ích Khác"
date: 2026-03-31
permalink: /2026/03/31/dynamodb-mics
categories: [Database, DynamoDB, Serverless, Caching]
tags: [Database, DynamoDB, Serverless, Caching]
img: /assets/41_dynamodb_mics/dax.png
summary: "Hãy kết thúc chuỗi bài về DynamoDB bằng một số tính năng hữu ích chưa được đề cập, như DynamoDB Stream để xử lý dữ liệu thay đổi theo thời gian thực, Global Table để tăng khả năng chịu lỗi và mở rộng toàn cầu, DAX để làm bộ đệm, hay các tính năng sao lưu và tự động xoá dữ liệu cũ."
---

Hãy kết thúc chuỗi bài về DynamoDB bằng một số tính năng hữu ích chưa được đề cập, như **DynamoDB Stream** để xử lý dữ liệu thay đổi theo thời gian thực, **Global Table** để tăng khả năng chịu lỗi và mở rộng toàn cầu, **DAX** để làm bộ đệm, hay các tính năng sao lưu và tự động xoá dữ liệu cũ.



## Trong bài này:

- [1. Xử lý Dữ liệu Thay đổi với Stream](#stream)
    - [1.1. Cơ chế Hoạt động](#mechanism)
    - [1.2. View Type](#view-type)
- [2. DynamoDB Global Table](#global-table)
- [3. DynamoDB Accelerator (DAX)](#dax)
    - [3.1. Đọc](#dax-read)
    - [3.2. Ghi](#dax-write)
- [4. Sao lưu và Phục hồi](#backup-restore)
    - [4.1. On-Demand Backup](#ondemand-backup)
    - [4.2. Point-in-Time Recovery (PITR)](#pitr)
- [5. Time to Live (TTL)](#ttl)
- [Tài liệu tham khảo](#reference)



<a name = "stream"></a>

## 1. Xử lý Dữ liệu Thay đổi với Stream

Trong các ứng dụng hướng sự kiện (*event-driven*) hiện đại, việc phản ứng nhanh với các thay đổi dữ liệu là rất quan trọng. Hẳn bạn đọc còn nhớ [kiến trúc fanout với SNS và SQS](/2026/01/25/sqs#fanout) để xử lý sự kiện bất đồng bộ, hay [Kinesis Data Stream](/2026/01/31/kinesis-glue#kinesis-data-streams) để xử lý theo thời gian thực. 

Nếu sự kiện là các thay đổi dữ liệu trong DynamoDB, ngoài Kinesis Data Stream, có một tính năng ưu việt hơn được tích hợp sẵn, đó là **DynamoDB Stream**. 


<a name = "mechanism"></a>

### 1.1. Cơ chế Hoạt động


Nếu bật DynamoDB Stream cho một bảng, mỗi khi một item trong bảng được cập nhật (thêm, sửa, xóa), DynamoDB Stream sẽ ghi lại sự thay đổi đó. Mỗi sự thay đổi được gọi là một **record**, được sắp xếp theo thời gian, và giữ lại trong vòng 24 giờ. Consumer (phổ biến nhất là [Lambda Function](/2026/01/18/lambda-fundamental)) có thể đọc các record trong Stream để xử lý tiếp.

Đặc điểm:

- **Thời gian thực:** dữ liệu thay đổi được đưa vào Stream gần như ngay lập tức.
- **Thứ tự:** các thay đổi của cùng một item được sắp xếp theo đúng trình tự thời gian.
- **Duy nhất:** mỗi thay đổi chỉ được ghi lại một lần duy nhất.
- **Hiệu năng**: Stream chạy tách biệt hoàn toàn, không ảnh hưởng đến [RCU/WCU](/2026/03/22/dynamodb-index#capacity) của bảng chính.


<a name = "view-type"></a>

### 1.2. View Type

Là cách chỉ định thông tin nào được ghi vào record trong Stream. Có 4 loại:
- **KEYS_ONLY**: chỉ ghi khoá chính của item thay đổi.
- **NEW_IMAGE**: ghi toàn bộ item **sau khi thay đổi**.
- **OLD_IMAGE**: ghi toàn bộ item **trước khi thay đổi**.
- **NEW_AND_OLD_IMAGES**: ghi cả item **trước và sau khi thay đổi**.

Trong [ví dụ ở bài DynamoDB](/2026/03/18/dynamodb#update), nếu bật Stream cho bảng với view type là `NEW_AND_OLD_IMAGES`, khi chạy lệnh `update-item` như ở ví dụ đó để cập nhật thuộc tính `Status` từ `Pending` sang `Delivered`, một Stream record định dạng JSON được tạo ra như sau:

```json
{
    "Records": [
        {
            "eventID": "1",
            "eventName": "MODIFY",
            "dynamodb": {
                "ApproximateCreationDateTime": 1740921600,
                "Keys": {
                    "OrderType": {"S": "Electronics"},
                    "Date": {"S": "2026-01-01"}
                },
                "OldImage": {
                    "OrderType": {"S": "Electronics"},
                    "Date": {"S": "2026-01-01"},
                    "Status": {"S": "Pending"},
                    "Amount": {"N": "500"}
                },
                "NewImage": {
                    "OrderType": {"S": "Electronics"},
                    "Date": {"S": "2026-01-01"},
                    "Status": {"S": "Delivered"},
                    "Amount": {"N": "500"}
                },
                "SequenceNumber": "123456789",
                "SizeBytes": 150,
                "StreamViewType": "NEW_AND_OLD_IMAGES"
            },
            "awsRegion": "us-east-1",
            "eventSource": "aws:dynamodb"
        }
    ]
}
```

Với record này, Lambda Function có thể dễ dàng lấy thông tin về item trước và sau khi thay đổi, rồi thực hiện các hành động phù hợp, như tổng hợp, phân tích, đồng bộ dữ liệu, gửi thông báo, v.v.

Bạn đọc có thể tham khảo thêm cách kết hợp DynamoDB Stream với Lambda Function trong [tài liệu chính thức của AWS](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.Lambda.Tutorial.html).




<a name = "global-table"></a>

## 2. DynamoDB Global Table

Như đã giới thiệu trong [bài trước](/2026/03/22/dynamodb-index#consistency), một bảng DynamoDB thông thường chỉ tồn tại trong một Region, với các Storage Node tại các AZ. Nếu muốn giảm độ trễ khi phục vụ người dùng ở nhiều Region khác nhau, hoặc muốn tăng khả năng chịu lỗi và phục hồi sau thảm hoạ, cần sử dụng **DynamoDB Global Table**.

Sau khi đã tạo bảng (nhưng chưa có dữ liệu nào), có thể sử dụng lệnh `update-table` để bật tính năng Global Table như sau:

```bash
aws dynamodb update-table \
    --table-name AwsCobanOrders \
    --replica-updates '[{"Create": {"RegionName": "us-east-2"}}]' \
    --multi-region-consistency STRONG \
    --region us-east-1
```

Trong đó:
- `--replica-updates` chỉ định các Region muốn tạo (*create*), cập nhật (*update*), hoặc xoá (*delete*) bản sao của bảng chính.
- `--multi-region-consistency` chỉ định chế độ nhất quán cho các bản sao, có thể là `STRONG` hoặc `EVENTUAL`. Mặc định là `EVENTUAL`.



Khác với [Aurora Global Database](/2026/01/15/aurora#aurora-global-db) chỉ có bản sao tại Region chính cho phép ghi, DynamoDB Global Table **cho phép ghi dữ liệu tại tất cả Region** (gọi là **multi-master write**). Khi một item được ghi vào bản sao ở bất kỳ Region nào, nó sẽ tự động đồng bộ đến tất cả các Region khác. Quá trình đồng bộ này dựa trên [DynamoDB Stream](#stream) đã đề cập ở trên, với độ trễ dưới 1 giây.


Trong một số trường hợp, có thể xảy ra xung đột, khi cùng một item trong bảng được ghi gần như đồng thời ở nhiều Region. Khi đó quy tắc giải quyết xung đột là **Last Writer Wins** (Ai ghi cuối sẽ thắng).



<a name = "dax"></a>

## 3. DynamoDB Accelerator (DAX)

DynamoDB vốn đã cung cấp độ trễ rất thấp (single-digit millisecond). Nhưng một vài ứng dụng yêu cầu độ trễ còn thấp hơn nữa, cỡ **microsecond**. Trong những trường hợp đó, có thể sử dụng **DynamoDB Accelerator (DAX)**, một loại bộ nhớ đệm (*caching*) được thiết kế đặc biệt cho DynamoDB.



<p>
<image src="/assets/41_dynamodb_mics/dax.png" alt="DAX" style="max-width:80%;height:auto;display:block;margin:0 auto;"/>
</p>

DAX bản chất là một cụm máy chủ chạy bộ nhớ đệm lưu trên RAM (*in-memory cache*). Được triển khai trên cùng [VPC](/2025/11/13/vpc) với ứng dụng, mỗi node trong cụm DAX được đặt trên 1 AZ, một node được chọn làm node chính (*primary node*) để xử lý tất cả các yêu cầu đọc/ghi. Dữ liệu trong node chính được đồng bộ với các node khác (*replica node*), đảm bảo tính sẵn sàng cao. 


DAX hoạt động như một lớp trung gian giữa ứng dụng và DynamoDB. Ứng dụng làm việc trực tiếp với DAX thay vì DynamoDB, DAX sẽ xử lý yêu cầu từ ứng dụng, nếu cần thiết sẽ tương tác với DynamoDB và trả kết quả về. Cụ thể như sau:


<a name = "dax-read"></a>

### 3.1. Đọc

Cơ chế đọc của DAX là **read-through**. Khi có yêu cầu đọc (bằng các lệnh như `get-item`, `batch-get-item`, `query`, `scan`), DAX sẽ xử lý như sau:

- Nếu là **Eventually Consistent Read** (chế độ mặc định), DAX sẽ cố gắng đọc item từ bộ nhớ đệm của mình:
    - **Cache Hit:** DAX có sẵn item, trả về kết quả ngay lập tức.
    - **Cache Miss:** DAX không có item, chuyển yêu cầu tới DynamoDB. Khi nhận được phản hồi từ DynamoDB, DAX trả kết quả cho ứng dụng, đồng thời ghi kết quả đó vào bộ nhớ đệm tại node chính để phục vụ các lần đọc sau.

- Nếu là **Strongly Consistent Read**, DAX sẽ chuyển trực tiếp yêu cầu tới DynamoDB. Trong trường hợp này, kết quả từ DynamoDB **không được lưu vào cache** của DAX mà trả thẳng về cho ứng dụng. **DAX không hỗ trợ Strongly Consistent Read** vì nó không thể đảm bảo dữ liệu trong cache luôn là mới nhất, do đó nếu ứng dụng yêu cầu dữ liệu nhất quán mạnh, không cần phải dùng DAX.


<a name = "dax-write"></a>

### 3.2. Ghi

Với các yêu cầu ghi như `put-item`, `batch-write-item`, `update-item`, `delete-item`, cơ chế là **write-through**: đầu tiên dữ liệu sẽ được ghi vào DynamoDB, sau đó DAX sẽ cập nhật bộ nhớ đệm của mình với dữ liệu mới.

Nếu bạn đọc cần ôn lại các chiến lược caching như read-through, write-through, v.v., có thể tham khảo thêm [bài viết này trên Viblo](https://viblo.asia/p/chien-luoc-caching-caching-strategies-zXRJ8jPOVGq).



<a name = "backup-restore"></a>

## 4. Sao lưu và Phục hồi 

DynamoDB cung cấp 2 tính năng sao lưu và phục hồi dữ liệu:


<a name = "ondemand-backup"></a>

### 4.1. On-Demand Backup

Nghĩa là **Sao lưu theo yêu cầu**. Khi cần, người dùng có thể tạo một bản sao lưu của bảng bất kỳ lúc nào, và lưu trữ bản sao lưu đó trong S3. Bản sao lưu này sẽ giữ nguyên trạng thái của bảng tại thời điểm sao lưu, và có thể được phục hồi lại bất cứ lúc nào.


Có thể tạo bản sao trên giao diện, hoặc sử dụng lệnh CLI sau:

```bash
aws dynamodb create-backup --table-name AwsCobanOrders \
 --backup-name AwsCobanOrders-Backup
```


Để phục hồi bảng từ bản sao lưu, sử dụng 2 lệnh sau:

```bash
aws dynamodb list-backups # để lấy `backup-arn` của bản sao lưu

aws dynamodb restore-table-from-backup \
--target-table-name AwsCobanOrders \
--backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/AwsCobanOrders/backup/01489173575360-b308cd7d 
```

<a name = "pitr"></a>

### 4.2. Point-in-Time Recovery (PITR)

Nếu bật tính năng này, DynamoDB sẽ tự động sao lưu dữ liệu của bảng. Khi cần, có thể phục hồi bảng về bất kỳ thời điểm nào trong một khoảng thời gian định sẵn (từ 1 đến 35 ngày). 

Ví dụ, lệnh dưới đây sẽ bật tính năng Point-in-Time Recovery cho bảng `AwsCobanOrders`, cho phép phục hồi dữ liệu trong vòng 30 ngày:

```bash
aws dynamodb update-continuous-backups \
--table-name AwsCobanOrders \
--point-in-time-recovery-specification PointInTimeRecoveryEnabled=true,RecoveryPeriodInDays=30
```


<a name = "ttl"></a>

## 5. Time to Live (TTL)

Đây là tính năng giúp tự động xoá các item đã cũ không còn giá trị. Cơ chế hoạt động rất đơn giản:

- Tạo một thuộc tính trong bảng, có kiểu dữ liệu là `Number`, dùng để lưu trữ thời điểm hết hạn của item dưới dạng [UNIX timestamp](https://en.wikipedia.org/wiki/Unix_time).
- Bật tính năng TTL cho bảng, chỉ định thuộc tính đã tạo ở bước trên làm thuộc tính TTL.
- DynamoDB chạy một tiến trình ngầm để quét các item. Nếu thời gian hiện tại lớn hơn giá trị timestamp trong thuộc tính TTL, item đó sẽ bị đánh dấu là hết hạn. 
- Các item hết hạn sẽ được xoá tự động trong một khoảng thời gian (thường trong vài ngày sau khi bị đánh dấu). Item sẽ bị xoá khỏi bảng chính, [GSI](/2026/03/22/dynamodb-index#gsi), và [LSI](/2026/03/22/dynamodb-index#lsi).

Thường dùng TTL nếu dữ liệu chỉ là tạm thời, như phiên làm việc, đơn hàng đã hoàn thành từ lâu, log lỗi đã xử lý xong, v.v. 
Điều quan trọng nhất là việc xoá bằng TTL **không tiêu tốn WCU** của bảng, khác với việc xoá bằng `delete-item`, nên không tốn chi phí nào.





<a name = "reference"></a>

## Tài liệu tham khảo

1. [DynamoDB Streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html)
2. [Kết hợp DynamoDB Streams với Lambda](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.Lambda.html)
3. [DynamoDB Global Table](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/globaltables-CoreConcepts.html)
4. [Cách Hoạt động của DAX](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.concepts.html)
5. [Các chiến lược Caching](https://viblo.asia/p/chien-luoc-caching-caching-strategies-zXRJ8jPOVGq)
6. [TTL trong DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)


Trong bài này, ta đã đề cập đến DAX là một giải pháp caching cho DynamoDB. Với các dịch vụ CSDL khác như [RDS](/2026/01/08/rds-fundamental), [Aurora](/2026/01/15/aurora), có thể sử dụng **ElasticCache**, sẽ được giới thiệu trong bài tiếp theo.
