# 22110357, Dinh Tan Dung
# Lab #1: Buffer Overflow
# Task 1: Stack smashing by mermory overwritten
## 1.3. bof3.c
*In this program, the variable 'buf' was declared with the size of 128 byte:*

    char buf[128];
   
*But the function 'fgets' was used with the availability to read up to 133 bytes from the standard input(stdin):*

     fgets(buf,133,stdin);

=> This can lead to buffer overflow, because the input data from the function exceeds the size of 'buf' and overwrites adjacent mermory regions, including the pointer of the function 'func()'. 

*Step 1: Compile the file with option to turn off the stack protector:*

![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/683f84c5-ee21-4bf0-a97e-8400d6ca67bd)

![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/1155b112-51ef-40fe-a1e6-e279989fe1d9)

*Điều này cho thấy bộ đệm có kích thước 0x84 byte (132 byte).*

![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/d865963e-cbd3-4152-85ff-174ecb16b78a)

*Dịa chỉ của hàm shell là 0x0804845b. Bây giờ, chúng ta sẽ tiếp tục với việc tạo payload và khai thác chương trình.*

*Bạn có thể sử dụng Python để tạo payload như sau*

python -c 'print("A" * 132 + "\x5b\x84\x04\x08")' > payload

![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/daa0c43a-8c1c-4284-b2f2-b0a01e9c66e0)

![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/eab89de7-13ce-4d34-91b3-e3f54fe3b317)

*Mặc dù chương trình khai báo bộ đệm có kích thước 132 byte, thực tế EIP bị ghi đè sau 128 byte. Trong phần này, tôi sẽ giải thích lý do cho sự khác biệt này và cách tôi khai thác lỗ hổng.*

2. Phân tích Chương trình
   
Chương trình bof3.out có đoạn mã sau:

    void main()
    {
        int var;
        void (*func)()=sup;
        char buf[128];
        fgets(buf, 133, stdin);
        func();
    }

Bộ đệm buf có kích thước 128 byte.

fgets đọc vào bộ đệm buf với kích thước 133 byte từ stdin.

3. Phân tích Bộ nhớ và Stack

Mặc dù buf có kích thước 128 byte, nhưng khoảng cách từ buf đến EIP là 128 byte, không phải 132 byte như mong đợi. Điều này có thể do các yếu tố sau:

Padding (Điền thêm):
Compiler có thể thêm padding vào các biến cục bộ để căn chỉnh bộ nhớ, nhưng trong trường hợp này, padding không làm tăng khoảng cách từ buf đến EIP.

Bố trí bộ nhớ:
Có thể các biến cục bộ khác hoặc cấu trúc hàm ảnh hưởng đến cách bố trí bộ nhớ.

Compiler Optimizations:
Các tối ưu hóa của compiler có thể thay đổi bố trí của các biến cục bộ, làm cho khoảng cách từ đầu bộ đệm đến EIP không giống như mong đợi.

## Quy trình Khai thác

1. Tạo chuỗi mẫu:

Tôi đã sử dụng chuỗi mẫu để xác định chính xác vị trí của EIP:

    python -c 'print("A" * n + "B" * 4)' > payload
    gdb ./bof3.out
    (gdb) run < payload
    
![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/3ce41eae-71d3-43cb-88cf-79602edea21b)

*Với số n dao động xung quanh giá trị 132 bytes dự tính ban đầu, mỗi lần tăng lên hoặc giảm đi 4 bytes để căn chỉnh sao cho EIP sẽ bị dãy ("B" * 4) ghi đè lên, lúc đó ta sẽ biết được kích thước thật đại diện cho khoảng cách từ 'bù' tới EIP. Bằng
phương pháp sai và thử lại, tôi xác định được size đó là 128 bytes.*

    seed@a15995f85a75:~/seclabs$ python -c 'print("A" * 128 + "\x5b\x84\x04\x08")' > payload
    seed@a15995f85a75:~/seclabs$ xxd payload

### Giải thích cách tạo payload và cách nó ghi đè lên vùng nhớ

#### 1. Tạo Payload

Lệnh sau đây được sử dụng để tạo payload:
```sh
python -c 'print("A" * 128 + "\x5b\x84\x04\x08")' > payload
```

**Giải thích:**

