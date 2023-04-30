---
title: "Pwn 101: Buffer Overflow Part 1"
date: 2023-04-30T00:00:00+07:00
draft: false
ShowToc: true
tags:
    - pwn
    - 101
author:
    - Lio
summary: "Buffer Overflow là gì? Điều gì gây nên lỗi tràn bộ đệm? Làm sao để khai thác?"
---

## Disclaimer

Series **Pwn 101** này tôi viết cho vui trong lúc rảnh nhằm mục đích chia sẻ lại mấy cái kiến thức mà tôi học được (hoặc tự tôi hiểu được nó là như thế) cũng như kinh nghiệm từ khi mới bắt đầu theo mảng Pwnable cho đến giờ. Mấy cái mớ trong này chưa chắc đã đúng hết 100% nhưng nếu thấy không vừa mắt thì cứ coi như là tôi viết linh tinh rồi tắt tab này đi là được.

## Buffer Overflow là gì?

Well, theo như con ChatGpt nó gen cho tôi thì, để chép nguyên văn luôn.

```txt
Lỗi buffer overflow là một lỗi bảo mật phổ biến trong các ứng dụng máy tính, đặc biệt là trong các chương trình được viết bằng các ngôn ngữ lập trình thấp như C và C++. Nó xảy ra khi một chương trình cố gắng ghi vào một vùng nhớ đệm (buffer) vượt quá kích thước đã cấp phát cho nó, gây ra việc ghi đè dữ liệu vào vùng nhớ khác trong bộ nhớ hoặc thậm chí là tràn ra ngoài vùng nhớ của chương trình, gây ra lỗi hoặc crash chương trình.
```

Nói chung là, chương trình nó cho bạn cái vùng nhớ rộng 10 bytes, nhưng không nói là bạn được viết bao nhiêu byte, thế là bạn viết cmn 11 bytes hay thậm chí 69420 bytes nó vẫn lấy hết và điều này gây ra lỗi.

Lỗi Buffer Overflow (tràn bộ đệm) là một trong những lỗi chương trình phổ biến mà các pwner thường khai thác, và cùng là nền tảng cho những kỹ thuật pwn phức tạp như ret2shell, ret2libc, ROP, ...

## Định vị Buffer Overflow như thế nào?

Cái này thì vô số biến thể luôn, nhưng chung quy lại thì vẫn là chương trình nó cho phép bạn ghi vào một cái vùng nhớ nhiều dữ liệu hơn mức cái vùng nhớ đó chứa được. Để dễ tiếp cận hơn một cách cơ bản thì hãy cùng xem qua ví dụ sau.

```c
#include <stdio.h>

void main() {
    int num = 0;
    char buffer[10];
    gets(buffer);
    return 0;
}
```

Một đoạn code C trông có vẻ vô hại. Chương trình này khai báo một biến `num` kiểu int có giá trị bằng 0 và một biến `buffer` là dãy 10 ký tự kiểu char. Sau đó chương trình tiến hành lấy chuỗi input từ người dùng nhập vào biến ‘buffer’ và kết thúc chương trình mà không làm gì cả.

Nhưng nếu chương trình thật sự làm gì đó thì sao? Hãy modify cái chương trình này lại một xíu nhé.

```c
#include <stdio.h>

void main() {
    int num = 0;
    char buffer[10];
    gets(buffer);
    if(num != 0) {
        puts("Wait... How?!");
    }
    return 0;
}
```

Tại đây ta thấy, sau khi chương trình nhận vào input của người dùng bỏ vào biến `buffer` thì sẽ kiểm tra giá trị của biến `num` xem có khác giá trị 0 hay không. Trong suốt quá trình chạy chương trình, ta không hề thực hiện bất kỳ một tác vụ nào làm thay đổi biến `num` nên xét theo lẽ thường, điều kiện if không thể đạt được.

Vấn đề đặt ra ở đây là liệu có cách nào để thay đổi biến `num` khi chạy chương trình trên hay không? Câu trả lời là có.

## Memory Layout trên kiến trúc Intel

Thông thường, trong kiểu kiến trúc intel x86, các biến local của hàm khi khai báo sẽ được lưu vào trong vùng nhớ stack, và khi ta compile chương trình bằng gcc, các biến có kiểu dữ liệu kích thước cố định sẽ được ưu tiên xếp ở dưới.

Thêm một điều nữa, vì intel x86 là kiểu kiến trúc Little Endian nên ngoại trừ các chuỗi ký tự, khi chương trình tiến hành đọc hay lưu dữ liệu tại địa chỉ của vùng nhớ bất kỳ thì các byte giá trị sẽ được xếp ngược lại. Tức là nếu địa chỉ chứa một biến có kiểu dữ liệu int (4 bytes) đang hiển thị là 0x04030201 thì các byte của nó có thứ tự là 01 02 03 04.

Vậy nên theo như các biến ta đã khai báo như trên thì vùng nhớ sẽ cơ bản được thiết lập như sau:

```txt
                bắt đầu của buffer
                      |
                      V
                | 00 00       |
   kết thúc     | 00 00 00 00 |
  của buffer -> | 00 00 00 00 |
                | 00 00 00 00 | <- kết thúc của num
                  ^
                  |
            bắt đầu của num
```

Do đó ta có thể thấy, khác với địa chỉ bắt đầu của chuỗi `buffer` là nằm ở trên cùng và đọc dần xuống dưới cuối theo thứ tự từ trái sang phải, khi ta đọc biến `num` thì chương trình sẽ tự hiểu và đọc ngược các byte từ cuối về đầu, tức là từ trái sang phải. Đừng hỏi tôi, cái memory nó xếp như vậy tôi đâu có thiết kế mấy cái củ shit này.

