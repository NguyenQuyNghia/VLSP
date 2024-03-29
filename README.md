# VLSP
## Giới thiệu bài toán
Bài toán Comparative Opinion Mining from Vietnamese Product Reviews đặt ra yêu cầu cần trích xuất đánh giá của khách hàng và xác định ý kiến so sánh của người đánh giá.

Với bài toán như vậy, các bước thực hiện là: 
  - Đầu tiên cần phân loại được câu đánh giá đó là câu so sánh hay không so sánh. 
  - Với những câu so sánh, cần trích xuất được các đối tượng so sánh, khía cạnh so sánh, từ hoặc cụm từ so sánh, từ đó phân loại câu so sánh đó (so sánh hơn, so sánh bằng,...).

Mô hình được xây dựng theo kiến trúc sau
![Screenshot 2024-01-11 004757](https://github.com/NguyenQuyNghia/VLSP/assets/100202140/ccae6f78-6e2c-4126-9239-1d86d7f26856)

## Stage 1
### Bước 1
Trong bước này mô hình sẽ sử dụng một mô hình được tiền huấn luyện trên dữ liệu tiếng Việt là PhoBERT để qua đó phân loại được các câu so sánh hoặc không so sánh. Trong mô hình nguyên bản của tác giả do huấn luyện trên dữ liệu tiếng Anh nên sẽ sử dụng mô hình tiền huấn luyện trên bộ dữ liệu tiếng Anh nhưng với dữ liệu là tiếng Việt, BERT không thể hiểu được ngữ nghĩa của tiếng Việt nên mô hình phải thay thế BERT bởi PhoBERT.

PhoBERT là một mô hình xử lý ngôn ngữ tự nhiên dựa trên BERT được huấn luyện trên dữ liệu tiếng Việt phần nào hiểu được dữ liệu tiếng Việt.Ở đây để tokenize mô hình cũng sử dụng tokenizer được cung cấp bởi PhoBERT.

Với một câu đầu vào sẽ được tokenize bởi PhoBERT: 

```math
X = [x_1, x_2, \ldots, x_n]
```

Chúng ta sẽ thực hiện việc chèn hai token đặc biệt là `<s>` và `</s>` trong quá trình xử lý dữ liệu.. Token `<s>` được chèn vào đầu câu làm nhiệm vụ phân loại câu so sánh hay không so sánh, token `</s>` được chèn vào cuối câu nhằm nhiệm vụ thông báo một câu đã kết thúc hay chưa.

Đầu ra cuối cùng sau đi qua PhoBERT sẽ được biểu diễn như:

```math
h = [ h_{\text{<} s \text{>}} , h_1 , h2 , \ldots , h_n , h_{\text{<} /s \text{>}} ]
```

Với mỗi biểu diễn đầu ra như trên ta sẽ truyền biểu diễn của token làm nhiệm vụ phân loại hay h`<s>` vào một tầng softmax để xác định câu X có phải câu so sánh không. Đầu ra của lớp softmax này sẽ là:

```math
y_c = \text{softmax}(W_c \cdot h_{\text{<} s \text{>}} + b_c)
```
  
Với W<sub>c</sub> và b<sub>c</sub> lần lượt là ma trận trọng số và bias, y<sub>c</sub> sẽ xác định câu X là so sánh hay không tương ứng với câu so sánh là nhãn {1} không so sánh là nhãn {0}

### Bước 2
Để trích xuất các thành phần so sánh trong câu mô hình sử dụng 4 mô đun CRF riêng biệt tương ứng với việc trích xuất ra 4 tập.

Mô hình sử dụng 4 mô đun CRF (Conditional Random Fields) riêng biệt để trích xuất các thành phần tránh trường hợp các thành phần bị trùng nhau. 

Ví dụ như trong câu sau: 

>"Mặc dù cả hai thiết bị đều sở hữu viên pin có dung lượng tương đương, nhưng tuổi thọ pin có chênh lệch khá lớn".

Ở ví dụ này sub và obj đều được xác định là "cả hai" khi đó nếu sử dụng một mô đun CRF thì có thể sẽ không trích xuất được sub hoặc obj.

Từ đầu ra của PhoBERT ta sẽ truyền biểu diễn từ vào 4 mô đun CRF riêng biệt để trích xuất ra thành 4 phần so sánh. CRF sẽ gán nhãn các từ dựa trên 5 tag Begin-Middle-End-Single-Outside (BMESO). Để đánh dấu từ nào sẽ là từ bắt đầu của thành phần, từ nào sẽ là từ ở giữa hay kết thúc, còn từ nào không nằm trong thành phần so sánh đang được trích xuất hoặc các từ là thành phần đang được trích xuất mà đứng đơn lẻ. 
Hàm mất mát cho bước thứ nhất được tính toán như sau:

```math
L = \lambda_c \cdot L_{\text{csi}} + \lambda_e \cdot \sum L_{\text{cee}}
```

Với λ<sub>c</sub> , λ<sub>e</sub> là các trọng số và L<sub>csi</sub> là hàm mất mát cho bước phân loại câu so sánh. L<sub>cee</sub> là hàm mất mát cho từng thành phần trích xuất.