- `python -c`: Chạy mã Python từ dòng lệnh.
- `'print("A" * 128 + "\x5b\x84\x04\x08")'`: Đây là đoạn mã Python tạo ra chuỗi payload.
  - `"A" * 128`: Tạo ra chuỗi gồm 128 ký tự 'A'. Đây là phần dùng để điền đầy bộ đệm (buffer).
  - `"\x5b\x84\x04\x08"`: Đây là địa chỉ của hàm `shell` (`0x0804845b`) được viết ngược lại theo định dạng little-endian. Trong định dạng little-endian, các byte của địa chỉ được sắp xếp từ byte ít quan trọng nhất đến byte quan trọng nhất.
- `> payload`: Chuyển hướng đầu ra của lệnh Python vào file `payload`.

#### 2. Cách Payload ghi đè lên vùng nhớ

1. **Bố trí bộ nhớ trong chương trình:**

   Khi chương trình chạy, stack sẽ chứa các biến cục bộ và địa chỉ trả về của hàm. Với chương trình `bof3.out`, bộ nhớ sẽ được bố trí như sau:
   - Bộ đệm (buffer) `buf` có kích thước 128 byte.
   - Con trỏ hàm `func` nằm ngay sau bộ đệm `buf`.
   - Địa chỉ trả về của hàm (EIP) nằm ngay sau con trỏ hàm `func`.

   Cấu trúc stack:
   ```
   |------------------|
   | Buffer (128 byte)|
   |------------------|
   |   Func pointer   |
   |------------------|
   |  Return address  |
   |------------------|
   ```

2. **Ghi đè lên EIP:**

   Khi `fgets` đọc vào bộ đệm `buf`, nếu dữ liệu đầu vào vượt quá 128 byte, nó sẽ ghi đè lên các vùng nhớ tiếp theo trong stack. Bằng cách đưa vào chuỗi có 128 ký tự 'A' và 4 byte địa chỉ hàm `shell`, payload sẽ ghi đè lên địa chỉ trả về của hàm.

   - Chuỗi `"A" * 128` điền đầy bộ đệm `buf`.
   - Địa chỉ `"\x5b\x84\x04\x08"` sẽ ghi đè lên địa chỉ trả về (EIP).

3. **Kết quả của ghi đè:**

   - Khi chương trình kết thúc hàm `main` và cố gắng quay lại địa chỉ được lưu trong EIP, nó sẽ nhảy đến địa chỉ `0x0804845b` thay vì địa chỉ ban đầu.
   - Địa chỉ `0x0804845b` là địa chỉ của hàm `shell`, do đó hàm `shell` sẽ được thực thi.

   Khi payload được gửi vào chương trình, EIP sẽ bị ghi đè bởi địa chỉ hàm `shell`:
   ```
   EIP: 0x0804845b
   ```

#### 3. Thực hiện Payload

Để kiểm tra payload, bạn có thể sử dụng GDB để chạy chương trình với payload và kiểm tra xem EIP có bị ghi đè đúng cách không:
```sh
gdb ./bof3.out
(gdb) run < payload
```

### Kết luận

Trong bài tập này, việc tạo payload bao gồm:
- Tạo chuỗi dài 128 ký tự 'A' để điền đầy bộ đệm.
- Thêm địa chỉ hàm `shell` ở cuối chuỗi để ghi đè lên địa chỉ trả về.
- Payload này được sử dụng để lợi dụng lỗ hổng buffer overflow, ghi đè lên EIP và chuyển hướng luồng điều khiển của chương trình đến hàm `shell`.

Bằng cách sử dụng các bước này, bạn có thể khai thác thành công lỗ hổng buffer overflow và kiểm soát luồng điều khiển của chương trình.

![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/907f4a87-3f03-4ece-af56-a12a29aec220)

*Xác nhận Payload được tạo đúng cách.*

![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/5fb9e3eb-ebf9-4fcb-8e4d-9a7cdfcd28c0)

Dưới đây là các kinh nghiệm và kiến thức rút ra từ bài tập này mà bạn có thể trình bày trong báo cáo của mình:

### Kinh nghiệm và Kiến thức rút ra từ bài tập

---

**Kinh nghiệm và Kiến thức từ Bài tập Khai thác Lỗ hổng Buffer Overflow trong `bof3.out`**

#### 1. Kiến thức về Buffer Overflow

