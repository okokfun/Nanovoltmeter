# 纳伏电压表
低噪声纳伏表设计文件与代码

该设计包含五块电路板:
1. 纳伏表主板
2. 一块开关电源子板(SMPS)，用于将电池电压降压至设计中使用的多种电源轨电压
3. 一块配备电池管理系统(BMS)的电池板，适用于4串21700电池组
4. 一块带电池充电器的后面板电路板
5. 一块前面板

此代码库还将包含STM32U575微控制器的源代码，该控制器用于连接各类外围设备。

![ADR1399-boardfront](https://github.com/curtisseizert/Nanovoltmeter/assets/22351001/9ea7e242-64c5-4fc7-a24f-2e9ce9876d46)
![ADR1399-boardrear](https://github.com/curtisseizert/Nanovoltmeter/assets/22351001/88aedb9b-be14-4447-87e4-bf99c72b94a3)

注意: 当前版本的电路板未经修改无法正常工作，但原理图中已添加注释说明所需修改。本项目仍应视为处于开发阶段，但此处提供了完整的设计资料，以供从事低噪声、高分辨率电压传感项目的研究者参考使用。

当前状态
-------------
由于我当前正在完成另一项目，电路板的C修订版正处于间歇性开发阶段。新版设计将着力提升系统的稳健性，重点解决以下问题：对共模噪声的明显敏感性、电源保护不足导致特定故障时的级联失效，以及与电池放电相关的轻微漂移现象。我正在研究改进从ADC转换中获取SPI数据的方法，旨在实现AD4030-24芯片的最大采样率(2 MSPS)，以降低噪声并减少混叠可能性。本次修订还包括对关键前端放大器的微调设计，以提升增益和稳定性。此外，新版将取消ADR1399基准电压源选项。

固件2.0版本优化了数据处理与用户输入的交互逻辑，并通过增强DMA利用率降低了CPU负载。单元测试正在进行中，部分功能尚未完全实现，但即将陆续发布。该版本采用类SCPI语法构建命令体系，虽未完全遵循SCPI规范，但升级后的固件将更有利于开展系统性能测试工作。

我已制作了两块此类电路板：一块采用LTC6655基准源，另一块采用ADR1399基准源。针对ADR1399版本，我通过内部短路和外部1 mV基准信号对其噪声与偏移性能进行了测试。为使两块电路板正常工作均需进行若干修改，我已在C修订版原理图中进行了相应调整(但尚未完成版图设计)。在内部短路测试条件下，采用LTC6655基准源的电路板展现出更优异的噪声性能与温度稳定性。

设计背景
-----------------

此前已构建的输入级概念验证模型显示，其噪声谱密度(NSD)在约1.2 nV/$\sqrt{Hz}$水平保持平坦。在每秒1次采样的条件下，峰峰值噪声约为 15nV。该仪表的温度系数约为1 nV/K，但对温度波动的敏感性较为明显，且电池放电过程中功耗变化导致约 1nV/小时的漂移。此原型机还因输入调制器开关的电荷注入效应存在约 3nA 的偏置电流。本修订版本拟针对这些问题进行如下改进：

1. 电路板的主要电源轨采用低噪声降压转换器(LT8608S)与Cuk转换器，并集成在带屏蔽的子板上，以确保板载低压差LDOs在电池放电全程保持稳定的压降裕量
2. 优化布局设计降低了对温度变化的敏感性。输入级被置于屏蔽外壳内，周围环绕隔离槽。此外，主要的功率耗散元件被安置在屏蔽罩外侧的对向位置。
3. 输入开关现采用低电容四路MOSFET阵列(PE4140)，其栅极驱动电压由四通道DAC独立产生。模型实验已证明，通过调节DAC开关电平与死区时间，将偏置电流降至<5pA具有可行性，但该方面仍需进一步优化。
4. 现采用高速精密ADC(AD4030-24)检测开关周期间的偏移误差，通过16位DAC调制输入JFET的沟道电流，从而控制输入差分放大器的偏移电压。根据原理图设计，该DAC的微调范围约为+/-625μV(最小分辨率LSB=20nV)。微调差分放大器的低增益特性，使得其在输入极限条件下的跨导变化率＜1%。这种设计可在仪器预热阶段测量微调增益，实现控制回路的单周期稳定。
5. 控制回路的数字化实现还允许关闭自动调零功能，或以极低频率运行输入开关，从而最大程度减少电荷注入效应。

其他功能包括:
1. 提供两种电压基准的封装选项: 采用ADR1399可实现最低噪声/最高稳定性，选用LTC6655则能降低功耗
2. 一个光耦隔离的外部触发输入接口，例如可用于将时钟同步至市电周期。
3. 输入保护功能在差分电压达2V时启动限流(典型值10mA，最大值15mA)，可防止端子意外接入100V电压时受损。电路同时配备气体放电管(GDT)，为更高电压的瞬态冲击提供防护。
4. 两个输入量程：±2 mV 与 ±20 mV。
5. (采用LTC6655时)功耗降低

工作原理
----------------------

<img width="1753" alt="SimplifiedSchematic" src="https://github.com/curtisseizert/Nanovoltmeter/assets/22351001/f38c8695-773f-48de-b3a6-2627330c2b3a">

输入调制器(SW1)交替切换输入信号与反馈信号，将其连接至高增益、低噪声分立式JFET差分放大器的输入端。差分放大器的输出端通过同步解调器(SW2)连接至运算放大器(U10)的输入端口，前端输出信号即取自该运放。ADC读取调制器每个相位对应的输出电压。若两相位读数差值超过死区阈值，控制器将相应增减Vos DAC(U44)的数值，通过调节输入差分放大器的沟道电流来抵消差分放大器的偏移。沟道电流由U20构成的低增益跨导级控制。为消除ADC及相关信号调理电路的偏移电压与1/f噪声，在ADC驱动器输入端设置切换开关，将该模块的输入信号反向连接，使得调制器/解调器的工作频率成为ADC输入开关频率的整数倍。

该拓扑结构可将输入级的失调电压(及1/f噪声)调制到开关频率，而输入信号本身不被调制。通过失调归零反馈回路，这个经调制的误差信号被大幅抑制，从而显著降低斩波频率处的噪声功率，并采用用户可选阶数(1至3阶)的级联积分梳状(sinc)滤波器进行滤波。由于输入级JFET的输入电容较大(约75pF)，差分放大器失调归零的同时也降低了斩波频率下的电流脉冲干扰。

因此，理论上唯一存在的失调、漂移或1/f噪声源来自连接解调后差分放大器输出的运算放大器，而所有这些误差源都会受到差分放大器增益(约3500倍)的衰减。输入级JFET采用类Sziklai拓扑与PNP型双极晶体管Q16、Q21配对，以提升输入放大器的跨导，使其能在高增益下工作。漏极电阻R91和R100的取值是噪声、增益与稳定性之间的折衷：在133Ω阻值下，每个JFET的漏极电流约为1mA，双极晶体管的集电极电流也各约1mA。更大的电阻值会略微增加噪声、提高增益，并因环路增益增大而降低复合放大器的相位裕度。设计中采用的运算放大器(ADA4625-1)是一款高精度、低噪声器件，其失调和漂移极低，当被3500倍衰减后这些误差源可忽略不计。按典型规格书数值计算，该运放约带来4nV失调电压、0.06nV/K温度系数，其1/f噪声拐点频率约为0.1MHz。实际应用中，寄生效应(特别是热电动势)被证实是更大的误差源，前面板处未校正的失调电压约为250nV，1/f拐点频率略低于1MHz(取决于环境温度稳定性)。测量(内部)短路输入时的噪声谱密度＜1.2nV/√Hz，在10秒采样周期下，我观测到10000秒内的峰峰值噪声仅为1.36nV。

<img width="3249" height="2311" alt="ShortNoise_LTC6655_sinc2" src="https://github.com/user-attachments/assets/e446475a-df6a-4a1f-96a1-a6e8d4d18aea" />

观测记录
------------

调制器频率在600-2000Hz范围内较为理想，系统默认设定为1800Hz。在失调校正环路中设置死区是必要的，可防止模拟输出频谱中出现伪影。关键问题在于对低频噪声源（可能来自开关电源(SMPS)，也可能包括60Hz工频干扰）较为敏感。调制频率最高可提升至14.4kHz，但随之而来的前端稳定时间与ADC信号调理电路时耗会占据更大比例的潜在转换时间，从而导致噪声增加。

<img width="3277" height="2311" alt="NVM B no deadband, 1 LSB inc, offsets" src="https://github.com/user-attachments/assets/7f692774-247d-44a0-b52e-0cfccc16df01" />
NSD without deadband, showing aliasing of V_os correction loop

The biggest limitation in the utility of the instrument is the considerable increase in noise and change in offset that occurs when something is connected to the front panel. This effect occurs regardless of whether the input relay is measuring an internal short. It seems most likely that the case, which is tied to protective earth via the USB cable shield is shifting in potential relative to circuit ground, and this is capacitively coupling to something sensitive. However, connecting the USB cable through an isolator, which disconnects the shield from PE, actually makes the noise situation worse rather than better. 

Implementation of a Sinc2 filter improved some apparent aliasing artifacts that were present in the spectrum at very low (<10 mHz) frequencies. Use of a Sinc3 filter did not lead to any noticeable improvents. The flatness of the noise spectrum (with an internal short) to below 1 mHz was very repeatable provided that environmental conditions were suitably stable.

<img width="3249" height="2311" alt="ShortNoise_LTC6655_sinc1" src="https://github.com/user-attachments/assets/1055977f-d3ce-460c-9226-e360cdb6e724" />
<img width="3249" height="2311" alt="ShortNoise_LTC6655_sinc2" src="https://github.com/user-attachments/assets/e779fb28-2be0-43f0-8d62-cbfe5655bbc9" />

While the nanovoltmenter has an offset temperature coefficient of <1 nV/K, it does show some sensitivity to changes in the rate of temperature change (i.e., the offset voltage correlates to the second derivative of temperature with respect to time). This makes sense because those changes lead to changes in temperature gradients across the board, which influences parasitic thermocouple behavior at the input. The power supply is also somewhat more efficient at lower input voltages, which leads to reduced heat dissipation as the batteries discharge, influencing the offset. In very long captures, such as the LTC6655 reference example shown below, this leads to a drift of <0.1 nV/h. It is notable that the overall drift in this 100h capture was only about 8 nV despite about 2.5 K fluctuation in input temperature over that time.

<img width="3540" height="2340" alt="LongCapture_LTC6655_0p01SPS_sinc2" src="https://github.com/user-attachments/assets/bc8703da-5e35-4ad9-97cf-4228a2d44c8a" />
<img width="3540" height="2340" alt="LongCapture_ADR1399_0p01SPS" src="https://github.com/user-attachments/assets/30137af6-4506-4fc8-9191-a88c3c338448" />

The linearity of the amplifier is expected to be quite good as the offset correction loop keeps the input terminals of the frontend amplifier at the same voltage regardless of input. The main source of nonlinearity should be the gain setting resistors, as temperature changes due to increased power dissipation may lead to changes the resistor ratio and the gain of the frontend. To test the linearity of the amplifier, I used a highly accurate source based on an LTZ1000A and DAC11001B with a tested linearity error of <2 ppm and a 200K/20R divider to reduce the output voltage range to 0 to 1 mV. I averaged several 11 point sweeps over this range to obtain a maximum linearity error of 1.5 ppm. The averaging was necessary due the increased noise seen with something plugged in to the front panel. The increased sensitivity of the lower gain +/-20 mV range to external noise sources (probably due to reduced stability of the amplifier at lower gain) meant that determining linearity error for this range was not practical. In all measurements of external sources, I found it was optimal to wrap the test lead around a nanocrystalline toroid, which did reduce noise considerably.

<img width="2680" height="1569" alt="LinearitySweep" src="https://github.com/user-attachments/assets/795f5279-375e-4419-a274-6ec6c7499851" />

Performance Reductions with External Measurements
-------------------------------------------------

Taken together, my observations about the difficulty of performing measurements with external sources are difficult to untangle and may point to multiple simultaneous issues. I have collected them here to try to provide some insight into this behavior and possible solutions:
1. The 40 dB gain range is more sensitive than the 60 dB gain range. The lower gain range has lower phase margin and still takes 60 dB gain at higher frequencies, as this was necessary to ensure stability of the amplifier at lower gain. This points to stability related problems, such as degradation of phase margin (leading to ringing, etc. after switching events) when the amplifier sees an inductive source impedance. Inductive sources present a considerable challenge with such amplifiers because of the large gate-source capacitance of the paralleled JFETs. This can give resonant gain with inductive inputs.
2. The improvement in performance when using a common mode choke on the input suggests an effect from frequencies well above the amplifier's passband. The inductance of the common mode choke was likely aorund 800 uH, which would present negligible impedance at 60 Hz.
3. I recorded spectra from the analog outputs with the instrument powered by a bench power supply to remove SMPS signals. With 60 dB gain on the frontend and the 10 nF input capacitor switched off, the peaks at harmonics of the chopping frequency were about 20 dB higher with an external short than with an internal short. This is presumably an effect of inductance at the input as there are ferrite beads in the input protection circuitry between the internal and external shorts. Some of these artifacts may lead to aliasing.
Internal Short Analog Output, Gain = 1000
<img width="1920" height="1027" alt="NVM B Internal short Gain1000 Linear supply CapOff" src="https://github.com/user-attachments/assets/c67c16a7-33d1-4317-96ab-d731726bbf0c" />
External Short Analog Output, Gain = 1000<img width="1920" height="1027" alt="NVM B External short Gain1000 Linear supply CapOff" src="https://github.com/user-attachments/assets/356a7ddc-46ca-4e82-b0e1-3d37ad8144b1" />
4. Plugging a source into the input leads to a shift in offset with an internal short (ca. 200 nV). The source of this effect is not clear. The source mentioned in the linearity test section actually has an internal 100:1 attenuator for low voltages, but directly plugging the source into the NVM gave such large increases in noise that it would not be practical to measure ppm level offsets. The input connector is a LEMO-0S-304 type with the shell of the connector electically connected to the instrument chassis, which is itself connected to protective earth via the USB cable. In general, cables with the shield connected at both ends gave significantly poorer performance than those with the shield left unconnected at one end, suggesting that connection of the two chassis grounds was problematic despite their isolation from circuit ground. The use of nylon screws to connect the USB connector to the rear panel of the source (and hence remove the connection to protective earth) was necessary for reliable results in that case. Most of the sensitive circuitry of the input is very well shielded against electrostatic pickup by the use of board-level RF shields, but the input protection circuitry and switching relays, as well as the chopper demodulator and op amp, are outside of these shields. A ground loop is not necesasary to cause this effect, as it is observed even when an unconnected cable is plugged into the front panel. Some additional tests are necessary to determine the minimum configuration needed to cause this shift (e.g., does the behavior occur with just connection of the shell, or do the actual input leads (which provide a connection to the board) need to be connected as well? If it seems that additional shielding is needed to reduce this effect, it may be difficult to maintain the same form factor.
