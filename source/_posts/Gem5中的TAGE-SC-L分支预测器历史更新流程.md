---
title: Gem5中的TAGE-SC-L分支预测器历史更新流程
date: 2025-07-29 17:02:30
tags: [Gem5]
---

分析Gem5中的TAGE-SC-L分支预测器工作流程，重点关注预测性历史更新策略的技术实现。
<!-- more -->

---


## 一、指令创建

文件路径：./src/cpu/o3/fetch.cc

instruction在buildInst函数执行后生成，随后会逐步调用函数，调用顺序如下：

```C++
// fetch.cc:
->lookupAndUpdateNextPC()
    ->branchPred->predict()
// bpred_unit.cc:
    ->lookup()
// tage.cc:
    ->predict()
// tage-sc-l.cc:
    ->tage->tagePredict()
    ->loopPredictor->loopPredict()
    ->statisticalCorrector->scPredict()
```

## 二、TAGE预测

文件路径：./src/cpu/pred/tage_sc_l.cc.cc

在tage_sc_l.cc.cc中，会分别调用TAGE、Loop、SC预测器进行预测，首先讨论TAGE预测器。在调用tage->tagePredict()后，函数调用路径如下：

```C++
// tage-sc-l.cc:
    ->tage->tagePredict()
// tage-base.cc:
    ->calculateIndicesAndTags()
        // tage_sc_l.cc:
        ->calculateIndicesAndTags() // 使用的是重写后的实现。
        ->gindex()
            // 这里使用的历史是通过threadHistory结构进行维护的
            index = shifted_pc ^
                (shifted_pc >> ((int) abs(logTagTableSizes[bank] - bank) + 1)) ^
                threadHistory[tid].computeIndices[bank].comp ^
                F(threadHistory[tid].pathHist, hlen, bank);
        ->gtage()
            // 这里TAGE_SC_L_64KB.cc 重写了函数
            // tage_sc_l_64KB.cc:
            Addr shifted_pc = pc >> instShiftAmt;
            int tag = shifted_pc ^ threadHistory[tid].computeTags[0][bank].comp ^
              (threadHistory[tid].computeTags[1][bank].comp << 1);
            return (tag & ((1ULL << tagTableTagWidths[bank]) - 1));
            // 同样使用threadHistory结构进行历史计算
    ->bindex()
    //Look for the bank with longest matching history
    //Look for the alternate bank
    //computes the prediction and the alternate prediction

```

### 2.1 threadHistory结构

threadHistory结构体定义如下：

```C++
// Keep per-thread histories to
// support SMT.
struct ThreadHistory
{
    // Speculative path history
    // (LSB of branch address)
    int pathHist;           // 路径历史
    // Non-speculative path history
    int nonSpecPathHist;    // 非预测性路径历史

    // Speculative branch direction
    // history (circular buffer)
    std::vector<uint8_t> globalHist;    // 预测性分支历史

    // Index to most recent branch outcome
    int ptGhist;

    // Speculative folded histories.
    FoldedHistory *computeIndices;      // 实际参与计算的部分
    FoldedHistory *computeTags[2];      // 实际参与计算的部分
};

// 其中FoldedHistory是折叠的历史，每次新的历史值来了后，可以通过update进行更新，回滚也可以通过restore进行恢复。

struct FoldedHistory
{
    unsigned comp;
    int compLength;
    int origLength;
    int outpoint;
    int bufferSize;

    FoldedHistory()
    {
        comp = 0;
    }

    void init(int original_length, int compressed_length)
    {
        origLength = original_length;
        compLength = compressed_length;
        outpoint = original_length % compressed_length;
    }

    void update(uint8_t * h)
    {
        comp = (comp << 1) | h[0];
        comp ^= h[origLength] << outpoint;
        comp ^= (comp >> compLength);
        comp &= (1ULL << compLength) - 1;
    }

    void restore(uint8_t * h)
    {
        comp ^= h[origLength] << outpoint;
        auto tmp = (comp & 1) ^ h[0];
        comp = (tmp << (compLength-1)) | (comp >> 1);
    }
};

```


其中 pathHist/computeIndices/


## 三、非预测性路径历史更新

更新函数调用路径：

```C++
// fetch.cc:
// 如果是预测正确的指令，即没有发生流水线冲刷时的分支指令正常提交了
->branchPred->update()
    // bpred_unit.cc:
    while (!predHist[tid].empty() &&
        predHist[tid].back()->seqNum <= done_sn) {
        commitBranch(tid, *predHist[tid].rbegin());
        delete predHist[tid].back();
        predHist[tid].pop_back();
    }
    // 其中predHist是一个vector元素，保存的是History，
    // typedef std::deque<PredictorHistory*> History;
    // History是PredictorHistory指针的队列

    ->commitBranch()
        // tage_sc_l.cc
        ->update()
        // tage_base.cc
        ->tage->updateHistories()
                threadHistory[tid].nonSpecPathHist = 
                    calcNewPathHist(tid, branch_pc, bi->pathHist, taken, branchTypeExtra(inst), target);
            // 非预测性路径历史phist更新，phist是分支PC的某一位，通常是最低位。

```

## 三、预测性历史更新

```C++
// fetch.cc:
->lookupAndUpdateNextPC()
    ->branchPred->predict()
    // bpred_unit.cc
    ->updateHistories()
        // tage_sc_l.cc
        ->updateHistories()
            // tage.cc
            ->updateHistories()
            // 此时是预测性更新
            ->updatePathAndGlobalHistory()
                // tage_sc_l.cc
                ->updatePathAndGlobalHistory()
                    // Update path history
                    tHist.pathHist =
                        calcNewPathHist(tid, branch_pc, tHist.pathHist, taken, brtype, target);
                ->updateGHist()
                    // tage_base.cc
                    ->updateGHist()

```

## 四、预测性历史恢复

```C++
// fetch.cc:
->branchPred->squash()
    // bpred_unit.cc
    ->squash()
        // tage_sc_l.cc
        ->update()
            // tage_base.cc
            ->squash()
            ->updateHistories()
            ->restoreHistState()    // 在这个函数里面完成历史的恢复
            // 对于pathHist而言，其是随bi一起流动的
            // 对于globalHist而言，其是保存的vector数组，利用指针ptGhist进行恢复的
```