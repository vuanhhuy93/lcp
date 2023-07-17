# 1. Luồng đăng ký tài khoản 

### INPUT
Các client kết nối tới LCP để tạo tài khoản ví cần các tham số sau:
 * client_id
 * hash_request (Sử dụng để kiểm tra lại request có bị thay đổi thông tin không).

 Body phần data để tạo tài khoản, cần các tham số sau:
* owner_id (Dạng chữ) : ID kết nối tài khoản ví với ID của hệ thống đang yêu cầu tạo ví.
* wallet_type : Loại ví cần tạo

## Luồng tạo tài khoản ví:

![alt](/image/Đăng%20ký%20tài%20khoản%20ví.png)

## Diễn giải luồng làm việc với Database

Bước 1: Tìm kiếm thông tin client trong bảng client với câu lệnh:
> select * from client where code = $client_id

Bước 2: Tìm kiếm wallet-type trong bảng wallet-type với câu lệnh
> select * from wallet-type where code = $wallet_type

Bước 3: Từ kết quả tìm kiếm wallet-type ở bước 2, thực hiện truy vấn dữ liệu về wallet_link_type
> select * from wallet_link_type where code = wallet-type.wallet_link_code

Bước 4: Thực hiện insert vào table
> insert into wallet ("owner_id" , "wallet_type") values ($owner_id, $wallet_type);
# 2. Luồng tìm kiếm số dư trong của user

### INPUT 
Các client kết nối tới LCP để tạo tài khoản ví cần các tham số sau:
* client_id
* hash_request (Sử dụng để kiểm tra lại request có bị thay đổi thông tin không).

Body phần data để tạo tài khoản, cần các tham số sau:
* owner_id (Dạng chữ) : ID kết nối tài khoản ví với ID của hệ thống đang yêu cầu tạo ví.
* wallet_id (Id ví) . Trong TH truyền ID ví sẽ ưu tiên tìm kiếm theo ID ví.
* wallet_type : Loại ví cần tìm kiếm (Trong TH client có nhiều loại wallet_type sẽ yêu cầu truyền cụ thể)
* currencies: Lấy về số dư những loại tiền tệ nào.

* 2 tham số: wallet_id và owner_id : phải có 1 trong 2 tham số này

### Luồng thông tin số dư tài khoản

![alt](/image/Tìm%20kiếm%20số%20dư.png)

Diễn giải luồng làm việc với Database

Bước 1: Tìm kiếm thông tin client trong bảng client với câu lệnh:
> select * from client where code = $client_id

Bước 2: Tìm kiếm wallet-type trong bảng wallet-type với câu lệnh
> select * from wallet-type where code = $wallet_type

Bước 3: Từ kết quả tìm kiếm wallet-type ở bước 2, thực hiện truy vấn dữ liệu về wallet_link_type
> select * from wallet_link_type where code = wallet-type.wallet_link_code

Bước 4: Tìm kiếm thông tin số dư
TH1: Tìm kiếm theo wallet_id
> select * from wallet where id = $wallet_id;
>
> select * from wallet_part where wallet_id = wallet.id;

TH2: Tìm kiếm theo owner_id
> select * from wallet where owner_id = $owner_id and wallet_type = $wallet_type;
>
> select * from wallet_part where wallet_id = wallet.id;



# 3. Luồng nạp tiền từ vnpay

Luồng nạp tiền qua vnpay bao gồm 2 bước, các bước được thể hiện như seaquence sau:

![alt](/image/nạp%20tiền%20vnpay%20-%20lcpv3.png)

## Bước 1: Khách hàng yêu cầu nạp tiền qua vnpay

### INPUT: 

Các client kết nối tới LCP để tạo tài khoản ví cần các tham số sau:
* client_id
* hash_request (Sử dụng để kiểm tra lại request có bị thay đổi thông tin không).

Body phần data để tạo tài khoản, cần các tham số sau:
* transaction_id (Dạng chữ) : mã giao dịch của hệ thống ( dùng cho việc đồng bộ hệ id).
* service_code : Mã dịch vụ mà client đã đăng ký với LCP
* amount : Số tiền giao dịch
* wallet_id: id ví cần nạp tiền. (Optional)
* owner_id: Id của đối tượng cần nạp tiền  (Optional)
* wallet_type : Loại ví cần tìm kiếm
* currency_code: code của tiền tệ
(Trong TH client có nhiều loại wallet_type sẽ yêu cầu truyền cụ thể)  (Optional)

