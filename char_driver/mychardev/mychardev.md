# 1. char 디바이스 드라이버 기본 예제 만들기

`mychardev.c`
```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/uaccess.h>

#define DEVICE_NAME "mychardev"

static int major;
static char buffer[256] = {0};
static int dev_open(struct inode *inode, struct file *file) {
    printk(KERN_INFO "mychardev: device opened\n");
    return 0;
}

static int dev_release(struct inode *inode, struct file *file) {
    printk(KERN_INFO "mychardev: device closed\n");
    return 0;
}

static ssize_t dev_read(struct file *filp, char __user *user_buf, size_t len, loff_t *offset) {
    return simple_read_from_buffer(user_buf, len, offset, buffer, strlen(buffer));
}

static ssize_t dev_write(struct file *filp, const char __user *user_buf, size_t len, loff_t *offset) {
    return simple_write_to_buffer(buffer, sizeof(buffer), offset, user_buf, len);
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = dev_open,
    .release = dev_release,
    .read = dev_read,
    .write = dev_write,
};

static int __init mychardev_init(void) {
    major = register_chrdev(0, DEVICE_NAME, &fops);
    if (major < 0) {
        printk(KERN_ALERT "mychardev: failed to register device\n");
        return major;
    }
    printk(KERN_INFO "mychardev: registered with major number %d\n", major);
    return 0;
}

static void __exit mychardev_exit(void) {
    unregister_chrdev(major, DEVICE_NAME);
    printk(KERN_INFO "mychardev: unregistered\n");
}

module_init(mychardev_init);
module_exit(mychardev_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("you");
MODULE_DESCRIPTION("Simple char device driver");

```

# 2. Makefile 작성

`Makefile`
```make
obj-m += mychardev.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

# 3. 빌드 및 로드

```sh
make
sudo insmod mychardev.ko
dmesg | tail
```

`[ 4156.348503] mychardev: registered with major number 238` 가 출력되야 정상
- major number 는 시스템마다 다르게 출력됨

# 4. 디바이스 파일 생성

### 디바이스 파일?
- Linux 에서는 하드웨어 장치(마우스,키보드,모니터,웹캠 등)을 파일로 만들어서 시스템에 연결함.
- 이러한 특수한 파일을 "디바이스 파일" 이라고 부르며, 실제로 파일은 아니고 일종의 가상의 파일임.
- 하드웨어 장치를 사용하고 싶은 프로그램은 `read`, `write` 등을 이 파일에 대해 호출해야하며, "디바이스 파일"에 대한 `read`, `write` 호출은 파일에 쓰기를 하는 것이 아니라, 연결된 커널모듈안에 등록된 `read`, `write` 함수를 호출함.


### 디바이스 파일 생성
- 디바이스 파일 `mychardevice`를 생성한다.

`238` 대신 본인의 시스템에 표시된 major number 를 넣는다.
```sh
sudo mknod /dev/mychardev c 238 0
sudo chmod 666 /dev/mychardev
```

- `/dev` 디렉토리에 생성한 `/dev/mychardev` 가 생김.
- 디바이스 파일은 일반 파일과 달리 `ls -l` 로 확인했을때, 맨 앞에 `c`나 `b` 가 붙는다.
- Linux 시스템에 새로운 장치(usb,마우스 등)을 연결하면 Linux 의 `udev` 데몬이 파일쓰기로 `/dev`에 디바이스 파일을 자동으로 생성한다.
```sh
ls -l /dev
```


# 5. 테스트
이제 이 `/dev/mychardev` 와 커널 드라이버 `mychardev.ko` 가 커널레벨에서 연결된 상태임.

- `/dev/mychardev` 파일에 쓰기를 하면, `mychardev.ko`가 처리한다.
```sh
echo "hello~ kernel driver~" > /dev/mychardev
cat /dev/mychardev
```
`hello~ kernel driver~` 가 표시된다.