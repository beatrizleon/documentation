

# sr_manipulation

This stack contains packages for performing grasping and manipulation (currently only grasping). See below for an overview of the functionality provided.

## Grasping

The purpose of the grasping functions provided by this stack are to plan quickly and execute a "best guess" of grasp for a given object. Currently, recognition of object is by ar_marker, but this could easily be extended to include other methods of recognition.

Once recognised, particular object types are associated to one or more possible grasp. The best available grasp can then be chosen and executed.

The stages of this process are outlined below.

### Launching

To start the object recognition and grasp building code, run ```roslaunch sr_manipulation_launch grasp_selector.launch```.

### Grasp Conductor

The simplest method for using the grasping functions is to use ```sr_grasp_conductor```. This is a collection of state machines which can be used/combined to execute grasps/pick and place.

Below is an example which picks a specific object (ID ```coke_can_0```) and moves it to a specified location.

Firstly define various objects required for controlling the robot, as well as for manipulating transforms.

```python
rospy.init_node('pick_and_place_with_grasping_state_machine')

hand_finder = HandFinder()
hand_parameters = hand_finder.get_hand_parameters()
hand_serial = hand_parameters.mapping.keys()[0]
hand_commander = SrHandCommander(hand_parameters=hand_parameters,
                                 hand_serial=hand_serial)
arm_commander = SrArmCommander()

tf_listener = tf.TransformListener()
```
Define pose to place object in.
```python
release_pose = PoseStamped()
release_pose.header.frame_id = 'world'
release_pose.pose.position.x = 0.7
release_pose.pose.position.y = -0.5
release_pose.pose.position.z = 0.7

release_pose.pose.orientation.x = 0.5
release_pose.pose.orientation.y = 0.5
release_pose.pose.orientation.z = 0.5
release_pose.pose.orientation.w = 0.5
```
Each state machine is defined and executed in turn. If execution of a given stage of the process fails, ```outcome``` is set to 'failed'.
```python
grasp_selector = GetGraspSM('coke_can_0', hand_commander)
outcome = grasp_selector.execute()

if "failed" == outcome:
    rospy.logerr("Grasp selection failed")
    exit(-1)

selected_grasp = grasp_selector.get_grasp()

approach_grasp = GraspApproachSM(selected_grasp, hand_commander, arm_commander, tf_listener)
outcome = approach_grasp.execute()

if "failed" == outcome:
    rospy.logerr("Grasp approach failed")
    exit(-1)

do_grasp = GraspSM(selected_grasp, hand_commander, arm_commander, tf_listener)
outcome = do_grasp.execute()

if "failed" == outcome:
    rospy.logerr("Grasp failed")
    exit(-1)

retreat = RetreatSM(selected_grasp, hand_commander, arm_commander, tf_listener)
outcome = retreat.execute()

place = PlaceSM(selected_grasp, hand_commander, arm_commander, tf_listener, release_pose)
outcome = place.execute()
```
The state machines for each stage above could equally be combined into one state machine and executed in one go.

### Selecting Grasps

