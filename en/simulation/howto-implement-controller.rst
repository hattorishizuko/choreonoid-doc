
Implementation of Controller
===============================

.. sectionauthor:: 中岡 慎一郎 <s.nakaoka@aist.go.jp>

.. contents::
   :local:

.. highlight:: cpp


Implementation of Controller
-------------------------------

The base of how to implement a controller is described below:

What a controller does is basically the following three things and it executes them repeatedly as "control loop".

1. To input the status of the robot;
2. To make control calculation; and
3. To output an order to the robot

These processes are executed by a single controller or in combination with multiple software components. A process where "control calculations" are bundled actually involves diversified processes like different recognitions and motion plans and my include inputs/outputs for something else than the robot. However, from the robot perspective, what a controller does can be sorted out to the above-mentioned three processes.

From this perspective, a controller is a software module having interfaces that handle the above three actions. The actual API for this varies among the controller formats but the essential part is identical.

Here, an explanation is provided using "SR1MinimumController" sample, which was also used in :doc:`howto-use-controller` . The format of the controller is the "simple controller" format designed for the samples in Choreonoid and its content of control is just to maintain the robot's posture. The descriptive language used is C++.

When actually developing a controller, the basics that were provided using this sample can be replaced with the desired controller format and the content of control. Generally speaking, diversified knowledge and skills related to control, programming, hardware, etc. are required to develop a controller for a robot. Many of those skills are out of scope of this manual. Study the knowledge and the skills required separately.


Source Code of Sample Controller
------------------------------------

First, the source code of SR1MinimumController is as follows: This source code is a file called "SR1MinimumController.cpp" under the directory "sample/SimpleController" of Choreonoid sources. ::

 #include <cnoid/SimpleController>
 #include <vector>
 
 using namespace cnoid;
 
 const double pgain[] = {
     8000.0, 8000.0, 8000.0, 8000.0, 8000.0, 8000.0,
     3000.0, 3000.0, 3000.0, 3000.0, 3000.0, 3000.0, 3000.0, 
     8000.0, 8000.0, 8000.0, 8000.0, 8000.0, 8000.0,
     3000.0, 3000.0, 3000.0, 3000.0, 3000.0, 3000.0, 3000.0, 
     8000.0, 8000.0, 8000.0 };
    
 const double dgain[] = {
     100.0, 100.0, 100.0, 100.0, 100.0, 100.0,
     100.0, 100.0, 100.0, 100.0, 100.0, 100.0, 100.0,
     100.0, 100.0, 100.0, 100.0, 100.0, 100.0,
     100.0, 100.0, 100.0, 100.0, 100.0, 100.0, 100.0,
     100.0, 100.0, 100.0 };
 
 class SR1MinimumController : public SimpleController
 {
     BodyPtr body;
     double dt;
     std::vector<double> qref;
     std::vector<double> qold;
 
 public:
 
     virtual bool initialize()
     {
         body = ioBody();
         dt = timeStep();
 
         for(int i=0; i < body->numJoints(); ++i){
             qref.push_back(body->joint(i)->q());
         }
         qold = qref;
         
         return true;
     }
 
     virtual bool control()
     {
         for(int i=0; i < body->numJoints(); ++i){
             Link* joint = body->joint(i);
             double q = joint->q();
             double dq = (q - qold[i]) / dt;
             double u = (qref[i] - q) * pgain[i] + (0.0 - dq) * dgain[i];
             qold[i] = q;
             joint->u() = u;
         }
         return true;
     }
 };
 
 CNOID_IMPLEMENT_SIMPLE_CONTROLLER_FACTORY(SR1MinimumController)


As for compile, it is described in: ::

 add_cnoid_simple_controller(SR1MinimumController SR1MinimumController.cpp)

