# ADR 032: Typed Events (Sự Kiện Có Kiểu)

## Nhật Ký Thay Đổi

* 28-09-2020: Bản nháp đầu tiên

## Tác Giả

* Anil Kumar (@anilcse)
* Jack Zampolin (@jackzampolin)
* Adam Bozanich (@boz)

## Trạng Thái

Đề Xuất

## Tóm Tắt

Hiện tại trong Cosmos SDK, các sự kiện được định nghĩa trong các handler cho mỗi message cũng như `BeginBlock` và `EndBlock`. Mỗi module không có các kiểu được định nghĩa cho mỗi sự kiện, chúng được triển khai dưới dạng `map[string]string`. Trước hết điều này làm cho các sự kiện này khó tiêu thụ vì nó đòi hỏi rất nhiều khớp chuỗi thô và phân tích cú pháp. Đề xuất này tập trung vào việc cập nhật các sự kiện để sử dụng **typed events** (sự kiện có kiểu) được định nghĩa trong mỗi module sao cho việc phát ra và đăng ký các sự kiện sẽ dễ dàng hơn nhiều. Quy trình làm việc này xuất phát từ kinh nghiệm của nhóm Akash Network.

## Bối Cảnh

Hiện tại trong Cosmos SDK, các sự kiện được định nghĩa trong các handler cho mỗi message, có nghĩa là mỗi module không có tập hợp kiểu chuẩn cho mỗi sự kiện. Trước hết điều này làm cho các sự kiện này khó tiêu thụ vì nó đòi hỏi rất nhiều khớp chuỗi thô và phân tích cú pháp. Đề xuất này tập trung vào việc cập nhật các sự kiện để sử dụng **typed events** được định nghĩa trong mỗi module sao cho việc phát ra và đăng ký các sự kiện sẽ dễ dàng hơn nhiều. Quy trình làm việc này xuất phát từ kinh nghiệm của nhóm Akash Network.

