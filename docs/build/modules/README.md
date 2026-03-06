---
sidebar_position: 0
---

# Danh sách module

Dưới đây là một số module “production-grade” có thể dùng trong các ứng dụng Cosmos SDK, kèm theo tài liệu tương ứng:

## Module thiết yếu

Module thiết yếu bao gồm các chức năng _bắt buộc_ phải có trong blockchain Cosmos SDK của bạn.
Các module này cung cấp các hành vi cốt lõi cần thiết cho người dùng và operator như theo dõi số dư,
khả năng proof-of-stake và governance.

* [Auth](./auth/README.md) - Xác thực tài khoản và giao dịch cho các ứng dụng Cosmos SDK.
* [Bank](./bank/README.md) - Chức năng chuyển token.
* [Circuit](./circuit/README.md) - Module “cầu dao” (circuit breaker) để tạm dừng các message.
* [Consensus](./consensus/README.md) - Module consensus để sửa các tham số đồng thuận ABCI của CometBFT.
* [Distribution](./distribution/README.md) - Phân phối phí và phân phối phần cung ứng token staking.
* [Evidence](./evidence/README.md) - Xử lý bằng chứng cho double signing, hành vi sai trái, v.v.
* [Governance](./gov/README.md) - Proposal và bỏ phiếu on-chain.
* [Genutil](./genutil/README.md) - Tiện ích genesis cho Cosmos SDK.
* [Mint](./mint/README.md) - Tạo các đơn vị mới của token staking.
* [Slashing](./slashing/README.md) - Cơ chế trừng phạt validator.
* [Staking](./staking/README.md) - Tầng Proof-of-Stake cho blockchain công khai.
* [Upgrade](./upgrade/README.md) - Xử lý và điều phối nâng cấp phần mềm.

## Module bổ trợ

Module bổ trợ là các module được duy trì trong Cosmos SDK nhưng không cần thiết cho
chức năng cốt lõi của blockchain. Có thể xem chúng như các cách để mở rộng khả năng
của blockchain hoặc chuyên biệt hoá thêm.

* [Authz](./authz/README.md) - Uỷ quyền để tài khoản thực hiện hành động thay mặt tài khoản khác.
* [Epochs](./epochs/README.md) - Đăng ký để các module SDK có thể có logic được thực thi theo các “nhịp” theo thời gian.
* [Feegrant](./feegrant/README.md) - Cấp hạn mức trả phí để thực thi giao dịch.
* [ProtocolPool](./protocolpool/README.md) - Quản lý mở rộng cho chức năng community pool.

## Module đã bị deprecate

Các module sau đã bị deprecate. Chúng sẽ không còn được duy trì và cuối cùng sẽ bị
loại bỏ trong một bản phát hành sắp tới của Cosmos SDK theo [quy trình phát hành](https://github.com/cosmos/cosmos-sdk/blob/main/RELEASE_PROCESS.md).

* [Crisis](./crisis/README.md) - _Deprecated_ dừng blockchain trong một số trường hợp nhất định (ví dụ nếu một invariant bị phá vỡ).
* [Params](./params/README.md) - _Deprecated_ kho tham số toàn cục.
* [NFT](./nft/README.md) - _Deprecated_ module NFT triển khai dựa trên [ADR43](https://docs.cosmos.network/main/build/architecture/adr-043-nft-module). Module này sẽ được chuyển sang repo `cosmos-sdk-legacy` để sử dụng.
* [Group](./group/README.md) - _Deprecated_ cho phép tạo và quản lý tài khoản multisig on-chain. Module này sẽ được chuyển sang repo `cosmos-sdk-legacy` để sử dụng “legacy”.

Để tìm hiểu thêm về quy trình xây dựng module, hãy xem [tài liệu tham chiếu về xây dựng module](https://docs.cosmos.network/main/building-modules/intro).

## IBC

Module IBC cho SDK được đội IBC Go duy trì trong [repo riêng](https://github.com/cosmos/ibc-go).

Ngoài ra, [module capability](https://github.com/cosmos/ibc-go/tree/fdd664698d79864f1e00e147f9879e58497b5ef1/modules/capability) từ v0.50+ được đội IBC Go duy trì trong [repo riêng](https://github.com/cosmos/ibc-go/tree/fdd664698d79864f1e00e147f9879e58497b5ef1/modules/capability).

## CosmWasm

Module CosmWasm cho phép smart contract; tìm hiểu thêm tại [trang tài liệu](https://book.cosmwasm.com/) hoặc xem [repo](https://github.com/CosmWasm/cosmwasm).

## EVM

Đọc thêm về viết smart contract bằng Solidity tại trang tài liệu chính thức của [`evm`](https://evm.cosmos.network/).

