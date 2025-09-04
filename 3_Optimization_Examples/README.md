# Optimization Examples
# 1. Optimize the Efficiency of Flashing Firmware
- Original Version
    1. When flashing the firmware, all physical blocks in NAND Flash must be erased to reset the data storage state. Physical blocks are the basic unit to erase in NAND Flash. In this version, the controller issues the erase command to one specific block at a time and does not issue the next erase command until the previous block has been fully erased. If there are four channels, there will be four corresponding busy latencies. The illustration is as follows:
    ![Flash_FW_1](../Image/3_Optimization_Examples/Flash_FW_1.png)

    2. Since only one physical block is erased at a time, handling error messages is straightforward if NAND Flash fails to erase a block.

- Request
    1. To improve production line efficiency, the firmware flashing time should be minimized.

- Refined Version
    1. In the SSD architecture, each channel operates independently, allowing them to process commands in parallel.

    2. By issuing erase commands to all channels simultaneously, NAND Flash only incurs a single erase latency, after which all targeted blocks across channels are erased together. The illustration is as follows: <br>
    ![Flash_FW_2](../Image/3_Optimization_Examples/Flash_FW_2.png)

    3. The error message contains the erase results of blocks across all channels. If an error occurs, the firmware must analyze the report to identify which block failed to erase. Once the failed block is pinpointed, the firmware can continue with the appropriate error-handling procedure.

    4. By shortening the total latency required to erase blocks across channels, the overall time spent flashing new firmware is reduced. This optimization improves firmware update efficiency and, in turn, increases productivity on the production line.

# 2. Shorten the Latency of Firmware Update
- Original Version
    1. The host can issue NVMe commands, such as Firmware Download and Firmware Commit, to update the SSD firmware after it has been flashed on the production line. To execute the new firmware, the device must pass through the ROM Code, which loads the updated firmware into the ATCM. During a firmware update, the CPU clock speed used in ROM Code execution is the same as that used for a normal power cycle. The flow of Firmware Update could be divided into following subsections: <br>
    ![fw_update](../Image/3_Optimization_Examples/fw_update.png)

    2. Both firmware updates and normal power cycles go through the ROM Code. However, their behaviors differ slightly. In the case of a firmware update, it is permissible to raise the CPU clock speed during ROM Code execution, which can help shorten the overall firmware update latency. 

- Request
    1. In order to meet customer requirements, the total firmware update latency must be reduced.

- Refined Version 
    1. Divide the process into multiple subsections and measure the latency of each.

    2. Identify that ROM code execution latency is longer compared to other subsections.
    
    3. Increase the CPU clock speed during ROM code execution.
    
# 3. Optimize the Placement of ARM Overlay to Enhance the FW Execution Efficiency 
- Original Version
    1. When the CPU executes firmware code, the code must reside in the ATCM region, which includes both the common code region and the overlay region. The code in the common region remains static, while the overlay region is dynamically updated by loading code segments from NAND Flash. This continuous loading induces overhead, which can hinder overall performance. If the placement of ARM Overlay could be optimized, it could improve the performance. <br> 

    Take the following figure as example. Since Func_C resides in Overlay_3, firmware has to spend additional efforts to load Overlay_3 from NAND Flash to ATCM after executing Overlay_2's Func_B.
    ![ARM_Overlay_1](../Image/3_Optimization_Examples/ARM_Overlay_1.png)

- Request: 
    1. In order to enhance the Read/Write performance, the frequency of overlay switching should be reduced.

- Refined Version
    1. According to the ARM compilation results, the overlay to which each function belongs is known.

    2. Take the above case as an example. Placing Func_C in Overlay_2 eliminates the need for the firmware to load Overlay_3 into ATCM, which saves extra effort. The illustration is shown as following:
    ![ARM_Overlay_2](../Image/3_Optimization_Examples/ARM_Overlay_2.png)

    3. To optimize read/write execution speed, I first identify the most frequently used function and its corresponding hierarchical calling path. My goal is to optimize the placement of functions within the overlay along the target calling path. 

    4. The calling path of the function could be expressed as a tree structure as following. The optimization scenario could be classified as two cases: <br>
    ![ARM_Overlay_3](../Image/3_Optimization_Examples/ARM_Overlay_3.png)
        1. Case_1 (Target on the specific function): <br>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Func_E(red circle) is a very low-level function, and it is preferred to be placed in ATCM’s Common Code. As long as the low-level function is placed in the Common Code, the firmware does not need to switch the Overlay Region regardless of which Overlay was last used, since the Common Code always resides in ATCM. 

        2. Case_2 (Target on the frequently used call path): <br>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Optimize the most frequently called function first, followed by the second most frequently called function, and so on. Take the following figure as example. Func_G is called more frequently than Func_F, Func_I, Func_A, so the path of Func_G →  Func_H →  Func_E (purple circle) would be optimized first. <br>
        &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;After optimizing the path Func_G → Func_H → Func_E, the next step is to optimize the path of Func_F with (Func_G → Func_H → Func_E) and Func_I with (Func_G → Func_H → Func_E). During the optimization of a target path, if a function has already been updated in the previous round, the path is considered fully updated, and optimization is halted to avoid duplicate computation.
        The pseudo code is expressed as following:
        ![ARM_Overlay_4](../Image/3_Optimization_Examples/ARM_Overlay_4.png)
    