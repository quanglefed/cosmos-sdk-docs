# ADR 3: Dynamic Capability Store

## Changelog

* 12 tháng 12 năm 2019: Phiên bản đầu tiên
* 02 tháng 4 năm 2020: Sửa đổi Memory Store

## Bối Cảnh

Triển khai đầy đủ [đặc tả IBC](https://github.com/cosmos/ibc) yêu cầu khả năng tạo và xác thực các object-capability key tại runtime (tức là trong quá trình thực thi giao dịch), như được mô tả trong [ICS 5](https://github.com/cosmos/ibc/tree/master/spec/core/ics-005-port-allocation#technical-specification). Trong đặc tả IBC, các capability key được tạo cho mỗi port & channel được khởi tạo mới, và được dùng để xác thực việc sử dụng port hoặc channel trong tương lai. Vì các channel và có thể cả port có thể được khởi tạo trong quá trình thực thi giao dịch, state machine phải có khả năng tạo các object-capability key vào thời điểm này.

Hiện tại, Cosmos SDK không có khả năng này. Các object-capability key hiện là các con trỏ (địa chỉ bộ nhớ) của các struct `StoreKey` được tạo khi khởi tạo ứng dụng trong `app.go` ([ví dụ](https://github.com/cosmos/gaia/blob/dcbddd9f04b3086c0ad07ee65de16e7adedc7da4/app/app.go#L132)) và được truyền cho các Keeper làm đối số cố định ([ví dụ](https://github.com/cosmos/gaia/blob/dcbddd9f04b3086c0ad07ee65de16e7adedc7da4/app/app.go#L160)). Keeper không thể tạo hoặc lưu trữ capability key trong quá trình thực thi giao dịch — mặc dù chúng có thể gọi `NewKVStoreKey` và lấy địa chỉ bộ nhớ của struct được trả về, việc lưu trữ nó vào Merklised store sẽ dẫn đến consensus fault, vì địa chỉ bộ nhớ sẽ khác nhau trên mỗi máy (điều này là có chủ ý — nếu không phải như vậy, các khóa sẽ có thể đoán trước và không thể đóng vai trò là object capability).

Keeper cần một cách để giữ một map riêng tư của các store key có thể được thay đổi trong quá trình thực thi giao dịch, cùng với cơ chế phù hợp để tái tạo các địa chỉ bộ nhớ duy nhất (capability key) trong map này bất cứ khi nào ứng dụng được khởi động hoặc khởi động lại, cùng với cơ chế để hoàn tác việc tạo capability khi tx thất bại. ADR này đề xuất interface và cơ chế như vậy.

## Quyết Định

Cosmos SDK sẽ bao gồm một abstraction `CapabilityKeeper` mới, chịu trách nhiệm cung cấp, theo dõi và xác thực các capability tại runtime. Trong quá trình khởi tạo ứng dụng trong `app.go`, `CapabilityKeeper` sẽ được kết nối với các module thông qua các tham chiếu hàm duy nhất (bằng cách gọi `ScopeToModule`, được định nghĩa bên dưới) để nó có thể xác định module gọi khi được gọi sau này.

Khi trạng thái ban đầu được tải từ đĩa, hàm `Initialise` của `CapabilityKeeper` sẽ tạo các capability key mới cho tất cả các capability identifier đã được cấp phát trước đó (được cấp phát trong quá trình thực thi các giao dịch trước đó và được gán cho các mode cụ thể), và giữ chúng trong một memory-only store trong khi chain đang chạy.

`CapabilityKeeper` sẽ bao gồm một `KVStore` persistent, một `MemoryStore`, và một map trong bộ nhớ. `KVStore` persistent theo dõi capability nào thuộc sở hữu của module nào. `MemoryStore` lưu trữ forward mapping ánh xạ từ tên module, tuple capability đến tên capability và reverse mapping ánh xạ từ tên module, tên capability đến chỉ số capability. Vì chúng ta không thể marshal capability vào `KVStore` và unmarshal mà không thay đổi vị trí bộ nhớ của capability, reverse mapping trong KVStore sẽ chỉ ánh xạ đến một chỉ số. Chỉ số này sau đó có thể được dùng như một khóa trong go-map tạm thời để lấy capability tại vị trí bộ nhớ gốc.

`CapabilityKeeper` sẽ định nghĩa các kiểu và hàm sau:

`Capability` tương tự như `StoreKey`, nhưng có một `Index()` duy nhất toàn cục thay vì tên. Một phương thức `String()` được cung cấp để debug.

Một `Capability` đơn giản là một struct, địa chỉ của nó được lấy làm capability thực sự.

```go
type Capability struct {
  index uint64
}
```

Một `CapabilityKeeper` chứa một store key persistent, memory store key, và mapping của các tên module đã được cấp phát.

```go
type CapabilityKeeper struct {
  persistentKey StoreKey
  memKey        StoreKey
  capMap        map[uint64]*Capability
  moduleNames   map[string]interface{}
  sealed        bool
}
```

`CapabilityKeeper` cung cấp khả năng tạo các sub-keeper *có phạm vi* được gắn với một tên module cụ thể. Các `ScopedCapabilityKeeper` này phải được tạo khi khởi tạo ứng dụng và được truyền cho các module, các module này sau đó có thể dùng chúng để claim các capability họ nhận được và lấy các capability mà họ sở hữu theo tên, ngoài việc tạo capability mới và xác thực capability được truyền bởi các module khác.

```go
type ScopedCapabilityKeeper struct {
  persistentKey StoreKey
  memKey        StoreKey
  capMap        map[uint64]*Capability
  moduleName    string
}
```

`ScopeToModule` được dùng để tạo một scoped sub-keeper với một tên cụ thể, phải là duy nhất. Nó PHẢI được gọi trước `InitialiseAndSeal`.

```go
func (ck CapabilityKeeper) ScopeToModule(moduleName string) ScopedCapabilityKeeper {
	if k.sealed {
		panic("cannot scope to module via a sealed capability keeper")
	}

	if _, ok := k.scopedModules[moduleName]; ok {
		panic(fmt.Sprintf("cannot create multiple scoped keepers for the same module name: %s", moduleName))
	}

	k.scopedModules[moduleName] = struct{}{}

	return ScopedKeeper{
		cdc:      k.cdc,
		storeKey: k.storeKey,
		memKey:   k.memKey,
		capMap:   k.capMap,
		module:   moduleName,
	}
}
```

`InitialiseAndSeal` PHẢI được gọi đúng một lần, sau khi tải trạng thái ban đầu và tạo tất cả các `ScopedCapabilityKeeper` cần thiết, để điền vào memory store với các capability key mới được tạo phù hợp với các khóa đã được claim trước đó bởi các module cụ thể và ngăn việc tạo bất kỳ `ScopedCapabilityKeeper` mới nào.

```go
func (ck CapabilityKeeper) InitialiseAndSeal(ctx Context) {
  if ck.sealed {
    panic("capability keeper is sealed")
  }

  persistentStore := ctx.KVStore(ck.persistentKey)
  map := ctx.KVStore(ck.memKey)
  
  // khởi tạo memory store cho tất cả tên trong persistent store
  for index, value := range persistentStore.Iter() {
    capability = &CapabilityKey{index: index}

    for moduleAndCapability := range value {
      moduleName, capabilityName := moduleAndCapability.Split("/")
      memStore.Set(moduleName + "/fwd/" + capability, capabilityName)
      memStore.Set(moduleName + "/rev/" + capabilityName, index)

      ck.capMap[index] = capability
    }
  }

  ck.sealed = true
}
```

`NewCapability` có thể được gọi bởi bất kỳ module nào để tạo một tham chiếu object-capability mới, duy nhất và không thể giả mạo. Capability mới được tạo tự động được lưu trữ; module gọi không cần gọi `ClaimCapability`.

```go
func (sck ScopedCapabilityKeeper) NewCapability(ctx Context, name string) (Capability, error) {
  // kiểm tra tên chưa được dùng trong memory store
  if capStore.Get("rev/" + name) != nil {
    return nil, errors.New("name already taken")
  }

  // lấy chỉ số hiện tại
  index := persistentStore.Get("index")
  
  // tạo capability mới
  capability := &CapabilityKey{index: index}
  
  // đặt persistent store
  persistentStore.Set(index, Set.singleton(sck.moduleName + "/" + name))
  
  // cập nhật chỉ số
  index++
  persistentStore.Set("index", index)
  
  // đặt forward mapping trong memory store từ capability đến tên
  memStore.Set(sck.moduleName + "/fwd/" + capability, name)
  
  // đặt reverse mapping trong memory store từ tên đến chỉ số
  memStore.Set(sck.moduleName + "/rev/" + name, index)

  // đặt mapping trong bộ nhớ từ chỉ số đến con trỏ capability
  capMap[index] = capability
  
  // trả về capability mới được tạo
  return capability
}
```

`AuthenticateCapability` có thể được gọi bởi bất kỳ module nào để kiểm tra xem một capability có thực sự tương ứng với một tên cụ thể hay không (tên có thể là đầu vào người dùng không đáng tin cậy) mà module gọi đã liên kết trước đó.

```go
func (sck ScopedCapabilityKeeper) AuthenticateCapability(name string, capability Capability) bool {
  // trả về liệu forward mapping trong memory store có khớp với tên không
  return memStore.Get(sck.moduleName + "/fwd/" + capability) === name
}
```

`ClaimCapability` cho phép một module claim một capability key mà nó đã nhận từ module khác để các lần gọi `GetCapability` trong tương lai sẽ thành công.

`ClaimCapability` PHẢI được gọi nếu một module nhận được capability muốn truy cập nó theo tên trong tương lai. Capability là đa chủ sở hữu, vì vậy nếu nhiều module có một tham chiếu `Capability` duy nhất, tất cả họ đều sở hữu nó.

```go
func (sck ScopedCapabilityKeeper) ClaimCapability(ctx Context, capability Capability, name string) error {
  persistentStore := ctx.KVStore(sck.persistentKey)

  // đặt forward mapping trong memory store từ capability đến tên
  memStore.Set(sck.moduleName + "/fwd/" + capability, name)

  // đặt reverse mapping trong memory store từ tên đến capability
  memStore.Set(sck.moduleName + "/rev/" + name, capability)

  // cập nhật owner set trong persistent store
  owners := persistentStore.Get(capability.Index())
  owners.add(sck.moduleName + "/" + name)
  persistentStore.Set(capability.Index(), owners)
}
```

`GetCapability` cho phép một module lấy một capability mà nó đã claim trước đó theo tên. Module không được phép lấy các capability mà nó không sở hữu.

```go
func (sck ScopedCapabilityKeeper) GetCapability(ctx Context, name string) (Capability, error) {
  // lấy chỉ số của capability bằng reverse mapping trong memstore
  index := memStore.Get(sck.moduleName + "/rev/" + name)

  // lấy capability từ go-map bằng chỉ số
  capability := capMap[index]

  // trả về capability
  return capability
}
```

`ReleaseCapability` cho phép một module giải phóng một capability mà nó đã claim trước đó. Nếu không còn chủ sở hữu nào, capability sẽ bị xóa toàn cục.

```go
func (sck ScopedCapabilityKeeper) ReleaseCapability(ctx Context, capability Capability) err {
  persistentStore := ctx.KVStore(sck.persistentKey)

  name := capStore.Get(sck.moduleName + "/fwd/" + capability)
  if name == nil {
    return error("capability not owned by module")
  }

  // xóa forward mapping trong memory store
  memoryStore.Delete(sck.moduleName + "/fwd/" + capability, name)

  // xóa reverse mapping trong memory store
  memoryStore.Delete(sck.moduleName + "/rev/" + name, capability)

  // cập nhật owner set trong persistent store
  owners := persistentStore.Get(capability.Index())
  owners.remove(sck.moduleName + "/" + name)
  if owners.size() > 0 {
    // vẫn còn chủ sở hữu khác, giữ capability
    persistentStore.Set(capability.Index(), owners)
  } else {
    // không còn chủ sở hữu, xóa capability
    persistentStore.Delete(capability.Index())
    delete(capMap[capability.Index()])
  }
}
```

### Mẫu Sử Dụng

#### Khởi Tạo

Bất kỳ module nào sử dụng dynamic capability đều phải được cung cấp một `ScopedCapabilityKeeper` trong `app.go`:

```go
ck := NewCapabilityKeeper(persistentKey, memoryKey)
mod1Keeper := NewMod1Keeper(ck.ScopeToModule("mod1"), ....)
mod2Keeper := NewMod2Keeper(ck.ScopeToModule("mod2"), ....)

// logic khởi tạo khác ...

// tải trạng thái ban đầu...

ck.InitialiseAndSeal(initialContext)
```

#### Tạo, Truyền, Claim và Sử Dụng Capability

Xét trường hợp `mod1` muốn tạo một capability, liên kết nó với một tài nguyên (ví dụ: một kênh IBC) theo tên, rồi truyền nó cho `mod2` sẽ sử dụng sau này:

Module 1 sẽ có code sau:

```go
capability := scopedCapabilityKeeper.NewCapability(ctx, "resourceABC")
mod2Keeper.SomeFunction(ctx, capability, args...)
```

`SomeFunction`, chạy trong module 2, sau đó có thể claim capability:

```go
func (k Mod2Keeper) SomeFunction(ctx Context, capability Capability) {
  k.sck.ClaimCapability(ctx, capability, "resourceABC")
  // logic khác...
}
```

Sau đó, module 2 có thể lấy capability đó theo tên và truyền nó cho module 1, module 1 sẽ xác thực nó với tài nguyên:

```go
func (k Mod2Keeper) SomeOtherFunction(ctx Context, name string) {
  capability := k.sck.GetCapability(ctx, name)
  mod1.UseResource(ctx, capability, "resourceABC")
}
```

Module 1 sau đó sẽ kiểm tra rằng capability key này đã được xác thực để sử dụng tài nguyên trước khi cho phép module 2 sử dụng nó:

```go
func (k Mod1Keeper) UseResource(ctx Context, capability Capability, resource string) {
  if !k.sck.AuthenticateCapability(name, capability) {
    return errors.New("unauthenticated")
  }
  // làm gì đó với tài nguyên
}
```

Nếu module 2 truyền capability key cho module 3, module 3 có thể claim nó và gọi module 1 giống như module 2 đã làm (trong trường hợp đó module 1, module 2 và module 3 đều có thể sử dụng capability này).

## Trạng Thái

Đề Xuất.

## Hậu Quả

### Tích Cực

* Hỗ trợ dynamic capability.
* Cho phép CapabilityKeeper trả về cùng con trỏ capability từ go-map trong khi hoàn tác các ghi vào `KVStore` persistent và `MemoryStore` trong bộ nhớ khi tx thất bại.

### Tiêu Cực

* Yêu cầu một keeper bổ sung.
* Có một số chồng lấp với hệ thống `StoreKey` hiện tại (trong tương lai chúng có thể được kết hợp, vì đây là chức năng superset).
* Yêu cầu thêm một cấp độ indirection trong reverse mapping, vì MemoryStore phải ánh xạ đến chỉ số, chỉ số này sau đó phải được dùng làm khóa trong go-map để lấy capability thực sự.

### Trung Lập

(không có gì đã biết)

## Tài Liệu Tham Khảo

* [Thảo luận gốc](https://github.com/cosmos/cosmos-sdk/pull/5230#discussion_r343978513)
