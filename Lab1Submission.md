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

![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/907f4a87-3f03-4ece-af56-a12a29aec220)

*Xác nhận Payload được tạo đúng cách.*

![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/5fb9e3eb-ebf9-4fcb-8e4d-9a7cdfcd28c0)


