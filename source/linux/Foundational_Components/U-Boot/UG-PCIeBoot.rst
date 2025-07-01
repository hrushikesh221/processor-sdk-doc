.. _pcie_boot:

==============
PCIe Boot Mode
==============

Overview
--------

PCIe (Peripheral Component Interconnect Express) boot mode allows
|__PART_FAMILY_NAME__| family of devices, configured as a PCIe endpoint, to boot by
receiving boot loader images over PCIe from a host system acting as a
PCIe root complex (RC). In this mode, the root complex is
responsible for discovering the endpoint device and transferring the
required boot loader images for each boot stage.

This document provides a step-by-step guide to configuring and using
PCIe boot mode. PCIe boot is supported on the |__PART_FAMILY_NAME__| SoC family. The
following instructions and switch settings are specific to the |__PART_FAMILY_NAME__| EVM;
for other boards, consult the corresponding hardware documentation.

Boot Mode Switch Settings
-------------------------

To enable PCIe boot mode, configure the boot mode switches as follows:

.. ifconfig:: CONFIG_part_variant in ('AM64X')

   .. code-block:: text

      SW2 (B0 - B7):   1 1 0 1 0 1 1 0
      SW3 (B8 - B15):  0 0 0 0 0 0 0 0

.. note::

   DIP switch settings are EVM-specific and may not apply to all board designs.

Board Setup (Connection Example)
--------------------------------

The following is an example of a connection in which the root complex (host) and the
endpoint (|__PART_FAMILY_NAME__| device) are physically connected using a PCIe cable
or connector, as shown in the figure below.

.. image:: /images/AM64X_PCIe_boot.jpg

Both boards should be powered off before making the connection, and the PCIe link
securely established before powering on the devices.

Other hardware configurations are possible. So adapt the setup steps as
applicable to your board design.

Endpoint Configuration
----------------------

The following configuration options are used to set up the |__PART_FAMILY_NAME__| device
as a PCIe endpoint for PCIe boot. These options must be set in the
board's defconfig in U-BOOT for the corresponding boot loader image.

- ``CONFIG_PCI_DFU_BAR_SIZE``:
  Configures the size of the PCIe BAR (Base Address Register) that is
  exposed for device firmware update (DFU) and boot loader image download.

- ``CONFIG_PCI_DFU_VENDOR_ID``:
  Specifies the PCIe vendor ID to be advertised by the endpoint.

- ``CONFIG_PCI_DFU_DEVICE_ID``:
  Specifies the PCIe device ID to be advertised by the endpoint.

- ``CONFIG_PCI_DFU_MAGIC_WORD``:
  Magic word written by the root complex at the end of the image transfer to
  signal to the endpoint that the boot loader image is ready for processing.

- ``CONFIG_PCI_DFU_BOOT_PHASE``:
  Specify the current boot phase when booting via DFU over PCIe.
  This value can be read by the root complex to determine the
  current boot phase. Value of this config is written to memory
  location (BAR_start + PCI_DFU_BAR_SIZE - 70). Max size of this
  config is 63 bytes.

.. ifconfig:: CONFIG_part_variant in ('AM64X')

   .. note::

      All the configs required for PCIe boot are enabled in
      ``am64x_evm_a53_defconfig`` and ``am64x_evm_r5_defconfig`` by default.

      By default, PCIe root complex mode is enabled in the device tree.
      To enable endpoint mode, the boot loaders must be built with the
      device tree overlay ``k3-am642-evm-pcie0-ep.dtso``.

Ensure these configuration options are set appropriately in the build
environment to enable a successful PCIe boot process.

PCIe Boot Procedure
-------------------

1. After configuring the boot mode switches on the endpoint and
   connecting it to the root complex as shown in the figure, power
   on the endpoint.

2. On the root complex, rescan the PCIe bus to enumerate the PCIe
   endpoint. The endpoint will appear as a RAM device on the root
   complex. The enumeration may look similar to the following:

   .. code-block:: text

        01:00.0 RAM memory: Texas Instruments Device b010
            Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
            Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
            Latency: 0
            Interrupt: pin A routed to IRQ 526
            Region 0: Memory at 68100000 (32-bit, non-prefetchable) [size=1M]
            Region 1: Memory at 68200000 (32-bit, prefetchable) [size=2M]
            Region 2: Memory at 6a000000 (64-bit, prefetchable) [size=32M]
            Region 4: Memory at 6c000000 (64-bit, prefetchable) [size=32M]

3. Copy ``tiboot3.bin`` to Region 1.

   .. ifconfig:: CONFIG_part_variant in ('AM64X')

      After the root complex has finished copying the image,
      it must write the PCIe boot data address to ``0x701BCFE0``.

      For example, if the root complex loads the image at offset
      ``0x1000``, then it should write ``0x70001000`` (Internal RAM
      memory base + offset) to ``0x701BCFE0``. This notifies the ROM
      that the image is ready to be authenticated and processed.

