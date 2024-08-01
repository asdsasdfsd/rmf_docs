# 如何搭建和部署Mir相关的rmf集群

##如何搭建一个本地的rmf集群

### 1.创建虚拟机(Ubantu版本是22.04)
我使用的是`Vagrant`的虚拟机管理工具和`VirtualBox`的虚拟机构建工具

1.1创建一个文件夹,比如`my-vagrant-ubuntu2204`,进入文件夹

1.2在命令行使用`vagrant init`初始化`Vagrant`。这个命令会创建一个默认的`Vagrantfile`,`Vagrant`会根据这个`Vagrantfile`中定义的配置来构建虚拟机，需要下载文档[Vagrantfile](download/Vagrantfile)文件来替换,同时要下载脚本文件[bootstrap.sh](download/bootstrap.sh) ,这个文件应该是为了登录桌面版ubantu时可以使用密码

1.3用vagrant up启动虚拟机和vagrant ssh链接虚拟机

### 2.按照ros-humble-desktop

1.1安装Ubantu桌面版
```bash
sudo apt update
sudo apt install ubuntu-desktop -y
sudo reboot
```
中间出现的package configuration直接TAB，然后选OK就可以了

安装完以后可以用VBox来登录桌面版，密码是123456wk
1.2安装ROS2桌面版

调整本地的编码和语言
```bash
locale  # check for UTF-8

sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

locale  # verify settings
```
最后一行locale会在命令行输出下面的消息
```bash
vagrant@ubuntu2204-rmf:~$ locale
LANG=en_US.UTF-8
LANGUAGE=
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC="en_US.UTF-8"
LC_TIME="en_US.UTF-8"
LC_COLLATE="en_US.UTF-8"
LC_MONETARY="en_US.UTF-8"
LC_MESSAGES="en_US.UTF-8"
LC_PAPER="en_US.UTF-8"
LC_NAME="en_US.UTF-8"
LC_ADDRESS="en_US.UTF-8"
LC_TELEPHONE="en_US.UTF-8"
LC_MEASUREMENT="en_US.UTF-8"
LC_IDENTIFICATION="en_US.UTF-8"
LC_ALL=
```
安装一些可能需要的依赖
```bash
sudo apt install software-properties-common
sudo add-apt-repository universe
```
安装ROS仓库密钥并且将包地址添加到apt源对应的文件夹
```bash
sudo apt update && sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```
安装ROS2桌面版
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install ros-humble-desktop -y
```

### 3.安装openrmf需要的相关仓库
将ros2相关的包在每次启动命令行工具的时候都自动安装到环境变量里面
```bash
source /opt/ros/humble/setup.bash
echo "source /opt/ros/humble/setup.bash">>~/.bashrc
```
Install all non-ROS dependencies of Open-RMF packages
```bash
sudo apt update && sudo apt install ros-dev-tools -y
```
(rosdep helps install dependencies for ROS packages across various distros and will be installed along with ros-dev-tools. However, it is important to update it.)
```bash
sudo rosdep init # run if first time using rosdep.
rosdep update
```
安装gazebo相关的包到apt源
```bash
sudo sh -c 'echo "deb http://packages.osrfoundation.org/gazebo/ubuntu-stable `lsb_release -cs` main" > /etc/apt/sources.list.d/gazebo-stable.list'
wget https://packages.osrfoundation.org/gazebo.key -O - | sudo apt-key add -
```
安装python相关的包
```bash
sudo apt update
sudo apt install python3-pip -y
python3 -m pip install flask-socketio fastapi uvicorn
```
Update colcon mixin if you have not done so previously.(colcon是构建ros2工作空间的工具)
```bash
colcon mixin add default https://raw.githubusercontent.com/colcon/colcon-mixin-repository/master/index.yaml
colcon mixin update default
```
创建ROS2对应的工作空间，[multistop.repos](download/multistop.repos)下载
```bash
cd Documents
mkdir -p rmf_ws/src
cd rmf_ws

