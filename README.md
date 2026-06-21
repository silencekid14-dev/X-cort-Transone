# Transone chip can have even more than 4 binary cores but upto 8, or 16 or 32 binary cores!.
* To support 8, 16, or 32 binary cores without choking the interface, the Transone-OptimusP hard macro would simply scale its internal translation layout.

* Instead of a single translation pipe, the OptimusP can be configured with multiple parallel conversion channels (e.g., 4 or 8 independent Bit-to-Voltage clusters acting at the same time).

* Core Assignment: If you use 32 binary cores, you can group them into clusters of 4. Each cluster gets its own dedicated conversion pipeline into the Analog Network-on-Chip (ANoC) mesh.

* Bulk Streaming Multiplication: The streaming bulk conversion speed would scale far past the original 20 Million words per second limit. This allows massive datasets to flow from the high-capacity binary DRAM into the analog array simultaneously, unlocking smooth, high-frame-rate rendering for massive legacy applications.
