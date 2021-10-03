# Gazebo Controllers Configuration

This repository explains systematic steps for setting Gazebo controllers config file to properly start spawned robots controllers. This content completes the previous two repos [rrr_arm]() and [rrr_arm_config](). The first shows a simple arm built from primitive geometry and interfaced wih Gazebo. The second explains how to use MoveIt assistant with the simple arm.

I noticed an issue with MoveIt's setup assistant that is the auto generated controllers config files do not work directly as expected. This results in starting Gazebo with arms fallen down. I will explain the systematic steps to get a join controlled from writing the joint model to launching its controller. 

1. The joint definition starts at the URDF file. Let's assume a joint of type `revolute`. The definition of the joint should be similar to the following

```
<joint name="right_arm_shoulder_pan_joint" type="revolute">
    <parent link="right_arm_shoulder" />
    <child link="right_arm_shoulder_pan_link" />
    <origin xyz="0 0.025 0" rpy="0 0 0" />
    <limit lower="-1.57" upper="1.57" effort="10" velocity="3" />
    <axis xyz="0 0 1" />
</joint>
```

2. Define a transmission with the name of the joint in the URDF 
```
<transmission name="tran1">
    <type>transmission_interface/SimpleTransmission</type>
    <joint name="right_arm_shoulder_pan_joint">
        <hardwareInterface>EffortJointInterface</hardwareInterface>
    </joint>
    <actuator name="motor1">
        <hardwareInterface>EffortJointInterface</hardwareInterface>
        <mechanicalReduction>1</mechanicalReduction>
    </actuator>
</transmission>
```

3. Write the `control.yaml` file. There is a convention to use the `joint_name`_position_controller. For example: right_arm_shoulder_pan_joint_position_controller.

```
# Publish all joint states
joint_state_controller:
  type: joint_state_controller/JointStateController
  publish_rate: 50  
  
# Position Controllers
right_arm_shoulder_pan_joint_position_controller:
  type: effort_controllers/JointPositionController
  joint: right_arm_shoulder_pan_joint
  pid: {p: 100.0, i: 0.01, d: 10.0}
```
**Notice** the above control config is **without a namespace**, which will eliminate a lot of confusion specially if multi-arm system is to be controlled. The confusion with namespace is because it needs to be set in the `control.yaml`, passed as an argument in the `controller spawner node`, and the robot state publisher joint  should remap its default `/joint_states` to `/namespace/joint_states`. **It is recommended to avoid namespaces and make use of expressive joint names that show the arm/robot to which the joint belongs. **

**Notice** it is normal that `control.yaml` file gets huge as we control multi-arm systems or arms with multi-finger hands. Be careful to list all the controllers as described above. A single typo in the `control.yaml` is enough not to get any joint controller launched. 

4. Write the controller spawner launch file 

```
<?xml version="1.0"?>
<launch>
    <!-- Load joint controller configurations from YAML file to parameter server -->
    <rosparam file="$(find robot_pkg)/config/control.yaml" command="load" />
    <!-- load the controllers -->
    <node name="controller_spawner" pkg="controller_manager" type="spawner" respawn="false" output="screen" args="right_arm_shoulder_pan_joint_position_controller" />
</launch>
```

**Notice** all the controllers should follow in the above `controller_spawner` node args like  `args="right_arm_shoulder_pan_joint_position_controller left_shoulder_pan_joint_position_controller "` and so on ...

### Common errors with controllers setting: 
Error: 
```
Controller Spawner: Waiting for service controller_manager/load_controller
[WARN] [1629886820.301987, 290.848000]: Controller Spawner couldn't find the expected controller_manager ROS interface.
```
Solution:
Install pre-requisite packages of your ROS distro
```
sudo apt install ros-$ROS_DISTRO-ros-control*

```
This command installs `ros-control` and `ros-controllers` packages. If the previous error persists and you are using namespaces in the control.yaml file, double check the points where the namespace should be included (point 3 above). 

Error: 
```
Could not switch controllers due to resource conflict.
```
Solution:
The reason is that the same joint is assigned to different controllers. Make sure there is a single controller per joint. 

### References: 
1. [ROS Control documentation](http://wiki.ros.org/ros_control).
2. [ROS Control Tutorials from the Construct](https://www.theconstructsim.com/robotigniteacademy_learnros/ros-courses-library/ros-control-101/).