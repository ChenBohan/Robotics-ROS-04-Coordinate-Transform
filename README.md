# Robotics-ROS-04-Coordinate-Transform
ROS course on Shenlanxueyuan.com

More:

[Setting up your robot using tf](http://wiki.ros.org/navigation/Tutorials/RobotSetup/TF)

[ROS探索总结（二十二）——设置机器人的tf变换](http://www.guyuehome.com/355?replytocom=47816)

## TF

<img src="https://github.com/ChenBohan/Robotics-ROS-04-Coordinate-Transform/blob/master/readme_img/TF.png" width = "80%" height = "80%" div align=center />

### TF broadacaster

<img src="https://github.com/ChenBohan/Robotics-ROS-04-Coordinate-Transform/blob/master/readme_img/broadacaster.png" width = "80%" height = "80%" div align=center />

```cpp
void poseCallback(const turtlesim::PoseConstPtr& msg)
{
    // tf广播器
    static tf::TransformBroadcaster br;

    // 根据乌龟当前的位姿，设置相对于世界坐标系的坐标变换
    tf::Transform transform;
    transform.setOrigin( tf::Vector3(msg->x, msg->y, 0.0) );
    tf::Quaternion q;
    q.setRPY(0, 0, msg->theta);
    transform.setRotation(q);

    // 发布坐标变换
    br.sendTransform(tf::StampedTransform(transform, ros::Time::now(), "world", turtle_name));
}
```

- The ``Quaternion`` implements quaternion to perform linear algebra rotations in combination with Matrix3x3, Vector3 and Transform.

- Set the quaternion using fixed axis RPY.
```cpp
void setRPY (const tfScalar &roll, const tfScalar &pitch, const tfScalar &yaw)
```

- StampedTransform
```
tf::StampedTransform::StampedTransform	(	const tf::Transform & 	input,
const ros::Time & 	timestamp,
const std::string & 	frame_id,
const std::string & 	child_frame_id 
)	
```
Sending a transform with a TransformBroadcaster requires five arguments. 
1. First, we pass in the rotation transform, which is specified by a btQuaternion for any rotation that needs to occur between the two coordinate frames. 
2. Second, a btVector3 for any translation that we'd like to apply. 
3. Third, we need to give the transform being published a timestamp, we'll just stamp it with ros::Time::now(). 
4. Fourth, we need to pass the name of the parent node of the link we're creating.
5. Fifth, we need to pass the name of the child node of the link we're creating, in this case "base_laser."

### TF listener

<img src="https://github.com/ChenBohan/Robotics-ROS-04-Coordinate-Transform/blob/master/readme_img/listener.png" width = "80%" height = "80%" div align=center />

```cpp
    // 通过服务调用，产生第二只乌龟turtle2
    ros::service::waitForService("spawn");
    ros::ServiceClient add_turtle =
    node.serviceClient<turtlesim::Spawn>("spawn");
    turtlesim::Spawn srv;
    add_turtle.call(srv);
    
    // 定义turtle2的速度控制发布器
    ros::Publisher turtle_vel = node.advertise<geometry_msgs::Twist>("turtle2/cmd_vel", 10);
    
    // tf监听器
    tf::TransformListener listener;
```

```cpp
    while (node.ok())
    {
        tf::StampedTransform transform;
        try
        {
            // 查找turtle2与turtle1的坐标变换
            listener.waitForTransform("/turtle2", "/turtle1", ros::Time(0), ros::Duration(3.0));
            listener.lookupTransform("/turtle2", "/turtle1", ros::Time(0), transform);
        }
        catch (tf::TransformException &ex) 
        {
            ROS_ERROR("%s",ex.what());
            ros::Duration(1.0).sleep();
            continue;
        }

        // 根据turtle1和turtle2之间的坐标变换，计算turtle2需要运动的线速度和角速度
        // 并发布速度控制指令，使turtle2向turtle1移动
        geometry_msgs::Twist vel_msg;
        vel_msg.angular.z = 4.0 * atan2(transform.getOrigin().y(),
                                        transform.getOrigin().x());
        vel_msg.linear.x = 0.5 * sqrt(pow(transform.getOrigin().x(), 2) +
                                      pow(transform.getOrigin().y(), 2));
        turtle_vel.publish(vel_msg);

        rate.sleep();
    }
```

```cpp
bool waitForTransform (const std::string &target_frame, const std::string &source_frame, const ros::Time &time, const ros::Duration &timeout, const ros::Duration &polling_sleep_duration=ros::Duration(0.01), std::string *error_msg=NULL) const
``` 	
Block until a transform is possible or it times out. 

```cpp
void lookupTransform (const std::string &target_frame, const ros::Time &target_time, const std::string &source_frame, const ros::Time &source_time, const std::string &fixed_frame, StampedTransform &transform) const
``` 	
Get the transform between two frames by frame ID assuming fixed frame. 


### CMakelist

<img src="https://github.com/ChenBohan/Robotics-ROS-04-Coordinate-Transform/blob/master/readme_img/cmakelist.png" width = "80%" height = "80%" div align=center />


### Launch

```
 <launch>
    <!-- 海龟仿真器 -->
    <node pkg="turtlesim" type="turtlesim_node" name="sim"/>

    <!-- 键盘控制 -->
    <node pkg="turtlesim" type="turtle_teleop_key" name="teleop" output="screen"/>

    <!-- 两只海龟的tf广播 -->
    <node pkg="learning_tf" type="turtle_tf_broadcaster"
          args="/turtle1" name="turtle1_tf_broadcaster" />
    <node pkg="learning_tf" type="turtle_tf_broadcaster"
          args="/turtle2" name="turtle2_tf_broadcaster" />

    <!-- 监听tf广播，并且控制turtle2移动 -->
    <node pkg="learning_tf" type="turtle_tf_listener"
          name="listener" />
 </launch>
```



