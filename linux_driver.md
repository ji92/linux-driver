# Standard Network card driver framework
使用总线设备驱动框架，当有新的驱动注册，总线进行两者的匹配，匹配后调用probe函数，进行初始化。
+ PCI设备的扫描发生操作系统的启动过程中
## driver
``` c


```
## device
``` c

```
# Device Tree
## DTC/DTS/DTB

![driver.png](https://raw.githubusercontent.com/ji92/linux-driver/main/img/driver.png)
+ DTS经DTC编译后生成DTB文件，可由linux内核解析
+ DTS文件可能包含很多公共部分，可以提炼为dtsi文件（对应C语言的头文件），在DTS文件中包含
+ 为电路板制作镜像时，一般会单独留下一块很小的区域存放dtb文件，bootloader引导内核时会先读取dtb到内存
## DTS grammer
### hardware structure
![driver.png](https://raw.githubusercontent.com/ji92/linux-driver/main/img/hardware_structure.png)
### DTC code
``` dts
/{  //root
    compatible = "acme, coyotes-revenge";   //root node compatibility, contain 2 or over 2 compatibility string. Format:<manufacturer>, <model> 
    #address-cells = <1>;   //address code, determine format of reg 
    #size-cells = <1>;
    interrupt-parent = <&intc>;     //board-level interrupt controller

    cpus {
        #address-cells = <1>;
        #size-cells = <0>;
        cpu@0 {
            compatible = "arm, cortex-a9";  //device node compatibility.It's for binding between driver and device when struct of_device_id contain the same string.
            reg = <0>;
        };
        cpu@1 {
            compatible = "arm, cortex-a9";
            reg = <1>;            
        };
    };

    serial@101f0000 {
        compatible = "arm, p1011";
        reg = < 0x101f0000 0x1000 >;
        interrupts = < 1 0 >;
    };
    serial@101f2000 {
        compatible = "arm, p1011";
        reg = < 0x101f2000 0x1000 >;
        interrupts = < 2 0 >;
    };

    gpio@101f3000 {
        compatible = "arm, p1061";
        reg = < 0x101f3000 0x1000
                0x101f4000 0x0010 >;
        interrupts = < 3 0 >;
    };

    intc: interrupt-controller@10140000 {   //intc is a label accessed by &label(pointer handle)
        compatible = "arm, p1190";
        reg = < 0x10140000 0x1000 >;
        interrupt-controller; //interrupt controller flag
        #interrupt-cells = <2>; //refer to binding document for meaning of each cell 
    };

    spi@10115000 {
        compatible = "arm, pl1022";
        reg = < 0x10115000 0x1000 >;
        interrupt = < 4 0 >;    //a node can have multiple interrupts like <4 0>, <5, 0>, the key interrupt-names can be used for alias. Corresponding interrupt will be got by platform_get_irq_byname() in platform_driver.
    };

    external-bus {
        #address-cells = <2>;
        #size-cells = <1>;
        ranges = <0 0 0x10100000 0x10000    
                  1 0 0x10160000 0x10000
                  2 0 0x30000000 0x1000000>;    //<cs chip_offset mem_addr length> describe memory map between cpu memzone and external memzone


        ethernet@0,0 {  //chip select and address offset
            compatible = "smc, smc91c1111";
            reg = < 0 0 0x1000 >;
            interrupts = < 5 2 >;
        };

        i2c@1,0 {
            compatible = "acme,a1234-i2c-bus";
            #address-cells = <1>;
            size-cells = <0>;
            reg = < 1 0 0x1000 >;
            intertupt = < 6 2 >;
            rtc@58 { // 58 is i2c address 
                compatible = "maxim,ds1338";
                reg = <58>;
                interrupts = < 7 3 >;
            };
        };

        flash@2,0 {
            compatible = "samsung,k8f1315ebm", "cfi-flash";
            reg = < 2 0 0x4000000 >;
        }
    };
}
```
