# TỔNG QUAN DỰ ÁN BỘ TĂNG TỐC PHẦN CỨNG FORS
## (Forest of Random Subsets Hardware Accelerator Project Overview)

---

## 1. MỤC TIÊU DỰ ÁN

Dự án này tập trung nghiên cứu, thiết kế và hiện thực hóa **Bộ tăng tốc phần cứng (Hardware Accelerator IP)** và **Hệ thống trên vi mạch (System-on-Chip - SoC)** cho thuật toán **FORS (Forest of Random Subsets)**, tuân thủ chuẩn mật mã hậu lượng tử của Viện Tiêu chuẩn và Kỹ thuật Quốc gia Hoa Kỳ: **NIST FIPS 205 (SLH-DSA / SPHINCS+)**.

* **Mục tiêu thuật toán:** Thực thi hai chức năng mật mã cốt lõi:
  1. `FORS_SIGN` (Tạo chữ ký vài lần - Few-Time Signature).
  2. `FORS_PK_FROM_SIG` (Xác thực chữ ký và tái tạo Khóa công khai ứng viên).
* **Bộ tham số trọng tâm:** 
  - **SLH-DSA-SHAKE-128f** (Bộ tham số cơ sở, tối ưu cho tốc độ ký nhanh).
  - Khả năng mở rộng tham số lên **SLH-DSA-SHAKE-256f** mà không cần thay đổi kiến trúc phần cứng.

---

## 2. BÀI TOÁN CẦN GIẢI QUYẾT

Trong lược đồ chữ ký số SPHINCS+, **FORS là nút thắt cổ chai tính toán lớn nhất**:
- FORS trực tiếp ký lên bản băm của thông điệp bằng cách duyệt một "khu rừng" gồm **$k = 33$ cây nhị phân độc lập**, mỗi cây có độ cao $a = 6$ ($2^6 = 64$ lá) $\implies$ Tổng cộng **2.112 lá**.
- Nếu chạy thuần túy bằng phần mềm trên CPU/vi điều khiển, quá trình này tiêu tốn **hơn 6.300 phép băm Keccak-f[1600] / SHAKE256** (chiếm tới 70% – 85% tổng thời gian tạo chữ ký).
- **Giải pháp của dự án:** Xây dựng phần cứng chuyên dụng chạy song song và tự động hóa toàn bộ việc sinh khóa, băm lá, leo cây Merkle và nén gốc, giúp giảm thời gian ký từ hàng trăm mili-giây xuống **dưới 1 mili-giây**.

---

## 3. HAI CẤP ĐỘ THIẾT KẾ CỦA DỰ ÁN

Dự án được phân cấp rõ ràng thành 2 tầng kiến trúc độc lập:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        TẦNG 2: FORS SoC PLATFORM                       │
│                                                                        │
│   ┌──────────────┐     Bus MMIO      ┌─────────────────────────────┐   │
│   │   Host CPU   │◄─────────────────►│ Register-Driven DMA Engine  │   │
│   └──────┬───────┘                   └──────────────┬──────────────┘   │
│          │                                          │                  │
│          │ Port A (32b)                             │ Port B (64b)     │
│          ▼                                          ▼                  │
│   ┌────────────────────────────────────────────────────────────┐       │
│   │             Shared True Dual-Port BRAM (8 - 16 KiB)        │       │
│   └────────────────────────────────────────────────────────────┘       │
│                                                     │                  │
│                                       AXI4-Stream   │ din / dout       │
│                                       (Zero-Copy)   ▼                  │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                TẦNG 1: FORS ACCELERATOR CORE (IP)              │   │
│   │                                                                │   │
│   │   [fors_controller] ◄──► [fors_memory (Stack Zero-BRAM)]       │   │
│   │          │                                                     │   │
│   │          ▼                                                     │   │
│   │   [hash_adapter_fors] ◄──► [hash_engine (Keccak-f[1600])]      │   │
│   └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### Cấp độ 1: Lõi tăng tốc FORS độc lập (Standalone FORS IP Core)
- **Kiến trúc 4 khối:** `fors_controller` (bộ não FSM), `fors_memory` (kho chứa thanh ghi), `hash_adapter_fors` (bộ đệm tiền tố 48B & padding), và `hash_engine` (lõi băm Keccak).
- **Tối ưu tài nguyên (Zero-BRAM):** Toàn bộ quá trình leo cây Merkle sử dụng ngăn xếp thanh ghi tại chỗ ($a$ tầng), hoàn toàn **không tốn bất kỳ khối Block RAM nào**.
- **Tái sử dụng lõi băm:** Sử dụng **duy nhất 1 lõi Keccak** dùng chung cho mọi thao tác băm lá, băm nút và nén gốc $T_k$.
- **Giao diện chuẩn hóa:** Giao tiếp qua kênh luồng AXI4-Stream 64-bit với cơ chế bắt tay `valid`/`ready` và cờ phân tách gói `last`.

### Cấp độ 2: Hệ thống trên Vi mạch (FORS SoC Platform)
- **Giải phóng CPU:** CPU chỉ cần ghi Context vào BRAM, cấu hình thanh ghi DMA và bấm `START`. Toàn bộ quá trình vận chuyển dữ liệu 64-bit do DMA tự động bơm vào Core và hứng kết quả về BRAM.
- **Bộ nhớ chia sẻ (True Dual-Port BRAM):** Cho phép CPU chuẩn bị dữ liệu (Port A) cùng lúc với việc DMA đang truyền dữ liệu cho Core (Port B) mà không gây xung đột bus.
- **DMA hướng thanh ghi (Register-Driven):** Cắt giảm 42% diện tích so với mô hình descriptor phức tạp, cực kỳ tinh gọn cho FPGA nhúng.

---

## 4. NỀN TẢNG PHẦN CỨNG MỤC TIÊU (TARGET FPGAs)

Mã nguồn RTL được viết bằng **SystemVerilog chuẩn, trung lập với nhà sản xuất (Vendor-Neutral)**, sẵn sàng tổng hợp trên 2 dòng kit FPGA phổ biến:
1. **AMD / Xilinx Artix-7 (XC7A100T):** Mục tiêu tần số $F_{max} \ge 150 - 200\text{ MHz}$, thời gian ký ước tính $\approx 0.76\text{ ms}$.
2. **Intel / Altera Cyclone IV E (EP4CE115 trên kit DE2-115):** Mục tiêu tần số $F_{max} \ge 100\text{ MHz}$, thời gian ký ước tính $\approx 1.26\text{ ms}$.

---

## 5. HỆ THỐNG TÀI LIỆU DỰ ÁN (DOCUMENTATION MAP)

Để theo dõi chi tiết kỹ thuật của từng phần, dự án cung cấp 4 tài liệu chuyên sâu:

| File Tài Liệu | Ngôn Ngữ | Đối Tượng & Nội Dung Chính |
| :--- | :---: | :--- |
| [`README_FORS_ACCELERATOR_SPEC.md`]| **Tiếng Anh** | **Bản đặc tả kỹ thuật chính thức của Core FORS:** Cấu trúc 4 khối, giao thức AXI4-Stream, ADRS generation, tính toán Work counts chuẩn WOTS, quy tắc bảo mật và ma trận nghiệm thu. |
| [`README_FORS_SOC_SPEC.md`]| **Tiếng Anh** | **Bản đặc tả kỹ thuật tích hợp SoC:** Cấu trúc CPU, True Dual-Port BRAM, Register-Driven DMA, bản đồ thanh ghi MMIO CSR và luồng hoạt động 7 bước. |
