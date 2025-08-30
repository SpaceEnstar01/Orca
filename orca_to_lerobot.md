本文档说明如何将 Orca 机械臂成功集成到 LeRobot 框架中，实现通过 teleoperate.py 进行远程控制。
(so101 -> Orca Arm)

---
1. 文件结构
```markdown
 LeRobot 项目路径为：  

```

lerobot/
├── src/lerobot/
│   ├── robots/
│   │   ├── **init**.py
│   │   ├── utils.py
│   │   ├── orca/
│   │   │   ├── **init**.py
│   │   │   └── orca.py
│   ├── teleoperate.py

```

Orca SDK 位于独立目录：  

```

orca/
└── orca\_sdk/
├── interface/
│   └── interface.py
└── ...

```
```

---
2. 在 lerobot/robots/orca/__init__.py 注册 OrcaRobot
```python
from .orca import OrcaRobot
from lerobot.robots.config import RobotConfig

# 将 OrcaRobot 注册到 RobotConfig 的 ChoiceRegistry
RobotConfig.register_choice(OrcaRobot)
```

* 作用：让 RobotConfig 识别 orca 类型的机器人。
* 这样在 teleoperate.py 里就可以通过 --robot.type=orca 选择 Orca 机械臂。

---

3. OrcaRobot 配置类和实现
   在 lerobot/robots/orca/orca.py 中：

```python
from dataclasses import dataclass, field
from lerobot.robots.config import RobotConfig
from lerobot.robots.robot import Robot
from lerobot.robots.utils import ensure_safe_goal_position
from orca_sdk.interface.orca_interface_v2 import C_OrcaInterface_V2

@dataclass
class OrcaConfig(RobotConfig):
    port: str
    disable_torque_on_disconnect: bool = True
    max_relative_target: int | None = None
    use_degrees: bool = False
    cameras: dict[str, any] = field(default_factory=dict)

class OrcaRobot(Robot):
    config_class = OrcaConfig
    name = "orca"

    def __init__(self, config: OrcaConfig):
        super().__init__(config)
        self.config = config
        # 初始化 Orca SDK
        self.interface = C_OrcaInterface_V2(can_name=config.port)
        # 初始化其它状态/动作等
        ...
```

* OrcaRobot 继承自 Robot，实现必要的抽象方法：

  * action_features
  * observation_features
  * get_observation()
  * send_action()
  * is_connected()
  * calibrate() 等

---

4. 修改 lerobot/robots/utils.py 以支持 Orca
   在 `make_robot_from_config` 中添加：

```python
elif config.type == "orca":
    from .orca.orca import OrcaRobot
    return OrcaRobot(config)
```

* 作用：当 RobotConfig.type == "orca" 时，创建 OrcaRobot 实例。

---

5. teleoperate 使用示例
   成功集成后，可以通过 teleoperate.py 控制 Orca 机械臂：

```bash
python -m lerobot.teleoperate \
  --robot.type=orca \
  --robot.port=can0 \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM0 \
  --teleop.id=R00 \
  --display_data=true
```

* 注意：

  * --display_data=true 会弹出 Rerun 可视化界面。
  * 校准文件需要匹配 --robot.id 或指定 --robot.calibration_dir。

---

6. 完整配置流程总结
7. 注册机器人类型

   * 在 robots/orca/__init__.py 注册 OrcaRobot 到 RobotConfig。
8. 实现机器人类

   * OrcaRobot 继承自 Robot。
   * 实现抽象方法和动作/观测接口。
   * 使用 Orca SDK 初始化机械臂。
9. 支持配置创建

   * utils.py 中 make_robot_from_config 支持 orca。
10. 校准文件

    * 确保 RobotConfig.id 与已有校准文件匹配。
    * 可选设置 calibration_dir。
11. 运行 teleoperate

    * 设置 --robot.type=orca。
    * 设置 Orca CAN 端口。
    * 设置 teleop 类型及端口。

---

7. 注意事项

* 如果不需要 UI 可视化，使用：

  ```bash
  --display_data=false
  ```
* Orca SDK 的 C_OrcaInterface_V2 需要正确安装并可导入。
* 所有动作、关节控制和 gripper 控制通过 send_action() 接口传入字典。
* 若出现校准提示，可确认 id 与 calibration_dir 是否匹配。

---
