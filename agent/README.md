[TOC]



# Agent

## Các khái niệm quan trọng

### Rules

> Đơn giản là một tập hợp các hướng dẫn (instructions) mà AI sẽ luôn tuân theo khi chat hoặc tạo code. Nó giống như việc bạn "dạy" AI về thói quen và quy chuẩn của bạn trước khi bắt đầu làm việc để tránh lặp lại trong mỗi đoạn chat.

`Cursor` cung cấp nhiều loại rule như user, project và sử dụng một số format đặc biệt nhưng chúng ta chỉ follow theo duy nhất 1 pattern đơn giản đó là sử dụng `AGENTS.md` nằm ngày luôn ở root folder
**Trong này mục đích chính:**

- Hiểu Context dự án: Giải thích cấu trúc thư mục, các quy tắc đặt tên (naming convention).
- Working protocol: AI hành động đúng role, những core principles, protocol cần follow ví dụ cần phải đọc những tài liều nào(overview), rồi những file overview sẽ có path theo pattern như thế nào
- Task-Specific Directives: AI phải hiểu được task mình đang làm là gì, với mỗi task thì sẽ có những yêu cầu cụ thể khác nhau  
- Đồng bộ style code: Coding Standards & Quality

### Commands

> Command sẽ được gọi nhanh thông qua `/`
> Nó define 1 specific task intruction. Ví dụ như giúp mình to_en(translate), en(giải thích từ)...

Có 2 loại là project command và global command. Để thuận tiện, chúng ta sẽ chỉ sử dụng global command và có script để giúp copy command folder trong devdocs folder ra global ở `~/.cursor/commands`

### Skills

### Templates

AI sẽ render theo các template mà chúng ta mong muốn