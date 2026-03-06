# ADR 039: Epoched Staking (Staking Theo Epoch)

## Nhật Ký Thay Đổi

* 10-02-2021: Bản nháp đầu tiên

## Tác Giả

* Dev Ojha (@valardragon)
* Sunny Aggarwal (@sunnya97)

## Trạng Thái

Đề Xuất

## Tóm Tắt

ADR này cập nhật module proof of stake để đệm các cập nhật trọng số staking trong một số block trước khi cập nhật trọng số staking của đồng thuận. Độ dài của bộ đệm được gọi là epoch. Chức năng trước đó của module staking sau đó là một trường hợp đặc biệt của module được trừu tượng hóa, với epoch được đặt là 1 block.

## Bối Cảnh

Module proof of stake hiện tại đưa ra quyết định thiết kế áp dụng các thay đổi trọng số staking cho engine đồng thuận ngay lập tức. Điều này có nghĩa là các delegation và unbond được áp dụng ngay lập tức cho tập validator. Quyết định này chủ yếu được thực hiện vì nó đơn giản nhất từ góc độ triển khai, và vì vào thời điểm đó chúng ta tin rằng điều này sẽ dẫn đến UX tốt hơn cho các client.

Một lựa chọn thiết kế thay thế là cho phép đệm các cập nhật staking (delegation, unbond, validator tham gia) trong một số block. Proof of stake theo epoch này cung cấp đảm bảo rằng trọng số đồng thuận cho các validator sẽ không thay đổi trong giữa epoch, ngoại trừ trường hợp có điều kiện slash.

Ngoài ra, rào cản UX có thể không đáng kể như đã nghĩ trước đây. Điều này là vì có thể cung cấp cho người dùng xác nhận ngay lập tức rằng bond của họ đã được ghi lại và sẽ được thực thi.

Hơn nữa, theo thời gian ngày càng rõ ràng rằng việc thực thi ngay lập tức các sự kiện staking đi kèm với các hạn chế, chẳng hạn như:

* **Mật mã học dựa trên ngưỡng.** Một trong những hạn chế chính là vì tập validator có thể thay đổi thường xuyên, nó làm cho việc chạy tính toán đa bên bởi một tập validator cố định trở nên khó khăn. Nhiều tính năng mật mã học dựa trên ngưỡng cho blockchain như đèn hiệu ngẫu nhiên và giải mã ngưỡng đòi hỏi quy trình DKG tốn kém tính toán (sẽ mất nhiều hơn 1 block để tạo). Để sử dụng hiệu quả những điều này, chúng ta cần đảm bảo rằng kết quả của DKG sẽ được sử dụng trong một thời gian hợp lý. Sẽ không khả thi để chạy lại DKG mỗi block. Bằng cách epoch hóa staking, nó đảm bảo chúng ta chỉ cần chạy DKG mới một lần mỗi epoch.

* **Hiệu quả light client.** Điều này sẽ giảm bớt overhead cho IBC khi có sự biến động cao trong tập validator. Trong thuật toán bisection light client Tendermint, số lượng header bạn cần xác minh liên quan đến việc giới hạn sự khác biệt trong tập validator giữa header được tin cậy và header mới nhất. Nếu sự khác biệt quá lớn, bạn xác minh nhiều header hơn ở giữa. Bằng cách hạn chế tần suất thay đổi tập validator, chúng ta có thể giảm kích thước trường hợp xấu nhất của các bằng chứng lite client IBC, xảy ra khi tập validator có biến động cao.

