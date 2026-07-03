1. 背景
Cachegrind是Valgrind的一个子功能，它可以在一个软件模拟的处理器上解释执行二进制文件中的指令，精确捕获所有的内存访问通过软件模拟Cache的Hit和Miss行为，以达到分析程序的Cache使用情况的目的。也由于是解释执行的缘故，Cachegrind也能精准捕捉到每个函数消耗的指令数（注意不是Cycle数）。

对于嵌入式软件开发来说，Cachegrind比起其它类似工具有如下优点：

结果稳定：程序完全在模拟环境解释执行，不存在Cache被别的系统程序占用污染的问题。分析数据可稳定重现，利于比较。
无附加干扰：程序完全在非侵入状态下执行不需要插桩，所以不存在插桩或打点代码自身污染Cache的问题。
针对性强：可以只跟踪PC-IT中特定测试用例，从而实现对特定业务流程的分析。对于业务团队来说实用性更强。
精准精细：相对于基于踩点统计采样的工具来说，基于解释执行的Cachegrind在访存事件的捕捉上可以做到非常精准，适合进行非常细粒度（代码行级别）的分析。
不需要上板：直接在Remote ARM Server上可以完成全部采集和分析工作，所有工具已预装。一次分析时间小于十分钟。
由于Cachegrind是纯软件模拟Cache行为，所以使用前也必须认识到其有如下局限性：

Cachegrind无法精确模拟硬件预取器的行为（仅具备简单流式预取模拟，不支持1383所支持的步长预取等高级功能），所以要验证最终结果必须以上板运行后的结果为准，Cachegrind只能作为定位排查Cache Miss问题的工具。

同样的，软件添加的Data Prefetch指令也无法在Cachegrind中看到效果，因此不能用来验证预取的结果。

由于指令Cache的命中率与函数代码段在二进制文件中的分布情况直接相关，所以用PCIT在Cachegrind上跑出来的I-Cache Miss数据可能失真较大，在进行I-Cache优化时要仔细评估其数据是否可参考。

2. Cachegrind使用
2.1. 数据采集
一般我们SSH到远程编译服务器以后运行Cachegrind采集PCIT的某个测试流程以捕获特定业务流程中的数据：

$ valgrind --tool=cachegrind --I1=[L1指令Cache配置] --D1=[L1数据Cache配置] --L2=[L2 Cache配置] [待测程序] [待测程序参数]
以L2 CTRL PC-IT捕捉PucchMuCapbilityTest.PucchCapbilityTest用例Cache数据为例：

$ valgrind --tool=cachegrind --I1=65536,4,64 --D1=65536,4,64 --L2=524288,8,64 ./l2_pc_it_ctrl --gtest_filter=PucchMuCapbilityTest.PucchCapbilityTest
分析完成后会得到如下Cache访问摘要：

[-----------------------------------  RESULT  --------------------------------]
[==========] 1 tests from 1 test cases ran (439325 ms total)
[  PASSED  ] 1 tests
 
==13556==
==13556== I   refs:      31,577,068,886
==13556== I1  misses:       615,689,926
==13556== LLi misses:        35,032,534
==13556== I1  miss rate:           1.95%
==13556== LLi miss rate:           0.11%
==13556==
==13556== D   refs:      16,050,676,200  (11,012,012,431 rd   + 5,038,663,769 wr)
==13556== D1  misses:       402,156,003  (   373,736,913 rd   +    28,419,090 wr)
==13556== LLd misses:       349,750,337  (   323,960,739 rd   +    25,789,598 wr)
==13556== D1  miss rate:            2.5% (           3.4%     +           0.6%  )
==13556== LLd miss rate:            2.2% (           2.9%     +           0.5%  )
==13556==
==13556== LL refs:        1,017,845,929  (   989,426,839 rd   +    28,419,090 wr)
==13556== LL misses:        384,782,871  (   358,993,273 rd   +    25,789,598 wr)
==13556== LL miss rate:             0.8% (           0.8%     +           0.5%  )
以及名为cachegrind.out.[PID]的输出文件，该文件可用分析工具进一步解析。

