# <span style="color:#008a00;">MemCache Roadmap 2026 H1</span>

MemCache在2026年H1阶段的目标是增强在多级缓存方面的能力、持续提升集群可靠性和易用性，具体的研发Roadmap如下：

---

## <span style="color:#0066cc;">特性功能</span>		

[] [***330***] MemCache支持SSD扩展缓存容量，预计330提供POC测试版本


## <span style="color:#0066cc;">集群可靠性</span>
[x] [***2025H4***] 支持meta service主备切换，元数据备份


---

# <span style="color:#008a00;">分支节奏策略</span>		


## <span style="color:#9933ff;">master</span>		

[x] [***稳定演进***] 最新feature试用测试分支，通过内部smoke pipeline用例覆盖    

## <span style="color:#9933ff;">develop</span>		

[x] [***快速演进***] 最新feature开发分支，通过内部smoke pipeline用例后，会按照1周~半月的时间节奏同步到master    

## <span style="color:#9933ff;">稳定分支</span>		

[] [***v1.1.x***] 下一个迭代推出的稳定分支，预计于330推出，对应版本号1.1.x    
[x] [***v1.0.0***] 最新稳定分支，只会合入必要修改的bugfix，在必要bugfix合入之后，会推送到[pypi](https://pypi.org/project/memcache-hybrid/#history)，对应版本号 1.0.x，x为从0开始递增的数字    

---