* **Sự công bằng trong bầu leader xác định.** Hiện tại chúng ta không có cách nào để lý luận về sự công bằng của bầu leader xác định khi có thay đổi staking mà không có epoch (tendermint/spec#217). Phá vỡ sự công bằng của bầu leader có lợi nhuận cho các validator, vì họ kiếm thêm phần thưởng từ việc là người đề xuất. Việc thêm epoch ít nhất làm cho bầu leader xác định của chúng ta dễ dàng khớp với một điều gì đó chúng ta có thể chứng minh là an toàn. (Mặc dù vậy, chúng ta vẫn chưa chứng minh liệu thuật toán hiện tại có công bằng với > 2 validator khi có thay đổi stake hay không)

* **Thiết kế dẫn xuất staking.** Hiện tại, phân phối phần thưởng được thực hiện lười biếng sử dụng phân phối phí F1. Trong khi tiết kiệm độ phức tạp tính toán, kế toán lười biếng đòi hỏi triển khai staking có nhiều trạng thái hơn. Hiện tại, mỗi mục delegation phải theo dõi thời gian rút tiền lần cuối. Việc xử lý điều này có thể là thách thức đối với một số thiết kế dẫn xuất staking tìm kiếm tính fungibility cho tất cả các token được staked cho một validator duy nhất. Buộc rút tiền phần thưởng có thể giúp giải quyết điều này, tuy nhiên không khả thi để buộc rút tiền phần thưởng cho người dùng trên cơ sở mỗi block. Với epoch, một chain có thể dễ dàng thay đổi thiết kế để có phần thưởng được rút bắt buộc (chỉ lặp qua các tài khoản delegator một lần mỗi epoch), và do đó có thể loại bỏ thời gian delegation khỏi trạng thái. Điều này có thể hữu ích cho một số thiết kế dẫn xuất staking nhất định.

## Cân Nhắc Thiết Kế

### Slash

Có một cân nhắc thiết kế về việc áp dụng slash ngay lập tức hay vào cuối epoch. Một sự kiện slash nên chỉ áp dụng cho các thành viên thực sự được staked trong thời gian của vi phạm, cụ thể là trong epoch xảy ra sự kiện slash.

Việc áp dụng ngay lập tức có thể được coi là cung cấp bảo mật lớp đồng thuận cao hơn, với chi phí tiềm năng cho các trường hợp sử dụng đã đề cập. Lợi ích của việc slash ngay lập tức cho bảo mật lớp đồng thuận đều có thể đạt được bằng cách thực thi việc jail validator ngay lập tức (do đó loại bỏ nó khỏi tập validator), và trì hoãn thay đổi slash thực sự đối với trọng số của validator cho đến ranh giới epoch. Đối với các trường hợp sử dụng được đề cập ở trên, các cách giải quyết có thể được tích hợp để tránh các vấn đề, như sau:

* Đối với mật mã học dựa trên ngưỡng, thiết lập này sẽ sử dụng trọng số epoch gốc, trong khi đồng thuận có một cập nhật cho phép nó hưởng lợi nhanh hơn từ bảo mật bổ sung.
* Đối với hiệu quả light client, có thể có một bit được bao gồm trong header chỉ ra slash trong epoch (ala https://github.com/tendermint/spec/issues/199).
* Đối với sự công bằng trong bầu leader xác định, áp dụng slash hoặc jail trong epoch sẽ phá vỡ đảm bảo chúng ta tìm cách cung cấp.
* Đối với thiết kế dẫn xuất staking, không có vấn đề gì được giới thiệu. Điều này không tăng kích thước trạng thái của các bản ghi staking, vì liệu có xảy ra slash hay không là hoàn toàn có thể truy vấn được dựa trên địa chỉ validator.

### Khóa Token

Khi ai đó thực hiện giao dịch để delegation, mặc dù họ không được staked ngay lập tức, token của họ nên được chuyển vào pool được quản lý bởi module staking, pool này sau đó sẽ được sử dụng vào cuối epoch. Điều này ngăn ngừa các mối lo ngại nơi họ stake, và sau đó chi tiêu những token đó mà không nhận ra chúng đã được phân bổ cho staking, và do đó làm cho tx staking của họ thất bại.

### Pipeline Epoch

Đặc biệt đối với mật mã học dựa trên ngưỡng, chúng ta cần một pipeline cho các thay đổi epoch. Điều này là vì khi chúng ta ở trong epoch N, chúng ta muốn trọng số epoch N+1 được cố định để tập validator có thể thực hiện DKG tương ứng. Vì vậy, nếu chúng ta hiện đang ở epoch N, trọng số stake cho epoch N+1 nên đã được cố định, và các thay đổi stake mới nên được áp dụng cho epoch N + 2.

Điều này có thể được xử lý bằng cách tạo tham số cho độ dài pipeline epoch. Tham số này không nên thay đổi được ngoại trừ trong các hard fork, để giảm thiểu độ phức tạp triển khai khi chuyển đổi độ dài pipeline.

Với độ dài pipeline 1, nếu tôi redelegate trong epoch N, thì redelegation của tôi được áp dụng trước khi bắt đầu epoch N+1.
Với độ dài pipeline 2, nếu tôi redelegate trong epoch N, thì redelegation của tôi được áp dụng trước khi bắt đầu epoch N+2.

### Phần Thưởng

Mặc dù tất cả các cập nhật staking được áp dụng tại ranh giới epoch, phần thưởng vẫn có thể được phân phối ngay lập tức khi chúng được yêu cầu. Điều này là vì chúng không ảnh hưởng đến trọng số stake hiện tại, vì chúng ta không triển khai auto-bonding phần thưởng. Nếu tính năng như vậy được triển khai, nó sẽ phải được thiết lập sao cho phần thưởng được auto-bonded tại ranh giới epoch.

### Tham Số Hóa Độ Dài Epoch

Khi chọn độ dài epoch, có sự đánh đổi giữa tích lũy trạng thái/tính toán trong hàng đợi, và chống lại các hạn chế đã thảo luận trước đó của việc thực thi ngay lập tức nếu chúng áp dụng cho một chain nhất định.

Cho đến khi một cơ chế ABCI cho thời gian block thay đổi được giới thiệu, không khuyến nghị sử dụng độ dài epoch cao do tích lũy tính toán. Điều này là vì khi thời gian thực thi của một block lớn hơn thời gian block mong đợi từ Tendermint, các vòng có thể tăng lên.

## Quyết Định

**Bước 1**: Triển khai đệm tất cả các message staking và slashing.

Đầu tiên chúng ta tạo một pool để lưu trữ các token đang được bonded, nhưng nên được áp dụng tại ranh giới epoch gọi là `EpochDelegationPool`. Sau đó, chúng ta có hai hàng đợi riêng biệt, một cho staking, một cho slashing. Chúng ta mô tả những gì xảy ra khi mỗi message được giao dưới đây:

### Các Message Staking

* **MsgCreateValidator**: Chuyển tự-bond của người dùng vào `EpochDelegationPool` ngay lập tức. Xếp hàng một message cho ranh giới epoch để xử lý tự-bond, lấy tiền từ `EpochDelegationPool`. Nếu thực thi Epoch thất bại, trả lại tiền từ `EpochDelegationPool` về tài khoản người dùng.
* **MsgEditValidator**: Xác nhận message và nếu hợp lệ xếp hàng message để thực thi vào cuối Epoch.
* **MsgDelegate**: Chuyển tiền của người dùng vào `EpochDelegationPool` ngay lập tức. Xếp hàng một message cho ranh giới epoch để xử lý delegation, lấy tiền từ `EpochDelegationPool`. Nếu thực thi Epoch thất bại, trả lại tiền từ `EpochDelegationPool` về tài khoản người dùng.
* **MsgBeginRedelegate**: Xác nhận message và nếu hợp lệ xếp hàng message để thực thi vào cuối Epoch.
* **MsgUndelegate**: Xác nhận message và nếu hợp lệ xếp hàng message để thực thi vào cuối Epoch.

### Các Message Slashing

* **MsgUnjail**: Xác nhận message và nếu hợp lệ xếp hàng message để thực thi vào cuối Epoch.
* **Sự kiện Slash**: Bất cứ khi nào một sự kiện slash được tạo ra, nó được xếp hàng trong module slashing để áp dụng vào cuối epoch. Các hàng đợi nên được thiết lập sao cho slash này được áp dụng ngay lập tức.

### Các Message Bằng Chứng

* **MsgSubmitEvidence**: Được thực thi ngay lập tức, và validator bị jail ngay lập tức. Tuy nhiên trong slashing, sự kiện slash thực tế được xếp hàng.

Sau đó chúng ta thêm các phương thức vào end blockers, để đảm bảo rằng tại ranh giới epoch các hàng đợi được xóa và cập nhật delegation được áp dụng.

**Bước 2**: Triển khai truy vấn các tx staking trong hàng đợi.

Khi truy vấn hoạt động staking của một địa chỉ đã cho, trạng thái nên trả về không chỉ số lượng token được staked, mà còn nếu có bất kỳ sự kiện stake nào trong hàng đợi cho địa chỉ đó. Điều này sẽ đòi hỏi nhiều công việc hơn trong logic truy vấn, để theo dõi các sự kiện staking sắp tới trong hàng đợi.

Như triển khai ban đầu, điều này có thể được triển khai như một tìm kiếm tuyến tính qua tất cả các sự kiện staking trong hàng đợi. Tuy nhiên, đối với các chain cần epoch dài, cuối cùng chúng nên xây dựng hỗ trợ bổ sung cho các node hỗ trợ truy vấn để có thể tạo ra kết quả trong thời gian không đổi. (Điều này có thể thực hiện bằng cách duy trì một hashmap phụ trợ để lập chỉ mục các sự kiện staking sắp tới theo địa chỉ)

**Bước 3**: Điều chỉnh gas

Hiện tại gas đại diện cho chi phí thực thi một giao dịch khi được thực hiện ngay lập tức. (Gộp chi phí của p2p overhead, chi phí truy cập trạng thái và chi phí tính toán) Tuy nhiên, bây giờ một giao dịch có thể gây ra tính toán trong một block tương lai, cụ thể là tại ranh giới epoch.

Để xử lý điều này, chúng ta ban đầu nên bao gồm các tham số để ước tính lượng tính toán trong tương lai (tính theo gas), và thêm điều đó như một khoản phí cố định cần thiết cho message. Chúng ta để ngoài phạm vi cách tính trọng số tính toán trong tương lai so với tính toán hiện tại trong giá gas, và đặt cho chúng được tính trọng số bằng nhau hiện tại.

## Hậu Quả

### Tích Cực

* Trừu tượng hóa module proof of stake cho phép giữ nguyên chức năng hiện có
* Cho phép các tính năng mới như mật mã học dựa trên ngưỡng cho tập validator

### Tiêu Cực

* Tăng độ phức tạp của việc tích hợp các cơ chế giá gas phức tạp hơn, vì chúng giờ phải xem xét cả chi phí thực thi trong tương lai.
* Khi epoch > 1, các validator không thể rời mạng ngay lập tức và phải chờ đến ranh giới epoch.