2.2. 数据解析
使用cg_annotate [cachegrind.out.xxx]可以对Cachegrind的输出文件进行解析。一般在分析的时候由于指令Cache和L2 Cache的数据没有参考价值，所以可以通过--show开关来仅显示有价值的部分，通过--sort开关来指定按照D-Cache Miss的数量排序，指定--threshold=0显示所有函数的分析数据备查（如有必要的话，默认为仅显示0.1%以上热点函数），打开--auto=yes对每个函数进行自动代码注释（该功能非常好用）。

如：使用如下命令解析并标注一个输出文件，把解析结果保存到cg_result.txt中：

$ cg_annotate --show=Ir,Dr,D1mr,Dw,D1mw --sort=D1mr,D1mw --threshold=0 --auto=yes cachegrind.out.18509 > cg_result.txt
经过解析生成的文件包含两个部分：上半部分为统计总表，列出了各函数在运行过程中的指令数和Cache访问情况。分析时我们主要关注D1mr（D-Cache读Miss）和D1mw（D-Cache写Miss）这两列指标。这部分数据可以很容易地导入Excel进行分析和排序，将优化目标缩小到函数级：

--------------------------------------------------------------------------------
           Ir          Dr       D1mr          Dw       D1mw  file:function
--------------------------------------------------------------------------------
    2,704,344     746,809     92,260     189,523        546  /app/l2/pucchsch/resmng/comm/pucchsch_resmng_usrmng.c:PUCCHSCH_ForceClearUserPool
    1,598,660     554,122     80,534     164,674          0  /app/l2/ulsch/puschusrmng/common/ulsch_usrmng_usrmem.c:ULSCH_USRMEM_MemCheckProc
   31,022,632   9,240,784     78,919   4,620,392        125  /app/l2/mcm/cellinstmng/mcm_cellmng_paratblmng.c:MCM_GetCellTagBuffer
    2,365,536   1,125,072     70,918     288,480          0  /app/l2/lib/parallel/l2lib_trclib_historytrc.c:TRCLIB_HistoryTrcRecord
   21,002,367   7,516,241     69,688   3,377,004        536  /app/l2/inf/l2infsal/eg/l2inf_egcheck.c:L2INF_IsRcvPidMatchedEg
    7,776,462   2,666,709     60,529   1,555,491          3  /app/l2/inf/l2infhal/common/dbg/l2inf_hal_log.c:L2INF_HAL_WriteLog
      991,650     312,520     60,100     132,220          0  /app/l2/cchp/ra/raservice/cchp_ra_ttiprocess.c:CCHP_RaCtrlTrpTtiProc
...
下半部分为代码标注。如下是其中一个标注了的函数的例子：

--------------------------------------------------------------------------------
-- Auto-annotated source: /5g_build/5g_Main/WN_5G_BTS_L2L3_21A/app/l2/dlsch/lib/dlsch_lib.c
--------------------------------------------------------------------------------
     Ir     Dr  D1mr    Dw D1mw
 
      .      .     .     .    .  UINT8 DLSCH_GetPdschHarqProcNum(L2L3_PDSCH_HARQ_PROC_NUM_ENUM_UINT8 harqProcNum)
  4,726      0     0 2,363    0  {
 23,630  4,726     0 2,363    0      UINT8 num = (UINT8)VC_MIN(harqProcNum, PDSCH_HARQ_PROC_NUM_BUTT);
 11,815  7,089 1,768     0    0      return g_pdschHarqProcNum[num];
  4,726      0     0     0    0  }
标注后的代码非常直观，比如我们可以看到以上函数在返回g_pdschHarqProcNum[num]的时候出现了大量的D-Cache读Miss。通常我们在前面的统计总表中定位到感兴趣的函数之后，再到这里搜索进一步定位到代码行和数据结构级别。

2.3. 数据比较
由于测试环境本身在初始化和插桩时产生底噪，为了能更好地识别出Cache数据中与容量直接相关的部分，可以利用cg_diff工具进行比较求差值。

cg_diff是Valgrind工具集中的一个Perl小脚本，它可以计算两份cachgrind.out文件的统计差值，并生成一份新的out文件（通过输出重定向）。新产生的out文件可以再利用cg_annotate解析，转换为可读的报告。在实际使用中，cg_diff主要用在如下场景：

