[![Line Coverage](https://img.shields.io/badge/Line%20Coverage-90.1%25-darkgreen.svg)](./code_coverage_report.info) [![Function Coverage](https://img.shields.io/badge/Function%20Coverage-97.0%25-darkgreen.svg)](./code_coverage_report.info)

# Vector Pursuit Controller

This [ROS2 Humble](https://docs.ros.org/en/humble/index.html) package contains a plugin for the [Nav2 Controller Server](https://docs.nav2.org/configuration/packages/configuring-controller-server.html) that implements the [Vector Pursuit](https://apps.dtic.mil/sti/pdfs/ADA468928.pdf) path tracking algorithm. It leverages [Screw Theory](https://en.wikipedia.org/wiki/Screw_theory) to achieve **accurate path tracking** and comes with **active collision detection**. The controller has a **very low computational overhead** and is very easy and **simple to deploy**. It tracks path orientation in a geometrically-meaningful way making it an ideal replacement for the [Pure Pursuit Algorithm](https://in.mathworks.com/help/nav/ug/pure-pursuit-controller.html) in scenarios where path following accuracy is vital. It consumes 15% of a single core on an ARM cortex-A72 CPU @ 1.8GHz and is designed to track paths at speeds upto 4.5m/s. 

https://github.com/user-attachments/assets/5a660fa0-054c-4b14-aecd-d9cfe471930b

## Get Started
These are minimal, to-the-point instructions for experienced ROS2/Nav2 users. Beginners are recommended to read the [Quickstart Tutorial](#quickstart-tutorial) for a simple 5-step example to try out the controller in a Gazebo simulation.

1. Clone the repo into a ROS2 colcon workspace.
    ```bash
    git clone https://github.com/blackcoffeerobotics/vector_pursuit_controller.git
    ```

2. Edit the `controller_server` parameters of the Nav2 stack to include the vector pursuit plugin along with its [default configuration](#default-parameters). Nav2's controller server supports multiple controller plugins at the same time and instructions for setting it up can be found in the [official docs](https://docs.nav2.org/configuration/packages/configuring-controller-server.html).

3. Build the workspace and source it, then run the nav2 stack/controller server.

## Features Offered
These are additional features on top of the core Vector Pursuit algorithm that extend its functionality.
| Feature | Description |
|---------|-------------|
| **Adaptive Lookahead Distance** | The lookahead distance is used to find the target pose on the path which is used to guide the robot along the path. This distance can be scaled as per the robot's velocity to ensure that the robot aims further along the path when moving at greater velocity. |
| **Collision Aware Linear Velocity** | The robot's linear velocity is automatically scaled preemptively when in proximity to obstacles and completely halted when a collision is imminent. |
| **Approach Aware Linear Velocity** | The robots linear velocity is automatically scaled when nearing a goal pose, this prevents overshoot. |
| **On Point Rotation** | The controller will first rotate on point when attempting to chase a target pose at a heading which is more than the robots current heading by a configurable angle. |
| **Optional Reversing** | The controller will output reverse velocity if the lookahead point is behind the robot (x coordinate of the lookahead point in the robot's base_link is -ve). |

## Configuration

### Core Parameters
The following parameters tune the core path-tracking algorithm and are not needed by the [additional features](#features-offered).

| Parameter                          | Description                                                     |
|------------------------------------|-----------------------------------------------------------------|
| `p_gain`                           | Proportional constant for controller.                            |
| `desired_linear_vel`               | Maximum desired linear velocity.                                 |
| `min_linear_velocity`              | Minimum velocity achievable.                                     |
| `min_turning_radius`               | Minimum turning radius. The `min_turning_radius` of the controller should be equal to or less than the minimum turning radius of the planner (in case it is available). This ensures the controller can follow the path generated by the planner and not get stuck in a loop. |
| `max_lateral_accel`                | Maximum lateral acceleration.                                    |
| `max_linear_accel`                 | Maximum linear acceleration.                                     |
| `use_interpolation`                | Calculate lookahead point exactly at the lookahead distance.    |
| `use_heading_from_path`            | Will use the orientation of the path poses, if set to true. Will compute orientations from path (x,y) points otherwise, only set to true if using a planner that takes robot heading into  account like [Smach Planner](https://docs.nav2.org/configuration/packages/configuring-smac-planner.html).|

### Feature Parameters
These parameters are used to tune and control the behaviour of 
| Feature | Parameter | Description |
|---------|-----------|-------------|
| Adaptive Lookahead Distance | `use_velocity_scaled_lookahead_dist` | enable/disable |
|| `min_lookahead_dist`               | Minimum lookahead distance.                                      |
|| `max_lookahead_dist`               | Maximum lookahead distance.                                      |
|| `lookahead_time`                   | Time factor for velocity-scaled lookahead distance.              |
| Collision Aware Linear Velocity | `use_collision_detection`          | Enable/disable collision detection. |
|| `use_cost_regulated_linear_velocity_scaling` | Enable/disable cost-regulated linear velocity scaling. |
|| `inflation_cost_scaling_factor`    | Factor for inflation cost scaling.                               |
|| `max_allowed_time_to_collision_up_to_target` | Maximum time allowed for collision checking.           |
|| `cost_scaling_dist`                | Distance for cost-based velocity scaling.                        |
|| `cost_scaling_gain`                | Gain factor for cost-based velocity scaling.                     |
| Approach Aware Linear Velocity | `approach_velocity_scaling_dist` | The distance to goal at which velocity scaling will begin. Set to 0 to disable. |
|| `min_approach_linear_velocity` | The minimum velocity this scaling can produce. |
| On Point Rotation | `use_rotate_to_heading` | Enable/disable rotate-to-heading behavior. Will override reversing if both are enabled. |
|| `rotate_to_heading_angular_vel`    | Angular velocity for rotating to heading.                        |
|| `rotate_to_heading_min_angle`      | Minimum angle to trigger rotate-to-heading behavior.             |
|| `max_angular_accel`                | Maximum angular acceleration.                                    |
| Optional Reversing | `allow_reversing`                | Will move in reverse if the lookahead point is behind the robot. |

## Default Parameters
```yaml
controller_server:
  ros__parameters:
    use_sim_time: True
    controller_frequency: 20.0
    min_x_velocity_threshold: 0.001
    min_y_velocity_threshold: 0.5
    min_theta_velocity_threshold: 0.001
    failure_tolerance: 0.3
    progress_checker_plugin: "progress_checker"
    goal_checker_plugins: ["general_goal_checker"]
    controller_plugins: ["FollowPath"]

    # Progress checker parameters
    progress_checker:
      plugin: "nav2_controller::SimpleProgressChecker"
      required_movement_radius: 0.25
      movement_time_allowance: 10.0

    # Goal checker parameters
    general_goal_checker:
      plugin: "nav2_controller::SimpleGoalChecker"
      xy_goal_tolerance: 0.25
      yaw_goal_tolerance: 0.25
      stateful: True
    
    FollowPath:
      plugin: "vector_pursuit_controller::VectorPursuitController"
      p_gain: 6.0
      desired_linear_vel: 0.5
      min_turning_radius: 0.2
      lookahead_dist: 0.4
      min_lookahead_dist: 0.3
      max_lookahead_dist: 0.9
      lookahead_time: 1.5
      rotate_to_heading_angular_vel: 1.8
      transform_tolerance: 0.1
      use_velocity_scaled_lookahead_dist: false
      min_linear_velocity: 0.05
      min_approach_linear_velocity: 0.05
      approach_velocity_scaling_dist: 0.6
      max_allowed_time_to_collision_up_to_target: 1.0
      use_collision_detection: true
      use_cost_regulated_linear_velocity_scaling: true
      cost_scaling_dist: 0.6
      cost_scaling_gain: 1.0
      inflation_cost_scaling_factor: 3.0
      use_rotate_to_heading: true
      allow_reversing: false
      rotate_to_heading_min_angle: 0.5
      max_angular_accel: 3.2
      max_linear_accel: 2.0
      max_lateral_accel: 0.2
      max_robot_pose_search_dist: 10.0
      use_interpolation: true
      use_heading_from_path: false
      approach_velocity_scaling_dist: 1.0
      min_approach_linear_velocity: 5.0
```

## Quickstart Tutorial
We require a robot running the Nav2 stack to use the Vector Pursuit Controller. This example will utilise [BCR Bot](https://github.com/blackcoffeerobotics/bcr_bot), a simulated differential drive robot with a pre-configured Nav2 stack. An installation of [ROS2 Humble](https://docs.ros.org/en/humble/Installation.html) is needed along with [Git](https://git-scm.com/downloads) on an Ubuntu Machine.

1. Create a ROS2 workspace and clone the repositories into this workspace.
	```bash
	mkdir -p ~/ros2_ws/src && cd ~/ros2_ws/src
    
	git clone https://github.com/blackcoffeerobotics/vector_pursuit_controller.git
	git clone https://github.com/blackcoffeerobotics/bcr_bot.git

	# install package dependencies
	cd ~/ros2_ws && rosdep install --from-paths src --ignore-src -r -y
	```

2. Install other ROS2 dependencies required for this tutorial.
	```bash

	# fortress
	sudo apt-get install ros-humble-ros-gz-sim ros-humble-ros-gz-bridge ros-humble-ros-gz-interfaces

	# nav2
	sudo apt-get install ros-humble-navigation2 ros-humble-nav2-bringup
	```

3. Edit the [nav2_params.yaml](https://github.com/blackcoffeerobotics/bcr_bot/blob/ros2/config/nav2_params.yaml) file in the bcr_bot repository to replace `controller_server` parameters, copy these parameters from the [default parameters](#default-parameters) section.

4. Build the workspace and source it.
	```bash
	cd ~/ros2_ws && colcon build && source install/setup.bash
	```

5. Run the nodes in two different terminals.
	```bash
	# in terminal 1
	ros2 launch bcr_bot ign.launch.py

	# in terminal 2, make sure to source the workspace
	ros2 launch bcr_bot nav2.launch.py
	```

## Vector Pursuit Algorithm

Vector pursuit is a geometric path-tracking method that uses the theory of screws. It is similar to other geometric methods in that a look-ahead distance is used to define a current goal point, and then geometry is used to determine the desired motion of the vehicle. On the other hand, it is different from current geometric path-tracking methods, such as follow-the-carrot or pure pursuit, which do not use the orientation at the look-ahead point. Proportional path tracking is a geometric method that does use the orientation at the look-ahead point. This method adds the current position error multiplied by some gain to the current orientation error multiplied by some gain, and therefore becomes geometrically meaningless since terms with different units are added. Vector pursuit uses both the location and orientation of the look-ahead point while remaining geometrically meaningful. 
It first calculates two instantaneous screws for translation and rotation, and then combines them to form a single screw representing the required motion. Then it calculates a desired turning radius from the resultant screw.

The centerlines of the instantaneous screws are constrained to lie on the vehicle's y-axis to accomodate one of the non-holonomic contraints. The screw for correcting the translational error, \$<sub>t</sub> is chosen as the center of a circle which intersects the origins of both the vehicle coordinate system and the look-ahead coordinate system and is tangent to the vehicle's current orientation. The screw for correcting the rotational error, \$<sub>r</sub> is chosen to be centered at the vehicle frame origin so that no translation is associated with it.

![translation_screw](https://github.com/user-attachments/assets/67652137-2cf9-41d9-86a9-e43119fae68b)

### Code Coverage
To generate an HTML webpage to view code coverage use:
```bash
genhtml code_coverage_report.info --output-directory ~/vector_pursuit_code_coverage_report
```
