# ADR 007: Nhóm Chuyên Biệt Hóa

## Changelog

* 31 tháng 7 năm 2019: Bản nháp đầu tiên

## Bối Cảnh

Ý tưởng này được hình thành lần đầu tiên để đáp ứng trường hợp sử dụng tạo một Đội Phản Ứng Khẩn Cấp Máy Tính Phi Tập Trung (dCERT), các thành viên của đội sẽ được cộng đồng quản trị bầu chọn và sẽ đóng vai trò điều phối cộng đồng trong các tình huống khẩn cấp. Suy nghĩ này có thể được trừu tượng hóa thêm thành khái niệm "nhóm chuyên biệt hóa blockchain".

Việc tạo ra các nhóm này là khởi đầu của các khả năng chuyên biệt hóa trong một cộng đồng blockchain rộng lớn hơn, có thể được dùng để cho phép một mức độ trách nhiệm được ủy quyền nhất định. Ví dụ về chuyên biệt hóa có thể có lợi cho cộng đồng blockchain bao gồm: kiểm toán code, ứng phó khẩn cấp, phát triển code, v.v. Loại tổ chức cộng đồng này mở đường cho các stakeholder cá nhân ủy quyền phiếu bầu theo loại vấn đề, nếu trong tương lai các đề xuất quản trị bao gồm một trường cho loại vấn đề.

## Quyết Định

Một nhóm chuyên biệt hóa có thể được chia nhỏ thành các chức năng sau (kèm theo ví dụ):

* Tiếp Nhận Thành Viên
* Chấp Nhận Thành Viên
* Thu Hồi Thành Viên
    * (có thể) Không Có Hình Phạt
        * thành viên từ chức (tự Thu Hồi)
        * được thay thế bởi thành viên mới từ quản trị
    * (có thể) Có Hình Phạt
        * do vi phạm thỏa thuận mềm (được xác định thông qua quản trị)
        * do vi phạm thỏa thuận cứng (được xác định bởi code)
* Thực Hiện Nhiệm Vụ
    * Các giao dịch đặc biệt chỉ thực thi cho thành viên của nhóm chuyên biệt hóa (ví dụ: thành viên dCERT bỏ phiếu để tắt các route giao dịch trong tình huống khẩn cấp)
* Bồi Thường
    * Bồi thường nhóm (phân phối thêm do nhóm chuyên biệt hóa quyết định)
    * Bồi thường cá nhân cho tất cả thành phần của nhóm từ cộng đồng lớn hơn

Việc tiếp nhận thành viên vào nhóm chuyên biệt hóa có thể diễn ra qua nhiều cơ chế khác nhau. Ví dụ rõ ràng nhất là thông qua bỏ phiếu chung của toàn bộ cộng đồng, tuy nhiên trong một số hệ thống, cộng đồng có thể muốn cho phép các thành viên đã trong nhóm chuyên biệt hóa tự bầu chọn thành viên mới, hoặc cộng đồng có thể giao quyền cho một nhóm chuyên biệt hóa cụ thể để bổ nhiệm thành viên vào các nhóm bên thứ ba khác. Thực sự không giới hạn về cách cấu trúc tiếp nhận thành viên. Chúng ta cố gắng nắm bắt một số khả năng này trong một interface chung gọi là `Electionator`. Để triển khai ban đầu như một phần của ADR này, chúng tôi khuyến nghị cung cấp trừu tượng bầu cử chung (`Electionator`) cũng như một triển khai cơ bản của trừu tượng đó cho phép bầu cử liên tục các thành viên của nhóm chuyên biệt hóa.