- **Buffer Overflow là gì:**
  - Buffer overflow xảy ra khi dữ liệu vượt quá giới hạn bộ đệm được cấp phát, dẫn đến ghi đè lên các vùng nhớ khác, bao gồm cả địa chỉ trả về của hàm.

- **Tầm quan trọng của việc kiểm tra kích thước bộ đệm:**
  - Việc không kiểm tra kỹ lưỡng kích thước bộ đệm và dữ liệu đầu vào có thể dẫn đến lỗ hổng bảo mật nghiêm trọng.

#### 2. Kỹ năng Phân tích và Gỡ lỗi

- **Sử dụng GDB để phân tích chương trình:**
  - Sử dụng GDB để đặt breakpoint, kiểm tra các thanh ghi (registers), và phân tích stack giúp xác định chính xác vị trí bị ghi đè trong bộ nhớ.

- **Sử dụng các công cụ hỗ trợ phân tích:**
  - Sử dụng các công cụ như `xxd` để kiểm tra nội dung của payload và xác định đúng định dạng little-endian.

- **Kiểm tra và điều chỉnh payload:**
  - Tạo và điều chỉnh payload dựa trên kết quả phân tích từ GDB để khai thác lỗ hổng một cách hiệu quả.

#### 3. Hiểu về Cấu trúc Bộ nhớ và Stack

- **Cách bộ nhớ được bố trí trong stack:**
  - Hiểu rằng bộ nhớ trong stack được bố trí theo cách mà các biến cục bộ và các giá trị như EIP được xếp chồng lên nhau. Khoảng cách từ đầu bộ đệm đến EIP có thể bị ảnh hưởng bởi padding và các biến khác.

- **Sự khác biệt giữa kích thước bộ đệm và khoảng cách đến EIP:**
  - Nhận thức rằng mặc dù kích thước bộ đệm được khai báo là 132 byte, khoảng cách thực tế từ đầu bộ đệm đến EIP là 128 byte do cách bố trí bộ nhớ của compiler.

#### 4. Quy trình Khai thác Buffer Overflow

- **Xác định vị trí của EIP:**
  - Sử dụng chuỗi mẫu để xác định chính xác vị trí của EIP, từ đó điều chỉnh payload để ghi đè lên EIP một cách chính xác.

- **Tạo payload cuối cùng:**
  - Tạo payload với độ dài chính xác và địa chỉ hàm `shell` theo định dạng little-endian để khai thác lỗ hổng thành công.

#### 5. Bài học Kinh nghiệm

- **Tầm quan trọng của việc kiểm tra kích thước đầu vào:**
  - Luôn kiểm tra kích thước của dữ liệu đầu vào và đảm bảo rằng nó không vượt quá kích thước của bộ đệm được cấp phát.

- **Sử dụng các công cụ gỡ lỗi và phân tích:**
  - Sử dụng các công cụ như GDB để phân tích và gỡ lỗi chương trình một cách hiệu quả.

- **Hiểu rõ về cấu trúc bộ nhớ:**
  - Nắm vững cách bộ nhớ được bố trí trong stack và các yếu tố có thể ảnh hưởng đến khoảng cách từ bộ đệm đến EIP.

- **Kỹ năng tạo và điều chỉnh payload:**
  - Tạo và điều chỉnh payload một cách chính xác dựa trên kết quả phân tích để khai thác lỗ hổng một cách hiệu quả.

#### 6. Ứng dụng Kiến thức vào Thực tế

- **Viết mã an toàn:**
  - Áp dụng kiến thức về buffer overflow để viết mã an toàn, tránh các lỗ hổng bảo mật tiềm ẩn.

- **Phân tích và gỡ lỗi chương trình:**
  - Sử dụng kỹ năng phân tích và gỡ lỗi để kiểm tra và bảo mật mã nguồn trong các dự án thực tế.

- **Tăng cường bảo mật hệ thống:**
  - Áp dụng các kỹ năng và kiến thức để tăng cường bảo mật cho các hệ thống và ứng dụng, đảm bảo an toàn trước các cuộc tấn công buffer overflow.

---

Bạn có thể sử dụng các điểm trên để trình bày trong báo cáo của mình, nhấn mạnh vào các kinh nghiệm và kiến thức rút ra từ quá trình làm bài tập. Điều này sẽ giúp giáo sư của bạn hiểu rõ hơn về những gì bạn đã học và áp dụng trong bài tập này.
