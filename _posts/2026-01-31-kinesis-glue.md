---
layout: post
title: "28. Kinesis và Glue"
title2: "Kinesis và Glue"
date: 2026-01-31 00:00:00 +0700
permalink: /2026/01/31/kinesis-glue
categories: [Serverless]
tags: [Serverless]
img: /assets/28_kinesis_glue/glue.png
summary: "Tiếp tục series về các dịch vụ không máy chủ, bài này giới thiệu về Amazon Kinesis và AWS Glue. Kinesis xử lý streaming data thời gian thực rất mạnh, còn Glue giúp xây dựng luồng ETL (Extract, Transform, Load) để xử lý và chuẩn bị dữ liệu cho các tác vụ phía sau."
---

Tiếp tục series về các dịch vụ không máy chủ, bài này giới thiệu về Amazon Kinesis và AWS Glue. Kinesis xử lý streaming data thời gian thực rất mạnh, còn Glue giúp xây dựng luồng ETL (Extract, Transform, Load) để xử lý và chuẩn bị dữ liệu cho các tác vụ phía sau.

## Trong bài này:

- [1. Amazon Kinesis](#kinesis)
    - [1.1. Kinesis Data Streams](#kinesis-data-streams)
    - [1.2. Kinesis Data Firehose](#kinesis-data-firehose)
    - [1.3. Kinesis Video Streams](#kinesis-video-streams)
- [2. AWS Glue](#glue)
- [Tài liệu tham khảo](#reference)

<a name = "kinesis"></a>

## 1. Amazon Kinesis

Kinesis là tổ hợp dịch vụ xử lý dữ liệu streaming thời gian thực của AWS, bao gồm ba dịch vụ chính: Data Streams, Data Firehose và Video Streams.


<a name = "kinesis-data-streams"></a>

### 1.1. Kinesis Data Streams

<p>
<image src="/assets/28_kinesis_glue/kinesis-data-streams.png" alt="API Gateway" style="max-width:90%;height:auto;display:block;margin:0 auto;"/>
</p>


Kinesis Data Streams (KDS) giúp thu thập dữ liệu streaming theo thời gian thực, có hai thành phần chính: **producer** gửi dữ liệu streaming đến một stream, và **consumer** đọc và xử lý dữ liệu từ stream đó.

Nghe khá quen, phải không? Cơ chế này nghe rất giống [SQS](/2026/01/25/sqs), vì cùng lưu trữ dữ liệu tạm thời để xử lý. Tuy nhiên, khác với SQS:

- KDS dùng cho việc xử lý dữ liệu streaming thời gian thực, yêu cầu kỹ thuật sẽ cao hơn nhiều so với hàng đợi SQS dùng cho giao tiếp bất đồng bộ.
- KDS cho phép nhiều ứng dụng đọc dữ liệu từ cùng một stream, trong khi một hàng đợi SQS chỉ cho phép một consumer lấy và xử lý tin nhắn.


**Các thuật ngữ chính**:

- **Stream**: một kênh chứa dữ liệu streaming gửi vào từ các producer.
- **Shard**: mỗi stream được chia thành một hoặc nhiều shard, mỗi shard chứa một chuỗi các bản ghi dữ liệu (data record, trình bày ngay dưới đây). Tại thời điểm viết bài, mỗi shard có tốc độ ghi tối đa 1 MB/s, tốc độ đọc tối đa 2 MB/s. Khi tải thay đổi, ta có thể [tăng hoặc giảm số lượng shard](https://docs.aws.amazon.com/streams/latest/dev/kinesis-using-sdk-java-resharding.html) để tối ưu khả năng xử lý và chi phí. 

*Nếu bạn đang ôn thi chứng chỉ AWS, ghi nhớ*: **merge cold shard** để giảm số lượng shard hay giảm tải, và **split hot shard** để tăng số lượng shard khi cần tăng tải.

- **Data Record**: là đơn vị dữ liệu nhỏ nhất trong KDS, bao gồm một [partition key](https://docs.aws.amazon.com/streams/latest/dev/key-concepts.html#partition-key) để xác định shard lưu trữ nó, một [sequence number](https://docs.aws.amazon.com/streams/latest/dev/key-concepts.html#sequence-number) để xác định vị trí của nó trong shard, và dữ liệu thực tế, **tối đa 1 MB**. 


Consumer của một Stream thường là một cụm [EC2](/2025/12/16/ec2-fundamental), đầu ra có thể được gửi đến các dịch vụ khác như [S3](/2025/11/30/s3-fundamental), [Lambda](/2026/01/18/lambda-fundamental), Redshift, v.v, để lưu trữ hoặc xử lý tiếp.
<!-- TODO: include link to Redshift -->


<a name = "kinesis-data-firehose"></a>

### 1.2. Kinesis Data Firehose

Đây là dịch vụ giúp **chuyển tiếp** dữ liệu streaming theo thời gian thực tới các dịch vụ đích như S3, Redshift, OpenSearch, hoặc các endpoint HTTP/HTTPS bên ngoài. Người dùng cấu hình producer gửi dữ liệu đến Firehose, Firehose tự động chuyển đổi dữ liệu (nếu cần, sử dụng Lambda) rồi chuyển tiếp đến đích.
<!-- TODO: include link to OpenSearch -->

<p>
<image src="/assets/28_kinesis_glue/kinesis-firehose.png" alt="Kinesis Data Firehose" style="max-width:90%;height:auto;display:block;margin:0 auto;"/>
</p>

Firehose thường được sử dụng để xây dựng data lake, data warehouse, hoặc trong các ứng dụng phân tích dữ liệu thời gian thực. Hơn hết, Firehose hoàn toàn serverless, tự động mở rộng theo nhu cầu sử dụng.


<a name = "kinesis-video-streams"></a>

### 1.3. Kinesis Video Streams

Kinesis Video Streams (KVS) là ứng dụng giúp truyền dữ liệu video trực tiếp từ camera lên AWS. Dữ liệu này sau đó có thể được xem trực tiếp, hoặc đưa vào các ứng dụng AI như Amazon Rekognition hoặc AWS SageMaker để phân tích theo thời gian thực, nhận diện khuôn mặt, phát hiện đối tượng, phân tích hành vi, v.v.
<!-- TODO: include link to Rekognition, SageMaker  -->

<p>
<image src="/assets/28_kinesis_glue/kinesis-video-streams.png" alt="Kinesis Video Streams" style="max-width:90%;height:auto;display:block;margin:0 auto;"/>
</p>

Một ứng dụng kinh điển kết hợp KVS và Rekognition là giám sát an ninh, cảnh báo nhận diện người lạ. Dữ liệu video từ camera được truyền lên KVS, rồi tích hợp với Rekognition để nhận diện khuôn mặt, so sánh với một bộ dữ liệu khuôn mặt đã được thu thập sẵn.
Đầu vào của Rekognition là các khung hình video từ KVS, nên đầu ra (là người lạ hay không, thời điểm nào) cũng là một luồng dữ liệu thời gian thực, do đó ta đưa chúng vào một Kinesis Data Stream để chờ xử lý tiếp. Một Lambda Function được dùng để xử lý dữ liệu từ KDS, rồi gửi cảnh báo đến một [SNS Topic](/2026/01/22/step-function#sns) mà quản trị viên đã đăng ký nhận thông báo. SNS sẽ lo phần việc gửi thông báo qua email, SMS, hoặc các kênh khác.


<a name = "glue"></a>

## 2. AWS Glue

Glue là dịch vụ ETL (Extract - Transform - Load) không máy chủ của AWS. ETL là quy trình thu thập dữ liệu từ nhiều nguồn (*Extract*), xử lý, biến đổi dữ liệu theo yêu cầu (*Transform*), rồi tải dữ liệu đã xử lý vào kho dữ liệu đích (*Load*), hay dùng trong các tác vụ phân tích, báo cáo, học máy, v.v.

<p>
<image src="/assets/28_kinesis_glue/glue.png" alt="AWS Glue" style="max-width:90%;height:auto;display:block;margin:0 auto;"/>
</p>

**Các thành phần chính của Glue**:

- **Data Catalog**: lưu trữ metadata định nghĩa cấu trúc các bảng dữ liệu, các thông tin về quy trình ETL.
- **Crawler**: là các chương trình tự động quét các nguồn dữ liệu, suy luận cấu trúc dữ liệu, thu thập metadata và cập nhật vào Data Catalog.
- **Glue Job**: thành phần quan trọng nhất, là các tác vụ ETL thực tế. Người dùng định nghĩa các bước thu thập, biến đổi dữ liệu và đưa về kho chứa đích.
- **Trigger**: tác nhân khởi chạy Glue Job, có thể chạy theo nhu cầu, theo lịch trình định sẵn, hoặc dựa trên các sự kiện. Thường tích hợp với Event Bridge.
<!-- TODO: include link to Event Bridge  -->

Thông thường quy trình làm việc với Glue như sau:

1. Định nghĩa nguồn và đích của dữ liệu trong Data Catalog.
2. Dùng Crawler để quét các kho dữ liệu nguồn, thu thập metadata và cập nhật vào Data Catalog.
3. Tạo Glue Job để định nghĩa quy trình ETL. Viết mã nguồn.
4. Tạo Trigger để khởi chạy Glue Job theo nhu cầu.
5. Giám sát và quản lý các Glue Job qua giao diện.


<a name = "reference"></a>

## Tài liệu tham khảo

1. [Các Thuật ngữ trong Kinesis Data Streams](https://docs.aws.amazon.com/streams/latest/dev/key-concepts.html)
2. [Thay đổi Số lượng Shard trong Kinesis Data Streams](https://docs.aws.amazon.com/streams/latest/dev/kinesis-using-sdk-java-resharding.html)
3. [Kinesis Data Firehose](https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html)
4. [Kinesis Video Streams](https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/what-is-kinesis-video.html)
5. [AWS Glue](https://docs.aws.amazon.com/glue/latest/dg/components-key-concepts.html)

Trên đây là phần giới thiệu cơ bản về Amazon Kinesis và AWS Glue, đủ dùng để ôn tập cho các kỳ thi chứng chỉ AWS. Bạn đọc vẫn cần thực hành thực tế để hiểu sâu hơn về các dịch vụ này. Mình cũng xin tạm dừng series Serverless tại đây, tiếp theo hãy quay lại EC2 để xem xét các dịch vụ giúp tăng tính khả dụng và khả năng phục hồi. 