# Polling setup



[reference](https://events.static.linuxfound.org/sites/events/files/slides/lemoal-nvme-polling-vault-2017-final_0.pdf)


## Setting 
sysfs 에서 [queue](https://www.kernel.org/doc/html/latest/block/queue-sysfs.html) 파일에 있는 io_poll 을 통해 enable 할 수 있다. 

```bash
$ echo 1 > /sys/block/nvme0n1/io_poll
```

### Error handling 
`invalid argument` error이 발생한 경우, 커널 파라미터인 `nvme.poll_queues` 룰 0이 아닌 수로 세팅해주어야 한다.

```bash
$ echo 1 > /sys/module/nvme/parameters/poll_queues
```

하지만 이를 세팅하여도 io_poll 똑같은 에러가 발생한다.

### Solve
Kernel nvme driver 에서 Nvme device 를 unbind 한 후 다시 bind 를 해주고 `io_poll`을 수정하니 에러가 없어졌다.

## Test
사용 SSD: Optane SSD
사용 워크로드: FIO (hipri option 을 통해 polling 수행)

### Using Pvsync2 engine 
- pvsync2 engine 상에서 FIO test 를 수행하였다. direct random read with Interrupt mode vs. direct random read with Polling mode.
#### Interrupt mode
![image](https://user-images.githubusercontent.com/18457707/66545200-2a9acd80-eb75-11e9-826c-1afb3bff5064.png)
#### Polling mode
![image](https://user-images.githubusercontent.com/18457707/66545284-4c945000-eb75-11e9-98a4-42be21c4fda7.png)
- ctx 가 9856080 --> 44 로 확연히 감소한 것을 확인할 수 있으며 IOPS, BW, latency 입장에서 약 18% 의 성능 개선 확인
- 따로 로그로 뽑지 않았으나 htop 을 통해서 모니터링 한 결과 polling 모드에서 하나의 cpu 를 100% 로 계속 사용하는 것을 확인

### Using IO_URING engine
- IO_URING 모드에서 polling 을 사용하는데 몇가지 에러 사항이 있었다. 해상 이슈는 [issue#4](https://github.com/Csoyee/documents/issues/4) 참조

- IO_URING engine 상에서 FIO test 수행하였다. 
#### Interrupt mode (Buffered)
#### Interrupt mode (Direct)
#### POlling mode (Direct)

---
## TODO
- 실험 옵션 추가
- io_uring 엔진 실험 결과 추가 