4. Once ``tiboot3.bin`` is transferred, rescan the PCIe bus on the
   root complex to enumerate the PCIe endpoint device, in order to
   transfer the next stage boot loader. The enumeration may now look
   like the following:

   .. code-block:: text

        0000:01:00.0 RAM memory: Texas Instruments Device b010
            Subsystem: Device 7003:beef
            Flags: bus master, fast devsel, latency 0, IRQ 644
            Memory at 12000000 (32-bit, prefetchable) [size=4M]
            Capabilities: [80] Power Management version 3
            Capabilities: [90] MSI: Enable+ Count=1/1 Maskable+ 64bit+
            Capabilities: [b0] MSI-X: Enable- Count=1 Masked-
            Capabilities: [c0] Express Endpoint, IntMsgNum 0
            Capabilities: [100] Advanced Error Reporting

5. At this stage, only one memory region will be visible. Copy
   ``tispl.bin`` to this region. After the copy, the root complex
   must write a 4-byte magic word (defined in the defconfig) at the
   end of the memory region. This indicates to the endpoint that the
   boot loader image has been copied.

6. Repeat steps 4 and 5 to transfer ``u-boot.img`` using the same
   procedure.

Sample Host Program for Image Transfer
--------------------------------------

The following sample C program can be used on the root complex to
copy boot loader images (such as ``tiboot3.bin``, ``tispl.bin``, and
``u-boot.img``) to the PCIe endpoint device by writing them to the
appropriate memory regions using ``/dev/mem``.

.. code-block:: c

   #include <sys/mman.h>
   #include <sys/stat.h>
   #include <fcntl.h>
   #include <unistd.h>
   #include <stdio.h>
   #include <stdlib.h>
   #include <string.h>

   int main(int argc, char *argv[])
   {
      char *bootfilename = NULL;
      off_t bar1_address = 0;
      int fd;
      void *map_base;
      long image_size, map_size;
      char *buffer;
      unsigned int *buffer_32;
      int i;
      FILE * fptr;
      off_t load_addr, load_addr_offset, start_addr_offset;

      if (argc != 3) {
         printf("Usage: %s <bar_address> <binary_file>\n", argv[0]);
         return 0;
      }

      bar1_address = strtoul(argv[1], NULL, 16);
      bootfilename = argv[2];

      printf("bootfilename: %s\n", bootfilename);
      printf("bar1_address: 0x%lx\n", bar1_address);

      if(!strcmp(bootfilename,"tiboot3.bin"))
      {
         load_addr = 0x70000000;
         load_addr_offset = 0x1000;
         start_addr_offset = 0x1bcfe0;
      }
      else
      {
         load_addr = 0xdeadbeef;
         load_addr_offset = 0x00;
         start_addr_offset = 0x3ffffc;
      }

      printf("load_addr: 0x%lx\n", load_addr);
      printf("load_addr_offset: 0x%lx\n", load_addr_offset);
      printf("start_addr_offset: 0x%lx\n", start_addr_offset);

      printf("try to open /dev/mem.\n");
      fd = open("/dev/mem", O_RDWR | O_SYNC);
      if (fd < 0) {
         printf("open /dev/mem failed.\n");
         return 0;
      }
      printf("/dev/mem opened.\n");
      (void)fflush(stdout);

      fptr = fopen(bootfilename, "rb");
      if (!fptr) {
         printf("open %s failed.\n", bootfilename);
         return 0;
      }
      printf("%s opened.\n", bootfilename);

      (void)fseek(fptr, 0L, SEEK_END);
      image_size = ftell(fptr);
      printf("image_size: %ld\n", image_size);
      fseek(fptr, 0, SEEK_SET);

      printf("%s: image_size: %ld\n", __func__, image_size);

      map_size = 0x400000;

      printf("%s: map_size: %ld\n", __func__, map_size);

      map_base = mmap(0, map_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, bar1_address);
      if (map_base == MAP_FAILED) {
         printf("mmap failed.\n");
         return 0;
      }
      printf("map_base: 0x%lx\n", (unsigned long)map_base);
      (void)fflush(stdout);

      buffer = malloc(image_size);
      if (!buffer) {
         printf("malloc failed.\n");
         return 0;
      }

      buffer_32 = (unsigned int *)buffer;
      fread(buffer, (size_t)image_size, 1, fptr);
      printf("Read image of %ld bytes\n", image_size);

      printf("Writing image to memory\n");
      for(i = 0; i < (int)image_size; i+=4)
      {
         *(unsigned int *)( map_base + load_addr_offset + i) = buffer_32[i/4];
      }
      printf("done.\n");
      fflush(stdout);

      sleep(1);

      *(unsigned int *)(map_base + start_addr_offset) = (unsigned int)(load_addr_offset + load_addr);
      return 0;
   }

Usage Example
^^^^^^^^^^^^^

To copy a boot loader file (e.g., ``tiboot3.bin``) to the PCIe device,
run:

.. code-block:: bash

   sudo ./pcie_boot_copy 0x68200000 tiboot3.bin

Replace ``0x68200000`` with the appropriate BAR region address as
enumerated on the root complex, and specify the correct binary file.
