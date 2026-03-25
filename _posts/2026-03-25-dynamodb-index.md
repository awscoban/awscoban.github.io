---
layout: post
title: "40. DynamoDB: Capacity Unit và Index"
title2: "DynamoDB: Capacity Unit và Index"
date: 2026-03-25
permalink: /2026/03/25/dynamodb-index
categories: [Database, DynamoDB, Serverless]
tags: [Database, DynamoDB, Serverless]
img: /assets/40_dynamo_db_index/dynamodb-consistency.png
summary: "Bài trước đã giới thiệu các khái niệm cơ bản trong DynamoDB, cách thực hiện các thao tác với dữ liệu và cách thiết kế khoá chính. Trong bài này, hãy tìm hiểu về cách tính hiệu năng và chi phí của DynamoDB thông qua Read/Write Capacity Unit, cũng như cách sử dụng 2 loại Secondary Index để tối ưu truy vấn."
---

Ở bài trước, mình đã giới thiệu các khái niệm cơ bản trong DynamoDB, cách thực hiện các thao tác với dữ liệu và cách thiết kế khoá chính. Trong bài này, hãy tìm hiểu về cách tính hiệu năng và chi phí của DynamoDB thông qua *Read/Write Capacity Unit*, cũng như cách sử dụng 2 loại Secondary Index để tối ưu truy vấn.


## Trong bài này:

