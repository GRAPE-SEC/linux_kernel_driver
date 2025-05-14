# 디바이스 드라이버의 배포와 개발 (웹캠 제조사 개발자 입장)

1. USB 웹캠을 제조한다.

→ USB 장치에는 **Vendor ID (VID)**와 **Product ID (PID)**가 있음
→ 예: 0x1234, 0x5678

2. 리눅스용 드라이버를 개발 (커널 모듈 형태가 일반적)

- UVC(USB Video Class)를 지원하면 기본 드라이버로 작동할 수 있음 (uvcvideo)
- 아니면, 자체 커널 드라이버를 개발해서 등록

```c
// ... 중략
usb_register(&mywebcam_driver);
//
```

3. 드라이버가 커널에 장치를 등록하면

- 커널이 **장치 등록 이벤트**를 발생시킴
- udev가 이 이벤트를 잡아서 /dev/videoX 같은 디바이스 파일 자동 생성



# 리눅스 커널 드라이버의 종류

### Character Device Driver (char driver)
- 1바이트 단위로 데이터 처리 (stream 형식)
- 순차적이고 연속적인 입출력에 적합
- 예: 시리얼 포트, 터미널, 센서, 가상 디바이스, /dev/null, /dev/tty, /dev/random
- 사용자 공간과의 인터페이스: read, write, open, close

### Block Device Driver (block driver)
- **블록 단위(예: 512B, 4KB)**로 읽고 쓰는 장치
- 임의 접근(random access)이 가능
- 예: HDD, SSD, USB 저장소, /dev/sda, /dev/nvme0n1, /dev/loop0
- 보통 make_request_fn, bio 구조체 기반

### Network Device Driver (net driver)
- 네트워크 패킷을 전송/수신하는 장치용
- 예: 유선/무선 NIC (eth0, wlan0), 가상 인터페이스 (tun/tap), USB Ethernet
- net_device 구조체 등록, ndo_start_xmit, ndo_open, ndo_stop 등 NetDeviceOps 콜백

### Platform Driver
- 보통 SoC(System on Chip) 기반 시스템에서 쓰이는 내장 하드웨어 장치 드라이버
- 디바이스 트리(Device Tree) 또는 ACPI 기반으로 매핑
- 예: I2C/SPI 센서, GPIO, 내장 UART
- platform_driver와 platform_device 구조체 기반

    Embedded/ARM 환경에서 매우 많이 쓰임

### USB Driver
- USB 장치를 제어하는 드라이버
- 예: USB 카메라, USB 무선랜카드, USB MIDI, USB Mass Storage
- usb_register() 기반 등록 -> probe, disconnect 콜백 함수로 장치 이벤트 대응

### PCI Driver
- PCI/PCIe 버스를 통해 장치를 제어
- 예: 그래픽 카드, NVMe SSD, 이더넷 카드, 사운드 카드 등
- pci_driver 구조체 등록 -> probe(), remove() 콜백 사용
- 장치 ID (Vendor ID, Device ID) 매칭 필요

### Virtual Driver (가상 디바이스 드라이버)
- 실제 하드웨어 없이 소프트웨어적으로 구현된 디바이스
- 예: /dev/zero, /dev/null, loopback NIC, fuse 기반 파일 시스템
- 디바이스 드라이버 원리를 테스트하거나 기능을 에뮬레이션할 때 사용

### File System Driver
- 파일 시스템을 해석/관리하는 드라이버
- 예: ext4, xfs, vfat, ntfs3, btrfs

### Input Driver
- 키보드, 마우스, 터치스크린 등 입력 장치용 드라이버
- 예: evdev, atkbd, synaptics, touchscreen
- input_event, input_register_device 등 사용 -> /dev/input/event*로 연결됨

### Display / GPU Driver
- 디스플레이/그래픽 카드와 관련된 장치 제어
- 예: Intel DRM, AMDGPU, NVIDIA Nouveau, framebuffer
- drm, fbdev 서브시스템 사용 -> KMS (Kernel Mode Setting)와 연동