2.3.1. 过滤底噪
在性能优化过程中我们可能只关心那些和容量/终端数相关的性能数据，而不希望被系统本身的初始化/测试桩/框架本身产生的开销所干扰，此时可以利用cg_diff对这类底噪进行过滤。

如：在分析下行调度性能时，我们可以先对1UE的测试输入采集一次Cachegrind数据，然后改为10UE再跑一次，最后对比两份数据：

$ cg_diff cachegrind.out.1UE cachegrind.out.10UE > delta.out
即可得到一份从10UE的数据中扣除1UE部分的差值输出文件（理论上就是9UE的开销）。再用cg_annotate对得到的差值输出进行解析，识别出其中与UE数量成比例相关的部分（通常在这一步可以排除掉大量Init和Stub函数）。

2.3.2. 检查改动的结果
在对代码进行修改之后，我们要重新采集Cachegrind数据与修改之前的结果做比较，从而确定对代码复杂度或者Cache的优化是否达到预期效果。此时使用cg_diff可以凸显变化的部分，便于结果对比分析。

cg_diff的另一个重要用处是可以用来在优化效果不如预期时帮助定位问题。如：在尝试通过调整结构体内字段排布试图优化DLSCH_FillGroupingUsrSchInfo的D-Cache写Miss时发现最终结果不降反升，使用cg_diff比较修改前后的Cache数据即可发现虽然DLSCH_FillGroupingUsrSchInfo的Cache Miss如预期下降，但是别的一些用到这个数据结构的函数的Cache Miss反而上升，这说明了这次数据结构调整有副作用。此时可进一步查看受影响函数的代码行Cache Miss标注，调整修改方案：

--------------------------------------------------------------------------------
       Ir      Dr   D1mr    Dw   D1mw  file:function
--------------------------------------------------------------------------------
        0       0      0     0 -3,403  /app/l2/dlsch/usr_sch_info/fill_usr_sch_info/dlsch_fill_usr_sch_info.c:DLSCH_FillGroupingUsrSchInfo
   -5,884       0      0     0  1,506  /app/l2/dlsch/usr_sch_info/fill_usr_sch_info/dlsch_fill_usr_drb_info.c:DLSCH_SetSingleDrbInfo
        0       0      0     0   -122  /app/l2/dlsch/scheduler/usr_group_sch/lf/dlsch_spa_grp_comm.c:DLSCH_SpaGrpSaveUserToGroup
        0       0      0     0    113  /app/l2/pdcchsch/pdcchctrl/comm/pdcchsch_ctrl.c:PDCCHSCH_FillCceResult：
2.4. 数据过滤
前面得到的测试用例中可能仍包含大量测试框架和桩函数的结果干扰分析，可以简单地用grep和正则表达式过滤掉总表中的非业务函数部分：

$ grep -vE "(::)|test/|(\?\?\?)|stub" [cg_annotate输出文件] > [过滤后文件]
3. 举个栗子
使用Cachegrind分析测试用例PucchMuCapbilityTest.PucchCapbilityTest分别在1UE和400UE时的性能数据，用cg_diff获取两者的差值，并过滤掉非业务代码后，发现SUSR_RiValidCheck函数产生了最多的D-Cache读Miss（D1mr）：

		Ir			Dr		 D1mr			Dw	D1mw	file:function
96,960,496	33,170,696	4,791,182	20,412,736	   0	app/l2/susr/susrcomm/meaproc/comm/susr_meaproc_cqiproc.c:SUSR_RiValidCheck
在400UE的cg_annotate输出文件中查询对应函数的代码标注：

--------------------------------------------------------------------------------
-- Auto-annotated source: /5g_build/5g_Main/WN_5G_BTS_L2L3_21A/app/l2/susr/susrcomm/meaproc/comm/susr_meaproc_cqiproc.c
--------------------------------------------------------------------------------
        Ir        Dr      D1mr         Dw  D1mw
         .         .         .          .     . static UINT8 SUSR_RiValidCheck(UINT16 usrId, UINT64 currTti, UsrCsiMeaInfoStru *csiMeaInfo)