in CMakeList.txt unde the same directory. See "src/SimpleControllerPlugin/library/CMakeLists.txt" for detail of this function. Basically, it is OK to link with the library "CnoidSimplerController". (In case of Linux, the file name of the library will be "libCnoidCimpleController.so".

SimpleController Class
-------------------------

A controller of simple controller format is implemented by inheriting SimpleController class. This class becomes available by including cnoid/SimpleController head by ::

 #include <cnoid/SimpleController>

.

The relevant part to the description of this section in the definition of SimpleController class is as follows: (The actual class definition is made in "src/SimpleControllerPlugin/library/SimpleController.h" of Choreonoid sources. See it for your reference.) ::

 class SimpleController
 {
 public:
     virtual bool initialize() = 0;
     virtual bool control() = 0;

 protected:
     Body* ioBody();
     double timeStep() const;
     std::ostream& os() const;
 };


This class has the following pure virtual function as a member function:

* **virtual bool initialize()**

 Initialise the controller:

* **virtual bool control()**

 Perform input, control and output processes of the control. This function will be executed repeatedly as a control loop under control.

To implement the controller, define a class that inherits SimpleController class. Implement the processes of the controller by overriding the above function here.

Also, SimpleController class is equipped with the following protected member functions:

* **Body\* ioBody()**

 It returns a Body object to be used for input/output.

* **double timeStep() const**

 It returns the time step of the control. The above control function is called repeatedly under control with this time interval.

* **std::ostream& os() const**

 It returns an output stream to output a text. By outputting to this stream, a text message can be displayed on the message view of Choreonoid.
 
These member functions can be used in the above-mentioned initialise() and control() functions.

Once you define a class inheriting SimpleController function, you need to define its factory function. You can describe it using a macro as follows:

CNOID_IMPLEMENT_SIMPLE_CONTROLLER_FACTORY(SR1MinimumController)

With this factory function, the binary file built from this source becomes available from the simple controller item.


Body Object
----------------

The simple controller inputs and outputs via a "Body item" returned by ioBody(). A Body object is an internal expression of Choreonoid of :doc:`../handling-models/bodymodel`, and an instance of "Body class" defined in C++. Since a Body class has data structure storing the status of the body model, elements like joint angle, torque and sensor status subject to output can of course be stored. The simple controller inputs and outputs via this Body class object.

.. note:: A Body class has various information and functions related to the body model, so it is an over-qualified class for input/output only. This type of class is not usually used for an input/output interface. Generally, a data structure optimised for exchanging only input/output elements is used. So, please be reminded of this point when you apply the description of this section to other controller formats. For example, RT component of OpenRTM normally uses "data port" interface for input/output by data type.

Joint Angle and Joint Torque
--------------------------------

The joint angle and the joint torque are the fundamental input and output elements to control a robot. With these elements, each joint can be motioned by PD control. In that case, the joint angle is the output value from the robot and the joint torque is an output order to the robot. You had better check first how these values are input and output in the controller format to be used.

To perform input/output in the simple controller, the "Link object" of the corresponding joint is used. A Link object is an instance of "Link class" that expresses each link of the body model and it can be retrieved from the Body object using the following member function:

* **int numJoints() const**

 It returns the number of the functions owned by the model.

* **Link\* joint(int id)**

 It returns the Link object that corresponds to the joint number.


For the Link object retrieved, it is possible to access to the joint status value using the following member function.

* **double& q()**

 It returns the reference to the double value of the joint angle. The unit is radian. As it is a reference, you can substitute another value.

* **double& u()**

 It returns the reference to the double value of the joint torque. The unit is [N･m]. As it is a reference, you can substitute another value.

For a simple controller, input/output is performed using the above-mentioned function to the Link object retrieved from ioBody(). That is to say, you can input the current joint angle by reading q() and output the torque order value to the robot by writing u().

.. note::  Regarding the input of the joint angle, the above-mentioned q() returns the true value of the model and the value actually input in the robot depends on the accuracy of the encoder. Additional processes are required if you want to reflect the accuracy of the encoder in a simulation, too. As for output of the order value, the actual robot has diversified forms like the joint angle, the ampere value, etc., but, in a simulation, they have to be output as torque values eventually. However, some simulator items have "high gain" mode that allows outputting the target angle as an order value.


Initialisation Process
--------------------------

initialize() function of SimpleController inheriting class initialises the controller.

In the sample, the Body object for input and output is obtained with: ::

 body = ioBody();

. This object will be accessed repeatedly, so it is stored in body variable for efficiency and descriptive simplification for use.

Similarly, the time step value is stored in dt variable with: ::

 dt = timeStep();

for control calculation.

Next, ::

 for(int i=0; i < body->numJoints(); ++i){
     qref.push_back(body->joint(i)->q());
 }
 qold = qref;

This set the robot's joint angle when initialised (when the simulation is started) to a variable called qref where the target joint angle is stored. qold is a variable in which the joint angle one step before is stored and this will also be used for control calculation. qold is initialised to the identical value to qref.

Here the descriptive statement: ::

 body->joint(i)->q()

inputs the joint angle of the i-th angle.

By returning true in the end, it informs the simulator of the successful initialisation.

Control Loop
---------------

SimpleController inheriting class states a control loop in its control() function.

In the sample, control calculation is performed with: ::

 for(int i=0; i < body->numJoints(); ++i){
     ...
 }

for all the joints. The content of this is the processing code.

First, with: ::

 Link* joint = body->joint(i);

the Link object corresponding to the i-th joint is obtained.

Next, input the current joint angle: ::

 double q = joint->q();

Calculate the order value of the joint torque by PD torque. First, calculate the joint angular velocity from the difference between the control loop and the previous joint angle. ::

 double dq = (q - qold[i]) / dt;

Since the control target is to maintain the initial posture, calculate the torque order value with the joint angle being the initial joint angle and the angular velocity being 0 (state of rest) as a target. ::

 double u = (qref[i] - q) * pgain[i] + (0.0 - dq) * dgain[i];

This retrieves the gain value related to each joint from the disposition of pgain and dgain, which were configured in the beginning of the source. The gain value requires tuning for each model, but how to tune it is omitted here.

Save the joint angle in qold variable for next calculation. ::

 qold[i] = q;

Output the calculated torque value. By this, the angles can be controlled so that the initial joint angle can be maintained. ::

 joint->u() = u;

When the above settings are applied to all the joints, the total posture of the robot can be maintained.

Finally, control() function informs the simulator of the continuation of the control by repeating true. Therefore, control() function is called repeatedly.

Devices
----------

In the above example, the joint angle was input and the joint torque was output. In other words, the inputs/outputs are made to the devices like an encoder and en actuator that are equipped in the joint.

There are many other different devices as the target of inputs/outputs. For example, the followings are the target of inputs as a sensor like an encoder:

* Force sensor, acceleration sensor and angular velocity sensor (rate gyroscope)
* Camera and laser range finder
* Microphone

and other devices.

The followings are the target of outputs and work to the external world as an actuator:

* Speaker
* Display
* Light

and other devices.

In the actual controller development, it is necessary to input/output to/from these diversified devices. To do so,

* it is necessary to understand how the devices are defined in the model, and
* how to access the specified devices in the controller format to be used

.. _simulation-device-object:

.

Device Objects
--------------------

In a Body model of Choreonoid, the device information is represented as "Device object". It is an instance that inherits "Device class" and a different type is defined for each different device type. The device types defined as standard are as follows: ::

 + Device
   + ForceSensor 
   + RateGyroSensor  (angular velocity sensor)
   + AccelerationSensor 
   + Camera 
     + RangeCamera (camera + distance image sensor)
   + RangeSensor 
   + Light
     + PointLight 
     + SpotLight 

The device information included in a robot is usually described in a model file. For a model file in OpenHRP format, :ref:`oepnrhp_modelfile_sensors` in :doc:`../handling-models/modelfile/modelfile-openhrp` is described.

In a simple controller, similarly to a Body object, Device objects, which are internal expressions of Choreonoid, are used as they are to the devices for input and output. A Device object can be retrieved from a Body object using the following function:

* **int numDevices() const**

 It returns the number of the devices.

* **Device\* device(int i) const**

 It returns the i-th device. The order of the devices are the order described in the model file.

* **const DeviceList<>& devices() const**

 It returns the list of the devices.

* **template<class DeviceType> DeviceList<DeviceType> devices() const**

 It returns the list of the devices of the type specified.

* **template<class DeviceType> DeviceType\* findDevice(const std::string& name) const**

 It returns any device having the type and the name specified.

Use a template class DeviceList to get the devices of a specific type. DeviceList is an array that stores the device objects of the type specified and it allows extracting only the corresponding type using its constructor, the extraction operator (<<), etc. from DeviceList having other types For example, if you want to retrieve the force sensor owned by the Body object "body", type: ::

 DeviceList<ForceSensor> forceSensors(body->devices());

or add it to the existing list as follows: ::

 forceSensors << body->devices();

.

DeviceList has functions and operators similar to std::vector. For example, with the following: ::

 for(size_t i=0; i < forceSensors.size(); ++i){
     ForceSensor* forceSensor = forceSensor[i];
     ...
 }

different objects can be accessed.

By using findDevice function, you can identify a device with its type and name and get it. For example, SR1 model has an acceleration sensor called "WaistAccelSensor" mounted in the waist link. You can type as follows ::

 AccelerationSensor* waistAccelSensor =
     body->findDevice<AccelerationSensor>("WaistAccelSensor");

to Body object, then you can get it.

The devices that SR1 model has are as follows:

.. tabularcolumns:: |p{3.5cm}|p{3.5cm}|p{6.0}|

.. list-table::
 :widths: 30,30,40
 :header-rows: 1

 * - Name
   - Type of device
   - Description
 * - WaistAccelSensor
   - AccelerationSensor
   - Acceleration sensor mounted in the waist link
 * - WaistGyro
   - RateGyroSensor
   - Gyro mounted in the waist link
 * - LeftCamera
   - RangeCamera
   - Distance image sensor corresponding to the left eye
 * - RightCamera
   - RangeCamera
   - Distance image sensor corresponding to the right eye
 * - LeftAnkleForceSensor
   - ForceSensor
   - Force sensor mounted in the left ankle
 * - RightAnkleForceSensor
   - ForceSensor
   - Force sensor mounted in the right ankle


Input/Output to/from Device
------------------------------

Inputs/outputs to/from a Device object are performed in the following way:

* **Input**

 Obtain the value using the member function of the corresponding Device object.

* **Output**

 Configure the value using the member function of the corresponding Device object and run "notifyStateChange()" function of the Device object.

To do so, you must know the class definition of the device to be used. For example, for "AccelerationSensor", which is the class of an acceleration sensor, there is a member function "dv()" to access to its state. This function returns three-dimension vector in Vector3 type.

Thus, the acceleration of the acceleration sensor waistAccelSensor can be obtained as follows: ::

 Vector3 dv = waistAccelSensor->dv();

.

Similarly, it is possible to input the state using the relevant member function for ForceSensor and RateGyroSensor, too.

Use of visual sensors like a camera or a range sensor requires some preparation. This will be described in :doc:`vision-simulation` .

For output to a device, see the sample "TankJoystickLight.cnoid", which turns on and off the light.

