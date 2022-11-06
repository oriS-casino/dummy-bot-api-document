# Dummy bot server

**Contact Me:** https://t.me/DontBeSad_Cuz_MinhFatIsHere

_BaseURL_: http://localhost:9704

Document cho API Dummy Bot Service.

**Các tính năng chính:**
- Chịu tải lớn, nhanh, độ trễ thấp, có thể chạy đồng thời hàng nghìn con bot.
- Chơi giống hệt người, không thể phân biệt được đâu là Bot đâu là người.
- Điều chỉnh độ thông minh một cách linh hoạt, giúp vận hành bot một cách dễ dàng nhất.
- Deploy sử dựng Docker.
- Kích thước nhỏ nhẹ, toàn bộ source chỉ vài MB.


## Contents

- [Deploy](#deploy)
- [Config](#config)
- [Card Value](#card-value)
- [API](#api)
    * [API Ping](#api-ping)
    * [API Select Best Action](#api-select-best-action)
    * [Player](#player)
    * [Action](#action)
    * [Action Type](#action-type)
    * [Status Code](#status-code)

## Deploy

quick deploy
```shell
sh restart.sh
```

using docker
```shell
# build
sudo docker compose build 

# start 
sudo docker compose up -d

# stop 
sudo docker compose stop --timeout=60
```

Log lưu trong thư mục `./logs`

Test ping
```shell
sh ping.sh
```

## Config

Điều chỉnh config trong file **configs/conf.yaml**, **default config** sẽ như sau:
Lưu ý:

- _default_interactions_, _default_min_thinking_time_, _default_max_thinking_time_ không nên sửa, nếu muốn thay đổi thì
  truyền vào khi call [API Select Best Action](#api-select-best-action).
- _log_path_ nếu thay đổi sang đường dẫn khác thì tạo trước thư mục và cấp quyền đọc ghi cho thư mục đó.

```yaml
# mật khẩu
key: abcXYZ123456789
# thời gian nhiều nhất dành cho 1 request, tính bằng mili giây
timeout: 15000
# số lượng tối đa request được đưa vào hàng đợi
max_queue_size: 1024
# số lượng tối đa request được xử lý cùng 1 lúc
max_workers: 32
default_interactions: 1000000
# thời gian tối thiểu suy nghĩ 1 nước
default_min_thinking_time: 500
# thời gian tối đa suy nghĩ 1 nước
default_max_thinking_time: 1000
# đường dẫn tới folder log
log_path: ./logs
# format tên của file log
# mặc định là YYYY-MM-DD
# Ví dụ:
# 2006-01-02 : YYYY-MM-DD
# 02-01-2006 : DD-MM-YYYY
file_log_name_format: 2006-01-02
# tự định nghĩa giá trị mỗi lá bài truyền lên server
rank_rule:
  two: 0
  three: 1
  four: 2
  five: 3
  six: 4
  seven: 5
  eight: 6
  nine: 7
  ten: 8
  jack: 9
  queen: 10
  king: 11
  ace: 12
# tự định nghĩa giá trị của mỗi chất truyền lên server
suit_rule:
  spade: 0
  club: 1
  diamond: 2
  heart: 3
# tự định nghĩa giá trị của mỗi loại action
action_rule:
  # rút 1 lá lên
  chow: 0
  # đánh 1 lá ra
  discard: 1
  # ăn lá dưới bàn
  dump: 2
  # hạ phỏm
  expose: 3
  # gửi bài
  send: 4
```

## Card Value

**Suit**: giá trị tương ứng với chất của lá bài được định nghĩa trong file [configs/conf.yaml](#config). (ví dụ: Bích,
Cơ, Rô, Nhép)

**Rank**: giá trị tương ứng với giá trị của lá bài được định nghĩa trong file [configs/conf.yaml](#config). (ví dụ 9,
10, J, Q, K)

Công thức chuyển đổi từ **card value** sang **suit** và **rank**

```c#
card_value = rank * 4 + suit 

suit = card_value % 4

rank = (card_value - suit) / 4
```

## API

_BaseURL_: http://localhost:9704


### API Ping

```http request
GET /api/v1/ping
Content-Type: application/json
```

_Response_

| Parameter                             | Type     | Description                             |
|:--------------------------------------|:---------|-----------------------------------------|
| `status`                              | `string` |                                         |
| `version`                             | `string` |                                         |
| `number_of_request_are_being_handled` | `int`    | số lượng request đang được xử lý        |
| `number_of_request_in_queue`          | `int`    | số lượng request đang đợi để được xử lý |
| `number_of_threads`                   | `int`    | số luồng hệ thông đang chạy             |
| `cpu_free`                            | `string` | số cpu free (theo %)                    |
| `ram_free`                            | `string` | số ram free (theo %)                    |

### API Select Best Action

```http request
POST /api/v1/best-actions
Content-Type: application/json
Authorization: Bearer API-KEY
```

_Body_

| Parameter           | Type                  | Description                                                                                                                                                                                                                                                                                                |
|:--------------------|:----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `players`           | [`Player[]`](#player) | Danh sách người chơi trên bàn., theo thứ tự của vòng chơi ví dụ trên bàn còn 4 người chơi A, B, C, D, đang lượt chơi của C và tiếp theo sẽ là D, A, B theo thứ tự. thì mảng này sẽ sắp xếp theo thứ tự [A, B, C, D], hoặc [B, C, D, A] hoặc [C, D, A, B], ..etc. miễn sao là theo thứ tự của vòng là được. |
| `current_player_id` | `string`              | Id của người chơi hiện tại                                                                                                                                                                                                                                                                                 |
| `deck`              | `int[]`               | Bộ bài đang úp theo thứ tự từ lá dưới cùng đến là trên cùng (ví dụ [2, 3, 4, 5, 6] thì lá 6 là lá trên cùng, nếu người chơi bốc thì sẽ bốc lá 6)                                                                                                                                                           |
| `pile_head`         | `int`                 | Lá Head của Pile                                                                                                                                                                                                                                                                                           |
| `pile`              | `int[]`               | Các lá trong Pile theo thứ tự lá gần nhất nằm cuối ví dụ đánh thêm 1 lá ra thì thêm lá này vào cuối pile. **Chú ý là sai thứ tự lá là sẽ hỏng hết code**                                                                                                                                                   |
| `interactions`      | `int`                 | (không được đặt nhỏ hơn 10 có thể gây lỗi) Chỉ số điều chỉnh độ thông minh của Bot, nếu unlimit có thể đặt là 1M, chú ý nếu chỉ số này càng thấp thì càng tốn ít tài nguyên của máy chủ, nên với những trường hợp cụ thể không cần bot thông minh thì có thể đặt chỉ số này là 50                          |
| `min_thinking_time` | `int`                 | Thời gian nhỏ nhất suy nghĩ                                                                                                                                                                                                                                                                                |
| `max_thinking_time` | `int`                 | Thời gian lớn nhất suy nghĩ (không nên để quá 2000)                                                                                                                                                                                                                                                        |

_Response_

| Parameter     | Type                  | Description                                                                                                                                                                                              |
|:--------------|:----------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `status_code` | `int`                 | mã lỗi Success = 0, Error > 0                                                                                                                                                                            |
| `status_desc` | `string`              | mô tả lỗi                                                                                                                                                                                                |
| `actions`     | [`Action[]`](#action) | danh sách các hành động cho bot (theo thứ tự lần lượt). Trong trượp hơp biến **is_challenge = true** thì biến này sẽ trả về mảng có 1 phần tử duy nhất, và **actions[0].Type = challenge** hoặc **fold** |

### Player

| Parameter | Type      | Description                                                    |
|:----------|:----------|----------------------------------------------------------------|
| `id`      | `string`  | ID của người chơi, các người chơi không được trùng ID với nhau |
| `cards`   | `int[]`   | các lá bài trên tay người chơi                                 |
| `melds`   | `int[][]` | các bộ phỏm đã hạ dưới bàn của người chơi                      |
| `score`   | `int`     | điểm của người chơi hiện tại                                   |
| `is_bot`  | `bool`    | true nếu người này là bot                                      |

### Action

| Parameter | Type    | Description                                                                            |
|:----------|:--------|----------------------------------------------------------------------------------------|
| `type`    | `int`   | loại hành động. [Xem thêm](#action-type)                                               |
| `card`    | `int`   | lá đơn đính kèm hành động, nếu là đánh ra 1 lá, dump hoặc gửi bài thì sẽ có trường này |
| `meld`    | `int[]` | bộ phỏm đính kèm hành động, nếu là hạ phỏm, dump hoặc gửi bài thì sẽ có trường này     |

### Action Type

Giá trị của action type được định nghĩa trong file configs/conf.yaml

| Action    | Description      |
|:----------|:-----------------|
| `Chow`    | Rút bài          |
| `Dump`    | Ăn lá dưới bàn   |
| `Send`    | Gửi bài vào phỏm |
| `Discard` | Đánh ra 1 lá     |
| `Expose`  | Hạ phỏm          |

### Status Code

| Status Code | Description                             |
|:------------|:----------------------------------------|
| 0           | Request Success                         |
| 1           | Input gửi lên sai                       |
| 2           | Meld bị sai format                      |
| 3           | Số lượng người chơi không hợp lệ        |
| 4           | ID của nhiều người bị trùng nhau        |
| 5           | Không tìm thấy người chơi hiện tại      |
| 6           | Card value không hợp lệ (>= 0 và <= 51) |
| 7           | Người chơi gửi lên là null              |
| 8           | ID của người chơi gửi lên là null       |
| 9           | Thời gian suy nghĩ gửi lên không hợp lệ |
| 10          | Timeout                                 |
| 11          | Lỗi server                              |
