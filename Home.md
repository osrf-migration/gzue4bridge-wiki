# Gazebo and Unreal Engine Integration

## Architecture

Here's an overview of the different components in the Gazebo + Unreal integration setup

* gzue4bridge: A plugin that will be loaded when the unreal project is launched. This plugin should operate in both editor mode and game mode. It is responsible for communicating with Gazebo over the `gzbridge` websocket server.

* gzbridge: A websocket server that converts protobuf messages published by the Gazebo simulation server to JSON format and republish them over port 8080. Similarly, it can receive messages in JSON format from websocket clients (e.g. `gzue4bridge`) and convert them to protobuf messages before sending them to Gazebo.


# Communication Channel

Gazebo provides publish/subscribe communication over named topics in the same way as ROS. These topics form the main communication channel between Gazebo and Unreal and are used to keep the two worlds in synchronization.

## Topics

The `gzue4bridge` plugin subscribes to the following topics to create gazebo entities in the Unreal world and update their pose.

* `~/scene`: provides initial scene information. The message contains a scene tree of all objects in the world. There should typically only be one message received on this topic after a client connects. 

* `~/model/info`: provides information about a new or existing model in gazebo. Each time a model is added to the world, a message containing all its properties including links, joints, visuals, and nested models is published to this topic. Similarly, when a model property is modified, a message is also published to this topic.

* `~/pose/info`: provides timestamped pose information of entities in gazebo. An entity can be a model, link, or collision. The message will only contain entities whose pose have changed in that time step. The rate at which the messages are published to this topic can be configured through gzbridge. For lock-stepping Unreal with Gazebo, it is important to receive the pose messages at full rate, i.e. 1000Hz.

The `gzue4plugin` publishes to the following gazebo topics to create Unreal objects in Gazebo and keep the pose in sync.

* `~/factory`: creates a model in gazebo and place them at the specified location. The message takes in an SDF string that describes the model to be created. 

* `~/model/modify`: updates the properties of a model in gazebo, including the properties of its links, joints, and collisions. 

## Synchronization

Synchronization between the two worlds are currently achieved by exchanging messages over the Gazebo topics described above.

### Gazebo to Unreal

**Initialization**: The `gzue4bridge` plugin subscribes to the `~/scene` topic and creates Unreal actors representing these Gazebo models in the Unreal world. 

**Pose Sync**: The `gzue4bridge` subscribes to the `~/pose/info` topic and updates the actors and their components in the Unreal world based on the received pose data.


## Unreal to Gazebo

**Initialization**: The `gzue4bridge` plugin publishes to the `~/factory` topic to create models in Gazebo that represents the actors in Unreal. Currently only Unreal's skeletal mesh actors are created in Gazebo.

**Pose Sync**: The `gzue4bridge` plugin publishes to the `~/model/modify` topic to update the pose of Gazebo models and links that correspond to their counterparts in Unreal. Currently only the pose of Unreal's skeletal mesh actors are kept in sync.