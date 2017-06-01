# joystickでturtleを操作

* joystick(PS3コントローラ)で[turtlesim](http://wiki.ros.org/turtlesim)を動かす。
* [WritingTeleopNode](http://wiki.ros.org/joy/Tutorials/WritingTeleopNode)の日本語訳及び、indigoバージョン

<img src ="learning_joy/output.gif" width="240">


## 1.1 パッケージ作成
```bash
$ catkin_create_pkg learning_joy roscpp geometry_msgs joy
```

##1.2 簡単なNode記述
###1.2.1 コード
learning_joy/src内にturtle_teleop_joy.cppを作成し、以下のコードを貼り付ける。
```cpp
#include <ros/ros.h>
#include <geometry_msgs/Twist.h>
#include <sensor_msgs/Joy.h>


class TeleopTurtle
{
public:
  TeleopTurtle();

private:
  void joyCallback(const sensor_msgs::Joy::ConstPtr& joy);
  
  ros::NodeHandle nh_;

  int linear_, angular_;
  double l_scale_, a_scale_;
  ros::Publisher vel_pub_;
  ros::Subscriber joy_sub_;
  
};


TeleopTurtle::TeleopTurtle():
  linear_(1),//linear_に1を代入
  angular_(2)//angular_に2を代入
{

  nh_.param("axis_linear", linear_, linear_);
  nh_.param("axis_angular", angular_, angular_);
  nh_.param("scale_angular", a_scale_, a_scale_);
  nh_.param("scale_linear", l_scale_, l_scale_);

  //turtle1に指令を出すためのパブリッシャ
  vel_pub_ = nh_.advertise<geometry_msgs::Twist>("turtle1/cmd_vel", 1);

  //joysticからデータを取得するためのサブスクライバ
  joy_sub_ = nh_.subscribe<sensor_msgs::Joy>("joy", 10, &TeleopTurtle::joyCallback, this);

}

void TeleopTurtle::joyCallback(const sensor_msgs::Joy::ConstPtr& joy)
{
  geometry_msgs::Twist twist;
  twist.angular.z = a_scale_*joy->axes[angular_];//角度
  twist.linear.x = l_scale_*joy->axes[linear_];//距離
  vel_pub_.publish(twist);
}


int main(int argc, char** argv)
{
  ros::init(argc, argv, "teleop_turtle");
  TeleopTurtle teleop_turtle;

  ros::spin();
}

```

###1.2.2 コードの説明
```cpp
#include <ros/ros.h>
#include <geometry_msgs/Twist.h>
#include <sensor_msgs/Joy.h>
```
`geometry_msgs/Twist.h`はturtleに指令を出すためのmsgを持っている。
`sensor_msgs/Joy.h`はjoysticのデータを取得するためのmsgを持っている。

```cpp
class TeleopTurtle
{
public:
  TeleopTurtle();

private:
  void joyCallback(const sensor_msgs::Joy::ConstPtr& joy);
  
  ros::NodeHandle nh_;

  int linear_, angular_;
  double l_scale_, a_scale_;
  ros::Publisher vel_pub_;
  ros::Subscriber joy_sub_;
  
};
```
ここでは、TeleopTurtleクラスを作成し、joy msgを受け取るためのコールバック関数を定義している。また、ノードハンドル、パブリッシャ、サブスクライバを作成している。

```cpp
TeleopTurtle::TeleopTurtle():
  linear_(1),//linear_に1を代入
  angular_(2)//angular_に2を代入
{

  nh_.param("axis_linear", linear_, linear_);
  nh_.param("axis_angular", angular_, angular_);
  nh_.param("scale_angular", a_scale_, a_scale_);
  nh_.param("scale_linear", l_scale_, l_scale_);
```
ここでは、パラメータの初期化を行っている。

```cpp
  //turtle1に指令を出すためのパブリッシャ
  vel_pub_ = nh_.advertise<geometry_msgs::Twist>("turtle1/cmd_vel", 1);

  //joysticからデータを取得するためのサブスクライバ
  joy_sub_ = nh_.subscribe<sensor_msgs::Joy>("joy", 10, &TeleopTurtle::joyCallback, this);
```
ここでは、`turtle1/cmd_vel`としてパブリッシュするパブリッシャと、`joy`をサブスクライブするサブスクライバを定義している。

```cpp
void TeleopTurtle::joyCallback(const sensor_msgs::Joy::ConstPtr& joy)
{
  geometry_msgs::Twist twist;
  twist.angular.z = a_scale_*joy->axes[angular_];//角度
  twist.linear.x = l_scale_*joy->axes[linear_];//距離
  vel_pub_.publish(twist);
}
```
ここでは、joysticからのデータを取得し、それをスケーリングしたのち、turtleにパブリッシュしている。

##1.3 コンパイル
CMakeLists.txtの一番下に以下を追加する。
```bash
add_executable(turtle_teleop_joy src/turtle_teleop_joy.cpp)
target_link_libraries(turtle_teleop_joy ${catkin_LIBRARIES})
```
追加できたら、catkin_wsで`$ catkin_make`する。

##1.4 Joysticのセッティング
※コントローラの接続ができていれば、とばして良い。

[ros_joy_tutorial](https://github.com/lancer-evolution/ros_joy_tutorial)よりコントローラとPCとの接続を行う。

##1.5 接続確認
※トピックの確認だけなのでとばして良い。

joyのパッケージを実行してjoysticのデータがトピックとしてパブリッシュされているかを確認するために以下のコマンドを実行する。
```bash
$ rostopic echo joy
```

##1.6 launchファイルの記述
```bash
<launch>

 <!-- Turtlesim Node-->
  <node pkg="turtlesim" type="turtlesim_node" name="sim"/>


 <!-- joy node -->
  <node respawn="true" pkg="joy"
        type="joy_node" name="turtle_joy" >
    <param name="dev" type="string" value="/dev/input/js0" />
    <param name="deadzone" value="0.12" />
  </node>

 <!-- Axes -->
  <param name="axis_linear" value="1" type="int"/>
  <param name="axis_angular" value="0" type="int"/>
  <param name="scale_linear" value="2" type="double"/>
  <param name="scale_angular" value="2" type="double"/>

  <node pkg="learning_joy" type="turtle_teleop_joy" name="teleop"/>

</launch>
```
`turtlesim`でturtleを生成、`joy`でjoysticのノード生成、`learning_joy`でturtleとjoysticをつなげる。
