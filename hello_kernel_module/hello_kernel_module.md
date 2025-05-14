
# 1. 커널 헤더 설치

```sh
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r)
```

# 2. 커널 모듈 코드 작성

- 커널 모듈 : Linux 기반 시스템에서 커널에 기능을 추가할 수 있게 Linux 가 제공하는 기능. 대부분의 배포용 드라이버가 커널 모듈을 이용하여 만들어진다. 확장자는 `.ko`. 예를 들어 네트워크 카드 드라이버, USB 장치 드라이버 등이 있음

- 커널 모듈이 아닌 드라이버는 "빌트인 드라이버임". 부팅 시 자동로드되며, 부팅 이후 언로드할 수 없음. 커널 자체에 포함되어있으며, 부팅에 반드시 필요한 저장장치 드라이버(SATA, NVMe 등)이 빌트인 드라이버임.

`hello.c`
```c
#include <linux/module.h>
#include <linux/init.h>

static int __init hello_init(void) {
    printk(KERN_INFO "Hello, Kernel!\n");
    return 0;
}

static void __exit hello_exit(void) {
    printk(KERN_INFO "Goodbye, Kernel!\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("YourName");
MODULE_DESCRIPTION("A simple Hello World kernel module");
```

`Makefile`
```make
obj-m += hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
# 3. 빌드 및 테스트

1) 컴파일
```sh
make
```

빌드가 정상적으로 완료되면 `hello.ko` 가 생성된다.


2) 커널 모듈 삽입
커널 모듈을 커널에 삽입하려면 root 권한이 필요함.
```sh
sudo insmod hello.ko
sudo dmesg | tail
```

3) 커널 모듈 제거
```sh
sudo rmmod hello
sudo dmesg | tail
```
