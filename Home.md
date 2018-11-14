# Gazebo and Unreal Engine Integration

## Architecture

Here's an overview of the different components in the Gazebo + Unreal integration setup

![Lab126 GzUE4Bridge Architecture.jpg](https://bitbucket.org/repo/XXGa7Rg/images/3523798422-Lab126%20GzUE4Bridge%20Architecture.jpg)

Apart from Unreal and Gazebo, here are the two main components:

* gzue4bridge: A plugin that will be loaded when the unreal project is launched. It is responsible for communicating with Gazebo over the `gzbridge` websocket server.

* gzbridge: A websocket server that converts protobuf messages published by the Gazebo simulation server to JSON format and republish them over port 8080. Similarly, it can receive messages in JSON format from websocket clients (e.g. `gzue4bridge`) and convert them to protobuf messages before sending them to Gazebo.


# Communication Channel

Gazebo provides publish/subscribe communication over named topics in the same way as ROS. These topics form the main communication channel between Gazebo and Unreal and are used to keep the two worlds in synchronization.

## Synchronization

Here's an overview of the basic synchronization process taken by the `gzue4bridge` plugin. Synchronization is done in both ways: from Gazebo to Unreal and vice versa. More information about the topics can be found in `Gazebo Topics` section.

### Gazebo to Unreal

**Initialization**: The `gzue4bridge` plugin subscribes to the `~/scene` topic and creates Unreal actors representing these Gazebo models in the Unreal world. 

**Model Sync**: The `gzue4bridge` plugin subscribes to the `~/model/info` topic and creates a new Unreal actor whenever a message containing an unseen model is received.

**Pose Sync**: The `gzue4bridge` plugin subscribes to the `~/pose/info` topic and updates the actors and their components in the Unreal world based on the received pose data.

## Unreal to Gazebo

**Initialization**: The `gzue4bridge` plugin publishes to the `~/factory` topic to create models in Gazebo that represent the actors in Unreal. Currently only Unreal's skeletal mesh actors are created in Gazebo.

**Model Sync**: The `gzue4bridge` plugin publishes to the `~/factory` topic to create new models in Gazebo.

**Pose Sync**: The `gzue4bridge` plugin publishes to the `~/model/modify` topic to update the pose of Gazebo models and links that correspond to their counterparts in Unreal. Currently only the pose of Unreal's skeletal mesh actors are kept in sync.


## Gazebo Topics

The topics below are not an exhaustive list of all available Gazebo topics. These are the topics that are currently used by the `gzue4bridge` plugin. 

* `~/scene`: provides initial scene information. The message contains a scene tree of all objects in the world. There should typically only be one message received on this topic after a client connects. 

* `~/model/info`: provides information about a new or existing model in gazebo. Each time a model is added to the world, a message containing all its properties including links, joints, visuals, and nested models is published to this topic. Similarly, when a model property is modified, a message is also published to this topic.

* `~/pose/info`: provides timestamped pose information of entities in gazebo. An entity can be a model, link, or collision. The message will only contain entities whose pose have changed in that time step. The rate at which the messages are published to this topic can be configured through gzbridge. For lock-stepping Unreal with Gazebo, it is important to receive the pose messages at full rate, i.e. 1000Hz.

* `~/factory`: creates a model in gazebo and place them at the specified location. The message takes in an SDF string that describes the model to be created. 

* `~/model/modify`: updates the properties of a model in gazebo, including the properties of its links, joints, and collisions. 

* `~/world/control`: commands are sent over this topic to step, pause, and play the Gazebo physics simulation.

## Lock-Stepping

The main strategy for lock-stepping Unreal with Gazebo is as follows:

1. Begin simulation with Unreal and Gazebo paused.

1. For any new actors in Unreal, the `gzue4bridge` plugin sends factory messages to Gazebo to create models representing these actors. It waits until the Gazebo models have been created.

1. The `gzue4bridge` creates Unreal actors to represent any new models in Gazebo

1. For any pose changes of actors in Unreal, the `gzue4bridge` plugin sends pose messages to Gazebo to update models representing these actors. It waits and verifies that the pose of Gazebo models have been updated.

1. The `gzue4bridge` plugin updates the pose of Unreal actors representing Gazebo models based on new pose messages received from Gazebo.

1. The `gzue4bridge` waits until it receives a Gazebo timestamp that matches the expected sim time. The two worlds are now in sync.

1. The `gzue4bridge` allows the Unreal actors to take one step and also sends a message to Gazebo to take one step.

1. Repeat from step 2.

# Videos

Currently, the work is more of a prototype as the result of a feasibility study than a complete fully working integration. Here're a few videos:

https://youtu.be/9qHb_AXWSOA
https://youtu.be/Lbo8pNLH3Co
https://youtu.be/OjTNTiBuRDY