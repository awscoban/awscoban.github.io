---
layout: post
title: "42. ElastiCache"
title2: "ElastiCache"
date: 2026-04-06
permalink: /2026/04/06/elasticache
categories: [Caching]
tags: [Caching]
img: /assets/42_elasticache/elasticache.png
summary: "ElastiCache là một dịch vụ lưu trữ dữ liệu trong bộ nhớ (in-memory data store), thường dùng làm bộ nhớ đệm (cache) để giảm độ trễ và giảm tải cho cơ sở dữ liệu chính. Bài này trình bày tổng quan về kiến trúc, vai trò, và sự khác nhau giữa các loại Engine như Redis/Valkey và Memcached."
---

ElastiCache là một dịch vụ lưu trữ dữ liệu trong bộ nhớ (in-memory data store), thường dùng làm bộ nhớ đệm (cache) để giảm độ trễ và giảm tải cho cơ sở dữ liệu chính. Bài này trình bày tổng quan về kiến trúc, vai trò, và sự khác nhau giữa các loại Engine như Redis/Valkey và Memcached. 


## Trong bài này:

- [1. Tổng quan](#overview)
- [2. Các Khái niệm Cơ bản](#concepts)
- [3. Redis vs Memcached](#redis-vs-memcached)
- [4. Câu hỏi Ôn tập](#question)
- [Tài liệu tham khảo](#reference)


<a name = "overview"></a>

## 1. Tổng quan

Như tên gọi, ElastiCache được dùng làm bộ nhớ đệm (*cache*). Mục đích của cache nói chung là lưu trữ dữ liệu tạm thời để **giảm độ trễ** và **giảm tải cho CSDL chính**, đặc biệt là khi ứng dụng chịu tải đọc lớn (*read-heavy*).


Khác với [DAX](/2026/03/26/dynamodb-mics#dax) là cache được thiết kế riêng cho [DynamoDB](/2026/03/18/dynamodb), ElastiCache là dịch vụ cache tổng quát hơn, có thể dùng với nhiều loại CSDL khác nhau như [RDS](/2026/01/08/rds-fundamental), [Aurora](/2026/01/15/aurora), hoặc thậm chí là dùng thay CSDL trong một số trường hợp.


Ở [bài trước](/2026/03/26/dynamodb-mics#dax), ta đã biết cơ chế cache của DAX là *write-through*, dữ liệu được ghi vào cache ngay sau khi ghi vào CSDL. ElastiCache cũng hỗ trợ **write-through**, nhưng phổ biến hơn là **lazy loading** (còn gọi là **cache-aside**), với cơ chế như hình sau:


<p>
<image src="/assets/42_elasticache/elasticache.png" alt="ElastiCache" style="max-width:90%;height:auto;display:block;margin:0 auto;"/>
</p>


- Khi người dùng yêu cầu dữ liệu, ứng dụng sẽ kiểm tra cache trước. Nếu dữ liệu đã có trong cache (**cache hit**), trả về dữ liệu từ cache. 
- Nếu dữ liệu không có trong cache (**cache miss**), truy vấn CSDL, sau đó lưu kết quả vào cache để phục vụ các yêu cầu tiếp theo, rồi trả về dữ liệu cho người dùng.

Tên gọi **lazy loading** xuất phát từ việc dữ liệu chỉ được lưu vào cache khi có yêu cầu truy cập, không phải được lưu ngay khi ghi vào CSDL như *write-through*. Còn tên gọi khác là **cache-aside** (tạm dịch: *cache nằm ở một bên*) là do ở đây cache không phải trung gian giữa ứng dụng và CSDL như *write-through*, mà là một thành phần khác mà ứng dụng phải tương tác ngoài CSDL.

Bạn đọc có thể đọc thêm các chiến lược caching khác trong [bài viết này](https://viblo.asia/p/chien-luoc-caching-caching-strategies-zXRJ8jPOVGq).


<a name = "concepts"></a>

## 2. Các Khái niệm Cơ bản

- **Engine**: ElastiCache hỗ trợ 3 engine chính: **Redis**, **Valkey** và **Memcached**, đây là 3 phần mềm lưu trữ dữ liệu trong RAM phổ biến. Trong đó Redis và Valkey có thiết kế và tính năng tương tự nhau, ở đây mình sẽ lấy Redis làm đại diện.

- **Cluster** & **Node**: ElastiCache thường được tổ chức thành các cụm (**cluster**), mỗi cluster có một hoặc nhiều **node**. Mỗi node là một máy ảo chạy cùng Engine, cùng cấu hình RAM, CPU. 

- **Shard**: chỉ cho Redis/Valkey, là một **nhóm 1-6 node**, cluster có thể chứa nhiều shard. Trong mỗi shard, một node sẽ được chỉ định làm **primary node** (hỗ trợ cả đọc/ghi), các node còn lại là **replica**, đồng bộ dữ liệu từ primary node nhưng chỉ hỗ trợ đọc.





<a name = "redis-vs-memcached"></a>

## 3. Redis vs Memcached


Trong các đề thi chứng chỉ AWS, nội dung thường được hỏi về ElastiCache là chọn loại Engine phù hợp với yêu cầu đặt ra. Dưới đây là bảng so sánh các đặc điểm chính:

| Đặc điểm | **Redis** | **Memcached** |
| :--- | :--- | :--- |
| **Kiểu dữ liệu** | Phức tạp (List, Set, HashMap, Geospatial) | Đơn giản (String, Object) |
| **Kiến trúc luồng** | Đơn luồng (Single-threaded nhưng cực nhanh) | Đa luồng (**Multi-threaded**) |
| **Khả năng sẵn sàng** | Cao (hỗ trợ Multi-AZ, Replication, Failover tự động) | Thấp (không replication, không backup) |
| **Trường hợp sử dụng** | Leaderboard, Pub/Sub, Chat, Caching phức tạp | Caching đơn giản, giảm tải DB cơ bản |

<p></p>

**Redis** có nhiều tính năng hơn, hỗ trợ nhiều kiểu dữ liệu **phức tạp**. Ngược lại, chỉ dùng **Memcached** cho những trường hợp **đơn giản**.
Chỉ có một tính năng hơi ngược logic này là nếu cần thiết kế bộ đệm **đa luồng** (**multi-threaded**), thì phải chọn Memcached. Bạn đọc nên nhớ kỹ điểm này, đây là một điểm hay bị đánh lừa trong các câu hỏi thi.



<a name = "question"></a>

## 4. Câu hỏi Ôn tập


> **Question:** A financial services company is optimizing its high-frequency trading platform. The backend requires an in-memory key-value data store that supports multi-threaded execution to leverage large instance types with multiple CPU cores. The solution must provide sub-millisecond latency for database query results and must be able to scale its capacity elastically as market volatility increases or decreases throughout the day.
> 
> Which of the following is the MOST suitable service for this requirement?
>
> *   **A.** CloudFront
> *   **B.** ElastiCache for Memcached
> *   **C.** ElastiCache for Redis
> *   **D.** DynamoDB Accelerator (DAX)

Tạm dịch:
> **Câu hỏi:** Một công ty dịch vụ tài chính muốn tối ưu hóa nền tảng giao dịch tần suất cao. Backend yêu cầu một in-memory key-value data store, hỗ trợ thực thi đa luồng để tận dụng các loại instance lớn với nhiều lõi CPU. Giải pháp phải cung cấp độ trễ dưới 1ms cho kết quả truy vấn cơ sở dữ liệu và phải có khả năng mở rộng dung lượng một cách linh hoạt khi biến động thị trường tăng hoặc giảm trong suốt cả ngày.
> 
> Dịch vụ nào là phù hợp nhất cho yêu cầu này?

Các từ khoá trong câu hỏi này là:
- **In-memory key/value data store**: là đặc điểm của ElastiCache.
- **Multi-threaded execution**: chỉ có Memcached hỗ trợ đa luồng, Redis là đơn luồng.

Như vậy, đáp án đúng là **B**.
- **A** sai vì [CloudFront](/2026/02/20/cloudfront) là CDN, tuy cũng là một dịch vụ caching, nhưng không phải in-memory key/value data store.
- **C** sai vì Redis là đơn luồng, không hỗ trợ đa luồng.
- **D** sai vì [DAX](/2026/03/26/dynamodb-mics#dax) là cache chuyên dụng cho DynamoDB. Đề bài không đề cập đến DynamoDB.




<a name = "reference"></a>

## Tài liệu tham khảo

1. [Tổng quan về ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.corecomponents.html)
2. [Các Khái niệm Cơ bản](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.Components.html)
3. [Các chiến lược Caching](https://viblo.asia/p/chien-luoc-caching-caching-strategies-zXRJ8jPOVGq)
4. [So sánh Redis và Memcached](https://aws.amazon.com/elasticache/redis-vs-memcached/)



Trên đây là những kiến thức về ElastiCache hay gặp trong các kỳ thi chứng chỉ AWS. Bạn đọc có thể tham khảo thêm tài liệu chính thức của AWS để hiểu rõ hơn cách sử dụng dịch vụ này trên thực tế. Tiếp theo, hãy chuyển sang một chủ đề lớn khác: làm việc trong môi trường hybrid với hạ tầng ở cả on-premise và trên AWS, bắt đầu với VPN và Direct Connect.

