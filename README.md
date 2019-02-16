# cartographer_using

---

## Reference
- [Cartographer Document][1]
- [ros中坐标系设置][2]


## 1 cartographer build and install
```
sudo apt-get install -y python-wstool python-rosdep ninja-build

mkdir -p cartographer/src
cd cartographer/src
git clone https://github.com/googlecartographer/cartographer.git
git clone https://github.com/googlecartographer/cartographer_ros.git
git clone https://github.com/ceres-solver/ceres-solver.git

cd cartographer
src/cartographer/scripts/install_proto3.sh
sudo rosdep init
rosdep update
rosdep install --from-paths src --ignore-src --rosdistro=${ROS_DISTRO} -y

catkin_make_isolated --install --use-ninja
```

安装方式：
```
alias sourcecartographer="source ~/code/cartographer/install_isolated/setup.zsh"# terminator中使用
# 或者
source ~/code/cartographer/install_isolated/setup.zsh #非terminator使用
```

使用方式：
先执行sourcecartographer
然后执行sourcerosws

## 2 问题记录
- terminator不支持绝对路径的获取，下面一句在terminator终端中无法正确执行
```
alias sourcerosws='source $(cd "$(dirname "$0")";pwd)/devel/setup.zsh'
```
- ros工作空间source的时候，如果使用相对路径会覆盖掉之前的ros path，使用绝对路径不会
- 如果是在terminator中使用，使用这种方式，并且首先sourcerosws，然后sourcecartographer
```
alias sourcerosws='source devel/setup.zsh'
alias sourcecartographer="source ~/code/cartographer/install_isolated/setup.zsh"
```
- 如果不是在terminator中使用，这样每次都是使用绝对路径source，不会覆盖
```
alias sourcerosws='source $(cd "$(dirname "$0")";pwd)/devel/setup.zsh'
source ~/code/cartographer/install_isolated/setup.zsh
```

- 每次在cartographer内部增删内容或者添加文件需要重新编译才会生效
```
catkin_make_isolated --install --use-ninja
```

## 3 cartographer doc记录
### 3.1 launch启动文件区别
- backpack_3d.launch 
利用robot_state_publisher加载机器人urdf文件，发布机器人传感器之间的tf树。
利用**cartographer_node和cartographer_occupancy_grid_node**两个节点进行online slam地图创建。
运行时需要提供.lua配置文件和.urdf机器人描述文件（描述机器人坐标位置关系）
**online slamming时候，cartographer不知道什么时候你要结束，所以你需要执行下列命令**
```
# Finish the first trajectory. No further data will be accepted on it.
rosservice call /finish_trajectory 0

# Ask Cartographer to serialize its current state.
# (press tab to quickly expand the parameter syntax)
rosservice call /write_state "{filename: '${HOME}/Downloads/b3-2016-04-05-14-14-00.bag.pbstream', include_unfinished_submaps: 'true'}"
```
然后cartographer会运行结束，并生成.pbstream地图文件。

- demo_backpack_3d.launch
包含了backpack_3d.launch，通过播放bag进行offline地图创建和rviz显示
需要提供.lua，.urdf，bag_filename。


- offline_backpack_3d.launch
和demo_backpack_3d.launch类似，不过建图执行的更快。
利用**cartographer_offline_node和cartographer_occupancy_grid_node**进行offline建图。
需要提供.lua，.urdf，bag_filenames，可以提供多个bag文件。
**数据播放结束会自动生成.pbstream地图文件。**

- demo_backpack_3d_localization.launch 
利用**cartographer_node和cartographer_occupancy_grid_node**两个节点。
是offline工作模式，需要提供.lua，.urdf，bag_filename，load_state_filename（.pbstream文件），然后会在这个地图的基础上进行定位（localization）

- assets_writer_backpack_3d.launch
利用**cartographer_assets_writer**从.pstream文件中提取数据。
需要提供.lua，.urdf，bag_filenames， pose_graph_filename（.pbstream文件）。
**知道如何配置，如何使用生成pcd地图文件**

>对pbstream文件的认识：是一个压缩的protobuf文件，包含cartographer建图过程中使用的数据结构。为了实时有效地运行，Cartographer立即抛出大部分传感器数据，只能使用其输入的一小部分，内部使用的映射（并保存在.pbstream文件中）非常粗糙。然而，当算法完成并且建立了最佳轨迹时，可以将其与完整传感器数据后验地重新组合以创建高分辨率图。
地图创建结束后，可以利用cartographer_assets_writer将pbstream文件和bag文件共同加工出其他格式的真正的高质量的地图文件，比如pcd或者ply文件。

### 3.2 use pbstream to write pcd map
assets_writer_pcd.lua配置文件如下
```
include "transform.lua"

options = {
    tracking_frame = "base_link",
    pipeline = {
        {
            action = "min_max_range_filter",
            min_range = 1.,
            max_range = 20.,
        },
        {
            action = "dump_num_points",
        },
        -- downsample
        {
            action = "fixed_ratio_sampler",
            sampling_ratio = 0.01,
        },
        -- write to pcd file
        {
            action = "write_pcd",
            filename = "mymap.pcd";
        },
    }
}

return options
```

设置的关键点：

- min_max_range_filter保留点云的范围
- sampling_ratio = 0.01 可以通过这个降采样来有效减小pcd大小

生成的pcd过大（eg: 我的是9G）的时候，pcl_viewer将无法加载，猜测是内存限制的原因。通过参数设置减小pcd大小后，pcl_viewer可以顺利打开地图文件。

指令执行结束终端会有如下红色信息提示，这并不是执行失败了，而是node执行结束了。
```
================================================================================
REQUIRED process [cartographer_assets_writer-2] has died!
process has finished cleanly
log file: /home/nrsl/.ros/log/8ebaaa64-317c-11e9-8d1a-5800e3e94927/cartographer_assets_writer-2*.log
Initiating shutdown!
================================================================================
```


  [1]: https://google-cartographer-ros.readthedocs.io/en/latest/algo_walkthrough.html
  [2]: https://community.bwbot.org/topic/227/ros%E5%9D%90%E6%A0%87%E7%B3%BB%E7%BB%9F-%E5%B8%B8%E8%A7%81%E7%9A%84%E5%9D%90%E6%A0%87%E7%B3%BB%E5%92%8C%E5%85%B6%E5%90%AB%E4%B9%89
