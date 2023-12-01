+++
title = 'Map Reduce 的实现'
date = 2023-12-01T20:24:26+08:00
draft = false
tags = ['Distributed System', 'Map Reduce']
+++


## 数据结构设计

状态都存在 Coordinator 中， Worker 无状态

### Task 结构
每次 Worker 向 Coordinator 请求 Task,
```go
type Task struct {
	Input         string // Map 阶段的输入文件名
	TaskPhase     Phase // Worker眼中的阶段： Map/Reduce/Exit/Wait
	NReduce       int   // Reduce 的数量
	Index         int   // 该 Task 在 Coordinator中的索引
	Intermediates []string  // Map生成的中间文件名
	Output        string // Reduce 生成的最终输出文件名
}
```
### Coordinator
```go

type Coordinator struct {
	TaskQueue     chan *Task // 待处理的 Task 队列
	TaskMetadatas map[int]*TaskMetadata // Coordinator 维护的 Task 元数据
	Phase         Phase // Coordinator 处于哪个阶段： Map/Reduce/Exit/Wait
	NReduce       int
	InputFiles    []string  // Map 阶段的输入文件名
	Intermediates [][]string 

	mutex sync.Mutex // 用于保护并发安全
}

type TaskMetadata struct {
	TaskStatus TaskStatus 	// Coordinator 关心的 task状态 Unassigned/Working/Finished
	StartTime  time.Time    // 用于防止 Worker 超时
	Task       *Task
}
```
## MapReduce 的执行流程

1. Coordinator 启动，并初始化，监听 Worker 的RPC请求。同时要启动一个goroutine 检查超时的任务，超时的任务要重新插入人物队列。
    * 两种RPC, 一个是Worker请求任务的RPC, 还有一个是Worker完成任务的通知的RPC
2. Worker 启动，向 Coordinator 请求任务，根据返回的 task 的 Phase (Map/Reduce/Exit/Wait) 分别进行下一步操作
    * Map 阶段：读取输入文件，调用用户提供的 Map 函数，生成中间文件，然后向 Coordinator 通知任务完成
    * Reduce 阶段：读取中间文件，调用用户提供的 Reduce 函数，生成最终输出文件，然后向 Coordinator 通知任务完成
    * Exit 阶段：直接 return
    * Wait 阶段：sleep 5 s
3. Coordinator 每次收到 Worker 完成任务的通知，除了进行状态的更新，都会检查是否所有任务都已经完成，如果是，就进入下一个阶段
4. 最终 Coordinator 到达 Exit Phase, 退出