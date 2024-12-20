---
tags: zookeeper
---

用一个例子来说明zk在分布式系统中组件的主备倒换中的作用

## 1. 环境准备

- 起一个zookeeper的docker容器，注意端口映射

compose文件：

```yaml
services:
  syf:
    image: zookeeper
    ports:
      - "2181:2181"
      - "2888:2888"
      - "3888:3888"
      - "8080:8080"
    stdin_open: true  # 等价于加上-i参数
    tty: true         # 等价于加上-t参数
```

```bash
# 后台启动容器
docker compose up -d
```

- 准备python虚拟环境，安装kazoo

```bash
python3 -m venv venv
source venv/bin/activate
pip install kazoo
```

## 2. 编写测试脚本

```python
import time
import threading
import random
from kazoo.client import KazooClient

ZK_HOST = "127.0.0.1:2181"  # ZooKeeper 地址
COMP_PATH = "/components"


class CPSClient:
    def __init__(self, component_name, monitor_type):
        self.zk = KazooClient(hosts=ZK_HOST)
        self.zk.start()
        self.component_name = component_name
        self.monitor_type = monitor_type  # "heartbeat" 或 "process"
        self.node_path = None  # 当前节点在 ZooKeeper 中的路径

        if not self.zk.get(COMP_PATH):
            self.zk.create(COMP_PATH)
            if not self.zk.get(COMP_PATH):
                raise Exception(f"Create {COMP_PATH} failed")

    def register_component(self):
        """注册组件，创建带序号的临时节点"""
        if not self.node_path:
            self.node_path = self.zk.create(
                f"/components/{self.component_name}-", ephemeral=True, sequence=True
            )

        print(f"{self.component_name} 已注册，路径：{self.node_path}")
        self.try_to_become_leader()

    def monitor_component(self):
        """根据监控方式选择监控实现"""
        if self.monitor_type == "heartbeat":
            self.start_heartbeat_monitor()
        elif self.monitor_type == "process":
            self.start_process_monitor()

    def start_heartbeat_monitor(self):
        """心跳监控模拟"""
        print(f"{self.component_name} 使用心跳监控...")
        while True:
            time.sleep(5)  # 模拟心跳间隔
            if not self.send_heartbeat():
                if self.node_path and self.zk.get(self.node_path):
                    print(f"{self.component_name} 进程失效，删除临时节点{self.node_path}")
                    self.zk.delete(self.node_path)  # 删除临时节点
                self.node_path = None
            else:
                if self.node_path:
                    # 心跳正常
                    continue
                else:
                    # 从失败中恢复
                    self.register_component()

    def start_process_monitor(self):
        """进程监控模拟"""
        print(f"{self.component_name} 使用进程监控...")
        while True:
            time.sleep(5)
            if not self.check_process():
                if self.node_path and self.zk.get(self.node_path):
                    print(f"{self.component_name} 进程失效，删除临时节点{self.node_path}")
                    self.zk.delete(self.node_path)
                self.node_path = None
            else:
                if self.node_path:
                    continue
                else:
                    print(f"Start to register again")
                    self.register_component()
                    self.try_to_become_leader()

    def send_heartbeat(self):
        """模拟发送心跳，随机失效"""
        # 模拟随机失效
        return random.random() > 0.3

    def check_process(self):
        """模拟进程监控"""
        # 假设进程不存在时返回 False
        return random.random() > 0.3

    def watch_for_leader_change(self):
        """监听 ZooKeeper 的主节点变化"""

        def on_node_deleted(event):
            print(f"{self.component_name} 监听到主节点失效，尝试接管")
            self.try_to_become_leader()

        children = self.zk.get_children("/components")
        if not children:
            raise Exception("All nodes have been dead")

        children.sort()  # 排序，最小序号的节点是主节点
        leader_path = f"/components/{children[0]}"
        print(f"当前主节点：{leader_path}")

        # 监听主节点的状态
        self.zk.exists(leader_path, watch=on_node_deleted)

    def try_to_become_leader(self):
        """尝试成为主节点"""
        children = self.zk.get_children("/components")
        if not children:
            raise Exception("No children available")
        orders = [x.split("-")[-1] for x in children]
        orders.sort()

        if self.node_path.split("-")[-1] == orders[0]:
            print(f"{self.component_name} 成为新的主节点，启动组件")
            self.start_component()
        else:
            print(f"{self.component_name} 成为备节点，等待主节点失效")

    def start_component(self):
        """启动组件逻辑"""
        print(f"{self.component_name} 启动")


# 启动模拟的主备节点
def run_component(name, monitor_type):
    client = CPSClient(name, monitor_type)
    client.register_component()
    client.monitor_component()


# 启动两个节点模拟
threading.Thread(target=run_component, args=("kafka1", "heartbeat")).start()
threading.Thread(target=run_component, args=("kafka2", "process")).start()
```

## 3. 启动脚本，观察输出


```bash
python3 zk_example.py
```

输出如下，可以看到环境中模拟的kafka组件可以正常选主、倒换

同时，可以注意到zookeeper的临时节点的序号在消亡后会持续保持递增

```bash
$ python3 zk_example.py 
kafka2 已注册，路径：/components/kafka2-0000000107
kafka1 已注册，路径：/components/kafka1-0000000108
kafka2 成为新的主节点，启动组件
kafka2 启动
kafka2 使用进程监控...
kafka1 成为备节点，等待主节点失效
kafka1 使用心跳监控...
kafka1 进程失效，删除临时节点/components/kafka1-0000000108
kafka2 进程失效，删除临时节点/components/kafka2-0000000107
Start to register again
kafka1 已注册，路径：/components/kafka1-0000000109
kafka2 已注册，路径：/components/kafka2-0000000110
kafka1 成为新的主节点，启动组件
kafka1 启动
kafka2 成为备节点，等待主节点失效
kafka2 成为备节点，等待主节点失效
kafka2 进程失效，删除临时节点/components/kafka2-0000000110
kafka1 进程失效，删除临时节点/components/kafka1-0000000109
Start to register again
kafka2 已注册，路径：/components/kafka2-0000000111
kafka2 成为新的主节点，启动组件
kafka2 启动
kafka1 已注册，路径：/components/kafka1-0000000112
kafka2 成为新的主节点，启动组件
kafka2 启动
kafka1 成为备节点，等待主节点失效
kafka1 进程失效，删除临时节点/components/kafka1-0000000112
kafka2 进程失效，删除临时节点/components/kafka2-0000000111
Start to register again
kafka2 已注册，路径：/components/kafka2-0000000113
kafka1 已注册，路径：/components/kafka1-0000000114
kafka2 成为新的主节点，启动组件
kafka2 启动
kafka1 成为备节点，等待主节点失效
kafka2 成为新的主节点，启动组件
kafka2 启动
kafka1 进程失效，删除临时节点/components/kafka1-0000000114
kafka2 进程失效，删除临时节点/components/kafka2-0000000113
kafka1 已注册，路径：/components/kafka1-0000000115
kafka1 成为新的主节点，启动组件
kafka1 启动
```