The service ```grasp_for_known_object``` can be used to retrieve a list of possible grasps for a given type of object. The object types are defined as explained [below](#other-grasp-information--object-associations).

The arguments to the service call are:
* Object type.
* The serial number of the hand used for grasping (if this is omitted, the first available hand will be used).
* The id of the object frame.

A [moveit_msgs/Grasp](http://docs.ros.org/indigo/api/moveit_msgs/html/msg/Grasp.html) is returned.

### Grasp Definitions

Grasps are defined as ```moveit_msgs/Grasp``` messages. They contain various information to help execute a grasp.

 * Firstly, postures define how the hand/gripper will be shaped while approaching and grasping an object.
 * Secondly, a pose for the gripper in the frame of the object. This defines the relative position of hand an object during the grasp.
 * Thirdly, approach and retreat gripper translations define how the gripper will move in to attempt the grasp, how it will retreat once the object is grasped, and (as yet unimplemented) how the gripper should retreat once the object has been released again. These are also defined in the object frame.
 * Finally, a measure of grasp quality is required. This will assist in choosing which to use of the possible grasps for a given object.


#### Grasp Postures
The pre-grasp and grasp postures are stored as named robot states in the moveit warehouse database. During the grasping process, the gripper is moved into pre-grasp posture before approaching the object. The grasp-posture is then applied to close the gripper onto the object.

The best way to save new postures into the database is via the Shadow Grasp Controller rqt plugin.

#### Other Grasp information + Object Associations

The gripper pose and approach/retreat vectors are defined relative to the objects we might try to grasp. They are defined in a yaml file ([sr_manipulation_launch/config/grasp_mapping.yaml](https://github.com/shadow-robot/sr_manipulation/blob/indigo-devel/sr_manipulation_launch/config/grasp_mapping.yaml)) which also contains the list of known objects. An example object + grasp definition is shown below:

```yaml
object_to_grasp_mapping:
    coke_can:
        -
            pre_grasp: super_amazing_grasp_open
            grasp: super_amazing_grasp_closed
            pose: [0.1, 0.1, 0.1, 0, 0, 0, 1]
            pre_grasp_approach: [-0.1, 0, 0]
            post_grasp_retreat: [0.05, 0, 0]
            quality: 0.1
        -
            pre_grasp: super_amazing_grasp_open
            grasp: super_amazing_grasp_closed
            pose: [0.1, -0.1, 0.1, 0, 0, 0, 1]
            pre_grasp_approach: [-0.1, 0, 0]
            post_grasp_retreat: [0.05, 0, 0]
            quality: 0.2
        -
            pre_grasp: super_amazing_grasp_open
            grasp: super_amazing_grasp_closed
            pose: [-0.05, 0.0, 0.1, 0.5, 0.5, 0.5, 0.5]
            pre_grasp_approach: [0, 0, -0.1]
            post_grasp_retreat: [0, 0, 0.05]
            quality: 0.3
```
Any number of grasps can be defined for a particular object type. In the example above, each of the three grasps use the same postures and only differs by the gripper pose and approach/retreat vectors. This need not be the case however.

### Object Recognition

Recognised objects are published on the topic ```/known_objects``` Messages are of type [sr_manipulation_msgs/AvailableObjects](https://github.com/shadow-robot/sr_manipulation/blob/indigo-devel/sr_manipulation_msgs/msg/AvailableObjects.msg).

Objects are defined by their type and a unique ID. The ID is of the format ```object_type_n``` where n is an index starting at 0 and increasing by one for each object of that type identified. For example, if there are two coke cans and a tennis ball, they will have IDs ```coke_can_0```, ```coke_can_1``` and ```tennis_ball_0```.

#### Ar Tracking

Currently objects are recognised by being attached to ar_markers. Marker IDs are asscociated with object types via a yaml file ([sr_manipulation_launch/config/object_ar_mapping.yaml](https://github.com/shadow-robot/sr_manipulation/blob/indigo-devel/sr_manipulation_launch/config/object_ar_mapping.yaml))

```yaml
ar_marker_to_object_mapping:
    - marker_id: ar_marker_0
      type: coke_can
      transform: [0, 0, 0, 0, 0, 0, 1]

    - marker_id: ar_marker_1
      type: tennis_ball
      transform: [0, 0, 0, 0, 0, 0, 1]
```

The marker id also gives the name of the frame whose transform is published by the ar_track_alvar node. The transforms specified above give the tf from the marker's frame to the object's frame, named named with the object's unique ID.

## Grasp Teacher

The grasp teaching gui can be launched with the command

```
roslaunch sr_manipulation_grasp_teacher teach_gui.launch
```

To run the gui, you need a hand and arm (real or simulated), a state warehouse with associated services, and a ```known_objects``` topic as described above being published. Grasps using this gui will be saved in the file ```object_grasp_mapping.yaml``` as explained above. Hand states recorded during the process are stored in the moveit warehouse.
