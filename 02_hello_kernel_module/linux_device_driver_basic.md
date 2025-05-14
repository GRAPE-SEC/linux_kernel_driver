# 디바이스 드라이버

커널 드라이버들은 메인보드에 어떻게 연결되는지에 따라 이미 잘 만들어져있는 라이브러리나 프레임워크같은 틀이 있고, 그걸 기반으로 만들어지며,

usb 나 pci 장치로 분류되어있는 특수한 드라이버들은 미리 정의된 함수를 통해서 udev 를 거쳐 자동으로 /dev 에 디바이스 파일을 자동으로 만듦.

USB 장치 → `usb_driver`, `usb_register()`

PCI 장치 → `pci_driver`, `pci_register_driver()`

Platform 장치 → `platform_driver`

블록 장치 → `block_device_operations`

네트워크 → `net_device_ops`

이런 드라이버들은 장치가 메인보드를 통해 연결되는 즉시 커널이 드라이버의 `probe()` 를 호출하고, `udev`를 통해 `/dev` 아래에 자동으로 디바이스 파일 생성.

`probe()` 는 디바이스 드라이버 코드에 선언되어있는 함수임. 예를 들어, usb 장치의 코드에서, `.probe = myusb_probe,` 처럼 등록됨.

```c
#include <linux/module.h>
#include <linux/usb.h>
#include <linux/fs.h>
#include <linux/uaccess.h>
#include <linux/cdev.h>
#include <linux/device.h>

#define DEVICE_NAME "myusb"
#define USB_MY_VENDOR_ID  0x1234
#define USB_MY_PRODUCT_ID 0x5678

static dev_t dev_number;
static struct cdev my_cdev;
static struct class *my_class;

static int myusb_open(struct inode *inode, struct file *file) {
    pr_info("myusb: device opened\n");
    return 0;
}

// ... 중략

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = myusb_open,
    .release = myusb_release,
};

static int myusb_probe(struct usb_interface *interface, const struct usb_device_id *id) {
    pr_info("myusb: USB device plugged in (vendor=0x%x, product=0x%x)\n", id->idVendor, id->idProduct);

    // 장치 번호 할당 및 /dev/myusb0 생성
    alloc_chrdev_region(&dev_number, 0, 1, DEVICE_NAME);
    cdev_init(&my_cdev, &fops);
    cdev_add(&my_cdev, dev_number, 1);

    my_class = class_create(THIS_MODULE, DEVICE_NAME);
    device_create(my_class, NULL, dev_number, NULL, "myusb0");

    return 0;
}

// ... 중략

MODULE_DEVICE_TABLE(usb, myusb_table);

static struct usb_driver myusb_driver = {
    .name = DEVICE_NAME,
    .id_table = myusb_table,
    .probe = myusb_probe,
    .disconnect = myusb_disconnect,
};

module_usb_driver(myusb_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("You");
MODULE_DESCRIPTION("Example USB driver");

```


### 호스트 컨트롤러 드라이버
빌트인 드라이버(OS가 설치될 때 설치됨. 부팅 시 로드되며 작동 중 해제되지 않음) 이며, 클라이언트 
`ehci_hcd`(USB 2.0) `xhci_hcd`(USB 3.0) `usbhid` (HID장치 즉 마우스,키보드) 가 있음.

- 컴퓨터에 연결되는 디지털 장치들은 컴퓨터에 연결되면 자신의 `VID`와 `PID`를 컴퓨터로 전송하도록 되어있음.
- 커널에 로드되어있는 호스트 컨트롤러 드라이버는 장치가 연결되면, 장치의 `VID` 와 `PID` 를 보고 일치하는 "클라이언트 드라이버"를 찾은다음, 안에 있는 `probe()` 를 호출함.
    - `VID`는 Vendor ID 로, 이런 단체(https://www.usb.org/)에 회사를 등록해야함.
    - `PID`는 Product ID 로, 걍 장치랑 디바이스 드라이버를 만든 사람이 임의로 지음
    - Linux 커널은 `VID` 와 `PID` 조합으로 장치를 식별함.

### 클라이언트 드라이버
호스트 컨트롤러 드라이버에 의해 호출 당하면 해당 장치를 위한 작업을 수행함.
`uvcvideo` (USB 웹캠) 등


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