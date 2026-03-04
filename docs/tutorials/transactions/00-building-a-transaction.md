# Xây Dựng Một Giao Dịch

Đây là các bước để xây dựng, ký và broadcast giao dịch sử dụng ngữ nghĩa v2.

1. Thiết lập import đúng cách

```go
import (
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"

	apisigning "cosmossdk.io/api/cosmos/tx/signing/v1beta1"
	"cosmossdk.io/client/v2/broadcast/comet"
	"cosmossdk.io/client/v2/tx"
	"cosmossdk.io/core/transaction"
	"cosmossdk.io/math"
	banktypes "cosmossdk.io/x/bank/types"
	codectypes "github.com/cosmos/cosmos-sdk/codec/types"
	cryptocodec "github.com/cosmos/cosmos-sdk/crypto/codec"
	"github.com/cosmos/cosmos-sdk/crypto/keyring"
	authtypes "github.com/cosmos/cosmos-sdk/x/auth/types"

	"github.com/cosmos/cosmos-sdk/codec"
	addrcodec "github.com/cosmos/cosmos-sdk/codec/address"
	sdk "github.com/cosmos/cosmos-sdk/types"
)

```

2. Tạo kết nối gRPC

```go
clientConn, err := grpc.NewClient("127.0.0.1:9090", grpc.WithTransportCredentials(insecure.NewCredentials()))
if err != nil {
	log.Fatal(err)
}
```

3. Thiết lập codec và interface registry

```go
	// Thiết lập interface registry và đăng ký các interface cần thiết
	interfaceRegistry := codectypes.NewInterfaceRegistry()
	banktypes.RegisterInterfaces(interfaceRegistry)
	authtypes.RegisterInterfaces(interfaceRegistry)
	cryptocodec.RegisterInterfaces(interfaceRegistry)

	// Tạo ProtoCodec để mã hóa/giải mã
	protoCodec := codec.NewProtoCodec(interfaceRegistry)

```

4. Khởi tạo keyring

```go

	ckr, err := keyring.New("autoclikeyring", "test", home, nil, protoCodec)
	if err != nil {
		log.Fatal("error creating keyring", err)
	}
	kr, err := keyring.NewAutoCLIKeyring(ckr, addrcodec.NewBech32Codec("cosmos"))
	if err != nil {
		log.Fatal("error creating auto cli keyring", err)
	}


```

5. Thiết lập tham số giao dịch

```go

	// Thiết lập tham số giao dịch
	txParams := tx.TxParameters{
		ChainID:  "simapp-v2-chain",
		SignMode: apisigning.SignMode_SIGN_MODE_DIRECT,
		AccountConfig: tx.AccountConfig{
			FromAddress: "cosmos1t0fmn0lyp2v99ga55mm37mpnqrlnc4xcs2hhhy",
			FromName:    "alice",
		},
	}

	// Cấu hình thiết lập gas
	gasConfig, err := tx.NewGasConfig(100, 100, "0stake")
	if err != nil {
		log.Fatal("error creating gas config: ", err)
	}
	txParams.GasConfig = gasConfig

	// Tạo auth query client
	authClient := authtypes.NewQueryClient(clientConn)

	// Lấy thông tin tài khoản của người gửi
	fromAccount, err := getAccount("cosmos1t0fmn0lyp2v99ga55mm37mpnqrlnc4xcs2hhhy", authClient, protoCodec)
	if err != nil {
		log.Fatal("error getting from account: ", err)
	}

	// Cập nhật txParams với account number và sequence đúng
	txParams.AccountConfig.AccountNumber = fromAccount.GetAccountNumber()
	txParams.AccountConfig.Sequence = fromAccount.GetSequence()

	// Lấy thông tin tài khoản của người nhận
	toAccount, err := getAccount("cosmos1e2wanzh89mlwct7cs7eumxf7mrh5m3ykpsh66m", authClient, protoCodec)
	if err != nil {
		log.Fatal("error getting to account: ", err)
	}

	// Cấu hình thiết lập giao dịch
	txConf, _ := tx.NewTxConfig(tx.ConfigOptions{
		AddressCodec:          addrcodec.NewBech32Codec("cosmos"),
		Cdc:                   protoCodec,
		ValidatorAddressCodec: addrcodec.NewBech32Codec("cosmosval"),
		EnabledSignModes:      []apisigning.SignMode{apisigning.SignMode_SIGN_MODE_DIRECT},
	})
```

6. Xây dựng giao dịch

```go
// Tạo transaction factory
	f, err := tx.NewFactory(kr, codec.NewProtoCodec(codectypes.NewInterfaceRegistry()), nil, txConf, addrcodec.NewBech32Codec("cosmos"), clientConn, txParams)
	if err != nil {
		log.Fatal("error creating factory", err)
	}

	// Định nghĩa message giao dịch
	msgs := []transaction.Msg{
		&banktypes.MsgSend{
			FromAddress: fromAccount.GetAddress().String(),
			ToAddress:   toAccount.GetAddress().String(),
			Amount: sdk.Coins{
				sdk.NewCoin("stake", math.NewInt(1000000)),
			},
		},
	}

	// Xây dựng và ký giao dịch
	tx, err := f.BuildsSignedTx(context.Background(), msgs...)
	if err != nil {
		log.Fatal("error building signed tx", err)
	}


```

7. Broadcast giao dịch

```go
// Tạo broadcaster cho giao dịch
	c, err := comet.NewCometBFTBroadcaster("http://127.0.0.1:26657", comet.BroadcastSync, protoCodec)
	if err != nil {
		log.Fatal("error creating comet broadcaster", err)
	}

	// Broadcast giao dịch
	res, err := c.Broadcast(context.Background(), tx.Bytes())
	if err != nil {
		log.Fatal("error broadcasting tx", err)
	}

```

8. Các hàm hỗ trợ
    
```go
// getAccount lấy thông tin tài khoản bằng địa chỉ được cung cấp
func getAccount(address string, authClient authtypes.QueryClient, codec codec.Codec) (sdk.AccountI, error) {
	// Truy vấn thông tin tài khoản
	accountQuery, err := authClient.Account(context.Background(), &authtypes.QueryAccountRequest{
		Address: string(address),
	})
	if err != nil {
		return nil, fmt.Errorf("error getting account: %w", err)
	}

	// Giải nén thông tin tài khoản
	var account sdk.AccountI
	err = codec.InterfaceRegistry().UnpackAny(accountQuery.Account, &account)
	if err != nil {
		return nil, fmt.Errorf("error unpacking account: %w", err)
	}

	return account, nil
}
```
