# ADR 013: Quan Sát Hệ Thống

## Changelog

* 20-01-2020: Bản nháp đầu tiên

## Trạng Thái

Đề Xuất

## Bối Cảnh

Telemetry là yếu tố tối quan trọng trong việc debug và hiểu ứng dụng đang làm gì và hoạt động như thế nào. Chúng ta nhằm mục tiêu phát lộ các metric từ các module và các phần core khác của Cosmos SDK.

Ngoài ra, chúng ta nên nhắm đến việc hỗ trợ nhiều sink có thể cấu hình mà operator có thể chọn. Theo mặc định, khi telemetry được bật, ứng dụng nên theo dõi và phát lộ các metric được lưu trữ trong bộ nhớ. Operator có thể chọn bật thêm sink, nơi chúng ta chỉ hỗ trợ [Prometheus](https://prometheus.io/) hiện tại, vì nó đã được kiểm chứng thực tế, đơn giản để thiết lập, mã nguồn mở và có hệ sinh thái công cụ phong phú.

Chúng ta cũng phải nhằm mục tiêu tích hợp metric vào Cosmos SDK theo cách liền mạch nhất có thể để các metric có thể được thêm hoặc xóa theo ý muốn mà không gặp nhiều khó khăn. Để làm điều này, chúng ta sẽ sử dụng thư viện [go-metrics](https://github.com/hashicorp/go-metrics).

Cuối cùng, operator có thể bật telemetry cùng với các tùy chọn cấu hình cụ thể. Nếu được bật, metric sẽ được phát lộ qua `/metrics?format={text|prometheus}` qua API server.

## Quyết Định

Chúng ta sẽ thêm một block cấu hình bổ sung vào `app.toml` định nghĩa các cài đặt telemetry:

```toml
###############################################################################
###                         Cấu Hình Telemetry                              ###
###############################################################################

[telemetry]

# Có tiền tố với các khóa để phân tách dịch vụ
service-name = {{ .Telemetry.ServiceName }}

# Enabled bật chức năng telemetry của ứng dụng. Khi được bật,
# một in-memory sink cũng được bật theo mặc định. Operator cũng có thể bật
# các sink khác như Prometheus.
enabled = {{ .Telemetry.Enabled }}

# Bật tiền tố giá trị gauge với hostname
enable-hostname = {{ .Telemetry.EnableHostname }}

# Bật thêm hostname vào label
enable-hostname-label = {{ .Telemetry.EnableHostnameLabel }}

# Bật thêm service vào label
enable-service-label = {{ .Telemetry.EnableServiceLabel }}

# PrometheusRetentionTime, khi dương, bật một Prometheus metrics sink.
prometheus-retention-time = {{ .Telemetry.PrometheusRetentionTime }}
```

Cấu hình đã cho cho phép hai sink -- in-memory và Prometheus. Chúng ta tạo một kiểu `Metrics` thực hiện tất cả bootstrapping cho operator, vì vậy việc thu thập metric trở nên liền mạch.

```go
// Metrics định nghĩa một wrapper quanh chức năng telemetry của ứng dụng. Nó cho phép
// metric được thu thập vào bất kỳ thời điểm nào. Khi tạo đối tượng Metrics,
// nội bộ, một global metrics được đăng ký với một tập hợp sink như được cấu hình
// bởi operator. Ngoài ra các sink, khi một process nhận SIGUSR1, một
// dump các metric gần đây được định dạng sẽ được gửi đến STDERR.
type Metrics struct {
  memSink           *metrics.InmemSink
  prometheusEnabled bool
}

// Gather thu thập tất cả metric đã đăng ký và trả về GatherResponse trong đó
// các metric được mã hóa tùy thuộc vào kiểu. Metric được mã hóa qua
// Prometheus hoặc JSON nếu in-memory.
func (m *Metrics) Gather(format string) (GatherResponse, error) {
  switch format {
  case FormatPrometheus:
    return m.gatherPrometheus()

  case FormatText:
    return m.gatherGeneric()

  case FormatDefault:
    return m.gatherGeneric()

  default:
    return GatherResponse{}, fmt.Errorf("unsupported metrics format: %s", format)
  }
}
```

Ngoài ra, `Metrics` cho phép chúng ta thu thập tập hợp metric hiện tại vào bất kỳ thời điểm nào. Operator cũng có thể chọn gửi tín hiệu, SIGUSR1, để dump và in các metric được định dạng ra STDERR.

Trong giai đoạn bootstrapping và xây dựng của ứng dụng, nếu `Telemetry.Enabled` là `true`, API server sẽ tạo một instance tham chiếu đến đối tượng `Metrics` và sẽ đăng ký một metrics handler tương ứng.

```go
func (s *Server) Start(cfg config.Config) error {
  // ...

  if cfg.Telemetry.Enabled {
    m, err := telemetry.New(cfg.Telemetry)
    if err != nil {
      return err
    }

    s.metrics = m
    s.registerMetrics()
  }

  // ...
}

func (s *Server) registerMetrics() {
  metricsHandler := func(w http.ResponseWriter, r *http.Request) {
    format := strings.TrimSpace(r.FormValue("format"))

    gr, err := s.metrics.Gather(format)
    if err != nil {
      rest.WriteErrorResponse(w, http.StatusBadRequest, fmt.Sprintf("failed to gather metrics: %s", err))
      return
    }

    w.Header().Set("Content-Type", gr.ContentType)
    _, _ = w.Write(gr.Metrics)
  }

  s.Router.HandleFunc("/metrics", metricsHandler).Methods("GET")
}
```

Nhà phát triển ứng dụng có thể theo dõi counter, gauge, summary và metric key/value. Không cần thêm công sức từ module để tận dụng metric profiling. Để làm vậy, đơn giản như:

```go
func (k BaseKeeper) MintCoins(ctx sdk.Context, moduleName string, amt sdk.Coins) error {
  defer metrics.MeasureSince(time.Now(), "MintCoins")
  // ...
}
```

## Hậu Quả

### Tích Cực

* Hiểu rõ về hiệu suất và hành vi của ứng dụng

### Tiêu Cực

### Trung Lập

## Tài Liệu Tham Khảo
