---
sidebar_position: 1
---

# Cấu Hình

Tài liệu này đề cập đến file app.toml. Nếu bạn muốn đọc về config.toml, vui lòng truy cập [tài liệu CometBFT](https://docs.cometbft.com/v0.37/).

<!-- phần sau không phải tham chiếu python, tuy nhiên tô màu cú pháp giúp file dễ đọc hơn trong tài liệu -->
```python reference
https://github.com/cosmos/cosmos-sdk/blob/main/tools/confix/data/v0.47-app.toml 
```

## inter-block-cache

Tính năng này sẽ tiêu thụ nhiều RAM hơn một node thông thường nếu được bật.

## iavl-cache-size

Sử dụng tính năng này sẽ làm tăng mức tiêu thụ RAM.

## iavl-lazy-loading

Tính năng này dành cho các archive node, cho phép chúng có thời gian khởi động nhanh hơn.
