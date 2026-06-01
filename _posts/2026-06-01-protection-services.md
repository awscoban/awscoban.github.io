---
layout: post
title: "51. Bảo vệ Dữ liệu, Ứng dụng, Hạ tầng trên AWS"
title2: "Bảo vệ Dữ liệu, Ứng dụng, Hạ tầng trên AWS"
date: 2026-06-01
permalink: /2026/06/01/protection-services
categories: [Security]
tags: [Security]
img: /assets/51_protection/waf.png
summary: "Trong mô hình chia sẻ trách nhiệm, người dùng chịu trách nhiệm bảo mật trong đám mây (security IN the cloud), bao gồm bảo vệ dữ liệu, ứng dụng và cấu hình hạ tầng được triển khai. Hãy cùng tìm hiểu các dịch vụ giúp thực hiện mục tiêu này."
---

Trong [mô hình chia sẻ trách nhiệm](/2025/11/12/aws_infrastructure#shared-responsibility), người dùng chịu trách nhiệm bảo mật trong đám mây (**security IN the cloud**), bao gồm bảo vệ dữ liệu, ứng dụng và cấu hình hạ tầng được triển khai. Hãy cùng tìm hiểu các dịch vụ giúp thực hiện mục tiêu này.


## Trong bài này:

- [1. Macie - Bảo vệ Dữ liệu Nhạy cảm](#macie)
- [2. Web Application Firewall (WAF)](#waf)
- [3. AWS Shield - Chống DDoS](#shield)
    - [3.1. Shield Standard](#shield-standard)
    - [3.2. Shield Advanced](#shield-advanced)
- [4. GuardDuty - Phát hiện Nguy cơ](#guardduty)
- [5. Inspector - Phát hiện Lỗ hổng Bảo mật](#inspector)
- [6. Trusted Advisor - Khuyến nghị](#trusted-advisor)
- [Tài liệu tham khảo](#reference)


<a name = "macie"></a>

## 1. Macie - Bảo vệ Dữ liệu Nhạy cảm

Macie là dịch vụ bảo vệ dữ liệu nhạy cảm (**sensitive data**), sử dụng __Học Máy (*Machine Learning*)__ và __so khớp mẫu (*pattern matching*)__ để xác định và phân loại dữ liệu nhạy cảm lưu trên [S3](/2025/11/30/s3-fundamental), ví dụ như:

- Thông tin cá nhân có thể nhận dạng (*Personal Identifiable Information - PII*): tên, địa chỉ, số điện thoại, số CCCD, hộ chiếu, v.v.
- Dữ liệu tài chính (*Financial Data*): số thẻ tín dụng, số tài khoản ngân hàng, v.v.
- Mật khẩu, API Key, v.v.


Macie tự động quét các object trong tất cả bucket, phát hiện dữ liệu nhạy cảm và cung cấp báo cáo chi tiết (**sensitive data finding**) về loại dữ liệu, vị trí lưu trữ, và mức độ rủi ro. Finding có thể được đẩy sang EventBridge để tự động hóa việc xử lý tiếp theo (như gửi thông báo qua [SNS](/2026/01/22/step-function#sns), xoá dữ liệu, v.v).

Ngoài ra, Macie cũng **giám sát S3 Bucket** để phát hiện các cấu hình không an toàn, như đang ở chế độ public, không được mã hoá, các danh tính lạ có quyền truy cập, v.v. và cung cấp **policy finding** tương ứng.



<a name = "waf"></a>

## 2. Web Application Firewall (WAF) 

WAF là dịch vụ tường lửa, hoạt động ở **tầng ứng dụng** (Layer 7 trong mô hính OSI), nghĩa là nó có thể nhìn thấy nội dung của các yêu cầu HTTP/HTTPS.
WAF giúp bảo vệ ứng dụng khỏi các lỗ hổng bảo mật phổ biến như [SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection), [Cross-site Scripting (XSS)](https://owasp.org/www-community/attacks/xss/), [HTTP Flood](https://www.cloudflare.com/learning/ddos/http-flood-ddos-attack/), và các bot tấn công làm tiêu tốn tài nguyên, gây gián đoạn dịch vụ hoặc có ý đồ xâm nhập dữ liệu.



<p>
<image src="/assets/51_protection/waf.png" alt="WAF" style="max-width:80%;height:auto;display:block;margin:0 auto;"/>
</p>

Các thành phần chính:

- **Web ACL** (*Web Access Control List*): đơn vị làm việc chính của WAF, gắn vào các tài nguyên như [CloudFront Distribution](/2026/02/20/cloudfront), [Application Load Balancer](/2026/02/05/load-balancer#alb), hay [API Gateway](/2026/01/28/api-gateway) để bảo vệ chúng. Với Load Balancer, do WAF hoạt động ở tầng ứng dụng, nên chỉ hỗ trợ Application Load Balancer. Trong Web ACL, ta định nghĩa các **Rule** và **Action** để xác định cách xử lý yêu cầu. 
- **Rule**: tập hợp các tiêu chí để xác định một yêu cầu web có phải là tấn công hay không. Các tiêu chí phổ biến:
    - **Mã độc**: yêu cầu có chứa mã độc hại như SQL Injection, XSS, v.v. không.
    - **Địa chỉ IP**: địa chỉ IP của yêu cầu có nằm trong danh sách đen (*blacklist*) hoặc danh sách trắng (*whitelist*) không.
    - **Vị trí địa lý**: yêu cầu có đến từ các quốc gia bị chặn không.
    - **Độ dài**: kiểm tra độ dài của các thành phần như query string để chặn các yêu cầu dài bất thường.
    - WAF cũng hỗ trợ **kết hợp nhiều tiêu chí** bằng các toán tử logic (**AND, OR, NOT**) trong Rule.
- **Action**: hành động được thực hiện khi một yêu cầu khớp với các tiêu chí của Rule, bao gồm:
    - **Block**: chặn và trả về lỗi 403 (*Forbidden*).
    - **Allow**: cho phép đi qua mà không kiểm tra thêm.
    - **Count**: chỉ **đếm số lần khớp với rule**, **không chặn**, hữu ích để đánh giá hiệu quả của rule trước khi triển khai.
    - **CAPTCHA**: yêu cầu người dùng hoàn thành CAPTCHA để xác nhận họ không phải bot.


<a name = "shield"></a>

## 3. AWS Shield - Chống DDoS

Đây là dịch vụ chống lại **DDoS** (*Distributed Denial of Service*), là các cuộc tấn công nhằm mục đích gây gián đoạn dịch vụ bằng cách gây quá tải tài nguyên. Có 2 cấp độ dịch vụ:


<a name = "shield-standard"></a>

### 3.1. Shield Standard

Shield Standard được tự động **áp dụng cho tất cả tài khoản AWS**, hoàn toàn **miễn phí**, không cần đăng ký. 

Shield Standard hoạt động ở **tầng mạng (Layer 3)** và **tầng truyền tải (Layer 4)** trong mô hình OSI, chống lại các phương thức tấn công DDoS như [SYN Flood](https://www.cloudflare.com/learning/ddos/syn-flood-ddos-attack/), [UDP Flood](https://www.cloudflare.com/learning/ddos/udp-flood-ddos-attack/), [UDP Reflection](https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/udp-reflection-attacks.html), v.v. Thường dùng để bảo vệ các dịch vụ như [CloudFront](/2026/02/20/cloudfront), [Route 53](/2026/02/14/route-53), hay Global Accelerator.


<a name = "shield-advanced"></a>

### 3.2. Shield Advanced

Tốn phí **$3000/tháng** (tại thời điểm viết bài), cung cấp các tính năng nâng cao như:
- Bảo vệ toàn diện hơn, gồm cả **tầng ứng dụng (Layer 7)** và các dịch vụ khác như [ELB](/2026/02/05/load-balancer) (cả ALB, NLB), [EC2](/2025/12/16/ec2-fundamental), v.v.
- Hỗ trợ phản ứng nhanh với các chuyên gia bảo mật của AWS DDoS Response Team (DRT) trong trường hợp bị tấn công quy mô lớn.
- Cung cấp báo cáo chi tiết về các cuộc tấn công, bao gồm loại tấn công, lưu lượng, nguồn gốc, v.v., giúp người dùng hiểu rõ hơn về mối đe dọa và cải thiện chiến lược bảo mật.
- **DDoS Cost Protection**: trong trường hợp bị tấn công DDoS quy mô lớn dẫn đến chi phí tăng đột biến do tài nguyên tự động mở rộng (như [EC2 AutoScaling](/2026/02/02/ec2-autoscaling)), AWS sẽ **hoàn lại phần chi phí này**.


<a name = "guardduty"></a>

## 4. GuardDuty - Phát hiện Nguy cơ

GuardDuty là dịch vụ phát hiện nguy cơ (**threat detection**), sử dụng **Học Máy** và **phân tích hành vi** để liên tục giám sát và phân tích các dữ liệu như:
- [VPC Flow Logs](/2026/02/08/vpc-advanced#vpc-flow-log): theo dõi lưu lượng mạng ra vào VPC, phát hiện các hoạt động bất thường như quét cổng, kết nối đến địa chỉ IP độc hại, v.v.
- [CloudTrail](/2026/05/25/infra-management#cloudtrail) Event: theo dõi xem ai đang gọi API gì, phát hiện các hoạt động đáng ngờ như đăng nhập thất bại nhiều lần, thay đổi cấu hình bảo mật, v.v.
- DNS Log: theo dõi các truy vấn DNS, cảnh báo nếu kết nối đến tên miền độc hại, v.v.
- Các nguồn dữ liệu khác như ổ [EBS](/2025/12/20/ebs), [RDS](/2026/01/08/rds-fundamental), Runtime Monitoring Log của [Lambda](/2026/01/18/lambda-fundamental), [EC2 Instance](/2025/12/16/ec2-fundamental), EKS, v.v.

GuardDuty thường được kết hợp với các dịch vụ khác để tự động phản ứng khi phát hiện nguy cơ. [Finding của GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html) được đưa đến EventBridge như một sự kiện, rồi được sử dụng để gửi thông báo qua [SNS](/2026/01/22/step-function#sns), hay kích hoạt [Lambda Function](/2026/01/18/lambda-fundamental) thực hiện các hành động khắc phục, v.v.



<a name = "inspector"></a>

## 5. Inspector - Phát hiện Lỗ hổng Bảo mật 

GuardDuty giám sát hạ tầng và hành vi để phát hiện nguy cơ, còn Inspector tập trung vào việc quét **lỗ hổng bảo mật** (**_vulnerability_**) trong:
- **Phần mềm** và **hệ điều hành** cài đặt trên EC2 Instance, bao gồm **CVE** (*Common Vulnerabilities and Exposures*), và các thông số **CIS** (*Center for Internet Security*).
- **Container Image** được lưu trữ trong [ECR](/2026/05/12/ecs#ecr) hoặc đang chạy trên EKS.
- **Lambda Function**: quét mã nguồn và các thư viện phụ thuộc.
- **Network Reachability**: xem có lỗ hổng mạng nào dẫn đến việc truy cập trái phép vào tài nguyên không.


Giống như GuardDuty hay Macie, khi phát hiện lỗ hổng, Inspector sẽ tạo ra một [**Finding**](https://docs.aws.amazon.com/inspector/latest/user/findings-types.html) chi tiết về lỗ hổng đó, bao gồm mức độ nghiêm trọng, mô tả, và hướng dẫn khắc phục. Finding này cũng được đưa đến EventBridge để tự động hóa việc xử lý tiếp theo.


<a name = "trusted-advisor"></a>

## 6. Trusted Advisor - Khuyến nghị


Dịch vụ này giúp **tối ưu** hạ tầng bằng cách so sánh cấu hình hiện tại của bạn với các Best Practice được AWS khuyến nghị.
Trusted Advisor **quét toàn bộ tài khoản** và cung cấp các **check** (*kiểm tra*) trong 5 danh mục chính:

- **Chi phí**: phát hiện các tài nguyên đang sử dụng không hiệu quả, có thể loại bỏ để tiết kiệm chi phí.
- **Bảo mật**: kiểm tra các cấu hình bảo mật của các dịch vụ như [IAM](/2025/11/07/iam), [S3](/2025/11/30/s3-fundamental), [Security Group](/2025/11/27/sg-nacl#sg), v.v. để phát hiện các vấn đề tiềm ẩn.
- **Hiệu năng**: với các dịch vụ như [EC2](/2025/12/16/ec2-fundamental), [RDS](/2026/01/08/rds-fundamental), v.v. đưa ra khuyến nghị nếu cần nâng cấp loại Instance để cải thiện hiệu năng.
- **Khả năng chịu lỗi** ([Fault Tolerance](/2025/11/12/aws_infrastructure#ft)): các khuyến nghị để tăng tính khả dụng và khả năng chịu lỗi của hạ tầng.
- **Giới hạn dịch vụ**: cảnh báo nếu đang sử dụng gần đến giới hạn AWS cho phép của một dịch vụ nào đó.







<a name = "reference"></a>

## Tài liệu tham khảo

1. [Giới thiệu về Macie](https://docs.aws.amazon.com/macie/latest/user/what-is-macie.html)
2. [Cách Hoạt động của WAF](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html)
3. [WAF Rule](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rules.html)
4. [Action trong WAF Rule](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-action.html)
5. [Shield Standard](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-standard-summary.html)
6. [Shield Advanced](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-summary.html)
7. [Amazon GuardDuty](https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html)
8. [Amazon Inspector](https://docs.aws.amazon.com/inspector/latest/user/what-is-inspector.html)
9. [Trusted Advisor](https://docs.aws.amazon.com/awssupport/latest/user/get-started-with-aws-trusted-advisor.html)


Trên đây là những dịch vụ chính giúp bảo vệ dữ liệu, ứng dụng và hạ tầng trên AWS. Trong giới hạn các kỳ thi chứng chỉ, nhiêu đây là đủ để trả lời đúng các câu hỏi liên quan. Bạn đọc chỉ cần nhớ được mục đích chính của từng dịch vụ.
