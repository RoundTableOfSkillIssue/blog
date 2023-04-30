---
title: "Pwn 101: Buffer Overflow Part 1"
date: 2023-04-30T00:00:00+07:00
draft: false
toc: false
tags:
    - pwn
    - 101
author:
    - Lio
summary: "Buffer Overflow là gì? Điều gì gây nên việc tràn bộ đệm này?"
---

Series **`Pwn 101`** này tôi viết cho vui trong lúc rảnh nhằm mục đích chia sẻ lại mấy cái kiến thức mà tôi học được (hoặc tự tôi hiểu được nó là như thế) cũng như kinh nghiệm từ khi mới bắt đầu theo mảng Pwnable cho đến giờ. Mấy cái mớ trong này chưa chắc đã đúng hết 100% nhưng nếu thấy không vừa mắt thì cứ coi như là tôi viết linh tinh rồi tắt tab này đi là được.

## Buffer Overflow là gì?

Well, theo như con ChatGpt nó gen cho tôi thì, để chép nguyên văn luôn.

```
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