``` golang
// Trừu tượng Electionator bao phủ không gian khái niệm cho
// nhiều loại bầu cử khác nhau.
type Electionator interface {

    // liệu đối tượng bầu cử có đang chấp nhận phiếu bầu không.
    Active() bool

    // chức năng thực thi khi một phiếu bầu được cast trong cuộc bầu cử này, ở đây
    // trường vote được kỳ vọng sẽ được marshal thành kiểu vote được sử dụng
    // bởi một cuộc bầu cử.
    Vote(addr sdk.AccAddress, vote []byte)

    // ở đây chứa tất cả chức năng để xác thực và thực thi thay đổi cho
    // khi một thành viên chấp nhận được bầu chọn
    AcceptElection(sdk.AccAddress)

    // Đăng ký một đối tượng revoker
    RegisterRevoker(Revoker)

    // Không thể đăng ký thêm revoker sau khi hàm này được gọi
    SealRevokers()

    // đăng ký hook để gọi khi các sự kiện bầu cử xảy ra
    RegisterHooks(ElectionatorHooks)

    // truy vấn người chiến thắng hiện tại của cuộc bầu cử này dựa trên bộ quy tắc bầu cử tùy ý
    QueryElected() []sdk.AccAddress

    // truy vấn metadata cho một địa chỉ trong cuộc bầu cử này
    // ví dụ có thể bao gồm vị trí mà một địa chỉ
    // đang được bầu chọn trong nhóm
    QueryMetadata(sdk.AccAddress) []byte
}

// ElectionatorHooks, khi đã được đăng ký với Electionator,
// kích hoạt thực thi các hàm interface liên quan khi
// các sự kiện Electionator xảy ra.
type ElectionatorHooks interface {
    AfterVoteCast(addr sdk.AccAddress, vote []byte)
    AfterMemberAccepted(addr sdk.AccAddress)
    AfterMemberRevoked(addr sdk.AccAddress, cause []byte)
}

// Revoker định nghĩa hàm cần thiết cho một bộ quy tắc thu hồi thành viên
// được sử dụng bởi nhóm chuyên biệt hóa. Có thể dùng để tạo tự thu hồi,
// thu hồi dựa trên bằng chứng, v.v. Các kiểu Revoker có thể được tạo và
// tái sử dụng cho các loại bầu cử khác nhau.
//
// Khi thu hồi, các byte "cause" có thể được marshal tùy ý thành bằng chứng,
// memo, v.v.
type Revoker interface {
    RevokeName() string      // định danh cho loại revoker này
    RevokeMember(addr sdk.AccAddress, cause []byte) error
}
```

Một mức độ điểm chung nhất định có thể tồn tại giữa code hiện tại trong `x/governance` và chức năng cần thiết của bầu cử. Chức năng chung này nên được trừu tượng hóa trong quá trình triển khai. Tương tự, đối với mỗi triển khai vote, chức năng CLI/REST của client nên được trừu tượng hóa để tái sử dụng cho nhiều cuộc bầu cử.

Trừu tượng nhóm chuyên biệt hóa trước tiên mở rộng `Electionator` nhưng cũng định nghĩa thêm các đặc điểm của nhóm.

``` golang
type SpecializationGroup interface {
    Electionator
    GetName() string
    GetDescription() string

    // hợp đồng mềm chung nhóm được kỳ vọng
    // thực hiện với cộng đồng lớn hơn
    GetContract() string

    // các message có thể được thực thi bởi thành viên của nhóm
    Handler(ctx sdk.Context, msg sdk.Msg) sdk.Result

    // logic thực thi tại endblock, ví dụ có thể bao gồm
    // thanh toán trợ cấp cho các thành viên nhóm
    // vì tham gia vào nhóm bảo mật.
    EndBlocker(ctx sdk.Context)
}
```

## Trạng Thái

> Đề Xuất

## Hậu Quả

### Tích Cực

* tăng khả năng chuyên biệt hóa của blockchain
* cải thiện các trừu tượng trong `x/gov/` để chúng có thể được sử dụng với các nhóm chuyên biệt hóa

### Tiêu Cực

* có thể được dùng để tăng tập trung hóa trong cộng đồng

### Trung Lập

## Tài Liệu Tham Khảo

* [ADR dCERT](./adr-008-dCERT-group.md)