Trong TH truyền wallet_id sẽ không yêu cầu truyền owner_id và wallet_type. 
TH không truyền wallet_id  sẽ yêu cầu truyền owner_id và wallet_type.

### Luồng làm việc với database

Bước 1: Tìm kiếm thông tin client trong bảng client với câu lệnh:
> select * from client where code = $client_id

Bước 2: Tìm kiếm wallet-type trong bảng wallet-type với câu lệnh
> select * from wallet-type where code = $wallet_type

Bước 3: Từ kết quả tìm kiếm wallet-type ở bước 2, thực hiện truy vấn dữ liệu về wallet_link_type
> select * from wallet_link_type where code = wallet-type.wallet_link_code

Bước 4: Kiểm tra service code
> select * from client_service where client_code = $client_id and service_code = $service_code;

Bước 5: Tìm kiếm thông tin về wallet
TH1: Tìm kiếm theo wallet_id
> select * from wallet where id = $wallet_id;
>
> select * from wallet_part where wallet_id = wallet.id;

TH2: Tìm kiếm theo owner_id
> select * from wallet where owner_id = $owner_id and wallet_type = $wallet_type;
>
> select * from wallet_part where wallet_id = wallet.id;

Bước 6: Kiểm tra currency
> select * from currency where code = $transaction.currency_code;

Bước 6. Tạo transacion sau khi nhận được thông tin từ vnpay phản hồi
> insert into transaction (client_code ,request_id, external_transaction_id ,service_code, transaction_type, balance, source_wallet, to_wallet , status, currency_code, wallet_part_balance_transfer),  
> values ($client_code ,$transaction_id, $vnpay_transaction_id , $service_code, $client_service.transaction_type, $amount, 
> $client_service.wallet_id, $wallet.id,  1, $currency_code,  {"from_wallet": [ {"part_id" : 1, "amount": -100}], "to_wallet":[{"part_id" : 10, "amount": 100}] })

## Bước 2:  LUỒNG VNPAY CALLBACK

### INPUT
VNPAY callback ngược lại thông báo công tiền thành công
Hiện tại chưa rõ các tham số vnpay trả về nhưng chắc chắn có request_id/transaction_id của giao dịch bên vnpay

### Luồng làm việc với database

Bước 1: Tìm kiếm thông tin giao dịch với transaction_id mà vnpay cung cấp
> select * from transaction where external_transaction_id = $vnpay_transaction_id;

Bước 2: Từ service_code, transaction_type, currency_code tìm kiếm thông tin cấu hình

> select * from client_service where client_code = $transaction.client_code and service_code = $transaction.service_code
> select * from transation_type where code = $transaction.transaction_type
> select * from currency where code = $transaction.currency_code;

Bước 3: Kiểm tra wallet_part

>select wallet_part_to from wallet_part where id = $transaction.wallet_part_balance_transfer.to_wallet[0].part_id;
>select wallet_part_source from wallet_part where id = $transaction.wallet_part_balance_transfer.source_wallet[0].part_id;

Bước 4: Tạo một giao dịch nạp tiền vào ví của vnpay

> insert into transaction (client_code ,request_id, external_transaction_id ,service_code, "NAP TIEN VNPAY", balance, to_wallet, status, currency_code, wallet_part_balance_transfer),  
> values ($client_code ,$transaction_id, $vnpay_transaction_id , $service_code, $client_service.transaction_type, $amount,
> $client_service.wallet_id,  3, $currency_code,  {"from_wallet": [], "to_wallet":[{"part_id" : 10, "amount": 100}] })


Cộng tiên và ví vnpay

> update wallet_part set balance = balance + $transaction.amount where id = $wallet_part.id

Chuyển tiền từ ví vnpay sang ví user

> update wallet_part_source set balance = balance - $transaction.amount where id = $wallet_part.id

> update wallet_part_to set balance = balance + $transaction.amount where id = $wallet_part.id

update lại status của transaction
> update trannsaction set status = 3 where id = $transaction.id