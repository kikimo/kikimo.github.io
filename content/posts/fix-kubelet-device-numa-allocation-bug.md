---
title: "修复一个 kubelet numa 节点设备分配的 bug"
date: 2021-05-22T14:28:17+08:00
draft: false
---

k8s 在 1.18 以后发布了 topology manager 功能，允许用户根据硬件设备的 numa 拓扑来分配资源。
topology manager 支持多种分配策略，其中一种叫 single numa node policy，
这种策略承诺分配给 pod 的硬件都属于同一个 numa 节点。
然而我们发现某些场景下，虽然选择的是 single numa node policy，
但是 kublet 分配的设备却可能跨越多个 numa 节点。
通过排查我们发现这个问题的根源在于 devicemanager 中设备分配的实现算法上，具体问题代码在`pkg/kubelet/cm/devicemanager/manager.go:filterByAffinity()`

```go
func (m *ManagerImpl) filterByAffinity(podUID, contName, resource string, available sets.String) (sets.String, sets.String, sets.String) {
	// If alignment information is not available, just pass the available list back.
	hint := m.topologyAffinityStore.GetAffinity(podUID, contName)
	if !m.deviceHasTopologyAlignment(resource) || hint.NUMANodeAffinity == nil {
		return sets.NewString(), sets.NewString(), available
	}

	// Build a map of NUMA Nodes to the devices associated with them. A
	// device may be associated to multiple NUMA nodes at the same time. If an
	// available device does not have any NUMA Nodes associated with it, add it
	// to a list of NUMA Nodes for the fake NUMANode -1.
	perNodeDevices := make(map[int]sets.String)
	nodeWithoutTopology := -1
	for d := range available {
		if m.allDevices[resource][d].Topology == nil || len(m.allDevices[resource][d].Topology.Nodes) == 0 {
			if _, ok := perNodeDevices[nodeWithoutTopology]; !ok {
				perNodeDevices[nodeWithoutTopology] = sets.NewString()
			}
			perNodeDevices[nodeWithoutTopology].Insert(d)
			continue
		}

		for _, node := range m.allDevices[resource][d].Topology.Nodes {
			if _, ok := perNodeDevices[int(node.ID)]; !ok {
				perNodeDevices[int(node.ID)] = sets.NewString()
			}
			perNodeDevices[int(node.ID)].Insert(d)
		}
	}

	// Get a flat list of all of the nodes associated with available devices.
	var nodes []int
	for node := range perNodeDevices {
		nodes = append(nodes, node)
	}

	// Sort the list of nodes by how many devices they contain.
	sort.Slice(nodes, func(i, j int) bool {
		return perNodeDevices[i].Len() < perNodeDevices[j].Len()
	})

	// Generate three sorted lists of devices. Devices in the first list come
	// from valid NUMA Nodes contained in the affinity mask. Devices in the
	// second list come from valid NUMA Nodes not in the affinity mask. Devices
	// in the third list come from devices with no NUMA Node association (i.e.
	// those mapped to the fake NUMA Node -1). Because we loop through the
	// sorted list of NUMA nodes in order, within each list, devices are sorted
	// by their connection to NUMA Nodes with more devices on them.
	var fromAffinity []string
	var notFromAffinity []string
	var withoutTopology []string
	for d := range available {
		// Since the same device may be associated with multiple NUMA Nodes. We
		// need to be careful not to add each device to multiple lists. The
		// logic below ensures this by breaking after the first NUMA node that
		// has the device is encountered.
		for _, n := range nodes {
			if perNodeDevices[n].Has(d) {
				if n == nodeWithoutTopology {
					withoutTopology = append(withoutTopology, d)
				} else if hint.NUMANodeAffinity.IsSet(n) {
					fromAffinity = append(fromAffinity, d)
				} else {
					notFromAffinity = append(notFromAffinity, d)
				}
				break
			}
		}
	}

	// Return all three lists containing the full set of devices across them.
	return sets.NewString(fromAffinity...), sets.NewString(notFromAffinity...), sets.NewString(withoutTopology...)
}
```