nano multistop.repos #复制粘贴，写入multistop中的内容
vcs import src < multistop.repos
```
安装依赖项
```bash 
cd rmf_ws
rosdep install --from-paths src --ignore-src --rosdistro humble -y --skip-keys=gz_sim_vendor
```
安装和更新编辑器配置
```bash
sudo apt update
sudo apt install clang clang-tools lldb lld libstdc++-12-dev -y
```
构建`open-rmf`
```bash
export CXX=clang++
export CC=clang
colcon build --mixin release lld
```
将`openrmf`的二进制文件加入终端的启动脚本里面
```bash
echo "source ~/Documents/rmf_ws/install/setup.bash">>~/.bashrc
source ~/.bashrc
```
安装`dds`通信的工具
```bash
sudo apt update
sudo apt install ros-humble-rmw-cyclonedds-cpp
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```
运行`openrmf`的`launch`文件来检查是否成功编译(这一步要在`ubantu`桌面版里面进行)
```bash
ros2 launch rmf_demos_gz office.launch.xml
```
### 4.安装rmf_web相关仓库
安装nvm
```bash
sudo apt update && sudo apt install curl
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.37.2/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
nvm install 20
```
安装`pnpm`和`nodejs`
```bash
curl -fsSL https://get.pnpm.io/install.sh | bash -
source /home/vagrant/.bashrc
pnpm env use --global 20
```
安装`pipenv`就`python3-venv`
```bash
pip3 install pipenv
sudo apt install python3-venv
```
下载rmf_web的仓库,切换到humble分支
```bash
cd ~/Documents
git clone https://github.com/open-rmf/rmf-web.git
cd rmf-web
git checkout -b humble origin/humble
```
安装rmf_web用到的依赖
```bash
pnpm install
```
验证是否成功安装包和依赖(在虚拟机的浏览器中看看是否可以进入localhost:3000发布任务)
```bash
cd packages/dashboard
pnpm start
ros2 launch rmf_demos_gz office.launch.xml #这个要在ubuntu桌面版运行
```

### 5.下载rmf-adapter-mir相关的文件（详细解析在导航栏rmf页中）
下载仓库并编译
```bash
# Create the MiR workspace and clone the repository
mkdir -p mir_ws/src
cd mir_ws/src
git clone https://github.com/osrf/fleet_adapter_mir

# Source the Open-RMF underlay workspace and build
cd ~/Documents/mir_ws/mir_ws
colcon build
#如果遇到问题就用pip install setuptools==67.0.0降低setuptools版本，并且删除fleet_adapter_mirfm和mir_fleet_client文件夹(这个也用安装需要的python包解决)
source ~/Documents/rmf_ws/mir_ws/install/setup.bash
echo "source ~/Documents/rmf_ws/mir_ws/install/setup.bash" >> ~/.bashrc
```
这个要使用要改一些参数

configs/mir_config.yaml里面要将`base_url`对应的值改为 `'http://mir.com/api/v2.0.0/'`
`password`对应的值要改为mir机器人的api_token `'因为这个比较私密最好到机器人里面看一些'`
configs/nav_graph.yaml里面的文件要替换为[nav_graph.yaml]()

mir机器人里面要创建两个`mission`,分别是`dock_to_charger`和`rmf_default_move_mission`。其中`rmf_default_move_mission`任务中要加一个`action`,`action`三个参数的名字分别是'x','y','yaw'

### 6.我自己写了一个launch文件(这个文件是为了不运行rmf-adapter-mir对应的包以及使用办公室的虚拟环境）
下载我的launch.xml文件
```bash
cd Documents
git clone https://github.com/asdsasdfsd/launch.git
cd launch
colcon build
```
运行launch文件
```bash
source ~/Documents/launch/install/setup.bash
echo "source ~/Documents/launch/install/setup.bash" >> ~/.bashrc
ros2 launch test office.launch.xml #这个要在ubantu桌面版里面运行
```
### 7.启动所有需要的文件
```bash
ros2 launch test office.launch.xml
ros2 run fleet_adapter_mir fleet_adapter_mir -c /home/vagrant/Documents/rmf_ws/mir_ws/src/fleet_adapter_mir/configs/mir_config.yaml -n /home/vagrant/Documents/launch/src/test/configs/0.yaml 
#后面的参数是两个配置文件
cd packages/dashboard
pnpm start
```
