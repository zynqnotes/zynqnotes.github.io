---
layout: post
title:  "用 XSCT 读取 MPSoC 的寄存器"
date:   2018-07-06 12:00:00
categories: Experiences
tags:
- XSCT
- MPSoC
---

## 快速读取一个模块的所有寄存器

```
xsct% target 8
xsct% target
  1  PS TAP
     2  PMU
     3  PL
  4  PSU
     5  RPU
        6  Cortex-R5 #0 (Halted)
        7  Cortex-R5 #1 (Lock Step Mode)
     8* APU
        9  Cortex-A53 #0 (Running)
       10  Cortex-A53 #1 (Power On Reset)
       11  Cortex-A53 #2 (Power On Reset)
       12  Cortex-A53 #3 (Power On Reset)

xsct% rrd
             adma_ch0               adma_ch1               adma_ch2
             adma_ch3               adma_ch4               adma_ch5
             adma_ch6               adma_ch7                 afifm0
               afifm1                 afifm2                 afifm3
               afifm4                 afifm5                 afifm6
             ams_ctrl          ams_pl_sysmon          ams_ps_sysmon
         apm_cci_intc                apm_ddr           apm_intc_ocm
          apm_lpd_fpd                    apu           axipcie_dma0
         axipcie_dma1           axipcie_dma2           axipcie_dma3
      axipcie_egress0        axipcie_egress1        axipcie_egress2
      axipcie_egress3        axipcie_egress4        axipcie_egress5
      axipcie_egress6        axipcie_egress7       axipcie_ingress0
     axipcie_ingress1       axipcie_ingress2       axipcie_ingress3
     axipcie_ingress4       axipcie_ingress5       axipcie_ingress6
     axipcie_ingress7           axipcie_main                  bbram
                 can0                   can1                cci_gpv
              cci_reg    coresight_a53_cti_0    coresight_a53_cti_1
  coresight_a53_cti_2    coresight_a53_cti_3    coresight_a53_dbg_0
  coresight_a53_dbg_1    coresight_a53_dbg_2    coresight_a53_dbg_3
  coresight_a53_etm_0    coresight_a53_etm_1    coresight_a53_etm_2
  coresight_a53_etm_3    coresight_a53_pmu_0    coresight_a53_pmu_1
  coresight_a53_pmu_2    coresight_a53_pmu_3      coresight_a53_rom
   coresight_r5_cti_0     coresight_r5_cti_1     coresight_r5_dbg_0
   coresight_r5_dbg_1     coresight_r5_etm_0     coresight_r5_etm_1
     coresight_r5_rom    coresight_soc_atm_0    coresight_soc_atm_1
  coresight_soc_cti_0    coresight_soc_cti_1    coresight_soc_cti_2
  coresight_soc_etf_1    coresight_soc_etf_2      coresight_soc_etr
    coresight_soc_ftm   coresight_soc_funn_0   coresight_soc_funn_1
 coresight_soc_funn_2   coresight_soc_replic      coresight_soc_rom
    coresight_soc_stm     coresight_soc_tpiu    coresight_soc_tsgen
              crf_apb                crl_apb                    csu
               csudma                csu_wdt                   ddrc
              ddr_phy           ddr_qos_ctrl          ddr_xmpu0_cfg
        ddr_xmpu1_cfg          ddr_xmpu2_cfg          ddr_xmpu3_cfg
        ddr_xmpu4_cfg          ddr_xmpu5_cfg                     dp
                dpdma                  efuse                fpd_gpv
             fpd_slcr        fpd_slcr_secure           fpd_xmpu_cfg
        fpd_xmpu_sink               gdma_ch0               gdma_ch1
             gdma_ch2               gdma_ch3               gdma_ch4
             gdma_ch5               gdma_ch6               gdma_ch7
                 gem0                   gem1                   gem2
                 gem3                   gpio                    gpu
                 i2c0                   i2c1                iou_gpv
            iou_scntr             iou_scntrs        iou_secure_slcr
             iou_slcr                    ipi                lpd_gpv
             lpd_slcr        lpd_slcr_secure               lpd_xppu
        lpd_xppu_sink              mbistjtag                   nand
                  ocm           ocm_xmpu_cfg            pcie_attrib
           pmu_global                    puf                   qspi
                  rpu                    rsa               rsa_core
                  rtc          sata_ahci_hba  sata_ahci_port0_cntrl
sata_ahci_port1_cntrl       sata_ahci_vendor                    sd0
                  sd1                 serdes                   siou
             smmu_gpv               smmu_reg                   spi0
                 spi1                   swdt                   ttc0
                 ttc1                   ttc2                   ttc3
                uart0                  uart1                 usb3_0
          usb3_0_xhci                 usb3_1            usb3_1_xhci
             vcu_slcr                    wdt        alg_vcu_dec_top
      alg_vcu_enc_top

xsct% rrd uart0
                 control: 00000114                      mode: 00000020
               intrpt_en: 00000000                intrpt_dis: 00000000
             intrpt_mask: 00000000              chnl_int_sts: 00000e1a
           baud_rate_gen: 000000ad              rcvr_timeout: 00000000
rcvr_fifo_trigger_level0: 00000020                modem_ctrl: 00000000
               modem_sts: 000000fb               channel_sts: 0000000a
             tx_rx_fifo0: 00000000         baud_rate_divider: 00000004
              flow_delay: 00000000    tx_fifo_trigger_level0: 00000020
     rx_fifo_byte_status: 00000000

```