这个函把当前的可用设备按 numa 亲和性、非 numa 亲和性、无 numa 拓扑结构来划分。
它存在的问题是，当一个设备属于多个 numa 节点时（没错，一个物理设备是可以同时属于多个 numa 节点的），
它会把 numa 亲和性的设备划分到非 numa 亲和性的集合里。
例如，设备 dev1 同时属于 nuam1 和 numa2 节点，设备 dev2 属于 numa1 节点，假设我们现在要求选择 numa2 节点上的设备，
这时候如果`for _, n := range nodes`这条迭代语句中`nodes`也就是 numa 节点列表的顺序是`[numa1, numa2]`，
那么根据后面的迭代语句中的逻辑，dev1 就会被归类到非亲和性的设备集合里。
那为什么`nodes`中 numa 节点的顺序是怎么确定的呢？
我们看`nodes`列表是怎么构造的：

```go
	// Get a flat list of all of the nodes associated with available devices.
	var nodes []int
	for node := range perNodeDevices {
		nodes = append(nodes, node)
	}

	// Sort the list of nodes by how many devices they contain.
	sort.Slice(nodes, func(i, j int) bool {
		return perNodeDevices[i].Len() < perNodeDevices[j].Len()
	})
```

先从字典`perNodeDevices`中构造`nodes`列表，注意因为`perNodeDevices`是字典，迭代的顺序无法保证，
所以即便是同样的内容也可能产生不一样的`nodes`列表。
然后对`nodes`进行排序，但是排序的 comparator 显然写错了，
`return perNodeDevices[i].Len() < perNodeDevices[j].Len()`明显应该是`return perNodeDevices[nodes[i]].Len() < perNodeDevices[nodes[j]].Len()`。
字典迭代结果的不确定性和排序 comparator 的错误显然让`nodes`的排序结果彻底变得不确定，这直接影响到后面设备的分类操作。
那我们按上面说的修复 comparator 的 bug 就可以了吗？
我们先试着理解这个 comparator 的逻辑，numa 节点按上面的连接的设备数量升序排列。
在我们所举的例子中，numa2 上关联一个设备，numa1 上关联两个设备，`nodes`排列顺序是`[numa1, numa2]`,
似乎没有问题。
但如果我们再考虑这样一个场景，numa2 上还有 dev3, dev4，这时候`nodes`的排列顺序又变成`[numa1, numa2]`，
dev2 又会被划到非亲和性设备集里。
所以只是简单的根据 numa 节点上关联的设备来排序不足以修复这个问题。
要修复这个问题的本质在于 numa 节点的的排序上：如果一个 numa 节点在`hint.NUMANodeAffinity`集合里，
那么他应该比不在这个集合里的 numa 节点排得靠前，具体的 fix 如下：

```go

	// Sort the list of nodes by:
	// 1) Nodes contained in the 'hint's affinity set
	// 2) Nodes not contained in the 'hint's affinity set
	// 3) The fake NUMANode of -1 (assuming it is included in the list)
	// Within each of the groups above, sort the nodes by how many devices they contain
	sort.Slice(nodes, func(i, j int) bool {
		// If one or the other of nodes[i] or nodes[j] is in the 'hint's affinity set
		if hint.NUMANodeAffinity.IsSet(nodes[i]) && hint.NUMANodeAffinity.IsSet(nodes[j]) {
			return perNodeDevices[nodes[i]].Len() < perNodeDevices[nodes[j]].Len()
		}
		if hint.NUMANodeAffinity.IsSet(nodes[i]) {
			return true
		}
		if hint.NUMANodeAffinity.IsSet(nodes[j]) {
			return false
		}

		// If one or the other of nodes[i] or nodes[j] is the fake NUMA node -1 (they can't both be)
		if nodes[i] == nodeWithoutTopology {
			return false
		}
		if nodes[j] == nodeWithoutTopology {
			return true
		}

		// Otherwise both nodes[i] and nodes[j] are real NUMA nodes that are not in the 'hint's' affinity list.
		return perNodeDevices[nodes[i]].Len() < perNodeDevices[nodes[j]].Len()
	})
```

目前这个 bug 的[相关 fix](https://github.com/kubernetes/kubernetes/pull/101893/files)已经提交给社区并 merge 到主干了。
