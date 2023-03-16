# 流图执行

```go
func (r *Rule) Execute(inputMessage defs.NodeInputData) (err error) {
	nodeQueue := make(chan int, 10)
	nodeQueue <- 0
	var wg sync.WaitGroup
	for len(nodeQueue) != 0 {
		currentWaitNodes := len(nodeQueue)
		for i := 0; i < currentWaitNodes; i++ {
			wg.Add(1)
			node := <-nodeQueue
			go func() {
				defer func() {
					wg.Done()
				}()
				nodeRef := r.graph.nodeTable[node]
				response, err := nodeRef.ExecuteNodeAction(r.scriptEngine, inputMessage)
				if err != nil {
					response.Label = "Failed"
					response.Msg.MetaData = err.Error()
					trace.Error.Println("node execute failed,error:", err.Error(), "node type:", nodeRef.GetNodeType())
					return
				}
				set := mapset.NewThreadUnsafeSet()
				nextNodes := r.graph.table[node][response.Label]
				if nextNodes != nil {
					for _, nextNode := range nextNodes {
						set.Add(nextNode)
					}
				}
				for nextNodesNoRepeat := range set.Iterator().C {
					nodeQueue <- nextNodesNoRepeat.(int)
				}
			}()
		}
		wg.Wait()
	}
	return
}

```

