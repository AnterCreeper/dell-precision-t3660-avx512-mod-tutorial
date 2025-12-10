由于 BootGuard 限制，任何对 BIOS 的修改都会触发完整性检查警告，从而无法启动。
1. 检查微码版本，确认其小于等于 0x15 。
2. 物理上检查 CPU 的 Batch Code, 确认其物理上支持 AVX512。本人的 CPU 为 i5-12500 六大核, 步进为 5。有小核的请关掉小核。
3. 使用物理编程器导出全部数据。确认 Nor Flash 芯片工作电压为 3.3V
4. 使用 UEFITool 打开二进制数据
5. 分析 SiInitFsp 模块  
```
B9 AF 00 00 00 mov eax 0xAF
0F 30 wrmsr
```  
- 0xAF is an undocumented MSR Register: Bit0 for disabling both AVX2 and AVX-512 features, Bit1 is for AVX-512 disabling. So, if there is 0xAF=0x2, then only AVX-512 is disabled.

6. 一共有两个 Branch ，一个是 AVX 禁用，而另一个是 AVX512 禁用。我们注意到
```
83 C8 02 or ecx 2
```
根据 5 中的定义，修改为
```
83 C8 00 or ecx 0
```
