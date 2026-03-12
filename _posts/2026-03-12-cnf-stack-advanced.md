---
layout: post
title: "38. Các Cấu hình Stack Nâng cao"
title2: "Các Cấu hình Stack Nâng cao"
date: 2026-03-12
permalink: /2026/03/12/cnf-stack-advanced
categories: [IaC, CloudFormation, Infrastructure]
tags: [IaC, CloudFormation, Infrastructure]
img: /assets/38_cfn_stack_adv/cfn-nested-stack.png
summary: "Bài này trình bày một số cấu hình nâng cao của Stack, gồm Nested Stack, Cross-Stack Reference để tái sử dụng tài nguyên, Change Set để xem trước thay đổi khi cập nhật Stack, và StackSet để quản lý Stack trên nhiều tài khoản và Region."
---

Bài này trình bày một số cấu hình nâng cao của Stack, gồm Nested Stack, Cross-Stack Reference để tái sử dụng tài nguyên, Change Set để xem trước thay đổi khi cập nhật Stack, và StackSet để quản lý Stack trên nhiều tài khoản và Region.


## Trong bài này:

- [1. Nested Stack](#nested-stack)
- [2. Cross-Stack Reference](#cross-stack-reference)
- [3. Xem trước Thay đổi với Change Set](#change-set)
- [4. Triển khai Stack Đa Tài khoản, Đa Vùng với StackSet](#stackset)
- [Tài liệu tham khảo](#reference)






<a name = "nested-stack"></a>

## 1. Nested Stack

Trong ngành công nghệ, "nested" mang nghĩa "lồng nhau", tức một thứ nằm bên trong thứ khác, như *nested function*, *nested loop*, v.v. Tương tự, trong CloudFormation, Nested Stack là kỹ thuật chia một Stack lớn thành nhiều Stack nhỏ, bằng cách thêm *Resource* là Stack.

<p>
<image src="/assets/38_cfn_stack_adv/cfn-nested-stack.png" alt="cfn-nested-stack" style="max-width:40%;height:auto;display:block;margin:0 auto;"/>
</p>


**Tại sao phải chia nhỏ Stack?**

Khi quy mô hạ tầng tăng lên, việc lặp lại các cấu hình tài nguyên giống nhau gây dư thừa và khó quản lý. Thay vào đó, nên chia nhỏ thành các Stack nhỏ, mỗi Stack quản lý một phần chức năng riêng biệt, rồi kết hợp chúng lại bằng Nested Stack. Cách này giúp tăng khả năng **tái sử dụng**, giảm lỗi, và dễ bảo trì hơn. Giống như việc chia một chương trình lớn thành nhiều hàm nhỏ, rồi gọi các hàm này khi cần, tránh viết lại cùng một đoạn mã nhiều lần.

Hơn nữa, CloudFormation có giới hạn số lượng Resource trong một Template (tại thời điểm viết bài là 500), nên nếu hạ tầng lớn, việc sử dụng Nested Stack là bắt buộc.

Ví dụ, xét một ứng dụng web đơn giản gồm một VPC, và một EC2 Instance. Thay vì tạo tất cả tài nguyên trong một Stack, có thể chia thành 2 Stack con: 

- Một Stack tạo VPC, Subnet, [Security Group](/2025/11/27/sg-nacl#sg), v.v. Gọi là `NetworkStack`. Đầu vào của Stack này là [CIDR](/2025/11/13/vpc#cidr) cho VPC và Subnet, đầu ra là VPC ID và Subnet ID được tạo, **để sử dụng trong Stack khác**:

```yaml
Parameters: # Đầu vào của Stack 
  VpcCidr:
    Type: String
  SubnetCidr:
    Type: String

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
  SubnetA:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC 
      CidrBlock: !Ref SubnetCidr
    DependsOn: VPC 

Outputs: # xuất ra thông tin cần thiết để Stack khác sử dụng
  VpcId:
    Value: !Ref VPC
  SubnetId:
    Value: !Ref SubnetA
```

- Một Stack tạo EC2, gọi là `AppStack`:

```yaml
Parameters: # Đầu vào của Stack 
  VpcId:
    Type: String
  SubnetId:
    Type: String

Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0abcdef1234567890
      InstanceType: t2.micro
      SubnetId: !Ref SubnetId

Outputs:
  InstanceId:
    Value: !Ref WebServer
```

Stack chính của ứng dụng sẽ được tạo từ hai Stack con này như sau:


```yaml
Parameters: # Đầu vào của Stack chính, để truyền vào Stack con
    VpcCidr:
        Type: String
    SubnetCidr:
        Type: String

Resources:
  NetworkStack: # Stack con tạo VPC
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/vpc-template.yaml # Template của stack con tạo VPC
      Parameters:
        VpcCidr: !Ref VpcCidr
        SubnetCidr: !Ref SubnetCidr

  AppStack: # Stack con tạo EC2
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack # Đảm bảo VPC có trước
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-bucket/app-template.yaml # Template của stack con tạo EC2
      Parameters:
        # Lấy thông tin từ NetworkStack (chứa VPC và Subnet ID) để truyền vào Stack này
        VpcId: !GetAtt NetworkStack.Outputs.VpcId 
        SubnetId: !GetAtt NetworkStack.Outputs.SubnetId
```

Template cũng khá dễ hiểu. Tương tự như việc gọi hàm trong lập trình, Stack chính gọi hai Stack con, truyền đầu vào cần thiết. Stack con `NetworkStack` sẽ tạo VPC và Subnet, rồi xuất ra VPC ID và Subnet ID tương ứng, truyền cho Stack con `AppStack` để tạo EC2 Instance.


**Khi nào nên dùng Nested Stack?**
Dùng khi các tài nguyên là **cùng vòng đời**, như trong cùng một ứng dụng. Vì khi xoá Stack chính, tất cả Stack con cũng bị xoá theo.




<a name = "cross-stack-reference"></a>

## 2. Cross-Stack Reference

Đây cũng là một kỹ thuật tái sử dụng tài nguyên. Cross-Stack Reference cho phép một Stack xuất ra (*export*) một Output (gọi là Stack nguồn), và các Stack khác có thể nhập (*import*) giá trị đó để sử dụng (gọi là Stack đích). Các Stack này hoàn toàn độc lập, không có liên hệ chặt chẽ như Nested Stack, chỉ chia sẻ thông tin với nhau thông qua Output.

Trở lại ví dụ trên, có thể sử dụng Cross-Stack Reference thay vì Nested Stack như sau:

- Trong `NetworkStack`, sau khi tạo VPC và Subnet, xuất ra Subnet ID:

```yaml
# NetworkStack.yaml
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

  SubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24

Outputs:
  ExportedVpcId:
    Value: !Ref VPC

  ExportedSubnetId:
    Value: !Ref SubnetA
    Export: # Export để Stack khác có thể sử dụng
      Name: awscoban-SubnetA-ID # Tên định danh duy nhất theo Region và tài khoản
```


- Trong `AppStack`, nhập giá trị Subnet ID đã export từ `NetworkStack` để tạo EC2 Instance:

```yaml
# AppStack.yaml
Resources:
  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: ami-0abcdef1234567890
      SubnetId: !ImportValue awscoban-SubnetA-ID # Import Subnet ID được export từ NetworkStack
```

Các vấn đề cần lưu ý:

- Tên trong trường `Export` phải là **duy nhất** trong Region và tài khoản AWS.
- Stack nguồn phải được tạo trước Stack đích. Dễ hiểu, vì không thể nhập vào tài nguyên chưa tồn tại.
- Không thể xoá Stack nguồn nếu vẫn còn Stack đích đang sử dụng giá trị đã export. Cần xoá tất cả Stack đích trước.



**Khi nào nên dùng Cross-Stack Reference?**
- Khi hạ tầng **dùng chung** cho nhiều bộ phận. Ví dụ: nhiều ứng dụng chạy trên cùng VPC.
- Khi muốn **phân quyền** quản lý tài nguyên: team mạng quản lý Stack tạo VPC, team ứng dụng quản lý Stack tạo EC2, v.v., chỉ cần chia sẻ các thông tin cần thiết.
- Khi các Stack có **vòng đời khác nhau**. Ví dụ, Stack VPC thường tồn tại lâu dài, trong khi Stack EC2 có thể xoá và tạo lại nhiều lần.






<a name = "change-set"></a>

## 3. Xem trước Thay đổi với Change Set

Khi dùng CloudFormation, việc cập nhật một Stack đang chạy, đặc biệt trong môi trường Production, cần thực hiện cẩn thận để tránh những lỗi không mong muốn. Change Set là công cụ giúp **xem trước** những thay đổi khi cập nhật Stack, giúp giảm rủi ro cho hệ thống.

<p>
<image src="/assets/38_cfn_stack_adv/cfn-change-set.png" alt="cfn-change-set" style="max-width:70%;height:auto;display:block;margin:0 auto;"/>
</p>


Giả sử ta có một Stack đang chạy, và muốn cập nhật nó bằng một Template mới. Quy trình cập nhật với Change Set như sau:

- Tạo một Change Set trong Stack, tải Template mới lên giao diện CloudFormation, hoặc sử dụng lệnh CLI sau:

```bash
aws cloudformation create-change-set \
    --stack-name awscoban-stack \
    --template-body file://updated-template.yaml \
    --change-set-name OurChangeSet
```

Trong đó, `updated-template.yaml` là Template mới muốn cập nhật. Có thể tạo nhiều Change Set khác nhau để so sánh các phương án cập nhật.

- Xem chi tiết các thay đổi trong Change Set, bao gồm loại thay đổi (Add, Modify, Remove) và tài nguyên bị ảnh hưởng. Có thể xem trên giao diện hoặc dùng lệnh CLI:

```bash
aws cloudformation describe-change-set \
    --change-set-name OurChangeSet \
    --stack-name awscoban-stack
``` 

Đầu ra sẽ liệt kê các thay đổi dưới dạng JSON. Còn nếu xem Change Set trên giao diện, các thay đổi sẽ được trình bày trực quan hơn như sau:

| Thành phần | Ý nghĩa |
|:-----|:------|
| **Action** | Loại thay đổi: Add, Modify, Remove. |
| **Logical ID** | Tên tài nguyên bị tác động. |
| **Replacement** | **True**, **False**, hoặc **Conditional**. Nếu là **True**, tài nguyên cũ sẽ bị xóa và tạo mới, có nguy cơ mất dữ liệu. |
| **Scope** | Các thuộc tính cụ thể bị thay đổi (Properties, Tags, Metadata). |


<p></p>


- Nếu hài lòng với các thay đổi, có thể thực thi Change Set để cập nhật Stack:

```bash
aws cloudformation execute-change-set \
    --change-set-name OurChangeSet \
    --stack-name awscoban-stack
```


Change Set giống như lệnh `git diff` trong lập trình, giúp xem trước những thay đổi đang có trước khi commit. Có thể nói gần như bắt buộc phải sử dụng Change Set khi cập nhật Stack trong môi trường Production.



<a name = "stackset"></a>

## 4. Triển khai Stack Đa Tài khoản, Đa Vùng với StackSet

CloudFormation StackSet là công cụ giúp triển khai và quản lý Stack trên nhiều tài khoản AWS và nhiều Region cùng lúc. thay vì phải làm thủ công trên từng tài khoản và Region. 

StackSet sẽ hữu ích khi cần:

- Thiết lập hạ tầng dùng chung cho nhiều tài khoản, như [VPC](/2025/11/13/vpc), [IAM Role](/2025/11/07/iam#iam-role), CloudTrail, AWS Config, v.v.
<!-- TODO: include link to CloudTrail, AWS Config-->
- Đảm bảo tính nhất quán của hạ tầng trên nhiều tài khoản và Region, như khi triển khai ứng dụng với nhiều bản sao ở các Region khác nhau.




### 4.1. Các Khái niệm Chính

- **Tài khoản Admin**: là tài khoản AWS dùng để quản lý StackSet, tạo StackSet, và triển khai Stack Instance. Thường là [Management Account](/2025/11/06/aws-account#management-account) trong [AWS Organizations](/2025/11/06/aws-account#organization).
- **Tài khoản Đích** (*Target Account*): nơi các Stack trong StackSet được triển khai. 
- **Stack**: là khái niệm Stack ta vẫn hay sử dụng trong loạt bài này. Mỗi Stack được triển khai trong **một tài khoản đích** và **một Region**
- **Stack Instance**: là một "con trỏ" đến một Stack.
- **StackSet**: là tập hợp các Stack Instance, được đặt tại tài khoản Admin. Bạn đọc có thể hiểu StackSet theo nghĩa tường minh là *a set of stack*. Chứa Template mẫu và các tham số cần thiết.


### 4.2. Cách Sử dụng

Để triển khai StackSet, cần thực hiện các bước sau:

- Tạo StackSet trong tài khoản Admin, cung cấp Template và các tham số cần thiết. Ví dụ:

```sh
aws cloudformation create-stack-set \
  --stack-set-name awscoban-stackset \
  --template-url https://s3.us-east-1.amazonaws.com/awscoban-cfn-sample1123/stackset.template \
  --parameters ParameterKey=KeyPairName,ParameterValue=TestKey
```

- Tạo Stack Instance để triển khai Stack đến các tài khoản đích và Region mong muốn. Ví dụ, dưới đây ta tạo Stack ở hai tài khoản đích `account_ID_1` và `account_ID_2`, và ở hai Region `us-east-1` và `ap-southeast-1`, dùng chung Template trong StackSet.

```sh
aws cloudformation create-stack-instances \
  --stack-set-name awscoban-stackset \
  --accounts account_ID_1 account_ID_2 \
  --regions us-east-1 ap-southeast-1 \
  --operation-preferences MaxConcurrentCount=1,FailureToleranceCount=0

```

Hai tham số `MaxConcurrentCount` và `FailureToleranceCount` sẽ được trình bày ngay sau đây.

### 4.3. Các Tính năng Hữu ích

- **Drift Detection**: kiểm tra xem Template trong tài khoản đích có bị sửa khác đi so với Template ban đầu hay không.
- **Maximum concurrent**: giới hạn số lượng tài khoản được triển khai cùng lúc, tránh quá tải và an toàn hơn. 
- **Failure tolerance**: ngưỡng thất bại cho phép. Ví dụ, khi đặt ngưỡng này là 2, nếu có hơn 2 tài khoản bị lỗi, StackSets sẽ tự động dừng lại.
- Khi xoá StackSet, có thể chọn xoá Stack trong tài khoản đích, hoặc giữ lại để tránh mất dữ liệu.





<a name = "reference"></a>

## Tài liệu tham khảo

1. [Chia nhỏ Template dùng Nested Stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-nested-stacks.html)
2. [Giới hạn tài nguyên trong CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html)
3. [Cross-Stack Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html)
4. [Dùng Change Set để Cập nhật Stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html)
5. [Describe Change Set - AWS CLI](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/describe-change-set.html)
6. [Quản lý Stack ở Tài khoản và Region khác với StackSet](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html)

Qua loạt 3 bài vừa rồi, mình đã trình bày những điều cần nắm vững về CloudFormation, từ cơ bản đến nâng cao. Tất nhiên không thể đề cập toàn bộ, nhưng đây là những kiến thức hay gặp trong các kỳ thi chứng chỉ AWS. Tiếp theo, hãy quay lại chủ đề cơ sở dữ liệu, với **DynamoDB**, NoSQL Database trên AWS.