- [1. Consistency Model của DynamoDB](#consistency)
- [2. Capacity Unit](#capacity)
    - [2.1. Read Capacity Unit (RCU)](#rcu)
    - [2.2. Write Capacity Unit (WCU)](#wcu)
    - [2.3. Ví dụ](#example)
- [3. Index trong DynamoDB](#index)
    - [3.1. Global Secondary Index (GSI)](#gsi)
    - [3.2. Local Secondary Index (LSI)](#lsi)
- [Tài liệu tham khảo](#reference)


Nhưng trước hết, ta cần nắm được một khái niệm quan trọng.


<a name = "consistency"></a>

## 1. Consistency Model của DynamoDB

Với các hệ thống CSDL phân tán nói chung, **mô hình nhất quán** (**_consistency model_**) là khái niệm quan trọng. Các máy chủ được phân phối trên nhiều vùng địa lý, người dùng thường sẽ đọc/ghi dữ liệu trên các máy chủ khác nhau, nên có thể xảy ra tình trạng dữ liệu không đồng bộ. Mô hình nhất quán là quy tắc xác định khi nào sự thay đổi dữ liệu ở một máy chủ sẽ được phản ánh ở các máy chủ khác.

Đây là một chủ đề rất rộng, bạn đọc nếu muốn tìm hiểu sâu hơn, có thể tra cứu với từ khoá [CAP Theorem](https://en.wikipedia.org/wiki/CAP_theorem). Trong phạm vi bài này, ta chỉ cần biết rằng, DynamoDB cung cấp hai loại nhất quán: 

- **Eventually Consistent Read**: dịch ra là "*cuối cùng rồi sẽ nhất quán*", tức có xác suất trả về dữ liệu cũ (do mới được ghi vào một máy chủ, và chưa kịp đồng bộ sang các máy khác), nhưng sau một khoảng thời gian ngắn, dữ liệu trên các máy chủ sẽ nhất quán.
- **Strongly Consistent Read**: dịch ra là "*nhất quán mạnh*", tức dữ liệu luôn là mới nhất, cập nhật nhất.



<p>
<image src="/assets/40_dynamo_db_index/dynamodb-consistency.png" alt="DynamoDB Consistency Model" style="max-width:100%;height:auto;display:block;margin:0 auto;"/>
</p>

Trong hình trên, DynamoDB có 3 node lưu trữ tại 3 AZ. Một node sẽ được chọn là **Leader**. Khi có một thao tác ghi, dữ liệu sẽ được ghi vào Leader. Sau đó, dữ liệu được đồng bộ sang 2 node còn lại trong thời gian cực ngắn (milisecond). 

Khi có yêu cầu đọc:

- Nếu người gửi yêu cầu đọc chọn chế độ **Eventually Consistent Read**, thì dữ liệu sẽ được **đọc từ một node ngẫu nhiên** (có thể là Leader hoặc node thông thường). Do đó, nếu node thông thường được chọn, có xác suất dữ liệu ở node này chưa được đồng bộ, nên trả về dữ liệu cũ. Tuy nhiên, do thời gian đồng bộ rất ngắn, và chi phí đọc theo chế độ này rẻ hơn một nửa, nên thường đây là lựa chọn chấp nhận được, và thực tế Eventually Consistent Read là mặc định.
- Nếu chọn **Strongly Consistent Read**, thì dữ liệu sẽ được **đọc từ node Leader**. Đây là node phục vụ tác vụ ghi, nên luôn có dữ liệu mới nhất. Tất nhiên, chi phí đọc theo chế độ này sẽ cao hơn, và độ trễ có thể cao hơn một chút. Người dùng chọn chế độ này bằng cách thêm tham số `--consistent-read` vào các lệnh đọc như `get-item` hoặc `query`.



<a name = "capacity"></a>

## 2. Capacity Unit 


DynamoDB tính phí dựa trên số lượng **Read Capacity Unit (RCU)** sử dụng khi đọc và **Write Capacity Unit (WCU)** sử dụng khi ghi. Cụ thể như sau. Khi tạo bảng, ngoài việc đặt `--billing-mode PAY_PER_REQUEST`, ta có thể chọn `--billing-mode PROVISIONED` để đặt số lượng RCU và WCU cần thiết cho bảng, nếu có thể ước tính được tải của ứng dụng. Cách này sẽ rẻ hơn so với `PAY_PER_REQUEST`.


<a name = "rcu"></a>

### 2.1. Read Capacity Unit (RCU)

1 RCU đại diện cho:
- **2 lần đọc** mỗi giây theo chế độ **Eventually Consistent Read** với mỗi **4 KB** dữ liệu. 
- **1 lần đọc** mỗi giây theo chế độ **Strongly Consistent Read** với mỗi **4 KB** dữ liệu. 

Với cả 2 chế độ, nếu dữ liệu nhỏ hơn 4 KB, ta làm tròn lên 4 KB. Nếu lớn hơn 4 KB, sẽ tính thêm RCU cho mỗi phần 4 KB (làm tròn lên nếu cần).

**Công thức** tính RCU cần thiết cho Strongly Consistent Read như sau:


\\( \text{RCU}_{SCR} = \text{Number of Reads per Second} \times \left\lceil \frac{\text{Item Size (KB)}}{4} \right\rceil \\)    


Nếu sử dụng **Eventually Consistent Read**, chỉ cần một nửa RCU so với Strongly Consistent Read (chi phí rẻ hơn một nửa), như đã đề cập ở phần trước:

\\( \text{RCU}_{ECR} = \frac{1}{2} \times \text{Number of Reads per Second} \times \left\lceil \frac{\text{Item Size (KB)}}{4} \right\rceil \\)    


<a name = "wcu"></a>

### 2.2. Write Capacity Unit (WCU)

1 WCU đại diện cho **1 lần ghi** mỗi giây với mỗi **1 KB** dữ liệu. Tương tự, nếu dữ liệu nhỏ hơn 1 KB, ta làm tròn lên 1 KB. Nếu lớn hơn 1 KB, sẽ tính thêm WCU cho mỗi phần 1 KB (làm tròn lên nếu cần).

**Công thức** tính WCU cần thiết như sau:

\\( \text{WCU} = \text{Number of Writes per Second} \times \left\lceil \text{Item Size (KB)} \right\rceil \\)

<a name = "example"></a>

### 2.3. Ví dụ

Giả sử ta có một bảng Dynamo DB với kích thước item *trung bình* là **21.3 KB**. Một [Lambda Function](/2026/01/18/lambda-fundamental) gửi **240** yêu cầu **đọc Eventually Consistent Read** mỗi giây, và **80** yêu cầu **ghi** mỗi giây, kích thước trung bình của item cần ghi là **11.4 KB**. Cần tính số lượng RCU và WCU cần thiết cho bảng này.


- Với RCU, kích thước item trong bảng là 21.3 KB, ta làm tròn lên 24 KB (chia hết cho 4). Do yêu cầu là Eventually Consistent Read, số lượng RCU cần thiết là:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; \\( \text{RCU}_{ECR} = \frac{1}{2} \times 240 \times \left\lceil \frac{21.3}{4} \right\rceil = \frac{1}{2} \times 240 \times \frac{24}{4} = 720 \\)

- Với WCU, kích thước item cần ghi là 11.4 KB, ta làm tròn lên 12 KB. Số lượng WCU cần thiết là:
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; \\( \text{WCU} = 80 \times \left\lceil \frac{11.4}{1} \right\rceil = 80 \times 12 = 960 \\)


Các bài toán dạng như vừa rồi rất phổ biến khi thi chứng chỉ, bạn đọc cần ghi nhớ công thức và cẩn thận tránh nhầm lẫn.



<a name = "index"></a>

## 3. Index trong DynamoDB

Ở [bài trước](/2026/03/18/dynamodb#query), ta đã biết muốn truy vấn dữ liệu hiệu quả trong DynamoDB, bắt buộc phải cung cấp **Partition Key**. Nếu muốn truy vấn theo một thuộc tính thông thường, phải dùng lệnh [`scan`](/2026/03/18/dynamodb#scan) để quét toàn bộ bảng, rất chậm và tốn kém. 

Để giải quyết vấn đề này, DynamoDB cung cấp hai loại **Secondary Index**: **Global Secondary Index (GSI)** và **Local Secondary Index (LSI)**. Cả hai loại đều cho phép truy vấn dữ liệu linh hoạt hơn theo thuộc tính khác mà không bị giới hạn bởi khoá chính.


<a name = "gsi"></a>

### 3.1. Global Secondary Index (GSI)   

Đây là loại linh hoạt và phổ biến nhất. Có thể chọn **bất kỳ thuộc tính nào** trong bảng chính làm [Partition Key](/2026/03/18/dynamodb#concepts) và [Sort Key](/2026/03/18/dynamodb#concepts) cho GSI, rồi truy vấn theo GSI. 
Sở dĩ gọi là "Global" là vì với thiết kế này, có thể truy vấn dữ liệu trên toàn bộ bảng chính (gọi là **base table**). 

Một vài điểm quan trọng khác:
- GSI có thể được tạo sau khi tạo bảng chính, và có thể có nhiều GSI trên cùng một bảng.
- GSI lưu trữ một bản sao dữ liệu từ bảng chính, được sao lưu bất đồng bộ. Do đó, **chỉ hỗ trợ Eventually Consistent Read** khi truy vấn, **không hỗ trợ Strongly Consistent Read**.
- GSI có RCU và WCU riêng, độc lập với bảng chính. Khi tạo GSI, cần xác định số lượng RCU và WCU cho GSI, hoặc chọn chế độ `PAY_PER_REQUEST` để tính phí theo mức sử dụng thực tế.


<a name = "lsi"></a>

### 3.2. Local Secondary Index (LSI)


Với LSI, **Partition Key phải giống với bảng chính**, nhưng có thể **chọn một thuộc tính khác làm Sort Key**. Do đó, LSI chỉ cho phép truy vấn dữ liệu trong phạm vi một phân vùng cụ thể của bảng chính, không thể truy vấn trên toàn bộ bảng như GSI. Đây là lý do tại sao gọi là "Local".

Có thể tạo **tối đa 5 LSI** cho mỗi bảng. LSI "gắn" chặt với ổ lưu trữ của bảng chính, chia sẻ cùng Partition Key và các phân vùng vật lý, do đó:

- LSI phải được **tạo cùng lúc** với bảng chính, không thể thêm sau. 
- LSI **hỗ trợ Strongly Consistent Read** khi truy vấn.
- LSI chia sẻ RCU và WCU với bảng chính, không có RCU/WCU riêng như GSI. 









<a name = "reference"></a>

## Tài liệu tham khảo

1. [CAP Theorem](https://en.wikipedia.org/wiki/CAP_theorem)
2. [DynamoDB Read Consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
3. [Chi phí của các Thao Tác Đọc/Ghi](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/read-write-operations.html)
4. [AWS re:Invent 2018: Amazon DynamoDB Under the Hood: How We Built a Hyper-Scale Database](https://www.youtube.com/watch?v=yvBR71D0nAQ)
5. [Global Secondary Index (GSI)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html)
6. [Local Secondary Index (LSI)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/LSI.html)

Trên đây là những kiến thức về RCU/WCU, GSI/LSI hay xuất hiện trong các kỳ thi chứng chỉ. Nếu muốn tìm hiểu sâu hơn, bạn đọc có thể tham khảo các tài liệu trên. Tiếp theo, mình sẽ giới thiệu các tính năng hữu ích khác của DynamoDB như Stream, DAX, Global Table, và TTL.