Ông nào học môn Kiến trúc máy tính xong có khi sẽ hiểu sơ sơ cái này còn không thì tạm thời cứ cho là như vậy trước đi.

## Việc nguy hiểm khi sử dụng hàm không có bảo vệ

Quay trở lại vấn đề chính, đầu tiên hãy nhìn vào Linux Programmer’s Manual dành cho hàm **gets**. Ta biết được rằng hàm **gets** đọc một dòng từ stdin vào bộ đệm (buffer) được trỏ tới bởi tham số con trỏ s và dừng lại cho đến khi gặp ký tự xuống dòng (‘\n’) hoặc ký tự EOF (End of File). Không những thế, ta con nhận được một dòng cảnh báo được gạch chân rất rõ ràng: “Đừng bao giờ sử dụng hàm này.”

```txt
GETS(3)                 Linux Programmer's Manual                GETS(3)
NAME
       gets - get a string from standard input (DEPRECATED)
SYNOPSIS
       #include <stdio.h>

       char *gets(char *s);
DESCRIPTION
       Never use this function.

       gets() reads a line from stdin into the buffer pointed to by s
       until either a terminating newline or EOF, which it replaces with
       a null byte ('\0').  No check for buffer overrun is performed
       (see BUGS below).
RETURN VALUE
       gets() returns s on success, and NULL on error or when end of
       file occurs while no characters have been read.  However, given
       the lack of buffer overrun checking, there can be no guarantees
       that the function will even return.
BUGS
       Never use gets().  Because it is impossible to tell without
       knowing the data in advance how many characters gets() will read,
       and because gets() will continue to store characters past the end
       of the buffer, it is extremely dangerous to use.  It has been
       used to break computer security.  Use fgets() instead.
```

Bản chất của hàm **gets** là đọc vào chuỗi ký tự từ người dùng hoặc từ file, tuy nhiên nó lại không hề kiểm tra số lượng ký tự mà nó nhận vào. Chính do đó mà cho đến khi chưa gặp ký tự xuống dòng hoặc EOF thì nó vẫn sẽ tiếp tục đọc, và điều này dẫn đến một lỗi vô cùng phổ biến đó chính là Buffer Overflow.

## Khai thác Buffer Overflow để thay đổi biến local

Giả sử ta nhập vào một chuỗi 10 ký tự ‘a’ (mã hex 0x61). Hàm **gets** đọc vào chuỗi này và tiến hành lưu vào trong vùng nhớ từ bắt đầu của biến `buffer` cho đến kết thúc của biến `buffer`, vừa vặn 10 bytes. Mọi thứ diễn ra như bình thường, chương trình chạy đúng như những gì chúng ta dự đoán lúc trước và không có điều gì kỳ lạ xảy ra cả.

Lúc này vùng nhớ của ta sẽ như sau:

```txt
                bắt đầu của buffer
                      |
                      V
                | 61 61       |
   kết thúc     | 61 61 61 61 |
  của buffer -> | 61 61 61 61 |
                | 00 00 00 00 | <- kết thúc của num
                  ^
                  |
            bắt đầu của num
```

Nhưng nếu ta nhập vào chuỗi 11 ký tự ‘a’ thì sao?

Hàm **gets** vẫn sẽ nhận vào chuỗi 11 ký tự đó, tiến hành lưu vào vùng nhớ từ bắt đầu của biến `buffer` cho đến kết thúc của biến `buffer`, nhưng độ dài vùng nhớ của biến này chỉ có 10 bytes. Câu hỏi đặt ra là: byte thứ 11 sẽ đi về đâu?

Câu trả lời chính là nó sẽ tràn xuống vùng nhớ ở dưới, không đâu khác chính là vùng nhớ của biến `num`.

```txt
                bắt đầu của buffer
                      |
                      V
                | 61 61       |
   kết thúc     | 61 61 61 61 |
  của buffer -> | 61 61 61 61 |
                | 00 00 00 61 | <- kết thúc của num
                  ^
                  |
            bắt đầu của num
```

Vậy khi này nếu như chương trình tiến hành đọc dữ liệu của biến `num` thì nó sẽ không còn mang giá trị 0 như ban đầu nữa mà lại là 0x61 (hay 0x00000061), tức bằng 97. Ta đã có thể thay đổi được giá trị của biến `num` và điều chỉnh được luồng thực thi của chương trình.

```shell
$ ./test
aaaaaaaaaaa
Wait... How?!
```

Nếu nhập vào nhiều ký tự hơn nữa, ta không những có thể ghi đè lên biến `num` mà còn tiếp tục ghi đè xuống được vùng nhớ ở dưới biến `num`. Nhưng vấn đề này sẽ được đè cập tới ở phần tiếp theo, trong một kỹ thuật khác bắt nguồn từ việc khai thác lỗi Buffer Overflow này.

Một lưu ý nho nhỏ, là đối với các phiên bản gcc mới hơn, compiler sẽ luôn ưu tiên đặt các biến buffer ở vùng địa chỉ cao hơn các biến riêng lẻ khác (tức là nằm ở dưới trong stack) vậy nên việc ghi đè lên biến `num` như ví dụ trên sẽ không còn khả thi nữa.

Trên thực tế, không chỉ mỗi hàm **gets** mà còn có nhiều những hàm khác có thể gây nên lỗi tràn bộ đệm. Và việc tràn bộ đệm này có thể giúp các attacker thay đổi được luồng thực thi của chương trình, và hơn nữa là chiếm được shell của hệ thống.

<!-- ![clown](https://media.discordapp.net/attachments/950580594788687932/1102262087067127808/shell.png?width=502&height=675) -->