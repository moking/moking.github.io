The page shows how we can use CXL memory as normal RAM.

# Prerequisite
1. Linux kernel is compiled with CXL support. Check [here](https://sunfishho.github.io/jekyll/update/2022/07/07/setting-up-qemu-cxl.html) for more detailed instructions.
2. The ndctl tool has been installed. Check [ndctl ](https://github.com/pmem/ndctl) for instructions to compile and install ndctl tools (cxl, ndctl and daxctl).
3. Installing cxl related modules (if compared as kernel modules): ``` modprobe -a cxl_acpi cxl_core cxl_pci cxl_port cxl_mem```
4. Make sure qemu patched with [hw/i386/pc.c: CXL Fixed Memory Window should not reserve e820 in bios](https://lore.kernel.org/linux-cxl/20221026004737.3646-2-gregory.price@memverge.com/). Should be the case for new Qemu release in 2023.

## Step 1: Create a cxl region with cxl command
* For example: **cxl create-region -m -d decoder0.0 -w 1 mem0 -s 512M**, the output looks below. We can also use **cxl list -R** command to show the region created.
```
{
  "region":"region0",
  "resource":"0xa90000000",
  "size":"512.00 MiB (536.87 MB)",
  "interleave_ways":1,
  "interleave_granularity":256,
  "decode_state":"commit",
  "mappings":[
    {
      "position":0,
     "memdev":"mem0",
      "decoder":"decoder2.0"
    }
  ]
}
cxl region: cmd_create_region: created 1 region
```
# Step 2: Create a namespace with ndctl command
```
cmd: ndctl create-namespace -m dax -r region0                                                                                                                                                        
{                                                                                                                                                                                                    
  "dev":"namespace0.0",                                                                                                                                                                              
  "mode":"devdax",                                                                                                                                                                                   
  "map":"dev",                                                                                                                                                                                       
  "size":257949696,                                                                                                                                                                                  
  "uuid":"8fb092ba-a4ef-4a9a-8d83-d022c518ddf7",                                                                                                                                                     
  "daxregion":{                                                                                                                                                                                      
    "id":0,                                                                                                                                                                                          
    "size":257949696,                                                                                                                                                                                
    "align":2097152,                                                                                                                                                                                 
    "devices":[                                                                                                                                                                                      
      {                                                                                                                                                                                              
        "chardev":"dax0.0",                                                                                                                                                                          
        "size":257949696,                                                                                                                                                                            
        "target_node":1,                                                                                                                                                                             
        "align":2097152,                                                                                                                                                                             
        "mode":"devdax"                                                                                                                                                                              
      }                                                                                                                                                                                              
    ]                                                                                                                                                                                                
  },                                                                                                                                                                                                 
  "align":2097152                                                                                                                                                                                    
}                                         
```
* Note: after the operation, we should see a new device under /dev, like /dev/dax0.0.

# Step 3: Converting a regular devdax mode device to system-ram mode with daxctl
```
cmd: lsmem                                                                                                                                                                                           
RANGE                                 SIZE  STATE REMOVABLE BLOCK                                                                                                                                    
0x0000000000000000-0x000000007fffffff   2G online       yes  0-15                                                                                                                                    
0x0000000100000000-0x000000027fffffff   6G online       yes 32-79                                                                                                                                    
                                                                                                                                                                                                     
Memory block size:       128M                                                                                                                                                                        
Total online memory:       8G                                                                                                                                                                        
Total offline memory:      0B                                                                                                                                                                        
                                                                                                                                                                                                     
cmd: daxctl reconfigure-device --mode=system-ram --no-online dax0.0                                                                                                                                  
reconfigured 1 device                                                                                                                                                                                
[                                                                                                                                                                                                    
  {                                                                                                                                                                                                  
    "chardev":"dax0.0",                                                                                                                                                                              
    "size":257949696,                                                                                                                                                                                
    "target_node":1,                                                                                                                                                                                 
    "align":2097152,                                                            
    "mode":"system-ram",                                                        
    "online_memblocks":0,                                                       
    "total_memblocks":1                                                         
  }                                                                             
]                                                                               
                                                                                
cmd: lsmem                                                                      
RANGE                                  SIZE   STATE REMOVABLE BLOCK             
0x0000000000000000-0x000000007fffffff    2G  online       yes  0-15             
0x0000000100000000-0x000000027fffffff    6G  online       yes 32-79             
0x0000000a98000000-0x0000000a9fffffff  128M offline             339             
                                                                                
Memory block size:       128M                                                   
Total online memory:       8G                                                   
Total offline memory:    128M                        
```
* Note: compared the lsmem outputs before and after the daxctl operation, we can see a new memory has shown up as "offline".

# step 4: Online the memory
```
cmd: daxctl online-memory dax0.0                                                                                                                                                                     
onlined memory for 1 device                                                                                                                                                                          
                                                                                                                                                                                                     
cmd: lsmem                                                                                                                                                                                           
RANGE                                  SIZE  STATE REMOVABLE BLOCK                                                                                                                                   
0x0000000000000000-0x000000007fffffff    2G online       yes  0-15                                                                                                                                   
0x0000000100000000-0x000000027fffffff    6G online       yes 32-79                                                                                                                                   
0x0000000a98000000-0x0000000a9fffffff  128M online       yes   339                                                                                                                                
                                                                                                                                                                                                     
Memory block size:       128M                                                                                                                                                                        
Total online memory:     8.1G                                                                                                                                                                        
Total offline memory:      0B            
```
# step 5: Check if CXL memory shows as a new memory-only numa node
```
cmd: numactl -H                                                                 
available: 2 nodes (0-1)                                                        
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15                              
node 0 size: 7950 MB                                                            
node 0 free: 6573 MB                                                            
node 1 cpus:                                                                    
node 1 size: 128 MB                                                             
node 1 free: 128 MB                                                             
node distances:                                                                 
node   0   1~                                                                   
  0:  10  20~                                                                   
  1:  20  10~                
```

# step 6: run simple tests to use memory on different numa node
```
# Use regular RAM
cmd: time numactl --membind=0 dd if=/dev/zero of=/tmp/zero count=1000 bs=1KB                                                                                                                         
1000+0 records in                                                               
1000+0 records out                                                              
1000000 bytes (1.0 MB, 977 KiB) copied, 0.078505 s, 12.7 MB/s                   
0.03user 0.11system 0:00.16elapsed 96%CPU (0avgtext+0avgdata 1932maxresident)k  
136inputs+1960outputs (2major+172minor)pagefaults 0swaps                        
                                                                                
# Use CXL memory
cmd: time numactl --membind=1 dd if=/dev/zero of=/tmp/zero count=1000 bs=1KB    
1000+0 records in                                                               
1000+0 records out                                                              
1000000 bytes (1.0 MB, 977 KiB) copied, 0.713924 s, 1.4 MB/s                    
0.30user 0.65system 0:00.96elapsed 99%CPU (0avgtext+0avgdata 1924maxresident)k  
0inputs+1960outputs (0major+173minor)pagefaults 0swaps      
```

# References
* Related kernel patch:https://patchwork.kernel.org/project/linux-nvdimm/patch/20190225185740.8660866F@viggo.jf.intel.com/#22501531
* Using instructions from here: https://stevescargall.com/2019/07/09/how-to-extend-volatile-system-memory-ram-using-persistent-memory-on-linux/