## 读取 A53 内部特殊寄存器

比如要读取 `sctlr_el1`这个没有地址的 ARM 内部寄存器。

```
xsct% target 9
xsct% target
  1  PS TAP
     2  PMU
     3  PL
  4  PSU
     5  RPU
        6  Cortex-R5 #0 (Halted)
        7  Cortex-R5 #1 (Lock Step Mode)
     8  APU
        9* Cortex-A53 #0 (Running)
       10  Cortex-A53 #1 (Power On Reset)
       11  Cortex-A53 #2 (Power On Reset)
       12  Cortex-A53 #3 (Power On Reset)

xsct% rrd
      r0: 00000000ff000000        r1: 000000000000000a
      r2: 0000000000000000        r3: 0000000000000000
      r4: 000000007ff9e398        r5: 0000000000000000
      r6: 0000000000000020        r7: 0000000000000001
      r8: 000000007ff8cae8        r9: 0000000000000008
     r10: 000000007deae436       r11: 0000000000000002
     r12: 0000000000000002       r13: 00000000ffffffff
     r14: 000000007deaecbc       r15: 000000007ff7e0c0
     r16: 0000000000000000       r17: 0000000000000000
     r18: 000000007deaede8       r19: 000000007deafb30
     r20: 000000007ff6b210       r21: 000000007ff8d000
     r22: 000000007ff7d82e       r23: 0000000000000001
     r24: 0000000000000001       r25: 000000007ff8d318
     r26: 0000000000000000       r27: 000000000000000a
     r28: 000000007ff65af0       r29: 000000007deaeb40
     r30: 000000007ff2eeec        sp: 000000007deaeb40
      pc: 000000007ff2eef4      cpsr:         600003c9
     vfp                         sys
acpu_gic

xsct% rrd sys
 0   1   2   3   4   5   6   7   9  10  12  13  14


xsct% rrd sys 1
 sctlr_el1:         30d00800   actlr_el1:         00000000
 cpacr_el1:         00000000   sctlr_el2:         30c51835
 actlr_el2:         00000000     hcr_el2: 0000000000000002
  mdcr_el2:         00000006    cptr_el2:         000033ff
  hstr_el2:         00000000    hacr_el2:         00000000
 sctlr_el3:         00c8181f   actlr_el3:         00000000
   scr_el3:         00000731  sder32_el3:         00000000
  cptr_el3:         00000000    mdcr_el3:         00000000
```
