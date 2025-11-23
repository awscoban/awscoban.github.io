---
layout: post
title: "1. Tài khoản AWS"
title2: "Tài khoản AWS"
date: 2025-11-06
permalink: /2025/11/06/aws-account
img: /assets/1_taikhoanaws/overview.png
categories: [Account & Organization]
tags: [Account & Organization]
summary: Một cách ngắn gọn, có thể hiểu tài khoản AWS như một "container" chứa các thực thể được quản lý bởi tài khoản (IAM User, IAM Group, IAM Role)
---

Để làm việc với AWS, tất nhiên bạn cần có một tài khoản. Bài này mình sẽ giới thiệu các khái niệm, thành phần quan trọng của một tài khoản AWS.

## Trong bài này:

- [1. Tổng quan](#tong-quan)
- [2. Root User](#root-user)
- [3. AWS Organizations](#organization)
    - [Tài khoản quản lý (Management Account)](#management-account)
    - [Tài khoản thành viên (Member Account)](#member-account)
    - [Đơn vị tổ chức (Organizational Unit - OU)](#ou)
    - [Service Control Policy - SCP](#scp)
- [Tài liệu tham khảo](#tai-lieu-tham-khao)


<a name="tong-quan"></a>

## 1. Tổng quan

<img src="/assets/1_taikhoanaws/overview.png" alt="AWS Account" style="max-width:50%;height:auto;display:block;margin:0 auto;">

Một cách ngắn gọn, có thể hiểu tài khoản AWS như một "container" chứa các thực thể được tạo bởi tài khoản (IAM User, IAM Group, IAM Role, sẽ được trình bày dưới đây) và tài nguyên AWS được cung cấp cho các thực thể đó. 
Tất cả chi phí phát sinh trong quá trình sử dụng các dịch vụ, bất kể được dùng bởi thực thể nào, sẽ được tính vào chi phí của tài khoản và được khấu trừ theo phương thức thanh toán.
Mỗi tài khoản có định danh duy nhất và được liên kết với thông tin thanh toán, thực thể và các tài nguyên mà AWS cung cấp. 

<a name="root-user"></a>

## 2. Root User
Khi tạo tài khoản AWS, bạn cung cấp email, tên tài khoản, và cách thức thanh toán (thường là thẻ tín dụng/ghi nợ). Root User là tài khoản chính được tạo sau khi đăng ký. 

Root User có tất cả quyền đối với tài khoản, có thể truy cập đầy đủ vào tất cả dịch vụ và tài nguyên, và **không thể bị hạn chế** (có nghĩa một IAM User dù có quyền `AdministratorAccess` cũng không thể xoá quyền của Root User, ngược lại Root User có thể xoá quyền của IAM User admin đó). 
Do đó, thông thường rất ít dùng Root User, chỉ nên sử dụng cho các tác vụ quản trị cấp cao nhất (xem thêm [tại đây](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html#root-user-tasks)), để tránh các vấn đề bảo mật không mong muốn có thể xảy ra.
*Nên dùng Root User tạo một IAM User với quyền `AdministratorAccess`, rồi đăng nhập vào IAM User admin này để quản lý công việc*.

Các danh tính như IAM User, IAM Group, IAM Role sẽ được trình bày kỹ hơn trong bài [IAM](/2025/11/07/iam).
<a name="organization"></a>


## 3. AWS Organizations

Ở trên ta đã tìm hiểu về tài khoản AWS. Một công ty thường sẽ có nhiều tài khoản AWS để quản lý tốt hơn. Ví dụ, có thể nhóm các tài khoản theo chức năng của phòng ban, hoặc theo dự án, hoặc theo môi trường (Development, Staging, Production). Khi đó, sẽ có nhiều vấn đề phát sinh liên quan đến bảo mật, cấp quyền, tối ưu chi phí. 
AWS Organizations sẽ giúp ta giải quyết các vấn đề này.

Organizations bản chất là một tập hợp các tài khoản AWS được quản lý tập trung, có cấu trúc phân tầng hình cây như sau: 

<img src="/assets/1_taikhoanaws/org.png" alt="AWS Organizations" style="max-width:80%;height:auto;display:block;margin:0 auto;">


<a name="management-account"></a>

#### Tài khoản quản lý (Management Account)
Là tài khoản có quyền cao nhất, dùng để quản lý các tài khoản thành viên (Member Account) khác. Mọi chi phí phát sinh từ các tài khoản thành viên sẽ được tổng hợp và thanh toán qua tài khoản quản lý này.


<a name="member-account"></a>

#### Tài khoản thành viên (Member Account):
Có thể được tạo ra trong Organizations hoặc là một tài khoản AWS có sẵn được mời tham gia. Mỗi tài khoản thành viên có thể có các thực thể riêng (IAM User, IAM Group, IAM Role) và tài nguyên AWS riêng, nhưng quyền của chúng trong Organizations sẽ bị giới hạn bởi quyền của tài khoản thành viên (được đặt bởi tài khoản quản lý).

**Tại một thời điểm, một tài khoản AWS chỉ có thể tham gia tốt đa một Organization**. Nếu tài khoản đã tham gia một Organization, cần phải rời khỏi Organization đó trước khi tham gia Organization mới.

<a name="ou"></a>

#### Đơn vị tổ chức (Organizational Unit - OU)
Là các nhóm con trong Organizations, dùng để tổ chức các tài khoản thành viên theo cấu trúc phân tầng. Mỗi OU có thể chứa nhiều tài khoản thành viên và/hoặc các OU con khác. Cao nhất trong phân cấp OU là Root OU, chứa tất cả các tài khoản thành viên và OU con trong tổ chức. 
**Lưu ý, không nên nhầm lẫn Root OU trong Organizations với Root User của tài khoản AWS**. Root OU cũng giống với các OU khác, là một nhóm các tài khoản con (chứa tất cả), không có quyền đặc biệt nào cả. Quyền cao nhất trong Organizations luôn thuộc về tài khoản quản lý.

Việc sử dụng OU giúp quản lý chính sách và quyền truy cập một cách hiệu quả hơn thông qua việc áp dụng các chính sách giới hạn dịch vụ (Service Control Policy - SCP) cho các OU, thay vì lặp lại nhiều lần cho từng tài khoản thành viên.

<a name="scp"></a>

#### SCP (Service Control Policy) 

Là các chính sách được áp dụng ở cấp độ Organizations, dùng để **giới hạn** quyền mà tài khoản thành viên có thể có. Lưu ý, SCP không cấp quyền. Ví dụ, nếu một SCP được áp dụng cho một OU ngăn chặn việc sử dụng S3, thì tất cả các tài khoản thành viên trong OU đó sẽ không thể sử dụng S3, bất kể quyền IAM User hay IAM Role của họ có cho phép sử dụng S3 hay không.

- [Phía trên](#root-user) mình đã đề cập Root User có tất cả quyền đối với một tài khoản AWS, không thể bị hạn chế. Vậy có điều gì mâu thuẫn ở đây không? Câu trả lời là không. Root User không bị hạn chế **trong phạm vi tài khoản AWS đó**, nhưng **những gì tài khoản này được phép làm** khi tham gia Organizations sẽ bị giới hạn trong SCP! (trừ khi đây là tài khoản quản lý).

- Như đã đề cập, việc sử dụng OU giúp quản lý chính sách và quyền truy cập một cách hiệu quả hơn thông qua SCP. SCP được áp dụng cho OU sẽ tự động áp dụng cho tất cả các tài khoản thành viên bên trong OU đó và các OU con (nếu có). Cấu trúc phân tầng trong Organizations sẽ giúp giới hạn dần quyền truy cập khi đi sâu vào các OU con, do các OU con sẽ kế thừa tất cả SCP từ các OU phía trên. Ví dụ trong hình trên, `OU 1.1` sẽ kế thừa SCP từ `Root OU` và `OU 1`, ngoài SCP riêng của nó.

- Tài khoản quản lý có thể nằm trong bất kỳ OU nào, không nhất thiết phải là Root OU (hoặc đứng riêng), do OU về bản chất chỉ là một nhóm các tài khoản chia sẻ chung SCP. **Tuy nhiên, dù nằm ở OU nào, SCP của OU đó sẽ không ảnh hưởng đến quyền của tài khoản quản lý**.

Về mặt kỹ thuật, SCP có định dạng JSON giống các IAM policy, sẽ được trình bày kỹ hơn trong [bài về IAM](/2025/11/07/iam).


Tóm lại, với Organizations, ta có thể:
- Tạo, quản lý và thanh toán tập trung nhiều tài khoản AWS.
- Áp dụng SCP để quản lý quyền của các tài khoản trong tổ chức.







<a name="reference"></a>

### Tài liệu tham khảo

1. [Các tác vụ dành cho Root User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html#root-user-tasks)
2. [Nguyên tắc thiết kế đa tài khoản](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/design-principles-for-your-multi-account-strategy.html)
3. [Service Control Policy](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)


Ở bài này mình đã đề cập đến các thực thể trong tài khoản AWS như IAM User, IAM Group, IAM Role. Trong bài tiếp theo, ta sẽ đi tìm hiểu sâu hơn về IAM.
