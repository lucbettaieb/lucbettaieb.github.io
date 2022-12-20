---
layout: post
title:  "ROS nodes for ROS haters"
date:   2022-09-10 22:38:37 -0700
categories: update
---

**This post is a work in progress!  Please forgive any typos or code errors.  I just want to get the idea down for now and will come back to make it totally correct later.**

## Motivation
I've been thinking about writing this post for a while now.  I've been using ROS, the Robot Operating System, with varying degrees of intensity over the past 9 years.  It was how I was introduced to learning about distributed systems programming techniques used for robotics, and the software paradigms that it enforces have stuck with me and helped form how I view a lot of software.  The middleware itself, at least ROS 1, has it's definite downsides -- some of which have been addressed in ROS 2, but one set of issues take the cake for me.  Building your software with the middleware baked in.

ROS developers have all seen this and are often guilty of it, especially when trying to get something to just work.  We write a node and just start gluing things to it.  Maybe bringing in some other libraries when we need to, but keeping all of the application logic baked into the ROS-sy bits.

This, in my humble opinion, really makes things messy.  You're forced to use rostest, you are stuck with needing to use ROS to run the executable, and heavens forbid what would happen if you wanted to switch to a different middleware.

The best solution I've found to this is to make sure you split your ROS executable and any application logic.  I'd go as far as to decouple the ROS datatypes that come in via messages.  This might lead to a lot of boilerplate, code, and it's definitely not the right solution all the time, but when a project is quite large with lots of moving pieces, it can be so much cleaner.

## Approach

If you think this approach might be useful for your project, follow this design pattern.  Write your ROS node in the following way. (C++)


```cpp

/**
 * Main ROS executable for the ThingDoer node
 */

// C++ Standard Library

// ROS
#include <ros/ros.h>

// Thing Doer
#include "thing_doer_params.h"
#include "thign_doer_ros.h"

int main(int argc, char const *argv[]) {  // ROS styleguide is weird
  ros::init(argc, argv, "thing_doer_node");
  ros::NodeHandle nh;

  // Maybe grab some parameters for your thing doer
  ThingDoerParams params(nh);

  // ROS interface for the thing doer.  No core logic here, just ROS boilerplate that can be changed if needed.
  ThingDoerROS thing_doer_ros(nh, params);

  // Normal ROS "do the thing" code.
  ros::Rate r(params.service_hz);
  while (ros::ok()) {
    thing_doer_ros.execute();

    r.sleep();
    ros::spinOnce();
  }

  return 0;
}

```

You can see here, all the node is doing is wrapping a ROS interface for for some class called "`thing_doer`" and is loading in some parameters for that "`thing_doer`".  Let's take a quick look at the parameter object.

```cpp
// C++ Standard Library
#include <string.h>  // Include what you use!

// ROS
#include <ros/ros.h>

// I like using structs for params since we probably don't need any private members.
struct ThingDoerParams {
  /// A quick comment as to what this parameter does
  const unsigned int service_hz;
  const unsigned int kServiceHzDefault = 10;  // Default for the param!

  /// Another nice comment as to what this thing is
  const std::string string_thing_topic;
  const std::string kStringThingTopicDefault = "/string_thing";

  /// Another comment!
  const std::string thing_doer_topic;
  const std::string kThingDoerTopicDefault = "/string_thing_done"

  // Default constructor
  ThingDoerParams() :
    service_hz(kServiceHzDefault),
    string_thing_topic(kStringThingTopicDefault),
    thing_doer_topic(kThingDoerTopicDefault) 
  {}

  // ROS Parameter server constructor
  ThingDoerParams(const ros::NodeHandle& nh) :
    service_hz(nh.param("service_hz"), kServiceHzDefault),
    string_thing_topic(nh.param("string_thing_topic"), kStringThingTopicDefault),
    thing_doer_topic(nh.param("thing_doer_topic", kThingDoerTopicDefault)) 
  {}
};
```

Again, some boilerplate, but an easy-enough template to follow for nice results!  Bonus points -- you can add in whatever kind of constructor you want, and use that in the node.  This is quite powerful if you want to ditch the ROS parameter server (which I think you should do) and use something else.  I've used picojson and yamlcpp in the past as specific libraries (with a bit of wrapping) to load in params.

Now let's take a look at the ROS wrapper for the actual library.

```cpp

// ROS
#include <ros/ros.h>

// ThingDoer
#include "thing_doer_params.h"
#include "thing_doer.h"

class ThingDoerROS
{
public:
  ThingDoerROS(const ros::NodeHandle& nh, const ThingDoerParams& params) :
    nh_(nh),
    params_(params),
    thing_doer_(params) {
    sub_string_thing_ = nh.subscribe
  }

  ~ThingDoerROS() = default;

  void execute() {
    // More application specific logic could go here!
    thing_doer_.execute();
  }

private:
  /// NodeHandle to handle nodes, ya dig?
  ros::NodeHandle nh_;

  /// Params for the thing doer, all adjustable things go here
  ThingDoerParams params_;

  /// The actual core library that does things
  ThingDoer thing_doer_;

  /// ROS interface for subscribing to the string thing
  ros::Subscriber sub_string_thing_;

  void cbStringThing(const std_msgs::String& msg) {
    thing_doer_.cbStringThing(std::string(msg.data));
  }
};


```

You can see that all this piece of code is doing is injecting the non-ros bit of data into a callback internal to the library that's doing the thing.  This is where the line of differentiation exists between the ROS-sy bits and the bits that can be portable.  The library can be unit-tested without the weird ROS ecosystem in the way and can live on and be integrated into whatever middleware your project may be using in the future.

```cpp

// ThingDoer
#include "thing_doer_params.h"

class ThingDoer
{
public:
  ThingDoer(const ThingDoerParams& params) :
    params_(params) {
  }

  ~ThingDoer() = default;

  void execute() {
    if (data_ != kDefaultData) {
      std::cout << "Data: " << data_ << std::endl;
    }
  }

  void cbStringThing(const std::string& data) {
    data_ = data;
  }

  const std::string kDefaultData = "kDefaultData";

private:
  /// Params for the thing doer, all adjustable things go here
  ThingDoerParams params_;

  /// Some piece of data
  std::string data_ = kDefaultData;
};


```

And there ya go, an extremely boilerplate way of printing out some std_msgs::String received on some topic.  The example may be trivial and a bit ridiculous, but I hope you may have gotten some value out of the design paradigm!