[Nền tảng của chúng tôi](http://github.com/ovrclk/akash) đòi hỏi nhiều tương tác lập trình on-chain cả ở phía nhà cung cấp (trung tâm dữ liệu - để đặt giá cho các đơn hàng mới và lắng nghe các hợp đồng thuê được tạo) và người dùng (nhà phát triển ứng dụng - để gửi manifest ứng dụng tới nhà cung cấp). Ngoài ra nhóm Akash hiện đang duy trì IBC [`relayer`](https://github.com/ovrclk/relayer), một quy trình hướng sự kiện khác. Khi làm việc trên các phần cơ sở hạ tầng cốt lõi này và tích hợp bài học từ phát triển Kubernetes, nhóm của chúng tôi đã phát triển một phương thức tiêu chuẩn để định nghĩa và tiêu thụ typed events trong các module Cosmos SDK. Chúng tôi thấy rằng nó cực kỳ hữu ích trong việc xây dựng loại ứng dụng hướng sự kiện này.

Khi Cosmos SDK được sử dụng rộng rãi hơn cho các ứng dụng như `peggy`, các peg zone khác, IBC, DeFi, v.v... sẽ có nhu cầu bùng nổ về các ứng dụng hướng sự kiện để hỗ trợ các tính năng mới mà người dùng mong muốn. Chúng tôi đề xuất đưa các phát hiện của mình vào Cosmos SDK để cho phép tất cả các ứng dụng Cosmos SDK nhanh chóng và dễ dàng xây dựng các ứng dụng hướng sự kiện hỗ trợ ứng dụng cốt lõi của họ. Ví, sàn giao dịch, trình khám phá và các giao thức DeFi đều được hưởng lợi từ công việc này.

Nếu đề xuất này được chấp nhận, người dùng sẽ có thể xây dựng các ứng dụng Cosmos SDK hướng sự kiện bằng go chỉ bằng cách viết `EventHandler` cho các kiểu sự kiện cụ thể của họ và truyền chúng tới `EventEmitter` được định nghĩa trong Cosmos SDK.

Cuối đề xuất này chứa ví dụ chi tiết về cách tiêu thụ các sự kiện sau khi tái cấu trúc này.

Đề xuất này đặc biệt về cách tiêu thụ các sự kiện này như một client của blockchain, không phải cho giao tiếp liên module.

## Quyết Định

**Bước 1**: Triển khai chức năng bổ sung trong gói `types`: hàm `EmitTypedEvent` và `ParseTypedEvent`

```go
// types/events.go

// EmitTypedEvent nhận typed event và phát ra chuyển đổi nó thành sdk.Event
func (em *EventManager) EmitTypedEvent(event proto.Message) error {
	evtType := proto.MessageName(event)
	evtJSON, err := codec.ProtoMarshalJSON(event)
	if err != nil {
		return err
	}

	var attrMap map[string]json.RawMessage
	err = json.Unmarshal(evtJSON, &attrMap)
	if err != nil {
		return err
	}

	var attrs []abci.EventAttribute
	for k, v := range attrMap {
		attrs = append(attrs, abci.EventAttribute{
			Key:   []byte(k),
			Value: v,
		})
	}

	em.EmitEvent(Event{
		Type:       evtType,
		Attributes: attrs,
	})

	return nil
}

// ParseTypedEvent chuyển đổi abci.Event trở lại thành typed event
func ParseTypedEvent(event abci.Event) (proto.Message, error) {
	concreteGoType := proto.MessageType(event.Type)
	if concreteGoType == nil {
		return nil, fmt.Errorf("failed to retrieve the message of type %q", event.Type)
	}

	var value reflect.Value
	if concreteGoType.Kind() == reflect.Ptr {
		value = reflect.New(concreteGoType.Elem())
	} else {
		value = reflect.Zero(concreteGoType)
    }

	protoMsg, ok := value.Interface().(proto.Message)
	if !ok {
		return nil, fmt.Errorf("%q does not implement proto.Message", event.Type)
	}

	attrMap := make(map[string]json.RawMessage)
	for _, attr := range event.Attributes {
		attrMap[string(attr.Key)] = attr.Value
	}

	attrBytes, err := json.Marshal(attrMap)
	if err != nil {
		return nil, err
	}

	err = jsonpb.Unmarshal(strings.NewReader(string(attrBytes)), protoMsg)
	if err != nil {
		return nil, err
	}

	return protoMsg, nil
}
```

Ở đây, `EmitTypedEvent` là một phương thức trên `EventManager` nhận typed event như đầu vào và áp dụng tuần tự hóa json trên nó. Sau đó nó ánh xạ các cặp key/value JSON tới `event.Attributes` và phát ra dưới dạng `sdk.Event`. `Event.Type` sẽ là URL kiểu của proto message.

Khi chúng ta đăng ký các sự kiện được phát ra trên websocket CometBFT, chúng được phát ra dưới dạng `abci.Event`. `ParseTypedEvent` phân tích sự kiện trở lại thành proto message gốc của nó.

**Bước 2**: Thêm định nghĩa proto cho typed events cho các msg trong mỗi module:

Ví dụ, hãy lấy `MsgSubmitProposal` của module `gov` và triển khai kiểu của sự kiện này.

```protobuf
// proto/cosmos/gov/v1beta1/gov.proto
// Thêm định nghĩa typed event

package cosmos.gov.v1beta1;

message EventSubmitProposal {
    string from_address   = 1;
    uint64 proposal_id    = 2;
    TextProposal proposal = 3;
}
```

**Bước 3**: Tái cấu trúc phát ra sự kiện để sử dụng typed event đã tạo và phát ra sử dụng `sdk.EmitTypedEvent`:

```go
// x/gov/handler.go
func handleMsgSubmitProposal(ctx sdk.Context, keeper keeper.Keeper, msg types.MsgSubmitProposalI) (*sdk.Result, error) {
    ...
    types.Context.EventManager().EmitTypedEvent(
        &EventSubmitProposal{
            FromAddress: fromAddress,
            ProposalId: id,
            Proposal: proposal,
        },
    )
    ...
}
```

### Cách Đăng Ký Typed Events trong `Client`

> LƯU Ý: Xem ví dụ code đầy đủ bên dưới

Người dùng có thể đăng ký sử dụng `client.Context.Client.Subscribe` và tiêu thụ các sự kiện được phát ra sử dụng `EventHandler`.

Akash Network đã xây dựng một [`pubsub`](https://github.com/ovrclk/akash/blob/90d258caeb933b611d575355b8df281208a214f8/pubsub/bus.go#L20) đơn giản. Điều này có thể được sử dụng để đăng ký `abci.Events` và [công bố](https://github.com/ovrclk/akash/blob/90d258caeb933b611d575355b8df281208a214f8/events/publish.go#L21) chúng dưới dạng typed events.

Vui lòng xem mẫu code bên dưới để biết thêm chi tiết về luồng này trông như thế nào với các client.

## Hậu Quả

### Tích Cực

* Cải thiện tính nhất quán của triển khai cho các sự kiện hiện tại trong Cosmos SDK
* Cung cấp một cách xử lý sự kiện thuận tiện hơn nhiều và tạo điều kiện cho việc viết các ứng dụng hướng sự kiện
* Triển khai này sẽ hỗ trợ hệ sinh thái middleware của `EventHandler`

### Tiêu Cực

## Ví Dụ Code Chi Tiết về Công Bố Sự Kiện

ADR này cũng đề xuất thêm các phương tiện để phát ra và tiêu thụ các sự kiện này. Theo cách này, các nhà phát triển chỉ cần viết `EventHandler` định nghĩa các hành động họ muốn thực hiện.

```go
// EventEmitter là kiểu mô tả các hàm phát sự kiện
// Điều này nên được định nghĩa trong `types/events.go`
type EventEmitter func(context.Context, client.Context, ...EventHandler) error

// EventHandler là kiểu hàm xử lý các sự kiện đến từ event bus
// Điều này nên được định nghĩa trong `types/events.go`
type EventHandler func(proto.Message) error

// Ví dụ sử dụng các hàm bên dưới
func main() {
    ctx, cancel := context.WithCancel(context.Background())

    if err := TxEmitter(ctx, client.Context{}.WithNodeURI("tcp://localhost:26657"), SubmitProposalEventHandler); err != nil {
        cancel()
        panic(err)
    }

    return
}

// SubmitProposalEventHandler là ví dụ về event handler in thông tin đề xuất
// khi bất kỳ EventSubmitProposal nào được phát ra.
func SubmitProposalEventHandler(ev proto.Message) (err error) {
    switch event := ev.(type) {
    // Xử lý sự kiện tạo đề xuất quản trị
    case govtypes.EventSubmitProposal:
        // Người dùng định nghĩa logic nghiệp vụ ở đây vd:
        fmt.Println(ev.FromAddress, ev.ProposalId, ev.Proposal)
        return nil
    default:
        return nil
    }
}

// TxEmitter là ví dụ về event emitter chỉ phát ra các sự kiện giao dịch. Điều này có thể và
// nên được triển khai ở đâu đó trong Cosmos SDK.
func TxEmitter(ctx context.Context, cliCtx client.Context, ehs ...EventHandler) (err error) {
    // Khởi tạo và khởi động CometBFT RPC client
    client, err := cliCtx.GetNode()
    if err != nil {
        return err
    }

    if err = client.Start(); err != nil {
        return err
    }

    // Khởi động pubsub bus
    bus := pubsub.NewBus()
    defer bus.Close()

    // Khởi tạo error group mới
    eg, ctx := errgroup.WithContext(ctx)

    // Công bố các sự kiện chain lên pubsub bus
    eg.Go(func() error {
        return PublishChainTxEvents(ctx, client, bus, simapp.ModuleBasics)
    })

    // Đăng ký các sự kiện bus
    subscriber, err := bus.Subscribe()
    if err != nil {
        return err
    }

	// Xử lý tất cả sự kiện đến từ bus
	eg.Go(func() error {
        var err error
        for {
            select {
            case <-ctx.Done():
                return nil
            case <-subscriber.Done():
                return nil
            case ev := <-subscriber.Events():
                for _, eh := range ehs {
                    if err = eh(ev); err != nil {
                        break
                    }
                }
            }
        }
        return nil
	})

	return group.Wait()
}
```

## Tham Khảo

* [Công bố Sự kiện Tùy chỉnh qua bus](https://github.com/ovrclk/akash/blob/90d258caeb933b611d575355b8df281208a214f8/events/publish.go#L19-L58)
* [Tiêu thụ sự kiện trong `Client`](https://github.com/ovrclk/deploy/blob/bf6c633ab6c68f3026df59efd9982d6ca1bf0561/cmd/event-handlers.go#L57)