12,766,000         0         0 12,766,000     0 {
         .         .         .          .     .     L2INF_FUNCTRACE(SUSR_RiValidCheck);
10,212,800 5,106,400 1,914,900  2,553,200     0     UINT8 codeBookType = SUSR_GetCsiMeaInfoPmiType(csiMeaInfo->ucenPmiCodeBookType);
 7,659,600 2,553,200         0          0     0     if (codeBookType >= SUSR_CODE_BOOK_TYPE_BOTTOM) {
...
 
 7,659,600 5,106,400 1,914,900  2,553,200     0     UINT8 pmiFormatIndicator = csiMeaInfo->csiMeaInfoPmiType[SUSR_CODE_BOOK_TYPE1].pmiFormatIndicator;
 7,659,600 2,553,200         0          0     0     if (pmiFormatIndicator >= CSIRS_PMI_FORMAT_BUTT) {
         .         .         .          .     .         L2INF_WRITELOG(L2_LOG_SUSR_1048, L2INF_ERROR_LOG, PID_5G_SUSR,
...
 
22,978,800 5,106,400         0  2,553,200     0     UsrCsiMeaInfoCommPmiTypeStru *csiMeaInfoPmiType = &(csiMeaInfo->csiMeaInfoPmiType[codeBookType]);
17,872,400 7,659,600   964,598          0     0     if (csiMeaInfoPmiType->riPmiInfo.riValidFlag[pmiFormatIndicator] == L2INF_FALSE) {
 5,106,400         0         0          0     0         return L2INF_FALSE;
         .         .         .          .     .     }
可以看到有三处D1mr飙涨的代码行，分别是访问：

csiMeaInfo->ucenPmiCodeBookType
csiMeaInfo->csiMeaInfoPmiType[0].pmiFormatIndicator
csiMeaInfo->csiMeaInfoPmiType[codeBookType].pmiFormatIndicator.riPmiInfo.riValidFlag[pmiFormatIndicator]
其中①属于对外部传入非栈内结构体的第一次访问，这类Cache Miss通常无法避免；③是对数组的单次随机访问，这类访问导致的Cache Miss通常在数组内成员结构较大的时候也难以避免。

而②是一个看上去很有可能优化的点，因为它是对内嵌结构体数组的第一个元素的访问，所以是有可能通过将其紧密排布在①所访问的成员下方来减少Cache Miss的。

研究一下这个结构体定义：

typedef struct {
2=> UsrCsiMeaInfoCommPmiTypeStru csiMeaInfoPmiType[SUSR_CODE_BOOK_TYPE_BOTTOM];
 
    UINT8 cqiEnableCompareFlag;
    UINT8 cqiFirstRptFlag;
    UINT8 cqiFirstRptFlag2Weight;
    UINT8 cqiRptValidFlag;
    SusrRsCsiMeasTypeEnumUint8 csiMeasType;
1=> UINT8 ucenPmiCodeBookType;
    UINT8 ucenPmiCodeBookMode;
    UINT8 ucenPannelNum;
    ...
} __L2_BYTE_ALIGN_8__ UsrCsiMeaInfoStru;
可以看到第二次访问的地址在第一次访问的前方较远处——这一定会导致Cache Miss。所以我们可以考虑把这个数组调整到第一次访问的地址后面试试看？运气好的话第一次Cache Miss所加载的64字节可能就能包含第二次访问的地址呢：

typedef struct {
    UINT8 cqiEnableCompareFlag;
    UINT8 cqiFirstRptFlag;
    UINT8 cqiFirstRptFlag2Weight;
    UINT8 cqiRptValidFlag;
    SusrRsCsiMeasTypeEnumUint8 csiMeasType;
1=> UINT8 ucenPmiCodeBookType;
    UINT8 ucenPmiCodeBookMode;
    UINT8 ucenPannelNum;
 
2=> UsrCsiMeaInfoCommPmiTypeStru csiMeaInfoPmiType[SUSR_CODE_BOOK_TYPE_BOTTOM];
    ...
} __L2_BYTE_ALIGN_8__ UsrCsiMeaInfoStru;
数组排好了，再来看看这个结构体数组内要被访问的csiMeaInfoPmiType[0].pmiFormatIndicator是不是离前面的ucenPmiCodeBookType足够近。再看看UsrCsiMeaInfoCommPmiTypeStru的定义：

typedef struct {
    UINT16 subBandCsiNum;
    UINT8 cqiValidFlag;
    UINT8 validBeamNum;
    USRSCH_PDSCH_CQI_TYP_ENUM_UINT8 cqiUptTyp;
    UINT8 portNum;
    SUSRPDCCH_CQI_TYP_ENUM_UINT8 enCqiPushType;
 
2=> CSIRS_PMI_FORMAT_ENUM_UINT8 pmiFormatIndicator;
 
    ...
} __L2_BYTE_ALIGN_8__ UsrCsiMeaInfoCommPmiTypeStru;
还行吧，正好在头8个字节，也就是在外部大数据结构UsrCsiMeaInfoStru的第16字节处，被一个Cache Line覆盖的概率还是挺高的，所以我们可以再跑一把Cachegrind试试看。

因为这个例子中数据结构只是8字节对齐，并非Cache Line对齐。所以在一些情况下可能前16字节正好分别处于两个Cache Line中，依然无法减少Cache Miss。这种情况下如果数据结构足够大（能抵消Cache Line对齐带来的存储开销），并且优化后带来的性能提升值得去优化，则可以考虑将外层数据结构改为Cache Line对齐。

对代码做如下小修改（简单地挪了一下csiMeaInfoPmiType数组的位置）：

diff --git a/app/l2/include/susr/susrmng/susrmng_susrcommmeatagstru.h b/app/l2/include/susr/susrmng/susrmng_susrcommmeatagstru.h
index ff3f204a529..5815b6fe67a 100644
--- a/app/l2/include/susr/susrmng/susrmng_susrcommmeatagstru.h
+++ b/app/l2/include/susr/susrmng/susrmng_susrcommmeatagstru.h
@@ -450,8 +450,6 @@ typedef struct {
 } __L2_BYTE_ALIGN_8__ UsrCsiMeaInfoCommPmiTypeStru;
 
 typedef struct {
-    UsrCsiMeaInfoCommPmiTypeStru csiMeaInfoPmiType[SUSR_CODE_BOOK_TYPE_BOTTOM];
-
     UINT8 cqiEnableCompareFlag;
     UINT8 cqiFirstRptFlag;
     UINT8 cqiFirstRptFlag2Weight;
@@ -461,6 +459,8 @@ typedef struct {
     UINT8 ucenPmiCodeBookMode;
     UINT8 ucenPannelNum;
 
+    UsrCsiMeaInfoCommPmiTypeStru csiMeaInfoPmiType[SUSR_CODE_BOOK_TYPE_BOTTOM];
+
     SusrAssistTrpList csiMeasAssistTrpList;
再跑一遍Cachegrind，对比一下数据：

--------------------------------------------------------------------------------
-- Auto-annotated source: /5g_build/5g_Main/WN_5G_BTS_L2L3_21A/app/l2/susr/susrcomm/meaproc/comm/susr_meaproc_cqiproc.c
--------------------------------------------------------------------------------
        Ir        Dr      D1mr         Dw  D1mw
         .         .         .          .     . static UINT8 SUSR_RiValidCheck(UINT16 usrId, UINT64 currTti, UsrCsiMeaInfoStru *csiMeaInfo)
12,766,000         0         0 12,766,000     0 {
         .         .         .          .     .     L2INF_FUNCTRACE(SUSR_RiValidCheck);
10,212,800 5,106,400 1,914,900  2,553,200     0     UINT8 codeBookType = SUSR_GetCsiMeaInfoPmiType(csiMeaInfo->ucenPmiCodeBookType);
 7,659,600 2,553,200         0          0     0     if (codeBookType >= SUSR_CODE_BOOK_TYPE_BOTTOM) {
...
 
 7,659,600 5,106,400         0  2,553,200     0     UINT8 pmiFormatIndicator = csiMeaInfo->csiMeaInfoPmiType[SUSR_CODE_BOOK_TYPE1].pmiFormatIndicator;
 7,659,600 2,553,200         0          0     0     if (pmiFormatIndicator >= CSIRS_PMI_FORMAT_BUTT) {
...
 
         .         .         .          .     .     /* 当前RI已经无效，则直接返回 */
25,532,000 5,106,400         0  2,553,200     0     UsrCsiMeaInfoCommPmiTypeStru *csiMeaInfoPmiType = &(csiMeaInfo->csiMeaInfoPmiType[codeBookType]);
17,872,400 7,659,600   964,598          0     0     if (csiMeaInfoPmiType->riPmiInfo.riValidFlag[pmiFormatIndicator] == L2INF_FALSE) {
 5,106,400         0         0          0     0         return L2INF_FALSE;
         .         .         .          .     .     }
可以看到之前访问csiMeaInfo->csiMeaInfoPmiType[0].pmiFormatIndicator产生的Cache Miss消失了，说明改动符合我们的预期。

再比较一下修改前后的总Cache Miss数（400UE、320TTI情况）：

Before:
==6760== D   refs:      16,050,632,904  (11,011,958,530 rd   + 5,038,674,374 wr)
==6760== D1  misses:       404,417,472  (   376,018,652 rd   +    28,398,820 wr)
==6760== D1  miss rate:            2.5% (           3.4%     +           0.6%  )
 
After:
==13556== D   refs:      16,050,676,200  (11,012,012,431 rd   + 5,038,663,769 wr)
==13556== D1  misses:       402,156,003  (   373,736,913 rd   +    28,419,090 wr)
==13556== D1  miss rate:            2.5% (           3.4%     +           0.6%  )
可以看到在总访问存数量不变的情况下，仅调整了一行代码的版本D-Cache Miss数减少了226万次。

注意：

从经验上来说，在Cachegrind验证时必须要能在这类大规模性能测试用例中看到百万次以上Cache Miss减少的效果，上板度量时才会有可观测到的性能增益。

在观测到明显的D1 Miss减少时，应该再使用cg_diff＋cg_annotate进一步确认减少的D-Cache Miss均来自于业务代码而非桩或库代码。本例中验证log没有保存故无法展示。可以用以下grep命令过滤cg_annotate结果中的业务函数部分：

grep -E "/5g_build/5g_Main/.*?/app" diff.txt > app_diff.txt
上板测试修改后的代码：以b7a995254b7版本为基准，测得SUSR_JOBID_UPDATE_USR_PER_TTI任务平均每TTI每用户开销（Cycle数）变化如下：

UE数	基准均值	修改后均值	差值	性能增益
400	25508	23761	1747	7.35%
800	27824	25507	2317	9.08%
由上可看出修改的实际效果符合预期。

注意这个案例之所以有优化空间主要在于这个函数里面实际访问的数据长度并不多只有几个字节，所以才有可能将几次分散的访问合并到一个Cache Line里。然而对于常见的Cache Miss的重灾区：大数据结构的初始化函数通常不满足这样的条件。比如在一个函数里初始化了92字节的结构体成员，那么其中踩到2~3次Cache Miss都是正常的，很可能优化不掉。所以在分析看似油水很大的函数时还是要先估算一下实际读写的数据量，如果在理论值范围内大可不必浪费时间。

所以Cachegrind的Annotate Source功能对于帮助我们识别和解决D-Cache Miss瓶颈点帮助非常大，不再需要像大海捞针一般基本靠猜。

4. 写在后面
优化软件的Cache利用率从来不是件容易的事——即使是有了Cachegrind等工具的帮助，实际操作起来也非常困难，需要极大的努力和不断的试错才能得到一点点突破。Valgrind作者的这段话写得到位：

Even if you know a line of source code is causing a lot of cache misses, and you are confident the misses are slowing down the program a lot, it is not always clear what can be done to improve this. It can be strongly indicative that, for example, a data structure could be redesigned. But the results rarely provide a “smoking gun”, and some non-trivial insight about how the program interacts with the caches is required to act upon them.

There is no silver bullet; optimising a program’s cache utilisation is hard, and often requires trial and error. What Cachegrind does is turn an extremely difficult problem into a moderately difficult one.
