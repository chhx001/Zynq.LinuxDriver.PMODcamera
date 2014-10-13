VDMA上的DDR时钟，一定要用ARM引出来的时钟，不要用外部时钟。

OV7670传入数据时GBR，如果需要转换YUV,则要先转为BGR之后再转YUV.

OpenCV需要的是原GBR数据，无需转换。


/***************************内核配置**************************/


bash> git clone git://git.xilinx.com/linux-xlnx.git
bash> cd $ZYNQ_TRD_HOME/linux-xlnx
bash> git checkout -b zynq_base_trd_14_3 xilinx-14.3-build2-trd
bash> cp $ZYNQ_TRD_HOME/patches/zynq_base_trd_14_3.patch . // copy the 14.3 TRD patch from package to dev PC
bash> git apply --stat zynq_base_trd_14_3.patch       // display contents of patch
bash> git apply --check zynq_base_trd_14_3.patch      // check if patch can be applied
bash> git am zynq_base_trd_14_3.patch                 // apply the patch
bash> make ARCH=arm zynq_base_trd_defconfig
bash> make ARCH=arm uImage
此时uImage生成在arch/arm/boot/uImage


若要用这个内核启动ubuntu,请用ubuntu12.04版本，可登录

若要使用camera,修改内核目录下driver/video/xilinx/xvdma.c
在相应函数(括号内函数)后面，加上
EXPORT_SYMBOL(xvdma_get_dev_info);
EXPORT_SYMBOL(xvdma_start_transfer);
EXPORT_SYMBOL(xvdma_prep_slave_sg);
EXPORT_SYMBOL(xvdma_device_control);
重新编译内核



/**********************devivetree.dts***************/

vdma的deviceteee:

axi_vdma_0: axivdma@43000000 {
                        #address-cells = <1>;
                        #size-cells = <1>;
                        compatible = "xlnx,axi-vdma";
                        ranges = <0x43000000 0x43000000 0x10000>;
                        reg = <0x43000000 0x10000>;
                        xlnx,flush-fsync = <0x1>;
                        xlnx,num-fstores = <0x3>;
                        dma-channel@43000000 {
                                compatible = "xlnx,axi-vdma-mm2s-channel";
                                interrupt-parent = <&ps7_scugic_0>;
                                interrupts = <0 59 4>;
                                xlnx,datawidth = <0x18>;
                                xlnx,device-id = <0x0>;
                                xlnx,genlock-mode = <0x3>;
                        } ;
                        dma-channel@43000030 {
                                compatible = "xlnx,axi-vdma-s2mm-channel";
                                interrupt-parent = <&ps7_scugic_0>;
                                interrupts = <0 58 4>;
                                xlnx,datawidth = <0x10>;
                                xlnx,device-id = <0x0>;
                                xlnx,genlock-mode = <0x2>;
                        } ;
                } ;



/***********************初始化配置********************/

使用mknod /dev/xvdma c 10 224创建xvdma设备

/***********************camera.c说明******************/

startVDMA函数为VDMA的配置以及启动函数

chan_cfg.config.vsize = 480;  //高度
chan_cfg.config.hsize = 640 * 3;  //单行数据量（字节）
chan_cfg.config.stride = 640 * 3;  //步长，需要大于等于数据量，两者相同就是连续数据，否则数据不连续

chan_cfg.config.park = 1;   //VDMA开启PARK模式
chan_cfg.config.park_frm = 0;   //初始PARK在第0缓冲区

buf_info.addr_base = 0x19950000;   //buffer的位置和长度
buf_info.buf_size = 0x00870000;

iowrite32(0x100,vdma_base+0x28);  //改为PARK第1缓冲区

0x28的第8-16位为写PARK寄存器，0就刷0缓冲区，1就刷1缓冲区
--------------------------
vivi_fillbuf将物理内存复制进虚拟内存

void *pbuf = buffer_base[frame];
     if (!vbuf)
          return;

     for (h = 0; h < hmax; h++){
          //memcpy(vbuf + h * wmax * dev->pixelsize,
          memcpy(vbuf+h*wmax*dev->pixelsize,pbuf+h*wmax*dev->pixelsize,wmax*dev->pixelsize);
     }

     unsigned int mask = ioread32(vdma_base + 0x28);  
     mask = mask & 0xFFFFE0FF | (frame << 8);  //切换PARK
     iowrite32(mask,vdma_base+0x28);

     frame = (frame+1) % 2;
-------------------------------------------------
vivi_create_instance（）初始化设备
//这部分映射物理地址
for(i = 0;i < 2; ++ i){
          buffer_base[i] = phys_to_virt(0x19950000 + BUFFER_OFFSET * i);
          printk("buf_base%d:%p\n",i,buffer_base[i]);
     }
     vdma_base = ioremap(0x43000000,4);

